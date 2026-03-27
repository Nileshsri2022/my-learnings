

# MobileLLM: Optimizing Sub-billion Parameter Language Models for On-Device Use Cases

---

## 🎯 1. THE ONE-LINER
**This paper figures out how to build the smartest possible tiny AI brain (under 1 billion parameters) that can run directly on your phone instead of needing a giant cloud computer.**

---

## 🤔 2. THE PROBLEM (Why does this paper exist?)

- **Real-world problem:** Running big AI models like ChatGPT requires massive cloud servers. If everyone relied on cloud-based LLMs for ~5% of their day, we'd need ~100 million H100 GPUs — equivalent to 160 Meta-sized companies. That's financially and environmentally insane.
- **Why should anyone care?** Imagine if every time you wanted to use a calculator, you had to call a warehouse full of people to do the math for you, and each call cost money. That's what using cloud LLMs is like. **MobileLLM is like putting a smart calculator in your pocket** — it works offline, it's fast, it's private, and it's cheap.
- **Mobile constraints are brutal:**
  - An iPhone's DRAM is only 6 GB, and apps should use <10% of that (~600 MB)
  - A 7B model (like LLaMA-v2) drains an iPhone battery in <2 hours
  - A 7B model runs at only 3-6 tokens/second on an iPhone
- **Limitations of previous approaches:**
  - Previous sub-billion models (OPT-125M, GPT-Neo, Pythia) used **shallow, wide architectures** (e.g., 12 layers) — just copied the design of larger models
  - The dominant "scaling law" belief (Kaplan et al., 2020) said **architecture doesn't matter**, only data and parameter count do — **this paper shows that's wrong for small models**
  - Model compression (pruning, quantization) of big models still often results in models too large for phones

---

## 💡 3. THE KEY IDEA (The "Aha!" moment)

**Core insight: For small models, architecture matters A LOT — specifically, making the model DEEPER and THINNER beats the conventional wide-and-shallow approach.**

### Everyday analogy:
Think of building a **skyscraper vs. a warehouse** with the same amount of building material:
- Previous approach: Build a **short, wide warehouse** (12 layers, wide dimensions) — lots of floor space but not very tall
- MobileLLM approach: Build a **tall, skinny skyscraper** (30+ layers, narrow dimensions) — reaches much higher with the same material

The skyscraper (deep model) can "see further" and understand more abstract concepts because each floor adds another level of processing. Then, they add clever tricks like **reusing floors** (weight sharing) — imagine an elevator that visits the same floor twice, getting more value from existing infrastructure without building new floors.

### The 4 key tricks combined:
```
Baseline (12-layer, 768-dim) ──→ 42.6% accuracy
  │
  ├── 1. Better activation (SwiGLU)     → +1.3%
  ├── 2. Deep & thin (30-layer, 512-dim) → +0.9%  
  ├── 3. Share input/output embeddings   → save 10% params
  ├── 4. Grouped query attention (GQA)   → +0.4%
  │
  = MobileLLM ──→ 46.3% accuracy
  │
  └── 5. Block-wise layer sharing        → +0.7%
  │
  = MobileLLM-LS ──→ 47.0% accuracy
```

---

## 🏗️ 4. HOW IT WORKS (The Method - Layer by Layer)

### Step 1: Switch to SwiGLU Feed-Forward Network
- **WHAT:** Replace the standard FFN (FC → ReLU → FC) with SwiGLU activation
- **WHY:** SwiGLU provides a gating mechanism that lets the network learn which features to pass through, yielding +1.3% accuracy
- **HOW it connects:** This is the foundation — every subsequent improvement builds on top of SwiGLU

### Step 2: Go Deep and Thin
- **WHAT:** Instead of 12 layers × 768-dim, use **30 layers × 512-dim** (same total parameters)
- **WHY:** They trained **19 different models** varying depth/width and found deeper consistently wins. A 30-layer 125M model significantly outperforms a 12-layer 125M model. This **contradicts the scaling law** that says architecture doesn't matter
- **HOW it connects:** The deeper structure provides more sequential processing stages, better capturing abstract reasoning with the same parameter budget

### Step 3: Share Input and Output Embeddings
- **WHAT:** Reuse the input embedding matrix (token → vector) as the output layer (vector → token prediction)
- **WHY:** In a 125M model, embeddings are **>20% of all parameters** (vs. only 3.7% in LLaMA-7B). Sharing saves ~16M params (11.8%) with only 0.2% accuracy drop. The saved params are reinvested into **more layers**
- **HOW it connects:** Freed parameters allow adding 2 more layers, which actually **increases** accuracy by 0.4 points net

