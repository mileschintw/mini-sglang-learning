# mini-rsglang 下一步：Phase 8 計畫與執行步驟

2026-10-10（六）grilling 的結論。目標：週二（10/14）career fair 的履歷能放上 mini-rsglang，並附上自己主導、有數字的成果。下週去認領 sglang 的 Rust 相關 issue。

- mini-rsglang 英文計畫（給 agent 看）：[`docs/plans/phase-8-p99-root-cause.md`](https://github.com/LiyangTseng/mini-rsglang/blob/feat/phase-8-p99-root-cause/docs/plans/phase-8-p99-root-cause.md)
- Draft PR：[LiyangTseng/mini-rsglang#19](https://github.com/LiyangTseng/mini-rsglang/pull/19)
- Issues：[#14](https://github.com/LiyangTseng/mini-rsglang/issues/14) → [#15](https://github.com/LiyangTseng/mini-rsglang/issues/15) → [#16](https://github.com/LiyangTseng/mini-rsglang/issues/16) → [#17](https://github.com/LiyangTseng/mini-rsglang/issues/17) → [#18](https://github.com/LiyangTseng/mini-rsglang/issues/18)

## 為什麼是 P99 TTFT

Phase 7 的結論是：Rust 在 S1（取消場景）的 P99 TTFT 是 4687 ms，python-best 只有 431 ms。FINDINGS 把原因歸給 ZMQ，但那只是推論。讀過程式碼後發現：

- Rust 和 Python 的 ZMQ 送出路徑幾乎一樣（都是 blocking send、預設 HWM），很難單靠它解釋十倍的差距
- Rust 有 Python 沒有的 per-request CPU 工作：
  - **每個 request 都 deep-clone 一份完整 tokenizer**（`hf_codec.rs:167-175`）
  - `encode` 直接在 tokio worker 上跑（`engine.rs:360`）

如果量測證實了這一點，故事就是「推翻原本的假說 → 找到真正的 root cause → 修好 → P99 從 4.7s 降到 X」。這剛好是面試想聽的 debugging 能力，也跟 sglang 的 Rust 前端遷移（sglang#23206）是同一個領域。

## 決策紀錄

| # | 主題 | 決定 |
|---|---|---|
| Q1 | 要認領的 sglang issue | Rust 移植相關。以目前的硬體和背景，這類最適合；mini-rsglang 可以當作前導作品 |
| Q2 | 履歷寫法 | 寫成整個專案的 co-author，加上自己主導的 bullet。**前提是 Ch7 學透**，面試被問到實作細節要答得出來 |
| Q3 | 週一前的成果標準 | 以有數字的 before/after 為目標；可運作的 prototype ＋設計文件當保底 |
| Q4 | GPU box | 是我自己的電腦，朋友不再使用，時間不受限 |
| Q5／Q8 | 學習順序 | 重排：今天先學 Ch7（併入 Ch6），明天學 Ch2 → Ch3 → Ch4，Ch5 只讀摘要 |
| Q6 | 方向 | 兩段式：Phase 8（P99 root cause ＋ fix）→ Phase 9（Rust radix cache via PyO3，以 flag 切換） |
| Q7 | 停損點 | 週日晚上設 checkpoint，用來決定「履歷上能寫什麼」，**不是停止點**。有新的高機率線索就繼續追。達標標準：S1 P99 ≤ 431 ms |
| Q9 | Quiz | 只有 Ch7 做完整 quiz。其他章節產生講義時，題目也一次產生完整，用 `⏸ 擱置` 標注，之後再補答 |
| Q10／Q19 | Agent 執行方式 | 在 desktop app 為 mini-rsglang 另開一個 Code session，貼上下方的 kickoff prompt。agent 可以自行 ssh 到 GPU box 跑 benchmark |
| Q11／Q15' | GSD | 退場。`.claude/CLAUDE.md` 重寫成手動維護的版本；`.planning/` 原地保留成唯讀歷史（因為 test 會讀 `STATE.md`）；刪除 `.worktreeinclude` 和 `milestone.lock`；`.gitignore` 的 GSD 條目保留 |
| Q12 | Phase 9 是否並行 | 不並行，Phase 8 完成後才開始（GPU 只有一張，WSL 的效能也會互相影響） |
| Q13／Q16 | Repo 與 profile | 直接放在 `LiyangTseng/mini-rsglang`。commit 用 `user39261580@gmail.com`；在 profile pin 這個 repo；README 加 Authors 行（要先問朋友） |
| Q14 | FINDINGS 的改法 | 保留原分析並標成 v1 hypothesis，另加 v2 章節 |
| Q17' | 驅動方式 | 由 `diagnosing-bugs` skill 驅動：**先建立 tight loop** 再提假說。最後跑一次 `/code-review` |
| Q18 | Phase 9 的設計 | 學完 Ch3 後另外做一輪 grilling |
| Q21 | Branch | 只開一條 `feat/phase-8-p99-root-cause`。commit 1 = de-GSD，commit 2 = skills 設定，commit 3 = 計畫文件，之後每個 issue 至少一個 commit。最後用 **merge commit**（不用 squash）合回 main |
| Q22 | 文件語言 | 學習 repo 用中文，mini-rsglang 用英文 |

## 已完成（10/10，由 Claude 執行）

- [x] mini-rsglang 建立 branch `feat/phase-8-p99-root-cause`
  - `8cb5fdc` chore: retire GSD tooling, keep .planning as read-only archive
  - `b55a12b` chore: configure mattpocock agent skills（`docs/agents/`、`CONTEXT.md`、triage labels）
  - `9d0d2ad` docs: add Phase 8 plan (P99 TTFT root cause)
- [x] 開好 issues #14–#18（`ready-for-agent`，依賴關係寫在每個 issue 的開頭）
- [x] 開了 draft PR #19
- [x] 確認 GPU box：WSL 上的 clone 在 `~/projects/mini-rsglang`，目前停在舊的 `phase-07-frontend-benchmarks` branch，沒有裝 GSD

注意：朋友的 root `claude.md` 規定 commit message 不能出現 Co-authored-by 或提到 AI，所以 mini-rsglang 的 commit 都沒有加這些。

## 執行步驟（給我自己的 checklist）

### 今天（10/10）

- [ ] 跟朋友說一聲，需要對齊的事：
  - Phase 8／9 的方向，以及 branch 和 PR #19
  - GSD 已經退場（他的 Claude Code 讀到新的 `.claude/CLAUDE.md` 時，行為也會跟著改變）
  - 最後會用 merge commit 合回 main
  - 能不能在 README 開頭加上 `Authors: @LiyangTseng, @mileschintw`
- [ ] 在 GitHub profile pin `LiyangTseng/mini-rsglang`（開過 PR 就算 contribution，現在就能 pin）
- [ ] desktop app → 新增 Code session → 資料夾選 `~/Desktop/repos/mini-rsglang`，切到 `feat/phase-8-p99-root-cause` → effort 調到 high → 貼上下方的 kickoff prompt
- [ ] 回到這個 session，把 effort 調高，開始學 Ch7（含完整 quiz）

### 明天（10/11）

- [ ] 學 Ch2 → Ch3 → Ch4（精簡 quiz，題目產生完整，但標注 ⏸ 擱置），Ch5 只讀摘要
- [ ] 中途看看 agent 在 issues 上留的進度，有問題就回覆
- [ ] **晚上 checkpoint**：讀 #14–#16 的結論，決定履歷 bullet 的寫法。還有線索就讓 agent 繼續追
- [ ] 學完 Ch3 後做 Phase 9 的 grilling

### 週一（10/12）

- [ ] 確認 #18 的 GPU benchmark 和 FINDINGS v2 都已完成
- [ ] 在 agent session 跑 `/code-review`，處理發現的問題
- [ ] 自己 review PR #19。看得懂、講得出來之後，標成 ready，用 **Create a merge commit** 合併
- [ ] 改履歷，準備 30 秒的 pitch

### 週二之後

- [ ] 補完 Ch2–4 擱置的 quiz
- [ ] 開始 Phase 9
- [ ] 去 sglang 找 Rust 相關的 issue 認領

## Kickoff prompt

貼到新的 mini-rsglang session：

```text
你在 mini-rsglang 的 feat/phase-8-p99-root-cause branch 上工作，負責執行 Phase 8。

先讀：
- docs/plans/phase-8-p99-root-cause.md（整個 phase 的計畫）
- CONTEXT.md、.claude/CLAUDE.md、根目錄的 claude.md（行為準則與 git 規則）
- docs/benchmarks/FINDINGS.md（Phase 7 的結論與 v1 假說）
- issues #14 → #15 → #16 → #17 → #18（gh issue view N --comments），依序執行

工作方式：
- 開始 #14 之前，先用 Skill tool 呼叫 `mattpocock-skills:diagnosing-bugs`，用它驅動 #14–#17。#14 的 tight loop 還沒建好之前，不要開始提假說或修 code。
- #18 完成後，用 Skill tool 呼叫 `code-review`（effort 用 high），針對整條 branch 對 main 的 diff 做檢查，修掉確認的問題後 push，再到 PR #19 留言總結。
- 每完成一個 issue：commit（message 引用 #N）→ push → 在該 issue 留言，附上指令、輸出和結論 → close issue → 在 PR #19 勾掉對應項目。
- 所有寫進 repo 的內容（code、commit、issue／PR 留言、文件）都用英文。
- Git：commit 作者用 mileschintw <user39261580@gmail.com>（先確認 git config user.email）。commit message 不要加 Co-authored-by，也不要提到 AI（claude.md 的規定）。不要 push 到 main，不要 force push，也不要把 PR 標成 ready。這些留給我。
- vendor/ 絕對不能改（frozen backend）。wire contract 和 parity 要維持。

GPU box（已授權你自行使用，不用每次問我）：
- ssh tw-desktop（WSL，RTX 3050 8GB）。repo 在 ~/projects/mini-rsglang，目前停在舊的 branch，請先 git fetch 再 checkout feat/phase-8-p99-root-cause。
- 每次跑之前先看 nvidia-smi。使用現有環境，不要安裝或修改套件。port 用 31000 以上。模型只用 Qwen3-0.6B。跑完要 kill 掉所有你啟動的 process。
- 不要在 GPU box 上啟動 claude，只透過 ssh 執行 script。

Checkpoint：10/11 晚上，在 PR #19 留言總結目前已證實的事（root cause／部分改善／被推翻的假說），附上數字。這不是停止點，有高機率的新線索就繼續做。

遇到需要我決定的事（例如修法有取捨、需要動 vendor/、目標明顯達不到），在對應的 issue 留言並停下來問我。
```

## Phase 9 待辦（學完 Ch3 後 grilling）

- 建置工具：maturin 還是 setuptools-rust（目前用 setuptools，沒有 PyO3）
- 要對齊 `kvcache/radix_cache.py` 的哪個 interface；用 `--cache rust-radix` 這類 flag 切換
- Benchmark：實驗 1，比較共享長 prefix 時的 TTFT
- 參考 sglang#20415 的 `UnifiedTreeCoreInterface`
