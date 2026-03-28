

# Hermes 3 Technical Report — Full Breakdown

---

## 🎯 1. THE ONE-LINER

**Hermes 3 is a family of AI chatbots (8B, 70B, 405B parameters) that are trained to do exactly what you tell them — without built-in moral refusals — and can reason, write code, use tools, and roleplay.**

---

## 🤔 2. THE PROBLEM (Why does this paper exist?)

- **Problem:** Most commercial chatbots (like ChatGPT) have heavy-handed guardrails baked into the model itself, which means they often **refuse valid requests** on moral grounds or behave in ways the user can't override. They also have a fixed "helpful assistant" personality that's hard to change.
- **Why should anyone care?** Imagine you buy a Swiss Army knife, but the manufacturer welded certain tools shut because *they* decided you shouldn't use them. Hermes 3 says: "Give the user the full knife. Let the *application builder* decide what to lock."
- **Limitations of previous approaches:**
  - Closed-weight models (GPT-4, Claude) can't be modified or inspected
  - Many open models still copy the refusal behavior of closed models in their training data
  - Older Hermes models had weaknesses in math, coding, and agentic tasks
  - Most instruct-tuned models aren't truly **steerable** — they default to "helpful assistant" regardless of the system prompt

---

## 💡 3. THE KEY IDEA (The "Aha!" moment)

- **Core insight:** **Neutral alignment + extreme system-prompt sensitivity.** Instead of embedding moral guardrails into the model's weights, train the model to be a **faithful executor of whatever system prompt and instructions it receives**. Safety should be enforced at the application layer, not the model layer.
- **Everyday analogy:** Think of it like a professional actor. A great actor doesn't refuse to play a villain — they **commit fully to whatever role the director gives them**. The director (application developer) decides what roles are appropriate for the audience (users). The actor (Hermes 3) just performs the role faithfully.
- **Novel features described step-by-step:**
  1. The model is trained on data that teaches it to **adopt the worldview of its system prompt**
  2. Special tokens like `<SCRATCHPAD>`, `<REASONING>`, `<THINKING>`, `<PLAN>` etc. are trained into the model for **structured, interpretable reasoning**
  3. Tool use via JSON schemas in `<tools>`, `<tool_call>`, `<tool_response>` tags
  4. RAG (retrieval) with citations via `<co>` tags
  5. Without a system prompt, the 405B model doesn't default to "helpful assistant" — it has **no baked-in persona**

---

## 🏗️ 4. HOW IT WORKS (The Method - Layer by Layer)

### Step 1: Choose Base Models
- **WHAT:** Start with **Llama 3.1** (8B, 70B, 405B) — Meta's open-weight foundation models with 128K context length
- **WHY:** Strong base capability + open weights = can be fine-tuned freely
- **CONNECTS TO:** These are the raw material that gets transformed into Hermes 3

### Step 2: Curate a Diverse Training Dataset (~390M tokens)
- **WHAT:** Build a carefully mixed dataset:
  - 60.6% General Instructions (236M tokens)
  - 12.8% Domain Expert (50M tokens)
  - 6.7% Math, 6.1% Roleplaying, 4.5% Coding
  - 4.3% Tool Use/Agentic/RAG
  - 3.0% Content Generation
  - **2.5% Steering and Alignment** (the "follow the system prompt exactly" data)
- **WHY:** Diverse data prevents the model from being a one-trick pony. The steering data teaches neutral compliance.
- **HOW:** Used both existing datasets and **Evol-Instruct-style synthetic generation**. Filtered out refusals, bad formatting, empty turns, and weak model outputs.
- **CONNECTS TO:** This dataset feeds into the SFT phase

### Step 3: Supervised Fine-Tuning (SFT)
- **WHAT:** Train on the dataset using standard instruction tuning
- **Key details:**
  - **AdamW optimizer**, learning rate **7×10⁻⁶** (halved to 3.5×10⁻⁶ for 405B)
  - **Cosine decay** schedule, 300 warmup steps, **4 epochs**
  - **Only response tokens contribute to loss** (instruction tokens are masked with ignore label = -100)
  - **Sample packing** using Flash Attention 2's variable-length support → 96% packing efficiency
  - Target sequence length: **8192 tokens**