### Step 4: Grouped Query Attention (GQA)
- **WHAT:** Use fewer key-value heads than query heads (e.g., 9 query heads but only 3 KV-heads)
- **WHY:** Reduces redundancy in key-value heads. Saves ~10% of parameters with minimal accuracy loss. Saved params are reinvested into a **wider embedding dimension** (512 → 576), yielding +0.4% gain
- **HOW it connects:** This is the final piece of the MobileLLM baseline — combining all 4 techniques creates a strong foundation

### Step 5: Immediate Block-Wise Layer Sharing (→ MobileLLM-LS)
- **WHAT:** Each transformer block is computed **twice** in a row before moving to the next block. Effectively doubles the "logical" depth (30 → 60 layers) with **zero extra parameters**
- **WHY:** Since on-device inference is **memory-bandwidth-bound** (not compute-bound), keeping the same block in SRAM cache and computing it twice costs almost nothing in latency (only 2.6% overhead!) but significantly boosts accuracy (+0.7%)
- **HOW it connects:** This is the cherry on top — the final model MobileLLM-LS achieves state-of-the-art results

```
                    ┌─────────┐
  Input ──────────► │ Block 1 │──┐
                    └─────────┘  │ (compute again)
                    ┌─────────┐  │
                    │ Block 1 │◄─┘  ← Same weights, stays in cache!
                    └─────────┘
                    ┌─────────┐
              ────► │ Block 2 │──┐
                    └─────────┘  │
                    ┌─────────┐  │
                    │ Block 2 │◄─┘
                    └─────────┘
                         ⋮
                    ┌──────────┐
              ────► │ Block 30 │──┐
                    └──────────┘  │
                    ┌──────────┐  │
                    │ Block 30 │◄─┘
                    └──────────┘
                         │
                      Output
```

---

## 📊 5. THE PROOF (Results & Experiments)

### Benchmarks:
- **Zero-shot commonsense reasoning:** ARC-easy, ARC-challenge, BoolQ, PIQA, SIQA, HellaSwag, OBQA, WinoGrande
- **Question answering:** TriviaQA
- **Reading comprehension:** RACE
- **Chat:** AlpacaEval, MT-Bench
- **API calling:** Custom synthetic dataset

### Key Numbers:

