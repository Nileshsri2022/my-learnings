

# WizardMath: Empowering Mathematical Reasoning for LLMs via Reinforced Evol-Instruct

---

## 🎯 1. THE ONE-LINER

**WizardMath teaches AI models to be much better at math by automatically creating harder and easier practice problems, then using a "coach" that checks every step of the work (not just the final answer) to help the AI learn from its mistakes.**

---

## 🤔 2. THE PROBLEM (Why does this paper exist?)

- **The real-world problem:** Open-source language models (like Llama, Mistral) are trained on general internet text and are **terrible at math**. Llama-2-7B only gets 14.6% on grade-school math (GSM8k) and 2.5% on high-school math (MATH). Meanwhile, closed-source models like GPT-4 cost money and aren't customizable.

- **Why should anyone care?** Imagine hiring a tutor who can write beautiful essays but can't solve "what's 15% of 80?" — that's what most open-source LLMs are like. Math reasoning is a **gateway skill** for science, engineering, finance, and everyday problem-solving.

- **Relatable analogy:** It's like a student who reads a lot of books but never practices math problems. They need (1) **a good problem set** with varying difficulty, and (2) **a teacher who checks each step**, not just whether the final answer is right.

- **Limitations of previous approaches:**
  - **WizardLM/WizardCoder:** Only focused on making instructions harder (SFT stage), and were prone to **learning wrong patterns** from the teacher model (hallucinated solutions)
  - **MetaMath:** Augmented questions from multiple perspectives but used simpler augmentation strategies
  - **Outcome-based reward models (ORM):** Only checked if the **final answer** was correct — a student could get the right answer by luck or flawed reasoning and still be "rewarded"
  - **Human-labeled process supervision (PRM800k):** Expensive, slow, requires expert annotators

---

## 💡 3. THE KEY IDEA (The "Aha!" moment)

The core insight is a **three-part recipe called RLEIF** (Reinforcement Learning from Evol-Instruct Feedback):

1. **Create a diverse math problem set** by evolving questions both **UP** (harder) and **DOWN** (easier) — not just harder like previous work
2. **Train TWO judges**: one that rates **question quality** (IRM) and one that grades **each reasoning step** (PRM) — both trained using GPT-4 labels instead of expensive human experts
3. **Use reinforcement learning (PPO)** where the reward = question quality × step-by-step correctness

### Everyday Analogy: The Math Dojo 🥋

Think of training a martial arts student:
- **Step 1 (Math Evol-Instruct):** Create a training curriculum with both beginner drills (downward evolution) AND advanced techniques (upward evolution) — you need BOTH to build mastery
- **Step 2 (IRM + PRM):** Hire two coaches — one who picks the **best training exercises** (IRM = "this drill is well-designed and challenging"), and one who watches **every move in real-time** and corrects form (PRM = "your second step was wrong")
- **Step 3 (PPO):** The student practices repeatedly, getting feedback from BOTH coaches simultaneously, with the combined score driving improvement

### The Key Trick:
**The final reward = r^q × r^a** (question quality × answer quality). This means the model is simultaneously pushed to (a) practice on **high-quality, well-defined** problems and (b) solve them with **correct reasoning at every step**.

---

## 🏗️ 4. HOW IT WORKS (The Method - Layer by Layer)

### Step 1: Math Evol-Instruct (Data Generation)
- **WHAT:** Take seed math problems from GSM8k and MATH training sets, then use GPT-4 to "evolve" them in two directions:
  - **Downward evolution** (2 rounds): Make problems easier — simplify variables, reduce constraints, create related but simpler problems
  - **Upward evolution** (3 rounds): Make problems harder — add constraints, concretize, increase reasoning steps
- **WHY:** Previous methods (WizardLM) only made things harder. But **covering the full difficulty spectrum** helps the model learn foundational concepts AND stretch to hard problems. Like learning to walk before you run.
- **Result:** Starting from ~7.5k seed problems → **418k unique evolved instructions** (after deduplication and contamination filtering). GPT-4 generates step-by-step answers for each.
- **HOW it connects to Step 2:** These evolved instructions become SFT training data AND the raw material for training reward models.