- **WHY:** SFT teaches the model to produce the right kind of outputs for instructions
- **Special trick:** They pick the **best epoch checkpoint** using a normalized composite score of GPT4All + AGIEval + IFEval + MT-Bench (not just one benchmark!)
- **CONNECTS TO:** The SFT checkpoint feeds into the optional DPO phase

### Step 4: Direct Preference Optimization (DPO) — Optional
- **WHAT:** Train a **LoRA adapter** (r=32, α=16) using DPO to further align outputs with preferences
- **WHY:** DPO refines the model's outputs to prefer "chosen" over "rejected" responses
- **KEY FINDING:** DPO helped the **8B model** (moderate improvements in TruthfulQA, AGIEval, MT-Bench) but gave **negligible gains for 70B and 405B** — so they shipped the SFT-only checkpoints for the larger models
- **Uses:** RMSProp optimizer, NEFTune (noisy embeddings) with α=5

### Step 5: Evaluation & Release
- **WHAT:** Evaluate on 17+ benchmarks, release all weights on HuggingFace
- **WHY:** Prove competitive performance and enable community use

---

## 📊 5. THE PROOF (Results & Experiments)

### Benchmarks Used:
GPT4All suite (ARC, BoolQ, HellaSwag, OpenBookQA, PIQA, WinoGrande), AGIEval, BBH, GPQA, MATH Lvl 5, MMLU, MMLU-PRO, IFEval, MT-Bench, MuSR, TruthfulQA

### Key Results (Hermes 3 vs. Llama 3.1 Instruct — same base model):

| Metric | Hermes 3 405B | Llama 3.1 Instruct 405B | Winner |
|--------|--------------|------------------------|--------|
| **AGIEval** | **61.84** | 58.60 | ✅ Hermes |
| **ARC-C** | **69.45** | 66.04 | ✅ Hermes |
| **HellaSwag** | **90.19** | 88.34 | ✅ Hermes |
| **GPQA** | **44.84** | 42.66 | ✅ Hermes |
| **TruthfulQA** | **65.57** | 64.83 | ✅ Hermes |
| MMLU | 85.02 | **86.14** | ❌ Llama |
| MMLU-PRO | 54.14 | **63.51** | ❌ Llama |
| MATH Lvl 5 | 30.85 | **35.98** | ❌ Llama |
| IFEval | 84.87 | **87.09** | ❌ Llama |

### Most impressive result in plain English:
- **Hermes 3 405B beats Llama 3.1 Instruct 405B on commonsense reasoning** (HellaSwag 90.19 vs 88.34) and **truthfulness** (TruthfulQA 65.57 vs 64.83) while being **neutrally aligned** (no refusal training)
- The **70B model outperforms Llama 3.1 Instruct 70B on AGIEval by +7.9 points** (56.18 vs 48.26)

### Limitations they admitted:
- **Underperforms on MMLU, MMLU-PRO, MATH, and IFEval** compared to Llama 3.1 Instruct at the 405B level
- DPO provided **negligible improvement** for larger models
- The 405B training required **CPU parameter offloading** with a **45% drop in training efficiency**
- 405B evaluations done under **FP8 quantization** (not full precision)

---

## 🧩 6. KEY TERMS GLOSSARY

- **Instruct-tuned model** → A language model trained to follow commands/instructions rather than just predict text
- **System prompt** → A hidden instruction that sets the model's overall behavior (like "act as a pirate")
- **Neutral alignment** → Training a model to follow instructions without built-in moral refusals
- **SFT (Supervised Fine-Tuning)** → Training a model on input-output pairs where the "correct" output is provided
- **DPO (Direct Preference Optimization)** → A method to improve models using pairs of "good" vs "bad" responses without needing a separate reward model
- **LoRA (Low-Rank Adaptation)** → A memory-efficient way to fine-tune only small adapter matrices instead of all weights
- **Sample packing** → Stuffing multiple short training examples into one long sequence to avoid wasted padding tokens
- **Flash Attention 2** → A fast, memory-efficient implementation of the attention mechanism
- **Evol-Instruct** → A technique for generating increasingly complex instructions synthetically
- **NEFTune** → Adding noise to embeddings during fine-tuning to improve generalization
- **RAG (Retrieval Augmented Generation)** → Letting a model cite external documents when answering questions
- **Tool use / Function calling** → The model can request external computations (API calls, calculator, etc.)
- **FSDP (Fully Sharded Data Parallelism)** → A PyTorch strategy to split model parameters across GPUs
- **AdamW** → A popular optimizer with weight decay for training neural networks
- **Cosine decay** → A learning rate schedule that smoothly decreases like a cosine curve
- **FP8 quantization** → Reducing model precision to 8-bit floating point to save memory
- **Agentic** → Relating to AI systems that can take multi-step actions autonomously
- **HGX node** → A server containing 8 NVIDIA H100 GPUs
- **MT-Bench** → A benchmark testing multi-turn conversational ability, scored by an LLM judge
- **IFEval** → A benchmark testing how well a model follows specific formatting/constraint instructions

