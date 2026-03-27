

# RecurrentGemma: Moving Past Transformers for Efficient Open Language Models

---

## 🎯 1. THE ONE-LINER
Google built a language model that's **just as smart as their Transformer-based model but uses way less memory**, so it can generate really long texts without slowing down or running out of space.

---

## 🤔 2. THE PROBLEM (Why does this paper exist?)

- **The problem:** Transformer-based language models (like GPT, Gemma) need to store a growing "memory notebook" (KV cache) for every word they've seen. The longer the text, the bigger the notebook, the more memory consumed, and the slower the generation.
- **Why should anyone care?** Imagine you're writing a very long essay. A Transformer is like a student who has to re-read *every previous page* before writing the next sentence. The longer the essay, the slower and more exhausting it gets. RecurrentGemma is like a student who keeps a **compact summary note** instead — always the same size, no matter how long the essay.
- **Limitations of previous approaches:**
  - **Full Transformers (Gemma/GPT):** KV cache grows linearly with sequence length → memory explodes on long sequences
  - **Local attention only:** Reduces cache but **hurts performance** because the model loses global context
  - **Pure recurrent models (old RNNs):** Historically underperformed Transformers on language quality

---

## 💡 3. THE KEY IDEA (The "Aha!" moment)

- **Core insight:** You **don't need global attention** to get great language modeling. Instead, mix two cheaper ingredients: **(1) linear recurrences** (a compact rolling summary of the past) + **(2) local attention** (pay close attention only to nearby words). This combo gives you Transformer-level quality with a **fixed-size memory footprint**.

- **Everyday analogy:** Think of reading a 500-page novel:
  - A **Transformer** takes detailed notes on every single page and carries all 500 pages of notes around (gets heavy!)
  - **RecurrentGemma** keeps a **1-page rolling summary** of the story so far, plus **reads the last few pages carefully**. Much lighter backpack, same comprehension.

- **The architecture (Griffin) in a nutshell:**
```
┌─────────────────────────────────┐
│         Griffin Block           │
│                                 │
│  ┌───────────┐ ┌──────────────┐ │
│  │  Linear   │ │    Local     │ │
│  │ Recurrence│ │  Attention   │ │
│  │ (RG-LRU)  │ │ (window=2K)  │ │
│  └─────┬─────┘ └──────┬───────┘ │
│        └──────┬────────┘         │
│               ▼                  │
│           MLP Layer              │
│               ▼                  │
│         Next Block               │
└─────────────────────────────────┘
         × 26 (2B) or × 38 (9B)
```

---

## 🏗️ 4. HOW IT WORKS (The Method - Layer by Layer)

**Step 1: Tokenize the input**
- **WHAT:** Text is split into tokens using SentencePiece (vocab size = 256K)
- **WHY:** Models operate on numbers, not raw text
- **NEXT:** Tokens get embedded into vectors

**Step 2: Embed and scale**
- **WHAT:** Each token becomes a vector; **multiplied by √(model_width)**
- **WHY:** Scaling stabilizes training (borrowed from Gemma). Input/output embeddings are tied (shared weights) but the scaling factor only applies to the input side
- **NEXT:** These vectors enter the Griffin blocks

**Step 3: Process through Griffin blocks (the core)**
- **WHAT:** Each block alternates between:
  - **Linear Recurrence (RG-LRU):** Updates a fixed-size hidden state — like a running average that compresses all past context into a compact vector. No weight decay applied to these params; gradient clipping for stability.
  - **Local Attention (window = 2048 tokens):** Standard attention but only over the **last 2K tokens**, capturing fine-grained local dependencies
- **WHY:** Recurrence gives **long-range memory** cheaply; local attention gives **precise short-range** understanding
- **NEXT:** Output goes through an MLP (feed-forward layer with 3× expansion)

**Step 4: Stack many blocks**
- **WHAT:** 26 blocks for 2B model, 38 blocks for 9B model
- **WHY:** Depth = more abstract representations
- **NEXT:** Final representation goes to output layer

**Step 5: Pre-train on 2T tokens**
- **WHAT:** Trained on English web docs, math, and code (same data as Gemma)
- **WHY:** Learn general language understanding
- **NEXT:** Optionally fine-tuned with instruction tuning + RLHF

**Step 6: Instruction tuning + RLHF**
- **WHAT:** Fine-tune with supervised dialogue data + reinforcement learning from human feedback
- **WHY:** Makes the model follow instructions and have safe conversations
- **FORMAT:** Uses special tokens (`<start_of_turn>`, `user`, `model`, etc.)

---

## 📊 5. THE PROOF (Results & Experiments)

