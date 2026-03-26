

# BERTopic: Neural Topic Modeling with a Class-Based TF-IDF Procedure

---

## 🎯 1. THE ONE-LINER

**BERTopic is a tool that automatically figures out what topics a big pile of documents are about by grouping similar documents together using AI and then finding the best words to describe each group.**

---

## 🤔 2. THE PROBLEM (Why does this paper exist?)

- **The real-world problem:** When you have thousands or millions of documents (news articles, tweets, research papers), you need an automated way to discover what themes/topics they cover.
- **Why should anyone care?** Imagine you're a librarian who just received 50,000 new books with no labels. You need to sort them into sections (sports, science, cooking, etc.) AND create a sign for each section describing what's in it. That's topic modeling — and it's crucial for anyone analyzing large text collections.
- **Limitations of previous approaches:**
  - **Classical models (LDA, NMF):** Treat documents as "bags of words" — they see "bank" the same whether it means a river bank or a money bank. **They ignore word meaning and context.**
  - **Newer clustering approaches (Top2Vec, Sia et al.):** Use embeddings (smart word representations) BUT pick topic words by finding words **closest to the center of each group**. Problem: **clusters aren't always neat spheres**, so the center might not represent the group well. Like picking the geographic center of a country to describe it — that point might be in a desert!

---

## 💡 3. THE KEY IDEA (The "Aha!" moment)

**Core insight:** Don't describe topics by what's near the cluster center. Instead, **treat each cluster as one giant mega-document and use a modified TF-IDF to find which words are uniquely important to each topic.**

### Everyday analogy:
Imagine you have 5 different book clubs. Each club has been reading and discussing different genres. To figure out what makes each club unique, you don't look at the "average" thing they discuss (centroid approach). Instead, you **collect ALL their conversations, mash them into one transcript per club**, and then ask: "What words does THIS club use a lot that OTHER clubs rarely use?" That's **class-based TF-IDF (c-TF-IDF)**.

### The 3-step pipeline (like a factory assembly line):
```
Documents → [BERT Embeddings] → [UMAP + HDBSCAN Clustering] → [c-TF-IDF] → Topics
              "Understand"           "Group similar ones"       "Label each group"
```

---

## 🏗️ 4. HOW IT WORKS (The Method - Layer by Layer)

### Step 1: **Embed Documents** (Understand what each document means)
- **WHAT:** Convert each document into a numerical vector (a list of numbers) using a pre-trained language model (Sentence-BERT).
- **WHY:** You need to represent documents as numbers so a computer can compare them. Documents about similar topics will have similar numbers.
- **HOW it connects:** These vectors become the "coordinates" for each document in a high-dimensional space, ready for grouping.

### Step 2: **Reduce Dimensions with UMAP** (Simplify the space)
- **WHAT:** Shrink the embedding vectors from ~768 dimensions down to a much lower number (e.g., 5-10 dimensions).
- **WHY:** In very high dimensions, **all points look equally far apart** (curse of dimensionality), making clustering nearly impossible. UMAP preserves the neighborhood structure while compressing.
- **HOW it connects:** The reduced-dimension vectors are now clusterable.

### Step 3: **Cluster with HDBSCAN** (Group similar documents)
- **WHAT:** Group the reduced embeddings into clusters of semantically similar documents. Each cluster = one topic.
- **WHY:** HDBSCAN finds clusters of **varying densities** (not just neat round blobs) AND can label some documents as **outliers/noise** (not belonging to any topic).
- **HOW it connects:** Now you have groups, but you need words to describe what each group is about.

### Step 4: **Generate Topic Representations with c-TF-IDF** (Name the topics)
- **WHAT:** Concatenate all documents in a cluster into one mega-document. Apply a **class-based TF-IDF** formula:
  ```
  W(t,c) = tf(t,c) · log(1 + A / tf(t))
  ```
  where `tf(t,c)` = frequency of word t in class c, `A` = average words per class, `tf(t)` = total frequency of t across all classes.
- **WHY:** This finds words that are **frequent within a topic** but **rare across other topics** — the signature words. Traditional TF-IDF works at document level; c-TF-IDF works at **topic/cluster level**.
- **HOW it connects:** The top-scoring words per cluster become the topic representation.