### Step 2: Train Two Reward Models (IRM + PRM)
- **WHAT (IRM):** An Instruction Reward Model that scores question quality on difficulty and definition clarity
  - GPT-4 ranks evolved instructions (e.g., "C > A = E > B > D")
  - Train IRM using **pairwise ranking loss**: L_IRM = -log σ(r^q_j - r^q_k - m)
  - This is like training a "curriculum designer" that picks the best homework problems
- **WHAT (PRM):** A Process-supervised Reward Model that scores **each individual reasoning step**
  - GPT-4 labels each step as correct (1), ambiguous (0), or wrong (-1)
  - Train PRM using **cross-entropy loss** on step-level labels
  - This is like training a "step-by-step grader"
- **WHY IRM:** Prevents evolved instructions from becoming gibberish or too easy — quality control for the curriculum
- **WHY PRM:** Catches **false positives** — solutions that reach the right answer through wrong reasoning (a critical flaw of outcome-only reward models)
- **HOW it connects to Step 3:** Both models provide reward signals during PPO training

### Step 3: Reinforcement Learning with PPO
- **WHAT:** Fine-tune the SFT model using Proximal Policy Optimization, guided by both reward models
  - For each (question q, generated answer a):
    - IRM gives **instruction reward r^q**
    - PRM gives step scores; the **minimum step score** = final answer reward **r^a**
    - **Final reward: r = r^q × r^a**
- **WHY minimum step score:** Using the MIN forces the model to get ALL steps right — one bad step tanks the whole reward. Like a chain being as strong as its weakest link.
- **WHY multiply:** Both signals must be high for a good reward. A great solution to a bad question, or a bad solution to a great question, both get low rewards.

### Step 4: PRM for Verification (Inference time)
- **WHAT:** At test time, generate N candidate solutions, then use PRM-weighted majority voting to pick the best answer
  - â = argmax_a Σ I(a_i = a) × PRM(q, a_i)
- **WHY:** Combines the wisdom of self-consistency (majority voting) with quality-weighted selection

---

## 📊 5. THE PROOF (Results & Experiments)

### Benchmarks:
- **GSM8k:** Grade school math (8.5k problems, ~2-8 step solutions)
- **MATH:** High school competition math (12.5k problems, 7 subtopics, much harder)
- **MWPBench:** 7 out-of-domain tasks (K-12, college, competition)

### Key Results (pass@1, greedy decoding, NO python tools):

| Model | GSM8k | MATH |
|-------|-------|------|
| Llama-2-7B (base) | 14.6 | 2.5 |
| **WizardMath-Llama-2 7B** | **84.1** | **43.5** |
| MetaMath-Mistral 7B | 77.9 | 28.6 |
| **WizardMath-Mistral 7B** | **90.7** | **55.4** |
| MetaMath-Llama-2 70B | 82.3 | 26.6 |
| **WizardMath-Llama-2 70B** | **92.8** | **58.6** |
| GPT-3.5-Turbo | 81.6 | 43.1 |
| GPT-4 (early/original) | 92.0 | 42.5 |

### Most impressive results in plain English:
- **WizardMath-Mistral 7B (a 7 billion parameter model) beats GPT-4-0314 on MATH** (55.4 vs 52.6) — a tiny open-source model beating a much larger proprietary one
- **WizardMath-70B surpasses GPT-3.5-Turbo by 11.2% on GSM8k and 15.5% on MATH**
- Even **GPT-2-XL (1.5B params!) with WizardMath reaches 58.9% on GSM8k**, matching Llama-2-70B's base performance
- **Data efficiency:** Math Evol-Instruct outperforms MetaMath by 15-20% on MATH at the same data size

