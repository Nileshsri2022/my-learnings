

# Evaluating Large Language Models Trained on Code (Codex Paper)

---

## 🎯 1. THE ONE-LINER
**OpenAI taught a GPT language model to write Python code by training it on GitHub, and built a new test (HumanEval) to check if the code actually works — not just if it looks right.**

---

## 🤔 2. THE PROBLEM (Why does this paper exist?)

- **What problem?** Before this, there was no reliable AI that could take a plain English description of a function and write working Python code. GPT-3 could barely do it (0% on their benchmark). Existing code evaluation methods (like BLEU score) measured whether code *looked* similar to a reference, not whether it actually *ran correctly*.

- **Why should anyone care?** Imagine you hire a chef, and to judge them you only check if their recipe *reads* similarly to a famous cookbook — not if the food actually *tastes good*. That's what BLEU score does for code. This paper says: **let's actually run the code and see if it passes tests**, and then builds a model that can genuinely write working programs.

- **Limitations of previous approaches:**
  - GPT-3 was trained mostly on text, not code → near 0% code correctness
  - **BLEU score** (the standard metric) doesn't capture whether code actually works — wrong code often scores higher than correct code
  - No clean, uncontaminated benchmark existed for evaluating code generation from docstrings
  - Prior code models (CodeBERT, PyMT5) focused on understanding/search, not generation of full working functions

---

## 💡 3. THE KEY IDEA (The "Aha!" moment)

**Two core insights:**

1. **Fine-tune GPT on code → Codex.** Take a powerful language model and specialize it on 159 GB of Python from GitHub. This is like taking a brilliant writer and having them read millions of cookbooks — they become a recipe-writing expert.

2. **"If at first you don't succeed, try 100 times."** Instead of demanding one perfect answer, **generate many samples and pick the best one**. With 100 attempts, Codex-S solves **77.5%** of problems (vs. 37.7% with one try). This is like taking 100 shots at a basketball hoop — you only need one to go in.

**Everyday analogy:** Think of Codex as an auto-complete for code, but on steroids. You write the instruction manual (docstring), and Codex writes the code. If you let it brainstorm 100 drafts and pick the one that works, it's shockingly good.

**The pipeline, step by step:**
```
[English docstring] → [Codex generates code] → [Run unit tests] → [Pass/Fail]
                                ↓
                    Generate k samples → Pick best one
```

---

## 🏗️ 4. HOW IT WORKS (The Method - Layer by Layer)

### Step 1: Collect training data (159 GB of Python)
- **WHAT:** Scraped 54 million public GitHub repos (May 2020), kept Python files < 1MB
- **WHY:** Need massive code corpus to teach the model programming patterns
- **Filtering:** Removed auto-generated files, files with long lines, low alphanumeric content
- → Final dataset: **159 GB** of clean Python

### Step 2: Fine-tune GPT on code → Codex
- **WHAT:** Take GPT-3 models (up to 12B parameters) and continue training on the code dataset
- **WHY:** GPT-3 knows language but not code; fine-tuning specializes it
- **HOW:** Standard language modeling (predict next token), trained for **100 billion tokens**
- **Key trick:** Added special whitespace tokens (indentation matters in Python!) → **30% fewer tokens** to represent code
- → Connects to Step 3 because now we need to evaluate it

### Step 3: Create HumanEval benchmark (164 problems)
- **WHAT:** Hand-wrote 164 programming problems with function signatures, docstrings, and unit tests (~7.7 tests per problem)
- **WHY:** Existing benchmarks were contaminated (solutions already on GitHub). Need **novel problems** the model hasn't seen
- **HOW:** Problems test language comprehension, algorithms, simple math — like easy coding interview questions
- → Connects to Step 4 for evaluation

