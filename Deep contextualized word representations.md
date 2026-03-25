

# Deep Contextualized Word Representations (ELMo)
**Peters et al., 2018 — Allen Institute for AI & University of Washington**

---

## 🎯 1. THE ONE-LINER

**This paper teaches computers to understand that the same word can mean different things depending on the sentence it's in**, by looking at all the layers of a language-reading neural network instead of just the top one.

---

## 🤔 2. THE PROBLEM (Why does this paper exist?)

- **The problem:** Before ELMo, word embeddings like Word2Vec and GloVe gave every word **one fixed vector**, regardless of context. The word "bank" got the same representation whether you said "river bank" or "bank account."
- **Why should anyone care?** Imagine you're a dictionary that can only store ONE definition per word. When someone says "I need to **play** the piano" vs. "Let's go watch a **play**," you can't tell them apart. That's what old word embeddings were like — **a dictionary with no context**.
- **Limitations of previous approaches:**
  - **GloVe/Word2Vec** → One vector per word, completely context-independent
  - **context2vec** (Melamud et al., 2016) → Used bidirectional LSTM but only the **top layer** output
  - **CoVe** (McCann et al., 2017) → Used a machine translation encoder for context, but was **limited by the size of parallel corpora** (needed translated text pairs)
  - **TagLM** (Peters et al., 2017) → Used only the **top layer** of a language model, missing richer information in lower layers

---

## 💡 3. THE KEY IDEA (The "Aha!" moment)

**Core insight:** Different layers of a deep language model capture **different types of linguistic information** — lower layers learn syntax (grammar), higher layers learn semantics (meaning). Instead of throwing away this rich layered information and using only the top layer, **let each downstream task learn its own custom weighted mixture of ALL layers**.

### Everyday Analogy: The Layer Cake 🎂
Think of a deep neural network like a **multi-layer cake**:
- **Bottom layer** (vanilla) = basic grammar/structure ("is this a noun or verb?")
- **Middle layer** (chocolate) = richer patterns
- **Top layer** (frosting) = meaning/semantics ("what does this word mean HERE?")

Previous approaches only ate the frosting. ELMo says: **"Give me the whole cake, and let me decide how much of each layer to eat based on what I'm hungry for (the task)."**

### Step-by-step for a friend:
1. Train a big bidirectional language model on tons of text
2. For any new sentence, run it through this model
3. Instead of taking just the final output, **grab the representations from every layer**
4. Let each task learn: "I need 60% syntax layer + 30% semantics layer + 10% raw characters"
5. Concatenate this custom blend with existing word embeddings → **instant upgrade to any model!**

---

## 🏗️ 4. HOW IT WORKS (The Method - Layer by Layer)

### Step 1: Train a Bidirectional Language Model (biLM)
- **WHAT:** Train a 2-layer biLSTM that reads text both **forward** (left→right, predicting next word) and **backward** (right→left, predicting previous word)
- **WHY:** Reading in both directions captures the full context around each word
- **Details:** 
  - Uses character-level CNN input (handles any word, even misspellings)
  - 4096 LSTM units, 512-dim projections, residual connections
  - Trained on **1 Billion Word Benchmark** (~30M sentences)
  - Forward and backward LSTMs share token embedding and softmax weights, but have **separate LSTM parameters**
- **HOW it connects:** This pretrained model becomes a frozen "feature extractor" for all downstream tasks

### Step 2: Extract Representations from ALL Layers
- **WHAT:** For each word in a sentence, collect **2L+1 = 5 vectors** (for L=2 layers):
  - Layer 0: Character-based token embedding (context-independent)
  - Layer 1: First biLSTM hidden state (forward + backward concatenated)
  - Layer 2: Second biLSTM hidden state (forward + backward concatenated)
- **WHY:** Each layer encodes different information — lower = syntax, higher = semantics
- **HOW it connects:** These stacked vectors form the "ingredient list" for the ELMo recipe

### Step 3: Compute Task-Specific Weighted Combination
- **WHAT:** For each task, learn **softmax-normalized weights** (s₀, s₁, s₂) and a **scalar γ**:
  ```
  ELMo_k = γ × (s₀·h₀ + s₁·h₁ + s₂·h₂)
  ```
