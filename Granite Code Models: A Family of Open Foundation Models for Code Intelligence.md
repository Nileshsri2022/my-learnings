

# IBM Granite Code Models: A Family of Open Foundation Models for Code Intelligence

---

## 🎯 1. THE ONE-LINER
IBM built a family of AI models (from small to large) that can **write, fix, explain, and translate computer code** in 116 programming languages, and released them for free so anyone can use them.

---

## 🤔 2. THE PROBLEM (Why does this paper exist?)

- **The real-world problem:** Software developers need AI assistants that can do *more* than just write code — they also need to fix bugs, explain existing code, translate between languages, edit code, and call functions. Most existing open-source code models are **one-trick ponies** that only generate code well but fail at other tasks.

- **Why should anyone care?** Imagine you hired a chef who can only cook pasta. Great for pasta night, but useless when you need sushi, steak, or dessert. **Enterprises need a chef who can do it all** — and do it affordably (small enough to deploy) and legally (with a clear, permissive license).

- **Limitations of previous approaches:**
  - **Big generalist models** (like Mixtral-8x22B) are great at coding but **too expensive to deploy** in enterprise settings
  - **Smaller code-focused models** (StarCoder2, CodeGemma, CodeLlama) are good at code *generation* but **bad at code fixing and explanation**
  - Many "open" models have **murky data provenance** — enterprises in regulated industries can't trust them
  - **License restrictions** on many open models complicate commercial use

---

## 💡 3. THE KEY IDEA (The "Aha!" moment)

The core insight is a **"well-rounded athlete" training strategy** rather than specializing in one thing:

- **Two-phase training:** First, gorge on massive amounts of code (3-4 trillion tokens, 116 languages) to learn syntax and patterns. Then, refine on a carefully designed mix of code + natural language + math (500B tokens) to develop **reasoning ability**.

- **Everyday analogy:** Think of training a soccer player. **Phase 1** is spending years doing drills — dribbling, passing, shooting (learning raw code). **Phase 2** is playing actual matches with strategy sessions — you mix in game film, tactics, and fitness (mixing in math, natural language, reasoning). The result is a player who doesn't just kick well but *thinks* well on the field.

- **The other trick: depth upscaling** for the 34B model — instead of training a 34B model from scratch (very expensive), they take the 20B model, duplicate it, surgically remove layers from each copy, and stitch them together into an 88-layer model. Then they just continue training. It's like **cloning a building and merging the two copies** to make a taller one.

**Step-by-step of depth upscaling (34B):**
```
┌────────────┐   ┌────────────┐
│  20B Model │   │  20B Copy  │
│  52 Layers │   │  52 Layers │
└────────────┘   └────────────┘
       │                │
  Remove last 8     Remove first 8
       │                │
┌────────────┐   ┌────────────┐
│  44 Layers │   │  44 Layers │
└────────────┘   └────────────┘
       │                │
       └───── MERGE ────┘
              │
       ┌────────────┐
       │  34B Model │
       │  88 Layers │
       └────────────┘
```

---

## 🏗️ 4. HOW IT WORKS (The Method - Layer by Layer)

### Step 1: Data Collection & Cleaning
- **WHAT:** Gathered code from GitHub (public repos, issues) + public datasets (GitHub Code Clean, StarCoderdata) across **116 programming languages**
- **WHY:** Need massive, diverse, high-quality code to teach the model all programming patterns
- **HOW it connects:** Clean data → better model. Garbage in = garbage out.
- Filtering rules: remove files <25% alphabetic chars, XML files, short/long JSON/YAML, etc.

### Step 2: Deduplication (Exact + Fuzzy)
- **WHAT:** Remove identical and near-identical code files using SHA256 hashing (exact) + MinHash/LSH with Jaccard similarity threshold 0.7 (fuzzy)
- **WHY:** Duplicate data biases the model toward memorizing specific snippets rather than learning general patterns
- **HOW it connects:** Deduplicated data → more diverse training signal

### Step 3: Safety Filtering (HAP/PII/Malware)
- **WHAT:** Remove hateful/abusive/profane content, redact personal info (names, emails, passwords, IPs), scan for malware with ClamAV
- **WHY:** Enterprise trust — can't have a model that leaks passwords or generates hate speech
- **HOW it connects:** Clean, safe data → trustworthy model for enterprise use

