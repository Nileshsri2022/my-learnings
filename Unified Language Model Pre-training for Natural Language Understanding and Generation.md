

# UNILM: Unified Language Model Pre-training for NLU and NLG

---

## 🎯 1. THE ONE-LINER

**Instead of needing separate AI brains for reading/understanding text and writing/generating text, this paper builds ONE brain that can do both — by training it with different "blinders" (attention masks) on the same neural network.**

---

## 🤔 2. THE PROBLEM (Why does this paper exist?)

- **The problem:** Before UNILM, models were good at EITHER understanding text (like BERT) OR generating text (like GPT), but **not both**. If you wanted to do both, you needed separate models with separate training.
  - BERT = great reader, terrible writer (its bidirectional nature makes generation awkward)
  - GPT = decent writer, weaker reader (only looks left-to-right, misses context)

- **Why should you care?** Imagine you hire two employees — one can only read and answer questions about documents, another can only write summaries. **Wouldn't it be better to have ONE employee who can do both?** That saves money (compute), time (training), and the employee who does both actually becomes better at each task because reading helps writing and vice versa.

- **Limitations of previous approaches:**
  - **BERT** uses bidirectional attention → every token sees everything → **can't do autoregressive generation** (you'd be "cheating" by seeing future words)
  - **GPT** uses left-to-right attention → good for generation → **misses right-side context** for understanding tasks
  - **ELMo** concatenates two separate unidirectional LMs → no deep cross-directional interaction
  - Having separate models is **expensive** and the representations aren't shared

---

## 💡 3. THE KEY IDEA (The "Aha!" moment)

**Core insight:** You don't need different architectures for different language tasks — you just need **different attention masks on the SAME Transformer**. The mask controls what each word can "see" during processing.

### Everyday Analogy: The Exam Room with Adjustable Blinders

Imagine students sitting in a row taking different exams:
- **Bidirectional LM (BERT-style):** No blinders. Everyone can see everyone else's paper → best for understanding
- **Left-to-right LM (GPT-style):** Each student can only see papers to their LEFT → good for writing one word at a time
- **Seq-to-Seq LM:** Students in Group 1 (source) see each other freely; students in Group 2 (target) can see all of Group 1 but only LEFT neighbors in their own group → perfect for "read an article, then write a summary"

**Same students, same classroom, same teacher — just different blinders!**

### ASCII Diagram of the Three Mask Patterns:

```
BIDIRECTIONAL LM         LEFT-TO-RIGHT LM      SEQ-TO-SEQ LM
(all see all)            (see only left)       (mixed)

  1 2 3 4 5               1 2 3 4 5            S1 S1 | S2 S2 S2
1[✓ ✓ ✓ ✓ ✓]           1[✓ ✗ ✗ ✗ ✗]        S1[✓  ✓ | ✗  ✗  ✗]
2[✓ ✓ ✓ ✓ ✓]           2[✓ ✓ ✗ ✗ ✗]        S1[✓  ✓ | ✗  ✗  ✗]
3[✓ ✓ ✓ ✓ ✓]           3[✓ ✓ ✓ ✗ ✗]        --|-----|---------|
4[✓ ✓ ✓ ✓ ✓]           4[✓ ✓ ✓ ✓ ✗]        S2[✓  ✓ | ✓  ✗  ✗]
5[✓ ✓ ✓ ✓ ✓]           5[✓ ✓ ✓ ✓ ✓]        S2[✓  ✓ | ✓  ✓  ✗]
                                              S2[✓  ✓ | ✓  ✓  ✓]
✓ = can attend, ✗ = blocked
```

---

## 🏗️ 4. HOW IT WORKS (The Method - Layer by Layer)

### Step 1: Input Representation
- **WHAT:** Tokenize input text using WordPiece. Add `[SOS]` at the start and `[EOS]` at segment ends. Compute input vector = token embedding + position embedding + segment embedding.
- **WHY:** The model needs a numerical representation. Segment embeddings also act as **LM type identifiers** — telling the model which mode (bidirectional, unidirectional, seq2seq) it's operating in.
- **HOW it connects:** These vectors feed into the Transformer stack.

### Step 2: Shared Multi-Layer Transformer
- **WHAT:** A 24-layer Transformer (same architecture as BERT-Large: 1024 hidden, 16 heads, ~340M params). Each layer computes Q, K, V projections and self-attention.
- **WHY:** The Transformer is the shared backbone. **Same weights for ALL LM types** — this forces the model to learn general-purpose representations.
- **HOW it connects:** The magic happens in the attention mask M added to the attention scores before softmax.

### Step 3: Self-Attention Masking (THE KEY MECHANISM)
- **WHAT:** A mask matrix M is added to attention scores:
  - `M_ij = 0` → token i CAN attend to token j
  - `M_ij = -∞` → token i CANNOT attend to token j (softmax drives this to 0)
