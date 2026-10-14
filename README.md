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
| 1 | [LLM serving 全景](chapters/01-serving-overview/guide.md)：TTFT/TPOT/throughput、prefill vs decode、continuous batching；process 架構與一個 request 的完整旅程 | `docs/structures.md`, `server/launch.py`, `core.py` | 1.5h | ✅ |
| 2 ★ | [KV cache 記憶體管理](chapters/02-kv-cache-memory/guide.md)：KV pool、page table、allocator、paged attention | `kvcache/mha_pool.py`, `scheduler/cache.py`, `scheduler/table.py` | 1.5h | 🟡 講義完成，quiz 待做 |
| 3 ★ | Radix prefix cache：match / insert / split / LRU evict / ref_count | `kvcache/radix_cache.py` | 1.5h | ⬜ |
| 4 ★ | Scheduler：prefill 優先、token budget、chunked prefill、overlap scheduling | `scheduler/*.py` | 2h | ⬜ |
| 5 | Engine 與 model forward：attention backend、CUDA graph、sampling、TP（帶過） | `engine/`, `attention/` | 1h | ⬜ |
| 6 | 前端與 IPC（已併入 Ch7）：已併入 Ch7 的 §2 | `message/`, `utils/mp.py`, `tokenizer/`, `server/api_server.py` | — | ↪ Ch7 |
| 7 ★ | [mini-rsglang](chapters/07-mini-rsglang/guide.md)（含 Ch6）：Python 前端與 IPC、架構、wire parity、request 旅程、lifecycle FSM、取消、驗證、FINDINGS 與 Phase 8 | `crates/`, `docs/benchmarks/FINDINGS.md` | 2h | 🟡 [quiz](chapters/07-mini-rsglang/quiz.md) 第 1 輪部分完成（Q7.4–7.8 擱置），pitch ✅ |
| 8 | 選題與動工計畫：已用 grilling 決定 → [Phase 8 計畫](plans/2026-10-10-mini-rsglang-phase8.md)（P99 TTFT root cause）。10/12 已 merge：S1 P99 4868 → 302 ms | — | — | ✅ |

## 時程

10/10 改版：學習和 mini-rsglang Phase 8 同步進行（agent 在另一個 session 執行），細節見 [Phase 8 計畫](plans/2026-10-10-mini-rsglang-phase8.md)。

| 日期 | 內容 |
|---|---|
| 10/9（五）1–2h | 環境設定 → Ch0 → Ch1 ✅ |
| 10/10（六） | 用 grilling 定好 Phase 8 計畫 ✅ → Ch7 講義 ✅ |
| 10/11（日） | Ch2 講義 ✅（原定的 Ch3、Ch4、Phase 9 grilling 延後） |
| 10/12（一） | Phase 8 merge ✅（[PR #19](https://github.com/LiyangTseng/mini-rsglang/pull/19)）→ 履歷 ✅ → Ch7 quiz（部分）✅ |
| 10/14（二） | career fair ✅ → Ch2 quiz → Ch3 → Phase 9 grilling ＋ 挑 sglang Rust issue（並行）→ Ch4（Ch5 只讀摘要） |

## 實驗（RTX 3050 @ `tw-desktop`，Qwen3-0.6B）

| # | 章 | 實驗內容 |
|---|---|---|
| 1 | 3 | 共享長 prefix 的 request：比較 radix cache 和 `--cache naive` 的 TTFT |
| 2 | 4 | 比較 overlap scheduling 開與關的吞吐量 |
| 3 | 7 | 重現 Rust 前端的 P99 TTFT 問題（`gpu_phase7_bench.sh`） |

## 每章的檔案

- `guide.md`：講義，包含概念、`file:line` 閱讀路線、面試題
- `quiz.md`：過關考題、我的答案、回饋，會一直追問到答對為止
