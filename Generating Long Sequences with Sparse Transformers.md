

# Generating Long Sequences with Sparse Transformers (Child et al., 2019, OpenAI)

---

## 🎯 1. THE ONE-LINER

**Instead of making every word/pixel pay attention to every other word/pixel (which gets impossibly slow for long sequences), this paper uses a clever shortcut pattern so the model only looks at a smart subset — making it fast enough to handle sequences of tens of thousands or even a million elements.**

---

## 🤔 2. THE PROBLEM (Why does this paper exist?)

- **The core problem:** Standard Transformers compute attention between *every pair* of elements in a sequence. If you have `n` elements, that's `n²` comparisons. For a 64×64 image (12,288 bytes) or a long audio clip (65,536+ samples), this is **computationally and memory-wise infeasible**.

- **Why should anyone care?**
  - Imagine you're a teacher grading essays. With 30 students, you can compare every pair of essays to check for plagiarism (30×30 = 900 comparisons). But with 10,000 students? That's 100 million comparisons — impossible. You need a **smarter way to compare** that doesn't require checking every single pair.
  - The same problem exists for AI: generating high-quality images pixel-by-pixel, producing long audio, or writing long text requires the model to remember and connect information across very long distances.

- **Limitations of previous approaches:**
  - **CNNs (PixelCNN, WaveNet):** Need many layers stacked to "see" far away — like passing a message down a long telephone chain. Slow to propagate distant information.
  - **Standard Transformers:** Can see everything at once (global receptive field in 1 layer!) but the **O(n²) cost** makes them impractical beyond ~1,000-2,000 tokens.
  - **Transformer-XL:** Uses memory/caching tricks for text but is domain-specific and doesn't naturally handle images or audio.
  - **Image Transformer (Parmar et al.):** Uses local blocks of attention, but loses the ability to model long-range dependencies easily.

---

## 💡 3. THE KEY IDEA (The "Aha!" moment)

### Core insight:
**You don't need every element to directly attend to every other element. If you split attention into two complementary sparse patterns, any element can still reach any other element in just 2 steps — like a layover flight instead of a direct flight.**

### Everyday analogy:
Think of a **postal system**. In a naive system (full attention), every house sends a letter directly to every other house — absurdly expensive. Instead, the Sparse Transformer works like a **hub-and-spoke system**:
- **Step 1 (local):** Each house talks to its nearby neighbors on the same street.
- **Step 2 (strided/summary):** Each street has a "hub" post office that collects summaries and communicates with all other hub post offices.

Any house can reach any other house in at most 2 hops, but the total number of connections drops from n² to ~n√n.

### The two sparse patterns:

```
STRIDED ATTENTION (good for images/audio):
Head 1: Attend to the previous √n local positions
Head 2: Attend to every √n-th position (strided)

FIXED ATTENTION (good for text):
Head 1: Attend to positions in same block of √n
Head 2: Attend to a small set of "summary" positions
         that aggregate info from each block
```

### ASCII visualization (for a 16-element sequence, stride=4):

```
Full Attention (standard):        Strided Sparse Attention:
pos attends to:                   Head 1 (local):  Head 2 (strided):
8 → [1,2,3,4,5,6,7,8]           8 → [5,6,7,8]   8 → [4,8]
                                  
Together: 8 can reach ALL positions via 2-hop paths
Cost: n² = 64                    Cost: 2 × n×√n = 2 × 16×4 = 128
                                  (But scales as O(n√n) vs O(n²))
```

---

## 🏗️ 4. HOW IT WORKS (The Method - Layer by Layer)

### Step 1: Tokenize everything as raw bytes
- **WHAT:** Images, text, and audio are all converted to sequences of discrete tokens (usually raw bytes, 0-255).
- **WHY:** This makes the architecture **universal** — one model for any data type. No domain-specific preprocessing.
- **HOW it connects:** These tokens become the input sequence to the Transformer.

### Step 2: Embed with positional information
- **WHAT:** Each token gets an embedding (learned vector) + positional embeddings that encode **where** it is (e.g., row, column, channel for images).
- **WHY:** Transformers are position-agnostic by default; embeddings tell the model about spatial structure.
- **Formula:** `embed = token_embedding + Σ positional_embeddings`

