

# MAmmoTH2: Scaling Instructions from the Web — Full Breakdown

---

## 🎯 1. THE ONE-LINER
**Instead of paying humans or using GPT-4 to create teaching examples for AI, this paper figured out how to automatically find 10 million ready-made question-answer pairs already hiding on the internet and used them to make AI much better at reasoning.**

---

## 🤔 2. THE PROBLEM (Why does this paper exist?)

- **The problem:** To make LLMs better at reasoning (math, science, engineering), you need lots of high-quality "instruction data" — question-answer pairs that teach the model how to think step-by-step. But getting this data is really hard.
- **Why should anyone care?**
  - Imagine you're training a student for a math competition. You need thousands of practice problems with detailed solutions. You could either:
    - (A) **Hire tutors** to hand-write problems — expensive and slow (human annotation)
    - (B) **Ask one smart student to make up problems** — fast but they keep making similar problems and sometimes get answers wrong (GPT-4 distillation)
    - (C) **Go find the millions of practice exams, textbook solutions, and forum answers already posted online** — free, diverse, and real ← **This is what the paper does**
- **Limitations of previous approaches:**
  - **Human annotation:** Costly, small-scale (datasets like GSM8K only have ~8K examples)
  - **GPT-4 distillation:** Prone to **hallucination** (making up wrong answers), **not diverse** (biased by seed data — usually just GSM8K/MATH), and limited by licensing/API costs
  - **Continued pre-training** on filtered web docs: Requires **massive compute** (100B+ tokens) and documents are noisy — not structured as Q-A pairs

---

## 💡 3. THE KEY IDEA (The "Aha!" moment)

> **"The internet already has millions of high-quality question-answer pairs scattered across educational websites, forums, and quiz sites. We just need a smart way to find them, extract them, and clean them up."**

### Everyday Analogy: Gold Panning 🪙
Think of the internet (Common Crawl) as a giant river full of mud and rocks. Hidden in that mud are **gold nuggets** — naturally occurring Q-A pairs on sites like Khan Academy, Stack Exchange, Chegg, and exam websites. The paper builds a **three-stage gold-panning system**:
1. **Pan for promising areas** (Recall) — use a fast classifier to find river sections likely to contain gold
2. **Pick out the nuggets** (Extract) — use an LLM to pull Q-A pairs out of messy HTML pages
3. **Polish the nuggets** (Refine) — use LLMs to clean up formatting and add missing explanations

### The pipeline in ASCII:
```
                    WEBINSTRUCT Pipeline
┌──────────┐     ┌──────────┐     ┌──────────┐
│  RECALL  │ ──→ │ EXTRACT  │ ──→ │  REFINE  │
│          │     │          │     │          │
│ fastText │     │ Qwen-72B │     │ Mixtral+ │
│ + GPT-4  │     │ extracts │     │ Qwen-72B │
│ URL filter│    │ Q-A pairs│     │ add CoT  │
└──────────┘     └──────────┘     └──────────┘
 Common Crawl      18M docs        5M raw QA → 10M refined QA
  → 18M docs      → 5M QA pairs     (WEBINSTRUCT)
```

---

## 🏗️ 4. HOW IT WORKS (The Method - Layer by Layer)

### Step 1: RECALL — Finding Relevant Documents
- **WHAT:** Find web pages likely to contain educational Q-A content from Common Crawl (~billions of pages)
- **HOW:**
  1. Crawl **100K seed examples** from quiz/exam websites across math, science, engineering
  2. Train a **fastText classifier** (positive: seed data, negative: random CC docs)
  3. Run classifier over Common Crawl → recall **100B tokens**
  4. Group by root URL → keep domains with 1000+ docs (~600K domains)
  5. Use **GPT-3.5** to scan domains and select ~50K educational ones
  6. **Re-train** fastText with better positive/negative examples from selected/non-selected domains
  7. Recall again → **40B tokens**, then use **GPT-4** to filter domains → **18M final documents**
- **WHY:** Can't process all of Common Crawl with expensive LLMs. Need a fast, cheap filter first. The **two-round recall** (train → filter → retrain → recall again) ensures higher precision.
- **CONNECTS TO:** These 18M documents feed into the extraction step

### Step 2: EXTRACT — Pulling Out Q-A Pairs
- **WHAT:** Use an LLM to identify and extract question-answer pairs from noisy web documents
- **HOW:**
  1. **Rule-based HTML preprocessing** — strip ads, boilerplate, navigation, site info
  2. Prompt **Qwen-72B** with few-shot examples to extract Q-A pairs
  3. Model can return "void" if no Q-A pair exists
  4. Only **30%** of documents contain extractable Q-A pairs → **~5M candidate pairs**
  5. **Decontamination:** Remove any page with 10-gram overlap with evaluation benchmarks
