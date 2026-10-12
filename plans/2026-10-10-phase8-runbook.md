# Phase 8 Runbook：預期 roadmap 與各階段 prompt

搭配 [Phase 8 計畫](2026-10-10-mini-rsglang-phase8.md) 使用。prompt 全部用英文寫，原因有兩個：

- agent 會模仿 prompt 的語言。prompt 用英文，它寫進 issue、PR、commit 的內容就自然是英文
- 真正公開的只有 GitHub 上的東西（commit、issue、PR、留言）。Code session 的對話紀錄不公開，但用英文就不必擔心中文混進公開紀錄

## 預期 roadmap

```
[已完成] Stage 0  setup：de-GSD、skills 設定、計畫文件、issues、draft PR
   │
   ▼  P0 環境準備（GPU box 上的新 session）
   │
   ▼  P1 Kickoff
Stage 1  #14 tight loop → #15 instrumentation → #16 root cause     （agent 自主執行）
   │
   ▼  ★ Gate A：agent 在 #17 留言列出修法選項，然後停下來
   │  P2 Decision（由你決定）
Stage 2  #17 fix ＋ regression test                                  （agent 自主執行）
   │
Stage 3  #18 GPU before/after ＋ FINDINGS v2 ＋ README               （agent 自主執行）
   │
Stage 4  code-review → 修正 → 在 PR 留言總結                          （agent 自主執行）
   │
   ▼  ★ Gate B：是否達標（S1 Rust P99 ≤ 431 ms）？
   │    ├─ 否，且還有線索 → P4 Follow-up round → 回到 Gate A
   │    └─ 是／決定收尾 ↓
   ▼  ★ Gate C：你自己 review PR #19，確定看得懂、講得出來
   │  P5 Finalize
Stage 5  按 Ready for review → Create a merge commit                 （你）
   │
Stage 6  回到學習 session：寫履歷 bullet、準備 pitch                  （這個 session，中文）
   │
Stage 7  Phase 9 grilling（學完 Ch3 後）→ 產生 Phase 9 的 kickoff    （這個 session）
```

| Stage | 觸發 | 執行者 | 預估時間（粗估） | 產出 | 需要 GPU |
|---|---|---|---|---|---|
| 1 | P1 | agent | 3–6h | #14–#16 的留言與 commit | 可能 |
| Gate A | agent 停下 | **你** | 10 分鐘 | 在 #17 記錄修法決定 | — |
| 2 | P2 | agent | 1–3h | fix ＋ regression test | ✗ |
| 3 | 接續 | agent | 1–2h | 新的 benchmark 數字、FINDINGS v2、README | ✓ |
| 4 | 接續 | agent | 30–60 分鐘 | review 修正、PR 總結留言 | ✗ |
| Checkpoint | 10/11 晚上 | agent（或用 P3 觸發） | — | PR #19 上的 checkpoint 留言 | — |
| Gate B／C | agent 停下 | **你** | 30–60 分鐘 | 決定是否收尾，並 review PR | — |

預估時間只是粗估。Stage 1 的不確定性最大，取決於 mock stack 能不能重現問題。

---

## P0：環境準備（GPU box 上的新 session，在 P1 之前跑一次）

10/11 改版：agent 改成直接跑在 GPU box（Windows ＋ WSL2）上，Mac 只用 Remote Control 連進去操作。

開 session 之前，你要先做好：
- 在 Windows 裝好 Claude Code 並登入，裝好 `mattpocock-skills` plugin
- session 的資料夾選 WSL 裡的 clone：`\\wsl.localhost\<distro>\home\user39261580\projects\mini-rsglang`
- 確認 Windows 不會睡眠，也不會因為 Windows Update 自動重開機

P0 只做準備，不碰 #14。跑完後，它會在 repo 根目錄留下一份不進 git 的 `CLAUDE.local.md`，記錄這台機器的操作方式。之後的 session 都會自動讀到它。

