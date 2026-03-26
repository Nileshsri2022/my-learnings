

# Toolformer: Language Models Can Teach Themselves to Use Tools

---

## 🎯 1. THE ONE-LINER
**A language model taught itself when and how to use tools like a calculator, search engine, and translator by practicing on its own, without humans telling it what to do.**

---

## 🤔 2. THE PROBLEM (Why does this paper exist?)

- **The problem:** Large language models (like GPT-3) are great at writing and understanding language, but they're *embarrassingly bad* at basic tasks like math (What's 3,847 × 29?), looking up today's date, or finding current facts. Meanwhile, a $5 calculator or a simple Google search solves these instantly.

- **Why should anyone care?** Imagine hiring a brilliant essayist who can write about anything, but they can't use a phone, a calculator, or look anything up. They'd make up facts, get math wrong, and not know what day it is. That's what LLMs are like — **brilliant but helpless without tools**.

- **Limitations of previous approaches:**
  - **Too much human labor:** Prior methods (LaMDA, Internet-augmented dialogue) required **massive human annotations** to teach tool use
  - **Too task-specific:** Other methods (PAL, ReAct) only worked when you **explicitly told the model** which tool to use for a specific task via few-shot prompting
  - **No generality:** None let the model **autonomously decide** when/how/which tool to use across arbitrary tasks

---

## 💡 3. THE KEY IDEA (The "Aha!" moment)

**Core insight:** Let the language model **annotate its own training data** with API calls, then **keep only the ones that actually helped** it predict the next words better. Then fine-tune on this self-curated dataset.

**Everyday analogy:** 🍳 Imagine a chef learning to use kitchen gadgets:
1. The chef tries using a blender, thermometer, and timer at random points while cooking various recipes
2. After each dish, they taste it — **did the gadget actually make the food better?**
3. They only **remember and practice** the gadget uses that improved the dish
4. Eventually, the chef instinctively knows: "This recipe needs a thermometer here, that one needs a blender there"

**No cooking teacher needed** — the chef learned from their own feedback!

**The method in 3 sentences:**
1. Use few-shot prompting to make the LM **generate candidate API calls** sprinkled throughout a text dataset
2. Execute each API call and check: **did having this result reduce the model's prediction error?**
3. Keep the helpful ones, fine-tune the model on this augmented dataset

---

## 🏗️ 4. HOW IT WORKS (The Method - Layer by Layer)

### Step 1: 📝 Sample API Calls
- **WHAT:** For each text in the dataset, use in-context learning (a few examples in a prompt) to have the LM propose where API calls might go and what they should say
- **WHY:** The LM already has some intuition about where external info would help — leverage that
- **HOW:** For each position *i* in text, compute the probability the model assigns to starting an API call token `<API>`. Keep positions where this probability exceeds threshold τ_s. Then sample up to *m* candidate API calls per position.
- **CONNECTS TO:** These are *candidates* — most will be bad. Step 2 gets the answers.

### Step 2: 📞 Execute API Calls
- **WHAT:** Actually run each candidate API call (ask the QA system, run the calculator, query Wikipedia, etc.) to get a text response
- **WHY:** We need the actual tool output to assess whether it's useful
- **HOW:** Depends on the tool — could be calling a neural network (Atlas for QA), running a Python script (calculator), or querying a retrieval system (Wikipedia BM25 search)
- **CONNECTS TO:** Now we have (API call + response) pairs. Step 3 decides which are worth keeping.

### Step 3: 🔍 Filter API Calls
- **WHAT:** Compare the model's **loss on future tokens** in three scenarios:
  - L⁺: Loss *with* the API call and its result provided → "How well does the model predict what comes next if it has the tool's answer?"
  - L⁻: The *minimum* of loss without any API call OR with the API call but no result → "How well does it do without help, or with just the question but no answer?"
- **WHY:** **An API call is useful only if having BOTH the question AND answer helps more than having neither or just the question.** This ensures the tool's *response* actually provides value.
- **HOW:** Keep only calls where L⁻ − L⁺ ≥ τ_f (the improvement exceeds a threshold)
- **CONNECTS TO:** The surviving calls become training data.

### Step 4: 🔀 Merge & Fine-tune
- **WHAT:** Insert surviving API calls into the original text at their positions. Fine-tune the LM on this augmented dataset using standard language modeling loss.
- **WHY:** The model learns to *produce* API calls at the right moments, because those moments are where API calls helped *it* predict better. The original text is preserved, so **no language ability is lost**.
- **HOW:** Standard fine-tuning with batch size 128, learning rate 1e-5, up to 2k steps on 8× A100 GPUs.

