# Ch7 Quiz 紀錄

完整 quiz（Q7.1–Q7.11），會一直追問到答對為止。Q7.11 的 pitch 在 Phase 8 完成後要再更新一次。

## 第 1 輪（10/14，career fair 當天的精簡版）

時間不夠，只答了一部分。Q7.4–Q7.8 先跳過，都是很細的實作題，留到技術面試前再補。

| 題 | 結果 | 診斷 |
|---|---|---|
| Q7.1 | 🟡 | 記憶體的推論正確：多出一整組 tokenizer process，每個都載入自己的 library 和 tokenizer。process 數錯一個：`--num-tokenizer=2` 時是 **4** 個（api server、2 個 tokenizer、1 個 detokenizer，也就是 N+2），不是 3 個。Rust 前端的專用 OS thread 漏了 **stdin reader**（讀 handshake、偵測 launcher 是否還活著）。 |
| Q7.2 | ❌ | 沒寫 hex。前提錯了：wire format 是 **named map**（`to_vec_named`），不是照順序讀的 array。**多加 `priority`**：upstream `deserialize_type` 會執行 `cls(**kwargs)`，遇到不認識的 key 就拋 `TypeError`，scheduler 解不出這則訊息（`vendor/.../minisgl/message/utils.py:61-67`）。**改成 f32**：msgpack 會編成 `ca`（float32），Python 照樣能讀成 float，**不會 crash**，但 0.9 會變成 0.8999999762，sampling 有細微差異，byte-identical parity 就破了。這種 bug 不會報錯，所以更陰險。 |
| Q7.3 | 🟡 | 「單一 FIFO、單一 writer」的方向正確，專用 socket 會讓 abort 插隊也正確。漏了兩點：**票根**——`abort` 必須出示 `Submitted`，而只有 submit 成功 enqueue 之後才拿得到，所以「先送 abort」在編譯期就不可能發生（`writer.rs:42-45` 有 `compile_fail` 測試）。**後果**——scheduler 把先到的 abort 當成未知 uid 忽略，接著 request 照常進來，一路跑到 `max_tokens`，白白浪費 GPU。原因也不是「網路狀況」：兩條 socket 之間本來就沒有順序保證，跟網路好壞無關。 |
| Q7.4 | ⏭️ | 未答 |
| Q7.5 | ⏭️ | 未答 |
| Q7.6 | ⏭️ | 未答 |
| Q7.7 | ⏭️ | 未答 |
| Q7.8 | ⏭️ | 未答 |
| Q7.9 | 🟡 | 「mock 上能重現 → 問題在前端，不在 GPU」正確，這正是 Phase 8 實際的推論方式。沒有分別回答 H1/H2/H3 各自的 histogram 長什麼樣子，也沒回答「重現不了」時的下一步。 |
| Q7.10 | ✅ | 中文一個字在 UTF-8 是 3 bytes，用 byte 數會切到字的中間，client 收到無法解碼的亂碼。補充：要用字元數，是因為 upstream Python 的 `len(str)` 算的是字元，兩邊單位必須一致。 |
| Q7.11 | 🟡 | 見下方。「遠勝於 Python 版」不成立（P99 是打平，大勝的只有記憶體和高負載表現）。「修了一個 bug」聽起來太小，要講成「找到 tail latency 的 root cause，並推翻原本的假設」。 |

→ 之後補：Q7.2 的 hex、Q7.4–Q7.8、Q7.9 的 H1/H2/H3 分項

### 你的原始答案

- **Q7.1**：2（api server + tokenizer）、3（api server + tokenizer + tokenizer）、1（單一 process 控制所有模組）。除了 tokio 之外還有 tx-zmq 和 rx-zmq。RAM 用量多很多，是因為多了一整組 tokenizer process，加上它們各自載入的 library，造成很大的差異。
- **Q7.2**：我覺得細節格式沒那麼重要。改成 f32 一樣會被當成 f64 讀取，導致出錯；多加欄位也會導致讀取錯誤，因為接收端是照順序讀取的，欄位順序是固定的。
- **Q7.3**：單一 FIFO 和單一 TX 都代表只有一個隊列，所以 abort（只可能在 request 之後才產生）自然不會比 request 更早出現。Submitted 票根不知道。如果有專用 socket，就可能因為網路狀況等因素，讓 abort 比 request 更早送到 scheduler。
- **Q7.9**：mock 上也能重現，代表問題不在 GPU，而是在前端。
- **Q7.10**：在 Unicode 裡一個中文字由三個 byte 組成，如果只看 byte 數，client 會收到無法解析成文字的亂碼。
- **Q7.11**：我負責設計與規劃題目和內容，同時修正了一個嚴重的 bug：它會讓大量 deep copy 同時發生，嚴重影響效能與記憶體占用。修正後，Rust 版成功展現遠勝於原本 Python 版的效果。

## Q7.11 pitch（Phase 8 完成後的版本）

**English**

> mini-rsglang is a drop-in Rust frontend for an LLM serving engine. It speaks the Python scheduler's wire protocol byte-for-byte, so the same GPU backend serves both frontends. Against tuned Python, it uses 24× less memory, starts 11% faster, and matches tail latency under 128 concurrent agents. I co-designed the project and led the performance work. Profiling each stage disproved our original hypothesis that IPC was the bottleneck, and traced the 4.9-second P99 to a per-request tokenizer deep copy. Fixing it cut P99 16×, to 302 ms.

**中文**

> 我和朋友用 Rust 改寫了一個 LLM serving engine 的前端。它跟原本的 Python scheduler 用 byte 級相容的協定溝通，GPU backend 完全不用改。跟調校過的 Python 比，記憶體少 24 倍，啟動快 11%，128 個並發 agent 下的 P99 打平。我負責共同設計這個專案，並主導效能分析：用每個階段的 latency histogram 推翻了原本「瓶頸在 IPC」的假設，找到 P99 高達 4.9 秒的原因是每個 request 都 deep copy 一份 tokenizer。修掉之後 P99 降了 16 倍，到 302 ms。

**被追問時的數字出處**（FINDINGS v2）：24× = S3 RSS 3.03 GB / 123 MB；11% = S3 冷啟動 8.22 s vs 9.28 s；P99 打平 = S1 302 vs 314 ms，差異 −4.0%，CI [−15.7%, +7.7%]；16× = S1 P99 4868 → 302 ms。