### Ablation highlights:
- **PRM alone:** +3-4% over SFT baseline
- **PRM + IRM:** Additional +2.5-4% → **Total RL improvement: 6-8%**
- **AI-labeled PRM beats human-labeled PRM800k** by 1.8% on GSM8k and 1.9% on MATH
- **Downward evolution alone:** +14.8% GSM8k, +19.6% MATH over original data
- **All 5 rounds combined:** +21.5% GSM8k, +31.4% MATH over original

### Admitted limitations:
- Heavy reliance on **GPT-4 for data generation and labeling** (costly)
- Performance gap when using open-source models (Llama-3-70B) for evolution vs GPT-4 (~5-6%)
- Not tested on **competition-level** math beyond MATH benchmark topics

---

## 🧩 6. KEY TERMS GLOSSARY

- **LLM** → Large Language Model; an AI model with billions of parameters trained on text data
- **CoT (Chain-of-Thought)** → Making the model write out its reasoning step-by-step before giving an answer
- **SFT (Supervised Fine-Tuning)** → Training a model on question-answer pairs to learn specific skills
- **PPO (Proximal Policy Optimization)** → A reinforcement learning algorithm that updates the model's behavior in small, stable steps
- **RLEIF** → Reinforcement Learning from Evol-Instruct Feedback; the paper's main method combining evolved data + RL
- **Evol-Instruct** → A technique to automatically generate diverse instructions by evolving existing ones using an LLM
- **Downward Evolution** → Making math problems easier (simpler, fewer steps) — NEW in this paper
- **Upward Evolution** → Making math problems harder (more constraints, deeper reasoning)
- **IRM (Instruction Reward Model)** → A model that scores the quality/difficulty of a math question
- **PRM (Process-supervised Reward Model)** → A model that scores the correctness of EACH step in a solution
- **ORM (Outcome-supervised Reward Model)** → A model that only scores the final answer (not individual steps)
- **pass@1** → Accuracy when the model gets only one attempt (greedy decoding)
- **pass@N / Best-of-N** → Generate N solutions, pick the best one
- **Self-Consistency** → Generate multiple answers, pick the most common one (majority vote)
- **False Positive** → When a model gets the right final answer through wrong reasoning
- **Pairwise Ranking Loss** → Training a model to correctly order items (better vs. worse) rather than assigning absolute scores
- **GSM8k** → Grade School Math 8K; a benchmark of ~8.5k grade-school-level word problems
- **MATH** → A benchmark of 12.5k high-school competition math problems across 7 topics

---

## 🔗 7. HOW IT CONNECTS

### Intellectual Family Tree:
```
WizardLM (Xu et al., 2023)          Lightman et al., 2023
  ├── Evol-Instruct idea              ├── PRM800k (human-labeled)
  ├── In-depth/breadth evolving        ├── Process supervision > Outcome
  │                                    │
  └─────────────┬──────────────────────┘
                │
         WizardMath (THIS PAPER)
         ├── Math-specific Evol-Instruct (adds downward evolution)
         ├── AI-labeled PRM (replaces human labelers)
         ├── IRM (new: quality control for evolved questions)
         └── Combined IRM×PRM reward for PPO
                │
   Builds on: RLHF (Ouyang et al.), PPO, MetaMath, MAmmoTH
   Compared with: MetaMath, DART-Math, Xwin-Math, Math-Shepherd, Skywork-Math
```

### Who would use this?
- **ML researchers** building open-source math reasoning models
- **Education tech companies** wanting AI tutors that show correct work
- **Anyone fine-tuning LLMs** for domains requiring multi-step reasoning (science, coding, logic)

### Future work enabled:
- Extending RLEIF to **science** and **coding** domains
- Using **open-source models instead of GPT-4** for the full pipeline (they showed Llama-3-405B is ~1.4% behind GPT-4 for PRM labeling)
- Scaling process supervision to **more granular** step decomposition
- Combining with **tool-integrated reasoning** (code execution)

---

## ⚖️ 8. CRITICAL ANALYSIS

### Hidden assumptions:
- **GPT-4's judgments are reliable ground truth** for both instruction ranking and step-level correctness — but GPT-4 itself makes math errors
- **The minimum step score** adequately represents answer quality — but one borderline step could unfairly tank an otherwise good solution
- **Multiplicative reward (r^q × r^a)** is the right combination — why not additive, weighted sum, or other combinations? No ablation provided

