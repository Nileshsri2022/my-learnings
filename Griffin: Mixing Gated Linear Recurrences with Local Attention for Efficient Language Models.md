

# Griffin: Mixing Gated Linear Recurrences with Local Attention for Efficient Language Models

---

## 🎯 1. THE ONE-LINER
**Griffin is a new type of language model that combines a smart "memory shortcut" (gated linear recurrences) with a "look nearby" trick (local attention) to understand language as well as the best models but run much faster, especially on long texts.**

---

## 🤔 2. THE PROBLEM (Why does this paper exist?)

- **The core problem:** Transformers (the architecture behind ChatGPT, etc.) get **really slow and memory-hungry** as text gets longer because their attention mechanism compares every word to every other word (quadratic cost). Their memory cache also grows linearly with sequence length, making inference slow.

- **Why should anyone care?** Imagine you're reading a book and for every new word, you have to re-read *every single previous word* to understand it. That's what Transformers do. It's exhausting and slow. Wouldn't it be better to have a running summary in your head (like an RNN) plus just glancing at the last few sentences (local attention)?

- **Limitations of previous approaches:**
  - **Traditional RNNs (LSTMs, GRUs):** Hard to train (slow, sequential), hard to scale, can't match Transformer quality
  - **State Space Models (S4, Mamba):** Better than old RNNs, but still **struggle with copying/retrieval tasks** and haven't convincingly matched Transformers at scale
  - **Pure local attention:** Loses information from distant context
  - **No prior work** convincingly showed an RNN-based model **matching Llama-2 quality at 7B+ scale** with comparable training efficiency

---

## 💡 3. THE KEY IDEA (The "Aha!" moment)

**Core insight:** Don't choose between RNNs and Transformers — **mix them strategically**. Use a novel gated linear recurrence (RG-LRU) for long-range memory compression, and sliding-window local attention for precise short-range recall. Each does what it's best at.

**Cooking analogy:** Think of it like making a stew:
- The **recurrent layer (RG-LRU)** is like a slow-simmering broth that captures the overall flavor of everything you've added (long-range context compressed into a fixed-size state)
- The **local attention** is like having the last few ingredients on the counter so you can taste-check exactly what you just put in (precise short-range memory)
- Together, you get **both** rich deep flavor AND precise recent additions

**The novel RG-LRU gate trick:**
- Most gates (GRU, Mamba) can **forget everything** from the past and replace it with new input
- The RG-LRU gate is **biased toward retaining information** — it interpolates between the standard LRU update and just keeping the old hidden state, effectively letting the model **ignore uninformative tokens** without losing past knowledge

**Architecture at a glance (ASCII):**

```
Input Tokens
    |
    v
[Embedding]
    |
    v
┌─────────────────────────┐
│   RESIDUAL BLOCK (×N)   │
│  ┌───────────────────┐  │
│  │ RMSNorm           │  │
│  │ Temporal Mix Block │  │  ← Either: Recurrent Block (RG-LRU)
│  │   + Skip Connect  │  │     OR: Local MQA Attention
│  ├───────────────────┤  │     (Griffin alternates 2:1 ratio)
│  │ RMSNorm           │  │
│  │ Gated MLP Block   │  │
│  │   + Skip Connect  │  │
│  └───────────────────┘  │
└─────────────────────────┘
    |
    v
[RMSNorm → Linear → Softmax]
    |
    v
Output Probabilities
```

---

## 🏗️ 4. HOW IT WORKS (The Method - Layer by Layer)

### Step 1: Token Embedding
- **WHAT:** Convert input tokens to vectors of dimension D
- **WHY:** Neural networks need numerical representations
- **CONNECTS TO:** Fed into the stack of residual blocks

### Step 2: Residual Block (repeated N times)
- **WHAT:** Two sub-blocks applied sequentially, each with RMSNorm → processing → skip connection
- **WHY:** The pre-norm residual pattern (from Transformers) enables stable deep training
- **CONNECTS TO:** First sub-block does temporal mixing, second does channel mixing (MLP)

### Step 3: Temporal Mixing (the key innovation)
Three variants exist:

**3a. Recurrent Block (used in Hawk & Griffin):**
- Input splits into **two parallel branches** (both via linear projections to dimension D_RNN)
- **Branch 1:** Conv1D (kernel size 4) → **RG-LRU layer** (the novel recurrence)
- **Branch 2:** GeLU activation
- Branches merge via **element-wise multiplication** → final linear projection back to D
- **WHY Conv1D?** Gives the recurrence a small local context before processing (borrowed from H3/Mamba)

**3b. Local Sliding-Window MQA (used in Griffin):**
- Standard multi-query attention but each token only attends to the **last 1024 tokens**
- Uses RoPE positional encoding
- **WHY:** Bounds KV cache size, still gives precise short-range attention

