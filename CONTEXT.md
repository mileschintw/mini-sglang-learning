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

## 關係

- 一個 **Upstream** 包含一個 **Frontend** 和一個 **Backend**
- mini-rsglang = Rust **Frontend** + Upstream 的 **Backend**，兩者透過 **Wire contract** 溝通
