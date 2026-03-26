

# WizardCoder: Empowering Code LLMs with Evol-Instruct

---

## 🎯 1. THE ONE-LINER
**The paper uses a clever trick to automatically make coding homework problems harder and harder, then trains an AI on these harder problems so it becomes much better at writing code.**

---

## 🤔 2. THE PROBLEM (Why does this paper exist?)

- **The problem:** Open-source code-generating AI models (like StarCoder) were trained on tons of code but never specifically taught to *follow instructions* well. Closed-source models like ChatGPT and GPT-4 were much better at coding tasks, and the gap was huge.

- **Why should anyone care?** Imagine you hire a junior programmer who's read millions of lines of code (pre-training) but has never actually practiced solving problems that get progressively harder (instruction tuning). They'd be okay, but not great. Now imagine giving them a **customized training program** that starts with easy problems and gradually ramps up difficulty — they'd improve much faster. That's the gap this paper fills for Code LLMs.

- **Limitations of previous approaches:**
  - Most Code LLMs focused only on **pre-training** (reading lots of code), not on instruction fine-tuning
  - Existing instruction-tuning methods (like Self-Instruct used by Alpaca) were designed for **general text**, not specifically for code
  - Code Alpaca's instruction data was too simple and limited (~20k basic samples)
  - No systematic method existed to **automatically increase the complexity** of code instructions

---

## 💡 3. THE KEY IDEA (The "Aha!" moment)

- **Core insight:** You don't need MORE training data — you need **HARDER** training data. By using GPT-3.5 to systematically evolve simple coding problems into harder ones (adding constraints, edge cases, complexity requirements, even buggy code as traps), you can dramatically boost a model's coding ability.

- **Everyday analogy:** Think of it like training for a math competition. Instead of doing 1000 easy addition problems, you take 20 problems and keep making each one harder:
  - Round 1: "Add 2+3"
  - Round 2: "Add 2+3, but now handle negative numbers too"
  - Round 3: "Add 2+3 with negative numbers, fractions, and do it in O(1) time"
  
  The student who trained on **escalating difficulty** beats the one who did 1000 easy problems.

- **The 5 evolution heuristics (ways to make problems harder):**

```
┌──────────────────────────────────────────┐
│       CODE EVOL-INSTRUCT METHODS         │
├──────────────────────────────────────────┤
│ 1. Add new constraints (+10 words)       │
│ 2. Replace common → rare requirements    │
│ 3. Add more reasoning steps              │
│ 4. Include BUGGY code as misdirection    │
│ 5. Require better time/space complexity  │
└──────────────────────────────────────────┘
```

---

## 🏗️ 4. HOW IT WORKS (The Method - Layer by Layer)

**Step 1: Start with seed data (Code Alpaca)**
- **WHAT:** Collect ~20,000 simple code instruction-response pairs from Code Alpaca
- **WHY:** You need a starting point — basic coding problems to evolve from
- **CONNECTS TO:** These become inputs to the evolution process

**Step 2: Apply Code Evol-Instruct (iteratively)**
- **WHAT:** Feed each instruction to GPT-3.5 with one of 5 heuristic prompts that say "make this harder by [method]"
- **WHY:** Automatically creates more complex, diverse training data without human effort
- **CONNECTS TO:** Each evolved instruction needs a response

**Step 3: Generate responses for evolved instructions**
- **WHAT:** Use GPT-3.5 to generate code solutions for each evolved instruction
- **WHY:** The model needs both the hard question AND the correct answer to learn from
- **CONNECTS TO:** This creates the evolved training dataset

**Step 4: Merge and accumulate data across rounds**
- **WHAT:** After each evolution round, merge evolved data from ALL previous rounds with the original dataset (~78k total samples after multiple rounds)
- **WHY:** Combining different difficulty levels provides a curriculum of easy → hard
- **CONNECTS TO:** This merged dataset becomes the fine-tuning input

**Step 5: Fine-tune the base Code LLM**
- **WHAT:** Fine-tune StarCoder-15B or CodeLlama-34B-Python on the evolved dataset
- **WHY:** The pre-trained model already "knows" code; now it learns to follow complex instructions
- **HOW:** Batch size 512, seq length 2048, **only 200 steps**, learning rate 2e-5, cosine scheduler
- **CONNECTS TO:** Output is WizardCoder

