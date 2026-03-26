

# Gorilla: Large Language Model Connected with Massive APIs

---

## 🎯 1. THE ONE-LINER
**Gorilla is a trained AI assistant that can correctly pick and write the right code to use thousands of software tools (APIs), something even the smartest AI like GPT-4 often gets wrong by making stuff up.**

---

## 🤔 2. THE PROBLEM (Why does this paper exist?)

- **The problem:** LLMs like GPT-4 are great at chatting and reasoning, but when you ask them to write code that *calls a specific tool/API*, they frequently **hallucinate** — they invent API names, use wrong arguments, or reference tools that don't exist.
- **Why should anyone care?** Imagine you ask a librarian to find you a specific book. Instead of admitting they don't know, they **confidently hand you a book with a made-up title that doesn't exist**. That's what GPT-4 does with API calls. If we want AI to actually *do things* in the real world (book flights, run ML models, control robots), it needs to call the right tools correctly.
- **Limitations of previous approaches:**
  - **Toolformer / HuggingGPT**: Only handled a **small, hand-curated set** of tools that fit in the prompt
  - **Prompting-based approaches**: Can't scale to thousands of overlapping APIs — you can't fit them all in context
  - **No systematic evaluation**: Prior work mostly demonstrated prompting tricks, not rigorous benchmarks
  - **Stale knowledge**: LLMs are trained once and can't adapt when API documentation updates

---

## 💡 3. THE KEY IDEA (The "Aha!" moment)

**Core insight:** Instead of just *prompting* a generic LLM with API docs, **fine-tune** the LLM specifically on API calls AND teach it to *use retrieved documentation* during both training and inference.

### Everyday Analogy: 🍳
Imagine training a chef (LLM) to cook:
- **Previous approach**: Hand the chef a recipe book at cooking time and hope they follow it (prompting with retrieval)
- **Gorilla's approach**: **Train the chef specifically on thousands of recipes** AND teach them to look up the latest version of a recipe when cooking. The chef learns both the *skill of cooking* and the *skill of reading recipes on the fly*.

### The method in 3 key moves:
1. **Build a massive API cookbook** (APIBench) — 1,645 APIs from HuggingFace, TorchHub, TensorHub
2. **Generate training data** using self-instruct (GPT-4 writes natural language questions for each API)
3. **Fine-tune LLaMA-7B** with retriever-aware training — the model learns to combine its own knowledge with freshly retrieved docs

---

## 🏗️ 4. HOW IT WORKS (The Method - Layer by Layer)

### Step 1: Build the API Database (APIBench)
- **WHAT:** Scraped model cards from 3 ML model hubs → 1,645 APIs total
  - TorchHub: 94 APIs (exhaustive)
  - TensorFlow Hub: 626 APIs (exhaustive after filtering)
  - HuggingFace: 925 APIs (top 20 per task domain)
- **WHY:** No existing benchmark measured LLMs' ability to call APIs at scale
- **HOW it connects:** Each API is stored as a JSON with fields: `{domain, framework, functionality, api_name, api_call, api_arguments, environment_requirements, example_code, performance, description}`

### Step 2: Generate Instruction-API Pairs (Self-Instruct)
- **WHAT:** Used GPT-4 to generate 10 natural language questions per API → **16,450 {instruction, API} pairs**
- **WHY:** Need realistic user questions to train on (e.g., "Help me classify objects in an image")
- **HOW:** Gave GPT-4 three in-context examples + API doc → asked it to generate use cases WITHOUT mentioning the API name
- **Connects to:** These pairs become the fine-tuning dataset

### Step 3: Fine-tune Gorilla (LLaMA-7B)
- **WHAT:** Standard instruction fine-tuning on LLaMA-7B using the generated pairs
- **WHY:** Teaching the model the specific mapping from task descriptions to correct API calls
- **Training details:** 5 epochs, lr=2e-5, batch size 64, 8×A100 GPUs

