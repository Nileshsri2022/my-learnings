

# Code Llama: Open Foundation Models for Code — Full Breakdown

---

## 🎯 1. THE ONE-LINER

**Meta took their smart text-reading AI (Llama 2) and taught it to become an expert programmer by training it on tons of code, then released it for free so anyone can use it.**

---

## 🤔 2. THE PROBLEM (Why does this paper exist?)

- **What problem?** Developers need AI assistants that can write, complete, and fix code — but the best ones (like GPT-4/Copilot) are closed-source and proprietary. Open-source code models existed (StarCoder, InCoder) but were **smaller and less capable**.

- **Why should anyone care?** Imagine you're writing an essay, and your spell-checker is locked behind a subscription, can't be customized, and you can't see how it works. You'd want a free, open, and customizable alternative. Same for coding AI — developers want tools they can inspect, modify, and deploy freely.

- **Limitations of previous approaches:**
  - **StarCoder/SantaCoder**: Open, but only ~15B parameters max, limiting performance
  - **Codex/Copilot**: Powerful but **closed-source**, no commercial use allowed
  - **AlphaCode**: Focused on competitive programming, not general coding assistance
  - Most code models were **trained from scratch on code only**, missing the broad language understanding that general models have
  - **Short context windows** (4K tokens) — can't see entire files or repos
  - No open model supported both **code completion AND infilling** (filling in the middle of code)

---

## 💡 3. THE KEY IDEA (The "Aha!" moment)

**Core insight:** Instead of training a code model from scratch, **start with a general-purpose language model (Llama 2) that already understands language, then specialize it on code through a cascade of fine-tuning stages.** This is like teaching an already-literate person to code, rather than teaching someone to read AND code simultaneously.