- **WHY:** This single mechanism lets one network behave as **four different LMs**:
  - **Bidirectional:** M = all zeros (full attention)
  - **Left-to-right:** M = upper triangular is -∞ (causal mask)
  - **Right-to-left:** M = lower triangular is -∞
  - **Seq-to-Seq:** Source tokens have full attention among themselves; target tokens attend to all source + left context only
- **HOW it connects:** Different masks → different context → different LM behaviors → all optimized jointly

### Step 4: Pre-training with Cloze Tasks
- **WHAT:** Randomly mask 15% of tokens (replace with `[MASK]`), predict them. Same masking procedure as BERT: 80% `[MASK]`, 10% random token, 10% keep original. Also 20% of the time mask bigrams/trigrams.
- **WHY:** Cloze prediction is the **universal training signal** — it works for both unidirectional and bidirectional settings (the mask controls what context is available).
- **Sampling schedule:** Each batch: 1/3 bidirectional, 1/3 seq2seq, 1/6 left-to-right, 1/6 right-to-left.
- **HOW it connects:** The model learns representations useful for all four LM types simultaneously.

### Step 5: Fine-tuning for Downstream Tasks
- **For NLU:** Use bidirectional mask. Take `[SOS]` vector → feed to task-specific classifier (like BERT).
- **For NLG (seq2seq):** Pack source + target as `[SOS] S1 [EOS] S2 [EOS]`. Use seq2seq mask. Mask tokens in target, learn to predict them. At inference, generate autoregressively with beam search.

---

## 📊 5. THE PROOF (Results & Experiments)

### Benchmarks Tested:

| Task Type | Dataset | Metric |
|-----------|---------|--------|
| NLU | GLUE (9 tasks) | Various |
| Extractive QA | SQuAD 2.0, CoQA | EM, F1 |
| Abstractive Summarization | CNN/DailyMail, Gigaword | ROUGE |
| Question Generation | SQuAD 1.1 | BLEU-4 |
| Generative QA | CoQA | F1 |
| Dialog Response | DSTC7 | NIST-4 |

### Key Numbers:

| Task | Previous Best | UNILM | Improvement |
|------|-------------|-------|-------------|
| CNN/DailyMail ROUGE-L | 38.47 | **40.51** | +2.04 |
| Gigaword ROUGE-L | 34.89 | **35.75** | +0.86 |
| CoQA Generative QA F1 | 45.4 | **82.5** | +37.1 (!!) |
| SQuAD Question Gen BLEU-4 | 18.37 | **22.12** | +3.75 |
| DSTC7 Dialog NIST-4 | 2.523 | **2.669** | (surpasses human: 2.65) |
| GLUE Score | 80.5 (BERT) | **80.8** | comparable |
| SQuAD 2.0 F1 | 81.8 (BERT) | **83.4** | +1.6 |

### Most impressive result in plain English:
**On the CoQA generative QA task, UNILM scored 82.5 F1, nearly DOUBLING the previous best generative model (45.4).** On dialog response generation, UNILM **surpassed human performance** (2.669 vs 2.650 NIST-4).

### Limitations/Failure Cases:
- The authors **don't report failure cases explicitly**
- GLUE improvement over BERT is marginal (80.8 vs 80.5) — the NLU gains are modest
- Only tested on English
- Model initialized from BERT-Large, so it's unclear how much improvement comes from the unified training vs. the BERT initialization
- No ablation separating the contribution of each LM objective

---

## 🧩 6. KEY TERMS GLOSSARY

- **Transformer** → A neural network architecture that processes all words simultaneously using attention mechanisms
- **Self-attention** → A mechanism where each word looks at other words to understand context
- **Attention mask (M)** → A matrix that blocks certain words from seeing other words during self-attention
- **Cloze task** → Fill-in-the-blank: hide a word and predict it from context
- **Bidirectional LM** → Language model where each word can see ALL surrounding words (left + right)
- **Unidirectional LM** → Language model where each word can only see words in ONE direction (left or right)
- **Sequence-to-sequence (Seq2Seq) LM** → Model that reads a full input, then generates output one word at a time
- **NLU** → Natural Language Understanding — tasks like classification, question answering, entailment
- **NLG** → Natural Language Generation — tasks like summarization, question generation, dialog
- **WordPiece** → A tokenization method that breaks words into subword units (e.g., "playing" → "play" + "##ing")
- **Fine-tuning** → Taking a pre-trained model and training it a little more on a specific task
- **Beam search** → A decoding strategy that keeps track of multiple best candidates while generating text
- **ROUGE** → A metric that measures overlap between generated and reference summaries
- **BLEU** → A metric that measures overlap between generated and reference text (common for translation/generation)
- **Label smoothing** → A regularization trick that softens the training targets from hard 0/1 to slightly smoother values
- **GLUE benchmark** → A collection of 9 NLU tasks used to evaluate language understanding models
- **[SOS] / [EOS]** → Special tokens marking start-of-sequence and end-of-sequence
- **[MASK]** → Special token that replaces words the model must predict during training
- **Segment embedding** → A learned vector that tells the model which segment (first or second) a token belongs to
- **Mixed precision training** → Using both 16-bit and 32-bit numbers to speed up training

