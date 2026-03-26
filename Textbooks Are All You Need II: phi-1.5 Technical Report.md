

# Textbooks Are All You Need II: phi-1.5 Technical Report

---

## 🎯 1. THE ONE-LINER
A tiny AI model (phi-1.5) trained on **fake textbooks written by a bigger AI** can reason and think almost as well as models **5-10x its size** that were trained on the entire internet.

---

## 🤔 2. THE PROBLEM (Why does this paper exist?)

- **The problem:** State-of-the-art language models require hundreds of billions of parameters and trillions of tokens of internet data, making them enormously expensive to train (>80,000 GPU hours), deploy, and study. This means only a few wealthy organizations can build them.
- **Why should anyone care?** Imagine you wanted to become a great chef. The current approach is like eating at every restaurant in the world (including terrible ones) and hoping you learn something. This paper asks: **what if you just read the best cookbook?**
- **Limitations of previous approaches:**
  - Bigger models = more cost, more energy, less accessibility
  - Web data is **noisy, toxic, biased**, and full of low-quality content
  - Small models trained on web data performed terribly at reasoning
  - Prior work (phi-1) showed this "textbook" approach worked for **code** — but could it work for **common sense reasoning**?

---

## 💡 3. THE KEY IDEA (The "Aha!" moment)

**Core insight:** You don't need more data or bigger models — you need **better data**. Specifically, **synthetically generated "textbook-quality" data** created by a large LLM can teach a small model to reason far better than oceans of messy web data.

**Cooking analogy:** Instead of feeding a culinary student 1 million random recipes from the internet (many wrong, many duplicated, many for dishes nobody wants), you write them a **perfect, comprehensive, well-organized textbook** covering 20,000 carefully chosen topics. The student learns faster and better, even though they read far less.

**The trick in detail:**
1. Use a powerful LLM (like GPT-4) to **generate synthetic textbook-like content** covering common sense reasoning, science, daily activities, theory of mind, etc.
2. Carefully select **20,000 seed topics** to ensure diversity and coverage
3. Use web data samples only for **diversity in prompting**, not as training data itself
4. Train a small 1.3B parameter model on this curated synthetic data
5. The result: a model that punches **way above its weight class**

---

## 🏗️ 4. HOW IT WORKS (The Method - Layer by Layer)

### Step 1: Design the Architecture
- **WHAT:** Standard Transformer with 24 layers, 32 attention heads, head dimension 64, context length 2048
- **WHY:** Deliberately kept identical to phi-1 (their prior coding model) to **isolate the effect of data quality** — any improvement comes from better data, not a fancier architecture
- **HOW it connects:** This is the "student brain" that will learn from the textbooks

### Step 2: Curate Seed Topics
- **WHAT:** Carefully select ~20,000 topics spanning common sense reasoning, science, daily activities, theory of mind, etc.
- **WHY:** Without strategic topic selection, the synthetic data would have gaps — like a textbook that covers biology but forgets chemistry
- **HOW it connects:** These topics become prompts for generating synthetic data

### Step 3: Generate Synthetic "Textbook" Data (~20B tokens new)
- **WHAT:** Use an existing large LLM to generate ~20 billion tokens of textbook-like training data, seeded by the chosen topics and flavored with web data samples for diversity
- **WHY:** Web data is noisy. Synthetic textbook data is **clean, structured, pedagogically organized** — like having a perfect teacher write explanations
- **HOW it connects:** This becomes the primary training diet (80% of training)

### Step 4: Combine with phi-1's Code Data (7B tokens)
- **WHAT:** Mix in the 7B tokens from phi-1's training (filtered code data)
- **WHY:** Retains coding ability; multi-task training doesn't degrade performance here (unlike with web data)
- **HOW it connects:** Makes the model versatile (NLP + code)

### Step 5: Train from Scratch
- **WHAT:** Train phi-1.5 from random initialization, constant learning rate 2e-4, batch size 2048, for 150B tokens total (multi-epoch over the 30B token dataset)
- **WHY:** Intentionally simple training config to **emphasize that data quality is the key variable**, not training tricks
- **HOW it connects:** Results in the final phi-1.5 model