### Step 4: Model Architecture Design
- **WHAT:** Transformer decoder-only architecture in 4 sizes: 3B, 8B, 20B, 34B
- **WHY:** Different sizes for different deployment scenarios (edge device → data center)
- Key architectural choices vary by size:
  - **3B:** Multi-Head Attention (MHA), RoPE, SwiGLU, RMSNorm, 2048 context
  - **8B:** Grouped-Query Attention (GQA, 8 KV heads), RoPE, SwiGLU, RMSNorm, 4096 context
  - **20B:** Multi-Query Attention (MQA, 1 KV head), learned embeddings, GELU, LayerNorm, 8192 context
  - **34B:** Depth-upscaled from 20B (88 layers), same architecture as 20B, 8192 context

### Step 5: Phase 1 Pretraining (Code Only)
- **WHAT:** Train on 3-4 trillion tokens of pure code across 116 languages
- **WHY:** Build deep understanding of programming syntax, patterns, and structure
- **Training objectives:** Causal Language Modeling (CLM) + Fill-in-the-Middle (FIM), weighted 50/50
- **HOW it connects:** Raw code competence → foundation for reasoning

### Step 6: Phase 2 Pretraining (Code + Language)
- **WHAT:** Continue training on 500B tokens with 80% code, 20% natural language (math, web docs, academic text)
- **WHY:** Inject reasoning and problem-solving ability that pure code alone doesn't teach
- **HOW it connects:** Reasoning ability → better code generation, fixing, explanation

### Step 7: Instruction Tuning (for Instruct variants)
- **WHAT:** Finetune base models on a mix of:
  - CommitPackFT (Git commits as instructions)
  - Math datasets (MathInstruct, MetaMathQA)
  - Code instruction datasets (Glaive, Self-OSS-Instruct, NL2SQL, API calling)
  - Language instruction datasets (HelpSteer, Platypus)
- **WHY:** Teach the model to follow natural language instructions, not just complete code
- **Trick:** Add embedding noise (NEFTune) to improve instruction following quality
- **HOW it connects:** Instruction-tuned model → practical, user-facing tool

---

## 📊 5. THE PROOF (Results & Experiments)

### Benchmarks tested on (comprehensive list of 19+ benchmarks):
| Category | Benchmarks |
|----------|-----------|
| Code Generation | HumanEvalSynthesize, MultiPL-E (18 langs), MBPP/MBPP+, DS-1000 |
| Repo-Level Code | RepoBench, CrossCodeEval |
| Code Infilling | SantaCoder-FIM |
| Code Explanation | HumanEvalExplain |
| Code Fixing | HumanEvalFix |
| Code Editing | CanItEdit |
| Code Translation | CodeLingua |
| Code Execution | CRUXEval |
| Math Reasoning | MATH, GSM8K, SAT, OCW |
| Function Calling | BFCL |
| Robustness | ReCode |

### Key results with specific numbers:

- **Granite-8B-Code-Base vs CodeGemma-7B on HumanEvalPack (avg across 3 tasks, 6 langs):** **33.2% vs 21.3%** (+12 points) despite training on fewer tokens (4.5T vs 7.5T)

- **Granite-8B-Code-Base on code explanation (HumanEvalExplain):** **26.4% avg** — beats CodeLlama-34B (17.1%) which is **4× larger**

- **Granite-8B-Code-Base on code fixing (HumanEvalFix):** **29.6% avg** — close to CodeLlama-70B (32.8%)

- **Granite-8B-Code-Base on GSM8K math:** **61.9%** vs Llama-3-8B's 49.8% (+12 points)

- **Granite-8B-Code-Base on MATH:** **21.4%** vs Llama-3-8B's 15.6% (+6 points)

- **Granite-3B-Code-Instruct surpasses CodeLlama-34B-Instruct** on HumanEvalSynthesize (39.6% vs 41.3% avg — very close, and 3B vs 34B!)

- **FIM (infilling):** Granite beats StarCoder and StarCoder2 at all sizes (76.6% avg for 8B vs 74.0% for StarCoder2-7B)

- **Function Calling:** Granite-8B-Code-Instruct beats CodeLlama-7B-Instruct by **+12% overall accuracy**