**Step 6: Controlled Evol Stop**
- **WHAT:** Use an external dev set to monitor performance; stop evolving if performance drops
- **WHY:** Too much evolution can lead to overly complex/nonsensical instructions
- **RESULT:** Best performance at **3 rounds** of evolution

---

## 📊 5. THE PROOF (Results & Experiments)

### Benchmarks tested:
| Benchmark | What it tests | Scale |
|-----------|--------------|-------|
| HumanEval | 164 Python problems | ~9.6 tests/problem |
| HumanEval+ | Same but ~774.8 tests/problem | Harder eval |
| MBPP | 500 Python problems | 3 tests each |
| DS-1000 | 1000 data science tasks (7 libraries) | Real-world |
| MultiPL-E | 8 programming languages | Multilingual |

### Key results (pass@1):

| Model | HumanEval | HumanEval+ | MBPP |
|-------|-----------|------------|------|
| StarCoder 15B (base) | 33.6 | 29.3 | 43.6 |
| **WizardCoder 15B** | **57.3** (+24!) | **40.9** | **51.8** |
| CodeLlama-Python 34B | 53.7 | — | 56.2 |
| **WizardCoder 34B** | **73.2** | **64.6** | **61.2** |
| Claude | 47.6 | 39.0 | — |
| Bard | 44.5 | 36.6 | — |
| GPT-3.5 (ChatGPT) | 48.1 | 63.4 | 52.2 |
| GPT-4 | 67.0/88.4 | 76.2 | — |

### Most impressive results in plain English:
- **WizardCoder 15B beat Claude and Bard** — a 15B open-source model beating major closed-source models
- **WizardCoder 34B matched ChatGPT on HumanEval (73.2 vs 48.1 reported/73.2 greedy) and BEAT it on HumanEval+ (64.6 vs 63.4)**
- On **MultiPL-E**, WizardCoder 34B beat all open-source models in **all 9 programming languages**
- On **DS-1000**, WizardCoder scored 32.8 overall vs StarCoder's 25.4 (insertion mode)

### Failure cases & limitations admitted:
- Still **significantly behind GPT-4** (88.4 on HumanEval)
- DS-1000 34B results couldn't be reported due to framework incompatibility
- Performance on MBPP for 15B (51.8) is lower than CodeLlama-Instruct 34B (57.0)
- Evolution beyond ~3 rounds shows **diminishing or negative returns**

---

## 🧩 6. KEY TERMS GLOSSARY

- **Code LLM** → A large AI model specifically trained on programming code
- **Instruction fine-tuning** → Teaching a pre-trained model to follow specific user instructions
- **Evol-Instruct** → A method from WizardLM that evolves simple instructions into complex ones using an LLM
- **Code Evol-Instruct** → This paper's code-specific adaptation of Evol-Instruct with 5 coding heuristics
- **pass@1** → The percentage of problems solved correctly on the first attempt
- **HumanEval** → A benchmark of 164 Python programming problems created by OpenAI
- **HumanEval+** → HumanEval with ~80x more test cases per problem (harder to pass)
- **MBPP** → "Mostly Basic Python Problems" — 500 simpler Python coding tasks
- **DS-1000** → 1000 data science coding problems across 7 Python libraries
- **MultiPL-E** → A multilingual code benchmark covering 18+ programming languages
- **StarCoder** → An open-source 15B parameter Code LLM from BigCode project
- **CodeLlama** → Meta's open-source Code LLM family based on Llama 2
- **Code Alpaca** → A dataset of ~20K simple code instruction-response pairs
- **Greedy decoding** → Generating code by always picking the most likely next token (no randomness)
- **Self-Instruct** → A method where an LLM generates its own training instructions
- **Adversarial sample** → Intentionally incorrect/tricky code used to make training harder
- **Heuristic** → A practical rule-of-thumb method (not a formal algorithm)
- **Evol Stop** → The mechanism to halt evolution when performance starts degrading

---

## 🔗 7. HOW IT CONNECTS