### Step 6 (Variant): Create phi-1.5-web
- **WHAT:** Also train a variant mixing synthetic data (40%) + filtered web data (40%) + code (20%)
- **WHY:** To probe whether web data adds value on top of synthetic data
- **HOW it connects:** This model (phi-1.5-web) shows that some web data helps, especially for reasoning tasks, but is not strictly necessary

---

## 📊 5. THE PROOF (Results & Experiments)

### Benchmarks tested:
| Category | Benchmarks |
|---|---|
| **Common Sense Reasoning** | WinoGrande, ARC-Easy, ARC-Challenge, BoolQ, SIQA |
| **Language Understanding** | PIQA, HellaSwag, MMLU, OpenbookQA, SQUAD |
| **Multi-Step Reasoning** | GSM8K (math), HumanEval (code), MBPP (code) |

### Key numbers:

- **Common sense:** phi-1.5 (1.3B) scores **0.734 on WinoGrande** vs Llama2-7B's **0.691** — a model 5x smaller beating a 5x larger one
- **Math (GSM8K):** phi-1.5 scores **40.2%** vs Llama2-7B's **14.6%** — nearly **3x better** with 5x fewer parameters
- **Coding (HumanEval):** phi-1.5 scores **34.1%** vs Llama2-7B's **12.8%** — outperforms even **Llama-65B** (23.7%) on coding despite being **50x smaller**
- **Training cost:** 1,500 GPU hours vs >80,000 for Llama-7B — **~53x cheaper**
- **Inference:** <3ms per token vs 14ms; 3.5GB memory vs 18GB

### Most impressive result in plain English:
**A model trained for 1,500 GPU hours on mostly AI-generated textbooks can solve grade-school math problems and write code better than a model that took 80,000+ GPU hours trained on the entire internet.**

### Limitations admitted:
- **Hallucinations** still occur
- **Toxic/biased content** can still be generated (34/86 prompts failed toxicity test)
- Performance on some **language understanding** tasks (HellaSwag, PIQA) still lags behind 7B models — these rely more on **memorized knowledge** which requires seeing more real-world data
- No instruction fine-tuning or RLHF — model doesn't always follow instructions perfectly
- Benchmark contamination concern: since synthetic data is generated by a large LLM that may have seen benchmark questions

---

## 🧩 6. KEY TERMS GLOSSARY

**Transformer** → The neural network architecture behind all modern LLMs; processes text using "attention" to relate words to each other

**Synthetic data** → Training data artificially generated by another AI model, not collected from the real world

**Textbook-quality data** → AI-generated text that explains concepts clearly and pedagogically, like a well-written textbook

**Common sense reasoning** → The ability to understand everyday knowledge (e.g., "rain makes things wet")

**Rotary embedding (RoPE)** → A way of encoding word positions in a sequence so the model knows word order

**Flash-attention** → A fast, memory-efficient implementation of the attention mechanism

**DeepSpeed ZeRO Stage 2** → A distributed training technique that splits model states across GPUs to save memory

**RLHF (Reinforcement Learning from Human Feedback)** → Training a model using human preferences to make it more helpful/safe

**Instruction fine-tuning** → Additional training that teaches a model to follow user instructions

**Chain-of-thought (CoT)** → Prompting a model to "think step by step" to improve reasoning

**Base model** → A model after pre-training but before any alignment/instruction tuning

**In-context learning** → A model's ability to learn from examples given in the prompt, without updating its weights

**Theory of mind** → Understanding that other people have their own thoughts and feelings

**Token** → A chunk of text (roughly ¾ of a word) that the model processes as one unit

**Hallucination** → When a model confidently generates false or made-up information

**Zero-shot** → Evaluating a model on a task without giving it any examples first

**Pass@1** → The accuracy when the model gets only one attempt to generate the correct answer

**fp16** → Half-precision floating point; uses 16 bits per number instead of 32 to save memory/speed

---

## 🔗 7. HOW IT CONNECTS