### Step 3: Apply sparse factorized self-attention (the key innovation)
- **WHAT:** Instead of full n×n attention, use **two types of attention heads** that each attend to only ~√n positions:
  - **Strided pattern:** Head 1 = local window, Head 2 = every-√n-th position
  - **Fixed pattern:** Head 1 = same block, Head 2 = designated "summary" positions
- **WHY:** Reduces computation from O(n²) to O(n√n). Still maintains **full connectivity** across 2 steps.
- **HOW it connects:** These heads can be interleaved across layers, merged, or run as multi-head attention.

### Step 4: Pre-activation residual blocks for depth
- **WHAT:** Use **pre-norm residual blocks** (LayerNorm → Attention → Add → LayerNorm → FFN → Add) instead of post-norm.
- **WHY:** Standard Transformers are hard to train with many layers. Pre-activation + **scaled initialization** (weights scaled by `1/√(2N)` where N = number of layers) keeps gradients stable.
- **HOW it connects:** Enables networks with **128+ layers**, which is critical for modeling complex distributions.

### Step 5: Recompute attention during backward pass (gradient checkpointing)
- **WHAT:** Don't store attention matrices in memory. Recompute them during backpropagation.
- **WHY:** Attention matrices for long sequences are **memory hogs**. This trades compute for memory, enabling sequences of 16,384+ on a single GPU.
- **HOW it connects:** Combined with sparse attention, allows training on sequences previously impossible.

### Step 6: Custom GPU kernels for block-sparse ops
- **WHAT:** Implemented custom CUDA kernels that slice out relevant sub-blocks and compute attention in blocks.
- **WHY:** Naive sparse attention doesn't automatically translate to speed on GPUs (GPUs like dense operations). Custom kernels make sparsity actually fast.
- **HOW it connects:** Makes the theoretical speedup a **practical reality**.

### Step 7: Mixed-precision training
- **WHAT:** Weights in float32, activations/gradients in float16 (half-precision).
- **WHY:** 2× memory reduction, faster Tensor Core utilization on V100 GPUs.
- **HOW it connects:** Further enables scaling to longer sequences and deeper models.

### Step 8: Output via softmax over vocabulary
- **WHAT:** Final layer norm → linear projection → softmax to predict next token probability.
- **WHY:** Standard autoregressive generation — predict each next byte/token.
- **HOW it connects:** Generate samples by sampling from this distribution autoregressively.

---

## 📊 5. THE PROOF (Results & Experiments)

### Benchmarks tested:

| Dataset | Type | Sequence Length | Previous SOTA | **Sparse Transformer** | Improvement |
|---------|------|----------------|---------------|----------------------|-------------|
| **CIFAR-10** | Images | 3,072 | 2.85 bpd (PixelSNAIL) | **2.80 bpd** | -0.05 bpd |
| **Enwik8** | Text | 12,288 | 1.03 bpd (Transformer-XL 88M) | **0.99 bpd** | -0.04 bpd |
| **ImageNet 64×64** | Images | 12,288 | 3.52 bpd (SPN) | **3.44 bpd** | -0.08 bpd |
| **Classical music** | Audio | 65,536 | N/A | **1.97 bpd** | First result |

*(bpd = bits per byte/dim, lower is better)*

### Speed comparison (Table 2):
| Model | Enwik8 Loss | Time/Iter |
|-------|-------------|-----------|
| Dense Attention | 1.00 | 1.31 |
| **Sparse (Fixed)** | **0.99** | **0.55** (2.4× faster) |
| Sparse (Strided) | 1.13 | 0.35 (3.7× faster) |

### Most impressive result in plain English:
**The Sparse Transformer matched a Transformer-XL model with 2.9× more parameters (277M vs 95M) on text compression, while being dramatically faster and using much less memory. It also generated coherent 64×64 ImageNet images directly from pixels without any multi-scale tricks.**

### Bonus: Million-length sequences
- They demonstrated attention on sequences of **1,048,576 timesteps** (audio), though with only 3M parameters due to memory constraints.