---

## 🔗 7. HOW IT CONNECTS

### Intellectual Family Tree:
```
ELMo (2018)         GPT (2018)          BERT (2018)
   │                    │                   │
   │ (contextual        │ (left-to-right    │ (bidirectional
   │  embeddings)       │  Transformer)     │  Transformer)
   └────────────────────┴───────────────────┘
                        │
                    UNILM (2019) ← combines ALL of these
                        │
           ┌────────────┼────────────┐
           ▼            ▼            ▼
     MASS (2019)    T5 (2019)    BART (2019)
  (seq2seq only)  (text-to-text) (denoising autoencoder)
```

### Related Papers:
- **BERT** (Devlin et al., 2018): UNILM directly builds on BERT's architecture and is initialized from BERT-Large
- **GPT** (Radford et al., 2018): Inspired the left-to-right LM component
- **MASS** (Song et al., 2019): A contemporary that also pre-trains seq2seq models, but uses a separate encoder-decoder; UNILM outperforms MASS significantly
- **T5** (Raffel et al., 2019): Later work that takes the "unified" idea further with text-to-text framing
- **BART** (Lewis et al., 2019): Another contemporary using denoising for seq2seq pre-training

### Who would use this:
- **NLP engineers** who need one model for both understanding AND generation tasks
- **Researchers** working on summarization, question generation, dialog systems
- **Companies** wanting to reduce the cost of deploying multiple specialized models

### Future work enabled:
- Cross-lingual unified models (mentioned by authors)
- Multi-task fine-tuning for NLU+NLG simultaneously
- Larger-scale unified pre-training (led to UniLMv2, later models)

---

## ⚖️ 8. CRITICAL ANALYSIS

### Hidden Assumptions:
- **Shared parameters are sufficient** — assumes bidirectional encoding and unidirectional decoding don't need specialized architectures
- **The training ratio (1/3, 1/3, 1/6, 1/6) is near-optimal** — no extensive ablation on this
- **BERT initialization is crucial** — UNILM starts from BERT-Large weights. How much of the performance comes from BERT's pre-training vs. the unified objectives?

### Weaknesses the authors DON'T mention:
- **No true encoder-decoder architecture** — both "encoder" and "decoder" share the same parameters and same depth. A real encoder-decoder (like T5/BART) has separate stacks, which may be better for generation tasks
- **Quadratic attention complexity** — source and target are concatenated into one sequence, making the attention O((|S1|+|S2|)²) instead of O(|S1|² + |S2|² + |S1|·|S2|) for separate encoder-decoder
- **No ablation study** — Which LM objective contributes most? What if you drop one?
- **Cherry-picked generation examples** — Appendix A admits "we sampled 10 times and hand-picked the best"
- **Initialization dependency** — Starting from BERT-Large confounds results; hard to know if UNILM's pre-training actually helps vs. just doing more BERT training
- **Limited to 512 tokens** — inherits BERT's length limitation

### Is the evaluation fair?
- Mostly fair — uses standard benchmarks and metrics
- But comparing against BERT on NLU when UNILM is initialized from BERT and trained further is **not a perfectly fair comparison** (extra training could help regardless of the unified objectives)
- NLG comparisons are strong — clearly SOTA on multiple benchmarks

### Would this work in the real world at scale?
- **Yes, with caveats.** The model proved influential (2000+ citations). The unified approach saves deployment cost but inference for generation is still autoregressive. Later models (T5, BART) adopted similar ideas with arguably cleaner formulations.

---

## 📝 9. MEMORY ANCHORS

### Memorable Metaphor:
> **UNILM is like a Swiss Army knife for language** — one tool body with different blades (attention masks) you flip out depending on whether you need to read (understand), write (generate), or translate/summarize (seq2seq). Previous models were like having a separate knife, screwdriver, and corkscrew.

### 3 Bullet Points (80% of the paper):
1. **One Transformer, many masks:** UNILM uses a single shared Transformer but swaps self-attention masks to behave as a bidirectional, unidirectional, or seq2seq language model
2. **Joint pre-training:** All three LM objectives are trained together (1/3 bidirectional, 1/3 seq2seq, 1/6 L→R, 1/6 R→L), so the shared parameters learn versatile representations
3. **SOTA on generation, matches BERT on understanding:** New SOTA on 5 NLG tasks (summarization, QG, generative QA, dialog), while matching BERT-Large on GLUE and QA