### Intellectual family tree:
```
TinyStories (2023, Eldan & Li)
  "How small can a model be and still speak English?"
  └──> phi-1 / "Textbooks Are All You Need" (2023, Gunasekar et al.)
         "1.3B model matches SOTA at Python coding via synthetic data"
         └──> phi-1.5 (THIS PAPER)
                "Extend textbook approach to common sense reasoning"
                └──> phi-2, phi-3 (future work)
```

### Related contemporaneous work:
- **Llama 2** (Meta, 2023): 7B-70B models trained on 2T tokens of web data — brute force scaling approach
- **Falcon** (TII, 2023): Focused on high-quality web data curation (RefinedWeb) but still used real web data, not synthetic

### Who would use this:
- **Researchers** studying interpretability, hallucinations, bias (small enough to experiment on)
- **Edge/mobile AI developers** needing capable but small models
- **AI safety researchers** studying how data quality affects toxicity
- **Organizations with limited compute** wanting capable LLMs

### Future work this enables:
- **Synthetic data engineering** as a new research field
- Potentially achieving **ChatGPT-level capabilities at 1B parameters**
- Fine-tuning phi-1.5 for specific domains
- Studying whether **data quality can fully substitute for scale**

---

## ⚖️ 8. CRITICAL ANALYSIS

### Hidden assumptions:
- **Assumes access to a powerful LLM** to generate the synthetic data — you need a GPT-4-class model to make the textbooks, which is circular if you're arguing for small models
- **Assumes the 20K topics provide sufficient coverage** — but who decides what "sufficient" means? The topic selection process is not detailed
- Assumes benchmarks accurately measure "common sense reasoning" — many benchmarks can be gamed

### Weaknesses the authors DON'T mention:
- **Benchmark contamination risk is severe**: The LLM generating synthetic data likely saw benchmark questions during its own training. The synthetic textbooks could inadvertently contain benchmark-like examples, inflating phi-1.5's scores
- **Reproducibility is limited**: The synthetic data generation process (prompts, topic selection, filtering) is described vaguely — "intricate iterations" and "deep understanding of knowledge gaps" are not reproducible instructions
- **The cost of generating synthetic data is not accounted for** — GPT-4 API calls to generate 20B tokens are expensive and not counted in the 1,500 GPU hours
- **Multi-epoch training** (150B tokens from 30B dataset = ~5 epochs) raises overfitting concerns that aren't discussed
- **No error analysis** on where/why the model fails

### Is the evaluation fair?
- ✅ They use their own consistent evaluation pipeline across all models
- ⚠️ They only compare against open-source models, not against other synthetic-data approaches
- ⚠️ The toxicity evaluation on 86 manually crafted prompts is tiny and manually graded
- ❌ No analysis of **whether the synthetic data leaks benchmark answers**

### Would this work at scale in the real world?
- **Partially.** The model is genuinely small and efficient
- But it still **hallucinates, lacks broad world knowledge** (e.g., current events), and struggles with tasks requiring memorized factual knowledge
- The approach likely hits a ceiling because synthetic data from one model can't exceed that model's own knowledge

---

## 📝 9. MEMORY ANCHORS

### Memorable metaphor:
**phi-1.5 is like a student who never went to school but was homeschooled with the world's best textbook** — they can reason brilliantly but might not know the latest gossip (factual/world knowledge gaps), and occasionally make things up (hallucinations).

### 3 bullet points (80% of the paper):
- 🔑 **Data quality > data quantity**: A 1.3B model trained on ~30B tokens of synthetic "textbook" data matches or beats 7-13B models trained on 1T tokens of web data
- 🔑 **Synthetic data from LLMs** can teach common sense reasoning, not just coding — and it naturally reduces toxic outputs since the training data doesn't contain internet toxicity
- 🔑 **The cost is 50x lower** (~1,500 vs 80,000+ GPU hours), making frontier-like capabilities accessible to researchers

### Understanding check question:
> *Why does phi-1.5 outperform models 5-10x its size on reasoning but NOT on knowledge-heavy tasks like HellaSwag, and what does this tell us about what synthetic textbook data is good at teaching vs. what it misses?*

---