- **WHY:** Raw HTML is full of noise. Even after HTML cleanup, the Q-A pairs are messy — missing explanations, formatting issues, irrelevant content mixed in.
- **CONNECTS TO:** These 5M candidates need quality improvement before use

### Step 3: REFINE — Cleaning and Enhancing Q-A Pairs
- **WHAT:** Use LLMs to reformat extracted pairs and add step-by-step reasoning
- **HOW:**
  1. Prompt **Mixtral-22B×8** and **Qwen-72B** (two different models for diversity)
  2. Each model reformats the Q-A pair, fixes formatting
  3. **If the answer lacks explanation**, the LLM generates intermediate reasoning steps
  4. Merge outputs from both models → **10M final Q-A pairs** = **WEBINSTRUCT**
- **WHY:** Many extracted answers are just bare answers (e.g., "D") without reasoning chains. Adding chain-of-thought is **critical** for teaching models to reason. Using **two models** increases diversity.
- **CONNECTS TO:** This dataset is used for supervised fine-tuning of base LLMs

### Step 4: FINE-TUNING — Training MAmmoTH2
- **WHAT:** Fine-tune base LLMs (Mistral-7B, Llama-3-8B, Mixtral-8×7B, Yi-34B) on WEBINSTRUCT
- **HOW:** Standard SFT with multi-turn instruction format, LR=5e-6 to 1e-5, batch=512, seq_len=4096, 2 epochs, 32 A100 GPUs
- **WHY:** SFT loss (only computing loss on the response, not the question) outperforms LM loss

### Step 5 (Optional): MAmmoTH2-PLUS
- **WHAT:** Further fine-tune on public datasets: OpenHermes 2.5, Code-Feedback, Math-Plus
- **WHY:** Adds math competition data (MATH/GSM-derived), code, and chat capabilities

---

## 📊 5. THE PROOF (Results & Experiments)

### Benchmarks:
7 reasoning benchmarks (TheoremQA, MATH, GSM8K, GPQA, MMLU-STEM, BBH, ARC-C) + code (HumanEval, MBPP) + chat (AlpacaEval 2.0, Arena Hard, MT-Bench)

### Key Results (specific numbers):

| Comparison | Metric | Improvement |
|---|---|---|
| **MAmmoTH2-7B vs Mistral-7B** | MATH | **11.2% → 36.7%** (+25.5) |
| **MAmmoTH2-7B vs Mistral-7B** | GSM8K | **36.2% → 68.4%** (+32.2) |
| **MAmmoTH2-7B vs Mistral-7B** | Average (7 benchmarks) | **+14.0 points** |
| **MAmmoTH2-8B vs Llama-3-8B** | Average | **+8.8 points** |
| **MAmmoTH2-8B-Plus vs Llama-3-8B-Instruct** | Average reasoning | **+5.7 points** |
| **MAmmoTH2-8×7B-Plus vs Qwen-1.5-110B** | Average | **Nearly matches** (with only 13B active params) |

### Most impressive result in plain English:
**A 7B model trained on auto-mined web data (no human annotation, no GPT-4) outperforms Llama-3-8B-Instruct (trained on 10M human-annotated examples) by 6 points on reasoning — essentially getting a "free" dataset that's better than one of the most expensive human-created datasets.**

### Scaling results:
- Performance **consistently improves** from 1M to 10M instructions (Figure 5)
- Refined QA > Extracted QA (refinement helps)
- SFT loss > LM loss

### Failure cases / Limitations admitted:
- **10% of refined examples introduce hallucinations** (Figure 6)
- Some extracted Q-A pairs have **unrecoverable formatting issues**
- Dataset is **heavily math-biased** (68% math, 82% science overall)
- Humanities and daily chat topics **underrepresented**
- Refinement LLMs can **modify original intent**, creating incorrect answers

---

## 🧩 6. KEY TERMS GLOSSARY

- **Instruction Tuning** → Teaching an LLM to follow instructions by training it on question-answer pairs
- **SFT (Supervised Fine-Tuning)** → Training a model on labeled examples where you know the correct output
- **LM Loss** → Computing training loss on the entire sequence (question + answer)
- **SFT Loss** → Computing training loss only on the answer portion (more effective for instruction tuning)
- **Common Crawl (CC)** → A massive free dataset of web pages crawled from the internet
- **fastText** → A lightweight, fast text classification model by Facebook — used here as a cheap document filter
- **Chain-of-Thought (CoT)** → Making a model show its reasoning step-by-step before giving a final answer
- **Distillation** → Using a powerful model (like GPT-4) to generate training data for a weaker model
- **Hallucination** → When an LLM generates plausible-sounding but factually incorrect information
- **Decontamination** → Removing training data that overlaps with test benchmarks to prevent data leakage
- **Continued Training (CT)** → Further pre-training an already-trained LLM on additional domain data
- **Mixture of Experts (MoE)** → Architecture where only a subset of parameters activate per input (e.g., Mixtral)
- **WEBINSTRUCT** → The 10M Q-A pair dataset created by this paper
- **MAmmoTH2** → The models fine-tuned on WEBINSTRUCT
- **MAmmoTH2-Plus** → MAmmoTH2 further trained on additional public datasets
- **n-gram** → A contiguous sequence of n words; used here for decontamination matching