### Step 5 (Optional): **Reduce Number of Topics**
- **WHAT:** Iteratively merge the least common topic with its most similar one (based on c-TF-IDF similarity).
- **WHY:** The user may want fewer, broader topics.

---

## 📊 5. THE PROOF (Results & Experiments)

### Benchmarks/Datasets:
| Dataset | Size | Type |
|---------|------|------|
| 20 NewsGroups | 16,309 articles | 20 categories, well-preprocessed |
| BBC News | 2,225 articles | News from 2004-2005 |
| Trump's Tweets | 44,253 tweets | Short text, 2009-2021 |
| UN General Debates | Transcriptions | 2006-2015, for dynamic modeling |

### Key Results (Table 1 — Topic Coherence / Topic Diversity):

| Model | 20NG (TC) | BBC (TC) | Trump (TC) |
|-------|-----------|----------|------------|
| LDA | .058 | .014 | -.011 |
| NMF | .089 | .012 | .009 |
| Top2Vec-Doc2Vec | **.192** | **.171** | -.169 |
| CTM | .096 | .094 | .009 |
| **BERTopic-MPNET** | **.166** | **.167** | **.066** |

- **BERTopic has the highest topic coherence on Trump's tweets** (.066 vs. next best .009)
- **Competitive with or near the best on 20 NewsGroups and BBC News**
- **Most impressive result in plain English:** BERTopic is the **most consistently good model across all datasets** — it never catastrophically fails, unlike Top2Vec-Doc2Vec which scores -.169 on Trump or Top2Vec-MPNET which scores -.213 on Trump.
- **Dynamic topic modeling:** BERTopic outperforms LDA Sequence on Trump (.079 vs .009 TC) and UN (.231 vs .173 TC)

### Limitations admitted:
- **Single-topic assumption:** Each document is assigned to ONE topic (reality: documents can cover multiple topics)
- **Topic words can be redundant:** Since c-TF-IDF uses bag-of-words, topic words may be synonyms of each other
- **CTM consistently has higher topic diversity** than BERTopic
- **NPMI may not correlate with human judgment for neural topic models** (they honestly cite Hoyle et al., 2021)

---

## 🧩 6. KEY TERMS GLOSSARY

- **Topic Model** → An algorithm that automatically discovers hidden themes in a collection of documents
- **LDA (Latent Dirichlet Allocation)** → A classical topic model that assumes each document is a mixture of topics, using bag-of-words
- **NMF (Non-Negative Matrix Factorization)** → A matrix decomposition method used for topic modeling
- **TF-IDF** → A score measuring how important a word is to a document; high if frequent in the doc but rare elsewhere
- **c-TF-IDF (class-based TF-IDF)** → BERTopic's modification of TF-IDF that measures word importance to a *topic cluster* instead of a single document
- **BERT** → A transformer-based language model that understands word context
- **Sentence-BERT (SBERT)** → A version of BERT optimized for generating sentence-level embeddings
- **Embedding** → A numerical vector representation of text that captures its meaning
- **UMAP** → A dimensionality reduction technique that preserves both local and global structure of data
- **HDBSCAN** → A density-based clustering algorithm that finds clusters of varying shapes/densities and identifies outliers
- **Curse of Dimensionality** → The phenomenon where distances become meaningless in very high-dimensional spaces
- **Topic Coherence (NPMI)** → A metric measuring how semantically related the top words in a topic are; higher = better
- **Topic Diversity** → The percentage of unique words across all topics; higher = less redundancy between topics
- **Top2Vec** → A topic model that uses Doc2Vec embeddings and finds topic words closest to cluster centroids
- **CTM (Contextualized Topic Model)** → A neural topic model using pre-trained transformer embeddings within a VAE framework
- **Dynamic Topic Modeling** → Tracking how topics change over time
- **Doc2Vec** → An older embedding technique that learns document and word vectors jointly
- **Bag-of-Words** → Representing a document as a collection of word counts, ignoring word order and context

---

## 🔗 7. HOW IT CONNECTS