### Comprehension Question:
> *How does UNILM use the exact same Transformer weights to function as both a bidirectional encoder (like BERT) and a left-to-right decoder (like GPT)?*

**Answer:** By changing only the self-attention mask matrix M. For bidirectional, M is all zeros (every token attends to every other). For left-to-right, M has -∞ in the upper triangle (blocking attention to future tokens). The same Q, K, V weight matrices are used — only the mask differs.

---

## 🗺️ 10. VISUAL MENTAL MAP

```
┌─────────────────────── PROBLEM ───────────────────────┐
│ BERT = great understanding, bad generation            │
│ GPT  = decent generation, weaker understanding        │
│ Need separate models → expensive, no shared learning  │
└──────────────────────────┬────────────────────────────┘
                           │
                           ▼
┌─────────────────────── KEY IDEA ──────────────────────┐
│ Same Transformer + different attention masks =        │
│ different LM behaviors (bidirectional/unidirectional/  │
│ seq2seq) all sharing ONE set of parameters            │
└──────────────────────────┬────────────────────────────┘
                           │
                           ▼
┌─────────────────────── METHOD ────────────────────────┐
│                                                       │
│  Input: [SOS] tokens... [EOS] (tokens...) [EOS]       │
│                    │                                  │
│                    ▼                                  │
│  ┌──────────────────────────┐                         │
│  │   24-Layer Transformer   │ ← shared weights        │
│  │   (BERT-Large size)      │                         │
│  └──────────┬───────────────┘                         │
│             │                                         │
│     ┌───────┼───────┬───────────┐                     │
│     ▼       ▼       ▼           ▼                     │
│  Bidir.  L→R LM   R→L LM   Seq2Seq LM               │
│   mask    mask     mask      mask                     │
│  (1/3)   (1/6)    (1/6)     (1/3)                     │
│     └───────┴───────┴───────────┘                     │
│             │                                         │
│    Joint Cloze Loss (predict [MASK])                  │
│                                                       │
└──────────────────────────┬────────────────────────────┘
                           │
                           ▼
┌─────────────────────── FINE-TUNE ─────────────────────┐
│                                                       │
│  NLU tasks ──→ use bidirectional mask                  │
│  NLG tasks ──→ use seq2seq mask                       │
│                                                       │
└──────────────────────────┬────────────────────────────┘
                           │
                           ▼
┌─────────────────────── RESULTS ───────────────────────┐
│  NLU: GLUE 80.8 (≈ BERT), SQuAD 83.4 F1 (> BERT)    │
│  NLG: CNN/DM ROUGE-L 40.51 (+2.04 SOTA)              │
│       CoQA Gen. QA 82.5 F1 (+37.1 !!!)               │
│       DSTC7 Dialog 2.67 NIST (> human 2.65)           │
└───────────────────────────────────────────────────────┘
```

---

## 🛠️ 11. IMPLEMENTATION SKETCH

### Pseudocode (~15 lines):
```python
# PRE-TRAINING
for batch in data_loader:
    # Sample LM type with probability 1/3, 1/3, 1/6, 1/6
    lm_type = random.choice(['bidir', 'seq2seq', 'l2r', 'r2l'],
                             p=[1/3, 1/3, 1/6, 1/6])
    
    tokens, segments = prepare_input(batch, lm_type)
    mask_positions = random_mask(tokens, prob=0.15)
    tokens[mask_positions] = [MASK]  # (80/10/10 rule)
    
    # Build attention mask matrix
    M = build_attention_mask(lm_type, len(tokens), segments)
    
    # Forward through shared Transformer
    H = transformer(tokens, segments, attention_mask=M)
    
    # Predict masked tokens
    logits = softmax_classifier(H[mask_positions])
    loss = cross_entropy(logits, original_tokens[mask_positions])
    loss.backward()
    optimizer.step()

# FINE-TUNING (e.g., summarization)
input = "[SOS] " + document + " [EOS] " + summary + " [EOS]"
M = build_seq2seq_mask(source_len, target_len)
# Mask target tokens, train to predict them
```

### Frameworks/Libraries:
- **PyTorch** (the official implementation uses PyTorch)
- **Hugging Face Transformers** (for BERT-Large initialization)
- **NVIDIA Apex** (for mixed-precision training)

### Estimated Compute Cost:
- **8× NVIDIA V100 32GB GPUs**
- **~770,000 training steps**
- **~7 hours per 10,000 steps** → **~540 GPU-hours total** (~22.5 days on 1 GPU, ~2.8 days on 8 GPUs)
- Pre-training data: English Wikipedia + BookCorpus (~16GB text)
- Estimated cloud cost: **~$2,000-4,000** at 2019 cloud GPU pricing
