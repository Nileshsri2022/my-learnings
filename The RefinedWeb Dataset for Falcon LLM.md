

# The RefinedWeb Dataset for Falcon LLM — Full Breakdown

---

## 🎯 1. THE ONE-LINER

**If you clean up internet text really well (remove junk and duplicates), you can train an AI that's just as smart as one trained on carefully hand-picked books and articles — and you'll have way more data to work with.**

---

## 🤔 2. THE PROBLEM (Why does this paper exist?)

- **The problem:** Large language models (LLMs) need *trillions* of tokens to train well. The conventional wisdom was that you need a *mix* of web data AND carefully curated "high-quality" sources (books, Wikipedia, scientific papers, Reddit). But those curated sources are **limited in size, expensive to collect, and legally risky**.
- **Why should anyone care?** Imagine you're cooking for 10,000 people. Everyone says you need rare, expensive ingredients from specialty shops (curated data). But those shops can only supply food for 1,000 people. This paper says: **if you're really careful about washing and sorting regular supermarket vegetables (web data), the meal is just as good — and you can feed everyone.**
- **Limitations of previous approaches:**
  - **Curated datasets don't scale:** The Pile is only ~340B tokens; optimally training a 175B model needs ~3,500B tokens
  - **Curation is labor-intensive:** Each source (books, code, papers) requires specialized processing
  - **Existing web datasets (C4, OSCAR) were considered inferior** — they used minimal filtering and deduplication
  - **Legal issues** with copyrighted books, papers, etc.
  - We may **"run out" of curated high-quality data** as models get bigger

---

## 💡 3. THE KEY IDEA (The "Aha!" moment)

**Core insight:** The problem was never that web data is inherently low quality — it's that **previous pipelines didn't clean it well enough**. With aggressive, multi-stage filtering AND deduplication (both fuzzy and exact), web data alone produces models that **match or beat** models trained on curated corpora.

**Everyday analogy:** Think of panning for gold. Previous miners (other datasets) scooped up river sand, did a quick rinse, and said "this river doesn't have much gold." The RefinedWeb team brought **industrial-grade sifting equipment** — multiple filters of different mesh sizes, plus magnets to catch even tiny flecks — and found **5 trillion gold flakes** (tokens) from the same river (CommonCrawl).

**The three-pillar approach:**
```
┌─────────────────────────────────────────────────┐
│           MACRODATA REFINEMENT (MDR)            │
├─────────────┬──────────────┬────────────────────┤
│  SCALE      │  STRICT      │  NEUTRAL           │
│  FIRST      │  DEDUP       │  FILTERING          │
│             │              │                     │
│ Web-only,   │ Both fuzzy   │ No ML-based         │
│ no curation │ AND exact,   │ quality filters     │
│ needed      │ ~50% removed │ (avoids bias)       │
└─────────────┴──────────────┴────────────────────┘
```

---

## 🏗️ 4. HOW IT WORKS (The Method - Layer by Layer)

The MDR pipeline has **three phases** with **seven steps**:

### Phase 1: DOCUMENT PREPARATION

**Step 1: URL Filtering**
- **WHAT:** Block bad URLs before downloading anything — adult sites, phishing, gambling, file hosting. Uses a 4.6M domain blocklist + a custom URL word-scoring system.
- **WHY:** Cheapest filter — saves compute by rejecting junk before heavy processing. Also blocks known HQ sources (Wikipedia, arXiv) so RefinedWeb is purely web-based.
- **HOW it connects:** Only ~2.2% of documents removed here, but prevents processing known-bad content.

**Step 2: Text Extraction**
- **WHAT:** Extracts main page content from raw HTML (WARC files) using `trafilatura`, stripping menus, headers, footers, ads.
- **WHY:** CommonCrawl's pre-processed WET files include navigation junk. `trafilatura` was benchmarked as the best open-source extractor.
- **HOW it connects:** Produces clean text documents from raw HTML.

**Step 3: Language Identification**
- **WHAT:** Uses fastText classifier (from CCNet) to identify language; keeps only English documents scoring ≥0.65.
- **WHY:** Non-English or garbled pages would be noise for an English model. **This is the biggest single filter — removes ~50% of documents.**
- **HOW it connects:** Output = **RW-Raw** (~48% of original CommonCrawl documents remain).

### Phase 2: FILTERING

**Step 4: Repetition Removal + Document-wise Filtering**
- **WHAT:** Removes documents with excessive repeated lines/paragraphs/n-grams (from Gopher/MassiveWeb heuristics). Also removes documents that are outliers in length, symbol-to-word ratio, etc.
- **WHY:** Catches machine-generated spam, keyword-stuffed SEO pages, broken crawls.
- **HOW it connects:** Removes ~24% of remaining documents.