### Intellectual Family Tree:
```
LDA (Blei 2003)
  ├── Dynamic LDA (Blei & Lafferty 2006)
  └── Word embedding + LDA hybrids (Liu 2015, Nguyen 2015)
  
Word/Doc Embeddings
  ├── Doc2Vec (Le & Mikolov 2014)
  ├── BERT (Devlin 2018)
  │     └── Sentence-BERT (Reimers 2019)
  ├── Clustering embeddings for topics (Sia et al. 2020) ←──┐
  ├── Top2Vec (Angelov 2020) ←─────────────────────────────┤
  └── CTM (Bianchi et al. 2020) ←──────────────────────────┤
                                                             │
                                           BERTopic (THIS PAPER)
                                           Key addition: c-TF-IDF
```

### Who would use this:
- **Journalists/analysts** exploring large document collections
- **Social scientists** analyzing public discourse (tweets, speeches)
- **Companies** doing customer feedback analysis or content tagging
- **Researchers** doing literature reviews or corpus analysis

### Future work it enables:
- Plug in better language models as they emerge → automatic improvement
- Dynamic topic modeling for tracking narrative shifts over time
- Multi-modal topic modeling (combining text with other data types)
- Topic modeling by metadata (by author, by journal, etc.)

---

## ⚖️ 8. CRITICAL ANALYSIS

### Hidden assumptions:
- **Documents about the same topic are semantically similar** in embedding space — this may not hold for nuanced or cross-cutting topics
- **One document = one topic** — a strong simplification
- **UMAP parameters** significantly affect results but are treated as hyperparameters without deep exploration
- The quality entirely depends on the **pre-trained embedding model** — garbage embeddings = garbage topics