### Most impressive result in plain English:
**The 8B model can explain and fix code almost as well as models 4-9× its size**, and the tiny 3B model outperforms the 34B CodeLlama on code generation — proving that smart training matters more than brute-force size.

### Limitations they admitted:
- No single model wins on every language across all sizes (achieving uniformly high performance across all languages remains challenging)
- Not trained with repo-level file packing (could improve repo-level performance)
- The 20B model sometimes underperforms smaller models on specific tasks (e.g., CrossCodeEval TypeScript)
- Scaling from 8B→34B doesn't help FIM/infilling tasks
- CodeGemma-7B beats Granite-8B on some benchmarks (CRUXEval, ReCode)

---

## 🧩 6. KEY TERMS GLOSSARY

- **Decoder-only transformer** → An AI architecture that generates text one token at a time, reading left-to-right (like writing a sentence word by word)
- **BPE (Byte Pair Encoding)** → A method to break text into small chunks (tokens) that the model can process
- **FIM (Fill-in-the-Middle)** → Training the model to predict what goes *between* given code, not just what comes next
- **PSM/SPM** → Two ways to arrange the prefix, suffix, and middle parts in FIM training
- **CLM (Causal Language Modeling)** → Standard next-token prediction training objective
- **GQA (Grouped-Query Attention)** → An efficiency trick where multiple attention heads share the same key/value pairs (reduces memory, keeps quality)
- **MQA (Multi-Query Attention)** → Extreme version of GQA where ALL heads share one key-value pair
- **MHA (Multi-Head Attention)** → Standard attention where each head has its own key/value (most flexible but most expensive)
- **RoPE (Rotary Position Embedding)** → A way to encode position information using rotation matrices (helps model understand word order)
- **RMSNorm** → A simpler, faster version of LayerNorm that normalizes using only the root mean square
- **SwiGLU** → A combination of Swish activation + Gated Linear Unit, used in the MLP blocks
- **Depth upscaling** → A technique to create a larger model by duplicating and merging layers of a smaller one
- **HAP filtering** → Removing Hateful, Abusive, and Profane content from training data
- **PII** → Personally Identifiable Information (names, emails, passwords, etc.)
- **MinHash/LSH** → Efficient algorithms for finding near-duplicate documents at scale
- **Jaccard similarity** → A measure of how similar two sets are (intersection/union)
- **Pass@1** → The percentage of problems solved correctly on the first attempt
- **RP@1** → Robust Pass@1 — pass rate under perturbed/noisy inputs
- **FlashAttention 2** → An optimized attention implementation that reduces GPU memory usage
- **3D parallelism** → Combining tensor, pipeline, and data parallelism to distribute training across many GPUs
- **CommitPackFT** → A dataset of Git commits (before/after code changes) paired with commit messages as instructions
- **NEFTune** → Adding random noise to embeddings during instruction finetuning to improve output quality
- **Apache 2.0 license** → A permissive open-source license allowing commercial use with minimal restrictions

---

## 🔗 7. HOW IT CONNECTS

### Intellectual Family Tree:
```
StarCoder (Li et al., 2023)
    ├── Data pipeline & FIM approach → Granite inherits these
    └── StarCoder2 (Lozhkov et al., 2024) → Direct competitor

CodeLlama (Roziere et al., 2023)
    └── Showed code-specific models beat general ones → Granite validates this

OctoCoder (Muennighoff et al., 2023)
    └── Instruction tuning with CommitPack → Granite uses filtered CommitPackFT

SOLAR (Kim et al., 2024)
    └── Depth upscaling technique → Granite uses for 34B model

MiniCPM / JetMoE (Hu et al., 2024; Shen et al., 2024)
    └── Two-phase training strategy → Granite adopts this
```

### Who would use this:
- **Enterprise developers** using IBM WatsonX Code Assistant
- **Companies in regulated industries** (finance, healthcare) who need clear data provenance
- **On-device applications** needing the 3B model
- **Researchers** needing a permissively licensed code model to build on
- **DevOps teams** for code review, documentation, and modernization

### Future work this enables:
- Long-context model variants
- Python/Java specialized models
- CodeNet instruction dataset integration
- Repo-level file packing for better repository-aware generation
- Potential for code agents that autonomously handle complex tasks

