

# CodeGemma: Open Code Models Based on Gemma — Full Breakdown

---

## 🎯 1. THE ONE-LINER

**Google took their Gemma language model and trained it on tons of code to create CodeGemma — a family of open AI models that can write, complete, and fill in code like a smart coding assistant.**

---

## 🤔 2. THE PROBLEM (Why does this paper exist?)

- **The problem:** Developers need AI coding assistants that are (a) **open** (not locked behind an API), (b) **fast** enough for real-time code completion in IDEs, and (c) **smart** enough to handle code generation, infilling, AND still understand natural language and math.
- **Why should you care?** Imagine you're writing an essay and you leave a blank in the middle — "The capital of France is _____ and it's beautiful." You need a tool that can fill in "Paris" instantly while you type. Now imagine doing that for code, where getting it wrong means the program crashes. That's what developers need every day.
- **Limitations of previous approaches:**
  - **Code Llama** (Meta): Strong but large and slower for real-time completion
  - **DeepSeek Coder**: Good accuracy but **~2x slower** at inference than CodeGemma 2B
  - **StarCoder2**: Decent but **~7x slower** than CodeGemma 2B for single-line completion
  - Most existing open code models were **either fast OR accurate**, not both
  - Many code models **lose natural language understanding** after heavy code training (catastrophic forgetting)

---

## 💡 3. THE KEY IDEA (The "Aha!" moment)

- **Core insight:** Start with a strong general-purpose language model (Gemma), then **continue training on 500B–1T tokens of primarily code** while carefully mixing in natural language to **avoid forgetting** general knowledge. Use **fill-in-the-middle (FIM) training** so the model can complete code not just at the end, but **anywhere in the middle** of a file.

- **Everyday analogy:** Think of Gemma as a **well-educated generalist** — they speak good English, understand math, etc. CodeGemma is like sending that generalist to an **intensive coding bootcamp** (500B+ tokens of code) while making sure they still remember how to have a normal conversation (20% natural language mix for 7B models). The FIM training is like teaching them to fill in crossword puzzles — they can look at what comes *before* AND *after* a blank and figure out what goes in the middle.

- **Three model variants:**
  ```
  ┌─────────────────────────────────────────────────┐
  │  CodeGemma 2B PT  → Speed demon for IDEs        │
  │  (100% code, FIM-focused, fastest inference)     │
  ├─────────────────────────────────────────────────┤
  │  CodeGemma 7B PT  → Strong all-rounder           │
  │  (80% code + 20% NL, code + language tasks)      │
  ├─────────────────────────────────────────────────┤
  │  CodeGemma 7B IT  → Chat/instruction following   │
  │  (SFT + RLHF on code + math data)               │
  └─────────────────────────────────────────────────┘
  ```

---

## 🏗️ 4. HOW IT WORKS (The Method - Layer by Layer)

### Step 1: Start from Gemma Pretrained Models
- **WHAT:** Take the existing Gemma 2B and 7B pretrained language models as the starting point.
- **WHY:** These already understand language, grammar, reasoning — no need to start from scratch.
- **HOW it connects:** Provides the foundation of language understanding that code training builds upon.

### Step 2: Continue Pretraining on Code + NL Data
- **WHAT:** Train further on **500B tokens (v1.0)** or **1T tokens (2B v1.1)** of primarily code from public repositories, plus web docs and math. 2B gets **100% code**; 7B gets **80% code / 20% natural language**.
- **WHY:** The code-heavy diet teaches the model programming syntax, patterns, and APIs. The 20% NL for 7B prevents catastrophic forgetting of language skills.
- **HOW it connects:** Creates the base pretrained code models (2B PT and 7B PT).

### Step 3: Fill-in-the-Middle (FIM) Preprocessing
- **WHAT:** 80-90% of training examples are reformatted so a random chunk is **removed from the middle** and placed at the end. The model learns to predict that missing chunk given prefix + suffix context.
- **WHY:** Real-world coding isn't just "type at the end of a file." Developers insert code between existing lines constantly. FIM makes the model useful for **code completion in the middle of files**.
- **HOW it connects:** Uses special tokens (`<|fim_prefix|>`, `<|fim_suffix|>`, `<|fim_middle|>`) and supports both PSM and SPM orderings.

