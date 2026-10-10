# mini-sglang-learning

兩天內讀懂 [mini-sglang](https://github.com/sgl-project/mini-sglang)（LLM serving 引擎的極簡版），再讀懂 [mini-rsglang](https://github.com/LiyangTseng/mini-rsglang)（它的 Rust 前端移植），週日開始動手做自己的部分。

- **對照版本**：mini-sglang `9a91cfa`，也就是 mini-rsglang `vendor/` 裡凍結的同一個 commit。講義中所有的 `file:line` 都以這個版本為準。
- **目標**：進入 LLM inference infra 領域，學到能寫進履歷、能在面試中講清楚的程度。
- **術語表**：[CONTEXT.md](CONTEXT.md)，會邊學邊更新。

## 路線圖

★ 表示核心章節，進度落後時不能砍；Ch5、Ch6 可以壓縮成只讀摘要。

| 章 | 主題 | 主要原始碼 | 預估 | 狀態 |
|---|---|---|---|---|
| 0 | [Transformer 推論速查](chapters/00-transformer-inference/guide.md)：先考試，答不好再補 | `models/`, `layers/attention.py` | 0.5h | ✅ |
| 1 | LLM serving 全景：TTFT/TPOT/throughput、prefill vs decode、continuous batching；process 架構與一個 request 的完整旅程 | `docs/structures.md`, `server/launch.py`, `core.py` | 1.5h | ⬜ |
| 2 ★ | KV cache 記憶體管理：KV pool、page table、allocator、paged attention | `kvcache/mha_pool.py`, `scheduler/cache.py`, `scheduler/table.py` | 1.5h | ⬜ |
| 3 ★ | Radix prefix cache：match / insert / split / LRU evict / ref_count | `kvcache/radix_cache.py` | 1.5h | ⬜ |
| 4 ★ | Scheduler：prefill 優先、token budget、chunked prefill、overlap scheduling | `scheduler/*.py` | 2h | ⬜ |
| 5 | Engine 與 model forward：attention backend、CUDA graph、sampling、TP（帶過） | `engine/`, `attention/` | 1h | ⬜ |
| 6 | 前端與 IPC：ZMQ、msgpack、tokenizer / detokenizer、API server | `message/`, `utils/mp.py`, `tokenizer/`, `server/api_server.py` | 1h | ⬜ |
| 7 ★ | mini-rsglang：架構、各 crate、wire parity、lifecycle FSM、writer thread、FINDINGS | `crates/`, `docs/benchmarks/FINDINGS.md` | 2h | ⬜ |
| 8 | 選題與動工計畫：比較 Rust radix cache、P99 TTFT 修正、L1 Rust 化 | — | 1h | ⬜ |

## 時程

| 日期 | 內容 |
|---|---|
| 10/9（五）1–2h | 環境設定 → Ch0 → Ch1 |
| 10/10（六）整天 | Ch2 → Ch8，含 3 個實驗 |
| 10/11（日） | 依照 Ch8 的計畫開始動工 |

## 實驗（RTX 3050 @ `tw-desktop`，Qwen3-0.6B）

| # | 章 | 實驗內容 |
|---|---|---|
| 1 | 3 | 共享長 prefix 的 request：比較 radix cache 和 `--cache naive` 的 TTFT |
| 2 | 4 | 比較 overlap scheduling 開與關的吞吐量 |
| 3 | 7 | 重現 Rust 前端的 P99 TTFT 問題（`gpu_phase7_bench.sh`） |

## 每章的檔案

- `guide.md`：講義，包含概念、`file:line` 閱讀路線、面試題
- `quiz.md`：過關考題、我的答案、回饋，會一直追問到答對為止
