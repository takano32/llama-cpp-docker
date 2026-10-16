# llama-cpp-docker

箱に1台の共有 [llama.cpp](https://github.com/ggml-org/llama.cpp)（`llama-server`）。公式イメージ `ghcr.io/ggml-org/llama.cpp` の server 版を、
埋め込み用と小型 LLM 用の2本で動かし、3本目に MCP の橋渡しを置く。ollama（[ollama-docker](https://github.com/takano32/ollama-docker)）の置き換え。
本家リポジトリは fork も改変もしない。版は `.env` の `LLAMA_CPP_TAG`（既定 `server-b11681`＝2026-10-16 時点の `:server` と同じ中身、`llama-server --version` は `0.7.0-dev (build 11681)`）。

## 構成

| サービス | コンテナ名 | 中身 | ホスト側の口 | 箱の中の口（external network `llama-cpp`） |
|---|---|---|---|---|
| `embed` | `llama-embed` | `llama-server --embeddings`、embeddinggemma 300m QAT Q8_0（名前 `embeddinggemma`） | `127.0.0.1:8081` | `http://llama-embed:8080/v1` |
| `llm` | `llama-llm` | `llama-server`、Qwen3 1.7B Q4_K_M（名前 `qwen3-1.7b`、thinking 切） | `127.0.0.1:8082` | `http://llama-llm:8080/v1` |
| `mcp` | `llama-mcp` | `mcp-proxy` ＋ `@pyroprompts/any-chat-completions-mcp`（`llm` を MCP の道具1個にする） | `127.0.0.1:8083/mcp` | `http://llama-mcp:8080/mcp` |

- ポートは全部 `127.0.0.1` だけに開けている（外には開けない）。鍵は無し（`--api-key` を付けていない。箱の外から届かない前提）。
- モデルは `llama-server` の `-hf`（Hugging Face からの取得機能）で落とし、名前付きボリューム `llama-cpp_models`（コンテナ内 `/models`＝`LLAMA_CACHE`）に残す。
  `docker compose down` に `-v` を付けなければ消えない。
- メモリ上限（`mem_limit`）は embed 1g、llm 3g、mcp 512m。箱（RAM 12GB、2 vCPU、GPU なし、aarch64）のほかのサービスを守る安全弁。
- llama-server は1プロセス1モデルが基本なので2本に分けた。この版の `--help` には router モード（`--models-dir`、1プロセスで複数モデルを切り替え）もあるが、
  メモリ上限をモデルごとに分けたいので使っていない（試してもいない）。

## 使い方

```sh
cd ~/opt/llama-cpp-docker
docker compose up -d                 # 初回は GGUF のダウンロード（embed 313MB、llm 1.2GB。この回線で数分）
docker compose logs -f llm           # "model loaded" と "listening on" が出れば使える（/health が 200）
curl -s http://127.0.0.1:8081/health; curl -s http://127.0.0.1:8082/health
docker compose down                  # -v を付けなければモデルは残る
```

### 埋め込み（OpenAI 互換 `/v1/embeddings`）

```sh
curl -s http://127.0.0.1:8081/v1/embeddings -H 'Content-Type: application/json' \
  -d '{"model":"embeddinggemma","input":["task: search result | query: 鎌倉に行くのはいつ？","title: none | text: 来週の火曜日に鎌倉へ出かける予定。"]}'
```

- 768 次元、L2 正規化済み（`--embd-normalize` の既定 2）。プーリングは mean（モデルの既定と同じ。`--pooling` を外しても同じ値が返るのを確かめた）。
- 接頭辞は呼ぶ側が付ける。embeddinggemma のカードどおり、問いは `task: search result | query: `、文書は `title: none | text: `。
- 1入力の上限は 2048 トークン（`--ctx-size 4096` をスロット2つで割る。埋め込みは非因果なので `-ub` もそれ以上にしてある）。

### 小型 LLM（OpenAI 互換 `/v1/chat/completions`）

```sh
curl -s http://127.0.0.1:8082/v1/chat/completions -H 'Content-Type: application/json' \
  -d '{"model":"qwen3-1.7b","messages":[{"role":"user","content":"1+1は？数字だけ答えて。"}],"temperature":0,"max_tokens":32}'
```

- thinking は起動時の `--reasoning off` で切ってある（Qwen3 系のテンプレートの `enable_thinking` を false にする）。
  2 vCPU だと thinking は1問 30〜180 秒で、0.8B と 2B は 1024 トークン考え続けて本文が出なかった（下の計測）。
- 文脈長は `--ctx-size 4096`（既定のモデル値 40960 だと KV キャッシュだけで数 GB になる）。スロットは1つ（同時に1リクエスト。次は待つ）。
- 応答の `timings`（llama-server 独自）に prompt／生成のトークン数と ms が入る。

### MCP の橋渡し（Claude Code から `llm` を道具として使う）

`mcp` サービスは、stdio の MCP サーバー `@pyroprompts/any-chat-completions-mcp` を `mcp-proxy` で streamable HTTP にして `127.0.0.1:8083/mcp` に出す。
道具は1つ `chat-with-local-llm`（引数 `content`：文字列。`llm` に1往復の chat を投げて本文を返す）。

```sh
# Claude Code への登録（local スコープ＝そのプロジェクトだけ。~/.claude.json に入る）
claude mcp add --transport http --scope local llama-cpp http://127.0.0.1:8083/mcp
claude mcp list
```

- 名前は `.env` の `LLAMA_MCP_NAME`（既定 `local-llm`。道具名が `chat-with-<名前>` になる）。モデルは `LLAMA_LLM_ALIAS` と同じものを指す。
- 1往復のタイムアウトは `LLAMA_MCP_TIMEOUT`（既定 300000 ms）。2 vCPU の生成は1問で数秒〜数十秒。
- 任意の system prompt は `LLAMA_MCP_SYSTEM_PROMPT`。

## 設定（.env）

| 変数 | 意味 | 既定 |
|---|---|---|
| `LLAMA_CPP_TAG` | イメージのタグ（`ghcr.io/ggml-org/llama.cpp:<タグ>`。arm64 があるのは `server-b<ビルド>` と `server`） | `server-b11681` |
| `LLAMA_EMBED_HF` | 埋め込みモデル（`<repo>[:quant]`。`-hf` に渡す） | `ggml-org/embeddinggemma-300m-qat-q8_0-GGUF` |
| `LLAMA_EMBED_ALIAS` | 埋め込みの API 上の名前 | `embeddinggemma` |
| `LLAMA_EMBED_POOLING` | プーリング（`--pooling`） | `mean` |
| `LLAMA_EMBED_PORT`／`LLAMA_LLM_PORT`／`LLAMA_MCP_PORT` | ホスト側（127.0.0.1）の口 | `8081`／`8082`／`8083` |
| `LLAMA_EMBED_MEM_LIMIT`／`LLAMA_LLM_MEM_LIMIT`／`LLAMA_MCP_MEM_LIMIT` | メモリ上限 | `1g`／`3g`／`512m` |
| `LLAMA_LLM_HF` | 小型 LLM（`<repo>:<quant>`） | `ggml-org/Qwen3-1.7B-GGUF:Q4_K_M` |
| `LLAMA_LLM_ALIAS` | LLM の API 上の名前（MCP の `AI_CHAT_MODEL` にも使う） | `qwen3-1.7b` |
| `LLAMA_LLM_CTX` | LLM の文脈長 | `4096` |
| `LLAMA_LLM_REASONING` | `--reasoning`（`on`／`off`／`auto`） | `off` |
| `LLAMA_THREADS` | 1プロセスのスレッド数 | `2` |
| `LLAMA_MCP_NAME`／`LLAMA_MCP_TIMEOUT`／`LLAMA_MCP_SYSTEM_PROMPT` | 橋渡しの道具名・タイムアウト（ms）・system prompt | `local-llm`／`300000`／（空） |
| `BUN_VERSION`／`MCP_PROXY_VERSION`／`ANY_CHAT_MCP_VERSION` | 橋渡しの版（oven/bun のタグ、npm の版） | `1.4.3-alpine`／`6.7.20`／`0.1.1` |

速さ優先で LLM を替えるなら `.env` に `LLAMA_LLM_HF=ggml-org/Qwen3.5-0.8B-GGUF:Q8_0` と `LLAMA_LLM_ALIAS=qwen3.5-0.8b` を書いて `docker compose up -d llm`
（初回は 794MB のダウンロード）。モデルの名前を変えると、Hindsight など呼ぶ側の `model` も合わせる。

## モデルの選び方（計測は下）

- **埋め込み＝embeddinggemma 300m の QAT Q8_0**（`ggml-org/embeddinggemma-300m-qat-q8_0-GGUF`、313MB、768 次元）。
  ollama のときの比較（12 モデル、hindsight-docker の README の表）で最終1位率・速さ・メモリの全部で最良だったものと同じ元のチェックポイント
  （Google の QAT q8_0 版。ggml-org の `embeddinggemma-300M-GGUF:Q8_0` は bf16 からの事後量子化で別物）。
- **LLM＝Qwen3 1.7B の Q4_K_M**（`ggml-org/Qwen3-1.7B-GGUF:Q4_K_M`、1.2GB）。ollama のときの5問（要約・分類・敬体化・外の事実の判定・JSON 抽出）で、
  thinking なしの4本中いちばんまとも（4問正解）。速さだけなら Qwen3.5 0.8B の Q8_0（1問 2.4 秒）だが、分類と判定を外した。
- GGUF は ggml-org（llama.cpp 本家の置き場）のものを使う。モデルのカードに `llama-server -hf ...` の起動例があり、量子化の由来がはっきりしている。

## 計測（2026-10-16、この箱、llama-server build 11681）

### 埋め込み：Hindsight の日本語 recall（ollama のときと同じ試験）

試験は hindsight-docker の README と同じ（自作 36 文、関係ある問い 42 本＋難しい 26 本、無関係 10＋5 本、LLM なし、リランカー rrf）。
Hindsight 1.0.0 を共有 PostgreSQL の一時データベースに繋ぎ、`openai` プロバイダで `llama-embed` を指した。終わったら一時データベースは消した。
最終1位＝Hindsight が返した順位で狙った文が1位だった率（基本／難しい）、意味1位＝意味スコアだけで並べたときの1位率、recall は1問の中央値、
メモリは待機時の埋め込み側（`docker stats`）。

| 経路 | モデル | 足切り | 最終1位 | 意味1位 | recall | 埋め込み側メモリ | 容量 |
|---|---|---|---|---|---|---|---|
| ollama 0.40.5（同日夕方） | `embeddinggemma:300m-qat-q8_0` | 0.3 | 100% / 92% | 100% / 96% | 0.06s | 627 MiB | 338MB |
| llama-server b11681 | `embeddinggemma`（同じ QAT Q8_0） | 0.3 | 93% / 58% | 100% / 100% | 0.07s | 371 MiB | 313MB |
| llama-server b11681 | 同 | **0.43** | **100% / 88%** | 100% / 100% | 0.07s | 371 MiB | |

- 意味の順位は ollama と同じか少し良い（難しい組の取りこぼし1件が消えた）。Hindsight の最終順位で落ちるのは、コサイン類似度の尺度が ollama より一律に 0.1 ほど高く出るため
  （関係ある文の最小 0.32→0.44、中央値 0.46→0.59、無関係な問いの1位の最大 0.3 未満→0.43）。Hindsight の既定の足切り 0.3 だと無関係な文が全部候補に入り、
  順位融合（RRF）で全文検索の雑音に負ける。**Hindsight から使うなら `HINDSIGHT_API_SEMANTIC_MIN_SIMILARITY=0.43`**（余裕は薄い。関係ある文の最小 0.44 と無関係の1位の最大 0.43 が接している）。
- 尺度が違う理由は確かめていない。`--pooling` を外しても値は同じだったので pooling の差ではない。ollama 側の GGUF の作りの違いを疑っているが、ollama はもう箱に無いので比べられない。
- メモリは 627→371 MiB（ollama 本体のぶんが無い）。速さは同じ。

### 小型 LLM：日本語5問（ollama のときと同じ bench）

5問＝要約3文／分類（質問・依頼・報告・雑談）／敬体への言い換え／外の事実の主張の有無（はい・いいえ）／JSON 抽出。temperature 0、max_tokens 400、thinking なし。
TTFT と1問は5問の中央値、合計は5問の和、生成 tok/s は llama-server の `timings`（ollama は eval_count/eval_duration）の平均。
待機＝モデルを載せた直後のコンテナの RSS（ollama の行は ollama 本体 約 0.6GiB を含む）。GGUF は ollama のときはそのレジストリの Q4_K_M／Q8_0、llama-server は ggml-org の同じ量子化。

| 経路 | モデル（量子化） | 読込 | 待機MiB | 最大MiB | TTFT | 1問 | 合計 | 生成tok/s | 入力tok/s | 日本語の出来（5問） |
|---|---|---|---|---|---|---|---|---|---|---|
| ollama | qwen3:1.7b (Q4_K_M) | 12.7s | 3019 | 3068 | 3.2s | 5.0s | 45s | 10.4 | 38.2 | 要約○ 分類○ 言換○ 判定×(いいえ) JSON○ |
| **llama-server** | **Qwen3-1.7B Q4_K_M** | （起動時に常駐） | 2586 | 2791 | 3.7s | 5.4s | 57s | 11.3 | 25.9 | 要約○(4文) 分類○ 言換△(「足りんから」が残り「のです」が付く) 判定×(いいえ) JSON○ |
| ollama | qwen3.5:0.8b (Q8_0) | 8.2s | 1621 | 1830 | 1.0s | 2.4s | 24s | 20.0 | 94.7 | 要約○ 分類×(質問) 言換△(だ・である) 判定× JSON○ |
| llama-server | Qwen3.5-0.8B Q8_0 | （起動時に常駐） | 971 | 1202 | 1.0s | 2.0s | 17s | 25.4 | 98.3 | 要約○(4文) 分類×(質問) 言換×(だ調のまま) 判定× JSON△(place が「横浜の開票所」) |

- 速さは ollama のときと同じ（どちらも中身は llama.cpp。差は計測の揺れの範囲。要約の1問だけ llama-server が 37s と遅かったのは、入力 250 トークンの処理が 26 tok/s と ollama より遅かったぶん）。
- メモリは qwen3 1.7B で 3019→2586 MiB（ollama 本体のぶんが無い。mmap で読んだモデルのページも cgroup の数字に入る）。
- 日本語の出来は、同じ Q4_K_M でも ollama のレジストリの GGUF と ggml-org の GGUF で言い換えの1問が違った（量子化の由来が違うので同じ答えにはならない）。
  判定の問題（外の事実の主張を「いいえ」と言う）は両方で同じ。thinking なしの 1.7B ではここが限界。
- thinking（`--reasoning on`）はこの箱では使わない。ollama のときの計測で、1.7B は1問 30〜70 秒、0.8B と 2B は 1024 トークン考え続けて本文が出なかった。llama-server では測り直していない。

### MCP：initialize → tools/list → tools/call

標準ライブラリだけの探り（streamable HTTP に JSON-RPC を3回）で `127.0.0.1:8083/mcp` を叩いた。

```
initialize  0.08s  server=any-chat-completions-mcp 0.1.0  protocol=2025-06-18（mcp-proxy が新しい側を話す。中の橋渡しは 2024-11-05）
tools/list  0.02s  chat-with-local-llm（引数 content）
tools/call  3.85s  「日本語で一言、自己紹介して。」→ qwen3-1.7b の返事（本文だけ、thinking なし）
```

待機メモリは `llama-mcp` 140MiB（bun と mcp-proxy と橋渡しの3プロセス）。`bunx` は初回だけ npm から落とし、名前付きボリューム `llama-cpp_bun_cache` に残す。

#### 橋渡しに `@pyroprompts/any-chat-completions-mcp` を選んだ理由

- 条件は「OpenAI 互換の chat を MCP の道具として出す、既存の公開パッケージ（自作しない）」。npm を `mcp openai chat completions` 系の語で引いて、中身を読んだのは3つ。
  - `@pyroprompts/any-chat-completions-mcp` 0.1.1（MIT、GitHub 164 star、最終更新 2025-05）：接続先を `AI_CHAT_BASE_URL` で指せる。道具1個（`chat-with-<名前>`、引数は文字列1つ）。
    外への通信は `AI_CHAT_BASE_URL` への chat.completions だけ。ファイルやシェルに触る道具は無い。`dotenv` が cwd の `.env` を読むが、コンテナ内に無いので何も起きない。依存は MCP SDK 0.6.0・openai・dotenv の3つ。
  - `@mzxrai/mcp-openai` 0.1.1（MIT）：接続先が api.openai.com 固定で、モデル名も gpt-4o 系の列挙に限る。llama-server には向けられない。
  - `mcp-openai` 0.0.1：中身が `status: 'coming-soon'` の1行だけ。
- 前の ollama のときの候補（`ollama-mcp-server` 等）は全部 ollama 専用の `/api/*` を叩く作りで、llama-server には使えない。stdio を HTTP にする `mcp-proxy`（6.7.20、MIT、293 star、2026-10 更新）だけ使い回した。
- 気になる点：橋渡しの MCP SDK が 0.6.0 と古い（protocol 2024-11-05）。mcp-proxy が外側を 2025-06-18 で話すので Claude Code からは問題なく繋がったが、中の橋渡しが更新されていない（2025-05 で止まっている）ことは覚えておく。
  橋渡しの cli は `node` を spawn するので `bunx --bun` で起こしている（bun が `node` の名で自分を見せる）。oven/bun のイメージには `node` の名で bun を呼ぶ代替（`/usr/local/bun-node-fallback-bin/node`）も入っているので、
  `--bun` 無しでも動くかもしれないが、試していない。

## ollama からの乗り換えで変わったこと

- イメージ：ollama 7.07GB → llama.cpp server 1.19GB（圧縮で約 300MB）。
- モデルの置き場：ollama のボリューム（独自の blob 形式）→ Hugging Face の GGUF をそのまま（`/models/models--<org>--<repo>/snapshots/<hash>/<file>.gguf`）。
- モデルの名前：ollama のタグ（`embeddinggemma:300m-qat-q8_0`）→ `--alias` で付けた名前（`embeddinggemma`）。呼ぶ側（hindsight-docker の `.env`）の `model` を合わせる。
- 常駐：ollama は使わないモデルを下ろす（`OLLAMA_KEEP_ALIVE`）→ llama-server は起動したら載せっぱなし（下ろさない。その代わり読み直しが無い）。
- 複数モデル：ollama は1プロセスで切り替え → llama-server は1モデル1プロセス（2本）。