**Step 5: Line-wise Corrections**
- **WHAT:** Scans each line and removes: social media counters ("3 likes"), navigation buttons ("Sign in"), call-to-actions ("Read more..."), all-caps lines, single-word lines. If >5% of document words are removed, the whole document is discarded.
- **WHY:** Even with good HTML extraction, artifacts leak through. **This is a novel contribution.**
- **HOW it connects:** Output = **RW-Filtered** (~23% of original CC documents remain).

### Phase 3: DEDUPLICATION

**Step 6: Fuzzy Deduplication (MinHash)**
- **WHAT:** Computes a "fingerprint" (sketch) of each document using MinHash with **9,000 hashes** over 5-grams, split into 20 buckets of 450. Documents with high similarity are clustered and all but one removed.
- **WHY:** Catches templated content (cookie notices, legal disclaimers, SEO spam) that differ only slightly. **Far more aggressive than prior work** (The Pile used only 10 hashes).
- **HOW it connects:** Reduces data by ~40%, making exact dedup feasible.

**Step 7: Exact Substring Deduplication**
- **WHAT:** Builds a suffix array over the entire corpus; finds any exact string match ≥50 tokens long and removes it (cuts the span out).
- **WHY:** Catches verbatim repeated paragraphs that MinHash might miss (e.g., a disclaimer that appears in thousands of otherwise-different documents).
- **HOW it connects:** Output = **RefinedWeb** (~11.67% of original CC; **~5 trillion tokens**).

**Bonus: URL Deduplication across dumps**
- CommonCrawl revisits URLs across its ~100 dumps. They track which URLs they've already kept and skip duplicates in later dumps.

---

## 📊 5. THE PROOF (Results & Experiments)

### Benchmarks
- **18 zero-shot tasks** across 4 aggregates: HellaSwag, LAMBADA, Winogrande, PIQA, ARC, OpenBookQA, BoolQ, COPA, CB, RTE, ReCoRD, ANLI, LogiQA, HeadQA, MathQA, PROST, PubMedQA, SciQ
- Evaluated using **EleutherAI LM Evaluation Harness**

### Key Results

| Comparison | Result |
|---|---|
| **RW vs. The Pile (1B, small-scale)** | **56.2% vs 53.4%** (+2.8% absolute) on small-agg |
| **RW vs. The Pile (3B, small-scale)** | **59.8% vs 57.9%** (+1.9% absolute) |
| **Falcon-RW-7B vs GPT-3 (6.7B)** | **Matches GPT-3** on main-agg at similar compute |
| **Falcon-RW vs all Pile-trained models** | **Significantly outperforms** GPT-Neo, Pythia, Cerebras-GPT, OPT |
| **Dedup boost on The Pile** | +1.1% from dedup alone; +1.8% from filter+dedup |
| **Dedup boost on OSCAR-22.01** | +2.9% from dedup alone (was undeduplicated!) |

### Most impressive result in plain English
> **A model trained on nothing but cleaned-up internet pages beat models trained on carefully hand-selected collections of books, scientific papers, and curated web content — even though Wikipedia, arXiv, StackOverflow, and Reddit were deliberately EXCLUDED from the web data.**

### Limitations admitted
- **Toxicity:** Similar to The Pile — not worse, but not better
- **English only** (though pipeline can extend to other languages)
- **Multiple epochs:** Uncertain whether large models can safely reuse deduplicated data
- No comparison with LLaMA (2.5x more compute, unfair comparison)
- **No ML-based quality filtering** — deliberate choice to avoid bias, but may miss some quality signal

---

## 🧩 6. KEY TERMS GLOSSARY

**CommonCrawl** → A free, public archive of billions of web pages scraped over 12+ years

**Token** → A chunk of text (roughly ¾ of a word on average); LLMs process text as tokens

**Curated corpus** → A dataset hand-assembled from specific high-quality sources (books, papers, Wikipedia)

**The Pile** → A popular 340B-token curated dataset mixing web data with books, code, papers, etc.

**Zero-shot** → Testing a model on tasks it was never explicitly trained for

**MinHash** → A technique to quickly estimate how similar two documents are by comparing compact "fingerprints"

**Fuzzy deduplication** → Removing documents that are approximately (not exactly) the same

**Exact substring deduplication** → Removing text passages that are character-for-character identical across documents

**Suffix array** → A data structure that sorts all possible endings of a text, enabling fast search for repeated strings

**Jaccard similarity** → A measure of overlap between two sets: |intersection| / |union|

**trafilatura** → An open-source library that extracts main content from web pages (ignoring menus, ads, etc.)

**fastText** → A fast, lightweight text classifier (used here to identify language)

**WARC files** → Raw web archive files containing full HTML responses

**WET files** → Pre-processed CommonCrawl files with only plain text (lower quality extraction)

**Scaling laws** → Rules predicting how model performance improves with more compute/data/parameters (Chinchilla)

