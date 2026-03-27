

# Orca-Math: Unlocking the potential of SLMs in Grade School Math

---

## 🎯 1. THE ONE-LINER
**A small AI model (7 billion parameters) learns to solve grade school math problems almost as well as models 10x its size, by practicing with a smart tutor that gives it feedback on its mistakes.**

---

## 🤔 2. THE PROBLEM (Why does this paper exist?)

- **The problem:** Small language models (SLMs) were terrible at solving math word problems. People believed you needed at least **34 billion parameters** to break 80% accuracy on the GSM8K benchmark.
- **Why care?** Imagine you want a math tutor app on your phone. Giant models like GPT-4 live in the cloud, cost lots of money, and are slow. **If a small model could do math well, everyone could have a local, cheap, fast math helper.**
- **Relatable analogy:** It's like saying "only a PhD professor can teach 5th grade math." This paper shows that with the right training program, a **teaching assistant** can do just as well.
- **Limitations of previous approaches:**
  - 🔴 **Brute force ensembling:** Run the model 48-100 times, pick the best answer via majority vote → expensive (200 API calls per question!)
  - 🔴 **Code generation:** Some models generate Python code instead of reasoning in natural language → fragile, requires code execution
  - 🔴 **Massive datasets:** Phi-GSM used **12 million** problems; Orca-Math uses only **200K**
  - 🔴 **External tools/verifiers:** Many approaches needed a separate verifier model (doubling compute)

---

## 💡 3. THE KEY IDEA (The "Aha!" moment)

**Two-part insight:**

1. **Quality over quantity in data:** Use a multi-agent system (Suggester + Editor) to create **diverse, progressively harder** math problems from a small seed set — like a curriculum that adapts.

2. **Let the student practice and get graded:** After initial training, let the small model **attempt problems itself**, then have the teacher (GPT-4) **grade those attempts**. The model learns from both its **correct and incorrect** solutions.

### 🍳 Cooking Analogy:
Imagine training a junior chef:
- **Step 1 (SFT):** The head chef demonstrates 200K recipes → junior watches and copies
- **Step 2 (Iterative Learning):** The junior tries cooking the same dishes on their own → the head chef **tastes each dish** and says "this one's great" ✅ or "this one's wrong, here's why" ❌
- **Step 3:** The junior tries again with lessons learned → gets even better feedback
- The magic: **the junior learns more from their own mistakes than from just watching**

### The Agent-Instruct Pipeline (ASCII):
```
Seed Problems (36K)
       │
       ▼
┌─────────────────┐
│ "Ask Me Anything"│──→ Flip each number in a problem
│     Agent        │     to create new question variants
└────────┬────────┘     (120K new problems)
         │
         ▼
┌─────────────────┐     ┌──────────┐
│  Suggester Agent │────▶│  Editor   │
│ "Make it harder" │◀────│  Agent   │
└─────────────────┘     └────┬─────┘
   (ideas only)              │ (creates harder problem)
                              │  ↕ 2 rounds
                              ▼
                     37K harder problems
                              │
         + 6K DMath problems  │
                              ▼
                    ╔═══════════════╗
                    ║  200K Dataset  ║
                    ║ + GPT-4 sols   ║
                    ╚═══════════════╝
```

---

## 🏗️ 4. HOW IT WORKS (The Method - Layer by Layer)

### Step 1: Build the Dataset (Agent-Instruct)
- **WHAT:** Create 200K diverse math word problems from a 36K seed set
- **WHY:** Quality + diversity of training data is the biggest lever for SLM performance
- **HOW:**
  - **Seed set:** Collect 36,217 problems from 11 open-source math datasets
  - **"Ask Me Anything" agent:** For each seed problem, replaces each number with "some" and makes that the unknown → generates ~120K new problems
  - **Suggester + Editor agents:** Iteratively make problems harder (add more variables, more steps, more complexity) → generates ~37K harder problems
  - **DMath:** Add 6,216 externally sourced problems
  - **Solution generation:** GPT-4-Turbo solves all 200K problems
- **Connects to:** Step 2 (SFT training)

### Step 2: Supervised Fine-Tuning (Iteration #1 → Model M1)
- **WHAT:** Fine-tune Mistral-7B on all 200K (question, GPT-4 solution) pairs
- **WHY:** Gives the model a strong baseline by learning from expert demonstrations
- **HOW:** Standard instruction-format fine-tuning, loss only on answer tokens, LR=1e-6, 1 epoch, 64 GPUs (8 nodes × 8 A100s)
- **Result:** M1 achieves **79.91%** on GSM8K (81.50% with 2 epochs)
- **Connects to:** Step 3 (generates student solutions)