---

## ⚖️ 8. CRITICAL ANALYSIS

### Hidden assumptions:
- **Assumption that permissive-license-only data is sufficient** — this restricts the training corpus significantly; models trained on all public code (regardless of license) may have seen more diverse patterns
- **HumanEval-family benchmarks are representative** — but they're relatively simple, standalone function problems; real-world enterprise code is messier
- **File-extension-based language detection is accurate** — this is a crude heuristic that can misclassify files

### Weaknesses the authors DON'T mention:
- **No comparison with GPT-4, Claude, or Gemini** — we don't know how Granite stacks up against closed-source SOTA
- **No human evaluation** — all benchmarks are automated; no developer satisfaction studies
- **The 20B and 34B models use different architectures (learned embeddings, GELU, LayerNorm) from the 3B/8B models** — this makes it hard to attribute improvements to scale vs. architecture
- **Carbon emissions of ~455 tCO2eq** are reported but not critically discussed
- **No discussion of hallucination rates** or security vulnerabilities in generated code
- **Instruct models only trained for 3 epochs** — no ablation on whether more epochs help

### Is the evaluation fair?
- ✅ They **re-run all baselines** with the same scripts (fair comparison)
- ✅ Very comprehensive benchmark suite (19+ tasks)
- ⚠️ The 8B model dominates the narrative, but results are more mixed for 3B and 20B
- ⚠️ Some benchmarks show Granite losing to CodeGemma or StarCoder2, but these are somewhat downplayed
- ⚠️ No error bars or confidence intervals reported for most results

### Would this work in the real world at scale?
- **Yes, likely** — the 3B and 8B models are practical sizes; Apache 2.0 license removes legal barriers
- **Concern:** Real-world code tasks involve much longer contexts (entire repos) — max 8192 context is limiting
- **Concern:** No RAG or agentic workflow evaluation
- The models are already deployed in **IBM WatsonX Code Assistant**, suggesting real-world viability

---

## 📝 9. MEMORY ANCHORS

### Memorable metaphor:
> **Granite Code models are like a Swiss Army knife for code** — while most code models are a sharp knife (great at cutting/generating), Granite has a screwdriver (fixing), magnifying glass (explaining), scissors (editing), and file (translating) built in. It's not always the sharpest single blade, but it's the most useful tool overall.

### 3 bullet points that capture 80% of the paper:
- **IBM trained 4 sizes (3B-34B) of code models on 116 languages using a two-phase strategy** (code-only → code+language), making them good at not just code generation but also fixing, explaining, editing, and translating code
- **Granite-8B beats models 4× its size** on code explanation/fixing tasks (12-point advantage over CodeGemma on HumanEvalPack average) thanks to smart data mixing and two-phase training
- **All models are released under Apache 2.0** with full transparency on data sourcing, targeting enterprise adoption in regulated environments

### Understanding check question:
> *"Why does Granite-8B outperform CodeGemma-7B by 12 points on HumanEvalPack despite being trained on significantly fewer tokens (4.5T vs 7.5T)?"*

**Answer:** Because of the **two-phase training strategy** and **data mixture design** — Phase 2 mixes in natural language and math data that improves reasoning, and the overall training emphasizes well-rounded capability (generation + fixing + explanation) rather than just generation. Quality of data curation matters more than raw quantity.

---

## 🗺️ 10. VISUAL MENTAL MAP

