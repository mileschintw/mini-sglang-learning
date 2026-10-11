# Ch1 Quiz 紀錄

## 第 1 輪（10/10）

| 題 | 結果 | 診斷 |
|---|---|---|
| Q1.1 | ❌ | 1110 是把三個 request **一個接一個跑**的步數。static batching 是三個 request 一起進 batch，但要等最長的那個跑完，整批才會結束。空轉 slot 數算錯了（120），continuous batching 的用途也答成了 static batching 本身的行為。 |
| Q1.2 | 🟡 | 4 個 process、只有 rank 0 送出 DetokenizeMsg，這兩點都對。「split brain」太籠統，沒有說出**具體會看到什麼現象**。 |
| Q1.3 | ✅ | 表格完全正確。沒有明寫，但從表格可以看出共 2 次 forward，結束原因是 `max_tokens` 用完（`remain_len = 0`）。 |
| Q1.4 | ✅ | 2,430,976,000 B（≈ 2.43 GB）對比 303,872 B，理由也正確。 |
| Q1.5 | ❌ | 160,000 ms = 8000 × 20 ms，把 prefill 當成「每個 token 跑一步 decode」來算，跟 Ch0 的 prefill 觀念相衝突。限制參數答「每批 prefill 的 chunk 大小」，方向正確（`max_extend_tokens` = 8192）。 |
| Q1.6 | ✅ | 答出 GIL，也說出了 GPU 閒置、TTFT/TPOT 抖動。補充一條更精確的因果鏈：tokenizer thread 握著 GIL → scheduler 迴圈拿不到 GIL，無法 launch 下一個 batch → GPU 在 step 之間空等。 |
| Q1.7 | 🟡 | 偵測者（API server）正確，prefill 期間要等到第一個 token 才會發現也正確。TP 部分正確：rank 0 會把收到的**每一則**訊息原樣轉發給其他 rank（`io.py:87-106`），不只 abort。訊息名稱錯了：應為 `AbortMsg`（API → tokenizer）與 `AbortBackendMsg`（tokenizer → scheduler）。漏了 scheduler 最後做的事。 |

→ 追問 Q1.1、Q1.2、Q1.5、Q1.7

### 你的原始答案

- **Q1.1**：1110 次 forward。浪費了 120 個 slot-step。可以平行跑三個 request，所以只需要 1000 步就能完成。
- **Q1.2**：4 個 process：一個 API server、一個 tokenizer/detokenizer、兩個 scheduler。DetokenizeMsg 由 scheduler 0 送出。如果 rank 1 也送，會導致 split brain 問題。
- **Q1.3**：表格（4/6/2/2 → 6/7/1/1 → 7/8/—/0，finished=True）
- **Q1.4**：只有最後一個 token 會決定下一個生成的 token。全部都算的話是 2,430,976,000 bytes，只取最後一個則是 303,872 bytes。
- **Q1.5**：所有現有使用者都要等 160,000 ms 才能繼續收到後續 token。設定每個 batch 的 prefill chunk 大小可以限制每一步的處理量。
- **Q1.6**：Python 的 GIL 會讓 GPU 和 CPU 互相等待，會看到 GPU 長時間閒置，TTFT 與 TPOT 的波動也很大。
- **Q1.7**：由 API server 發現，送出 userabort 和 decodeabort，從 API server 經過 tokenizer 到 scheduler，再由 rank 0 廣播給其他 TP rank。如果還在 prefill，要等到輸出第一個 token 時才會發現。

## 追問

| 題 | 結果 | 註記 |
|---|---|---|
| F1 | ✅ | 1000 步（由最長的 C 決定）；空轉 990 + 900 = 1890 slot-step；continuous batching 下，空出來的位置會立刻讓給新的 request。 |
| F2 | ✅ | 每個 token 都會出現兩次。 |
| F3 | ✅ | prefill 只跑 1 次 forward，所有 token 平行處理。延遲顯示在 TPOT 上。第一次答 40 ms，經追問後修正：prefill 那一步是 compute-bound，耗時遠大於 20 ms，所以要等 20 ms 加上 prefill 時間。這正是 Ch4 chunked prefill 要解決的問題。 |
| F4 | ✅ | 訊息是 `AbortMsg` → `AbortBackendMsg`，先從 prefill/decode 清單移除。`scheduler.py:201` 釋放 page table 的 row；`:202` 的 `cache_req` 不直接 free，而是把 KV 交給 prefix cache，原因是別的 request 可能正在共用，或之後還能重用。由 ref_count 決定這些 KV 什麼時候才能真正被 evict（Ch3）。 |

**Ch1 通過 ✅**
