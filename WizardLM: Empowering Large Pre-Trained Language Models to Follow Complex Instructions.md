

# WizardLM: Empowering Large Pre-Trained Language Models to Follow Complex Instructions

Published at **ICLR 2024** by Microsoft & Peking University

---

## 🎯 1. THE ONE-LINER
**Instead of paying humans to write harder and harder homework problems for AI, they taught an AI to automatically make easy problems harder — and the AI that practiced on those harder problems became much smarter.**

---

## 🤔 2. THE PROBLEM (Why does this paper exist?)

- **The problem:** To make LLMs good at following instructions, you need tons of diverse, high-quality instruction data. Hiring humans to write this data is **extremely expensive** and **time-consuming**.
- **Why should anyone care?** Imagine you're training for a math competition. If your practice problems are all "1+1=?", you'll never get better. You need progressively harder problems. But **humans are bad at consistently writing really hard problems** — they get tired, and most of what they write tends to be easy or medium difficulty.
- **Limitations of previous approaches:**
  - **Self-Instruct (Alpaca):** Generated instructions from seed examples, but **couldn't control difficulty** — everything stayed at roughly the same level
  - **Human-written data (Vicuna/ShareGPT):** Expensive, skewed toward easy instructions (see Figure 5a: ShareGPT avg difficulty ~4.6/10, Alpaca ~3.0/10)
  - **Closed-domain instructions (FLAN, T0):** Only cover NLP tasks, not real-world user needs; all samples in a dataset share the same few instructions

---

## 💡 3. THE KEY IDEA (The "Aha!" moment)

**Core insight:** You can use an LLM (ChatGPT) as a "question upgrader" — give it a simple question and specific prompts, and it will rewrite that question to be harder, deeper, or more constrained. Do this **repeatedly over multiple rounds**, and you get a dataset that spans from easy to very hard.

**Cooking analogy:** Imagine you have a basic recipe for "scrambled eggs." An expert chef (ChatGPT) can evolve it:
- **Add constraints:** "Make scrambled eggs, but only using one hand and no salt"
- **Deepen:** "Explain the Maillard reaction occurring in scrambled eggs"
- **Concretize:** "Make scrambled eggs using duck eggs on a cast-iron skillet at exactly 325°F"
- **Add reasoning:** "Calculate the optimal cook time for scrambled eggs at different altitudes"
- **Complicate input:** "Given this nutritional data table, make scrambled eggs meeting these macros"
- **Broaden (mutate):** "Now create a completely different breakfast dish using similar techniques"

Repeat this 4 times, and your simple recipe becomes a culinary masterclass.

**The method is called Evol-Instruct** and has two directions:

```
                    ┌─ Add Constraints
                    ├─ Deepening
  In-Depth ────────┼─ Concretizing          (make it HARDER)
  Evolving          ├─ Increase Reasoning
                    └─ Complicate Input

  In-Breadth ──────── Mutation               (make it DIFFERENT)
  Evolving
```

---

## 🏗️ 4. HOW IT WORKS (The Method - Layer by Layer)