**Griffin's mixing strategy:** Alternate **2 recurrent blocks** then **1 local attention block**, repeat.

### Step 4: The RG-LRU Layer (the star of the show)
```
r_t = σ(W_a · x_t + b_a)           # Recurrence gate (0 to 1)
i_t = σ(W_x · x_t + b_x)           # Input gate (0 to 1)  
a_t = a^(c · r_t)                   # Gated recurrent weight
h_t = a_t ⊙ h_{t-1} + √(1-a_t²) ⊙ (i_t ⊙ x_t)  # Update
```
- **WHAT it does:** Updates a fixed-size hidden state at each time step
- **WHY the gates?**
  - **Input gate i_t:** Controls how much of the new input to let in (like LSTM)
  - **Recurrence gate r_t:** Controls how fast information decays — **crucially, biased toward retention**
  - When r_t ≈ 0: behaves like standard LRU (normal decay + input)
  - When r_t ≈ 1: a_t → a^c ≈ 1, so h_t ≈ h_{t-1} (**ignores input, preserves memory**)
- **WHY diagonal?** All operations are element-wise → efficient on hardware, easy to parallelize via linear scan
- **Key design choice:** a = σ(Λ) ensures 0 ≤ a ≤ 1 (stability); initialized so a^c ∈ [0.9, 0.999]

### Step 5: Gated MLP Block
- **WHAT:** Two branches from input → linear layers expanding to 3D → one branch gets GeLU → multiply together → linear projection back to D
- **WHY:** Provides channel mixing (non-linear transformation), gating improves over standard MLP
- **CONNECTS TO:** Output added to input via skip connection

### Step 6: Final Output
- **WHAT:** RMSNorm → linear layer (weight-tied with embedding) → softmax
- **WHY:** Produces next-token probability distribution

---

## 📊 5. THE PROOF (Results & Experiments)

### Benchmarks Used:
- **Downstream tasks:** MMLU, HellaSwag, PIQA, WinoGrande, ARC-E, ARC-C
- **Scaling curves:** 100M to 14B parameters on MassiveText dataset
- **Long context:** Books and arXiv evaluation sets
- **Synthetic tasks:** Selective Copying, Induction Heads, Phonebook Lookup

### Key Results (specific numbers):

| Comparison | Result |
|---|---|
| **Griffin-7B vs Llama-2 7B** | Griffin avg 65.8% vs Llama-2 avg 65.3% — **matches with 7× fewer training tokens** (300B vs 2T) |
| **Griffin-14B vs Llama-2 13B** | Griffin avg 69.5% vs Llama-2 avg 69.3% — **matches with 7× fewer tokens** |
| **Hawk-3B vs Mamba-3B** | Hawk avg 59.5% vs Mamba avg 58.5% — **beats Mamba with 2× fewer tokens** (300B vs 600B) |
| **Inference throughput (1B, 4096 tokens)** | Hawk: **14.8×** faster, Griffin: **6.6×** faster than MQA Transformer |
| **Sequence extrapolation** | Griffin extrapolates to **4× longer** sequences than training length |

### Most impressive result in plain English:
**Griffin-14B matches Llama-2 13B performance while being trained on only 300B tokens vs Llama-2's 2 trillion tokens — nearly 7× more data-efficient.**

### Failure cases / limitations admitted:
- **Pre-trained Hawk/Griffin struggle at phonebook lookup** (exact retrieval from long context) without fine-tuning
- Hawk (pure RNN) is **slower to learn copying tasks** than Transformers
- Griffin's retrieval degrades when context exceeds the local attention window
- Training speed advantage over Transformers **diminishes at larger model widths** (linear layers dominate)
- Different datasets and hyperparameter tuning strategies make direct comparison with Mamba/Llama-2 imperfect

---

## 🧩 6. KEY TERMS GLOSSARY

**RNN (Recurrent Neural Network)** → A neural network that processes sequences one step at a time, maintaining a hidden "memory" state

**Transformer** → The dominant architecture for LLMs that uses attention to let every token look at every other token

**MQA (Multi-Query Attention)** → A faster version of attention where all query heads share a single key/value head, reducing cache size

**MHA (Multi-Head Attention)** → Standard attention with separate key/value heads per query head (slower but more expressive)

**Local/Sliding-Window Attention** → Attention where each token only looks at a fixed number of nearby tokens, not the entire sequence

**RG-LRU (Real-Gated Linear Recurrent Unit)** → The paper's novel recurrent layer: a diagonal linear recurrence with input and recurrence gates

**LRU (Linear Recurrent Unit)** → A simplified linear RNN from prior work (Orvieto et al., 2023) without gating

**SSM (State Space Model)** → A family of models (S4, Mamba, etc.) that use continuous-time linear systems discretized for sequence modeling