### Failure cases / limitations admitted:
- **Strided attention fails on text** (no natural 2D structure) — had to use fixed patterns instead
- **Quality degrades** for very long sequences because model capacity must shrink (3M params for 1M length)
- Sample quality is "unconditional" — no class-conditional generation tested
- The factorized patterns can't learn **all** the attention patterns a full model can (e.g., data-dependent global patterns from Figure 2c)

---

## 🧩 6. KEY TERMS GLOSSARY

**Autoregressive model** → A model that generates one element at a time, where each new element depends on all previous ones (like writing a sentence word by word)

**Self-attention** → A mechanism where each element in a sequence computes how much it should "pay attention to" every other element

**Attention matrix** → An n×n grid storing how much each element attends to every other element; the thing that's too expensive to compute

**Factorized attention** → Splitting the big n×n attention into multiple smaller attention steps that together cover all connections

**Stride** → The step size between attended positions (e.g., attending to every 128th position)

**Strided attention** → One head looks locally, the other looks at regularly spaced positions (good for data with grid structure)

**Fixed attention** → One head looks within a block, the other looks at designated "summary" positions (good for text)

**Sparse connectivity** → Not every element connects to every other — only a carefully chosen subset

**O(n√n)** → The computational cost grows as n times the square root of n, much slower than n² for large n

**Pre-activation residual block** → A building block where normalization comes BEFORE the transformation (helps train deep networks)

**Gradient checkpointing / recomputation** → Instead of storing intermediate values for backprop, recompute them on the fly to save memory

**Bits per byte (bpd)** → A measure of how well a model compresses data; lower = better understanding of data patterns

**Layer Normalization** → A technique that normalizes the activations within a layer to stabilize training

**GELU** → Gaussian Error Linear Unit — a smooth activation function (like ReLU but smoother)

**Mixed-precision training** → Using lower-precision numbers (float16) for most operations to save memory and compute

**Density modeling** → Learning the probability distribution of data so you can evaluate how likely any data point is AND generate new ones

**Receptive field** → The region of input that can influence a particular output; bigger = can see farther

**Merged head** → A single attention head that attends to the union of positions from all factorized patterns

---

## 🔗 7. HOW IT CONNECTS

### Intellectual family tree:
```
Transformer (Vaswani et al., 2017)
    ├── GPT (Radford et al., 2018) — decoder-only transformers for text
    ├── Transformer-XL (Dai et al., 2018) — memory for longer context
    ├── Image Transformer (Parmar et al., 2018) — local attention for images
    └── ★ Sparse Transformer (this paper, 2019) — factorized sparse attention
            ├── Longformer (Beltagy et al., 2020) — sliding window + global tokens
            ├── BigBird (Zaheer et al., 2020) — random + window + global attention
            ├── Linformer (Wang et al., 2020) — linear attention
            └── Flash Attention (Dao et al., 2022) — IO-aware exact attention

Also builds on:
- WaveNet (van den Oord, 2016) — dilated convolutions for audio
- PixelCNN/PixelSNAIL — autoregressive image models
- ResNet pre-activation (He et al., 2016) — deep residual networks
- Gradient checkpointing (Chen et al., 2016) — memory-efficient training
```

### Who would use this:
- **Generative AI researchers** building models for images, audio, text
- **Anyone working with long sequences**: genomics, time series, video
- **Hardware-constrained practitioners** who need to model long-range dependencies on limited GPU memory

### Future work enabled:
- Directly led to ideas in **Longformer**, **BigBird**, and other efficient Transformers
- Motivated research into **learnable sparsity** (letting the model decide where to attend)
- Inspired **Flash Attention** (different approach: make full attention fast via IO optimization)
- Paved the way for **very long context windows** now standard in modern LLMs

---

## ⚖️ 8. CRITICAL ANALYSIS

### Hidden assumptions:
- **Assumes attention is naturally sparse** — justified by Figure 2's empirical observation, but this may not hold for all domains or tasks
- **Assumes 2D factorization (p=2) is sufficient** — they mention extensibility to higher p but only test p=2
- **Assumes raw byte tokenization is adequate** — modern approaches (BPE, SentencePiece) may be more efficient for text