### Step 4: Evaluate with pass@k (functional correctness)
- **WHAT:** Generate n=200 samples per problem, count how many pass ALL unit tests, compute unbiased pass@k
- **WHY:** BLEU score doesn't work for code (wrong code can have high BLEU). **Actually running the code** is the true test
- **Key formula:** pass@k = 1 − C(n−c, k) / C(n, k), where c = correct samples out of n total
- **Safety:** Runs code in a **sandboxed gVisor container** to prevent malicious execution
- → Connects to Step 5 for improvements

### Step 5: Supervised fine-tuning → Codex-S
- **WHAT:** Further fine-tune Codex on curated (docstring → correct function) pairs from competitive programming sites and CI-traced functions
- **WHY:** GitHub has configs, scripts, data files — not all relevant to "docstring → function" task. **Distribution mismatch** hurts performance
- **HOW:** Collected ~10K competitive programming problems + ~40K CI-traced functions, filtered with Codex-12B (remove ambiguous/too-hard problems)
- → Boosts pass@1 from 28.8% to **37.7%**

### Step 6: Sample ranking heuristics
- **WHAT:** When you can't run tests (real-world usage), pick the sample with **highest mean log-probability**
- **WHY:** In deployment, you don't have unit tests for the user's code
- **Result:** Mean log-prob ranking → **44.5%** (vs. 28.8% random single sample)

### Step 7: Docstring generation (Codex-D)
- **WHAT:** Train a reverse model: given code, generate the docstring
- **WHY:** Safety tool — can describe what generated code does; enables "back-translation" ranking
- **Result:** Comparable but slightly lower pass rates than Codex-S

---

## 📊 5. THE PROOF (Results & Experiments)

### Benchmarks used:
- **HumanEval** (164 hand-written problems) — primary benchmark
- **APPS** (5000 competitive programming problems) — secondary benchmark

### Key numbers:

| Model | pass@1 | pass@10 | pass@100 |
|-------|--------|---------|----------|
| GPT-3 12B | ~0% | ~0% | ~0% |
| GPT-J 6B | 11.6% | 15.7% | 27.7% |
| **Codex 12B** | **28.8%** | **46.8%** | **72.3%** |
| **Codex-S 12B** | **37.7%** | — | **77.5%** |
| Codex-S + mean logp reranking | **44.5%** | — | — |
| TabNine | 2.6% | 4.4% | 7.6% |

### Most impressive result in plain English:
**GPT-J with 6B parameters performs like Codex with only 300M parameters (20× fewer).** Fine-tuning on code is incredibly effective — it's like the difference between a general reader and a trained programmer.

### APPS results:
- Codex-12B (1-shot) matches GPT-Neo 2.7B fine-tuned on APPS on introductory problems
- On competition-level problems, even with 1000 samples, only ~3-5% solved

### Failure cases & limitations admitted:
- **Not sample-efficient:** Needs hundreds of millions of lines of code; a CS student learns from far less
- **Performance drops exponentially with docstring complexity:** Each additional chained operation reduces pass rate by 2-3×
- **Variable binding errors:** When multiple variables need different operations, Codex confuses which operation goes with which variable
- **BLEU ≠ correctness:** Incorrect solutions often have *higher* BLEU scores than correct ones
- **Misalignment:** When given buggy prompts, Codex produces buggier code (gets *worse* with scale)
- **Insecure code:** 30-70% of cryptographic code samples used clearly insecure configurations

---

## 🧩 6. KEY TERMS GLOSSARY