### Step 5: 🔮 Inference
- **WHAT:** During text generation, when the model outputs the "→" token (indicating it expects a tool response), **pause decoding**, call the real API, insert the response, and continue generating.
- **WHY:** This is the payoff — the model now autonomously decides to call tools when needed.
- **TRICK:** They let `<API>` trigger not just when it's the #1 most likely token, but when it's in the **top k=10** tokens — to encourage tool use.

```
ASCII DIAGRAM OF THE PIPELINE:

  Raw Text Dataset (CCNet)
         │
         ▼
  ┌──────────────┐
  │ Step 1: LM    │──── Few-shot prompt ──→ Candidate API calls
  │ samples calls │                         at various positions
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │ Step 2: Run   │──── Calculator, QA, ──→ Get responses r₁, r₂...
  │ the API calls │     Search, etc.
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │ Step 3: Filter│──── L⁻ - L⁺ ≥ τ_f? ──→ Keep only helpful calls
  │ by usefulness │
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │ Step 4: Fine- │──── Augmented dataset ──→ Toolformer!
  │ tune the LM   │     C* = text + API calls
  └──────────────┘
```

---

## 📊 5. THE PROOF (Results & Experiments)

### Benchmarks tested:
| Category | Datasets |
|----------|----------|
| Fact completion | LAMA (SQuAD, Google-RE, T-REx) |
| Math reasoning | ASDiv, SVAMP, MAWPS |
| Question answering | WebQuestions, Natural Questions, TriviaQA |
| Multilingual QA | MLQA (6 languages) |
| Temporal reasoning | TEMPLAMA, DATESET |
| Language modeling | WikiText, CCNet subset |

### Key results (Toolformer = 6.7B params):

| Task | GPT-J (6.7B) | Toolformer (6.7B) | GPT-3 (175B) |
|------|-------------|-------------------|--------------|
| T-REx (LAMA) | 31.9 | **53.5** | 39.8 |
| ASDiv (math) | 7.5 | **40.4** | 14.0 |
| SVAMP (math) | 5.2 | **29.4** | 10.0 |
| MAWPS (math) | 9.9 | **44.0** | 19.8 |
| WebQS | 18.5 | **26.3** | **29.0** |
| DATESET (temporal) | 3.9 | **27.3** | 0.8 |

### Most impressive result in plain English:
**A 6.7B parameter model outperforms GPT-3 (175B) — a model 25× its size — on math and factual tasks**, simply by learning to use a calculator and QA system.

### Failure cases & limitations the authors admit:
- **No tool chaining:** Can't use output of one tool as input to another (e.g., get today's date → then ask QA using that date)
- **No interactive tool use:** Can't browse through search results or refine queries
- **Sample inefficient:** Processing 1M+ documents yields only ~1,000 useful calculator examples
- **Prompt sensitivity:** Whether the model calls an API depends heavily on exact wording
- **Falls short of GPT-3 on QA tasks** (WebQS, NQ, TriviaQA) — likely due to the simplistic BM25 search engine
- **Tool use only emerges at ~775M+ parameters** — smaller models can't learn to use tools effectively

---

## 🧩 6. KEY TERMS GLOSSARY

- **API call** → A request to an external tool (like asking a calculator to compute 5+3), formatted as text that can be inserted into a sentence
- **Self-supervised** → Learning without humans labeling data; the model teaches itself using its own prediction quality as the signal
- **In-context learning** → Giving a model a few examples inside the prompt so it learns the pattern on the fly (no weight updates)
- **Cross-entropy loss** → A measure of how surprised the model is by the next token; lower = better predictions
- **Perplexity** → How "confused" a model is when predicting text; lower = more fluent/accurate
- **Filtering threshold (τ_f)** → The minimum improvement an API call must provide to be kept in training data
- **Sampling threshold (τ_s)** → The minimum probability the model must assign to the `<API>` token for a position to be considered
- **GPT-J** → A 6.7 billion parameter open-source language model (the base model for Toolformer)
- **CCNet** → A large web-crawled text dataset used for training
- **BM25** → A classic keyword-matching search algorithm (not neural) used for Wikipedia retrieval
- **Atlas** → A retrieval-augmented QA model used as Toolformer's question-answering tool
- **NLLB** → Meta's "No Language Left Behind" translation model supporting 200 languages
- **Zero-shot** → Testing a model on a task it was never explicitly trained for, with no examples
- **Greedy decoding** → Always picking the most likely next token during text generation
- **Fine-tuning** → Continuing to train a pre-trained model on new data to add capabilities
- **Weighted loss** → A loss function where nearby tokens matter more than distant ones (using decaying weights)
- **Special tokens (`<API>`, `</API>`, `→`)** → Markers that signal the start, end, and result delimiter of tool calls

