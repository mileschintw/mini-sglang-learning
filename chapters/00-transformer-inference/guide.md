# Ch0 — Transformer 推論速查

> **用法：先做下面的考題，不要先讀講義。** 全部答對就直接進 Ch1；哪題卡住，再讀對應的小節。
> 本章只講 serving 會用到的部分，不涉及訓練、反向傳播或 loss。

程式碼連結都指向 upstream `9a91cfa`。本地路徑是 `~/Desktop/repos/mini-sglang/python/minisgl/`。

---

## 考題（請在對話中回答，紀錄會寫進 `quiz.md`）

**Q0.1 — Attention 的資料流**
在一個 attention layer 裡，Q、K、V 分別是從什麼東西算出來的？第 i 個 token 的 attention 輸出，取決於哪些 token 的哪些向量？causal mask 在這裡扮演什麼角色？

**Q0.2 — 為什麼 KV cache 成立**
模型已經生成了 t 個 token，現在要生成第 t+1 個。
(a) 這一步需要哪些 token 的 K 和 V？
(b) 前 t 個 token 的 K、V 為什麼可以沿用上一步算好的結果、不必重算？請說出兩個必要條件。
(c) 如果沒有 KV cache，生成 N 個 token 的總計算量大約是 N 的幾次方？有 KV cache 的話又是幾次方？

