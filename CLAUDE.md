# CLAUDE.md

## 這個 repo 是什麼

這是個人的**學習紀錄區**（mini-sglang / mini-rsglang），不是產品或函式庫。主要用途是教學講義、quiz 紀錄、實驗筆記與練習，所以：

- 以教學、記錄、練習為優先；不需要 CI、測試、release、複雜分支或嚴謹的工程流程。
- 工程類 skill（issue、triage、ADR 等）只在確實有幫助時輕量使用，不必為了流程而流程。
- 真正的工程實作在 `mini-rsglang` 裡進行，這裡只放學習產出與計畫。

## Agent skills

### Issue tracker

Issues live in GitHub Issues for `mileschintw/mini-sglang-learning` (via `gh` CLI). See `docs/agents/issue-tracker.md`.

### Triage labels

Default five-role vocabulary (`needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`). See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: root `CONTEXT.md` + `docs/adr/`. See `docs/agents/domain.md`.