- **Codex** → A GPT model fine-tuned on GitHub Python code to generate programs
- **Codex-S** → Codex further fine-tuned on curated (docstring → correct function) pairs
- **Codex-D** → A model trained to generate docstrings from code (reverse of Codex)
- **HumanEval** → A hand-written benchmark of 164 Python problems with unit tests
- **pass@k** → The probability that at least 1 of k generated samples passes all unit tests
- **Docstring** → A text description at the top of a function explaining what it does
- **Functional correctness** → Whether code actually produces the right output (vs. looking similar)
- **BLEU score** → A text-similarity metric borrowed from machine translation; unreliable for code
- **Nucleus sampling (top-p)** → A sampling strategy that picks from the most probable tokens summing to probability p
- **Temperature** → Controls randomness in sampling; higher = more diverse but less reliable outputs
- **Mean log-probability** → Average of log-probabilities of each token; used to rank sample quality
- **Fine-tuning** → Continuing to train a pre-trained model on a specialized dataset
- **Sandbox (gVisor)** → A secure container that safely runs untrusted generated code
- **Back-translation** → Using Codex-D to score how well a generated code sample matches the original docstring
- **Program synthesis** → Automatically generating a program from a specification
- **Program induction** → A model that produces outputs directly without generating explicit code
- **APPS** → A benchmark of 10K competitive programming problems
- **Continuous Integration (CI)** → Automated testing pipelines that run when code is committed
- **Scaling law** → The observation that model performance follows a power law with model size
- **Alignment** → Whether a model's behavior matches the user's actual intent

---

## 🔗 7. HOW IT CONNECTS

### Intellectual family tree:
```
GPT (Radford 2018) → GPT-2 → GPT-3 (Brown 2020) → Codex (this paper)
                                    ↑
             Scaling Laws (Kaplan 2020) informed design
                                    
CodeBERT (Feng 2020) → Code understanding (parallel work)
PyMT5 (Clement 2020) → Code translation (similar spirit)
SPoC (Kulal 2019) → pass@k metric inspiration
The Pile / GPT-Neo / GPT-J → Open-source comparisons
APPS (Hendrycks 2021) → Complementary benchmark
```

### Related contemporary papers:
1. **AlphaCode (DeepMind, 2022)** — Takes sampling even further (millions of samples + filtering) for competitive programming
2. **CodeGen (Salesforce, 2022)** — Multi-turn code generation with conversation-style prompting

### Who would use this and for what?
- **Software engineers** → Code autocomplete (GitHub Copilot)
- **Students** → Learning to code, getting hints
- **Non-programmers** → Writing simple scripts from descriptions
- **Researchers** → Benchmark for code generation models

### Future work this enables:
- GitHub Copilot (production descendant)
- RLHF for code (InstructGPT/ChatGPT lineage)
- Code execution feedback loops
- Multi-language code generation
- More sophisticated code benchmarks (HumanEval+ , MBPP, etc.)

---

## ⚖️ 8. CRITICAL ANALYSIS

### Hidden assumptions:
- **Python only** — Results may not transfer to other languages (C++, Java, Rust)
- **Standalone functions only** — Real software is multi-file, stateful, complex
- **Unit test completeness** — Passing tests ≠ fully correct code; tests may be incomplete
- **Docstrings are sufficient specifications** — In reality, specifications are ambiguous

### Weaknesses the authors DON'T emphasize:
- **HumanEval is small** (164 problems) — statistical power is limited
- **No multi-step or interactive coding** — Real programming involves debugging, iteration, context
- **Tokenizer inefficiency** even after whitespace fix — Other code-specific tokenizers could help
- **Training data contamination** is hard to fully rule out — even hand-written problems might resemble training data patterns
- **No comparison to retrieval-augmented approaches** — Looking up similar code from a database might work just as well

### Is the evaluation fair?
- **Mostly yes** — They release HumanEval, provide unbiased pass@k, compare fairly
- **But** — Comparing Codex (trained on 159GB code) to GPT-J (trained on 8% code in The Pile) isn't quite apples-to-apples on data volume
- The APPS evaluation is weaker — Codex isn't fine-tuned on APPS format, making comparison less meaningful

### Would this work at scale in the real world?
- **Yes, partially** — GitHub Copilot (descendant) is used by millions
- **But** — Over-reliance risk is real; generated code needs human review; performance degrades on complex multi-step tasks; security concerns remain

---

## 📝 9. MEMORY ANCHORS

