# SGLang Rust frontend: claimable issues (survey 2026-10-14)

Read-only survey of `sgl-project/sglang` via `gh`, aimed at someone who has already built a Rust drop-in frontend for mini-sglang (axum/tokio HTTP+SSE, HF `tokenizers` incremental detok, ZMQ+msgpack IPC, request FSM, cancellation, per-stage Prometheus histograms, a Welch-CI bench harness, and a P99 TTFT fix for per-request `Tokenizer` deep clones).

Nothing was commented on, claimed, or modified on GitHub.

## 1. #23206 status: "[RFC] SGLang non-GPU process rust migration"

- **Owner:** @rainj-me. Opened 2026-04-20, last body update 2026-06-18, last comment 2026-10-11. No labels or assignees. Draft RFC, v0.1.
- **Scope:** replace Python's HTTP server, `TokenizerManager` and `DetokenizerManager` with a Rust front half. It is thread-per-core, with ingress and egress FSMs and six thread groups (api-server, tokenizer-manager, encoder, tokenizer, detokenizer, Python scheduler). Out of scope: kernels, attention, KV cache, quantization. It must be a drop-in (same endpoints, schemas and streaming) behind a flag (`SGLANG_RUST_SERVER=1`), A/B-able against Python.
- **Architecture difference from mini-rsglang:** the Rust server is a **PyO3 extension running inside the scheduler process** (`python/sglang/srt/rust_server/server.py`). It talks to the scheduler through in-process bounded MPSC/SPSC channels and msgpack bytes, with no ZMQ. Tensors move through pinned or shm buffers.
- **Code locations (current main):**
  - `rust/sglang-server/`: the frontend. `api_server/http/{native_api,openai,info,metrics}.rs`, `tokenizer_manager/{tokenizer,detokenizer,to_scheduler,from_scheduler}.rs`, `message/` (wire types mirroring `io_struct.py`) and `utils/fsm.rs`.
  - Sibling crates: `rust/sglang-processor` (chat template rendering, reasoning and tool parsers, "renderer"), `sglang-grpc` (native gRPC, RFC #22558), `sglang-mm` (multimodal preprocessing), `sglang-radix-tree` (Rust TreeCore), `sglang-bench`, `sglang-api-types`/`-codegen`.
  - `sgl-model-gateway/` (the router/gateway, called "smg") and `experimental/sgl-router/` are separate Rust codebases.
  - Tokenizers come from `dynamo_tokenizers::Tokenizer`, which wraps an `Arc`. Per-request clones are cheap, so mini-rsglang's deep-clone P99 bug **does not exist here**. That experience is still relevant to item 16 (perf).
- **Work breakdown (Phase 1 table, plus claims from the comments):**

| # | Item | Status | Claimed by / PR |
|---|---|---|---|
| 1–6 | PoC, design, pickle→msgpack, single-process framework, tok/detok, `/generate`+stream | ✅ Done | #29799, #35239 merged |
| 7 | Encoder (multimodal) | 🟡 WIP | sherlockwu et al. |
| 8, 9 | PD disagg, DP attention | ✅ Done | |
| 10 | Metrics / logging | 🟡 Logging done, metrics WIP | Scheduler metrics re-served on `/metrics` by #43942 (merged 10-14). **Frontend latency histograms still missing** (see pick #1). |
| 11 | Port remaining HTTP APIs | ⏳ Pending | Piecemeal PRs: SageMaker #41462, Ollama #37440, `/weights_checker` #43926, `/model_info` #42622, ... |
| 12 | Unified multi-protocol lib | ⏳ Pending | (refactor, maintainer-owned) |
| 13 | Code refactor | 🟡 WIP | rainj-me |
| 14 | AuthZ | claimed | Kare0638, #38276 |
| 15 | Parallel sampling | claimed | Kare0638, #37202 |
| 16 | bench_serving perf | 🟡 WIP | rainj-me; jthomson04 allocation PRs (#42619, #42617) |
| 17 | Embedding | claimed | qiuhuiming, #36986 |
| 18 | Offline Engine API | unclaimed | |
| 19 | Other `TokenizedGenerateReqInput` flags | unclaimed, being picked off | e.g. `extra_key` #44207 merged 10-14 |
| 20 | `cache_tokens` | unclaimed | |
| 21 | `custom_logit_processor` | claimed | charliechenye, #34879 |
| 22 | `preferred_sampling_params` | claimed | daii-0818, #34224 |
| 23 | Standalone API server for DP attn | 🟡 WIP | |
| 24 | Anthropic API | (ishandhanani: "comes built into the frontend crates") | |
| 25 | SageMaker | claimed | PansaLegrand, #41462 |
| 26 | Vertex AI | **unclaimed again**: mrain's #33211 was auto-closed as a stale draft on 2026-09-26 | |
| 27 | Ollama | claimed | miter6, #37440 |
| 28 | Weight update APIs | claimed **2026-10-11 by @LiyangTseng** (the GitHub account tied to this session; coordinate if that is you or your friend) | overlaps #34531 (`save_remote_model`) |
| 29 | HiCache APIs | unclaimed | |

- **Related trackers:** Q4 roadmap #42473 lists "API server" (PoC @alexnails, @rainj-me) and "Rust migration" (PoC @ishandhanani, @rainj-me) with Slack `#rust-migration`. Renderer RFC #40920 (owner @sagearc, for `sglang-processor`) has its own TBD table, partly claimed: @hickeyma took item 3/7, @Edwardssss took item 4. Native gRPC RFC #22558 keeps getting auto-closed for inactivity, but its code ships.
- **Pace warning.** This area moves very fast. `rust/sglang-server` gets several merges a day. **@metamergebot** (apparently a maintainer-run agent account) lands many small "[Rust] Serve X from the Rust frontend" parity PRs; it had 4 merged on 10-13/14 and 2 more open (#44312, #44315). Many bug issues are filed **together with a fix PR by the reporter** (lagoon0o0, x14ngch3n, alilxxey). Almost every open Rust bug already has a linked PR, so the realistic openings are **unclaimed RFC items and gaps you find yourself**. Comment on #23206 before starting anything.

## 2. Ranked candidates

"Claimed?" covers assignees, "I'll take this" comments, and linked open PRs as of 2026-10-14.

| Rank | Item | Size | Skill match | Claimed? | Needs GPU? | Why |
|---|---|---|---|---|---|---|
| 1 | **Rust-frontend request-latency metrics.** Port `TokenizerMetricsCollector` (TTFT, ITL, e2e, queue-time histograms; prompt/generation token and request counters) to the Rust frontend. #23206 item 10. No issue exists yet. | M | ★★★ exact (per-stage Prometheus histograms, request FSM) | Item 10 is "WIP" with no named owner. Earlier attempt #33039 (haowu1234) was closed when #29799 landed. #34453 (sherlockwu "rust tm ttft breakdown") is a stale draft on the pre-refactor layout and is a debug microbench, not Prometheus. **Ask first.** | No for dev (cargo unit tests; CPU/MLX e2e). GPU nice for overhead validation. | Verified gap: `sglang:time_to_first_token_seconds` and friends live only in Python's `TokenizerManager` (`python/sglang/srt/observability/metrics_collector.py:1757`). The Rust replacement emits no histograms: `rust/sglang-server/src/` has none, and `/metrics` only merges scheduler multiprocess files. `native_api.rs:46` has `TODO: Move request lifecycle timing into a dedicated tracing/metrics module and align its design with Python's APIServerReqTimeStats`. Anyone running `SGLANG_RUST_SERVER=1` with Grafana currently loses their TTFT dashboards. |
| 2 | **HTTP `/abort_request`** (and siblings `/pause_generation`, `/continue_generation`, `/flush_cache`) in the Rust frontend. A #23206 item 11 slice. | S–M | ★★★ cancellation + FSM | No PR found. The gRPC-side Abort is in #42118 (jain-ria). HTTP is untouched. | No | Python serves all of these. Rust serves only `/generate`, `/health*`, `/model_info`, `/server_info`, `/weights_checker`, `/metrics`, `/v1/*`. The plumbing already exists: `ControlRequest::AbortReq` is sent on client disconnect (`tokenizer_manager/to_scheduler.rs:420-444`), and `info.rs` shows the control request/response pattern (`/weights_checker`). The work is mainly route wiring, `abort_all`/rid semantics, and FSM-correct stream finalization with an abort finish_reason. Small and reviewable, so it is a good first PR. |
| 3 | **#31206** sgl-router (PD): open circuit breaker still dispatches to decode | S | ★★ tokio/router, test writing | Abhisheklearn12's regression-test PR #31933 was closed by its author without review (2026-09-08). The reporter asked maintainers to decide (2026-09-13). Now unowned. | No (mock workers) | Re-verify on current `sgl-model-gateway` main, then revive the regression tests or propose closing the issue as fixed. Low risk, but gateway reviewers have been slow and this is not on the #23206 critical path. |
| 4 | **#23206 item 29: HiCache APIs** (`/hicache/storage-backend` attach/detach/query, `/hicache/storage-backend/clear`) | S–M | ★★ HTTP + control msgs | Unclaimed | Unit tests on CPU. Real e2e needs a GPU box with HiCache storage. | Same control-request pattern as pick 2. Less "core" for your profile. |
| 5 | **#23206 item 26: Vertex AI API** (`/vertex_generate`) | S | ★★ axum handler + schema mapping | mrain's #32585/#33211 were both closed as stale drafts. Ask @mrain whether you can take over. | No | Narrow adapter over the existing chat/generate handlers, a pattern like SageMaker #41462. Easy to land, but it shows less of your differentiated skills. |
| 6 | **#23206 items 19/20: remaining `TokenizedGenerateReqInput` flags, `cache_tokens`** | S each | ★★ msgpack wire contract | Unclaimed, but metamergebot is landing these quickly (`extra_key` #44207) | No | Exercises the "one canonical schema" rules in `rust/sglang-server/README.md`. High collision risk. Do this only after rainj-me confirms which flags are open. |
| 7 | **#23206 item 16: bench_serving perf.** Offer P99 TTFT profiling and A/B with CIs instead of claiming the item. | M | ★★★ (your P99 TTFT story, Welch CI harness) | Owned by rainj-me (WIP); jthomson04 has allocation PRs | Yes, for meaningful numbers | Strongest narrative fit, but it is owned. Offer measurement help, e.g. a TTFT stage breakdown under load that compares Rust and Python with CIs. Do not grab it. |

**Not claimable (already have a PR or claimer):** #39935 and #39936 (lagoon0o0, PRs #39932 and #40080 unreviewed since 09-18; reviewing them is a good way in), #44121 (Kylinny, #44136), #43862/#43882 (x14ngch3n, #43863/#43890), #42993 (#42994), #43888 (#43889), #42741 (#42740), #35156 (#35159), #39843 (two PRs), #41748 (#41785), #36678 (#43496), #44278 (two PRs). The detok "first-token spacing" gap (the decode stream is seeded with no prompt tokens, `tokenizer_manager/detokenizer.rs` `new_decoder`) is already being written up by @hickeyma under renderer #40920 item 3.

The "good first issue" and "help wanted" labels contain **no Rust items**. The repo has no `rust`, `router` or `gateway` labels.

## 3. Recommendation

1. **Claim #1, Rust-frontend latency metrics.** This is the best fit: a verified, user-visible parity gap on the RFC critical path (item 10), it uses exactly what you built, and it is CPU-testable. Post on #23206 first, tagging @rainj-me and @sherlockwu (#34453). Say you will port `TokenizerMetricsCollector`'s TTFT, ITL, e2e and queue histograms plus the token and request counters with Python-identical names, labels and buckets, and ask whether item 10 has an owner. Design question to raise: should Rust write prometheus_client-compatible mmap files into `PROMETHEUS_MULTIPROC_DIR`, so `utils/prometheus.rs` merges them unchanged, or keep a Rust-native registry and append its rendered output? Python parity tests are in `test/registered/observability/test_metrics.py`.
2. **As a quick warm-up or fallback, take #2, HTTP `/abort_request`** (plus pause/continue). It is small, it shows cancellation and FSM skills, and the internal `AbortReq` path already exists. Name it explicitly as an item 11 slice in the #23206 comment, so it does not collide with metamergebot's parity sweep.

If rainj-me says item 10 is taken, swap in #31206 or Vertex (#5) as the fallback.

## 4. Contribution process notes

- **Claiming:** there is no formal assignment. For #23206 the convention is a comment like "@rainj-me I'd like to take item N", then a PR that references #23206. For bugs, people comment "I'd like to work on this" and usually open the PR within a day. Expect review latency: several Rust PRs from September are still unreviewed.
- **Guide:** `docs/docs/developer_guide/contribution_guide.mdx`. Run pre-commit (`pre-commit run --all-files`). Registered Python tests use `CustomTestCase` and `register_*_ci(...)`. Security rule: never unpickle network data (msgpack only).
- **CI gating:** CI runs only when a PR has the **`run-ci`** label, which only users in `CI_PERMISSIONS.json` can apply (`/tag-and-rerun-ci`). New contributors must ask a maintainer. PR authors can `/rerun-failed-ci` their own PRs. A `/rerun-group rust-server` test group exists.
- **Rust CI** (`pr-test-rust.yml`, `pr-test-rust-workspace.yml`): runs on `ubuntu-latest` CPU runners. It runs `cargo +nightly fmt -- --check`, `cargo clippy --all-targets --all-features -- -D warnings`, `cargo test --workspace`, plus processor feature tests. Recent PRs also run `cargo clippy --locked -p sglang-server --all-targets -- -D warnings` and `cargo test --locked -p sglang-server`. Some e2e Rust-server tests are GPU (`test/registered/e2e/rust_server/*cuda*`), and some are CPU (`test_mm_transport_1node_cpu.py`).
- **GPU needs:** most `sglang-server` work is CPU-testable through cargo unit tests built on the existing mock patterns (`routes().with_state(..).oneshot(..)`, fake `TextTokenizer`). Contributors have run the Rust server e2e on **Apple Silicon/MLX** (`SGLANG_RUST_SERVER=1`, see #39935 and #39843), so a Mac works for functional checks. Use the GPU box only for perf numbers.
- **macOS build quirk:** linking the PyO3 crate on macOS needs `RUSTFLAGS='-C link-arg=-undefined -C link-arg=dynamic_lookup' cargo test --workspace` (fix PR #39361 still open).
- **Code-review principles** (`rust/sglang-server/README.md`): strongly typed boundaries (no `serde_json::Value`/`rmpv::Value` in production paths), one canonical schema aligned with `io_struct.py`, one representation per stage, explicit validation and versioning. Write PRs to these rules.
- **PR template** sections: Motivation / Modifications / Accuracy Tests / Speed Tests / Checklist. Recent merged Rust PRs list the exact cargo and pytest commands run, and state differences from Python behavior explicitly.
- **Slack:** `#rust-migration` and `#sglang-grpc-rfc-22558` on sgl-fru7574.slack.com (join via slack.sglang.io).