```text
This is a one-off setup pass for Phase 8 of mini-rsglang. Do NOT start any issue (#14–#18) and do not commit or push anything. Use English for everything. The goal is that, when you finish, I can paste the Phase 8 kickoff prompt into this session and work starts immediately.

Context: this machine is the GPU box, a Windows host with WSL2 and an RTX 3050 8GB. The repository lives inside WSL at ~/projects/mini-rsglang (user user39261580). The Rust and Python toolchains, the model cache and the GPU workloads all live in WSL. I will control this session remotely from my Mac.

1. Where you run. Determine whether you are running on native Windows or inside WSL. If on Windows, confirm that `wsl.exe` works, find the distro name, and work out one reliable pattern for running commands inside WSL in the repo, e.g. `wsl.exe -d <distro> --cd ~/projects/mini-rsglang -- bash -lc "<cmd>"`. Confirm your file tools can read and edit the repo through the \\wsl.localhost path. Every build, test, git, gh and benchmark command must run inside WSL, never with Windows-side git or toolchains, to avoid CRLF and file-permission problems.
2. Repository. Inside WSL: check `git status`, and stop and tell me if there are local changes. Set repo-local `user.name` to `mileschintw` and `user.email` to `user39261580@gmail.com` (the current value has a typo: `usser…`). Fetch, check out `feat/phase-8-p99-root-cause`, and fast-forward to origin. Confirm HEAD contains commit ed24db1.
3. GitHub access. Inside WSL, `gh` must be installed and authenticated with push access to LiyangTseng/mini-rsglang (`gh repo view LiyangTseng/mini-rsglang --json viewerPermission`), and git must push through it (`gh auth setup-git`). Verify with `git push --dry-run`. If gh is missing or not logged in, give me the exact commands to run myself and wait; never handle tokens or passwords yourself.
4. Skills. Confirm that the `mattpocock-skills:diagnosing-bugs` and `code-review` skills are available in this session. If not, tell me exactly what to install and wait.
5. Toolchain. Do not install or upgrade packages; use the existing environment. Report the cargo and rustc versions and the Python environment that the Phase 7 scripts use (read scripts/gpu_phase7_bench.sh and scripts/check_all.sh to see how they activate it). Build the workspace the way those scripts do, then run the fastest existing check that covers the Rust crates and the parity tests. Use `scripts/check_all.sh --offline` if it fits in about 15 minutes.
6. GPU smoke test. Check `nvidia-smi`. Confirm that Qwen3-0.6B is already in the local model cache (do not download anything). Confirm that ports 31000+ are free. Then run the shortest existing end-to-end smoke that drives the Rust server: the mock stack (`rsg-mock-stack`) first, then one short real-GPU launch on port 31000+ with a single request. Do NOT run the full Phase 7 benchmark. Kill every process you started and confirm with `nvidia-smi` and `ss -ltnp`.
7. Always-on check. Read-only: report the Windows sleep and hibernate timeouts (`powercfg /query` or `powercfg /a`), whether Windows Update could reboot soon, and whether WSL shuts down when idle (check `%UserProfile%\.wslconfig`, e.g. `vmIdleTimeout`). Do not change any system setting; tell me what to change.
8. Setup notes. Write `CLAUDE.local.md` at the repo root and add it to `.git/info/exclude`, never to .gitignore and never committed. It must contain: how to run commands inside WSL (the exact pattern from step 1), the repo paths on both sides, how to activate the Python environment, the build and test commands, how to start and stop the mock stack and the real server, and the benchmark rules: check `nvidia-smi` first; ports 31000+; Qwen3-0.6B only; no builds or tests while a benchmark is running; kill everything you started afterwards. Keep it short and factual.
9. Report. End with a checklist table (item / status / evidence) covering steps 1–8, and list exactly what I still need to do before the kickoff. Then stop.
```

## P1：Kickoff（P0 完成後，同一個 session 的下一則訊息）

```text
You are working on mini-rsglang, branch `feat/phase-8-p99-root-cause`, to execute Phase 8: find and fix the root cause of the Rust frontend's P99 TTFT tail spike.

Read first:
- docs/plans/phase-8-p99-root-cause.md (the phase plan, including the fix-decision gate)
- CONTEXT.md, .claude/CLAUDE.md, and the root claude.md (behaviour and git rules)
- docs/benchmarks/FINDINGS.md (Phase 7 results and the v1 root-cause hypothesis)
- Issues #14 → #18 (`gh issue view N --comments`), executed in order

How to work:
- Before starting #14, invoke the `mattpocock-skills:diagnosing-bugs` skill with the Skill tool and let it drive #14–#16. Do not form hypotheses or change code until the #14 feedback loop exists and has gone red.
- For each issue: commit (message references #N) → push → comment on the issue with the commands, outputs and conclusion → close the issue → tick the item in PR #19's description.
- STOP after #16. Do not start #17. Post the fix options on #17 (what changes, expected effect citing #16 data, risks such as the decoder-before-submit ordering and parity, effort, and your recommendation), then tell me in this chat and wait for my decision.
- Use English for everything: code, commits, issue and PR comments, docs, and your replies to me here.
- Git: author as mileschintw <user39261580@gmail.com> (check `git config user.email` first). No Co-authored-by trailers and no mention of AI in commit messages (root claude.md rule). Never push to main, never force-push, never mark PR #19 ready for review; those are mine.
- Never modify vendor/ (frozen backend). Keep the wire contract and output parity intact.

Machine (you run on the GPU box itself; you may use the GPU without asking each time):
- Follow CLAUDE.local.md from the setup pass: run every build, test, git, gh and benchmark command inside WSL using the pattern recorded there.
- Check `nvidia-smi` before every run. Use the existing environment; do not install or modify packages. Use ports 31000+. Qwen3-0.6B only. Kill every process you started when done.
- Never build or run tests while a benchmark is running: they share the CPU and distort tail latency.

Checkpoint: on the evening of 2026-10-11, post a comment on PR #19 summarising what is proven so far (root cause / partial improvement / rejected hypotheses) with numbers. This is not a stop; keep going if there is a strong new lead.

If you hit a decision that is mine (a fix with real trade-offs, anything that would touch vendor/, or the target looking unreachable), comment on the relevant issue, tell me here, and wait.
```