### Weaknesses the authors DON'T mention:
- **No principled way to choose the number of topics** — HDBSCAN decides automatically, but the "right" number is subjective
- **Sensitivity to HDBSCAN's min_cluster_size** — this hyperparameter heavily influences results but isn't discussed
- **Outlier documents** (not assigned to any topic) could be a large fraction — this is glossed over
- **Reproducibility concerns:** UMAP and HDBSCAN are stochastic — results vary across runs (they average 3 runs, but variance isn't reported)
- **No human evaluation** — they rely on NPMI which they themselves note may not correlate with human judgment for neural models

### Is the evaluation fair?
- **Partially.** They use standard metrics (NPMI, TD) and compare against reasonable baselines
- **But:** Only 3 datasets, no very large-scale corpus (millions of docs), no human evaluation study
- They fix HDBSCAN/UMAP params between BERTopic and Top2Vec (good), but don't report what those params are
- Averaging over 10-50 topics in steps of 10 is coarse

### Would this work in the real world at scale?
- **Yes, with caveats.** It's already very widely adopted (the GitHub repo has 5000+ stars)
- The modular design is a genuine strength — you can swap components
- **Scalability bottleneck:** Embedding large corpora with SBERT can be slow without GPUs
- For truly massive datasets (millions+), UMAP + HDBSCAN can be slow/memory-intensive

---

## 📝 9. MEMORY ANCHORS

### Memorable metaphor:
> **BERTopic is like a smart librarian who first *reads* every book (embeddings), then *groups* them by similarity on shelves (clustering), and finally writes a *unique label* for each shelf by finding words that appear a lot on THAT shelf but rarely on others (c-TF-IDF).**

### 3 bullet points that capture 80% of the paper:
- **Embed → Reduce → Cluster → Label:** Use SBERT for embeddings, UMAP for dimension reduction, HDBSCAN for clustering, c-TF-IDF for topic words
- **c-TF-IDF is the key innovation:** Instead of finding words near a cluster centroid, concatenate all docs in a cluster and find words uniquely important to that cluster vs. others
- **Modular design = flexibility:** Each step is independent, so you can swap embedding models, adjust topic labels without re-clustering, and extend to dynamic/temporal topics easily

### One question to test understanding:
> **Why does BERTopic use c-TF-IDF instead of finding words closest to the cluster centroid, and what problem does this solve?**

*Answer: Because clusters aren't always spherical — a centroid may not represent the cluster well. c-TF-IDF avoids this by directly measuring which words are distinctively important to a cluster's documents compared to all other clusters, regardless of cluster shape.*

---

## 🗺️ 10. VISUAL MENTAL MAP

```
                        BERTopic Pipeline
                        ================

 RAW DOCUMENTS          STEP 1: EMBED             STEP 2: REDUCE
 ┌──────────┐      ┌──────────────────┐      ┌──────────────────┐
 │ "Tesla    │      │                  │      │                  │
 │  launches │─────▶│  Sentence-BERT   │─────▶│      UMAP        │
 │  new car" │      │  (768-dim vector) │      │  (768→5-10 dim)  │
 │           │      │                  │      │                  │
 │ "Biden    │      │  [0.23, -0.11,   │      │  [1.2, -0.5,     │
 │  wins     │      │   0.87, ...]     │      │   0.3, ...]      │
 │  election"│      │                  │      │                  │
 └──────────┘      └──────────────────┘      └────────┬─────────┘
                                                       │
                                                       ▼
                   STEP 4: LABEL              STEP 3: CLUSTER
              ┌──────────────────┐      ┌──────────────────────┐
              │                  │      │                      │
              │   c-TF-IDF       │◀─────│     HDBSCAN          │
              │                  │      │                      │
              │ Cluster A:       │      │  ● ●●  Cluster A     │
              │  "car","tesla",  │      │  ●●                  │
              │  "electric"      │      │        ▲▲            │
              │                  │      │       ▲▲▲ Cluster B  │
              │ Cluster B:       │      │                      │
              │  "election",     │      │  ○  ← outlier/noise  │
              │  "vote","biden"  │      │                      │
              └──────┬───────────┘      └──────────────────────┘
                     │
                     ▼
              ┌──────────────────┐
              │   FINAL TOPICS   │
              │                  │
              │ Topic 1: Cars    │
              │ Topic 2: Politics│
              │ Topic -1: Noise  │
              └──────────────────┘

    Optional extensions:
    ┌─────────────────────────────────────────────┐
    │  DYNAMIC TOPICS: Split by time, re-apply    │
    │  c-TF-IDF per timestep using global IDF     │
    │                                             │
    │  2020: "tesla", "self-driving"              │
    │  1990: "ford", "gasoline"                   │
    │         (same global "cars" topic)           │
    └─────────────────────────────────────────────┘
```

---

## 🛠️ 11. IMPLEMENTATION SKETCH

### Pseudocode (~15 lines):
```python
# Core BERTopic Algorithm
def bertopic(documents, nr_topics=None):
    # Step 1: Embed documents
    embeddings = SentenceBERT("all-mpnet-base-v2").encode(documents)
    
    # Step 2: Reduce dimensionality
    reduced_embeddings = UMAP(n_components=5).fit_transform(embeddings)
    
    # Step 3: Cluster
    clusters = HDBSCAN(min_cluster_size=15).fit_predict(reduced_embeddings)
    
    # Step 4: c-TF-IDF for topic representations
    topics = {}
    for cluster_id in unique(clusters):
        docs_in_cluster = documents[clusters == cluster_id]
        mega_doc = concatenate(docs_in_cluster)
        tf = term_frequency(mega_doc)                    # tf(t,c)
        avg_words = mean([len(all_cluster_docs)])        # A
        total_freq = term_frequency_across_all_clusters  # tf(t)
        c_tfidf = tf * log(1 + avg_words / total_freq)
        topics[cluster_id] = top_n_words(c_tfidf, n=10)
    
    # Optional: reduce number of topics by merging
    if nr_topics:
        topics = iteratively_merge_smallest(topics, nr_topics)
    
    return clusters, topics
```

### Frameworks/Libraries needed:
- `sentence-transformers` (for SBERT embeddings)
- `umap-learn` (for UMAP dimensionality reduction)
- `hdbscan` (for clustering)
- `scikit-learn` (for TF-IDF vectorization utilities)
- `numpy`, `scipy` (numerical operations)
- Or just: **`pip install bertopic`** (the author's package wraps all of this)

### Estimated compute cost:
- **Small dataset (~2K docs):** ~30 seconds on a laptop with GPU
- **Medium dataset (~16K docs):** ~2-5 minutes with GPU
- **Large dataset (~44K docs):** ~5-10 minutes with GPU (see Figure 1: ~60-100s wall time)
- **Without GPU:** Embedding step becomes the bottleneck — 10-50x slower
- **CTM (comparison):** ~1400s for 44K docs — BERTopic is **~15-20x faster**
