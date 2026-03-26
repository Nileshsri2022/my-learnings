

# How Far Can Camels Go? Exploring the State of Instruction Tuning on Open Resources

---

## 🎯 1. THE ONE-LINER
This paper tests **12 different "teaching" datasets** for making AI chatbots smarter, finds that **no single dataset is best at everything**, and builds **TÜLU** — a hybrid model trained on the best mix — while showing it still can't beat ChatGPT.

---

## 🤔 2. THE PROBLEM (Why does this paper exist?)

- **The problem:** Everyone in 2023 was releasing "open-source ChatGPT killers" (Alpaca, Vicuna, etc.) claiming they matched ChatGPT, but **these claims were based on flimsy, narrow evaluations** — often just asking GPT-4 "which response do you like better?"
- **Why should anyone care?** Imagine 12 different tutoring companies all claim their method makes kids the smartest. But each one only tested on *one subject*. You'd want someone to **test all 12 methods across math, reading, science, and more** — that's exactly what this paper does for AI instruction datasets.
- **Limitations of previous approaches:**
  - Most evaluations only used **model-based preference** (asking GPT-4 to judge) — which is biased toward **longer, fancier-sounding answers**
  - No systematic comparison across datasets using the **same base model and same evaluation suite**
  - Claims of matching ChatGPT were **not backed by rigorous benchmarks** testing reasoning, coding, multilinguality, etc.

---

## 💡 3. THE KEY IDEA (The "Aha!" moment)

- **Core insight:** Different instruction datasets are like **different workout routines** — a coding dataset makes your model great at coding but might hurt its math reasoning. **No single dataset wins everywhere**, but a smart **combination** (mixing datasets) gives the best *average* performance.
- **Everyday analogy:** Think of it like cross-training in sports. If you only do sprints, you'll be fast but terrible at endurance. If you only lift weights, you'll be strong but slow. The best overall athlete **cross-trains** — and that's what TÜLU does by mixing human-written, GPT-generated, code, reasoning, and conversation data.
- **Second key insight:** GPT-4's evaluation of model quality is **strongly correlated (r=0.96!) with how many unique words a model generates**, not necessarily how correct it is. Models that produce verbose answers "win" in GPT-4 evaluation, even if they're not smarter.

---

## 🏗️ 4. HOW IT WORKS (The Method - Layer by Layer)

### Step 1: Collect 12 Diverse Instruction Datasets
- **WHAT:** Gather datasets spanning human-written (Dolly, Open Assistant), researcher-curated NLP tasks (FLAN V2, SuperNI), GPT-distilled (Alpaca, ShareGPT), and skill-specific (CoT for reasoning, Code-Alpaca for code)
- **WHY:** To have a representative sample of all the different *types* of instruction data available
- **HOW it connects:** These become the independent variables — each one trains a separate model

### Step 2: Unify Format into Chat Template
- **WHAT:** Convert all datasets into a standardized `<|user|>` / `<|assistant|>` multi-turn chat format
- **WHY:** Different datasets have wildly different formats; a unified format ensures **fair comparison** and enables multi-turn conversations
- **HOW it connects:** This format feeds directly into the training pipeline

### Step 3: Train Models with Loss Masking
- **WHAT:** Fine-tune LLaMA models (7B–65B) using teacher-forcing, but **only compute loss on the assistant's response tokens** (masking user input tokens)
- **WHY:** The model should learn to *generate good responses*, not memorize the user's question
- **HOW it connects:** This produces one model per dataset, all comparable because they share the same base model and training recipe

### Step 4: Create Mixture Datasets (TÜLU)
- **WHAT:** Build two mixtures: (a) **Human mix** (FLAN V2 + CoT + Dolly + Open Assistant); (b) **Human+GPT mix** (adds GPT4-Alpaca + Code-Alpaca + ShareGPT)
- **WHY:** Hypothesis: diversity of training data → better generalization
- **HOW it connects:** Models trained on the Human+GPT mix become **TÜLU** (named after a hybrid camel species)

### Step 5: Multi-Faceted Evaluation
- **WHAT:** Evaluate ALL models across 6 capability axes + 2 safety benchmarks + model-based + human evaluation
- **WHY:** To expose which datasets help which capabilities, and to test whether model-based evaluation (GPT-4 judging) agrees with actual benchmark performance
- **HOW it connects:** Produces the comprehensive results tables that are the paper's main contribution