- **WHY:** Different tasks need different mixes — NER might want more syntax, QA might want more semantics
- **γ** is critical: it scales ELMo to match the distribution of the task's own representations
- **Optional:** Layer normalization on each biLM layer before weighting; regularization (λ||w||²) to keep weights close to uniform average
- **HOW it connects:** Produces a single vector per word that replaces/augments existing embeddings

### Step 4: Plug into Any Existing Model
- **WHAT:** Concatenate ELMo with the task model's existing input: **[x_k ; ELMo_k]**
- **WHY:** This is a **drop-in enhancement** — no need to redesign the model architecture
- **Optional:** Also add ELMo at the **output** of the task's RNN: **[h_k ; ELMo_k]** (helps for tasks with attention layers like SNLI, SQuAD)
- Add dropout to ELMo representations for regularization
- **Freeze the biLM weights** during task training (only train the mixing weights s and γ)

---

## 📊 5. THE PROOF (Results & Experiments)

### Benchmarks & Results (Table 1):

| Task | Metric | Baseline → +ELMo | Relative Error Reduction |
|------|--------|-------------------|--------------------------|
| **SQuAD** (QA) | F1 | 81.1 → **85.8** | **24.9%** |
| **SNLI** (Entailment) | Acc | 88.0 → **88.7** | 5.8% |
| **SRL** (Semantic Roles) | F1 | 81.4 → **84.6** | 17.2% |
| **Coref** (Coreference) | Avg F1 | 67.2 → **70.4** | 9.8% |
| **NER** (Named Entities) | F1 | 90.15 → **92.22** | 21% |
| **SST-5** (Sentiment) | Acc | 51.4 → **54.7** | 6.8% |