### Intellectual family tree:
```
Self-Instruct (Wang et al. 2022)
    ↓
Alpaca (Taori et al. 2023) ──→ Code Alpaca (seed data)
    ↓
Evol-Instruct / WizardLM (Xu et al. 2023)
    ↓
╔════════════════════════════════╗
║  Code Evol-Instruct /          ║
║  WizardCoder (THIS PAPER)      ║
╚════════════════════════════════╝
    ↑                    ↑
StarCoder (Li 2023)   CodeLlama (Rozière 2023)
 (base model)          (base model)
```

### Comparable recent papers:
1. **OctoCoder (Muennighoff et al., 2023)** — Also does instruction tuning on StarCoder but uses commit-based data. WizardCoder 15B (57.3) significantly beats OctoCoder (46.2) on HumanEval.
2. **CodeLlama-Instruct (Rozière et al., 2023)** — Uses self-instruct for instruction tuning. WizardCoder 34B (73.2) crushes CodeLlama-Instruct 34B (41.5) on HumanEval.

### Who would use this:
- **AI researchers** wanting to improve open-source code generation models
- **Companies** needing code assistants without relying on closed-source APIs
- **Dataset engineers** looking to augment instruction datasets automatically

### Future work enabled:
- Applying Code Evol-Instruct to even larger base models
- Combining with RLHF or DPO for further alignment
- Multi-turn code instruction evolution
- Domain-specific evolution (e.g., for security, ML engineering)

---

## ⚖️ 8. CRITICAL ANALYSIS