---

## 📊 5. THE PROOF (Results & Experiments)

### Benchmarks Used:
| Benchmark | What it Tests |
|-----------|--------------|
| **MMLU** | Factual knowledge (57 subjects) |
| **GSM** | Math reasoning (grade-school math) |
| **BBH** | General reasoning (23 hard tasks) |
| **TyDiQA** | Multilingual QA (11 languages) |
| **HumanEval (Codex-Eval)** | Code generation |
| **AlpacaEval** | Open-ended instruction following (GPT-4 judge) |
| **ToxiGen** | Toxic language generation |
| **TruthfulQA** | Avoiding misinformation |

### Key Numbers:
- **TÜLU 65B average: 56.7%** across benchmarks vs **ChatGPT: 72.3%** vs **GPT-4: 86.9%**
- Best open model reaches on average **87% of ChatGPT** and **73% of GPT-4** performance
- **CoT dataset → GSM jumps from 14.5% to 40.0%** (13B LLaMA); domain-specific data massively helps
- **Code-Alpaca → Codex-Eval jumps from 28.6% to 34.2%**
- **AlpacaEval win-rate correlates with unique token count at r=0.96** — a damning finding about GPT-4 evaluation bias
- **LLaMA-2 7B + TÜLU mix = 45.7% avg**, beating LLaMA-1 13B + TÜLU = 45.2% — **base model quality matters enormously**
- Human evaluation acceptance: ChatGPT 90.1% vs TÜLU 65B 79.8%

### Most Impressive Result in Plain English:
**Simply switching from LLaMA-1 to LLaMA-2 (same size, better pretraining) improved TÜLU performance more than doubling the model size** — proving base model quality is king.

### Failure Cases / Limitations Admitted:
- TÜLU still **significantly lags ChatGPT and GPT-4** in all settings
- Some datasets **hurt** vanilla model performance (especially on GSM and TyDiQA) — instruction tuning can cause **catastrophic forgetting**
- TruthfulQA **doesn't scale** with model size — larger models hedge more
- No evaluation of multi-turn dialogue or summarization
- Possible data contamination in ChatGPT/GPT-4's training data

---

## 🧩 6. KEY TERMS GLOSSARY

- **Instruction Tuning** → Fine-tuning a pretrained LLM on (instruction, response) pairs so it follows user requests
- **LLaMA** → Meta's open-source family of pretrained language models (6.7B–65B parameters)
- **TÜLU** → The authors' best model, named after a hybrid camel; trained on a mixture of 7 datasets
- **MMLU** → Multiple-choice test covering 57 academic subjects to measure knowledge
- **GSM (Grade School Math)** → Math word problems testing step-by-step reasoning
- **BBH (Big-Bench Hard)** → 23 especially challenging reasoning tasks
- **TyDiQA** → Multilingual question answering across 11 diverse languages
- **HumanEval / Codex-Eval** → Benchmark where models write Python code from descriptions
- **AlpacaEval** → Evaluation where GPT-4 judges which model's response is better vs Davinci-003
- **ToxiGen** → Dataset measuring how much a model generates toxic/hateful text
- **TruthfulQA** → Benchmark measuring whether models avoid common misconceptions
- **Chain-of-Thought (CoT)** → Technique where the model shows its reasoning step-by-step before answering
- **Distillation** → Creating training data by having a stronger model (e.g., GPT-4) generate examples
- **Loss Masking** → Only computing training loss on the model's response, not the user's input
- **Teacher Forcing** → Training technique where the model is given the correct previous tokens when predicting the next one
- **DeepSpeed / ZeRO** → Libraries for efficiently training very large models across multiple GPUs
- **Win Rate** → Percentage of times GPT-4 prefers a model's response over a baseline
- **Pass@k** → Probability that at least one of k generated code samples passes all test cases
- **Few-shot** → Providing a few example input-output pairs in the prompt to guide the model

---

## 🔗 7. HOW IT CONNECTS