**ALiBi** → Attention with Linear Biases — a positional encoding method for transformers

**FlashAttention** → A memory-efficient attention algorithm that speeds up training

**PF-days** → PetaFLOP-days — a measure of total compute used

**NSFW** → Not Safe For Work — adult/inappropriate content

**Blocklist** → A list of domains/words that are automatically rejected

**MDR (MacroData Refinement)** → The paper's data processing pipeline name

---

## 🔗 7. HOW IT CONNECTS

### Intellectual Family Tree
```
Scaling Laws (Kaplan 2020, Hoffmann/Chinchilla 2022)
    │ "You need MORE data, not just bigger models"
    │
    ├── The Pile (Gao et al., 2020)
    │     "Mix web + curated sources"
    │
    ├── MassiveWeb / Gopher (Rae et al., 2021)
    │     "Quality heuristics for web filtering"
    │
    ├── Deduplication matters (Lee et al., 2022)
    │     "Removing duplicates improves LLMs"
    │
    ├── CCNet (Wenzek et al., 2020)
    │     "fastText language ID for web data"
    │
    └──► RefinedWeb (this paper)
          "Combine ALL of the above + do it at 5T scale"
              │
              └──► Falcon-40B (Almazrouei et al., 2023)
                    State-of-the-art open LLM
```

### Comparison with related papers:
- **vs. Lee et al. (2022) "Deduplicating Training Data":** RefinedWeb builds on their exact+fuzzy dedup methods but applies them at **15x larger scale** and with much more aggressive settings (9000 hashes vs 10)
- **vs. Pythia (Biderman et al., 2023):** Pythia found dedup had limited impact on The Pile; RefinedWeb argues this is because curated data has fewer harmful duplicates than raw web data

### Who would use this?
- **LLM developers** who need trillion-token datasets
- **Researchers** studying data quality for language models
- **Open-source AI community** (600B tokens released publicly)

### Future work enabled
- Multilingual RefinedWeb variants
- Studying deduplication in the data-constrained regime (multiple epochs)
- Combining RefinedWeb with targeted curated sources for even better models
- Better understanding of what makes training data "high quality"

---

## ⚖️ 8. CRITICAL ANALYSIS

### Hidden assumptions
- **Zero-shot benchmarks are the right measure of quality.** The paper acknowledges perplexity can disagree with task performance, but zero-shot accuracy also has limitations (task selection bias, prompt sensitivity)
- **"Web data only" is a clean separation.** CommonCrawl itself contains Wikipedia mirrors, book excerpts, academic papers — even after URL filtering, similar content exists across the web
- **Neutral filtering = no ML-based filtering.** This avoids bias but may also avoid catching genuinely low-quality content that heuristics miss

### Weaknesses the authors DON'T mention
- **No evaluation on generation quality, factuality, or downstream fine-tuning** — only zero-shot classification/completion tasks
- **No human evaluation** of output quality
- **Deduplication was done in 100 shards, not globally** — duplicate clusters spanning shards may survive
- **The pipeline software is NOT released** — only the data and models, limiting reproducibility
- **No analysis of diversity** — are certain domains over-represented? (Blogspot = 4.86%)
- **Domain distribution skews toward English-speaking developed countries** — geographic/cultural bias not discussed

### Is the evaluation fair?
- **Mostly fair:** They control for confounders by training identical architectures on different datasets, and use a widely-adopted evaluation framework
- **Some concern:** They compare against GPT-3 paper numbers (†) which used a different evaluation setup; the API results (*) are more apples-to-apples but show GPT-3 performing worse than its paper claims
- **LLaMA excluded** because of compute difference — reasonable but notable

### Would this work at scale in the real world?
- **Yes — and it already has.** Falcon-40B was trained on RefinedWeb and achieved state-of-the-art results at release
- **Compute costs for the pipeline are substantial** (100-250 large CPU instances, 2TB RAM machines for dedup)
- **CommonCrawl is continuously growing**, so the pipeline can keep producing fresh data

---

## 📝 9. MEMORY ANCHORS

### Memorable metaphor
> **RefinedWeb is like a water purification plant:** raw river water (CommonCrawl) passes through multiple increasingly fine filters (URL blocking → text extraction → language ID → quality heuristics → line corrections) and then through two types of decontamination (fuzzy dedup + exact dedup). The output is drinking water so clean it rivals bottled spring water (curated datasets) — and there's an ocean of it (5 trillion tokens).**

### 3 bullets that capture 80% of the paper
- 🧹 **Web data isn't bad data — it's badly processed data.** Aggressive filtering + deduplication of CommonCrawl yields 5 trillion tokens that train better LLMs than hand-curated datasets like The Pile.
- 🔄 **Deduplication is the single most consistent performance booster** — it helps across ALL datasets tested, while filtering heuristics sometimes help and sometimes hurt.
- 📊 **Models trained on RefinedWeb alone match GPT-3** in zero-shot benchmarks, despite deliberately excluding Wikipedia, arXiv, Reddit, StackOverflow, and books.

