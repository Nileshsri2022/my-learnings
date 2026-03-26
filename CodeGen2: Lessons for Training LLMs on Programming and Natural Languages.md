

# CodeGen2: Lessons for Training LLMs on Programming and Natural Languages

---

## 🎯 1. THE ONE-LINER

**This paper tries to find the best single recipe for training one AI model that can both write code AND understand English, and honestly shares what worked and what didn't.**

---

## 🤔 2. THE PROBLEM (Why does this paper exist?)

- **The real problem:** Training large language models for code is **absurdly expensive**, and practitioners face too many confusing choices — which architecture? which training method? which data? Every wrong choice wastes millions of dollars in compute.
- **Why should you care?** Imagine you're opening a restaurant and need to decide: gas or electric stove? French or Italian menu? Indoor or outdoor seating? Each experiment costs $100,000 to test. You'd *kill* for someone who already tried every combination and told you what works. **That's this paper** — for AI code models.
- **Limitations of previous approaches:**
  - You needed **separate models** for different tasks (code generation, code understanding, text tasks) — expensive!
  - Prior work like Codex, CodeGen, InCoder each made **different design choices** but never systematically compared them
  - Claims like "infill training is free" (Bavarian et al., 2022) were **unverified** by independent groups
  - No open-source, well-tested recipe existed for training code LLMs end-to-end

---

## 💡 3. THE KEY IDEA (The "Aha!" moment)

**Core insight:** Instead of blindly picking a design, **systematically test 4 major design axes** (architecture, learning method, sampling method, data mix) on 1B-parameter models, find what works, distill it into **5 lessons**, then scale up.

**Everyday analogy — Baking a cake:**
- You want ONE cake recipe that works for birthdays, weddings, AND casual dessert
- You test: oven type (convection vs regular), mixing method (hand vs stand mixer), frosting technique, and flour blend
- Some combinations bomb, some shine — you **write down what failed and why**, then share the final recipe
- That's exactly what this paper does, but for training code AI models