---

## 🔗 7. HOW IT CONNECTS

### Intellectual family tree:
```
In-context Learning (GPT-3, Brown et al. 2020)
     │
     ├── Self-Instruct (Wang et al. 2022)
     │        │
     ├── Dataset Generation with LMs (Schick & Schütze 2021)
     │        │
     │        └──────────┐
     │                   ▼
     │            ┌─────────────┐
     │            │ TOOLFORMER  │
     │            └─────────────┘
     │                   ▲
     ├── TALM (Parisi et al. 2022) ── similar self-supervised
     │   idea but task-specific
     │
     ├── PAL (Gao et al. 2022) ── tool use via prompting only
     │
     ├── LaMDA (Thoppilan et al. 2022) ── tool use via
     │   massive human supervision
     │
     └── REALM/RETRO (Guu 2020, Borgeaud 2021) ── retrieval-
         augmented LMs (always retrieve, never learn when)
```

### Who would use this:
- **LLM developers** wanting to add tool use without massive annotation budgets
- **AI assistants** that need to be accurate on math, facts, and current events
- **Researchers** studying how to extend LM capabilities beyond pure text generation

### Future work enabled:
- **Tool chaining** (use one tool's output as another's input)
- **Interactive search** (refine queries based on results)
- **More tools** (code interpreters, databases, image generators)
- **Iterative bootstrapping** (re-run the pipeline to improve data quality)
- This directly foreshadowed modern **agent frameworks** (AutoGPT, function calling in GPT-4, etc.)

---

## ⚖️ 8. CRITICAL ANALYSIS

### Hidden assumptions:
- **The model's language modeling loss is a good proxy for "usefulness."** But reducing perplexity ≠ being correct or helpful for downstream tasks
- **A handful of demonstrations per API is sufficient.** Quality/diversity of these few-shot examples could significantly bias what calls get generated
- **Filtering out all-filtered-out examples doesn't change the data distribution meaningfully.** They claim this but only weakly verify it

### Weaknesses the authors DON'T mention:
- **The tools themselves must be high quality.** A bad QA system → bad training signal → bad model. The approach inherits tool limitations
- **Only tested on a single base model (GPT-J).** Unclear if the approach transfers to other architectures (encoder-decoder, etc.)
- **The 5 tools are relatively simple.** No complex tools like code execution, database queries, or multi-step workflows
- **Latency cost at inference is ignored.** Every API call adds wall-clock time; no analysis of inference speed impact
- **No comparison with concurrent work like ChatGPT plugins** or other tool-augmented systems emerging at the time
- **The modified decoding (k=10) is a hack.** The model doesn't naturally want to call APIs enough — they have to artificially boost the probability

### Is the evaluation fair?
- **Lenient evaluation criteria** (checking first 5-20 words for answer) makes results less comparable to other work
- **Disabling specific tools for specific benchmarks** (e.g., QA tool for QA tasks, WikiSearch for LAMA) is reasonable but makes cross-comparison tricky
- **Zero-shot only** — no few-shot comparisons, which is where larger models often shine
- **No comparison with models that have built-in retrieval** (like RETRO) under matched conditions

### Would this work at scale in the real world?
- **Data generation is expensive:** Processing millions of documents with multiple API calls each requires significant compute
- **Only ~1k useful calculator examples from 1M+ docs** — very sample inefficient
- **Single API call per input limitation** is quite restrictive for real-world use cases
- **Modern systems (GPT-4 function calling, Claude tool use) have largely superseded this approach** with more flexible methods, but Toolformer's ideas influenced their design

---

## 📝 9. MEMORY ANCHORS

### Memorable metaphor:
**Toolformer is like a student who practices using a dictionary, calculator, and atlas while doing homework, keeps notes on which tools actually helped them get answers right, and then studies those notes before the exam.** No teacher needed — just self-reflection on what worked.

### 3 bullets that capture 80% of the paper:
- 🔧 **LMs can teach themselves to use external tools (calculator, search, QA, translator, calendar) by self-annotating training data with API calls and keeping only the ones that reduce prediction error**
- 📊 **A 6.7B model with tools beats GPT-3 (175B) on math (40.4 vs 14.0 on ASDiv) and fact completion, proving tools > scale for certain tasks**
- 🎯 **The key filtering criterion: keep an API call only if having both the call AND its response improves next-token prediction more than having neither or just the call alone**

### One question to test understanding:
> *Why does Toolformer compare L⁺ against the MINIMUM of (no API call) and (API call without response), rather than just comparing against "no API call"?*