---

## 🔗 7. HOW IT CONNECTS

### Intellectual Family Tree:
```
                    ┌─── Human annotation (FLAN, GSM8K, MATH)
                    │
Instruction Tuning ─┤─── GPT-4 Distillation (WizardMath, MetaMathQA, MathInstruct)
                    │         ↑ MAmmoTH (v1) is here
                    │
                    └─── Web Mining ← THIS PAPER (MAmmoTH2)
                              │
                              ├── Uses recall from DeepSeekMath's approach
                              ├── Uses fastText from OpenWebMath's approach  
                              └── Uses LLM refinement inspired by Kun (instruction back-translation)
```

### Who would use this:
- **LLM developers** wanting high-quality instruction data without GPT-4 costs
- **Researchers** studying data-efficient training
- **Educators** building AI tutoring systems
- **Open-source community** — dataset is released publicly

### Future work this enables:
- Mining instruction data for **other languages**
- Better **quality filtering** with learned selection models
- Extending to **multimodal** (images of equations, diagrams)
- Broader domain coverage (humanities, social sciences)

---

## ⚖️ 8. CRITICAL ANALYSIS

### Hidden assumptions:
- **Web Q-A pairs are correct.** The paper assumes educational websites have accurate answers, but many (e.g., Chegg, homework help sites) have **known error rates**
- **Open-source LLMs can reliably extract and refine.** The 10% hallucination rate is self-reported on only 50 samples — the true rate across 10M could be higher
- **fastText + URL filtering captures all relevant domains.** Many educational resources may be missed

### Weaknesses the authors DON'T mention:
- **Bias toward English** — Common Crawl is English-heavy; no multilingual evaluation
- **Copyright concerns** — Mining from educational sites (Chegg, CourseHero, Khan Academy) raises IP questions
- **GPT-4 is still used** for URL filtering — the claim of "no GPT-4 distillation" is technically true but GPT-4 plays a role in the pipeline
- **Math-Plus dataset** used in MAmmoTH2-Plus **does use GPT-4** to rewrite MATH training set problems — making the "Plus" comparison less clean
- **Decontamination via 10-gram matching** may miss paraphrased test questions

### Is the evaluation fair?
- Mostly yes — **7 held-out benchmarks** when training only on WEBINSTRUCT
- But for MAmmoTH2-Plus, **MATH and GSM8K are no longer held-out** (Math-Plus contains GSM/MATH derivatives)
- The Llama-3-Instruct comparison is framed as "apple-to-apple" but Llama-3-Instruct also includes RLHF/DPO which MAmmoTH2 doesn't use

### Would this work at scale in the real world?
- **Yes**, with caveats. The pipeline is reproducible with open-source tools. But the compute for refining 10M examples with 72B parameter models is non-trivial. The approach is fundamentally sound and likely to improve with better base models.

---

## 📝 9. MEMORY ANCHORS

### Memorable metaphor:
> **"MAmmoTH2 is like a treasure hunter with a metal detector (fastText) and a jewelry polisher (LLM refiner) — instead of manufacturing fake gold (GPT-4 distillation), it finds real gold nuggets (Q-A pairs) scattered across the internet's beaches (Common Crawl) and polishes them until they shine."**

### 3 bullet points (80% of the paper):
- **📥 Mine, don't manufacture:** 10M instruction-response pairs were harvested from Common Crawl using a 3-step pipeline (Recall → Extract → Refine) without human annotation or GPT-4 distillation
- **📈 Massive improvements:** Fine-tuning on this data boosted Mistral-7B from 11% to 37% on MATH and outperformed Llama-3-8B-Instruct (trained on 10M human-labeled examples) by 6 points
- **🔑 Refinement is critical:** Adding step-by-step explanations to bare answers via LLM refinement was the key quality enhancer, and using two different LLMs for refinement improved diversity

### Comprehension question:
> *Why is the refinement step important, and why do they use two different LLMs for it instead of just one?*

---

## 🗺️ 10. VISUAL MENTAL MAP