## 🗺️ 10. VISUAL MENTAL MAP

```
PROBLEM                          METHOD                              RESULT
═══════                          ══════                              ══════

Big LLMs are too             ┌─────────────────┐
expensive & hard    ───────> │ Step 1: Keep     │
to study                    │ architecture     │
                            │ small (1.3B,     │
"Do we really need          │ same as phi-1)   │
trillions of tokens?" ──┐   └────────┬─────────┘
                        │            │
                        │   ┌────────▼─────────┐
                        └──>│ Step 2: Generate  │        ┌──────────────────┐
                            │ 20B tokens of     │        │ Common Sense:    │
                            │ SYNTHETIC         │        │ ≈ Llama2-7B      │
Web data is noisy,          │ "textbook" data   │───────>│ (5x smaller!)    │
toxic, biased ──────────┐   │ using GPT-4       │        │                  │
                        │   └────────┬──────────┘        │ Math (GSM8K):    │
                        │            │                   │ 40% vs 14.6%     │
                        │   ┌────────▼──────────┐        │ (3x better!)     │
                        └──>│ Step 3: Mix with   │        │                  │
                            │ phi-1 code data    │        │ Coding:          │
                            │ (NO web data for   │───────>│ Beats Llama-65B! │
                            │ base phi-1.5)      │        │ (50x smaller!)   │
                            └────────┬──────────┘        │                  │
                                     │                   │ Toxicity:        │
                            ┌────────▼──────────┐        │ Lower than       │
                            │ Step 4: Train     │───────>│ web-trained      │
                            │ 150B tokens       │        │ models           │
                            │ 1,500 GPU hours   │        │                  │
                            │ (vs 80K for       │        │ Cost: 53x less   │
                            │ Llama-7B)         │        └──────────────────┘
                            └───────────────────┘

KEY INSIGHT: ════════════════════════════════════════════════
  "It's not about eating everything on the internet.
   It's about reading the right textbook."
═════════════════════════════════════════════════════════════
```

---

## 🛠️ 11. IMPLEMENTATION SKETCH

### Pseudocode (conceptual — the paper's method is mostly about data, not algorithm):
```python
# Phase 1: Synthetic Data Generation (the secret sauce)
topics = curate_20k_topics(["science", "daily_life", 
                            "theory_of_mind", "math", ...])
web_samples = sample_for_diversity(falcon_refined_web)

synthetic_data = []
for topic in topics:
    prompt = create_textbook_prompt(topic, web_samples)
    textbook_page = GPT4.generate(prompt)  # ~20B tokens total
    synthetic_data.append(textbook_page)

# Phase 2: Combine Training Data
train_data = mix(
    synthetic_data,          # 80% (~20B tokens)
    phi1_code_data,          # 20% (~7B tokens)
)  # Total: ~30B unique tokens

# Phase 3: Train the Model
model = Transformer(
    layers=24, heads=32, head_dim=64,
    rotary_dim=32, context_len=2048
)  # 1.3B parameters

optimizer = Adam(lr=2e-4, betas=(0.9, 0.98), eps=1e-7)
for epoch in range(5):  # 150B tokens / 30B ≈ 5 passes
    for batch in train_data.shuffle().batch(2048):
        loss = cross_entropy(model(batch), batch.shifted())
        loss.backward()
        optimizer.step()
```

### Frameworks/Libraries needed:
- **PyTorch** + **DeepSpeed ZeRO Stage 2** (distributed training)
- **FlashAttention** (efficient attention)
- **codegen-mono tokenizer** (from Salesforce)
- Access to **GPT-4 API** (for synthetic data generation)
- **LM-Eval Harness** (for benchmarking)

### Estimated compute to reproduce:
| Component | Cost |
|---|---|
| Training phi-1.5 | ~1,500 A100 GPU hours (~$3,000-4,500 at cloud rates) |
| Generating 20B tokens via GPT-4 | **Not disclosed** but likely $50K-200K+ at API prices |
| Training phi-1.5-web | ~3,000 A100 GPU hours |
| Total realistic cost | **$55K-$205K+** (the synthetic data cost is the hidden expense) |