**Cooking analogy:** Think of it like making a gourmet dish:
1. Start with a **high-quality base stock** (Llama 2's general language understanding)
2. Add the **main ingredient** (500B tokens of code training)
3. Add **specialty seasoning** (Python-specific training)
4. Teach it **table manners** (instruction fine-tuning for safety)
5. Give it a **bigger plate** (long context fine-tuning to handle huge files)

**The three killer features combined:**
```
┌──────────────────────────────────────────┐
│         CODE LLAMA RECIPE                │
│                                          │
│  1. Foundation: Llama 2 (general NLU)    │
│  2. Code specialization (500B tokens)    │
│  3. Infilling (fill-in-the-middle)       │
│  4. Long context (4K → 100K tokens)      │
│  5. Instruction following + Safety       │
└──────────────────────────────────────────┘
```

**Novel trick for long context:** Instead of the standard approach of linearly downscaling RoPE frequencies, they **increase the base period θ from 10,000 to 1,000,000**. This simple change lets the model attend to far-away tokens without architectural changes, enabling extrapolation to 100K tokens after training on only 16K.

---

## 🏗️ 4. HOW IT WORKS (The Method - Layer by Layer)

### Step 1: Start with Llama 2 Foundation
- **WHAT:** Take the pretrained Llama 2 models (7B, 13B, 34B, 70B) that were trained on 2T tokens of general text+code
- **WHY:** Starting from a model that already understands language gives a massive head start — a model trained from scratch on the same code data needs **~240B more tokens** to match
- **HOW it connects:** These weights are the initialization for all subsequent training

### Step 2: Code Training (500B-1T tokens)
- **WHAT:** Continue training on a **code-heavy dataset** (85% code, 8% natural language about code, 7% general text)
- **WHY:** Specializes the model for code while retaining natural language understanding. The small amount of natural language keeps the model well-rounded
- **HOW it connects:** Produces the base **Code Llama** model; this is the branching point for all variants

### Step 3: Infilling Training (for 7B, 13B, 70B)
- **WHAT:** Train with a **causal masking objective** — randomly split documents into prefix/middle/suffix, move middle to end, predict it autoregressively
- **WHY:** Enables the model to **fill in missing code** given surrounding context (essential for IDE autocomplete at cursor position)
- **Details:**
  - 90% of training examples use infilling format
  - Split equally between **PSM** (prefix-suffix-middle) and **SPM** (suffix-prefix-middle) formats
  - 4 special tokens added to mark prefix/middle/suffix boundaries
- **HOW it connects:** This runs simultaneously with Step 2, using a multi-task objective

### Step 4: Long Context Fine-Tuning (LCFT)
- **WHAT:** Fine-tune on **16,384-token sequences** (up from 4,096) while changing RoPE base period θ from 10,000 to 1,000,000
- **WHY:** Real-world code files can be enormous; need to see entire files or even repositories for better completion
- **Math intuition:** Higher θ → rotation frequencies change more slowly → model can distinguish positions even at very large distances → attention can "reach" farther back
- **HOW it connects:** Applied after code training, produces models that work up to 100K tokens at inference

### Step 5a: Python Specialization (→ Code Llama - Python)
- **WHAT:** Further train on **100B tokens** of a Python-heavy mix (75% Python)
- **WHY:** Python is the most popular programming language; specialization yields significant gains
- **HOW it connects:** Branches from Code Llama after Step 2, then gets its own LCFT

### Step 5b: Instruction Fine-Tuning (→ Code Llama - Instruct)
- **WHAT:** Fine-tune on ~5B tokens combining:
  - **Proprietary instruction data** from Llama 2 (RLHF V5)
  - **Self-instruct data** (~14K question-test-solution triplets generated by the model itself)
  - **Rehearsal data** (6% code + 2% natural language to prevent forgetting)
- **WHY:** Makes the model follow user instructions, be helpful, and be safe
- **Self-instruct recipe:**
  1. Llama 2 70B generates 62K programming questions
  2. De-duplicate to ~52K
  3. Code Llama 7B generates unit tests + 10 solutions per question
  4. Keep first solution that passes tests → ~14K validated triplets
- **HOW it connects:** Built on top of Code Llama (or Code Llama - Python for 70B)

---

## 📊 5. THE PROOF (Results & Experiments)

### Benchmarks Used:
| Benchmark | What it tests |
|-----------|--------------|
| **HumanEval** | 164 Python programming problems (zero-shot) |
| **MBPP** | 500 Python programming tasks (3-shot) |
| **APPS** | Competition/interview coding problems |
| **MultiPL-E** | HumanEval translated to 7 programming languages |
| **TruthfulQA/ToxiGen/BOLD** | Safety benchmarks |

### Key Results (specific numbers):

- **Code Llama - Instruct 70B: 67.8% on HumanEval pass@1** — matches GPT-4's 67.0%!
- **Code Llama - Python 7B outperforms Llama 2 70B** on both HumanEval (38.4% vs 30.5%) and MBPP (47.6% vs 45.4%) — a **10x smaller model** beating its parent
- **All Code Llama models outperform every open model on MultiPL-E** (multi-language)
- Code Llama 7B (26.3% avg on MultiPL-E) outperforms StarCoder 15.5B (25.0%) and CodeGen-Multi 16B (13.4%)
- **Infilling costs almost nothing**: only 0.6-1.1 percentage points lost on benchmarks
- **Long context works up to 100K tokens** with stable perplexity
- **Toxicity drops to ~0%** for all Instruct models (vs 17-23% for base models)

### Most impressive result in plain English:
**A 7-billion-parameter Code Llama model, which can run on a single GPU, writes better Python code than Llama 2 with 70 billion parameters (10x larger). And the biggest Code Llama model ties with GPT-4 on HumanEval while being completely open-source.**

### Limitations admitted:
- **Long context fine-tuning slightly hurts short-context performance** (~0.5% on HumanEval, ~1.9% on MBPP)
- **Instruction fine-tuning can cost coding performance** (34B Instruct: 41.5% vs 48.8% base on HumanEval)
- **Math reasoning degrades** compared to Llama 2 (Code Llama 34B: 32.7% vs Llama 2 34B: 42.2% on GSM8K)
- **False refusals** — sometimes refuses valid requests (e.g., "how to kill a process")
- Extrapolation beyond 16K tokens shows some retrieval degradation

---

## 🧩 6. KEY TERMS GLOSSARY

| Term | Simple Explanation |
|------|-------------------|
| **LLM** → Large Language Model; an AI trained on huge text datasets to understand/generate language |
| **Llama 2** → Meta's open-source general-purpose language model that Code Llama builds upon |
| **Fine-tuning** → Additional training on specialized data to improve performance on specific tasks |
| **Infilling / FIM** → Predicting missing code in the middle of a file, given the code before AND after the gap |
| **RoPE** → Rotary Position Embedding; a way to encode the position of each token using rotation matrices |
| **LCFT** → Long Context Fine-Tuning; training the model to handle longer sequences (16K+) |
| **pass@k** → Probability of getting a correct solution if you generate k attempts and pick the best |
| **HumanEval** → A benchmark of 164 Python programming problems with test cases |
| **MBPP** → Mostly Basic Programming Problems; 500 simple Python tasks |
| **MultiPL-E** → HumanEval translated into multiple programming languages |
| **BPE** → Byte Pair Encoding; a method to break text into sub-word tokens |
| **PSM/SPM** → Prefix-Suffix-Middle / Suffix-Prefix-Middle; two formats for arranging infilling training data |
| **Self-instruct** → Using the model itself to generate training data (questions, tests, solutions) |
| **RLHF** → Reinforcement Learning from Human Feedback; using human preferences to improve model outputs |
| **Nucleus sampling** → Sampling strategy that only considers the top-p probability mass of next tokens |
| **Greedy decoding** → Always picking the highest-probability next token |
| **AdamW** → An optimizer (algorithm for updating model weights during training) |
| **Cosine schedule** → Gradually decreasing the learning rate following a cosine curve |
| **Causal masking** → Training technique where parts of input are rearranged and predicted left-to-right |
| **Red teaming** → Deliberately trying to make the model produce harmful outputs to find vulnerabilities |
| **Token healing** → A technique to handle broken sub-word tokens at boundaries in infilling |
| **Perplexity** → How "surprised" the model is by text; lower = better language understanding |

---

## 🔗 7. HOW IT CONNECTS

### Intellectual Family Tree:
```
GPT-3 / Codex (OpenAI, 2021)
    ↓ (showed LLMs can code)
AlphaCode (DeepMind, 2022)
    ↓ (competition-level coding)
StarCoder / SantaCoder (BigCode, 2023)
    ↓ (open-source code models)
Llama → Llama 2 (Meta, 2023)
    ↓ (open foundation model)
═══════════════════════════
    CODE LLAMA (this paper)
═══════════════════════════
```

**Key ideas borrowed from:**
- **Codex** (Chen et al., 2021): Fine-tuning a general model for code (vs training from scratch)
- **Bavarian et al., 2022**: Fill-in-the-middle (FIM) training with PSM/SPM formats
- **Chen et al., 2023b**: Position interpolation for extending context (but Code Llama uses a different approach — base period scaling)
- **Llama 2** (Touvron et al., 2023b): Foundation model + RLHF safety approach

### Comparison with 2 related papers:
1. **StarCoder (Li et al., 2023)**: Trained from scratch on code only, 15.5B params, 8K context. Code Llama is bigger (up to 70B), starts from Llama 2 (better), handles 100K context, and supports infilling.
2. **phi-1 (Gunasekar et al., 2023)**: Uses "textbook-quality" filtered data + synthetic exercises. Code Llama instead uses raw public code without synthetic filtering, which may be less optimized for simple benchmarks but potentially more general.

### Who would use this:
- **IDE developers** (code autocomplete, infilling)
- **AI coding assistants** (chatbot-style code help)
- **Researchers** studying code generation, program synthesis
- **Companies** wanting customizable, self-hosted code models

### Future work enabled:
- Fine-tuning on domain-specific codebases (e.g., internal company code)
- Repository-level reasoning with 100K context
- Better execution-guided training (using Code Llama's self-instruct approach at larger scale)
- Multi-turn code editing and debugging assistants

---

## ⚖️ 8. CRITICAL ANALYSIS

### Hidden assumptions:
- **HumanEval/MBPP are good proxies for real coding ability** — but they're function-level, short-context problems that don't capture real software engineering
- **Public code datasets are representative** — but they may over-represent certain languages, coding styles, or simple code
- The LCC benchmark data **overlaps with training data** (acknowledged in a footnote)

### Weaknesses the authors DON'T mention:
- **No evaluation on real-world software engineering tasks** (refactoring, code review, debugging large codebases)
- **No evaluation on code security/vulnerability introduction** — generated code could have bugs or security holes
- **Math reasoning degradation is significant** (42.2% → 32.7% on GSM8K for 34B) but downplayed
- The **self-instruct data is only ~14K examples** — this is tiny, and quality is limited to what Code Llama 7B can verify
- **Benchmark contamination risk**: Training on public code likely includes code similar to benchmark solutions
- No user study or real developer evaluation

### Is the evaluation fair?
- **Mostly yes**: They use standard benchmarks, compare with many baselines, provide ablations
- **But**: pass@1 with greedy decoding for their models vs different settings for others makes direct comparison tricky. Some competitor numbers are self-reported from other papers
- The safety evaluation is **limited to English** and standard benchmarks — real-world safety risks (especially code-specific ones like generating vulnerable code) are not deeply explored

### Would this work at scale in the real world?
- **Yes, with caveats**: The 7B model is practical (single GPU), and the models are genuinely useful for code completion
- **But**: 100K context requires enormous memory; the 70B model needs significant compute
- False refusals and hallucinated code would require careful deployment guardrails
- Code quality verification (running tests, linting) would need to be added as a layer on top

---

## 📝 9. MEMORY ANCHORS

### Memorable metaphor:
**Code Llama is like taking a well-educated professor (Llama 2) and sending them to an intensive coding bootcamp (500B tokens of code) — they learn to code faster and better than someone starting from scratch because they already understand language, logic, and context.**

### 3 bullets that capture 80% of the paper:
- 🔧 **Code Llama = Llama 2 + 500B code tokens + infilling + long context (100K) + instruction tuning**, released in 3 variants × 4 sizes
- 📈 **Code Llama - Python 7B beats Llama 2 70B** on code benchmarks; Code Llama 70B matches GPT-4 on HumanEval — all open-source
- 🧪 **Key insight: Starting from a general language model is much better than training on code from scratch** (saves ~240B tokens to match), and infilling + long context can be added with minimal performance cost

### Comprehension test question:
> *Why does Code Llama change the RoPE base period θ from 10,000 to 1,000,000 instead of using linear frequency scaling, and what practical advantage does this provide?*

*(Answer: Increasing θ reduces the decay in attention scores over long distances, explicitly allowing far-away tokens to contribute to predictions. Unlike linear scaling which only maintains the range of rotations seen during pretraining, this approach enables extrapolation beyond the fine-tuning sequence length — allowing the model to handle up to 100K tokens despite being fine-tuned on only 16K.)*

---

## 🗺️ 10. VISUAL MENTAL MAP

```
PROBLEM                          METHOD                              RESULT
═══════                          ══════                              ══════

Open code models       ┌─────────────────────────┐
are too small/weak     │     LLAMA 2 (2T tokens)  │
                       │    General knowledge AI   │
                       └──────────┬──────────────┘
No infilling support              │
                                  ▼
Short context (4K)     ┌─────────────────────────┐
                       │  CODE TRAINING (500B)    │         Code Llama 7B-70B
Can't follow          │  85% code + 15% NL       │──────►  SOTA open model
instructions safely    │  + Infilling (FIM) obj   │         HE: up to 53%
                       └──────────┬──────────────┘
                                  │
                    ┌─────────────┼──────────────┐
                    ▼             ▼              ▼
            ┌──────────┐  ┌──────────┐  ┌───────────┐
            │   LCFT   │  │  Python  │  │ Instruct  │
            │ θ: 10K→  │  │  +100B   │  │  +5B      │
            │    1M    │  │  tokens  │  │  tokens   │
            │ 4K→100K  │  │          │  │ +safety   │
            └──────────┘  └──────────┘  └───────────┘
                 │             │              │
                 ▼             ▼              ▼
            ┌──────────┐  ┌──────────┐  ┌───────────┐
            │  CODE    │  │  CODE    │  │   CODE    │    CL-Python 7B
            │  LLAMA   │  │  LLAMA - │  │  LLAMA -  │──► beats
            │ (base)   │  │  PYTHON  │  │ INSTRUCT  │    Llama2 70B!
            │          │  │          │  │           │
            │ Infill ✓ │  │ Best for │  │ Safe +    │    CL-I 70B
            │ 100K ✓   │  │ Python   │  │ Helpful   │──► matches
            └──────────┘  └──────────┘  └───────────┘    GPT-4!

          ALL models beat every open model on MultiPL-E
          ALL Instruct models: ~0% toxic generations
```

---

## 🛠️ 11. IMPLEMENTATION SKETCH

### Pseudocode: Self-Instruct Data Generation Pipeline
```python
# Core self-instruct data generation (Section 2.5)

def generate_self_instruct_dataset():
    # Step 1: Generate questions using Llama 2 70B
    questions = llama2_70b.generate(
        prompt=INTERVIEW_QUESTION_PROMPT,  # Fig 10
        n=62000
    )
    
    # Step 2: Deduplicate
    questions = deduplicate(questions)  # ~52K remain
    
    dataset = []
    for question in questions:
        # Step 3a: Generate unit tests
        tests = code_llama_7b.generate(
            prompt=TEST_GEN_PROMPT.format(question),  # Fig 11
            n=1
        )
        
        # Step 3b: Generate 10 candidate solutions
        solutions = code_llama_7b.generate(
            prompt=SOLUTION_PROMPT.format(question, tests),  # Fig 12
            n=10
        )
        
        # Step 3c: Keep first passing solution
        for solution in solutions:
            if run_tests(solution, tests):
                dataset.append((question, tests, solution))
                break
    
    return dataset  # ~14K triplets
```

### Pseudocode: Infilling Training (FIM)
```python
def apply_fim_transform(document, prob=0.9):
    if random() > prob or document.spans_multiple_contexts:
        return document  # standard autoregressive
    
    # Random split points
    split1 = randint(0, len(document))
    split2 = randint(split1, len(document))
    
    prefix = document[:split1]
    middle = document[split1:split2]
    suffix = document[split2:]
    
    if random() < 0.5:  # PSM format
        return f"<PRE>{prefix}<SUF>{suffix}<MID>{middle}<EOT>"
    else:  # SPM format
        return f"<PRE><SUF>{suffix}<MID>{prefix}{middle}<EOT>"
```

### Frameworks/Libraries Needed:
- **PyTorch** (model training)
- **Fairscale / custom Meta training libraries** (distributed training)
- **SentencePiece** (tokenization, same as Llama 2)
- **A100-80GB GPUs** (training hardware)

### Estimated Compute Cost:
- **Total for all 12 models: ~1,400,000 A100-GPU hours**
- **Carbon footprint: 228.55 tCO2eq** (offset by Meta)
- To reproduce just the 7B model: Rough estimate ~50K-100K GPU hours
- **Inference**: 7B model runs on a single consumer GPU (e.g., RTX 3090/4090 with quantization)