### Step 3: Student Practice (Generate Solutions)
- **WHAT:** M1 generates **4 solutions** per problem (200K × 4 = 800K solutions) using sampling (top_p=0.95, temp=0.7)
- **WHY:** Creates on-policy data — the model's **own** correct and incorrect attempts, which are more informative for learning
- **HOW:** GPT-4 evaluates whether each student answer matches the teacher answer → labels each as **positive** or **negative**
- **Connects to:** Step 4 (preference learning)

### Step 4: Preference Learning (Iteration #2 → Model M2)
- **WHAT:** Train M1 using KTO (Kahneman-Tversky Optimization) on preference pairs (correct solution preferred over incorrect solution)
- **WHY:** Learning from both good AND bad examples is more powerful than just good examples. **"Knowing what's wrong is as valuable as knowing what's right."**
- **HOW:**
  - For each question: collect set of positive solutions (q⁺) and negative solutions (q⁻)
  - If ALL 4 student solutions were correct (~80K questions): borrow negatives from other questions (synthetic negatives)
  - Train with KTO (beta=0.3, LR=1e-6)
- **Result:** M2 achieves **85.06%**
- **Connects to:** Step 5 (repeat!)

### Step 5: Repeat (Iteration #3 → Orca-Math)
- **WHAT:** Use M2 to generate 4 new solutions per problem, rebuild preference dataset, train again with KTO
- **WHY:** Each iteration produces better solutions → better training signal → better model
- **HOW:** Same as Step 3-4 but starting from M2 instead of M1, with lower LR (1e-7)
- **Result:** Orca-Math achieves **86.81%** 🎉

---

## 📊 5. THE PROOF (Results & Experiments)

### Benchmarks:
- **Primary:** GSM8K (1,319 grade school math word problems)
- **Secondary:** AddSub, ASDiv, MultiArith, SingleOp, SingleEq, SVAMP

### Key Results (GSM8K pass@1):

| Model | Size | Accuracy |
|-------|------|----------|
| LLAMA-2 | 70B | 56.8% |
| WizardMath | 70B | 81.6% |
| MetaMath | 70B | 82.3% |
| GPT-3.5 | ??? | 77.4% |
| Gemini Pro | ??? | 86.5% (32 trials!) |
| **Orca-Math** | **7B** | **86.81%** (1 trial!) |
| Phi-GSM+V | 1.3B+1.3B | 81.5% (48 trials) |

### Most impressive result in plain English:
**A 7B model using only 200K training examples beats models 10x larger (70B), beats ChatGPT-3.5, and nearly matches Gemini Pro — all in a SINGLE attempt, with no code execution or verifiers.**

### Ablation highlights:
- SFT alone: 79.91% → SFT + KTO: 85.06% → SFT + KTO + KTO: **86.81%**
- **KTO consistently beats DPO** (85.06 vs 84.23 in iteration 2)
- **Model-generated positives matter:** removing them drops accuracy by ~2.3%
- **Synthetic negatives matter for DPO** (huge drop of 23.5 without them) but KTO is robust

### Limitations admitted:
- Only evaluated on grade school math (GSM8K-level), not harder competition math
- Relies on GPT-4-Turbo for data generation and evaluation (cost/access dependency)
- Contamination check found only 8 overlapping test questions (negligible)

---

## 🧩 6. KEY TERMS GLOSSARY

**SLM (Small Language Model)** → A language model with fewer parameters (e.g., 7B), small enough to run more cheaply than frontier models

**GSM8K** → A benchmark of 8,500 grade school math word problems (1,319 test problems)

**pass@1** → Accuracy when the model gets just ONE attempt per question (no retries)

**maj1@k** → Generate k responses, take the majority-voted answer as final

**verify_k@1** → Generate k responses, use a separate verifier model to pick the best

**SFT (Supervised Fine-Tuning)** → Training a model by showing it correct input-output examples

**DPO (Direct Preference Optimization)** → Training method that teaches a model to prefer good outputs over bad ones using paired comparisons

**KTO (Kahneman-Tversky Optimization)** → Like DPO but only needs binary "good/bad" labels, not pairs; inspired by prospect theory from behavioral economics

**Agent-Instruct** → The multi-agent pipeline used to generate the 200K training dataset

**Preference pairs** → A pair of (good answer, bad answer) for the same question, used to teach the model which is better

**Ensembling** → Running a model many times and combining answers to improve accuracy

**Mistral-7B** → An open-source 7B parameter language model used as the base model

**Seed set** → The initial collection of 36K existing math problems used as starting points

