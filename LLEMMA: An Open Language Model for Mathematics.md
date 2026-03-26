

# LLEMMA: An Open Language Model for Mathematics — Full Breakdown

---

## 🎯 1. THE ONE-LINER

**The researchers took an AI that's good at coding and fed it tons of math textbooks, papers, and math code, making it the best open-source AI for solving math problems.**

---

## 🤔 2. THE PROBLEM (Why does this paper exist?)

- **What problem?** General-purpose AI language models (like ChatGPT) are decent at many things but mediocre at math. The best math-specialized AI (Google's Minerva) was **closed-source** — nobody outside Google could use it, study it, or build on it.
- **Why should anyone care?** Imagine you're a student and the best math tutor in the world exists, but only one company has access to it. Everyone else has to use worse tutors. This paper builds **an equally good tutor and makes it free for everyone**.
- **Limitations of previous approaches:**
  - **Minerva** (Google, 2022): Strong at math but **completely closed** — no model weights, no training data, no code released. Dead-end for the research community.
  - **Open math models** (e.g., Proof-Pile-1 era): Far behind Minerva in performance.
  - Most approaches focused only on **text-based** math, ignoring **code tools** (Python, SymPy) and **formal theorem provers** (Lean, Isabelle).

---

## 💡 3. THE KEY IDEA (The "Aha!" moment)

**Core insight:** Don't train a math model from scratch. Instead, **take an already-good code model (Code Llama) and keep training it on a carefully curated math diet** — a mix of scientific papers, math-heavy web pages, and mathematical code.

**Everyday analogy:** Think of it like training a chef. You don't start from zero — you take someone who's already great at knife skills and kitchen basics (Code Llama = good at code), then send them to a **specialized math cooking school** (Proof-Pile-2) where they study math recipes from textbooks (ArXiv papers), online forums (OpenWebMath), and actual math software (AlgebraicStack). They emerge as a **math specialist** without losing their general cooking abilities.

**Why Code Llama as the starting point?** Math and code share similar reasoning patterns — step-by-step logic, symbolic manipulation, formal precision. Starting from a code model gives a head start.

**Step-by-step method description:**
```
┌─────────────────┐
│   Llama 2 (7B)  │  General language model
│  (base model)   │
└────────┬────────┘
         │ Train on 500B tokens of code
         ▼
┌─────────────────┐
│   Code Llama    │  Good at code, basic math
│   (7B or 34B)   │
└────────┬────────┘
         │ Continue training on Proof-Pile-2
         │ (55B tokens of math-specific data)
         ▼
┌─────────────────┐
│    LLEMMA       │  Specialized math model
│   (7B or 34B)   │  + tool use + theorem proving
└─────────────────┘
```

---

## 🏗️ 4. HOW IT WORKS (The Method - Layer by Layer)

### Step 1: Curate the Proof-Pile-2 Dataset (55B tokens)
- **WHAT:** Assemble three types of math-heavy data:
  - **ArXiv papers** (29B tokens): Scientific papers with heavy math notation
  - **OpenWebMath** (15B tokens): Web pages filtered for mathematical content (preserving LaTeX, AsciiMath)
  - **AlgebraicStack** (11B tokens): *New dataset* of math-related code from 17 programming languages (Python/SymPy, Lean, Isabelle, Julia, Fortran, etc.)
- **WHY:** The model needs diverse exposure to how math is expressed — in papers, on the web, and in code. The code component is novel and crucial for tool use and formal proving.
- **HOW it connects:** This becomes the "textbook" the model studies in the next step.

### Step 2: Add General Data as Regularization (~5%)
- **WHAT:** Mix in 2% from the Pile (general text) and 3% from RedPajama GitHub (general code).
- **WHY:** Prevents the model from **forgetting** general language abilities — like a specialist doctor who still needs to communicate with patients. This is a form of **regularization**.
- **HOW it connects:** Combined with Proof-Pile-2 (95%), this forms the final training mixture.

### Step 3: Optimize the Data Mixture Ratios
- **WHAT:** Test different ratios of ArXiv : Web : Code via short training runs. Selected **2:4:1** (arXiv:Web:Code).
- **WHY:** Not all data is equally valuable. Web math data (OpenWebMath) got the highest weight because it was most similar to the target math problems.
- **HOW it connects:** Determines sampling probabilities during training.

### Step 4: Continue Pretraining Code Llama
- **WHAT:**
  - **7B model:** Train for 200B tokens (42,000 steps) → ~23,000 A100-hours
  - **34B model:** Train for 50B tokens (12,000 steps) → ~47,000 A100-hours
  - Standard autoregressive language modeling (predict next token)
  - 256 A100 40GB GPUs, bfloat16, Flash Attention 2
- **WHY:** Continued pretraining is cheaper than training from scratch while still injecting domain knowledge deeply into the model's weights.
- **HOW it connects:** Produces the final LLEMMA models ready for evaluation.

### Step 5: Evaluate Across Multiple Math Tasks
- **WHAT:** Test on chain-of-thought math solving, tool-augmented solving (Python), and formal theorem proving (Lean/Isabelle) — all in **few-shot** settings (no task-specific fine-tuning).
- **WHY:** Shows the model is a versatile **base model**, not just trained for one benchmark.

---

## 📊 5. THE PROOF (Results & Experiments)

### Benchmarks Used:
| Benchmark | Description |
|-----------|-------------|
| **MATH** | 5K hard high-school competition problems |
| **GSM8k** | Middle-school math word problems |
| **OCWCourses** | MIT undergraduate STEM problems |
| **MMLU-STEM** | 18 STEM subjects from MMLU |
| **SAT** | 32 math questions from May 2023 SAT |
| **miniF2F** | Formal math Olympiad/undergrad statements |

### Key Results (Greedy Decoding):

| Model | Params | MATH | GSM8k |
|-------|--------|------|-------|
| Code Llama | 7B | 4.5% | 10.5% |
| Minerva | 8B | 14.1% | 16.2% |
| **LLEMMA** | **7B** | **18.0%** | **36.4%** |
| Code Llama | 34B | 12.2% | 29.6% |
| **LLEMMA** | **34B** | **25.0%** | **51.5%** |
| Minerva | 62B | 27.6% | 52.4% |

### Most Impressive Results in Plain English:
- **LLEMMA 7B beats Minerva 8B** on MATH (18.0% vs 14.1%) despite Minerva being closed-source and from Google
- **LLEMMA 34B nearly matches Minerva 62B** (a model almost 2x its size!) on MATH with majority voting (43.1% vs 43.4%)
- LLEMMA 34B improves over Code Llama 34B by **+20 points on GSM8k** and **+13 points on MATH**
- **Tool use works out of the box:** MATH+Python scores rise to 27.1% (34B) without any fine-tuning for tool use
- **Formal theorem proving without fine-tuning:** LLEMMA-7b matches ReProver (a fine-tuned specialist) at 26.23% on miniF2F

### Limitations Admitted:
- Training of 7B model stopped early (42K instead of 48K steps) due to NaN loss / hardware failures
- Only tested up to 34B parameters (no 70B+ version)
- Memorization analysis shows ~7% of MATH test problems appear in training data (though this doesn't clearly help accuracy)
- Not evaluated on many downstream tasks beyond math

---

## 🧩 6. KEY TERMS GLOSSARY

- **Continued pretraining** → Taking an already-trained model and training it more on new, specialized data
- **Proof-Pile-2** → The 55B-token math dataset created for this paper (papers + web + code)
- **AlgebraicStack** → New 11B-token dataset of math-related source code across 17 languages
- **OpenWebMath** → 15B-token dataset of math-filtered web pages
- **Code Llama** → Meta's open-source code-specialized language model (LLEMMA's starting point)
- **Chain-of-thought (CoT)** → Making the model show its work step-by-step before giving an answer
- **Majority voting (maj@k)** → Generate k solutions, pick the most common answer
- **Few-shot evaluation** → Testing a model by showing it just a few examples, without fine-tuning
- **Formal theorem proving** → Writing math proofs in a computer-checkable language (Lean, Isabelle)
- **Tactic prediction** → Generating the next step in a formal proof
- **Autoformalization** → Converting informal math (English + LaTeX) into formal proof code
- **miniF2F** → Benchmark of 488 formalized math competition/undergrad problems
- **RoPE** → Rotary Position Embedding — a way to encode position information in transformers
- **Best first search** → A tree search strategy for exploring proof steps
- **SymPy** → Python library for symbolic mathematics
- **Perplexity** → How "surprised" a model is by text (lower = model understands it better)
- **n-gram overlap** → Checking if sequences of n consecutive words appear in training data
- **Domain adaptation** → Specializing a general model for a specific field
- **Autoregressive language modeling** → Training objective where the model predicts the next token

---

## 🔗 7. HOW IT CONNECTS

### Intellectual Family Tree:
```
GPT-3 (Brown et al., 2020)
    │
    ├── PaLM (Chowdhery et al., 2022)
    │       └── Minerva (Lewkowycz et al., 2022) ← closed-source math LM
    │
    └── Llama 2 (Touvron et al., 2023)
            └── Code Llama (Rozière et al., 2023)
                    └── LLEMMA (this paper) ← open-source math LM
```

### Related Papers:
1. **Minerva** (Lewkowycz et al., 2022): Direct predecessor — same "continued pretraining for math" idea, but closed-source and based on PaLM
2. **WizardMath** (Luo et al., 2023) & **MetaMath** (Yu et al., 2023): Supervised fine-tuning approaches for math — boost specific benchmarks but less general

### Who Would Use This:
- **Researchers** studying mathematical reasoning in AI
- **Developers** building math tutoring or homework help tools
- **Formal verification** researchers needing a base model for theorem proving
- **Anyone** who wants to fine-tune an open math model for their specific use case

### Future Work Enabled:
- Reward modeling and RLHF for mathematical reasoning
- Long-context math problem solving
- Better formal theorem provers built on LLEMMA
- Studies of data composition effects on reasoning

---

## ⚖️ 8. CRITICAL ANALYSIS

### Hidden Assumptions:
- **Assumes Code Llama is the best starting point** — but they don't compare against continuing from Llama 2 directly (only show Code Llama → LLEMMA path)
- **Assumes the MATH training set is a good proxy** for selecting mixture weights — this could bias the model toward competition-style problems
- **Assumes more math data = better math ability** — doesn't deeply explore what specifically in the data drives the improvement

### Weaknesses NOT Mentioned:
- **No comparison with fine-tuned models on a level playing field** — WizardMath and MetaMath are shown only briefly in the appendix, not the main results
- **No ablation on starting from Llama 2 vs Code Llama** — how much does the code pretraining actually help?
- **The 7B model training crashed** (NaN at step 42K) and they couldn't fully investigate why — this raises reproducibility concerns
- **Evaluation prompts differ across models** — Minerva results are quoted from their paper with potentially different prompts, making comparison imperfect
- **No analysis of what types of math problems benefit most** from each data source

### Is the Evaluation Fair?
- Mostly yes, but **Minerva scores are self-reported** from Google's paper — different evaluation code could give slightly different numbers
- The SAT benchmark is very small (32 questions) — high variance
- Using MATH training set perplexity to choose data mixture could be seen as indirect data leakage to the benchmark

### Real-World Scalability:
- **Compute cost is reasonable:** ~23K A100-hours for 7B, ~47K A100-hours for 34B — accessible to well-funded labs
- **Open-source release is the key value proposition** — enables real-world deployment
- As a **base model** (not instruction-tuned), it requires additional work before consumer-facing deployment

---

## 📝 9. MEMORY ANCHORS

### Memorable Metaphor:
> **LLEMMA is like taking a skilled programmer (Code Llama) and sending them to math graduate school (Proof-Pile-2) — they come out as a math expert who can also write code to solve problems and even write formal proofs, and unlike the previous valedictorian (Minerva), they share all their notes with the class.**

### 3 Bullet Points (80% of the paper):
- 📚 **Data recipe:** Curate Proof-Pile-2 (55B tokens of ArXiv papers + math web pages + math code) and continue training Code Llama on it
- 📈 **Results:** LLEMMA 7B beats the closed-source Minerva 8B on MATH; LLEMMA 34B nearly matches Minerva 62B — while being fully open
- 🔧 **Versatility:** Without any fine-tuning, LLEMMA can use Python as a calculator AND write formal proofs in Lean/Isabelle

### Comprehension Check Question:
> *Why did the authors choose Code Llama (rather than Llama 2) as the starting point for continued pretraining, and what evidence supports this choice?*

---

## 🗺️ 10. VISUAL MENTAL MAP

```
PROBLEM                          METHOD                              RESULT
═══════                          ══════                              ══════

Best math LM              ┌──────────────────┐
is closed (Minerva)  ──►  │  Curate Data     │
                          │  Proof-Pile-2     │
No open model             │  ┌──────────────┐ │
can compete         ──►   │  │ ArXiv (29B)  │ │
                          │  │ Web   (15B)  │ │
Math needs code           │  │ Code  (11B)  │ │
& formal proofs    ──►    │  └──────────────┘ │
                          └────────┬─────────┘
                                   │
                                   ▼
                          ┌──────────────────┐        ┌─────────────────────┐
                          │  Continue Train   │        │   LLEMMA 7B & 34B   │
                          │  Code Llama       │──────► │                     │
                          │  on Proof-Pile-2  │        │ • MATH: 18% / 25%   │
                          │  (200B / 50B tok) │        │ • GSM8k: 36% / 52%  │
                          └──────────────────┘        │ • Tool use ✓        │
                                                       │ • Theorem proving ✓ │
                                                       │ • Beats Minerva 8B  │
                                                       │ • Matches Minerva62B│
                                                       │ • FULLY OPEN ✓      │
                                                       └─────────────────────┘
```

---

## 🛠️ 11. IMPLEMENTATION SKETCH

### Pseudocode:
```python
# 1. Build Proof-Pile-2
arxiv = load("RedPajama/ArXiv")           # 29B tokens
web = load("OpenWebMath")                   # 15B tokens
code = build_algebraic_stack(               # 11B tokens
    sources=["Stack", "GitHub", "Lean_proofsteps", "Isabelle_proofsteps"],
    languages=17, filter=math_keywords
)
proof_pile_2 = MixtureDataset(
    {arxiv: 2, web: 4, code: 1}            # Ratio 2:4:1
)

# 2. Add regularization data
general_text = load("Pile", exclude="ArXiv")  # 2% weight
general_code = load("RedPajama/GitHub")        # 3% weight
train_mix = combine(proof_pile_2=0.95, general_text=0.02, general_code=0.03)

# 3. Continue pretraining
model = load_pretrained("CodeLlama-7B")
model.rope_base = 10_000  # Contract RoPE for future long-context

for step in range(42_000):  # 200B tokens, batch=4M tokens
    batch = train_mix.sample(batch_size=4_000_000)
    loss = autoregressive_loss(model, batch)
    loss.backward()
    optimizer.step()  # AdamW, lr=1e-4 cosine decay

# 4. Evaluate few-shot
results = evaluate(model, benchmarks=["MATH", "GSM8k", "miniF2F", ...])
```

### Frameworks/Libraries Needed:
- **GPT-NeoX** (training framework)
- **PyTorch** + **Flash Attention 2**
- **DeepSpeed ZeRO Stage 1** (optimizer sharding)
- **Language Model Evaluation Harness** (evaluation)
- **LeanDojo** (Lean 4 theorem proving evaluation)
- **Portal-to-Isabelle** (Isabelle proof checking)

### Estimated Compute Cost:
| Model | GPU-Hours | GPUs | Wall Clock |
|-------|-----------|------|------------|
| LLEMMA 7B | ~23,000 A100-hours | 256× A100 40GB | ~3.7 days |
| LLEMMA 34B | ~47,000 A100-hours | 256× A100 40GB | ~7.6 days |
| **Total** | **~70,000 A100-hours** | | ~$150K-$250K at cloud rates |