---

## 🔗 7. HOW IT CONNECTS

### Intellectual Family Tree:
```
Transformer (2017)
    └── Llama 3.1 (Meta, 2024) ← BASE MODEL
         └── Hermes 3 (this paper)

Instruction Tuning lineage:
    FLAN (2022) → InstructGPT → ChatGPT → WizardLM/Evol-Instruct
                                              └── Hermes 3's data generation

Alignment lineage:
    RLHF → DPO (2023) → Hermes 3's preference phase

Efficiency lineage:
    LoRA (2022) + Flash Attention 2 (2024) + NEFTune (2024)
    → Used in Hermes 3's training pipeline
```

### Who would use this:
- **AI application developers** who want full control over model behavior
- **Researchers** studying alignment, steerability, and tool use
- **Hobbyists/roleplayers** who want a model that won't refuse creative scenarios
- **Agent builders** who need structured reasoning (scratchpad, planning tokens)

### Future work this enables:
- Better **agentic AI systems** using the structured reasoning tokens
- Research on **system-level safety** vs model-level safety
- Community fine-tuning of the largest open instruct model (405B)
- Exploration of **reward modeling** capabilities of instruct models

---

## ⚖️ 8. CRITICAL ANALYSIS

### Hidden assumptions:
- **Assumes safety at the application layer is sufficient** — but not all application developers will implement it
- Assumes the Llama 3.1 base model is strong enough that SFT alone can compete with models trained with more RLHF
- Assumes **benchmark performance correlates with real-world utility** (especially for creative/roleplay tasks where no benchmarks are used)

### Weaknesses the authors DON'T mention:
- **No human evaluation** — all evaluations are automated benchmarks. For creative writing and roleplaying (a key selling point), no subjective quality assessment is provided
- **No safety evaluation at all** — no red-teaming, no toxicity benchmarks, no discussion of misuse potential despite explicitly removing guardrails
- The **MMLU-PRO gap is huge** (54.14 vs 63.51 at 405B) — nearly 10 points behind Llama 3.1 Instruct — suggesting the synthetic data may be hurting factual knowledge retention
- **No comparison with other open instruct models** like Mistral, Qwen, or Yi — only Llama 3.1 Instruct
- **Training data details are vague** — no specific datasets named beyond the Hermes function calling set

### Is the evaluation fair?
- Mostly yes for benchmarks, but the **405B was evaluated in FP8** while Llama 3.1 Instruct may have been evaluated in higher precision
- The **composite scoring for epoch selection** is novel but the min-max normalization could be gamed by one metric dominating

### Would this work in the real world at scale?
- Yes — it's already deployed on HuggingFace. The models run with standard inference infrastructure (vLLM, etc.)
- The **neutral alignment philosophy is controversial** — companies deploying this must build their own safety layer

---

## 📝 9. MEMORY ANCHORS

### Memorable metaphor:
> **Hermes 3 is like a master actor with no ego** — it has no default personality, no lines it refuses to say. It becomes whatever character the director (system prompt) tells it to be. The responsibility for what plays get performed lies with the theater (application), not the actor.

### 3 bullets that capture 80% of the paper:
- 📦 **Fine-tuned Llama 3.1 (8B/70B/405B) on 390M tokens** of diverse, curated instruction data with emphasis on neutrality and system-prompt obedience
- 🎯 **Trained with SFT + optional DPO** using sample packing, special reasoning tokens, and tool-use capabilities built in
- 🏆 **Beats Llama 3.1 Instruct on reasoning/commonsense benchmarks** but trades off on knowledge-intensive tasks (MMLU, MATH) — intentionally removes moral guardrails in favor of system-level safety

