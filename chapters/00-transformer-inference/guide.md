# Ch0 — Transformer 推論：從零開始

> **用法**：從頭到尾讀一遍，約 40 到 50 分鐘。§3 的數值例子請**拿紙筆跟著算**，這是整章最重要的部分。讀完後做最下面的「第 2 輪考題」。
> 本章只講 serving 需要的部分，不講訓練、反向傳播和 loss。
> 程式碼連結都指向 upstream `9a91cfa`，本地路徑是 `~/Desktop/repos/mini-sglang/python/minisgl/`。

---

## 1. 大圖：LLM 只做一件事

**給定前面所有的 token，算出「下一個 token 是什麼」的機率分布。** 生成一段文字，就是把這件事一直重複：

```
"我 愛"      → 模型 → 下一個是「貓」的機率 60%、「狗」30%…  → 選出「貓」
"我 愛 貓"   → 模型 → 下一個是「。」的機率 70%…            → 選出「。」
...
```

這種「每次吐出一個 token，然後把它接回輸入」的方式叫 **autoregressive（自迴歸）生成**。

一次 forward 的完整流程如下：

```
token ids ─embed─▶ x ─[Layer 0]─▶ x ─[Layer 1]─▶ … ─[Layer L-1]─▶ norm ─lm_head─▶ logits ─sample─▶ next token
 (整數)          (向量)                                                     (vocab 個分數)
```