### Step 4: Retriever-Aware Training (The secret sauce 🌟)
- **WHAT:** During training, some examples include retrieved API documentation appended to the prompt: `"Use this API documentation for reference: <API_doc_JSON>"`
- **WHY:** Teaches the model to **parse and use** retrieved docs, enabling:
  - a) Adaptation to documentation changes at test time
  - b) Better performance via in-context learning
  - c) Reduced hallucination
- **Connects to:** At inference, a retriever fetches the latest docs and appends them the same way

### Step 5: Inference (Two Modes)
- **Zero-shot:** User prompt → Gorilla → API call
- **With retrieval:** User prompt → Retriever fetches top-1 API doc → concatenated → Gorilla → API call

### Step 6: Evaluation via AST Sub-Tree Matching
- **WHAT:** Parse the generated API call into an Abstract Syntax Tree, then check if it matches a subtree in the API database
- **WHY:** Multiple APIs can solve the same task; unit tests can't verify semantic correctness
- **Hallucination definition:** API call that matches NO subtree in the entire database = the model invented a tool

```
           ┌──────────────┐
           │  User Query  │
           └──────┬───────┘
                  │
          ┌───────┴────────┐
          │   Retriever?   │
          └───────┬────────┘
         yes/     │      \no
        ┌─────┐   │   ┌────────┐
        │BM25/│   │   │Zero-   │
        │GPT  │   │   │shot    │
        └──┬──┘   │   └───┬────┘
           │      │       │
     ┌─────▼──────▼───────▼─────┐
     │  "Query + Use this API   │
     │   doc for reference:..." │
     └──────────┬───────────────┘
                │
        ┌───────▼───────┐
        │   GORILLA     │
        │  (LLaMA-7B    │
        │  fine-tuned)  │
        └───────┬───────┘
                │
        ┌───────▼───────┐
        │  API Call      │
        │  Output        │
        └───────┬───────┘
                │
        ┌───────▼───────┐
        │ AST Matching  │
        │ Evaluation    │
        └───────────────┘
```

---

## 📊 5. THE PROOF (Results & Experiments)

### Benchmarks
- **APIBench**: 3 sub-datasets — TorchHub, HuggingFace, TensorFlow Hub
- **Baselines**: GPT-4, GPT-3.5, Claude, LLaMA-7B
- **4 retrieval settings**: Zero-shot, BM25, GPT-Index, Oracle

### Key Numbers (Zero-shot, the hardest setting):

| Model | TorchHub | HuggingFace | TensorFlow Hub |
|-------|----------|-------------|----------------|
| GPT-4 | 38.70% | 19.80% | 18.20% |
| GPT-3.5 | 48.38% | 16.81% | 41.75% |
| **Gorilla** | **59.13%** | **71.68%** | **83.79%** |