### Benchmarks tested:
MMLU, HellaSwag, PIQA, SIQA, BoolQ, Winogrande, ARC, TriviaQA, NQ, HumanEval, MBPP, GSM8K, MATH, AGIEval, BBH (18 benchmarks total)

### Key numbers:
| | Gemma-7B | RecurrentGemma-9B | Tokens trained on |
|---|---|---|---|
| **Average** | **56.9** | **56.1** | 6T vs **2T** |
| BBH | 55.1 | **55.2** | — |
| TriviaQA | 63.4 | **70.5** | — |

### Most impressive result in plain English:
> **RecurrentGemma-9B matches Gemma-7B despite training on only 1/3 the data**, and achieves **up to 100× faster token generation** on long sequences (9B model).

### Inference speed:
- RecurrentGemma's throughput **stays constant** as sequence length grows
- Gemma-7B's throughput **drops dramatically** (limited by growing KV cache)
- RecurrentGemma-9B: up to **two orders of magnitude faster** sampling than Gemma-7B

### Human evaluation:
- RecurrentGemma-9B IT beats Mistral 7B Instruct with **59.3% win rate** on instruction following
- Both sizes beat Mistral on **safety** (~60% win rate)

### Limitations admitted:
- MMLU scores are **noticeably lower** (60.5 vs 64.3 for 9B; 38.4 vs 42.3 for 2B)
- Throughput numbers are TPU-specific (Flax/Pallas kernel); **GPU performance will be worse**
- Safety testing "cannot cover all possible use cases"

---

## 🧩 6. KEY TERMS GLOSSARY

- **Transformer** → The standard architecture behind GPT/Gemma that uses attention to relate all words to each other
- **KV Cache** → Memory storage that Transformers need during generation; grows with each new token
- **Griffin** → The hybrid architecture combining linear recurrences + local attention (from De et al., 2024)
- **Linear Recurrence** → A way to update a hidden state using simple multiplication and addition (not full attention), runs in O(1) memory per step
- **RG-LRU (Real-Gated Linear Recurrent Unit)** → Griffin's specific recurrence mechanism with gating
- **Local Attention** → Attention that only looks at nearby tokens (last 2048) instead of the entire sequence
- **Global Attention** → Standard attention looking at ALL previous tokens (what Transformers normally do)
- **RLHF** → Reinforcement Learning from Human Feedback; teaching models to give responses humans prefer
- **SFT** → Supervised Fine-Tuning; training on curated instruction-response pairs
- **Instruction Tuning (IT)** → Fine-tuning a model to follow user instructions
- **SentencePiece** → A tokenizer that splits text into subword units
- **Pallas Kernel** → A custom low-level TPU operation for efficient computation
- **Multi-Head Attention (MHA)** → Each attention layer has multiple independent "attention perspectives" (larger KV cache)
- **Multi-Query Attention (MQA)** → A variant sharing keys/values across heads (smaller KV cache)
- **Tied Embeddings** → Using the same weight matrix for input tokens and output predictions

---

## 🔗 7. HOW IT CONNECTS

### Intellectual family tree:
```
Structured State Spaces (S4, Gu et al. 2021)
         │
         ▼
Linear Recurrent Units (Orvieto et al. 2023)
         │
         ├──── Longformer / Local Attention (Beltagy et al. 2020)
         │              │
         ▼              ▼
    Griffin Architecture (De et al. 2024) ◄── Gemini insights
         │
         ▼
   RecurrentGemma (THIS PAPER)
```

### Related recent papers:
1. **Mamba (Gu & Dao, 2023)** — Another linear recurrence model challenging Transformers; RecurrentGemma's Griffin is a competitor/alternative approach
2. **RWKV (Peng et al., 2023)** — RNN-based language model also targeting efficient inference; Griffin differs by mixing recurrence with local attention

### Who would use this:
- **Edge/mobile deployment** teams needing LLMs on limited hardware
- **Long-document applications** (summarization, chat with long context)
- **High-throughput serving** where cost-per-token matters

### Future work enabled:
- Scaling Griffin architecture to larger models (70B+?)
- Applying to multimodal settings (like Gemini)
- Better hybrid recurrence + attention recipes

---

## ⚖️ 8. CRITICAL ANALYSIS

### Hidden assumptions:
- Assumes **inference is memory-bound** (true for autoregressive generation, less so for prefill)
- Assumes local attention window of 2K is sufficient for most tasks — **no ablation on window size** is given in this paper
- Relies heavily on the Griffin paper (De et al., 2024) for architecture details — this paper is more of a **model release report** than a full research contribution