**Answer:** Because if just seeing the API call *without* the response already helps (e.g., the question itself contains useful info), then the tool's response isn't truly needed. The filter ensures the tool's **actual output** provides value beyond what the model could infer from the question alone.

---

## 🗺️ 10. VISUAL MENTAL MAP

```
THE PROBLEM                          THE METHOD                              THE RESULT
━━━━━━━━━━━━━━                      ━━━━━━━━━━━━━━                          ━━━━━━━━━━━━━━

LLMs are smart but...               SELF-SUPERVISED PIPELINE                Toolformer (6.7B)
                                                                             beats GPT-3 (175B)
❌ Can't do math      ──┐            ┌─────────────────────┐                 on math & facts
❌ Hallucinate facts  ──┤   ┌───────▶│ 1. Few-shot prompt  │
❌ No current info    ──┤   │        │    → sample API     │                 ✅ Math: 40.4 vs 14.0
❌ Bad at low-resource ─┤   │        │    calls in text    │                 ✅ LAMA: 53.5 vs 39.8
   languages           │   │        └────────┬────────────┘                 ✅ DATESET: 27.3 vs 0.8
                        │   │                 │
Previous solutions      │   │        ┌────────▼────────────┐                 While also...
are bad because...      │   │        │ 2. Execute calls    │                 ✅ No perplexity loss
                        │   │        │    (calc, QA, wiki,  │                 ✅ Keeps LM abilities
❌ Need tons of human ──┤   │        │    translate, cal)   │                 ✅ Works zero-shot
   annotations          │   │        └────────┬────────────┘
❌ Task-specific only ──┘   │                 │                               But limited by...
                            │        ┌────────▼────────────┐
        KEY INSIGHT         │        │ 3. Filter: Keep     │                 ❌ No tool chaining
        ━━━━━━━━━━━        │        │    only if           │                 ❌ No interactive use
Let the MODEL decide       │        │  L⁻ - L⁺ ≥ τ_f     │                 ❌ Needs ≥775M params
what's useful via its      │        └────────┬────────────┘                 ❌ Sample inefficient
own prediction loss!       │                 │
                            │        ┌────────▼────────────┐
                            │        │ 4. Fine-tune LM on  │
                            │        │    augmented dataset │
                            │        └────────┬────────────┘
                            │                 │
                            │        ┌────────▼────────────┐
                            └────────│ 5. Inference: Model │
                                     │    calls APIs when  │
                                     │    it decides to    │
                                     └─────────────────────┘
```

---

## 🛠️ 11. IMPLEMENTATION SKETCH

### Pseudocode (~20 lines):
```python
def create_toolformer_dataset(texts, model, apis, prompts, τ_s, τ_f):
    augmented_data = []
    for x in texts:
        all_calls = []
        for api in apis:
            prompt = prompts[api]
            # Step 1: Sample candidate positions & API calls
            positions = [i for i in range(len(x))
                        if p_model("<API>" | prompt(x), x[:i]) > τ_s]
            positions = top_k(positions, k=5)
            for i in positions:
                candidates = model.sample(prefix=prompt(x) + x[:i] + "<API>",
                                          stop="</API>", n=5)
                # Step 2: Execute
                for c in candidates:
                    r = api.execute(c)
                    # Step 3: Filter
                    L_plus = weighted_loss(model, x[i:], prefix=e(c, r))
                    L_minus = min(weighted_loss(model, x[i:], prefix=""),
                                  weighted_loss(model, x[i:], prefix=e(c, "")))
                    if L_minus - L_plus >= τ_f:
                        all_calls.append((i, c, r))
        # Step 4: Merge calls into text
        x_star = insert_api_calls(x, all_calls)
        augmented_data.append(x_star)
    # Fine-tune model on augmented_data
    model.finetune(augmented_data, lr=1e-5, batch_size=128)
    return model
```

### Frameworks/libraries needed:
- **PyTorch + HuggingFace Transformers** (GPT-J model)
- **DeepSpeed ZeRO-3** (distributed training)
- **Atlas model** (for QA tool)
- **NLLB model** (for translation)
- **Pyserini or similar** (BM25 retrieval for Wikipedia search)
- **fastText** (language detection)

### Estimated compute cost:
- **Data generation:** Most expensive part — running GPT-J inference on millions of texts with multiple API calls each. Likely **hundreds of GPU-hours** on A100s
- **API execution:** Running Atlas on ~100k+ QA queries, NLLB on ~10k+ translations
- **Fine-tuning:** 2k steps on 8× A100 40GB GPUs ≈ **~few hours**
- **Total estimated:** **~500-1000 A100 GPU-hours** to reproduce end-to-end (rough estimate)