**Q0.3 — 讀程式碼：一層 decoder layer**
打開 [`models/llama.py:39-42`](https://github.com/sgl-project/mini-sglang/blob/9a91cfa/python/minisgl/models/llama.py#L39-L42) 和 [`layers/attention.py:47-57`](https://github.com/sgl-project/mini-sglang/blob/9a91cfa/python/minisgl/layers/attention.py#L47-L57)，依序說出一個 token 的 hidden state 經過一層 decoder layer 時，經歷了哪些運算。RoPE 加在哪裡？它作用在 Q、K、V 中的哪幾個？`residual` 是什麼？

**Q0.4 — 算 KV cache 有多大**
Qwen3-0.6B 的設定：28 層，`num_attention_heads=16`，`num_key_value_heads=8`，`head_dim=128`，權重型別 bf16。
(a) 每個 token 的 KV cache 佔多少 bytes？
(b) 如果 GPU 留給 KV cache 的空間是 4 GB，最多可以存多少個 token？
(c) 這個數字對「同時服務多少個使用者」有什麼意義？

**Q0.5 — GQA**
Q0.4 中，Q 的 head 有 16 個，K/V 的 head 卻只有 8 個，這叫 Grouped-Query Attention。它節省了什麼？為什麼這對 serving 特別重要，對訓練則沒那麼重要？

**Q0.6 — 預測題：為什麼沒有 batch 維度？**
在 [`llama.py:61`](https://github.com/sgl-project/mini-sglang/blob/9a91cfa/python/minisgl/models/llama.py#L61) 和 [`attention.py:55`](https://github.com/sgl-project/mini-sglang/blob/9a91cfa/python/minisgl/layers/attention.py#L55) 中，hidden state 的形狀是 `[num_tokens, hidden]`，而不是教科書上常見的 `[batch, seq_len, hidden]`。如果三個 request 的長度分別是 5、7、2，同時被送進同一次 forward，`num_tokens` 是多少？serving 引擎為什麼要把所有 token 攤平成一維，而不是 padding 成 `[3, 7, hidden]`？那 attention 怎麼知道哪些 token 屬於同一個 request？（最後這題先猜就好，Ch2 會講）

---

## 講義

### 1. Decoder-only LLM 的一次 forward

```
token ids ──embed──▶ x ──[Layer 0]──▶ ... ──[Layer L-1]──▶ norm ──lm_head──▶ logits ──sample──▶ next token
```

- **embed**：查表，把 token id 轉成 `hidden_size` 維的向量（[`llama.py:61`](https://github.com/sgl-project/mini-sglang/blob/9a91cfa/python/minisgl/models/llama.py#L61)）
- **lm_head**：把最後的 hidden state 投影回 `vocab_size` 維，得到每個候選 token 的分數，也就是 logits（[`llama.py:79-82`](https://github.com/sgl-project/mini-sglang/blob/9a91cfa/python/minisgl/models/llama.py#L79-L82)）
- **sample**：從 logits 中選出下一個 token。greedy 取 argmax；temperature 和 top-p 則是加入隨機性。對應 `engine/sample.py`，Ch5 會再提到

### 2. 一層 decoder layer（Llama / Qwen 家族）

[`llama.py:39-42`](https://github.com/sgl-project/mini-sglang/blob/9a91cfa/python/minisgl/models/llama.py#L39-L42) 的結構就是：

```
x, residual = RMSNorm(x + residual)   # pre-norm；fused 版本會把 residual add 一起做掉
x = Attention(x)
x, residual = RMSNorm(x + residual)
x = MLP(x)                            # gate_up_proj → SiLU(gate)*up → down_proj
```

**residual**（殘差）指的是「把這一層的輸入原封不動加回輸出」的那條旁路。mini-sglang 把 residual 單獨傳遞，讓 add 和 norm 可以融合成一個 kernel。

### 3. Attention 本體

[`attention.py:47-57`](https://github.com/sgl-project/mini-sglang/blob/9a91cfa/python/minisgl/layers/attention.py#L47-L57) 中每個 token 的處理步驟：

1. `qkv_proj` 做一次矩陣乘法，一口氣算出 q、k、v，然後切開（第 49 行）
2. （Qwen3 才有）對 q、k 各做一次 RMSNorm（第 50-53 行）
3. **RoPE 只作用在 q 和 k 上**（第 54 行），用 `positions` 把「這個 token 在序列中的位置」旋轉進向量裡。V 不加位置資訊
4. 呼叫 attention backend（第 56 行）。數學上每個 head 計算：

$$\text{out}_i = \sum_{j \le i} \text{softmax}_j\!\left(\frac{q_i \cdot k_j}{\sqrt{d}}\right) v_j$$

`j ≤ i` 就是 **causal mask**：每個 token 只能看到自己以及之前的 token。真正的 kernel（FlashAttention、FlashInfer）會**順便把新的 k、v 寫進 KV cache**，再從 cache 讀出所有歷史 k、v 來計算，Ch2 和 Ch5 會詳細說明。

### 4. 為什麼可以有 KV cache

位置 j 的 k_j、v_j 只取決於：(1) 位置 j 以前的 token，因為有 causal mask，未來的 token 影響不到它；(2) 固定的模型權重。所以 token 一旦生成，它在每一層的 K、V 就**永遠不會再改變**，可以存起來重複使用。

- **沒有 cache**：第 t 步要重算整條長度為 t 的序列，總計算量是 1+2+…+N，也就是 O(N²) 次 token-forward
- **有 cache**：每一步只需要算 1 個新 token 的 forward。不過 attention 仍然要讀取全部 t 個歷史 K/V，所以 attention 部分是 O(t)，整體從 O(N²) 次 forward 降為 O(N) 次

### 5. 由此推出的兩個階段（Ch1 的主角）

| | **Prefill** | **Decode** |
|---|---|---|
| 輸入 | 整段 prompt，數百到數千個 token | 每個 request 只有 1 個新 token |
| 做的事 | 為所有 prompt token 算出 K/V 並寫進 cache，然後吐出第 1 個 token | 讀取整個 cache，吐出下一個 token |
| 瓶頸 | **compute-bound**：矩陣乘法很大，GPU 算力吃滿 | **memory-bound**：每一步都要把整份權重和 KV cache 從顯存讀一遍，卻只算出 1 個 token |
| 對應指標 | TTFT（time to first token，第一個 token 出現前的等待時間） | TPOT / ITL（之後每個 token 的間隔時間） |

decode 是 memory-bound 的，所以**把很多 request 湊成一個 batch 一起 decode**，同樣讀一次權重就能產出很多個 token。這就是 serving 引擎存在的根本理由，也是它所有複雜度的來源。

### 6. KV cache 有多大？

```
每個 token 的 KV bytes = 2 (K 和 V) × 層數 × num_kv_heads × head_dim × 每個元素的 bytes
```

這條公式會決定 GPU 能同時容納多少個 request。Q0.4 請自己算一次，Ch2 會用到這個結果。

### 7. GQA（Grouped-Query Attention）

多個 Q head 共用同一組 K/V head（[`attention.py:29`](https://github.com/sgl-project/mini-sglang/blob/9a91cfa/python/minisgl/layers/attention.py#L29) 有 `assert num_qo_heads % num_kv_heads == 0`）。這樣 KV cache 大小只取決於 `num_kv_heads`，Q head 再多也不影響。

---

## 面試題

1. **「為什麼 LLM 推論要分 prefill 和 decode 兩個階段？它們的效能瓶頸分別是什麼？」**
   要點：prefill 一次處理整段 prompt，矩陣乘法大，是 compute-bound；decode 每步只處理一個 token，需要反覆讀取權重和 KV，是 memory-bandwidth-bound。前者主要影響 TTFT，後者影響 TPOT。
2. **「估算一下某個 7B 模型處理 4K context 時，每個 request 的 KV cache 有多大？」**
   要點：說得出公式 `2 × L × H_kv × d × bytes × seq_len`；知道 GQA 和 MQA 是用來縮小這個數字的；知道 FP8 KV cache 可以再減半。
3. **「KV cache 為什麼成立？什麼情況下 cache 會失效？」**
   要點：causal mask 加上固定權重，所以舊 token 的 K/V 不變。失效的情況包括：換了 LoRA 或權重、position 改變（RoPE 會把位置編進 K）、序列前面的 token 被修改。