| 步驟 | 做什麼 | mini-sglang 程式碼 |
|---|---|---|
| embed | 查表，把每個 token id 換成一個 `hidden_size` 維的向量 | [`llama.py:61`](https://github.com/sgl-project/mini-sglang/blob/9a91cfa/python/minisgl/models/llama.py#L61) |
| L 層 decoder layer | 每層都「讀入一堆向量、吐出同樣形狀的一堆向量」，逐層把資訊加工得更精煉 | [`llama.py:63-64`](https://github.com/sgl-project/mini-sglang/blob/9a91cfa/python/minisgl/models/llama.py#L63-L64) |
| lm_head | 把最後的向量投影成 `vocab_size` 個分數，每個候選 token 一個分數，叫 logits | [`llama.py:79-82`](https://github.com/sgl-project/mini-sglang/blob/9a91cfa/python/minisgl/models/llama.py#L79-L82) |
| sample | 從 logits 選出下一個 token | [`engine/sample.py`](https://github.com/sgl-project/mini-sglang/blob/9a91cfa/python/minisgl/engine/sample.py) |

**一個關鍵觀察**：每個 token 在每一層都有**自己的一個向量**。一層 layer 由兩個子層組成：
- **MLP**：每個 token 各算各的，token 之間**完全不互動**
- **Attention**：**唯一**讓 token 之間交換資訊的地方

這個觀察是本章後面好幾個結論的基礎，請先記住。

---

## 2. Attention 的直覺：在圖書館裡找資料

想像每個 token 都是圖書館裡的一個人，每個人同時扮演三個角色：

| 向量 | 角色 | 比喻 |
|---|---|---|
| **q**（query） | 「**我**想找什麼資訊？」 | 我手上的搜尋關鍵字 |
| **k**（key） | 「**我**這裡有什麼樣的資訊？」 | 我貼在身上的標籤，讓別人判斷要不要看我 |
| **v**（value） | 「如果有人看我，我提供什麼內容？」 | 我實際交出去的內容 |

第 i 個 token 做 attention 的過程：
1. 拿自己的 **q_i**，和每個 token j 的 **k_j** 做內積，得到「我對 j 有多感興趣」的分數
2. 用 softmax 把分數變成加起來等於 1 的權重
3. 依照權重，把每個 token j 的 **v_j** 加權平均，得到結果

> ### ⚠️ 修正第 1 輪的誤解
> **q、k、v 不是先後發生的三個階段。** 它們是**同一個輸入向量 x_i**，分別乘上三個不同的權重矩陣，**同時**算出來的：
>
> ```
> q_i = x_i · W_q        k_i = x_i · W_k        v_i = x_i · W_v
> ```
>
> W_q、W_k、W_v 是訓練好的固定參數，推論時永遠不變。V 也不是「經過整個模型才產生」的，它跟 q、k 一樣，在這一層一開始就算好了。
>
> 在 mini-sglang 中，這三個乘法甚至合併成**一次**矩陣乘法，算完再切成三份：
> [`utils.py:120`](https://github.com/sgl-project/mini-sglang/blob/9a91cfa/python/minisgl/models/utils.py#L120) 的 `qkv_proj` 負責合併計算，[`attention.py:49`](https://github.com/sgl-project/mini-sglang/blob/9a91cfa/python/minisgl/layers/attention.py#L49) 的 `qkv.split(...)` 負責切開。

公式寫成：

$$\text{out}_i = \sum_{j} \underbrace{\text{softmax}_j\!\left(\frac{q_i \cdot k_j}{\sqrt{d}}\right)}_{\text{i 對 j 的注意力權重}} \; v_j$$

其中 √d（d 是向量維度）只是把分數縮小，避免 softmax 太尖銳，不影響理解。

---

## 3. 手算一次（請拿紙筆）

設定：序列是「我 愛 貓」，向量維度 d = 2。下面的數字是刻意挑過的，方便手算。

**輸入向量 x**（假設已經 embed 好了）：

| token | 位置 | x |
|---|---|---|
| 我 | 0 | [1, 0] |
| 愛 | 1 | [0, 1] |
| 貓 | 2 | [1, 1] |

**三個權重矩陣**（固定不變）：
- W_q = 單位矩陣，所以 **q = x**
- W_k = 交換兩個座標，所以 **k = [x₁, x₀]**
- W_v = 把第一個座標乘 2，所以 **v = [2·x₀, x₁]**

### 步驟 1：每個 token 各自算出自己的 q、k、v（互不相干，可以同時算）

| token | q | k | v |
|---|---|---|---|
| 我 (0) | [1, 0] | [0, 1] | [2, 0] |
| 愛 (1) | [0, 1] | [1, 0] | [0, 1] |
| 貓 (2) | [1, 1] | [1, 1] | [2, 1] |

### 步驟 2：算「貓」(i=2) 的 attention 輸出

1. **分數**：q₂ = [1,1] 分別和每個 k 做內積
   - q₂·k₀ = 1×0 + 1×1 = **1**
   - q₂·k₁ = 1×1 + 1×0 = **1**
   - q₂·k₂ = 1×1 + 1×1 = **2**
2. **縮放**：除以 √2，得到 0.71、0.71、1.41
3. **softmax**：e^0.71 ≈ 2.03，e^1.41 ≈ 4.11，總和 ≈ 8.17，所以權重是 **[0.25, 0.25, 0.50]**
4. **加權平均 v**：0.25×[2,0] + 0.25×[0,1] + 0.50×[2,1] = **[1.50, 0.75]**

「貓」把一半的注意力放在自己身上，另外兩個 token 各分到四分之一。輸出向量混入了「我」和「愛」的資訊，這就是 token 之間交換資訊的方式。

### 步驟 3：Causal mask

如果算的是「愛」(i=1) 呢？**它只能看自己和之前的 token（j ≤ 1），不能看到後面的「貓」**。理由很直接：生成「愛」的時候，「貓」根本還沒出現。訓練時也必須遵守同樣的規則，否則模型會學會偷看答案。

這個「只能往前看」的規則就叫 **causal mask**。實作上是把 j > i 的分數設成 −∞，softmax 之後權重就變成 0：

```
          看 j=0(我)  j=1(愛)  j=2(貓)
i=0 (我)    ✓         −∞       −∞
i=1 (愛)    ✓         ✓        −∞
i=2 (貓)    ✓         ✓        ✓
```

### 步驟 4：下一個 token 進來了，KV cache 登場

假設模型選出下一個 token「。」(i=3)，接下來要算它的 attention。需要哪些東西？
- **q₃**：「。」自己的 query，要新算
- **k₀..k₃、v₀..v₃**：所有 token（包含自己）的 key 和 value

**k₀..k₂、v₀..v₂ 要重算嗎？不用。** 它們上一步就已經算過了，而且值不會改變：
- 它們只取決於 x₀..x₂ 和 W_k、W_v
- **權重固定**，所以 W 不變
- **因為有 causal mask**，「我、愛、貓」在每一層的向量都只看得到自己和更前面的 token，看不到新加入的「。」，所以 x₀..x₂ 在**每一層**都不變

所以只要把算過的 K、V 存起來，下一步直接拿來用就好。**這就是 KV cache。**

> **兩個必要條件**：(1) 模型權重不變；(2) causal mask，也就是舊 token 看不到新 token。
>
> 反例：BERT 這類**雙向**模型沒有 causal mask，每個 token 都看得到所有其他 token。新 token 一加入，舊 token 在第 2 層以後的向量全部會改變，K、V 都得重算，KV cache 就不成立了。
>
> **為什麼不 cache Q？** 因為 q 只在「我去看別人」時使用。舊 token 的 q 在它自己那一步用完後，就再也用不到了。

### 步驟 5：省下多少計算？

我們用「做了幾次 token-forward」來計算（一個 token 通過整個模型一次算一次）：

| | 第 t 步要算幾個 token | 生成 N 個 token 的總量 |
|---|---|---|
| 沒有 KV cache | 整段 t 個 token 從頭重算 | 1+2+…+N ≈ N²/2，也就是 **O(N²)** |
| 有 KV cache | 只算 1 個新 token | **O(N)** |

（attention 的部分要讀取全部 t 個歷史 K、V，所以那部分仍然會隨長度增加。不過「重算」的浪費已經被消除了。第 1 輪答的 O(N^N) 是指數級，遠遠高估了。）

**代價**：KV cache 要佔用 GPU 記憶體，而且會隨著 token 數量一直成長。**LLM serving 引擎有一大半的複雜度，都是在管理這塊記憶體**，這是 Ch2 到 Ch4 的主題。

---

## 4. 真實模型比手算例子多了什麼

### 4.1 Multi-head attention

真實模型不只做一組 attention，而是把向量切成好幾份，叫作 **head**，每個 head 在自己的子空間裡獨立做一次 §3 的流程。有的 head 可能專門看語法關係，有的專門看指代對象。每個 head 的輸出最後接起來，再用 `o_proj` 投影回 `hidden_size`（[`utils.py:123`](https://github.com/sgl-project/mini-sglang/blob/9a91cfa/python/minisgl/models/utils.py#L123)）。

以 Qwen3-0.6B 為例：Q 有 16 個 head，每個 head 的維度 `head_dim` 是 128。

### 4.2 GQA（Grouped-Query Attention）

最早的 Multi-Head Attention（MHA）中，每個 Q head 都配有自己的一組 K、V head。**GQA 讓好幾個 Q head 共用同一組 K、V head**：

```
MHA:  Q head  0 1 2 3 4 5 …15     K/V head  0 1 2 3 4 5 …15    （16 組 KV）
GQA:  Q head  0 1 | 2 3 | … |14 15  K/V head  0  |  1  | … |  7   （8 組 KV，每 2 個 Q head 共用 1 組）
MQA:  所有 Q head 共用 1 組 KV                                    （1 組 KV）
```

- **省下什麼**：KV cache 的大小只取決於 KV head 的數量。16 組改成 8 組，**KV cache 就減半**
- **為什麼對 serving 特別重要**：
  - 訓練時沒有 KV cache，整段序列一次平行算完
  - 推論的 decode 階段，**每一步都要把整份 KV cache 從顯存讀出來**。所以 KV cache 越小，同一張卡可以塞進越多 request，每一步也讀得越快
- **品質影響**：Q head 數量不變，模型「問問題」的能力保留，實務上品質損失很小

程式碼中的痕跡：
- [`attention.py:29`](https://github.com/sgl-project/mini-sglang/blob/9a91cfa/python/minisgl/layers/attention.py#L29)：`assert num_qo_heads % num_kv_heads == 0`，Q head 的數量必須能整除成組
- [`attention.py:35-36`](https://github.com/sgl-project/mini-sglang/blob/9a91cfa/python/minisgl/layers/attention.py#L35-L36)：q 的維度是 16×128 = 2048，k、v 的維度各是 8×128 = 1024

### 4.3 RoPE：讓模型知道 token 的順序

回頭看 §3：如果把「我 愛 貓」打亂成「貓 我 愛」，每一對 token 的 q·k 分數**完全不變**。attention 本身不知道 token 的順序。

**RoPE（Rotary Position Embedding）** 的做法是：依照 token 的位置，把 q 和 k **旋轉一個角度**，位置越後面轉得越多。這樣 q_i·k_j 的結果就會包含「i 和 j 相距多遠」的資訊。

- **只作用在 q 和 k 上，不作用在 v**。位置資訊只需要影響「誰該注意誰」，也就是分數；v 是被取走的內容，不需要位置資訊
- 程式碼：[`attention.py:54`](https://github.com/sgl-project/mini-sglang/blob/9a91cfa/python/minisgl/layers/attention.py#L54)，用 `batch.positions` 告訴 RoPE 每個 token 的位置
- **你第 1 輪提到的「放大 context window」**，指的是 **RoPE scaling**（例如 YaRN）。它調整旋轉角度，讓模型能處理比訓練時更長的序列，算是 RoPE 的衍生技巧。對應 [`attention.py:42`](https://github.com/sgl-project/mini-sglang/blob/9a91cfa/python/minisgl/layers/attention.py#L42) 的 `rope_scaling`

> **🔑 對 serving 很重要的推論**：KV cache 裡存的是**套用 RoPE 之後**的 k，所以位置資訊已經「烤」進 cache 裡了。因此，同一段文字如果出現在**不同位置**，它的 KV **不能**直接共用。Ch3 的 prefix cache 只能共用「從頭開始完全相同」的前綴，原因就在這裡。

### 4.4 Qwen3 的 QK-norm

Qwen3 在 RoPE 之前，會先對 q、k 各做一次 RMSNorm，目的是讓訓練更穩定（[`attention.py:50-53`](https://github.com/sgl-project/mini-sglang/blob/9a91cfa/python/minisgl/layers/attention.py#L50-L53)，Qwen3 在 [`qwen3.py:20`](https://github.com/sgl-project/mini-sglang/blob/9a91cfa/python/minisgl/models/qwen3.py#L20) 開啟這個功能）。這對 serving 沒有影響，知道有這回事就好。

---

## 5. 一層 decoder layer 的完整運算順序

### 5.1 Residual stream（殘差流）

可以把每個 token 想像成有一條**一直往下流的主幹向量**，叫作 residual stream。每個子層（attention、MLP）的運作方式都是：

1. 讀取主幹的一份**正規化副本**
2. 算出一個「修正量」
3. 把修正量**加回**主幹

主幹本身從不被覆蓋，只會被累加。這就是 residual 的意思，也是深層網路能穩定訓練的原因。你第 1 輪說的「讓上一層的資料可以直接漏到下一層」，方向是對的。

### 5.2 對照程式碼

[`llama.py:39-42`](https://github.com/sgl-project/mini-sglang/blob/9a91cfa/python/minisgl/models/llama.py#L39-L42)：

```python
x, residual = self.input_layernorm.forward(x, residual)           # ①
x = self.self_attn.forward(x)                                     # ②
x, residual = self.post_attention_layernorm.forward(x, residual)  # ③
x = self.mlp.forward(x)                                           # ④
```

`RMSNormFused`（[`norm.py:32-38`](https://github.com/sgl-project/mini-sglang/blob/9a91cfa/python/minisgl/layers/norm.py#L32-L38)）把「加回主幹」和「正規化」融合成一個 kernel：`residual ← x + residual`，然後 `x ← RMSNorm(residual)`。在這裡，`residual` 變數就是主幹本身。

| 步驟 | 運算 | 細節 |
|---|---|---|
| ① | 把上一個子層的輸出加回主幹，再做 RMSNorm | 第 0 層時 residual 還是 None，直接把 embedding 當成主幹（`norm.py:35-36`） |
| ② | **Attention** | `qkv_proj` → 切成 q、k、v → (QK-norm) → **對 q、k 套 RoPE** → attention backend（**順便把這步的 k、v 寫進 KV cache**）→ `o_proj` |
| ③ | 把 attention 的輸出加回主幹，再做 RMSNorm | |
| ④ | **MLP** | `gate_up_proj` → SiLU(gate) × up → `down_proj`（[`utils.py:45-50`](https://github.com/sgl-project/mini-sglang/blob/9a91cfa/python/minisgl/models/utils.py#L45-L50)）。**每個 token 各算各的** |

④ 的輸出要等到**下一層的 ①** 才加回主幹。最後一層的輸出，則由 [`llama.py:65`](https://github.com/sgl-project/mini-sglang/blob/9a91cfa/python/minisgl/models/llama.py#L65) 的 final norm 加回。

**「寫進 KV cache」實際發生的位置**在 attention backend 裡，例如 FlashInfer 版本的 [`attention/fi.py:185-188`](https://github.com/sgl-project/mini-sglang/blob/9a91cfa/python/minisgl/attention/fi.py#L185-L188)：先執行 `store_kv` 把新的 k、v 寫進 cache，再從 cache 讀出所有歷史的 k、v 來算 attention。這兩行就是 §3 步驟 4 的真實版本。

---

## 6. 為什麼 serving 引擎把所有 token 攤平成一維

回到 §1 的關鍵觀察：除了 attention，其他運算（embed、各種 linear、norm、MLP、lm_head）**都是每個 token 各自獨立計算**。對這些運算來說，「這個 token 屬於哪個 request」根本不重要。

所以 serving 引擎會把**所有 request 的所有 token 接成一長條**，形狀是 `[num_tokens, hidden]`（[`attention.py:55`](https://github.com/sgl-project/mini-sglang/blob/9a91cfa/python/minisgl/layers/attention.py#L55) 的 `view(-1, ...)` 就是在做這件事）：
- **不需要 padding**：長度不同的 request 直接接在一起，不會把算力浪費在補零上
- **只有 attention 需要知道邊界**：每個 token 只能看**同一個 request 裡**、在自己之前的 token。引擎會另外傳一份「每個 request 有幾個 token、它的 KV 放在哪裡」的 **metadata** 給 attention backend。例如 [`fi.py:194-195`](https://github.com/sgl-project/mini-sglang/blob/9a91cfa/python/minisgl/attention/fi.py#L194-L195) 的 `seqlens_q` 和 `seqlens_k`

這種「長度參差不齊、直接接起來」的 batch 叫 **ragged batch**。metadata 怎麼建立，是 Ch2 和 Ch4 的內容。

---

## 7. KV cache 有多大？

```
每個 token 的 KV bytes = 2 (K 和 V) × 層數 × num_kv_heads × head_dim × 每個元素的 bytes
```

**範例：Qwen3-0.6B**（28 層、8 個 KV head、`head_dim` 128、bf16 每個元素 2 bytes）

- 每個 token：2 × 28 × 8 × 128 × 2 = **114,688 bytes ≈ 112 KiB**
- 4 GiB 能放：4 × 1024³ ÷ 114,688 ≈ **37,000 個 token**
- **意義**：這 3.7 萬個 token 是**所有 request 共用的總額度**
  - 每個對話佔 2,000 個 token 的話，最多同時服務大約 18 個人
  - 每個請求帶 32K context 的話，一次只能服務 1 個人
- 如果沒有 GQA（16 個 KV head），每個 token 要 224 KiB，能服務的人數**再減半**

**對照**：模型權重是 0.6B 參數 × 2 bytes ≈ 1.2 GB。在長 context、高並發的情況下，**KV cache 往往比模型本身還大**。這就是為什麼 KV cache 管理會成為 serving 的核心問題。

---

## 8. Prefill 與 Decode：兩個截然不同的階段

一個 request 的一生分成兩段：

| | **Prefill** | **Decode** |
|---|---|---|
| 輸入 | 整段 prompt，數百到數千個 token **一次全部送進去** | 每個 request 只送 **1 個**新 token |
| 做的事 | 為所有 prompt token 算出 K、V 並寫進 cache，然後吐出第 1 個 token | 讀取整份 KV cache，吐出下一個 token，一直重複到結束 |
| 瓶頸 | **compute-bound**（算力不夠） | **memory-bound**（顯存頻寬不夠） |
| 對應指標 | **TTFT**（time to first token） | **TPOT / ITL**（time per output token，每個 token 之間的間隔） |

**為什麼 decode 是 memory-bound？** 以 RTX 3050 為例，它的顯存頻寬大約是 224 GB/s：
- decode 一步要把 1.2 GB 的權重全部從顯存讀一遍，光是讀取就要約 **5 ms**
- 讀進來之後，如果只算 1 個 token，GPU 的算力大部分都閒著
- **但如果一次湊 32 個 request 一起 decode**，權重一樣只讀一次，時間幾乎還是 5 ms，產出卻變成 32 倍

> **這就是 batching 的根本理由，也是 serving 引擎存在的原因。** 要想辦法同時容納很多個 request，所以需要管理 KV cache（Ch2、Ch3）；要決定這一步讓誰上場，所以需要 scheduler（Ch4）。

**為什麼 prefill 是 compute-bound？** 一次處理 1,000 個 token，讀進來的每個權重會被重複使用 1,000 次，所以瓶頸變成算力，而不是讀取。

---

## 9. Sampling（帶過即可）

logits 轉成 token 的方式由每個 request 自己的 `SamplingParams` 決定（[`core.py:16-25`](https://github.com/sgl-project/mini-sglang/blob/9a91cfa/python/minisgl/core.py#L16-L25)）：
- **greedy**（`temperature=0`）：直接選分數最高的
- **temperature / top-k / top-p**：加入隨機性，讓輸出更多樣

同一個 batch 裡的 request 可以各自用不同的參數。Ch5 會再提到。

---

## 10. 讀完你應該能說清楚

- [ ] q、k、v 都是從同一個 x_i 同時算出來的；q 決定「我要看誰」，k 決定「我要不要被看」，v 決定「被看時交出什麼」
- [ ] causal mask 加上固定權重，讓舊 token 的 K、V 永遠不變，所以可以 cache；Q 不需要 cache
- [ ] KV cache 把重算量從 O(N²) 降到 O(N)，代價是會持續成長的顯存佔用
- [ ] GQA 讓多個 Q head 共用 KV，KV cache 因此縮小幾倍
- [ ] RoPE 只作用在 q、k；cache 中存的是旋轉後的 k，所以 KV 跟位置綁在一起
- [ ] 除了 attention，其他運算都是 per-token，所以可以把所有 token 攤平成 ragged batch
- [ ] 會算每個 token 的 KV bytes，並能換算成可服務的人數
- [ ] prefill 是 compute-bound，decode 是 memory-bound，batching 能讓 decode 的吞吐量大幅提升

---

## 面試題

1. **「為什麼 LLM 推論要分 prefill 和 decode？兩者的瓶頸分別是什麼？」**
   要點：prefill 一次處理整段 prompt，矩陣乘法夠大，是 compute-bound，影響 TTFT。decode 每步每個 request 只算 1 個 token，卻要讀取全部權重和 KV，是 memory-bandwidth-bound，影響 TPOT。batching 可以把讀權重的成本分攤到很多 request 上。
2. **「估算 Llama-3-8B 處理 8K context 時，每個 request 的 KV cache 有多大？」**
   要點：說得出公式 `2 × L × H_kv × d × bytes × seq_len`。Llama-3-8B 是 32 層、8 個 KV head、`head_dim` 128，每個 token 約 128 KiB，8K context 約 1 GiB。知道 GQA、MQA、FP8 KV cache 都是用來縮小這個數字的。
3. **「KV cache 為什麼成立？什麼時候會失效？」**
   要點：causal mask 加上固定權重。失效情況包括：換了權重或 LoRA、同一段文字出現在不同位置（RoPE 已經烤進 K）、序列前面的 token 被修改。
4. **「為什麼不 cache Q？」**
   要點：舊 token 的 q 只在它自己那一步使用，之後不再被需要。
5. **「MHA、GQA、MQA 的差別與取捨？」**
   要點：三者的差別在 KV head 的數量（全部、分組、只有 1 個）。KV cache 越小，吞吐量和並發越好，但品質可能下降。GQA 是目前的主流折衷方案。

---

## 第 2 輪考題

> 請在對話中作答。題目和第 1 輪不同，數字也換過了，目的是確認你真的理解，而不是記住講義裡的答案。

**R2.1** 用你自己的話說明：x_i 如何變成 q_i、k_i、v_i？三者之中，哪一個決定「我要看誰」，哪一個決定「別人看我時拿到什麼」？為什麼生成時不需要 cache Q？

**R2.2** KV cache 成立的兩個必要條件是什麼？如果把同一個模型的 causal mask 拿掉，改成雙向 attention，新 token 加入時會發生什麼事？

**R2.3** N = 1,000 時，沒有 KV cache 與有 KV cache 時，各需要做大約多少次 token-forward？（給個數量級即可）

**R2.4** 依序說出一層 decoder layer 的運算，從進入 ① 到離開 ④。RoPE 作用在誰身上？為什麼 V 不需要 RoPE？KV cache 存的是套用 RoPE「之前」還是「之後」的 K？這對「同一段 system prompt 在兩個 request 中都出現在開頭」和「出現在不同位置」這兩種情況，分別意味著什麼？

**R2.5** Llama-3.2-1B 的設定：16 層、32 個 Q head、8 個 KV head、`head_dim` 64、bf16。
(a) 每個 token 的 KV 佔多少 bytes？
(b) 4 GiB 能放多少個 token？
(c) 如果它改用 MHA（32 個 KV head），(b) 會變成多少？

**R2.6** 三個 request 的長度分別是 100、3、50，在同一次 forward 中做 prefill。
(a) `num_tokens` 是多少？
(b) 如果改成 padding 成 `[3, 100, hidden]`，有多少比例的計算浪費在補零上？
(c) 哪一種運算需要知道 request 的邊界？它是怎麼得知的？

**R2.7** 為什麼 decode 時，batch size 從 1 增加到 32，每一步的時間卻幾乎不變？在什麼情況下這個規律會失效？（第二個問題可以用猜的）