### Step 4: Multi-file Packing
- **WHAT:** Group related files from the same repo together in training examples using **dependency graphs** and **unit test co-location**.
- **WHY:** Real code lives in repositories with many files that depend on each other. Training on isolated files misses cross-file context.
- **HOW it connects:** Makes the model better at understanding imports, dependencies, and repo-level code.

### Step 5: Instruction Tuning (7B IT only)
- **WHAT:** Fine-tune the 7B PT model using:
  - **Math datasets** (MATH, GSM8k, MathQA, synthetic algebra)
  - **Synthetic code Q&A pairs** (generated via OSS-Instruct approach, then LLM-filtered)
  - **SFT + RLHF** pipeline
- **WHY:** Makes the model follow instructions, answer coding questions, and reason through problems step-by-step.
- **HOW it connects:** Produces the final CodeGemma 7B IT model ready for chat-style coding assistance.

---

## 📊 5. THE PROOF (Results & Experiments)

### Benchmarks Used:
| Category | Benchmarks |
|----------|-----------|
| Code Completion | HumanEval Infilling (single-line, multi-line) |
| Code Generation | HumanEval, MBPP |
| Multi-lingual Code | BabelCode (C++, C#, Go, Java, JS, Kotlin, Python, Rust) |
| Natural Language | BoolQ, PIQA, TriviaQA, ARC-C, HellaSwag, MMLU, WinoGrande |
| Math Reasoning | GSM8K, MATH |

### Key Results:

- **CodeGemma 2B is ~2x faster** than DeepSeek Coder 2B and **~7x faster** than StarCoder2 for code infilling, with **comparable or better accuracy** (78.4% vs 79.9% single-line, 51.4% vs 50.9% multi-line)
- **CodeGemma 7B PT: 44.5% HumanEval, 56.2% MBPP** — massive jump from base Gemma 7B PT (32.3% / 44.4%)
- **CodeGemma 7B IT 1.1: 60.4% HumanEval** — competitive with much larger models
- **Math reasoning:** CodeGemma IT 1.1 gets **47.3% GSM8K** and **22.3% MATH**, beating DeepSeek Coder (43.2% / 19.2%) and Code Llama (13.0% GSM8K)
- **Natural language retention:** CodeGemma outperforms Mistral 7B by 7.2% and Llama-2 13B by 19.1% on NL benchmarks

### Most Impressive Result in Plain English:
**The tiny 2B model completes code nearly as accurately as models 3-4x its size, but does it in half the time — making it the best choice for real-time code autocomplete in your editor.**

### Limitations Admitted:
- DeepSeek Coder 7B **outperforms** CodeGemma 7B on HumanEval Infilling (85.9% vs 76.1% single-line)
- The paper is relatively light on failure analysis and ablation studies
- Instruction-tuned models are **much slower** than pretrained ones (by 5-6x)

---

## 🧩 6. KEY TERMS GLOSSARY

- **Gemma** → Google's open-source base language model family that CodeGemma is built on
- **FIM (Fill-in-the-Middle)** → Training technique where the model learns to fill in a missing code chunk given what comes before AND after
- **PSM (Prefix-Suffix-Middle)** → FIM format: show prefix, then suffix, then ask for the middle
- **SPM (Suffix-Prefix-Middle)** → Alternate FIM format: show suffix first, then prefix, then ask for middle
- **PT (Pretrained)** → Model trained only on raw text/code, no instruction following
- **IT (Instruction-Tuned)** → Model further fine-tuned to follow user instructions and answer questions
- **SFT (Supervised Fine-Tuning)** → Training on curated question-answer pairs with correct outputs
- **RLHF (Reinforcement Learning from Human Feedback)** → Training the model using human preference signals to improve output quality
- **HumanEval** → Benchmark of 164 Python programming problems to test code generation
- **MBPP** → "Mostly Basic Python Problems" — 974 simple Python tasks for evaluation
- **BabelCode** → Framework to test code generation across multiple programming languages
- **Infilling** → Completing a gap in the middle of existing code (not just appending at the end)
- **Multi-file Packing** → Grouping related source files together in training examples to teach cross-file understanding
- **Dependency Graph** → A map of which files import/rely on which other files in a codebase
- **GSM8K** → 8,500 grade-school math word problems benchmark
- **MATH Dataset** → Challenging competition-level math problems benchmark
- **OSS-Instruct** → Technique from Magicoder paper for generating synthetic code instruction data from open source code
- **Latency-sensitive** → Applications where response speed is critical (e.g., real-time autocomplete)

---

## 🔗 7. HOW IT CONNECTS

### Intellectual Family Tree:
```
Gemini (proprietary, Google)
    │
    ├──► Gemma (open, 2024) ──► CodeGemma (this paper)
    │
Related lineage:
    ├── Code Llama (Meta, from Llama 2)
    ├── DeepSeek Coder (DeepSeek AI)
    ├── StarCoder2 (BigCode / ServiceNow)
    ├── InCoder (FIM pioneer, Fried et al. 2023)
    └── Magicoder/OSS-Instruct (Wei et al. 2023)
         └── Used for synthetic data generation in CodeGemma
```

### Who Would Use This:
- **IDE developers** (VS Code extensions, JetBrains plugins) — especially the 2B model for real-time completion
- **Individual developers** wanting a local, private coding assistant
- **Researchers** needing an open code model to fine-tune for specialized domains
- **Companies** wanting to self-host coding AI without sending code to external APIs

### Future Work This Enables:
- Fine-tuning for **domain-specific** code (medical, finance, embedded systems)
- **Agentic coding** — models that can plan, write, test, and debug code autonomously
- Smaller, faster distilled versions for **on-device** coding assistance
- **Repository-level** code understanding and generation (building on multi-file packing)

---

## ⚖️ 8. CRITICAL ANALYSIS

### Hidden Assumptions:
- Assumes **publicly available code** is representative of what developers actually write — but proprietary/internal code styles can differ significantly
- Assumes HumanEval/MBPP are good proxies for real coding ability — these are mostly **short, self-contained functions**, not complex system design

### Weaknesses the Authors DON'T Mention:
- **No ablation studies** — we don't know the individual contribution of FIM, multi-file packing, math training, etc.
- **No comparison at the 2B level for code generation** (HumanEval/MBPP), only at 7B class
- The **7B pretrained model significantly lags DeepSeek Coder** on infilling (76.1% vs 85.9%) — this is somewhat glossed over
- **No evaluation on long-context or complex multi-file generation** — despite multi-file packing being a key training innovation
- Training data details are vague — "publicly available code repositories" with no specifics on size, language distribution, or licensing

### Is the Evaluation Fair?
- Mostly yes — they use **standard benchmarks** and compare against strong baselines
- But they report **speed** only for CodeGemma and some competitors (Code Llama numbers are cited from another paper, not measured on same hardware)
- No confidence intervals or variance reported

### Would This Work at Scale in the Real World?
- **Yes, especially the 2B model** — it's designed exactly for real-world IDE deployment and was validated in live Google coding environments
- The 7B models need GPU servers but are practical for cloud deployment
- The open release enables real adoption (available on HuggingFace)

---

## 📝 9. MEMORY ANCHORS

### Memorable Metaphor:
> **CodeGemma is like taking a brilliant liberal arts student (Gemma) and putting them through an intensive coding bootcamp — they come out able to write great code while still being articulate and good at math.**

### 3 Bullet Points (80% of the paper):
- **CodeGemma takes Google's Gemma models and continues training on 500B–1T tokens of code**, producing 2B and 7B variants for code completion, generation, and instruction following
- **The 2B model is the speed star** — nearly 2x faster than competitors with comparable accuracy, perfect for IDE autocomplete
- **Fill-in-the-middle training + multi-file packing + math data mixing** let the models complete code mid-file, understand cross-file context, and retain strong reasoning abilities

### Comprehension Check Question:
> *Why does CodeGemma train the 7B model on 80% code / 20% natural language, but the 2B model on 100% code? What tradeoff does this reflect?*

**Answer:** The 7B model is meant to be a general-purpose coding model that also handles NL tasks (chat, reasoning, Q&A), so it needs NL to avoid catastrophic forgetting. The 2B model is laser-focused on fast code infilling/completion in IDEs, where NL understanding is less important than raw coding speed and accuracy.

---

## 🗺️ 10. VISUAL MENTAL MAP

```
PROBLEM                        METHOD                              RESULT
───────                        ──────                              ──────
                                                                   
Need fast, open,          ┌─────────────────────┐            ┌─────────────────┐
accurate code models  ──► │  Start: Gemma 2B/7B  │            │  CodeGemma 2B PT │
                          │  (general LLM)       │            │  • Fastest 2B    │
Previous models are       └────────┬────────────┘            │    code model    │
either slow or closed              │                          │  • 78% single-   │
                                   ▼                          │    line infill   │
                          ┌─────────────────────┐            ├─────────────────┤
                          │ Continue pretrain on │            │  CodeGemma 7B PT │
                          │ 500B-1T code tokens  │            │  • 44.5% HE     │
                          │ + FIM preprocessing  │            │  • 56.2% MBPP   │
                          │ + Multi-file packing │            │  • Retains NL   │
                          └────────┬────────────┘            ├─────────────────┤
                                   │                          │  CodeGemma 7B IT │
                                   ▼                          │  • 60.4% HE     │
                          ┌─────────────────────┐            │  • 47.3% GSM8K  │
                          │ Instruction Tuning   │            │  • Best math in │
                          │ • Math datasets      │───────►   │    7B code class │
                          │ • Synthetic code QA  │            └─────────────────┘
                          │ • SFT + RLHF         │
                          └─────────────────────┘

KEY INNOVATIONS:
  ★ FIM at 80-90% rate (PSM + SPM modes)
  ★ Dependency-graph multi-file packing  
  ★ 80/20 code/NL mix prevents forgetting
  ★ Math training boosts code reasoning
```

---

## 🛠️ 11. IMPLEMENTATION SKETCH

### Pseudocode: FIM Data Preprocessing
```python
def prepare_fim_example(source_code, fim_rate=0.8):
    if random() > fim_rate:
        return source_code  # Regular left-to-right training
    
    # Pick random span to mask
    start = randint(0, len(source_code) - 1)
    end = randint(start + 1, len(source_code))
    
    prefix = source_code[:start]
    middle = source_code[start:end]  # The part to predict
    suffix = source_code[end:]
    
    if random() < 0.5:  # PSM mode
        return f"<|fim_prefix|>{prefix}<|fim_suffix|>{suffix}<|fim_middle|>{middle}"
    else:  # SPM mode
        return f"<|fim_suffix|>{suffix}<|fim_prefix|>{prefix}<|fim_middle|>{middle}"

def multi_file_pack(repo_files):
    graph = build_dependency_graph(repo_files)
    remove_unimportant_edges(graph)  # Break cycles
    sorted_files = topological_sort(graph)
    # Pack sorted files with <|file_separator|> between them
    return join_with_separator(sorted_files)
```

### Frameworks/Libraries Needed:
- **Training:** JAX/Flax (Google's stack) or PyTorch with HuggingFace Transformers
- **Inference:** HuggingFace Transformers, vLLM, or Keras
- **Evaluation:** BigCode Evaluation Harness, HumanEval, BabelCode

### Estimated Compute to Reproduce:
- **Not explicitly stated**, but training on 500B–1T tokens with 2B-7B parameter models typically requires:
  - **2B model:** ~hundreds of TPU v4/v5 hours (likely days on a large TPU pod)
  - **7B model:** ~thousands of TPU hours
  - **Rough estimate:** Several hundred thousand dollars in cloud compute
  - Google used their internal TPU infrastructure, making this **not easily reproducible** outside Google