### Weaknesses the authors DON'T mention:
- **Generation speed is still O(n²) in wall-clock time** for autoregressive sampling (each token generated sequentially, and you still need √n attention per token × n tokens)
- **No comparison on downstream tasks** — only density modeling (bits per byte). How do these sparse patterns affect representations for classification, translation, etc.?
- **The fixed attention pattern's "summary positions" are manually designed**, not learned — could a learned routing (like later mixture-of-experts attention) do better?
- **Only tested on relatively small models** (59M-300M parameters) — unclear how this scales to billions of parameters
- **No ablation on the specific choice of stride** — is √n always optimal? What about data-dependent stride?

### Is the evaluation fair?
- **Mostly yes**, but:
  - Enwik8 comparison to Transformer-XL 277M is listed as "matching" at 0.99, but the Sparse Transformer has only 95M params — this is actually **impressive** but they don't emphasize the parameter efficiency enough
  - CIFAR-10 improvements are small (2.80 vs 2.85) and within a regime where other factors (data augmentation, etc.) might matter
  - Only 1 run for ImageNet-64 (no error bars)
  - Audio has no baseline comparison

### Would this work in the real world at scale?
- **The sparse attention patterns have been largely superseded** by Flash Attention (2022), which makes full attention practical via hardware-aware algorithms
- However, **the conceptual contribution remains vital** — modern models like GPT-4 and Gemini likely use ideas from sparse attention for very long contexts
- The custom CUDA kernel requirement was a **practical barrier** for adoption at the time
- The idea of **"you only need to attend to a smart subset"** is now validated and used extensively

---

## 📝 9. MEMORY ANCHORS

### Memorable metaphor:
> **The Sparse Transformer is like an airline hub system.** Instead of flying direct between every pair of 1000 cities (500,000 routes), you route through ~30 hub airports. Any city can reach any other city in 2 flights, but the total number of routes drops from 500K to ~30K. The paper proves that AI attention can use the same trick without losing quality.

### 3 bullet points that capture 80%:
- **Standard Transformers have O(n²) attention cost** — the Sparse Transformer introduces factorized sparse patterns (local + strided/fixed) that **reduce this to O(n√n)** while maintaining full information flow in 2 hops
- **Three engineering innovations** make it work in practice: pre-activation residual blocks with scaled init (enables 128+ layers), gradient checkpointing (saves memory), and custom block-sparse GPU kernels (makes sparsity actually fast)
- **State-of-the-art on CIFAR-10, Enwik8, and ImageNet-64** using a single unified architecture for images, text, AND audio — demonstrating that the same sparse attention trick works across data modalities

### One question to test understanding:
> *"Why can't you just randomly drop attention connections to make things sparse — what specific property do the strided and fixed patterns guarantee that random sparsity might not?"*
> 
> **Answer:** The factorized patterns guarantee that **any position can reach any other position through a path of at most p+1 = 3 steps** (the "validity" criterion). Random sparsity could disconnect positions, meaning some elements might never be able to influence others regardless of network depth. The structured patterns ensure full connectivity while being sparse.

---

## 🗺️ 10. VISUAL MENTAL MAP