### Weaknesses authors DON'T mention:
- **No perplexity numbers** — only downstream task accuracy is reported
- **No long-context evaluation** — despite the efficiency claim, they don't test on long-context benchmarks (e.g., SCROLLS, LongBench)
- The fixed-size state means **information compression is lossy** — for tasks requiring precise recall of distant tokens, this could hurt (hinted by lower MMLU scores)
- **No comparison with Mamba** or other state-space models — only compared against Transformers
- The 256K vocabulary inflates parameter count significantly (embeddings are ~25-30% of total params)

### Is the evaluation fair?
- **Partially.** The token-count disparity (2T vs 3T/6T) is noted, which is commendable. But it makes direct comparison ambiguous — is the architecture better or just undertrained?
- Safety benchmarks are included, which is good
- Human evaluation is only against Mistral 7B, not against Gemma itself

### Would this work at scale in the real world?
- **Yes for inference cost savings** — the fixed state is a genuine advantage
- **Uncertain for quality-critical applications** — the MMLU gap (3-4 points) may matter
- **GPU ecosystem support is weaker** — optimized primarily for TPUs with Pallas kernels

---

## 📝 9. MEMORY ANCHORS

### Memorable metaphor:
> RecurrentGemma is like a **speed reader with a Post-it note** instead of a **librarian carrying an ever-growing filing cabinet**. The Post-it captures the gist; local attention handles the fine print of nearby pages.

### 3 bullets that capture 80% of the paper:
- **RecurrentGemma uses the Griffin architecture** (linear recurrence + local attention) to replace global attention in Transformers, achieving a **fixed-size state** regardless of sequence length
- It **matches Gemma's quality with 1/3 the training data** and achieves **up to 100× faster generation** on long sequences
- Released as open models (2B and 9B) with pre-trained and instruction-tuned variants

### Comprehension test question:
> *Why does RecurrentGemma's inference throughput remain constant as sequence length increases, while a Transformer's throughput decreases?*

**Answer:** Because RecurrentGemma's state size is fixed (bounded by the local attention window of 2K), it doesn't need to load an ever-growing KV cache into memory. Transformers must load their full KV cache, which grows linearly with sequence length, making inference increasingly memory-bound.

---

## 🗺️ 10. VISUAL MENTAL MAP

```
PROBLEM                          METHOD                              RESULT
───────                          ──────                              ──────

Transformers have                Griffin Architecture:               Quality:
growing KV cache    ──────►     ┌──────────────────┐    ──────►     ~Same as Gemma
  → slow on long                │ Linear Recurrence │                (56.1 vs 56.9 avg)
    sequences                   │    (RG-LRU)       │
  → memory hungry               │        +          │               Speed:
                                │  Local Attention   │    ──────►   Up to 100× faster
                                │   (2K window)      │               sampling on long seq
Previous fix:                   │        +          │
  local-attn-only               │      MLP          │               Efficiency:
  → hurts quality               └──────────────────┘    ──────►    Trained on only 2T
                                 × 26 or 38 layers                  tokens (vs 3-6T)
                                        │
                                        ▼                           Open release:
                                 Pre-train (2T tok)                 2B & 9B models
                                        │                           PT & IT variants
                                        ▼                           JAX + PyTorch code
                                  SFT + RLHF
```

---

## 🛠️ 11. IMPLEMENTATION SKETCH

### Pseudocode (simplified Griffin forward pass):
```python
def griffin_forward(tokens, params):
    # Step 1: Embed + scale
    x = embed(tokens) * sqrt(model_width)
    
    # Step 2: Process through Griffin blocks
    state = init_fixed_state()  # Fixed size!
    for block in griffin_blocks:
        # Branch 1: Linear recurrence (RG-LRU)
        r_out, state = rg_lru(x, state, block.recur_params)
        
        # Branch 2: Local attention (window = 2048)
        a_out = local_attention(x, window=2048, block.attn_params)
        
        # Combine + MLP
        x = combine(r_out, a_out)
        x = mlp(x, expansion=3)
    
    # Step 3: Output logits (tied embeddings, no scaling)
    logits = x @ embed.weight.T
    return logits
```

### Frameworks needed:
- **JAX + Flax** (primary, with Pallas kernel for TPU recurrence)
- **PyTorch** (reference implementation available)
- TPU v4 or v5e for best performance; GPUs supported but slower

### Estimated compute to reproduce:
- **Not explicitly stated**, but training on 2T tokens at 2B-9B param scale likely requires:
  - ~Thousands of TPU-hours (ballpark: 2B model ~few thousand TPUv4 hours; 9B model significantly more)
  - Comparable to Gemma pre-training but with fewer tokens
  - **Not feasible for individual researchers** — this is a Google-scale effort