### Most impressive results in plain English:
- **Gorilla (zero-shot) beats GPT-4 by 20.43% overall** and reduces hallucination dramatically
- On HuggingFace: Gorilla hallucinates **10.95%** of the time vs GPT-4's **37.16%**
- With Oracle retriever: Gorilla reaches **91.26%** on HuggingFace (vs GPT-3.5's 89.71%)
- **Gorilla adapts to test-time API changes** — when documentation is updated, retrieved-aware Gorilla adjusts its output accordingly

### Surprising finding:
- **GPT-3.5 hallucinates less than GPT-4** across all hubs — suggesting RLHF may play a role in truthfulness
- **Adding a bad retriever can HURT performance** — non-optimal retrievers misguide the model

### Limitations admitted:
- Only tested on ML APIs (domain-specific)
- Retriever quality is a bottleneck — BM25 causes significant accuracy drops
- Didn't test actual execution of generated API calls

---

## 🧩 6. KEY TERMS GLOSSARY

- **API (Application Programming Interface)** → A specific way to call a software tool, like a phone number for a service
- **LLM (Large Language Model)** → AI models like GPT-4 that understand and generate text
- **Hallucination** → When an LLM confidently makes up something that doesn't exist (e.g., a fake API name)
- **LLaMA** → Meta's open-source language model, the base model Gorilla builds on
- **Fine-tuning** → Additional training of a pre-trained model on specific data to specialize it
- **Self-Instruct** → A technique where an LLM generates its own training examples from seed data
- **AST (Abstract Syntax Tree)** → A tree representation of code structure; used to compare if two code snippets are functionally the same
- **BM25** → A classic text search algorithm that finds relevant documents by keyword matching
- **GPT-Index** → A retrieval system using OpenAI's text-davinci-003 model
- **Oracle Retriever** → A "cheat" retriever that always returns the correct API doc (upper bound)
- **Retriever-Aware Training** → Training the model with retrieved documents included in the prompt, so it learns to use them
- **APIBench** → The benchmark dataset of 1,645 APIs created in this paper
- **Model Hub** → An online platform hosting pre-trained ML models (like an app store for AI models)
- **Instruction-tuning** → Fine-tuning an LLM to follow natural language instructions
- **Zero-shot** → Using a model without any examples or retrieved context — just the raw query

---

## 🔗 7. HOW IT CONNECTS

### Intellectual Family Tree:
```
Toolformer (2023)          Self-Instruct (2022)     LLaMA (2023)
  "LLMs can learn            "Generate training       "Open-source
   to use tools"              data with LLMs"          base model"
       \                          |                      /
        \                         |                     /
         ╚═════════════╦══════════╩════════════════════╝
                       ║
                  GORILLA (2023)
                  "Fine-tune LLaMA on
                   massive API calls +
                   retrieval-aware training"
                       ║
              ╔════════╩═════════╗
              ║                  ║
      TaskMatrix.AI        HuggingGPT
      (concurrent)         (concurrent)
```

### Who would use this:
- **Developers** wanting AI assistants that correctly call ML model APIs
- **Platform builders** creating LLM-powered automation tools
- **Researchers** studying tool-use in LLMs

### Future work enabled:
- Extending to **RESTful APIs** (web services, cloud APIs — millions of them)
- Better retrievers would unlock even higher performance
- Multi-step API call chains (Gorilla only does single calls)
- Execution verification of generated API calls

---

## ⚖️ 8. CRITICAL ANALYSIS

### Hidden Assumptions:
- **API calls are single-line**: Real-world usage often involves multi-step API orchestration
- **ML APIs are representative**: The paper assumes insights transfer to REST APIs, web APIs, etc. — this is unproven
- **Retriever always returns something relevant**: What happens when no API in the database matches the user's need?

### Weaknesses NOT mentioned:
- **Only 7B parameters** — unclear how much the small model size limits generalization to truly massive API spaces (millions of APIs)
- **Training data generated by GPT-4** — introduces a dependency on the very model they're trying to beat; also means training data quality is bounded by GPT-4's understanding
- **No execution-based evaluation** — they check if the API *name* is right via AST matching, but never run the code to see if it actually works
- **HuggingFace evaluation is relaxed** — for baseline models, they only check domain names (not full API calls), giving Gorilla an unfair advantage in direct comparison
- **Small scale** — 1,645 APIs is far from the "millions" of APIs mentioned in the motivation

### Is evaluation fair?
- Partially. The AST matching is a reasonable proxy but **not equivalent to functional correctness**
- The relaxed HuggingFace evaluation for baselines makes the comparison uneven
- The "oracle retriever" setting is somewhat unrealistic as an evaluation scenario

### Real-world scalability:
- At 1,645 APIs it works well; scaling to millions remains an open question
- Retriever quality is a clear bottleneck — BM25 causes 20-50% accuracy drops
- Documentation changes are handled, but only when the retriever finds the right updated doc

---

## 📝 9. MEMORY ANCHORS

### Memorable Metaphor:
> **Gorilla is like a well-trained phone operator for a company with 1,600 departments.** GPT-4 is like a smart person who *guesses* which department to connect you to (and often makes up departments that don't exist). Gorilla was specifically trained on the company directory AND can look up the latest directory in real-time.

### 3 Bullets That Capture 80%:
- 🦍 **Fine-tuning LLaMA-7B on 16K instruction-API pairs makes it beat GPT-4** at writing correct API calls while hallucinating far less
- 📚 **Retriever-aware training** (including API docs in training prompts) lets Gorilla adapt to documentation changes at test time
- 🌳 **AST sub-tree matching** provides a principled way to evaluate if a generated API call is functionally correct without running the code

### Comprehension Test Question:
> *Why does adding a retriever at test time sometimes HURT performance for a model that wasn't trained with retrieval, and how does retriever-aware training solve this?*

---

## 🗺️ 10. VISUAL MENTAL MAP

```
PROBLEM                          METHOD                              RESULT
═══════                          ══════                              ══════

LLMs hallucinate    ──►  Scrape 1,645 ML APIs         ──►  Gorilla zero-shot:
when calling APIs        from 3 model hubs                  59%/72%/84% accuracy
                              │                              (TH/HF/TFH)
GPT-4 invents              │                                    │
fake API names    ──►  Self-Instruct: generate        ──►  Beats GPT-4 by ~20%
                         16,450 {question, API}             overall
Can't scale to               │                                    │
1000s of APIs     ──►  Fine-tune LLaMA-7B             ──►  Hallucination: ~5-11%
                         on these pairs                     (vs GPT-4's 37-79%)
                              │                                    │
APIs change over  ──►  Retriever-Aware Training       ──►  Adapts to API doc
time                    (include docs in training)          changes at test time
                              │                                    │
Hard to evaluate  ──►  AST Sub-Tree Matching          ──►  Principled eval of
API correctness         for evaluation                      functional correctness
```

---

## 🛠️ 11. IMPLEMENTATION SKETCH

### Pseudocode (Core Pipeline):
```python
# === TRAINING ===
apis = scrape_model_hubs(["TorchHub", "TFHub", "HuggingFace"])  # 1,645 APIs
api_jsons = convert_to_json(apis)  # {domain, api_call, args, ...}

training_data = []
for api in api_jsons:
    examples = sample(seed_examples[api.hub], k=3)
    instructions = gpt4_generate(api, examples, n=10)  # Self-Instruct
    for inst in instructions:
        # Retriever-aware: 50% with doc, 50% without
        if random() > 0.5:
            prompt = inst + " Use this API doc: " + api.json()
        else:
            prompt = inst
        training_data.append({"user": prompt, "assistant": api.response})

gorilla = finetune(llama_7b, training_data, epochs=5, lr=2e-5)

# === INFERENCE ===
def gorilla_inference(user_query, use_retriever=True):
    if use_retriever:
        api_doc = retriever.search(user_query, top_k=1)  # BM25 or GPT-Index
        prompt = user_query + " Use this API doc: " + api_doc
    else:
        prompt = user_query
    return gorilla.generate(prompt)

# === EVALUATION ===
def evaluate(predicted_api_call, api_database):
    pred_ast = parse_to_ast(predicted_api_call)
    for ref_api in api_database:
        if is_subtree(pred_ast, ref_api.ast):
            return "correct" if args_match(pred_ast, ref_api) else "error"
    return "hallucination"  # No match in entire database
```

### Frameworks/Libraries Needed:
- **PyTorch** + **Hugging Face Transformers** (for LLaMA fine-tuning)
- **BM25 (rank_bm25)** or **LlamaIndex (GPT-Index)** for retrieval
- **Python `ast` module** for AST parsing and subtree matching
- **8× A100 40GB GPUs** for training

### Estimated Compute Cost:
- **Training:** ~few hours on 8×A100 (LLaMA-7B is relatively small)
- **Data generation:** ~$200-500 in GPT-4 API calls for 16,450 instructions
- **Inference:** Single GPU sufficient (7B model)
- **Total reproduction cost:** ~$500-1,000 (compute) + GPT-4 API costs