```
PROBLEM                          METHOD                              RESULT
─────────                        ──────                              ──────
                                                                     
Instruction data is        ┌─────────────────────┐            
either:                    │  1. RECALL           │            MAmmoTH2-7B:
• Expensive (human)        │  ┌────────────────┐  │            MATH: 11% → 37%
• Hallucinated (GPT-4)     │  │ Seed data 100K │  │            GSM8K: 36% → 68%
• Narrow (math-only)       │  │ → fastText     │  │            
                           │  │ → Common Crawl │  │            MAmmoTH2-8B-Plus
        │                  │  │ → URL filter   │  │            beats Llama-3-Instruct
        │                  │  │ → 18M docs     │  │            by +6 avg points
        ▼                  │  └────────────────┘  │            
                           │          │           │            MAmmoTH2-8x7B-Plus
Can we find natural        │          ▼           │            matches Qwen-1.5-110B
Q-A pairs already          │  2. EXTRACT          │            with only 13B active
on the web?                │  ┌────────────────┐  │            params
                           │  │ HTML cleanup   │  │            
        │                  │  │ → Qwen-72B     │  │            Also strong on:
        │                  │  │ → 5M Q-A pairs │  │            • Code generation
        ▼                  │  │ → Decontam.    │  │            • Chat benchmarks
                           │  └────────────────┘  │            • MMLU-Pro
YES! The web has           │          │           │            
millions of them.          │          ▼           │            Key finding:
Just need to:              │  3. REFINE           │            Performance scales
1) Find them               │  ┌────────────────┐  │            with # instructions
2) Extract them            │  │ Mixtral+Qwen   │  │            (1M → 10M)
3) Clean them up           │  │ → Reformat     │  │            
                           │  │ → Add CoT      │  │            78% improved after
                           │  │ → 10M pairs    │  │            refinement (10% error)
                           │  │ = WEBINSTRUCT  │  │            
                           │  └────────────────┘  │            
                           └─────────┬───────────┘            
                                     │                        
                                     ▼                        
                           ┌─────────────────────┐            
                           │  4. FINE-TUNE        │            
                           │  Base LLM + SFT loss │            
                           │  → MAmmoTH2          │            
                           │                      │            
                           │  + Public datasets    │            
                           │  → MAmmoTH2-Plus     │            
                           └──────────────────────┘            
```

---

## 🛠️ 11. IMPLEMENTATION SKETCH

### Pseudocode (Core Pipeline):
```python
# Step 1: RECALL
seed_data = crawl_quiz_websites(n=100_000)
neg_data = sample_common_crawl(n=100_000)
classifier_v1 = train_fasttext(pos=seed_data, neg=neg_data, dim=256, epochs=3)
docs_round1 = classifier_v1.predict(common_crawl, threshold=0.5)  # ~100B tokens

domains = group_by_root_url(docs_round1)
domains = filter(domains, min_docs=1000)  # ~600K domains
good_domains = gpt35_filter_domains(domains)  # ~50K domains

classifier_v2 = train_fasttext(pos=sample(good_domains), neg=sample(bad_domains + CC))
docs_round2 = classifier_v2.predict(common_crawl)  # ~40B tokens
final_docs = gpt4_filter_domains(docs_round2)  # 18M docs

# Step 2: EXTRACT
for doc in final_docs:
    clean_doc = remove_html_boilerplate(doc)
    qa_pair = qwen_72b.extract_qa(clean_doc, few_shot_examples)  # returns None if no QA
    if qa_pair and not is_contaminated(qa_pair, eval_benchmarks, n=10):
        candidates.append(qa_pair)  # ~5M pairs

# Step 3: REFINE  
for qa in candidates:
    refined_mixtral = mixtral_22bx8.refine(qa, add_chain_of_thought=True)
    refined_qwen = qwen_72b.refine(qa, add_chain_of_thought=True)
    webinstruct.extend([refined_mixtral, refined_qwen])  # ~10M pairs

# Step 4: FINE-TUNE
model = load_base_model("mistral-7b")
model.sft(webinstruct, lr=5e-6, batch=512, epochs=2, loss="sft")
```

### Frameworks/Libraries needed:
- **fastText** (Facebook's text classification library)
- **vLLM** or **TGI** for LLM inference at scale (Qwen-72B, Mixtral)
- **LLaMA-Factory** for fine-tuning
- **DeepSpeed ZeRO-3** for distributed training
- **trafilatura** or **readability** for HTML cleaning
- Access to **Common Crawl** dumps (~petabytes of raw data)

### Estimated compute cost:
- **fastText training:** Negligible (minutes on CPU)
- **LLM extraction (Qwen-72B on 18M docs):** ~2,000-5,000 A100-hours
- **LLM refinement (Mixtral + Qwen on 5M pairs × 2):** ~3,000-8,000 A100-hours
- **Fine-tuning (32 A100s, 2 epochs):** ~500-1,000 A100-hours per model
- **Total estimated:** ~**10,000-15,000 A100-hours** (~$15K-30K at cloud rates)
- Significantly cheaper than hiring humans to annotate 10M examples or running 10M GPT-4 API calls