```
PROBLEM: Code LLMs are one-trick ponies (generate only) + license/trust issues
                              │
                              ▼
┌─────────────────── DATA PIPELINE ───────────────────┐
│  GitHub repos ──► Language Filter ──► Quality Filter │
│  (116 langs)      (300+ → 116)      (alpha%, XML,   │
│                                       HTML, JSON)    │
│       │                                              │
│       ▼                                              │
│  Exact Dedup ──► Fuzzy Dedup ──► HAP/PII/Malware    │
│  (SHA256)       (MinHash+LSH,   (keyword filter,     │
│                  Jaccard>0.7)    StarPII, ClamAV)    │
│       │                                              │
│       ▼                                              │
│  + Natural Language Data (StackExchange, ArXiv,      │
│    OpenWebMath, FLAN, etc.)                          │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌──────────── MODEL ARCHITECTURE ─────────────────────┐
│  3B (MHA)  8B (GQA)  20B (MQA)  34B (depth-upscaled)│
│  2048 ctx  4096 ctx  8192 ctx   8192 ctx             │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌──────────── TWO-PHASE TRAINING ─────────────────────┐
│  Phase 1: 3-4T tokens, CODE ONLY (116 langs)        │
│  Objective: 0.5*CLM + 0.5*FIM                       │
│       │                                              │
│       ▼                                              │
│  Phase 2: 500B tokens, 80% CODE + 20% LANGUAGE      │
│  (math, web docs, academic text)                     │
│       │                                              │
│       ▼                                              │
│  ══════ GRANITE CODE BASE MODELS ══════              │
│       │                                              │
│       ▼                                              │
│  Instruction Tuning (CommitPackFT, Math,             │
│  Code Instructions, Language Instructions)           │
│  + NEFTune noise                                     │
│       │                                              │
│       ▼                                              │
│  ══════ GRANITE CODE INSTRUCT MODELS ══════          │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌──────────── RESULTS ────────────────────────────────┐
│  ✅ SOTA on HumanEvalPack (gen+fix+explain)          │
│  ✅ 8B beats 34B CodeLlama on explanation (+9%)      │
│  ✅ 8B beats Llama-3-8B on GSM8K math (+12 pts)     │
│  ✅ 3B-Instruct ≈ CodeLlama-34B-Instruct            │
│  ✅ Best FIM infilling across all sizes               │
│  ⚠️ Not always #1 on pure code generation            │
│  ⚠️ CodeGemma-7B wins on CRUXEval, ReCode           │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
            Apache 2.0 Release
         (Enterprise-ready, Trustworthy)
```

---

## 🛠️ 11. IMPLEMENTATION SKETCH

### Pseudocode of the core training approach:
```python
# Phase 1: Code-only pretraining
model = TransformerDecoder(config)  # 3B/8B/20B/34B
tokenizer = BPE_tokenizer(vocab_size=49152)  # StarCoder tokenizer

for batch in code_data_loader(116_languages, total=4T_tokens):
    tokens = tokenizer.encode(batch)
    
    # 50% causal LM, 50% fill-in-middle
    if random() < 0.5:
        loss = causal_lm_loss(model, tokens)
    else:
        prefix, middle, suffix = random_split(tokens)
        if random() < 0.5:  # PSM mode
            input = [PRE] + prefix + [SUF] + suffix + [MID]
        else:               # SPM mode
            input = [SUF] + suffix + [PRE] + prefix + [MID]
        loss = fim_loss(model, input, target=middle)
    
    loss.backward()
    optimizer.step()  # AdamW, cosine LR schedule

# Phase 2: Code + Language pretraining  
for batch in mixed_data_loader(code=80%, language=20%, total=500B):
    loss = 0.5 * causal_lm_loss + 0.5 * fim_loss
    # Same as Phase 1, exponential LR decay

# Instruction tuning (for Instruct variants)
for epoch in range(3):
    for batch in instruction_data_loader():
        # Add NEFTune noise
        embeddings += uniform_noise(magnitude=5/sqrt(N*h))
        loss = causal_lm_loss(model, batch)  # FIM disabled (α=1)
```

### Frameworks/Libraries needed:
- **NVIDIA Megatron-LM** (custom fork) for distributed training
- **FlashAttention 2** for memory-efficient attention
- **NVIDIA Apex** for fused kernels (LayerNorm, RMSNorm, Adam)
- **PyTorch** with BF16 mixed precision
- **Padding-Free Transformer** implementation for instruction tuning

### Estimated compute cost:
- **Infrastructure:** IBM Vela (A100 80GB) + Blue Vela (H100 80GB) clusters, thousands of GPUs
- **Carbon footprint:** ~455 tCO2eq total for all models
- **Rough estimate:** Training the 8B model on 4.5T tokens likely required **~2,000-4,000 A100 GPU-hours** (conservative); the full family likely cost **millions of GPU-hours**
- **Blue Vela runs on 100% renewable energy**
- To reproduce: expect **$1M-$5M+ in cloud compute costs** for the full family