**Synthetic negatives** → Incorrect solutions borrowed from other questions when all student solutions happen to be correct

**Greedy decoding** → Generating text by always picking the most probable next token (deterministic)

**Top_p sampling** → Generating text by sampling from the smallest set of tokens whose cumulative probability exceeds p

**AutoGen** → Microsoft's framework for building multi-agent AI workflows

---

## 🔗 7. HOW IT CONNECTS

### Intellectual Family Tree:
```
GPT-4 (teacher model)
    │
    ├── Orca (2023) ─── progressive learning from GPT-4 traces
    │       │
    │       ├── Orca 2 (2023) ─── teaching SLMs HOW to reason
    │       │       │
    │       │       └── Orca-Math (THIS PAPER)
    │       │
    ├── WizardMath ─── reinforced evol-instruct for math
    ├── MetaMath ─── bootstrapping math questions
    ├── Phi-GSM ─── tiny models + code + massive data
    │
DPO (Rafailov et al.) ── preference learning
    │
    └── KTO (Ethayarajh et al.) ── binary feedback variant
```

### Who would use this:
- **EdTech companies** building math tutoring apps on-device
- **Researchers** studying data-efficient training of small models
- **Companies** wanting to deploy math-capable models at low cost
- **Anyone** building SLMs for reasoning tasks

### Future work enabled:
- **Iterative self-improvement** of SLMs beyond math (coding, science, logic)
- **Multi-agent data synthesis** as a general methodology
- **KTO as a preferred training method** over DPO for noisy preference data
- Extending to **harder math** (competition-level, MATH benchmark)

---

## ⚖️ 8. CRITICAL ANALYSIS

### Hidden assumptions:
- **GPT-4-Turbo is assumed to be correct.** Solutions are treated as ground truth, but GPT-4 makes math errors too. The DMath filtering (6,216 of 7,943 matched gold answers) suggests ~23% GPT-4 error rate!
- **The "Ask Me Anything" agent** generates problems with similar narratives to seeds — assumes narrative diversity doesn't matter much
- **Grade school math has unique numerical answers** — this approach may not transfer to open-ended or proof-based math

### Weaknesses not mentioned:
- **Heavy reliance on GPT-4** for data creation, solution generation, AND evaluation — the entire pipeline is bottlenecked by a closed-source model
- **Generalization unknown:** Only tested on GSM8K-style problems. Performance on MATH (competition level) or out-of-distribution problems is not reported
- **The evaluation itself uses GPT-4** — there's a circular dependency (trained on GPT-4 data, evaluated by GPT-4)
- **Contamination risk** from seed datasets: GSM8K train set is used as seeds → augmented versions might encode similar reasoning patterns
- **No error analysis:** Which types of problems does it still fail on? No breakdown by problem difficulty or number of reasoning steps

### Is the evaluation fair?
- ✅ They use pass@1 (strictest metric) while competitors use maj1@32 or verify100@1
- ✅ They perform contamination checks
- ⚠️ Using GPT-4 for evaluation instead of exact string matching could introduce bias
- ⚠️ Only one benchmark focused deeply (GSM8K), other benchmarks are secondary

### Would this work at scale?
- ✅ The model itself (7B) is practical to deploy
- ⚠️ The **training pipeline** requires extensive GPT-4 API calls (200K problems × solutions × evaluation × multiple iterations)
- ⚠️ Hard to replicate without access to GPT-4-Turbo-level models

---

## 📝 9. MEMORY ANCHORS

### Memorable metaphor:
**Orca-Math is like a student who first watches the teacher solve problems (SFT), then takes practice exams (sampling), gets each answer graded (GPT-4 feedback), and studies both their correct and incorrect answers (KTO) — repeating this cycle until they ace the test.**

### 3 bullets that capture 80% of the paper:
- 🐋 **200K high-quality synthetic math problems** created by AI agents that both diversify and increase difficulty of seed problems
- 🔄 **Iterative preference learning (SFT → KTO → KTO)** where the student model practices, gets feedback, and improves over 3 rounds
- 🏆 **7B model achieves 86.81% on GSM8K** — beating 70B models, GPT-3.5, and matching Gemini Pro, all in a single attempt with no tools

### Comprehension check question:
> *Why is it important that the preference dataset includes both model-generated positive solutions AND synthetic negatives (borrowed from other questions), rather than using only teacher solutions as positives?*