### One question to test understanding:
> *Why did Nous Research choose to keep only the SFT checkpoint (without DPO) for the 70B and 405B models?*
> **Answer:** Because DPO provided negligible benchmark improvements at those larger sizes, so the added complexity wasn't justified.

---

## 🗺️ 10. VISUAL MENTAL MAP

```
┌─────────────────────────────────────────────────────────────────┐
│                        PROBLEM                                  │
│  Commercial models: over-censored, not steerable, closed-weight │
│  Open models: copy refusal patterns, weak at agentic tasks      │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                     DATA CURATION                               │
│  390M tokens across 8 categories                                │
│  ┌──────────────────────────────────────────────┐               │
│  │ 60.6% General │ 12.8% Expert │ 6.7% Math    │               │
│  │ 6.1% RP │ 4.5% Code │ 4.3% Tools/RAG       │               │
│  │ 3.0% Content │ 2.5% Steering/Alignment      │               │
│  └──────────────────────────────────────────────┘               │
│  + Evol-Instruct generation + Filtering refusals                │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    TRAINING RECIPE                               │
│                                                                  │
│  Llama 3.1 ──► SFT (4 epochs, sample packing, special tokens) │
│  (8B/70B/405B)     │                                            │
│                    ▼                                             │
│              Best epoch selected via composite benchmark score   │
│                    │                                             │
│                    ▼                                             │
│              DPO (LoRA) ──► Helped 8B only                      │
│                              Negligible for 70B/405B            │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                       RESULTS                                    │
│                                                                  │
│  ✅ WINS: AGIEval (+3.2), ARC-C (+3.4), HellaSwag (+1.9),      │
│          GPQA (+2.2), TruthfulQA (+0.7) vs Llama 3.1 Instruct  │
│                                                                  │
│  ❌ LOSES: MMLU (-1.1), MMLU-PRO (-9.4), MATH (-5.1),          │
│           IFEval (-2.2) vs Llama 3.1 Instruct                  │
│                                                                  │
│  🎭 UNIQUE: Neutral alignment, structured reasoning tokens,     │
│            tool use, RAG with citations, roleplay               │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🛠️ 11. IMPLEMENTATION SKETCH

### Pseudocode of core training pipeline (~20 lines):
```python
# Phase 1: SFT
base_model = load("meta-llama/Llama-3.1-{8B,70B,405B}")
dataset = load_and_filter("hermes3_sft_dataset")  # 390M tokens

for sample in dataset:
    sample.labels = mask_instruction_tokens(sample, ignore_id=-100)

packed_data = pack_samples(dataset, max_len=8192)  # 96% efficiency

optimizer = AdamW(lr=7e-6, weight_decay=0.01)  # 3.5e-6 for 405B
scheduler = CosineDecay(warmup_steps=300)

for epoch in range(4):
    for batch in packed_data:
        loss = cross_entropy(model(batch.input), batch.labels)
        loss.backward()
        optimizer.step()
    scores[epoch] = evaluate(model, ["GPT4All", "AGIEval", "IFEval", "MTBench"])

best_model = checkpoints[argmax(normalized_composite(scores))]

# Phase 2: DPO (8B only)
lora = LoRA(r=32, alpha=16, dropout=0.05, target="all_linear")
dpo_train(best_model, lora, preference_data, optimizer=RMSProp(lr=3e-6))
```

### Frameworks/libraries needed:
- **Axolotl** (modified) — training framework
- **PyTorch FSDP** — distributed training
- **Flash Attention 2** — efficient attention with variable-length packing
- **vLLM** — inference serving
- **llm-compressor** — FP8 quantization for 405B
- **lm-evaluation-harness** — benchmarking
- **HuggingFace Transformers** — model loading

### Estimated compute cost to reproduce:
| Model | GPUs | GPU Hours | Est. Cloud Cost (H100 @ ~$3/hr) |
|-------|------|-----------|--------------------------------|
| 8B | 48 H100s | 147 | **~$441** |
| 70B | 48 H100s | 648 | **~$1,944** |
| 405B | 128 H100s | 2,086 | **~$6,258** |
| **Total** | | **2,881** | **~$8,643** |

*(This is remarkably cheap for state-of-the-art models — the key cost-saving is using an already-strong base model and only doing SFT, not pretraining)*