### Most Impressive Results in Plain English:
- **New state-of-the-art on ALL six tasks** just by adding ELMo — no architecture changes needed
- **SQuAD: 4.7% F1 improvement** (vs. CoVe's 1.8% improvement on same baseline — **2.6× better**)
- **Sample efficiency:** SRL model with ELMo + 1% of training data ≈ baseline with 10% of training data
- **Training speed:** SRL baseline needs 486 epochs to peak; **with ELMo, it surpasses that in just 10 epochs** (98% fewer updates!)

### Key Ablation Findings:
- **All layers > top layer only** (Table 2: SQuAD 85.2 vs 84.7)
- **Task-learned weights (λ=0.001) > uniform average (λ=1) > last only** in most cases
- **biLM >> CoVe** on both WSD (69.0 vs 64.7 F1) and POS tagging (97.3 vs 93.3 accuracy)
- **Lower layers better for syntax** (POS tagging: Layer 1 = 97.3% > Layer 2 = 96.8%)
- **Higher layers better for semantics** (WSD: Layer 2 = 69.0% > Layer 1 = 67.4%)

### Limitations Admitted:
- Only uses a **2-layer biLSTM** (not deeper architectures)
- biLM perplexity (39.7) worse than the full CNN-BIG-LSTM (30.0) due to halved dimensions
- Where to insert ELMo (input vs. output) is **task-dependent** — requires experimentation

---

## 🧩 6. KEY TERMS GLOSSARY

**ELMo** → Embeddings from Language Models; contextualized word vectors from a deep biLM

**biLM** → Bidirectional Language Model; reads text both forward and backward to predict words

**LSTM** → Long Short-Term Memory; a type of neural network that processes sequences and remembers long-range dependencies

**biLSTM** → Bidirectional LSTM; two LSTMs running in opposite directions, outputs concatenated

**Context-independent embedding** → A fixed vector for each word regardless of surrounding words (e.g., GloVe)

**Contextualized embedding** → A vector for each word that changes depending on the surrounding sentence

**Polysemy** → When a single word has multiple meanings (e.g., "bank" = financial institution or river edge)

**Language model** → A neural network trained to predict the next (or previous) word in a sequence

**Softmax-normalized weights** → Weights that are forced to sum to 1, like probabilities

**Layer normalization** → Technique to normalize the outputs of a neural network layer to have consistent statistics

**Character convolutions (CNN)** → Using small filters over individual characters to build word representations, handling any word including rare/unseen ones

**Residual connection** → A shortcut that adds the input of a layer to its output, helping deep networks train

**Highway layers** → A gating mechanism that lets the network learn how much to transform vs. pass through information

**CoVe** → Contextualized Word Vectors; similar idea but uses a machine translation encoder instead of a language model

**CRF (Conditional Random Field)** → A model for sequence labeling that considers the dependencies between adjacent labels

**Perplexity** → A measure of how well a language model predicts text (lower = better)

**WSD (Word Sense Disambiguation)** → The task of figuring out which meaning of a word is intended in context

**POS tagging** → Labeling each word with its part of speech (noun, verb, adjective, etc.)

**Semi-supervised learning** → Using both labeled and unlabeled data for training

---

## 🔗 7. HOW IT CONNECTS

### Intellectual Family Tree:
```
Word2Vec (2013) & GloVe (2014)
  └── Fixed word embeddings (context-free)
       ├── context2vec (2016): biLSTM context, top layer only
       ├── CoVe (2017): MT encoder context, top layer only  
       ├── TagLM (Peters et al., 2017): biLM context, top layer only
       └── ★ ELMo (2018): biLM context, ALL LAYERS ★
            ├── GPT (Radford et al., 2018): Transformer LM, fine-tuning
            ├── BERT (Devlin et al., 2019): Bidirectional Transformer, fine-tuning
            └── GPT-2/3, T5, etc.: Scaling up transformers
```

### Who Would Use This:
- **NLP researchers** wanting to boost any existing model with minimal effort
- **Industry practitioners** with limited labeled data (ELMo dramatically improves sample efficiency)
- **Linguists** studying what neural networks learn about language structure

### Future Work Enabled:
- **BERT** (2019) directly builds on ELMo's insight that bidirectional context + deep representations matter, but uses Transformers instead of LSTMs and fine-tunes the whole model
- **GPT** takes the language model pretraining idea but with unidirectional Transformers
- The broader **"pretrain then transfer"** paradigm in NLP was catalyzed by ELMo

---

## ⚖️ 8. CRITICAL ANALYSIS

### Hidden Assumptions:
- Assumes a **2-layer biLSTM is deep enough** to separate syntax and semantics — what about 4, 8, or 12 layers?
- Assumes the biLM's learned representations are **general enough** to transfer across all domains
- Assumes **linear combination** of layers is sufficient (what about nonlinear combinations?)

### Weaknesses the Authors DON'T Mention:
- **Computational overhead at inference time:** Must run the entire biLM for every sentence, which is slow compared to just looking up static embeddings
- **LSTM architecture is sequential** — can't be parallelized like Transformers (this became BERT's advantage)
- **Only 2 layers** — the "deep" in the title is somewhat modest by modern standards
- **No fine-tuning of the biLM weights** — freezing the biLM limits adaptation. BERT later showed that fine-tuning the entire model is even better
- The paper doesn't discuss **memory requirements** of storing all intermediate representations

### Is the Evaluation Fair?
- ✅ **Six diverse tasks** is impressively comprehensive for 2018
- ✅ Proper ablations on layer weighting, ELMo placement, training set size
- ⚠️ Some baselines were their own reimplementations (might not be perfectly tuned)
- ⚠️ Comparison with CoVe is somewhat unfair since CoVe needs parallel corpora while ELMo uses monolingual data (much more abundant)

### Real-World Scalability:
- **Moderate concern:** Running a large biLSTM for every input adds latency
- **Positive:** Can be pre-computed and cached for fixed text corpora
- **Superseded:** In practice, BERT/Transformers replaced ELMo within ~1 year due to better performance and (eventually) efficient implementations

---

## 📝 9. MEMORY ANCHORS

### Memorable Metaphor:
> **ELMo is like a team of expert witnesses at a trial.** Traditional embeddings give you one character reference letter per person. ELMo gives you testimony from a grammar expert (lower layer), a meaning expert (upper layer), and a character analyst (token layer) — and **the judge (task) decides how much weight to give each witness.**

### 3 Bullets That Capture 80%:
- 🔑 **Word representations should change with context** — ELMo creates different vectors for "play" in "play music" vs. "watch a play" by using a bidirectional language model
- 🔑 **Use ALL layers, not just the top** — lower layers capture syntax, upper layers capture semantics, and each task learns its own weighted mix
- 🔑 **Drop-in upgrade** — just concatenate ELMo vectors with existing embeddings and get state-of-the-art across 6 diverse NLP tasks with up to 25% relative error reduction

