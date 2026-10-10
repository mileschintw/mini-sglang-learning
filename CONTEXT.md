# LLM Serving 術語表

這份術語表涵蓋 mini-sglang / mini-rsglang 學習過程中確定下來的術語。只收錄「這個詞在這個系統裡指什麼」，不寫實作細節。

## 系統邊界

**Upstream（上游）**:
原版 mini-sglang，釘在 commit `9a91cfa`。mini-rsglang 把它原封不動地放在 `vendor/`。
_Avoid_: 原版、Python 版（這兩個說法都會跟 Python frontend 混淆）

**Frontend（前端）**:
面向使用者的那一層：HTTP API、tokenize、detokenize、request 的生命週期管理。這是 mini-rsglang 用 Rust 改寫的唯一部分。
_Avoid_: server（太籠統）

**Backend（後端）**:
scheduler、KV cache 管理、model forward 與 kernel。在 mini-rsglang 中，這一層仍然是凍結的 upstream Python/CUDA 程式碼。
_Avoid_: engine（Engine 是後端裡的一個具體類別，不等於整個後端）

**Wire contract（線上協定）**:
前端與後端之間透過 ZMQ 傳遞的 msgpack 訊息格式。Rust 前端必須與 Python 前端做到 byte-for-byte 一致。

**Parity（一致性）**:
在相同的後端與輸入下，Rust 前端與 Python 前端的輸出完全相同。

## 推論基礎（Ch0）

**KV cache**:
每個 token 在每一層算出的 K、V 向量，存起來供後續 token 的 attention 重複使用。它之所以成立，是因為兩個條件：權重固定，加上 causal mask。
_Avoid_: cache（太籠統；系統裡還有 prefix cache）

**Prefill**:
一個 request 的第一個階段：把整段 prompt 一次送進模型，寫入 KV cache，並產出第 1 個 token。瓶頸在算力（compute-bound）。
_Avoid_: 預處理、encode

**Decode**:
prefill 之後的階段：每一步只送入 1 個新 token，產出下一個 token。瓶頸在顯存頻寬（memory-bound）。
_Avoid_: 生成（太籠統，prefill 也會產出 token）

**TTFT / TPOT**:
TTFT（time to first token）是從送出 request 到收到第 1 個 token 的時間，主要由 prefill 決定。TPOT（time per output token）是之後每個 token 之間的間隔，主要由 decode 決定。

**Ragged batch**:
把多個 request 的 token 直接接成一維 `[num_tokens, hidden]`、不做 padding 的 batch。request 的邊界由另外傳遞的 metadata 標示。

## 關係

- 一個 **Upstream** 包含一個 **Frontend** 和一個 **Backend**
- mini-rsglang = Rust **Frontend** + Upstream 的 **Backend**，兩者透過 **Wire contract** 溝通