### Weaknesses the authors DON'T emphasize:
- **Cost dependency on GPT-4:** The entire pipeline — evolution, answer generation, IRM labeling, PRM labeling — heavily uses GPT-4-0613. The API costs for 418k+ data points are substantial and not reported
- **Circular reasoning risk:** GPT-4 generates the training answers AND judges their correctness. If GPT-4 has systematic blind spots, these propagate
- **No error analysis by math topic difficulty:** Table 2 shows subtopic scores but doesn't analyze WHERE the method fails most or why (e.g., geometry at 48.3% vs algebra at 78.5%)
- **Reward model architecture not discussed:** They mention training Llama-2/Mistral as reward models but don't describe architectural modifications (linear head? token-level prediction?)
- **PPO hyperparameter sensitivity:** Only one set of hyperparameters reported; no sensitivity analysis

### Is the evaluation fair?
- **Mostly yes:** They use consistent evaluation (greedy decoding, CoT, no tools) and compare against many baselines
- **Good:** They check for data contamination explicitly
- **Concern:** The comparison with proprietary models uses different time snapshots (GPT-4 "early version" vs later versions), which isn't fully controlled
- **Strong addition in ICLR revision:** Added comparisons across many more base models (GPT-2 to Qwen2.5) and showed IRM+PRM works on other SFT backbones (DART-Math, Xwin-Math)

### Would this work at scale in the real world?
- **Yes, with caveats.** The method is model-agnostic and works from 100M to 70B parameters
- **The GPT-4 dependency is the bottleneck.** They show Llama-3.1-405B can partially substitute, but there's still a quality gap
- **RL training stability** with PPO is notoriously finicky — reproducibility may be challenging

---

## 📝 9. MEMORY ANCHORS

### Memorable metaphor:
> **WizardMath is like a martial arts training system:** it creates a curriculum with both basic and advanced moves (Evol-Instruct), hires a curriculum designer (IRM) and a form coach (PRM), then has the student spar repeatedly with real-time feedback from both coaches (PPO). The result: a small fighter that can beat much bigger opponents.

### 3 bullet points that capture 80% of the paper:
- 📚 **Math Evol-Instruct** generates diverse training data by evolving problems both easier (downward) AND harder (upward), producing 418k instructions from just 7.5k seeds
- 🔍 **Two AI-trained reward models** — IRM (rates question quality) and PRM (grades each solution step) — replace expensive human annotation and catch "right answer, wrong reasoning" errors
- 🚀 **Combined IRM×PRM reward in PPO** yields 6-8% improvement over SFT alone, enabling a 7B model to beat GPT-4 (early) on MATH

### One question to test understanding:
> *Why does WizardMath use the MINIMUM step score from PRM as the answer reward (r^a) rather than the average or final step score?*

**Answer:** Because using the minimum forces the model to get ALL steps correct — one wrong step in the middle means the whole chain of reasoning is unreliable. The average would dilute the penalty for a single critical error, and the final step might be correct even if intermediate steps were wrong (false positive). The min is the strictest criterion, like saying "a proof is only as valid as its weakest logical step."

---

## 🗺️ 10. VISUAL MENTAL MAP