**KV Cache** → Stored key/value vectors from previous tokens in attention, enabling efficient autoregressive decoding

**RoPE (Rotary Position Embedding)** → A relative positional encoding that encodes position via rotation of query/key vectors

**RMSNorm** → A normalization layer that normalizes by root-mean-square (simpler than LayerNorm)

**GeLU** → Gaussian Error Linear Unit — a smooth activation function commonly used in Transformers

**Diagonal recurrence** → A recurrence where the weight matrix is diagonal (element-wise), not a full matrix — much cheaper to compute

**Linear scan** → Processing a recurrence sequentially step by step (as opposed to parallel scan)

**Associative scan (parallel prefix sum)** → A parallel algorithm for computing recurrences using tree-like reductions

**Power-law scaling** → The observation that model performance improves as a predictable power function of compute

**FLOPs-to-byte ratio** → How many arithmetic operations per byte of memory transfer — determines whether code is compute-bound or memory-bound

**Megatron sharding** → A model parallelism strategy that splits weight matrices across devices with minimal communication

**ZeRO parallelism** → Distributing optimizer states and parameters across devices to save memory

**Pallas** → A JAX-based framework for writing custom TPU kernels

---

## 🔗 7. HOW IT CONNECTS

### Intellectual Family Tree:
```
LSTMs/GRUs (1997/2014)
    ↓ (gating idea)
S4 (2021) ← HiPPO initialization
    ↓
S4D/S5 (2022) ← diagonal simplification
    ↓
LRU (Orvieto 2023) ← simplified further, no HiPPO needed
    ↓
RG-LRU (this paper) ← adds novel gating to LRU
    
H3 (2022) ← Conv1D + SSM architecture
    ↓
Mamba (2023) ← input-dependent selection + hardware-efficient
    ↓
Griffin (this paper) ← novel gating + local attention hybrid

Longformer (2020) ← local/sliding window attention idea
    ↓
Mistral (2023) ← sliding window in production
    ↓
Griffin (this paper) ← combines with recurrence
```

### Who would use this:
- **LLM developers** wanting faster inference with comparable quality
- **Edge/mobile deployment** teams needing lower latency and memory
- **Long-document processing** applications (legal, medical, code)
- **High-throughput serving** scenarios (RLHF, code generation scoring)

### Future work enabled:
- Scaling Griffin beyond 14B parameters
- Exploring the hybrid ratio (how many recurrent vs attention layers)
- Applying to multimodal models (vision, audio)
- Better retrieval mechanisms for pure recurrent models
- Hardware co-design for diagonal RNN operations

---

## ⚖️ 8. CRITICAL ANALYSIS

### Hidden assumptions:
- **Training data quality:** Models trained on MassiveText (proprietary Google dataset) — results may not transfer to other data distributions
- **The comparison with Llama-2 and Mamba is not apples-to-apples:** Different data, different hyperparameter budgets, different tokenizers. The "7× fewer tokens" claim, while impressive, could partially be explained by data quality
- **Assumes diagonal recurrence is sufficient:** This is a strong structural constraint that may limit expressivity for some tasks

### Weaknesses the authors DON'T emphasize:
- **No evaluation on popular long-context benchmarks** like SCROLLS, LongBench, or Needle-in-a-Haystack beyond simple phonebook lookup
- **Local attention window size of 1024 is fixed** — they show (Appendix E) the gap with global attention grows at longer sequences, but don't deeply address this
- **No fine-tuning experiments** — they show pre-trained models struggle at retrieval but don't show if fine-tuning fixes this
- **TPU-specific optimizations** — the Pallas kernel and efficiency claims are on TPU-v3; GPU results may differ significantly
- **The 2:1 recurrent-to-attention ratio is presented without thorough ablation** of alternative ratios

### Is the evaluation fair?
- **Mostly yes** — they include their own MQA Transformer baseline on the same data, which is the fairest comparison
- **But** the downstream benchmarks are relatively narrow (6 tasks), and the model sizes where external comparisons are made (Mamba-3B, Llama-2 7B/13B) use different training setups
- Missing evaluation on **generation quality, instruction following, chat ability**

### Would this work in the real world at scale?
- **Likely yes for inference-constrained deployments** — the throughput gains (up to 14.8×) are massive
- **Training efficiency is roughly comparable** to Transformers, so no penalty there
- **The main concern:** retrieval from long context is weaker than Transformers, which matters for RAG-based applications
- **Scaling to 14B is demonstrated** — this is a serious scale, not just toy experiments

---

## 📝 9. MEMORY ANCHORS

### Memorable metaphor:
**Griffin is like a skilled note-taker at a lecture**: it keeps a compressed running summary of everything said (RG-LRU recurrence) while also having the last page of detailed notes visible (local attention). A Transformer, by contrast, is like a student who re-reads *all* their notes from the beginning every time they want to write the next word — thorough but increasingly slow.