### Memorable metaphor:
> **Codex is like a junior developer with photographic memory of millions of programs** — it can write code fast but needs a senior engineer to review its work, especially for complex or security-critical tasks. And if you let it try 100 times, it'll usually get it right at least once.

### 3 bullet points capturing 80% of the paper:
- 🔥 **Fine-tuning GPT-3 on 159 GB of GitHub Python code creates Codex, which solves 28.8% of hand-written coding problems (vs. 0% for GPT-3)**
- 🎯 **Generating 100 samples and picking the one that passes tests boosts performance to 77.5%** — repeated sampling is a surprisingly powerful strategy
- 📏 **They introduce HumanEval (164 problems) and pass@k as the right way to evaluate code models**, showing that BLEU score is unreliable for measuring functional correctness

### One question to test understanding:
> *Why is generating 100 samples and selecting the best one so much more effective than generating a single sample, and what does this tell us about the nature of the model's "knowledge"?*

---

## 🗺️ 10. VISUAL MENTAL MAP

```
PROBLEM                          METHOD                              RESULT
═══════                          ══════                              ══════

GPT-3 can't code     ──┐
(0% on coding tasks)   │
                       ├──► Fine-tune GPT on       ──► Codex-12B
BLEU score is broken   │    159GB GitHub Python         28.8% pass@1
(wrong code scores     │         │
 high)                 │         │
                       │         ▼
No clean benchmark  ───┘    Create HumanEval        ──► 164 hand-written
                            (functional correctness)     problems with
                                 │                       unit tests
                                 ▼
                         Further fine-tune on       ──► Codex-S-12B
                         curated problems                37.7% pass@1
                         (competitive prog + CI)
                                 │
                                 ▼
                         Sample 100 times +          ──► 77.5% pass@100
                         pick best (oracle)               (oracle)
                                 │                       44.5% (mean logp)
                                 ▼
                         Train reverse model         ──► Codex-D
                         (code → docstring)               safety tool
                                 │
                                 ▼
                         ┌───────────────────┐
                         │  KEY FINDINGS:     │
                         │  • Scale helps     │
                         │  • Sampling > 1    │
                         │  • BLEU is broken  │
                         │  • Long specs fail │
                         │  • Alignment issue │
                         └───────────────────┘
```

---

## 🛠️ 11. IMPLEMENTATION SKETCH

### Pseudocode of the core evaluation pipeline (~15 lines):
```python
def evaluate_codex(model, problems, n=200, k=100, temp=0.8):
    results = {}
    for problem in problems:
        prompt = assemble_prompt(problem.signature, problem.docstring)
        samples = []
        for _ in range(n):
            code = model.generate(prompt, temperature=temp,
                                   stop=['\nclass','\ndef','\n#','\nif'])
            samples.append(code)
        
        # Count correct samples (run in sandbox)
        c = sum(1 for s in samples if run_tests_in_sandbox(s, problem.tests))
        
        # Unbiased pass@k estimator
        if n - c < k:
            results[problem.id] = 1.0
        else:
            results[problem.id] = 1.0 - np.prod(1.0 - k / np.arange(n-c+1, n+1))
    
    return np.mean(list(results.values()))  # Average pass@k
```

### Frameworks/Libraries needed:
- **PyTorch** (model training)
- **Transformers / custom GPT implementation** (model architecture)
- **gVisor** (sandboxed code execution)
- **NumPy** (pass@k computation)
- **Kubernetes** (distributed training infrastructure)

### Estimated compute to reproduce:
- **Pre-training GPT-3 12B:** Hundreds of petaflop/s-days (~$1-5M estimated)
- **Fine-tuning to Codex-12B:** Similar compute to pre-training (~hundreds of petaflop/s-days)
- **Inference for evaluation:** 200 samples × 164 problems × multiple models = significant but feasible on a cluster
- **Total estimated cost:** **$2-10M+** (2021 prices, rough estimate)
- Not reproducible on academic budgets without donated compute