```
┌─────────────────────────────────────────────────────────────────┐
│                        PROBLEM                                   │
│  Open-source LLMs can't do math (Llama-2-7B: 14.6% GSM8k)      │
│  Previous methods: only harder data OR only outcome rewards      │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    MATH EVOL-INSTRUCT                             │
│                                                                   │
│   Seed Data (7.5k)──►  ↑ Upward (3 rounds): harder              │
│   GSM8k + MATH     ──►  ↓ Downward (2 rounds): easier           │
│                          │                                        │
│                     418k evolved instructions                     │
│                     + GPT-4 step-by-step answers                  │
└──────────┬──────────────────────────────┬───────────────────────┘
           │                              │
           ▼                              ▼
┌──────────────────┐          ┌──────────────────────────┐
│   SFT TRAINING   │          │   REWARD MODEL TRAINING  │
│                  │          │                          │
│ Fine-tune LLM on │          │  IRM: ranks question     │
│ 418k (q, a) pairs│          │  quality (GPT-4 labels)  │
│                  │          │                          │
│  → SFT Model     │          │  PRM: scores each step   │
│                  │          │  correctness (GPT-4)     │
└────────┬─────────┘          └────────────┬─────────────┘
         │                                 │
         └───────────────┬─────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│              REINFORCEMENT LEARNING (PPO)                         │
│                                                                   │
│  For each (question q, generated answer a):                       │
│                                                                   │
│    r^q = IRM(q)           ← "Is this a good question?"           │
│    r^a = min(PRM(steps))  ← "Is every step correct?"             │
│    r   = r^q × r^a        ← Combined reward                      │
│                                                                   │
│    Update model via PPO using reward r                            │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                        RESULTS                                    │
│                                                                   │
│  WizardMath-Mistral 7B:  GSM8k 90.7% | MATH 55.4%               │
│  WizardMath-Llama 70B:   GSM8k 92.8% | MATH 58.6%               │
│                                                                   │
│  Beats: GPT-3.5-Turbo, Claude-2, Gemini Pro, GPT-4 (early)      │
│  Beats: All open-source models at same size                       │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🛠️ 11. IMPLEMENTATION SKETCH

### Pseudocode (Core Pipeline):

```python
# PHASE 1: Data Generation
seed_data = load(GSM8k_train + MATH_train)  # ~7.5k problems
evolved_data = []
for q in seed_data:
    for round in range(5):  # 2 down + 3 up
        if round < 2:
            q_new = GPT4.evolve_downward(q)
        else:
            q_new = GPT4.evolve_upward(q)
        evolved_data.append(q_new)
        q = q_new
evolved_data = deduplicate(evolved_data)  # → 418k
answers = [GPT4.solve_stepwise(q) for q in evolved_data]

# PHASE 2: SFT
sft_model = finetune(base_llm, zip(evolved_data, answers), 
                      epochs=3, lr=2e-5)

# PHASE 3: Train Reward Models
# IRM training data
irm_rankings = GPT4.rank_quality(evolved_instruction_groups)
irm = train_ranking_model(sft_model, irm_rankings, 
                           loss=pairwise_ranking)

# PRM training data  
for q in instructions_90k:
    solutions = [sft_model.generate(q) for _ in range(5)]
    step_labels = GPT4.label_steps(q, solutions)
prm = train_classifier(sft_model, step_labels, 
                        loss=cross_entropy)

# PHASE 4: PPO Training
policy = sft_model.copy()
for q in training_questions:
    answer = policy.generate(q)
    r_q = irm.score(q)
    r_a = min([prm.score(step) for step in answer.steps])
    reward = r_q * r_a
    policy.ppo_update(q, answer, reward, lr=4e-7)
```

### Frameworks/Libraries:
- **DeepSpeed / FSDP** — for distributed training of 7B-70B models
- **TRL (Transformer Reinforcement Learning)** — for PPO implementation
- **HuggingFace Transformers** — model loading and fine-tuning
- **vLLM** — for fast inference during data generation
- **OpenAI API** — for GPT-4 calls (evolution, ranking, step labeling)

### Estimated Compute Cost:
- **GPT-4 API calls:** ~500k+ calls for evolution + answers + ranking + step labeling → **$5,000-$15,000+ in API costs**
- **SFT training (70B):** ~64-128 A100 GPUs × 24-48 hours
- **SFT training (7B):** ~8 A100 GPUs × 12-24 hours
- **Reward model training:** Similar to SFT but 1 epoch
- **PPO training:** 1.5-2× the compute of SFT (reference model + policy model + reward models)
- **Total estimated cost:** Likely **$20,000-$50,000+** including compute and API fees for the full 70B pipeline