## P2：Gate A 決策（agent 在 #17 列出選項後）

先在 #17 讀過選項。看不懂的地方，可以拿回這個學習 session 問我，或直接請 agent 解釋。決定後，把下面這段填好貼上：

```text
Decision for #17: go with option <X>.
Reasoning: <one or two sentences, e.g. "the #16 data shows decoder-build queueing dominates the tail, so removing the clone fixes the cause rather than hiding it; the encode pool can wait until data shows encode matters">.
<Optional constraints, e.g. "keep the change minimal; do not add a new dependency without asking me".>

Record this decision and reasoning as a comment on #17 (in English, written as the maintainer's decision), then continue: #17 → #18 → run the `code-review` skill (effort high) over the branch diff against main, fix confirmed findings, push, and post a final summary comment on PR #19 (before/after numbers, root cause, what changed, open follow-ups). Then stop and tell me whether the success target (S1 Rust P99 TTFT ≤ 431 ms) was met.
```

如果你想在決定之前先聽 agent 解釋某個選項：

```text
Before I decide: explain option <X> vs <Y> in more depth. What exactly changes in the request path, what would the per-stage histograms look like after each, and what could break (ordering, parity, memory)? Don't post this on GitHub; just answer here.
```

## P3：Checkpoint／狀態查詢（隨時都可以用）

```text
Status check. Summarise in this chat: which issues are done, what the data says so far (per-stage P50/P99, which hypotheses are confirmed or rejected), what you are doing now, the next step, and anything blocked on me. If it is the evening of 2026-10-11 and you have not posted the checkpoint comment on PR #19 yet, post it now.
```

## P4：Follow-up round（Gate B 未達標，但還有線索）

```text
The target (S1 Rust P99 TTFT ≤ 431 ms) is not met yet. Start a follow-up diagnosis round:
1. Open a new issue "Phase 8 follow-up: <remaining gap>" with the current numbers, what is already ruled out, and the leading hypothesis. Label it ready-for-agent and link it from PR #19.
2. Invoke `mattpocock-skills:diagnosing-bugs` again, reusing the existing feedback loop (tighten it if the remaining gap needs a sharper signal).
3. Same decision gate as before: once the cause is identified, post fix options on the new issue and wait for my decision.
Keep FINDINGS v2 honest: whatever is measured now goes in, even if the target is never met.
```

如果決定不追了，直接收尾：

```text
We stop here for Phase 8. Make sure FINDINGS v2 and the README state the measured result honestly, list the remaining gap and the leading hypothesis under future work, and post a closing summary on PR #19. Then stop.
```

## P5：Finalize（Gate C，你 review 完 PR 之後）

```text
I have reviewed PR #19. Finalize it:
1. Rewrite the PR description as a final summary: problem, method (feedback loop → instrumentation → hypotheses), root cause, fix, before/after numbers table, and links to FINDINGS v2 and each issue. Keep the "merge with a merge commit, not squash" note.
2. <Only if my co-author agreed:> add an authors line under the README title: "Authors: @LiyangTseng, @mileschintw". Commit and push.
3. Confirm CI is green on the latest commit and all of #14–#18 are closed.
Do not mark the PR ready and do not merge; I will do both.
```

## 輔助 prompt

**換新 session 接手**（context 用完或 session 中斷時）：

```text
Resume Phase 8 of mini-rsglang on branch `feat/phase-8-p99-root-cause`. Rebuild context from GitHub rather than memory: read docs/plans/phase-8-p99-root-cause.md, PR #19 (description and comments), and every open and closed issue from #14 onward with comments. Then tell me where things stand and what you will do next, and wait for my go-ahead. All earlier rules still apply: English only, no AI mentions or Co-authored-by in commits, never touch vendor/ or main, GPU box rules, and the decision gate before any fix.
```

**暫時讓出 GPU**（你自己要用 GPU box 時）：

```text
Pause all GPU box usage now: stop any running benchmark, kill every process you started, confirm with nvidia-smi, and continue only with work that does not touch the GPU or load the CPU heavily (reading code, writing docs, posting comments) until I say the GPU is free again.
```