### Comprehension test question
> *If deduplication consistently improves model performance while filtering improvements are "not systematic across datasets," why did the authors still include filtering in their pipeline — and what does this tell us about the relationship between data cleaning and model quality?*

---

## 🗺️ 10. VISUAL MENTAL MAP

```
PROBLEM                        METHOD                              RESULT
───────                        ──────                              ──────

LLMs need                ┌─── DOCUMENT PREP ───┐
trillions of       ───►  │ URL filter (2.2%)    │
tokens                   │ trafilatura extract  │
                         │ Language ID (50%)    │
  ↓                      └────────┬─────────────┘
                                  ▼                        
Curated data             ┌─── FILTERING ────────┐          ┌──────────────────┐
is limited         ───►  │ Repetition removal   │    ───►  │ 5 TRILLION TOKENS│
(~300-800B)              │ Doc-wise heuristics  │          │ (web-only)       │
                         │ Line-wise corrections│          └────────┬─────────┘
  ↓                      │ (~24% removed)       │                   │
                         └────────┬─────────────┘                   ▼
"Web data is                      ▼                        ┌──────────────────┐
low quality"      ───►   ┌─── DEDUPLICATION ────┐          │ Falcon-RW models │
(conventional            │ MinHash fuzzy (~40%) │    ───►  │ 1.3B and 7.5B    │
wisdom)                  │ Exact substr (~12%)  │          │                  │
                         │ URL dedup across     │          │ MATCH GPT-3 ✓    │
                         │ dumps                │          │ BEAT The Pile ✓  │
                         └──────────────────────┘          │ BEAT OSCAR ✓     │
                                                           │ BEAT C4 ✓        │
CommonCrawl ──────────────────────────────────────────────► └──────────────────┘
(100%)       only ~11.7% survives the pipeline              600B released 🌍

KEY FINDING: "Curation is not a silver bullet" — 
clean web data ≥ curated data for zero-shot generalization
```

---

## 🛠️ 11. IMPLEMENTATION SKETCH

### Pseudocode (Core Pipeline)
```python
def macrodata_refinement(commoncrawl_dumps):
    documents = []
    seen_urls = set()
    
    for dump in commoncrawl_dumps:
        for warc_record in read_warc(dump):
            url = warc_record.url
            
            # Step 1: URL filtering
            if url in BLOCKLIST or url_score(url) > THRESHOLD:
                continue
            if url in seen_urls:
                continue
            
            # Step 2: Text extraction
            text = trafilatura.extract(warc_record.html)
            if text is None:
                continue
            text = normalize_whitespace(remove_urls(text))
            
            # Step 3: Language ID
            lang, score = fasttext_langid(text)
            if lang != "en" or score < 0.65:
                continue
            
            # Step 4: Document-level filtering
            if has_excessive_repetition(text):  # Gopher heuristics
                continue
            if not passes_quality_heuristics(text):  # length, symbols, etc.
                continue
            
            # Step 5: Line-wise corrections
            text, pct_removed = filter_lines(text)  # remove nav, counters
            if pct_removed > 0.05:
                continue
            
            documents.append((url, text))
            seen_urls.add(url)
    
    # Step 6: Fuzzy dedup (MinHash, 9000 hashes, 20 buckets x 450)
    documents = minhash_dedup(documents, n_hashes=9000, 
                              bands=20, rows=450, ngram=5)
    
    # Step 7: Exact substring dedup (suffix array, ≥50 tokens)
    documents = exact_substr_dedup(documents, min_length=50)
    
    return documents  # ~5 trillion tokens
```

### Frameworks/Libraries Needed
- **`trafilatura`** — HTML text extraction
- **`fasttext`** — Language identification (CCNet model)
- **`warcio`** — Reading WARC files from CommonCrawl
- **`datasketch`** or custom — MinHash LSH implementation
- **`deduplicate-text-datasets`** (Google Research) — Suffix array exact dedup
- **AWS infrastructure** — CPU clusters + high-memory instances

### Estimated Compute Cost
- **CPU cluster:** 100-250 × AWS c5.18xlarge instances (72 vCPUs, 144GB each)
- **High-memory instance for exact dedup:** AWS x2iedn (up to 2 TiB RAM)
- **Scale:** Processing ALL of CommonCrawl (petabytes of raw data)
- **Rough estimate:** Likely **$50,000-$200,000+ in cloud compute** for the full pipeline
- **Model training:** 1.3B model on 350GT ≈ 32 PF-days; 7.5B model ≈ 182 PF-days (likely on a cluster of A100 GPUs)