### 3 bullets that capture 80% of the paper:
- 📌 **Griffin mixes gated linear recurrences (RG-LRU) with local sliding-window attention** — recurrence handles long-range, attention handles short-range precision
- 📌 **Griffin-14B matches Llama-2 13B quality with 7× less training data**, and achieves **up to 14.8× higher inference throughput** than Transformers
- 📌 **The RG-LRU is a novel gated diagonal RNN** whose recurrence gate is biased toward retaining history (unlike GRU/Mamba which can fully forget), enabling "super-exponential memory"

### One question to test understanding:
**"Why does Griffin use local attention instead of global attention in its attention layers, and what trade-off does this create?"**
*(Answer: Local attention keeps the KV cache bounded regardless of sequence length, preserving the efficiency benefit of the recurrent layers. The trade-off is that exact retrieval degrades when the relevant information falls outside the attention window, as shown in the phonebook lookup task.)*

---

## 🗺️ 10. VISUAL MENTAL MAP

```
PROBLEM                          METHOD                              RESULT
═══════                          ══════                              ══════

Transformers are slow     ──►  Design RG-LRU layer:          ──►  Quality:
on long sequences              • Diagonal linear recurrence        • Matches Llama-2 (7x fewer tokens)
(quadratic attention,          • Novel recurrence gate             • Beats Mamba (2x fewer tokens)
growing KV cache)              • Input gate                        • Power-law scaling up to 14B
                               • Biased toward retention
       │                              │                              Speed:
       │                              ▼                              • Up to 14.8x inference throughput
       │                                                             • Lower latency on long sequences
Old RNNs can't match     ──►  Build two architectures:       ──►  • Comparable training speed
Transformer quality            │                                    
at scale                       ├─► Hawk: Pure RNN                Long Context:
                               │   (MLP + Recurrent blocks)       • Extrapolates to 4x training length
       │                       │                                  • Can learn copy/retrieval tasks
       │                       └─► Griffin: Hybrid            ──►  • But pre-trained models still
       │                           (MLP + Recurrent + Local MQA)    struggle at exact retrieval
       │                           2 recurrent : 1 attention        without fine-tuning
       │                              │
       │                              ▼
SSMs struggle with        ──►  Engineering innovations:
retrieval and scaling          • Custom Pallas kernel (3x speedup)
                               • Megatron-style sharding
                               • Block-diagonal gate weights
                               • ZeRO parallelism
```

---

## 🛠️ 11. IMPLEMENTATION SKETCH

### Pseudocode of RG-LRU + Recurrent Block (~20 lines):

```python
# === RG-LRU Layer ===
def rg_lru(x_seq, W_a, b_a, W_x, b_x, Lambda, c=8):
    """x_seq: [T, D_RNN], returns y_seq: [T, D_RNN]"""
    a = sigmoid(Lambda)                    # base recurrent weight
    h = zeros(D_RNN)                       # initial hidden state
    outputs = []
    for t in range(T):
        r_t = sigmoid(W_a @ x_seq[t] + b_a)     # recurrence gate
        i_t = sigmoid(W_x @ x_seq[t] + b_x)     # input gate
        log_a_t = -c * softplus(-Lambda) * r_t   # log-space for stability
        a_t = exp(log_a_t)                        # gated recurrent weight
        h = a_t * h + sqrt(1 - a_t**2) * (i_t * x_seq[t])
        outputs.append(h)
    return stack(outputs)

# === Recurrent Block ===
def recurrent_block(x, W1, W2, W_out, rg_lru_params):
    """x: [T, D], returns [T, D]"""
    branch1 = x @ W1                      # [T, D_RNN]
    branch1 = conv1d(branch1, kernel=4)   # temporal conv
    branch1 = rg_lru(branch1, **rg_lru_params)
    
    branch2 = x @ W2                      # [T, D_RNN]
    branch2 = gelu(branch2)
    
    merged = branch1 * branch2            # element-wise gating
    return merged @ W_out                  # [T, D]
```

### Frameworks/Libraries needed:
- **JAX + Pallas** (for custom TPU kernels, as used in paper)
- OR **PyTorch** with custom CUDA kernels (for GPU implementation)
- **Flax/Haiku** for model definition
- Standard: NumPy, tokenizer (SentencePiece)

### Estimated compute to reproduce:
- **Full scaling study (100M→14B):** Thousands of TPU-v3 hours (likely **$500K-$1M+ in compute**)
- **Single 7B model (300B tokens):** ~500-1000 TPU-v3 hours (~**$50K-100K**)
- **Small ablation (400M model):** ~50-100 TPU-v3 hours (~**$5K-10K**)
- **Synthetic task experiments (250K params):** A few GPU hours (~**$10-50**)