### Intellectual Family Tree:
```
FLAN / T0 (2022)              Self-Instruct (2022)
[task diversity matters]       [LLMs can generate own training data]
        ↓                              ↓
SuperNI (2022)                 Alpaca / Vicuna (2023)
[1600+ NLP tasks]              [distill from ChatGPT/GPT-4]
        ↓                              ↓
        └──────────> THIS PAPER <──────┘
                         ↓
                    TÜLU model
                         ↓
              TÜLU 2 (follow-up work)
```

### Related Papers:
- **LIMA (Zhou et al., 2023)**: Argued "Less is More" — 1,000 high-quality examples suffice. This paper shows that while quality matters, **diversity across skills also matters**.
- **QLoRA (Dettmers et al., 2023)**: Also explored instruction tuning across datasets but with quantized, parameter-efficient training. This paper does **full fine-tuning** and covers a **broader evaluation**.
- **The False Promise of Imitating Proprietary LLMs (Gudibande et al., 2023)**: Argued imitation models only learn style, not substance. This paper partially agrees (gap to ChatGPT remains) but shows **diverse imitation data does improve actual capabilities**.

### Who Would Use This:
- **Researchers** building open instruction-tuned models
- **Practitioners** deciding which datasets to use for fine-tuning
- **Evaluation researchers** designing better benchmarks
- **Companies** building chatbots wanting to know where open models stand vs proprietary ones

### Future Work Enabled:
- Better **dataset mixing strategies** (weighted sampling, curriculum learning)
- **Mixture-of-experts** architectures for retaining multi-skill performance
- Better evaluation protocols that go beyond GPT-4 preference
- Development of **stronger open base models** (which they show matters most)

---

## ⚖️ 8. CRITICAL ANALYSIS

### Hidden Assumptions:
- Assumes the **6 benchmark axes** adequately represent "general intelligence" — but misses summarization, multi-turn dialogue, creative writing, etc.
- Assumes **simple concatenation** of datasets is a reasonable mixing strategy (no sophisticated sampling/weighting)
- Assumes LLaMA is representative enough that findings generalize to other architectures

### Weaknesses the Authors DON'T Mention:
- **No investigation of data quality within datasets** — treating all examples in a dataset as equally valuable
- **No ablation on dataset SIZE** — is CoT better for reasoning because of *what* it contains or simply because it has 100K reasoning examples? They don't downsample to control for this
- **Evaluation contamination risk goes both ways** — their models may also have seen evaluation data during pretraining (LLaMA was trained on web data)
- The **2-epoch training** may not be optimal for all datasets equally — datasets of vastly different sizes get different effective training