**Answer:** Model-generated positives teach the model that its OWN correct reasoning paths are valid (not just the teacher's), leading to +2.3% accuracy. Synthetic negatives ensure every question has contrastive signal even when the model already gets all attempts right, preventing ~80K questions from being wasted. Without synthetic negatives, DPO catastrophically fails (drops 23.5%).

---

## 🗺️ 10. VISUAL MENTAL MAP

```
╔══════════════════════════════════════════════════════════════════════╗
║                        ORCA-MATH PIPELINE                           ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  PROBLEM: Small models can't do math (need 34B+ for >80% GSM8K)     ║
║                              │                                       ║
║                              ▼                                       ║
║  ┌────────────── DATASET CREATION ──────────────┐                    ║
║  │  36K seed problems                            │                   ║
║  │      │                                        │                   ║
║  │      ├─→ "Ask Me Anything" Agent → 120K new   │                   ║
║  │      ├─→ Suggester + Editor Agents → 37K hard │                   ║
║  │      └─→ DMath → 6K verified                  │                   ║
║  │                                               │                   ║
║  │  = 200K problems + GPT-4 solutions            │                   ║
║  └───────────────────┬───────────────────────────┘                   ║
║                      │                                               ║
║                      ▼                                               ║
║  ┌─────────── ITERATION 1: LEARN ────────────┐                      ║
║  │  SFT on 200K (question, solution) pairs    │                      ║
║  │  → Model M1 (79.91%)                      │                      ║
║  └───────────────────┬────────────────────────┘                      ║
║                      │                                               ║
║                      ▼                                               ║
║  ┌─────────── ITERATION 2: PRACTICE ─────────┐                      ║
║  │  M1 generates 4 solutions per problem      │                      ║
║  │  GPT-4 grades: ✅ positive / ❌ negative    │                      ║
║  │  Build preference pairs                    │                      ║
║  │  Train with KTO → Model M2 (85.06%)        │                      ║
║  └───────────────────┬────────────────────────┘                      ║
║                      │                                               ║
║                      ▼                                               ║
║  ┌─────────── ITERATION 3: MASTER ───────────┐                      ║
║  │  M2 generates 4 solutions per problem      │                      ║
║  │  Same grading + preference construction    │                      ║
║  │  Train with KTO → Orca-Math (86.81%)  🏆  │                      ║
║  └────────────────────────────────────────────┘                      ║
║                                                                      ║
║  RESULT: 7B model > 70B models, GPT-3.5, ≈ Gemini Pro              ║
║          Using only 200K data, no tools, no ensembling               ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 🛠️ 11. IMPLEMENTATION SKETCH

### Pseudocode (Core Training Loop):
```python
# Phase 1: Dataset Creation
seed_problems = collect_from_11_datasets()           # 36K
expanded = ask_me_anything_agent(seed_problems)       # 120K
harder = suggester_editor_loop(seed_problems, rounds=2) # 37K
dmath = filter_dmath_with_gpt4()                      # 6K
dataset = expanded + harder + dmath                   # ~200K
solutions = gpt4_turbo.solve(dataset)                 # teacher solutions

# Phase 2: SFT (Iteration 1)
M1 = finetune(Mistral_7B, dataset, solutions, epochs=1)  # → 79.91%

# Phase 3: Iterative Preference Learning
for iteration in [2, 3]:
    model = M1 if iteration == 2 else M2
    for q in dataset:
        student_solutions = model.sample(q, n=4, top_p=0.95, temp=0.7)
        labels = gpt4.grade(student_solutions, solutions[q])  # +/-
    
    pref_pairs = build_preference_pairs(
        positives=student_correct + teacher_solution,
        negatives=student_wrong + synthetic_negatives_from_other_q
    )
    
    model = train_KTO(model, pref_pairs, beta=0.3, epochs=1)
    # Iter 2: M2 → 85.06% | Iter 3: Orca-Math → 86.81%
```

### Frameworks/Libraries Needed:
- **Transformers** (HuggingFace) for Mistral-7B
- **TRL** library for DPO/KTO training
- **AutoGen** (Microsoft) for multi-agent dataset generation
- **Azure OpenAI API** for GPT-4-Turbo calls
- **DeepSpeed/FSDP** for distributed training

### Estimated Compute Cost:
- **Dataset generation:** ~200K GPT-4-Turbo calls for solutions + ~800K for agent creation + ~1.6M for grading across iterations ≈ **$2,000-5,000 in API costs**
- **SFT Training:** 1 epoch on 64 A100 GPUs → **~2-4 hours**
- **KTO Training (per iteration):** Similar scale → **~2-4 hours**
- **Inference for sampling:** 200K × 4 solutions × 2 iterations → **~8-16 hours on 8 GPUs**
- **Total:** Roughly **$3,000-8,000 in compute** (affordable for a research lab, not a hobby project)