### Hidden assumptions:
- **GPT-3.5 is a good "teacher"** — The quality of evolved instructions and responses is bounded by GPT-3.5's capability
- **Complexity = better** — Assumes more complex instructions always lead to better training signal (diminishing returns after round 3 suggest this isn't always true)
- **Code Alpaca is a representative seed** — The diversity of the final dataset is limited by the diversity of the ~20K seed instructions

### Weaknesses the authors DON'T mention:
- **Heavy dependency on a proprietary model** (GPT-3.5) for data generation — raises reproducibility concerns
- **No human evaluation** of generated code quality, readability, or correctness beyond pass@1
- **No evaluation on more complex tasks** like multi-file projects, debugging, or code review
- The evolved responses from GPT-3.5 may contain **subtle bugs** that get trained into WizardCoder
- **Only 200 fine-tuning steps** — unclear if the model is undertrained and whether more training would help or hurt
- No ablation on **individual heuristic contributions** — which of the 5 methods matters most?

### Is the evaluation fair?
- Mostly yes — they use standard benchmarks and comparable settings
- But comparing with closed-source models at specific API snapshots is tricky (APIs change over time)
- The MBPP score for StarCoder had to be re-evaluated due to dataset discrepancies
- DS-1000 evaluation for 34B was skipped due to framework issues

### Would this work at scale?
- **Yes, likely** — the method is straightforward and has been shown to work with both 15B and 34B models
- **Cost concern:** Each evolution round requires OpenAI API calls for ~20K samples × multiple rounds
- The approach is **model-agnostic** — shown to work with StarCoder and CodeLlama foundations

---

## 📝 9. MEMORY ANCHORS

### Memorable metaphor:
> **WizardCoder is like a martial arts training program**: instead of just sparring against easy opponents (basic training data), you progressively fight tougher and tougher opponents (evolved instructions). The student (Code LLM) develops skills they didn't know they had, eventually beating fighters from prestigious dojos (Claude, Bard).

### 3 bullet points (80% of the paper):
- 🔄 **Code Evol-Instruct** takes simple coding problems and uses GPT-3.5 to make them progressively harder through 5 code-specific strategies (add constraints, add reasoning, add bugs, add complexity requirements, use rare requirements)
- 📈 Fine-tuning StarCoder/CodeLlama on this evolved data (~78K samples, only 200 steps) creates **WizardCoder**, which beats all open-source models and even some closed-source ones (Claude, Bard, nearly matches ChatGPT)
- 🔑 The key insight: **instruction complexity, not quantity**, is what drives performance gains (proven by controlled experiments with equal sample/token counts)

### Comprehension test question:
> *"If you trained WizardCoder with 5x more data but at the same difficulty level as the seed data, would it perform as well as training on evolved data? Why or why not?"*
> (Answer: No — Table 5 shows that even with the same number of samples/tokens, evolved round data beats seed data. The gain comes from complexity, not quantity.)

---

## 🗺️ 10. VISUAL MENTAL MAP

```
PROBLEM                          METHOD                              RESULT
═══════                          ══════                              ══════

Open-source Code LLMs     ┌─────────────────────┐
are weak at following     │  CODE EVOL-INSTRUCT  │
instructions              │                     │
        │                 │  Seed: Code Alpaca   │
        │                 │  (20K simple tasks)  │
        ▼                 │         │            │
┌──────────────┐          │         ▼            │         ┌──────────────────┐
│ Gap between  │          │  ┌──────────────┐    │         │   WIZARDCODER    │
│ open-source  │───────►  │  │ GPT-3.5      │    │────►    │                  │
│ & closed-    │          │  │ evolves each  │    │         │  15B: 57.3%      │
│ source models│          │  │ problem using │    │         │  (beats Claude,  │
└──────────────┘          │  │ 5 heuristics  │    │         │   Bard)          │
                          │  └──────┬───────┘    │         │                  │
                          │         │            │         │  34B: 73.2%      │
                          │    ×3 rounds         │         │  (≈ ChatGPT)     │
                          │         │            │         │                  │
                          │         ▼            │         │  SOTA on 5       │
                          │  78K evolved samples │         │  benchmarks      │
                          │         │            │         └──────────────────┘
                          │         ▼            │
                          │  Fine-tune StarCoder │         ┌──────────────────┐
                          │  or CodeLlama        │         │  KEY FINDING:    │
                          │  (200 steps only!)   │────►    │  Complexity >    │
                          │                     │         │  Quantity         │
                          └─────────────────────┘         └──────────────────┘
```

---

## 🛠️ 11. IMPLEMENTATION SKETCH

### Pseudocode:
```python
# Core Algorithm: Code Evol-Instruct + Fine-tuning
HEURISTICS = [
    "Add new constraints (+10 words)",
    "Replace common with rare requirement",
    "Add more reasoning steps",
    "Provide erroneous code as misdirection",
    "Require better time/space complexity"
]

def code_evol_instruct(seed_data, num_rounds=3):
    all_data = seed_data  # ~20K samples
    
    for round in range(num_rounds):
        evolved = []
        for instruction in seed_data:
            method = random.choice(HEURISTICS)
            prompt = f"Increase difficulty using: {method}\n{instruction}"
            new_instruction = gpt35_turbo(prompt)
            new_response = gpt35_turbo(new_instruction)
            evolved.append((new_instruction, new_response))
        
        evolved = filter_failed_evolutions(evolved)
        evolved = filter_data_leakage(evolved, test_sets)
        all_data = all_data + evolved  # accumulate
        seed_data = evolved  # next round evolves from this
        
        # Check dev set performance — stop if drops
        if eval_on_dev(all_data) < previous_best:
            break
    
    return all_data  # ~78K samples

def train_wizardcoder(base_model, evolved_data):
    # Fine-tune with standard supervised learning
    fine_tune(base_model, evolved_data,
              batch_size=512, lr=2e-5, steps=200,
              warmup=30, scheduler="cosine", fp16=True)
```

### Frameworks/libraries needed:
- **PyTorch** + **Hugging Face Transformers** for fine-tuning
- **OpenAI API** (gpt-3.5-turbo) for evolution and response generation
- **DeepSpeed** (likely, for 15B/34B model training)
- **bigcode-evaluation-harness** for MultiPL-E evaluation
- **EvalPlus** for HumanEval/HumanEval+ evaluation

### Estimated compute cost:
- **API cost:** ~78K samples × GPT-3.5-turbo calls ≈ **$50–200** (depending on token length)
- **Fine-tuning:** 200 steps with batch size 512, seq len 2048 on a 15B model ≈ **8–16 A100 GPUs for a few hours**
- **For 34B model:** Likely **16–32 A100 GPUs** for similar duration
- **Total:** Remarkably cheap compared to pre-training — this is one of the paper's underrated strengths