### Is the Evaluation Fair?
- **Mostly yes** — the multi-faceted approach is the paper's greatest strength
- **But:** They subsample GSM to 200 examples (from 1,319), which increases variance
- **GPT-4 evaluation** is shown to be biased, yet they still include it — though they properly flag the limitation
- Human evaluation only covers LLaMA-1 models (LLaMA-2 wasn't available yet)

### Would This Work in the Real World at Scale?
- **TÜLU itself is directly usable** — all models are released
- **The findings absolutely scale** — "use better base models" and "mix diverse data" are universally applicable
- **However:** The 65B model still needs significant compute, and is still significantly worse than ChatGPT (which has RLHF, which this paper doesn't explore)

---

## 📝 9. MEMORY ANCHORS

### Memorable Metaphor:
**TÜLU is like a cross-trained athlete**: each training dataset is a different sport. No single sport makes the perfect athlete, but training across many sports produces the best all-rounder — who still can't beat the specialist professional (ChatGPT/GPT-4) backed by a billion-dollar training program.

### 3 Bullet Points (80% of the paper):
- 📊 **No single instruction dataset wins everywhere** — CoT for reasoning, Code-Alpaca for coding, ShareGPT for open-ended chat — mixing them gives the best *average*
- 🏋️ **Base model quality > instruction data quality** — upgrading LLaMA-1→LLaMA-2 helps more than doubling model size
- ⚠️ **GPT-4 evaluation is dangerously biased toward verbose responses** (r=0.96 with unique token count) — benchmark + human evaluation are essential complements

### Understanding Check Question:
> *If a model trained on ShareGPT gets the highest AlpacaEval score but mediocre MMLU/GSM scores, while a model trained on FLAN V2 gets great MMLU scores but terrible AlpacaEval, what does this tell us about relying solely on model-based evaluation?*

**Expected answer:** Model-based evaluation (like AlpacaEval) rewards style and verbosity rather than actual knowledge or reasoning capability. ShareGPT produces long, chatty responses that GPT-4 prefers, while FLAN V2 produces short, precise answers that perform well on knowledge benchmarks but look "boring" to GPT-4. This proves you need multi-faceted evaluation.

---

## 🗺️ 10. VISUAL MENTAL MAP

```
┌─────────────────────────────────────────────────────────┐
│                      PROBLEM                             │
│  "Open models claim to match ChatGPT, but evaluations    │
│   are narrow and biased toward verbosity"                │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│                    DATA COLLECTION                        │
│  12 instruction datasets across 5 types:                 │
│  [NLP Tasks] [Human-written] [GPT-distilled]             │
│  [User-shared] [Skill-specific]                          │
│  15K─210K examples each                                  │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│                 UNIFIED TRAINING                          │
│  <|user|> ... <|assistant|> ... </s> format              │
│  Loss masking on user tokens                             │
│  LLaMA 7B/13B/30B/65B + OPT + Pythia                    │
│  2 epochs, lr=2e-5, DeepSpeed                            │
└────────────────────────┬────────────────────────────────┘
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
         Individual   Human Mix  Human+GPT Mix
         datasets    (4 datasets)  (7 datasets)
              │          │          │ ← TÜLU
              └──────────┼──────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│              MULTI-FACETED EVALUATION                     │
│                                                           │
│  Benchmarks:    [MMLU][GSM][BBH][TyDiQA][Codex-Eval]    │
│  Safety:        [ToxiGen][TruthfulQA]                    │
│  Model-based:   [AlpacaEval - GPT-4 judge]               │
│  Human:         [18 expert annotators, 332 prompts]      │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│                    KEY FINDINGS                           │
│                                                           │
│  1. No single dataset wins all tasks                     │
│  2. Base model quality >> data quantity                   │
│  3. TÜLU mix = best average (56.7%)                      │
│  4. Still < ChatGPT (72.3%) << GPT-4 (86.9%)            │
│  5. GPT-4 eval ≈ counting unique tokens (r=0.96)        │
└─────────────────────────────────────────────────────────┘
```

---

## 🛠️ 11. IMPLEMENTATION SKETCH

### Pseudocode (Core Training Loop):
```python
# 1. Load and format data
for dataset in [flan_v2, cot, dolly, oasst, gpt4_alpaca, code_alpaca, sharegpt]:
    formatted = convert_to_chat_format(dataset)  # <|user|> ... <|assistant|> ...
    all_data.append(formatted)

tulu_data = concatenate(all_data)  # Simple concatenation

# 2. Setup model
model = load_pretrained("llama-65b")
optimizer = AdamW(model.parameters(), lr=1e-5, weight_decay=0)
scheduler = LinearWarmupDecay(warmup_ratio=0.03)

# 3. Training loop (2 epochs)
for epoch in range(2):
    for batch in tulu_data:
        input_ids, labels = batch
        # Mask user tokens in labels (set to -100)
        labels = mask_user_tokens(input_ids, labels)  
        
        logits = model(input_ids)
        loss = cross_entropy(logits, labels, ignore_index=-100)
        
        loss.backward()
        optimizer.step()
        scheduler.step()

# 4. Evaluate
for benchmark in [MMLU, GSM, BBH, TyDiQA, CodexEval, AlpacaEval]:
    score = evaluate(model, benchmark)
    print(f"{benchmark}: {score}")
```

### Frameworks/Libraries Needed:
- **PyTorch** + **HuggingFace Transformers** (model loading, tokenization)
- **DeepSpeed** with **ZeRO Stage 3** (distributed training)
- **bitsandbytes** (8-bit inference)
- **AlpacaEval** library (GPT-4 based evaluation)
- **lm-evaluation-harness** (benchmark evaluation)

### Estimated Compute Cost:
- **7B model:** ~8 GPU-hours on A100s per dataset
- **13B model:** ~16 GPU-hours
- **65B model:** ~128+ GPU-hours (requires multi-node, 8+ GPUs)
- **Full paper reproduction** (all models × all datasets × all evaluations): **~2,000-3,000 GPU-hours** on AMD MI250x or equivalent
- **AlpacaEval costs:** ~$50-100 per model in GPT-4 API calls