**The final recipe (simplified):**
1. Use a **causal decoder** (like GPT) — not a Prefix-LM
2. Train with a **50/50 mix** of regular next-token-prediction AND span corruption
3. Apply span corruption **within file boundaries** (don't corrupt across files)
4. **Mix natural language and code data** together
5. **Train for multiple epochs** — seeing data again helps!

---

## 🏗️ 4. HOW IT WORKS (The Method - Layer by Layer)

### Step 1: Define 4 hypotheses to test
- **WHAT:** Formalize what they're testing: (1) Prefix-LM vs Causal Decoder, (2) mixed objectives, (3) "free lunch" infill, (4) NL+Code data mixing
- **WHY:** Without clear hypotheses, exploration is aimless
- **CONNECTS TO:** Each hypothesis becomes a lesson

### Step 2: Train many 1B-parameter models with different configurations
- **WHAT:** Train ~10+ model variants on different combos of architecture, objective, data
- **WHY:** 1B is cheap enough to experiment but large enough for meaningful signal
- **CONNECTS TO:** Results tell us which features to keep/drop

### Step 3: Evaluate on diverse benchmarks
- **WHAT:** Test each variant on:
  - **HumanEval** (code generation, left-to-right)
  - **HumanEval-Infill** (fill-in-the-middle code)
  - **XSum** (text summarization, few-shot)
  - **SuperGLUE** (NL understanding)
  - **CodeXGLUE** (code understanding)
- **WHY:** A single benchmark would be biased; need coverage across tasks
- **CONNECTS TO:** Determines which hypotheses to accept/reject

### Step 4: Distill findings into 5 lessons
- **WHAT:** 
  - Lesson 1: Prefix-LM **doesn't help** → use causal decoder
  - Lesson 2: Infill is **NOT free** → expect ~1 point drop
  - Lesson 3: A simple 50/50 mix of CLM + span corruption **works well**
  - Lesson 4: Mixing NL + code data is **promising** for one universal model
  - Lesson 5: Multi-epoch training **significantly helps** (CodeGen2.5: 5 epochs)
- **WHY:** Practitioners need actionable takeaways
- **CONNECTS TO:** Final training recipe

### Step 5: Scale up the final recipe to 1B, 3.7B, 7B, 16B
- **WHAT:** Train CodeGen2 family using the validated recipe on the Stack (permissively licensed code)
- **WHY:** Prove the recipe works at scale; release useful open-source models
- **CONNECTS TO:** Community gets models + code to build upon

### Step 6: Multi-epoch training → CodeGen2.5
- **WHAT:** Train 7B model for 5 epochs (1.4T tokens) on StarCoderData
- **WHY:** Tests whether repeated data still helps (it does, dramatically)
- **CONNECTS TO:** pass@1 jumps from 18.83 → 28.36 on HumanEval

---

## 📊 5. THE PROOF (Results & Experiments)

### Benchmarks used:
| Benchmark | Domain | Task Type |
|-----------|--------|-----------|
| HumanEval | Code | Zero-shot generation |
| HumanEval-Infill | Code | Fill-in-the-middle |
| XSum | NL | 1-shot summarization |
| SuperGLUE (BoolQ) | NL | Fine-tuned understanding |
| CodeXGLUE (Defect) | Code | Fine-tuned understanding |
| LAMBADA | NL | Zero-shot |
| PIQA | NL | Zero-shot |

### Key numbers:
- **CodeGen2.0-7B:** 19.09 pass@1 on HumanEval (competitive with InCoder-6.7B at 15.20)
- **CodeGen2.5-7B (multi-epoch):** **28.36 pass@1** — a **50% improvement** over CodeGen2.0-7B (18.83) just from training longer on repeated data
- **CodeGen2.0-7B infill:** 68.74 single-line, 38.88 multi-line (better than InCoder-6.7B's 66.69/34.62 despite being smaller)
- Prefix-LM vs Decoder: **~1.3 points worse** on HumanEval for Prefix-LM
- Infill training cost: **~1 point drop** in left-to-right HumanEval (not free!)

### Most impressive result in plain English:
**A 7B model trained for 5 epochs on the same data beats a 16B model trained for 1 epoch** — showing that seeing data multiple times can substitute for having a bigger model.

### Admitted limitations:
- Prefix-LM evaluation was on a **limited set of tasks** — might help on others
- The 16B model was still training at submission
- Couldn't fully disentangle what caused CodeGen2.5's improvement (more tokens? data augmentation? learning rate schedule?)
- Didn't reproduce the "free lunch" infill claim — could be implementation differences

---

## 🧩 6. KEY TERMS GLOSSARY

- **LLM** → Large Language Model; a neural network with billions of parameters trained on text
- **Causal Decoder** → A model that predicts the next word using only previous words (like GPT)
- **Prefix-LM** → A model where the first part of the input can look at all tokens (bidirectional), and the rest is causal — hybrid of encoder + decoder
- **Causal Language Modeling (CLM)** → Training by predicting the next token given all previous ones
- **Span Corruption** → Randomly masking out chunks of text and training the model to fill them back in (like T5)
- **Infill / Fill-in-the-Middle (FIM)** → Given code before AND after a gap, generate what goes in the gap
- **PSM (Prefix-Suffix-Middle)** → A sequence reordering trick: move the middle part to the end so a left-to-right model can learn infilling
- **HumanEval** → A benchmark of 164 Python programming problems to test code generation
- **pass@k** → The probability that at least 1 of k generated samples passes all test cases
- **The Stack** → A large dataset of permissively-licensed source code from GitHub
- **The Pile** → A large dataset of diverse English text (books, Wikipedia, web, etc.)
- **UL2** → "Unifying Language Learning Paradigms" — a training objective that mixes multiple denoising strategies
- **Zero-shot** → Using a model without any task-specific examples
- **Few-shot** → Giving the model a few examples of the task before asking it to perform
- **Neural Scaling Laws** → The observation that model performance predictably improves as you increase parameters/data/compute
- **Attention Mask** → Controls which tokens can "see" which other tokens during processing
- **File-level Corruption** → Applying span corruption only within individual files, not across file boundaries
- **Multi-epoch Training** → Training on the same data multiple times

---

## 🔗 7. HOW IT CONNECTS

### Intellectual family tree:
```
GPT (Radford 2018) ──→ GPT-3 (Brown 2020) ──→ Codex (Chen 2021)
                                                    ↓
BERT (Devlin 2019) ──→ T5 (Raffel 2020) ──→ UL2 (Tay 2022a)
          ↓                    ↓                    ↓
     CodeBERT           Prefix-LM concept     Mixed objectives
          ↓                    ↓                    ↓
     CodeGen (Nijkamp 2022)   InCoder (Fried 2022)  ↓
          ↓                    ↓                    ↓
          └────────── CodeGen2 (this paper) ────────┘
                              ↓
                    CodeGen2.5 (multi-epoch)
                              ↓
                    StarCoder (Li et al., 2023)
```

### Who would use this:
- **ML engineers** building code generation tools (Copilot-like products)
- **Researchers** designing new LLM training recipes
- **Companies** wanting to train their own code LLMs efficiently on permissive data
- **Open-source community** needing well-documented baselines

### Future work enabled:
- Better multi-epoch training strategies with ablated factors
- Improved infill training that truly has "zero cost"
- Cross-modal (code + text) models for conversational coding assistants
- Data mixture optimization for LLM pre-training

---

## ⚖️ 8. CRITICAL ANALYSIS

### Hidden assumptions:
- **HumanEval is a good proxy for code generation quality** — it's only 164 Python problems and may not represent real-world coding
- **1B ablations transfer to larger scales** — some findings at 1B might not hold at 16B+ (they acknowledge but don't verify all findings)
- **Permissive license data ≈ all code data** — filtering to permissive licenses removes a lot of code; findings may differ on larger/richer datasets

### Weaknesses the authors DON'T mention:
- **No comparison with Codex/GPT-4 class models** on the same benchmarks — hard to contextualize how competitive CodeGen2 really is
- **Limited NL evaluation** — only XSum, LAMBADA, PIQA; no rigorous NLU benchmark suite
- **No analysis of training instability** — multi-epoch training can cause overfitting; no evidence of monitoring for this
- **Confounds in CodeGen2.5** — they changed data (StarCoderData vs Stack), epochs, AND learning rate schedule simultaneously, making it impossible to attribute gains

### Is the evaluation fair?
- Mostly yes, but the **InCoder comparison uses different truncation heuristics**, which they note. This makes direct infill comparison imperfect.
- The Prefix-LM evaluation may be **underselling it** — they didn't test it with encoder-decoder tasks or with fine-tuning at scale

### Would this work in the real world at scale?
- **Yes, largely.** The recipe is simple, well-documented, and tested at multiple scales (up to 16B). The open-source release makes it reproducible.
- **Caveat:** The "mix NL + code" recipe doesn't *outperform* domain-specific models; it's a tradeoff for convenience.

---

## 📝 9. MEMORY ANCHORS

### Memorable metaphor:
**CodeGen2 is like a cooking show where the chef tries to make one "universal recipe" that works for breakfast, lunch, AND dinner — and honestly tells you "the universal pancake-steak-salad didn't work, but here's what I learned, and here's a damn good steak recipe."**

### 3 bullets that capture 80%:
- **Stick with a plain causal decoder** (not Prefix-LM), trained with a 50/50 mix of next-token-prediction and within-file span corruption
- **Infill isn't free** (~1 point cost) but is **cheap enough** to be worth it for practical code editing
- **Multi-epoch training is surprisingly powerful** — a 7B model trained 5x on the same data (CodeGen2.5) beats a 16B model trained 1x

### Comprehension question:
> *Why did the authors conclude that Prefix-LM is not beneficial, and what conflicting evidence did they find across different datasets?*

---

## 🗺️ 10. VISUAL MENTAL MAP

```
PROBLEM: Training code LLMs is expensive & design choices are unclear
    │
    ▼
HYPOTHESES (4 axes to unify)
    │
    ├── H1: Prefix-LM unifies encoder+decoder? ──→ ❌ No clear benefit
    │                                                    (worse on multilingual data)
    │
    ├── H2: Infill training is free? ────────────→ ❌ ~1pt cost on HumanEval
    │                                                    (cheap, not free)
    │
    ├── H3: Mix CLM + span corruption? ──────────→ ✅ Works! Simple 50/50 mix
    │                                                    + file-level corruption
    │
    └── H4: Mix NL + code data? ─────────────────→ ✅ Promising! Close to
         │                                              domain-specific models
         │
         └── Bonus: Multi-epoch training? ───────→ ✅✅ Huge gains!
                                                        18.83 → 28.36 pass@1
    │
    ▼
FINAL RECIPE
    │
    ├── Architecture: Causal Decoder (standard GPT-style)
    ├── Objective:    50% CLM + 50% file-level span corruption
    ├── Data:         The Stack (code) + optional NL mix
    └── Epochs:       Multiple epochs are fine (even 5x)
    │
    ▼
CODEGEN2 FAMILY: 1B, 3.7B, 7B, 16B (open-source)
CODEGEN2.5: 7B, 5 epochs, 28.36 pass@1 🏆
```

---

## 🛠️ 11. IMPLEMENTATION SKETCH

### Pseudocode (core training loop):
```python
for batch in dataloader(Stack + Pile, ratio=0.5):
    for sequence in batch:
        # Decide objective with 50/50 coin flip
        if random() < 0.5:
            # Standard causal language modeling
            input_seq = sequence
            target = sequence[1:]  # next token prediction
        else:
            # Span corruption (within file boundaries)
            files = split_by_file_boundary(sequence)
            corrupted_seq = []
            targets = []
            for file in files:
                ratio = uniform(0.0, 1.0)  # dynamic mask ratio
                span_len = sample_span_length()
                masked_file, spans = corrupt_spans(
                    file, ratio, span_len
                )
                corrupted_seq.append(masked_file)
                targets.append(spans)
            input_seq = concat(corrupted_seq + targets)
            target = input_seq[1:]  # still next-token prediction

        loss = cross_entropy(model(input_seq), target)
        loss.backward()
    optimizer.step()
```

### Frameworks/libraries needed:
- **JAX + Flax** (or PyTorch) for model training
- **FSDP / model parallelism** for multi-GPU training (16B model needs it)
- **HuggingFace Transformers** for model architecture
- **The Stack / StarCoderData** for code data, **The Pile** for NL data

### Estimated compute to reproduce:
- **1B model:** ~100-200 GPU-hours (A100s) — accessible for a research lab
- **7B model:** ~2,000-5,000 GPU-hours (A100s)
- **16B model:** ~10,000+ GPU-hours (A100s)
- **CodeGen2.5 (7B, 1.4T tokens):** ~15,000+ GPU-hours — substantial cluster needed
- **Total ablation study (all 1B experiments):** ~1,000-2,000 GPU-hours