```
PROBLEM                          METHOD                              RESULT
┌─────────────────┐    ┌────────────────────────────────┐    ┌──────────────────┐
│ Transformers     │    │                                │    │                  │
│ have O(n²)       │    │  FACTORIZED SPARSE ATTENTION   │    │ SOTA on:         │
│ attention cost   │───>│  ┌─────────┐  ┌─────────┐     │───>│ • CIFAR-10: 2.80 │
│                  │    │  │ Head 1:  │  │ Head 2:  │     │    │ • Enwik8:  0.99  │
│ Can't handle:    │    │  │ Local    │  │ Strided/ │     │    │ • ImgNet:  3.44  │
│ • Long audio     │    │  │ window   │  │ Fixed    │     │    │                  │
│ • Full images    │    │  │ (√n pos) │  │ (√n pos) │     │    │ Sequences up to  │
│ • Long text      │    │  └────┬────┘  └────┬────┘     │    │ 1,000,000 tokens │
└─────────────────┘    │       │    Together  │          │    │                  │
                       │       └──────┬───────┘          │    │ 2-4x faster than │
                       │              │                  │    │ dense attention   │
                       │    O(n²) → O(n√n)               │    └──────────────────┘
                       │                                │
                       │  + ENGINEERING TRICKS:          │
                       │  ┌───────────────────────────┐  │
                       │  │ • Pre-norm residual blocks │  │
                       │  │   + scaled init (1/√2N)    │  │
                       │  │   → enables 128+ layers    │  │
                       │  │                           │  │
                       │  │ • Gradient checkpointing   │  │
                       │  │   → recompute attn in      │  │
                       │  │     backward pass (memory↓)│  │
                       │  │                           │  │
                       │  │ • Custom GPU kernels       │  │
                       │  │   → block-sparse ops fast  │  │
                       │  │                           │  │
                       │  │ • Mixed precision (FP16)   │  │
                       │  │   → 2x memory savings      │  │
                       │  └───────────────────────────┘  │
                       └────────────────────────────────┘

KEY INSIGHT FLOW:
┌──────────┐     ┌──────────────┐     ┌──────────────┐     ┌───────────┐
│ Observe:  │     │ Idea: Factor-│     │ Ensure: any  │     │ Result:   │
│ Full attn │────>│ ize into 2   │────>│ position can │────>│ O(n√n)    │
│ is mostly │     │ sparse heads │     │ reach any    │     │ with full │
│ sparse    │     │ each ~√n     │     │ other in 2   │     │ info flow │
│ anyway!   │     │ connections  │     │ hops         │     │           │
└──────────┘     └──────────────┘     └──────────────┘     └───────────┘
 (Figure 2)       (Figure 3b,c)       (validity crit.)     (Tables 1,2)
```

---

## 🛠️ 11. IMPLEMENTATION SKETCH

### Pseudocode (core sparse attention):

```python
def sparse_attention(X, pattern='strided', stride=128):
    n, d = X.shape
    Q = X @ W_q  # (n, d_k)
    K = X @ W_k  # (n, d_k)
    V = X @ W_v  # (n, d_v)
    
    if pattern == 'strided':
        # Head 1: local attention (previous stride positions)
        out1 = local_attention(Q, K, V, window=stride)
        
        # Head 2: strided attention (every stride-th position)
        # Reshape into (n/stride, stride, d), transpose, local attn
        Q2 = rearrange(Q, '(b s) d -> b s d', s=stride).transpose(0,1)
        K2 = rearrange(K, '(b s) d -> b s d', s=stride).transpose(0,1)
        V2 = rearrange(V, '(b s) d -> b s d', s=stride).transpose(0,1)
        out2 = local_attention(Q2, K2, V2, window=n//stride)
        out2 = rearrange(out2, 's b d -> (b s) d')  # reshape back
        
    elif pattern == 'fixed':
        # Head 1: block attention (within blocks of size stride)
        out1 = block_attention(Q, K, V, block_size=stride)
        
        # Head 2: attend to last c positions of each block
        summary_idx = get_summary_positions(n, stride, c=8)
        out2 = attend_to_positions(Q, K, V, summary_idx)
    
    # Combine (interleave across layers, or merge, or multi-head)
    output = W_p @ combine(out1, out2)
    return output

def local_attention(Q, K, V, window):
    """Causal attention within a local window."""
    # Only compute attention within window, apply causal mask
    scores = Q @ K.T / sqrt(d_k)  # only for positions within window
    scores = apply_causal_mask(scores)
    weights = softmax(scores)
    return weights @ V
```

### Frameworks/libraries needed:
- **PyTorch** (core framework)
- **Custom CUDA kernels** (the paper wrote their own — now available via `triton` or similar)
- Modern alternative: Use **Flash Attention** with block-sparse masks via `xformers` or `triton`
- `einops` for tensor rearrangement
- 8× **V100 GPUs** (or modern A100/H100)

### Estimated compute cost to reproduce:
| Experiment | GPUs | Time | Estimated cost (cloud) |
|-----------|------|------|----------------------|
| CIFAR-10 (128 layers) | 8× V100 | ~2-3 days | ~$500-750 |
| Enwik8 (30 layers) | 8× V100 | ~3-5 days | ~$750-1,250 |
| ImageNet 64×64 (48 layers) | **64× V100** | **7 days** | **~$15,000-20,000** |
| Classical music (1M length) | 8× V100 | ~days | ~$1,000+ |