| Comparison | Improvement |
|---|---|
| MobileLLM-125M vs previous best 125M (RWKV-169M) | **+2.7% avg accuracy** while being **26% smaller** |
| MobileLLM-350M vs previous best ~350M (RWKV-430M) | **+4.3% avg accuracy** while being **20% smaller** |
| MobileLLM-LS-125M | Comparable to most **previous 350M models** |
| MobileLLM-LS-350M on AlpacaEval | **48.2% win rate** vs GPT-3 (GPT-3's self-win rate is 50%) |
| MobileLLM-350M API calling | **65.3% intent match** — comparable to LLaMA-v2 7B (62.8%) |
| MobileLLM-1.5B vs Qwen1.5-1.8B | **+2.9% avg accuracy** despite fewer params |

### Most impressive result in plain English:
**A 350M-parameter model (MobileLLM-LS) achieves nearly the same chat quality as GPT-3, and its API calling accuracy matches LLaMA-v2 7B (a model 20× larger)**

### On-device latency:
- MobileLLM-LS adds only **2.6% execution overhead** vs MobileLLM despite doubling logical depth
- A 60-layer model without sharing would add **86% overhead**
- 125M model can run at **50 tokens/s** on an iPhone (vs 3-6 tokens/s for LLaMA-7B)

### Limitations admitted:
- Knowledge distillation from LLaMA-v2 7B **didn't help** (comparable or worse accuracy, 2.6-3.2× slower training)
- Layer sharing beyond 2× repetition shows **diminishing returns**
- Still significantly behind 7B models on Rouge scores for API calling agent responses

---

## 🧩 6. KEY TERMS GLOSSARY

- **LLM (Large Language Model)** → An AI model trained on text to predict/generate language
- **Sub-billion parameter** → A model with fewer than 1,000,000,000 learnable weights
- **DRAM** → The main working memory of a device (6-12 GB on phones)
- **SRAM** → Ultra-fast but tiny cache memory on a chip (8-32 MB)
- **SwiGLU** → A modern activation function that uses gating to let the network control information flow; better than ReLU
- **Scaling law** → The theory (Kaplan et al., 2020) that model quality depends mainly on parameter count, data, and training steps — not architecture
- **Embedding** → Converting a word/token into a numerical vector the model can process
- **Embedding sharing** → Reusing the same weight matrix for both input (word → vector) and output (vector → word prediction)
- **GQA (Grouped Query Attention)** → Using fewer key-value heads than query heads; multiple queries share the same key-value pair
- **KV-heads** → The attention heads responsible for generating keys and values in the attention mechanism
- **Feed-Forward Network (FFN)** → The MLP layer in each transformer block that processes each position independently
- **Layer/block sharing** → Using the same transformer block weights multiple times in sequence
- **Memory-bounded** → When performance is limited by data transfer speed (reading weights from memory) rather than computation speed
- **Auto-regressive inference** → Generating text one token at a time, where each new token depends on all previous tokens
- **Post-Training Quantization (PTQ)** → Reducing the precision of model weights (e.g., from 16-bit to 8-bit) after training to save memory
- **W8A8** → 8-bit weights and 8-bit activations
- **Zero-shot** → Testing a model on a task it was never explicitly trained on
- **AlpacaEval** → A benchmark where GPT-4 judges chat model responses against a GPT-3 baseline
- **MT-Bench** → A multi-turn conversation benchmark scored by GPT-4
- **ExecuTorch** → PyTorch's framework for running models on mobile/edge devices

---

## 🔗 7. HOW IT CONNECTS

### Intellectual family tree:
```
Scaling Laws (Kaplan 2020)     OPT (Zhang 2022)
    ↓ (challenged by)              ↓ (embedding sharing)
    MobileLLM ◄──────────────── GQA (Ainslie 2023, PaLM)
        ↑                          ↑
  SwiGLU (Dauphin 2017)    Weight Sharing (Shen 2022, Reid 2021)
        ↑
  LLaMA (Touvron 2023)
```

### Comparison with related papers:
1. **TinyLLaMA (Zhang et al., 2024):** Smallest variant is still >1B parameters. MobileLLM-1B outperforms TinyLLaMA-1.1B (57.3 vs 54.2 avg accuracy)
2. **MobiLlama (Thawakar et al., 2024):** Released after MobileLLM. MobileLLM-600M beats MobiLlama-800M (54.3 vs 50.7) with fewer parameters

### Who would use this:
- **Mobile app developers** building on-device AI assistants
- **IoT/edge computing** engineers deploying AI on resource-constrained hardware
- **Privacy-focused applications** where data shouldn't leave the device
- **Researchers** studying efficient model architecture design

### Future work enabled:
- Architecture search for sub-billion transformers
- More sophisticated layer-sharing patterns
- Combining with quantization for even smaller models
- On-device fine-tuning of small models

---

## ⚖️ 8. CRITICAL ANALYSIS

### Hidden assumptions:
- Training on 1T tokens — **the data quality and composition are never disclosed**, making reproduction difficult
- The claim that "architecture matters" is demonstrated within a specific training regime; with 10T tokens, width might catch up
- SRAM cache reasoning for block-wise sharing assumes a specific hardware memory hierarchy that may differ across devices

### Weaknesses NOT mentioned:
- **No perplexity numbers reported** — only downstream task accuracy, which can be noisy
- **Only English** was evaluated; multilingual performance is unknown
- **Training data not specified** in detail — hard to separate architecture gains from data quality gains
- **Limited context length** analysis — sub-billion models likely struggle with longer contexts
- **Chat evaluations use GPT-4 as judge** — known to have biases and is not necessarily calibrated for small models
- The API calling dataset is **synthetic and only 5000 samples** — hard to know if results generalize to real-world APIs

### Is the evaluation fair?
- ✅ They re-evaluated all baselines using the same HuggingFace models and evaluation code — good practice
- ✅ Extensive ablation studies (19 depth/width models, head configurations, sharing strategies)
- ⚠️ However, some baselines (OPT, GPT-Neo) were trained on different data, so not purely apple-to-apple
- ⚠️ The "close to LLaMA-v2 7B" API calling claim is cherry-picked — Rouge scores are notably lower

### Would this work at scale?
- ✅ Yes, already demonstrated on iPhone 13 with real latency numbers
- ✅ Compatible with 8-bit quantization
- ✅ Design principles scale to 600M, 1B, 1.5B
- ⚠️ Real-world on-device deployment needs more than just a model (tokenizer, runtime, memory management)

---

## 📝 9. MEMORY ANCHORS

### Memorable metaphor:
**MobileLLM is like packing for a carry-on bag instead of checking a suitcase** — you can't take everything, so you need to be extremely clever about *how* you pack (deep & thin architecture), *what* you share (embedding + layer sharing), and *what you can reuse* (GQA). The result fits in the overhead bin and arrives faster than waiting at baggage claim (cloud inference).

### 3 bullets capturing 80% of the paper:
- **Depth > Width for small models:** A 30-layer narrow model beats a 12-layer wide model at the same parameter count, contradicting scaling laws
- **Triple weight sharing** (embedding sharing + GQA + block-wise layer sharing) maximizes the utility of every parameter without increasing model size
- **Block-wise layer sharing** doubles logical depth for free by reusing weights already in SRAM cache, adding only 2.6% latency

### One question to test understanding:
*"Why does immediate block-wise weight sharing add almost no latency on mobile devices, while a model with double the unique layers would be much slower?"*

**Answer:** Because on-device inference is memory-bandwidth-bound, not compute-bound. The shared block stays in the tiny fast SRAM cache and is simply computed again (cheap), whereas unique layers would require loading new weights from slow DRAM (expensive).

---

## 🗺️ 10. VISUAL MENTAL MAP

```
PROBLEM                          METHOD                              RESULT
┌──────────────┐    ┌───────────────────────────────────┐    ┌─────────────────────┐
│ LLMs too big │    │     Architecture Design            │    │ MobileLLM-125M      │
│ for phones   │    │  ┌─────────────────────────────┐  │    │  46.3% avg (↑2.7%)  │
│              │───►│  │ 1. SwiGLU activation        │  │───►│                     │
│ 7B model:    │    │  │ 2. Deep & thin (30L, 512d)  │  │    │ MobileLLM-350M      │
│ • 2hr battery│    │  │ 3. Embedding sharing        │  │    │  51.3% avg (↑4.3%)  │
│ • 3-6 tok/s  │    │  │ 4. Grouped Query Attention  │  │    │                     │
│ • 7GB memory │    │  └─────────────────────────────┘  │    │ MobileLLM-LS-350M   │
│              │    │              ↓                     │    │  52.1% avg           │
│ Sub-billion  │    │  ┌─────────────────────────────┐  │    │  48.2% chat win rate │
│ models too   │    │  │ 5. Block-wise layer sharing  │  │    │  vs GPT-3           │
│ weak (42%)   │    │  │    (2× depth, 0 extra params)│  │    │                     │
│              │    │  └─────────────────────────────┘  │    │ On iPhone 13:        │
│ Scaling law  │    │                                   │    │  50 tok/s (125M)     │
│ says arch    │    │  KEY INSIGHT:                      │    │  2.6% latency cost   │
│ doesn't      │    │  Depth > Width for small models   │    │  for layer sharing   │
│ matter       │    │  (contradicts Kaplan et al.)       │    │                     │
└──────────────┘    └───────────────────────────────────┘    └─────────────────────┘
```

---

## 🛠️ 11. IMPLEMENTATION SKETCH

### Pseudocode (core architecture):
```python
class MobileLLM:
    def __init__(self, vocab=32k, dim=576, n_layers=30, 
                 n_heads=9, n_kv_heads=3, layer_sharing=True):
        self.embed = Embedding(vocab, dim)          # Shared with output
        self.blocks = [TransformerBlock(dim, n_heads, n_kv_heads) 
                       for _ in range(n_layers)]
        self.output = Linear(dim, vocab, weight=self.embed.weight)  # Shared!
        self.layer_sharing = layer_sharing
    
    def forward(self, tokens):
        x = self.embed(tokens)
        for block in self.blocks:
            x = block(x)                            # First pass
            if self.layer_sharing:
                x = block(x)                        # Second pass (same weights!)
        return self.output(x)

class TransformerBlock:
    def __init__(self, dim, n_heads, n_kv_heads):
        self.attn = GroupedQueryAttention(dim, n_heads, n_kv_heads)
        self.ffn = SwiGLU_FFN(dim, hidden_dim=dim*8//3)  # SwiGLU
        self.norm1 = RMSNorm(dim)
        self.norm2 = RMSNorm(dim)
    
    def forward(self, x):
        x = x + self.attn(self.norm1(x))
        x = x + self.ffn(self.norm2(x))
        return x
```

### Frameworks/Libraries needed:
- **PyTorch** (training)
- **ExecuTorch** (on-device deployment on iOS)
- **Metal Performance Shaders** (GPU acceleration on Apple devices)
- **HuggingFace Transformers** (evaluation)
- Code available at: `github.com/facebookresearch/MobileLLM`

### Estimated compute to reproduce:
- **Hardware:** 32 × A100 GPUs
- **Exploration experiments:** 120k iterations on 0.25T tokens (~29 hours for 125M)
- **Full training:** 480k iterations on 1T tokens
- **Estimated total GPU hours:** ~1,000-4,000 A100-hours for full training of one model
- **Cost estimate:** ~$2,000-$8,000 at cloud GPU rates