### Step 1: Start with Seed Instructions
- **WHAT:** Take an existing dataset (Alpaca's 52k instructions, generated via Self-Instruct)
- **WHY:** Provides a starting point with minimal human involvement (only 175 human seed instructions)
- **HOW:** These become the initial instruction pool D⁰

### Step 2: Evolve Instructions (Instruction Evolver)
- **WHAT:** For each instruction, **randomly pick 1 of 6 evolution operations** (5 in-depth + 1 in-breadth) with equal probability. Feed the instruction + evolution prompt to ChatGPT.
- **WHY:** Creates progressively harder AND more diverse instructions
- **HOW:** The prompts tell ChatGPT things like *"rewrite this to make it a bit harder for AI systems, but still answerable by humans"* with constraints like **"only add 10-20 words"** to prevent runaway complexity
- **Key detail:** Each evolution only makes it "a bit harder" — **gradual difficulty increase** prevents the dataset from being all super-hard problems

### Step 3: Generate Responses
- **WHAT:** Use the same ChatGPT to generate responses for the evolved instructions
- **WHY:** Need (instruction, response) pairs for fine-tuning
- **HOW:** Simply feed the evolved instruction to ChatGPT and collect its answer

### Step 4: Eliminate Failed Evolutions (Instruction Eliminator)
- **WHAT:** Filter out bad evolutions based on 4 rules:
  1. No information gain (instruction didn't actually change meaningfully)
  2. LLM can't respond (response contains "sorry" + <80 words)
  3. Response is just punctuation/stop words
  4. Instruction copies prompt artifacts (e.g., "#Rewritten Prompt#")
- **WHY:** LLM-generated evolutions sometimes fail — need quality control
- **HOW:** Failed instructions go back into the pool for another attempt next round

### Step 5: Repeat for M=4 Rounds
- **WHAT:** Run the evolution loop 4 times, producing ~250k total instructions
- **WHY:** Each round increases average difficulty (Round 1: 5.48 → Round 4: 7.08 on 1-10 scale)
- **HOW:** After all rounds, merge all instructions from all epochs

### Step 6: Fine-tune LLaMA
- **WHAT:** Sample 70k instructions from the merged pool, fine-tune LLaMA 13B
- **WHY:** Fair comparison with Vicuna (also trained on 70k)
- **HOW:** Standard supervised fine-tuning with Adam optimizer, lr=2×10⁻⁵, 3 epochs on 8 V100 GPUs (140 hours)

---

## 📊 5. THE PROOF (Results & Experiments)

### Benchmarks Used (9 total):
| Benchmark | What it tests |
|-----------|--------------|
| MMLU | Academic multiple-choice questions |
| ARC | Grade-school science |
| HellaSwag | Commonsense inference |
| TruthfulQA | Factual accuracy |
| HumanEval | Code generation (164 problems) |
| GSM8k | Grade-school math |
| AlpacaEval | GPT-4 judged instruction following |
| MT-Bench | Multi-turn conversation quality |
| WizardEval | Their own difficulty-balanced test set (218 problems, 29 skills) |

### Key Results (WizardLM-13B vs. baselines):

| Metric | Alpaca-13B | Vicuna-13B | **WizardLM-13B** | ChatGPT |
|--------|-----------|-----------|-----------------|---------|
| **Average** | 43.44 | 54.60 | **58.96** | 76.15 |
| HumanEval (code) | 9.2 | 12.5 | **24.0** | 48.1 |
| GSM8k (math) | 8.35 | 24.34 | **37.15** | 80.8 |
| AlpacaEval | 33.25 | 70.43 | **75.31** | 89.37 |
| MT-Bench | 4.78 | 6.21 | **6.35** | 7.94 |

### Most impressive results in plain English:
- **WizardLM beats Vicuna (human-curated data) across almost all benchmarks** — despite using NO human-written instructions
- **Code: 24.0 vs 12.5** (nearly 2x Vicuna on HumanEval)
- **Math: 37.15 vs 24.34** (+53% over Vicuna on GSM8k)
- **Human eval:** WizardLM wins against Alpaca 116/218 times, against Vicuna 90/218 times
- **Scaling works:** WizardLM-70B achieves **71.33 avg** (AlpacaEval: 89.32, near ChatGPT's 89.37)
- **Each evolution round improves performance:** C0→C4 average scores: 41.25 → 48.39 → 53.83 → 55.75 → 57.61

### Admitted Limitations:
- GPT-4 and human evaluation methods have **scalability and reliability challenges**
- Test set may not represent all scenarios/domains
- Dependent on the quality of the evolving LLM (though they show LLaMA-2-70B works as substitute)

---

## 🧩 6. KEY TERMS GLOSSARY

- **Evol-Instruct** → The proposed method that uses an LLM to iteratively make instructions harder and more diverse
- **In-Depth Evolving** → Making an instruction more complex (5 ways: add constraints, deepen, concretize, more reasoning, complicate input)
- **In-Breadth Evolving** → Creating a completely new but related instruction (mutation) to increase diversity
- **Elimination Evolving** → Quality filter that removes failed evolution attempts
- **Instruction Pool** → The growing collection of instructions at various difficulty levels
- **Instruction Evolver** → The LLM (ChatGPT) that performs the evolution by following specific prompts
- **Instruction Eliminator** → The filter component that catches bad/failed evolutions
- **Self-Instruct** → Previous method (used by Alpaca) to generate instructions from seed examples — no difficulty control
- **Instruction Tuning** → Fine-tuning an LLM on (instruction, response) pairs so it follows instructions better
- **Open-domain Instructions** → Real-world, diverse instructions (vs. closed-domain = specific NLP tasks)
- **LLaMA** → Meta's open-source base language model used as the foundation
- **WizardEval** → Their custom test set with 218 instructions, 29 skills, balanced difficulty
- **ShareGPT** → Dataset of real user conversations with ChatGPT, used by Vicuna
- **pass@1** → Success rate of generating correct code/answer on the first try
- **t-SNE** → Visualization technique to show how spread out (diverse) data points are
- **Kappa score** → Measure of agreement between annotators (>0.6 is good)

---

## 🔗 7. HOW IT CONNECTS

### Intellectual Family Tree:
```
GPT-3 (Brown 2020)
    └── InstructGPT (Ouyang 2022) ← human-written instructions
         ├── ChatGPT
         ├── Self-Instruct (Wang 2022) ← auto-generate instructions
         │    └── Alpaca (Taori 2023) ← applied Self-Instruct to LLaMA
         ├── ShareGPT ← crowd-sourced user data
         │    └── Vicuna (Chiang 2023) ← fine-tuned on ShareGPT
         └── **Evol-Instruct (this paper)** ← EVOLVE instructions iteratively
              └── WizardLM ← fine-tuned on evolved data
```

### Comparison with 2 related papers:
1. **Self-Instruct (Wang et al., 2022):** Generates new instructions from seeds but **can't control complexity**. Evol-Instruct explicitly evolves difficulty upward.
2. **Orca (Mukherjee et al., 2023):** Learns from complex reasoning traces of GPT-4 (focuses on **response quality**). Evol-Instruct focuses on **instruction quality/difficulty**.

### Who would use this?
- **ML researchers** building instruction-following models without expensive human annotation
- **Companies** wanting to create specialized training data cheaply
- **Anyone fine-tuning open-source LLMs** who needs diverse, hard training examples

### Future work enabled:
- Domain-specific Evol-Instruct (code, math, medicine)
- Self-evolving models that create their own training data
- Quality-aware curriculum learning for LLMs
- **WizardCoder** and **WizardMath** (already published follow-ups!)

---

## ⚖️ 8. CRITICAL ANALYSIS

### Hidden Assumptions:
- **ChatGPT is a good judge of difficulty** — they use it to both create and evaluate instruction complexity. This is circular.
- **Harder instructions = better training** — assumes difficulty is always beneficial, but there's a ceiling where noise dominates
- **The evolving LLM won't introduce systematic biases** — ChatGPT's biases get baked into the evolved data

### Weaknesses NOT mentioned:
- **Data contamination risk:** ChatGPT may have seen benchmarks like MMLU/HumanEval during training, and evolved instructions might overlap with test sets
- **Factual correctness of responses:** They generate responses with ChatGPT but **never verify their correctness** — evolved hard questions may get wrong answers
- **Cost isn't really "low":** 624k API calls to ChatGPT isn't cheap; they don't report the dollar cost
- **Evolution may collapse diversity:** Despite in-breadth evolving, all data still originates from 52k Alpaca seeds — fundamental topic coverage is limited by the seed
- **No analysis of evolved instruction quality** beyond difficulty scores — are the evolved instructions actually coherent and useful?

### Is evaluation fair?
- **WizardEval is their own test set** — potential selection bias
- They compare 13B models fairly (same data size, same base model) ✓
- Using GPT-4 as judge is common but **known to have biases** (e.g., preferring longer responses)
- Human evaluation Kappa >0.6 is decent but not excellent

### Real-world scalability:
- ✅ Method is simple and scalable (just API calls + prompts)
- ✅ Works with open-source evolving models (LLaMA-2-70B)
- ⚠️ Quality degrades somewhat with weaker evolving models
- ⚠️ No guarantee that more evolution rounds keep helping (they only tested 4)

---

## 📝 9. MEMORY ANCHORS

### Memorable Metaphor:
> **Evol-Instruct is like a "homework leveling-up machine."** You feed it easy worksheets, and it keeps rewriting them into harder versions — like a video game that auto-generates progressively tougher levels. Train on those tougher levels, and your AI character levels up faster than one that only practiced on easy mode.

### 3 Bullets That Capture 80%:
- 📈 **Use an LLM to iteratively evolve simple instructions into harder ones** via 6 operations (5 depth + 1 breadth), running 4 rounds
- 🧹 **Filter out failed evolutions** using simple heuristics (no info gain, can't respond, garbage output, prompt leakage)
- 🏆 **Fine-tuning LLaMA on 70k evolved instructions beats models trained on 70k human-written instructions** (Vicuna) across 9 benchmarks

### Comprehension Check Question:
> *Why does Evol-Instruct limit each evolution step to only add 10-20 words and make it just "a bit harder," rather than maximizing difficulty in one step?*

**Answer:** To avoid filling the dataset with only extremely hard instructions, which would hurt generalization. Gradual difficulty increase ensures a smooth distribution from easy to hard, enabling better fine-tuning.

---

## 🗺️ 10. VISUAL MENTAL MAP

```
PROBLEM                          METHOD                              RESULT
═══════                          ══════                              ══════

Humans can't              ┌─────────────────────┐
scale instruction    ──►  │  Start: 52k Alpaca   │
writing                   │  seed instructions   │
                          └────────┬────────────┘
Human data is                     │
skewed easy             ┌─────────▼──────────┐
                        │  EVOLUTION ROUND    │◄──── Repeat 4x
                        │  ┌───────────────┐  │
                        │  │ Pick 1 of 6   │  │
                        │  │ evol prompts  │  │
                        │  └───────┬───────┘  │
                        │          ▼          │
                        │  ┌───────────────┐  │
                        │  │ ChatGPT       │  │      WizardLM-13B
                        │  │ evolves instr │  │      beats:
                        │  └───────┬───────┘  │      ✓ Alpaca
                        │          ▼          │      ✓ Vicuna
                        │  ┌───────────────┐  │      ✓ Baize
                        │  │ Generate      │  │      ✓ CAMEL
                        │  │ response      │  │      ✓ Tulu
                        │  └───────┬───────┘  │
                        │          ▼          │      On 9 benchmarks
                        │  ┌───────────────┐  │      including code,
                        │  │ Eliminate     │  │      math, GPT-4 eval
                        │  │ failures     │  │
                        │  └───────┬───────┘  │      Key: +160% code
                        └─────────┬──────────┘       +53% math
                                  ▼                  vs Vicuna
                        ┌─────────────────────┐
                        │  250k instructions   │
                        │  (sample 70k)       │
                        └─────────┬───────────┘
                                  ▼
                        ┌─────────────────────┐
                        │  Fine-tune LLaMA    │──►  WizardLM 🧙
                        │  13B / 65B / 70B    │
                        └─────────────────────┘
```

---

## 🛠️ 11. IMPLEMENTATION SKETCH

### Pseudocode (~15 lines):
```python
def evol_instruct(seed_instructions, M=4, evolver_llm="gpt-3.5-turbo"):
    pool = seed_instructions  # D⁰: list of (instruction, response)
    all_data = list(pool)
    
    for round in range(M):
        new_pool = []
        for instruction in pool:
            # Step 1: Pick random evolution type
            evol_type = random.choice([
                "add_constraints", "deepening", "concretizing",
                "increase_reasoning", "complicate_input", "in_breadth"
            ])
            # Step 2: Evolve instruction
            evol_prompt = get_evol_prompt(evol_type, instruction)
            evolved_instr = evolver_llm.generate(evol_prompt)
            
            # Step 3: Eliminate failures
            if not passes_elimination(evolved_instr, instruction, evolver_llm):
                new_pool.append(instruction)  # keep original for retry
                continue
            
            # Step 4: Generate response
            response = evolver_llm.generate(evolved_instr)
            new_pool.append(evolved_instr)
            all_data.append((evolved_instr, response))
        
        pool = new_pool
    
    # Step 5: Sample & fine-tune
    train_data = random.sample(all_data, k=70000)
    model = finetune(LLaMA_13B, train_data, lr=2e-5, epochs=3)
    return model
```

### Frameworks/Libraries Needed:
- **OpenAI API** (or any strong LLM API) for evolution & response generation
- **HuggingFace Transformers** for LLaMA model loading
- **DeepSpeed Zero-3** for distributed training
- **PyTorch** as backend

### Estimated Compute Cost:
- **API calls:** 624k calls to GPT-3.5-turbo ≈ **$300-600** (at ~$0.001/call in 2023)
- **Fine-tuning:** 8× V100 GPUs × 140 hours ≈ **$500-1000** on cloud
- **Total reproduction cost:** ~**$1,000-1,500** (very affordable for an ICLR paper)
- For the 70B model: significantly more GPU hours for fine-tuning