### Comprehension Check Question:
> *Why does using a learned weighted combination of all biLM layers outperform using just the top layer, and how does this relate to the different types of linguistic information encoded at different depths?*

---

## 🗺️ 10. VISUAL MENTAL MAP

```
PROBLEM                          METHOD                              RESULT
═══════                          ══════                              ══════

"bank" = 💰 or 🏞️?         ┌─────────────────────┐
Same vector for both!         │  PRETRAIN biLM       │
                              │  (1B Word Benchmark)  │
Traditional embeddings        │                       │
give ONE fixed vector         │  Forward LSTM ──────► │
per word                      │  ◄────── Backward LSTM│
        │                     │                       │
        │                     │  2 layers + char CNN  │
        ▼                     └──────────┬────────────┘
                                         │
  Can't handle                           ▼
  polysemy!              ┌───────────────────────────┐
                         │  EXTRACT ALL LAYER OUTPUTS │
                         │                             │
                         │  Layer 0: [char CNN embed]  │──► syntax info
                         │  Layer 1: [biLSTM hidden₁]  │──► grammar  
                         │  Layer 2: [biLSTM hidden₂]  │──► meaning  
                         └──────────────┬──────────────┘
                                        │
                                        ▼
                         ┌──────────────────────────────┐
                         │  TASK-SPECIFIC WEIGHTED MIX   │
                         │                                │     NEW SOTA on
                         │  ELMo = γ·(s₀h₀ + s₁h₁ + s₂h₂)│────► ALL 6 TASKS!
                         │                                │
                         │  Each task learns its own s,γ   │     SQuAD: +4.7 F1
                         └──────────────┬─────────────────┘     SRL:   +3.2 F1
                                        │                       NER:   +2.1 F1
                                        ▼                       Coref: +3.2 F1
                         ┌──────────────────────────────┐
                         │  PLUG INTO ANY EXISTING MODEL │
                         │                                │     + Massive sample
                         │  [original_embed ; ELMo] ──►RNN│     efficiency gains
                         │  Freeze biLM, train task model  │     (10x less data!)
                         └────────────────────────────────┘
```

---

## 🛠️ 11. IMPLEMENTATION SKETCH

### Pseudocode (Core Algorithm):
```python
# PRETRAINING (done once)
biLM = train_bidirectional_LM(corpus="1B_words", 
                               layers=2, hidden=4096, proj=512,
                               input="char_CNN", epochs=10)

# USING ELMo (for any downstream task)
def get_elmo(sentence, task_weights):
    # Step 1: Get all layer representations
    char_embed = biLM.char_cnn(sentence)          # Layer 0: h_0
    lstm1_fwd, lstm1_bwd = biLM.lstm1(char_embed)  # Layer 1
    lstm2_fwd, lstm2_bwd = biLM.lstm2(lstm1_out)    # Layer 2
    
    h0 = char_embed
    h1 = concat(lstm1_fwd, lstm1_bwd)
    h2 = concat(lstm2_fwd, lstm2_bwd)
    
    # Step 2: Task-specific weighted combination
    s = softmax(task_weights.s)  # learned per task
    gamma = task_weights.gamma    # learned per task
    
    elmo = gamma * (s[0]*h0 + s[1]*h1 + s[2]*h2)
    return elmo

# Step 3: Integrate with task model
def task_forward(sentence):
    x = word_embedding(sentence)        # GloVe etc.
    elmo = get_elmo(sentence, task_weights)
    x_enhanced = concat(x, elmo)         # Drop-in!
    output = task_model(x_enhanced)      # Existing arch
    return output
```

### Frameworks/Libraries Needed:
- **AllenNLP** (official implementation at `allennlp.org/elmo`)
- **PyTorch** (underlying framework)
- **h5py** (for caching ELMo representations)
- Pre-trained biLM weights available publicly

### Estimated Compute Cost:
- **biLM pretraining:** ~2-3 weeks on 3 GPUs (2018 hardware) on 1B Word Benchmark
- **Using pre-trained ELMo:** Moderate overhead — adds ~0.5-2s per batch for inference through the biLM
- **Task fine-tuning with ELMo:** Similar to baseline training time, but converges **much faster** (10 epochs vs. 486 for SRL)
- **Modern note:** You can download pre-trained ELMo weights and use them immediately — no pretraining needed
