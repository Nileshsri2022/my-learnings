

# ToolLLM: Facilitating Large Language Models to Master 16000+ Real-World APIs

---

## 🎯 1. THE ONE-LINER
**This paper teaches an open-source AI (like LLaMA) how to use over 16,000 real-world apps and services (APIs) by creating a massive training dataset and a smarter problem-solving strategy, making it almost as good as ChatGPT at using tools.**

---

## 🤔 2. THE PROBLEM (Why does this paper exist?)

- **What problem?** Open-source LLMs (like LLaMA) can chat, write, and answer questions—but they **can't use external tools** (booking flights, checking weather, searching databases). Meanwhile, ChatGPT and GPT-4 can do this well, but they're closed-source black boxes.

- **Why should anyone care?** Imagine you have a super-smart friend who can talk about anything, but they **can't use a phone, computer, or any app**. They can discuss restaurants but can't actually book a table. That's open-source LLMs today. This paper gives them the ability to "pick up the phone and call."

- **Limitations of previous approaches:**
  - **(1) Limited APIs:** Prior datasets used only ~50-400 fake or toy APIs, not real-world ones
  - **(2) Single-tool only:** Previous work only handled one tool at a time (real tasks often need combining weather + maps + restaurants)
  - **(3) Bad reasoning:** Used simple linear reasoning (ReACT/CoT) that **gets stuck in error loops** with no way to backtrack
  - **(4) No real execution:** Some prior work never actually *called* the APIs — they just predicted what the output *would be*

---

## 💡 3. THE KEY IDEA (The "Aha!" moment)

**Core insight:** Instead of hand-crafting a small dataset, **use ChatGPT itself as an automated factory** to generate 126,000+ training examples involving 16,000+ real APIs — and when ChatGPT gets stuck solving hard problems, use a **tree search strategy (DFSDT)** that lets it backtrack and try different paths, like a GPS rerouting when you hit a dead end.

**Everyday analogy — Learning to Cook:**
- Imagine you want to teach someone to cook using any recipe from a massive cookbook with 16,000+ recipes
- **Previous approach:** Give them 50 recipes and hope they figure out the rest (doesn't work for complex meals that combine techniques)
- **ToolLLM approach:**
  1. 📚 Collect all 16,000 recipes (API Collection)
  2. 🧑‍🍳 Ask a master chef (ChatGPT) to create realistic dinner party scenarios ("make a 3-course Italian meal for someone who's gluten-free")
  3. 🔍 Let the master chef try to solve each scenario — but instead of giving up when a dish fails, **let them try different paths** (use a different pasta substitute, swap a recipe) until they find something that works
  4. 📝 Write down all the successful attempts as a training manual
  5. 🎓 Train the student (LLaMA) on this manual

**DFSDT explained simply:**
```
Regular approach (ReACT):         DFSDT (this paper):
Try step 1 → OK                  Try step 1 → OK
Try step 2 → OK                  Try step 2 → OK  
Try step 3 → ERROR               Try step 3 → ERROR
Try step 4 → ERROR (stuck!)      ↩️ BACKTRACK to step 2
Try step 5 → Give up ❌           Try step 3b → OK
                                  Try step 4 → SUCCESS ✅
```

---

## 🏗️ 4. HOW IT WORKS (The Method - Layer by Layer)

### Step 1: API Collection 🗄️
- **WHAT:** Crawl **16,464 real REST APIs** from RapidAPI Hub spanning **49 categories** (finance, weather, movies, sports, etc.)
- **WHY:** Need a massive, diverse, real-world API pool — not fake/toy APIs
- **HOW:** For each API, collect: name, description, parameters, code snippets, example responses. Filter out broken/unreliable ones (53,190 → 16,464 APIs)
- **CONNECTS TO:** These APIs become the "tools" the model learns to use

### Step 2: Instruction Generation 📝
- **WHAT:** Use ChatGPT to generate **~200K diverse instructions** that require using these APIs
- **WHY:** Need realistic user requests that exercise single and multi-tool scenarios
- **HOW:** 
  - Sample a subset of APIs → feed their documentation to ChatGPT → ask it to generate instructions + identify which APIs are needed
  - Three instruction types:
    - **I1: Single-tool** (use 1 tool with multiple APIs)
    - **I2: Intra-category multi-tool** (combine 2-5 tools from same category, e.g., multiple movie tools)
    - **I3: Intra-collection multi-tool** (combine tools across categories)
- **CONNECTS TO:** These instructions become the "homework assignments" for Step 3

### Step 3: Solution Path Annotation 🛤️
- **WHAT:** Use ChatGPT to find valid sequences of API calls that solve each instruction
- **WHY:** Need (instruction, solution) pairs to train the student model
- **HOW:** 
  - ChatGPT reasons step-by-step, calling real APIs and getting real responses
  - Uses **DFSDT (Depth-First Search Decision Tree)** — when stuck, backtrack and try alternative paths
  - Two special end-actions: "Finish with Final Answer" or "Finish by Giving Up"
  - Only keep **successful solution paths** → **126,486 pairs**
- **CONNECTS TO:** These pairs become the training data (ToolBench)

### Step 4: Train ToolLLaMA 🎓
- **WHAT:** Fine-tune LLaMA-2 7B on the 126K instruction-solution pairs
- **WHY:** Transfer ChatGPT's tool-use ability to an open-source model
- **HOW:** Standard supervised fine-tuning (SFT); extend context window from 4096 → 8192 tokens using positional interpolation
- **CONNECTS TO:** ToolLLaMA is the final product

### Step 5: Train API Retriever 🔍
- **WHAT:** Train a BERT-based dense retriever to automatically find relevant APIs for a given instruction
- **WHY:** Users shouldn't have to manually pick from 16,000+ APIs
- **HOW:** Encode instruction & API documentation into embeddings; use contrastive learning (relevant APIs = positive, others = negative)
- **CONNECTS TO:** At inference, the retriever first selects top-5 APIs, then ToolLLaMA uses them

### Step 6: Evaluation via ToolEval 📊
- **WHAT:** Build an automatic evaluator using ChatGPT to judge tool-use quality
- **WHY:** Human evaluation is too slow; API responses change over time so fixed ground-truth doesn't work
- **HOW:** Two metrics:
  - **Pass Rate:** Did the model successfully complete the task?
  - **Win Rate:** Compare two solution paths — which is better?
- **87.1% agreement** with humans on pass rate, **80.3%** on win rate

---

## 📊 5. THE PROOF (Results & Experiments)

### Benchmarks/Datasets:
- **ToolBench** (their own, 6 test subsets: I1-Inst, I1-Tool, I1-Cat, I2-Inst, I2-Cat, I3-Inst)
- **APIBench** (OOD test: TorchHub, TensorHub, HuggingFace)

### Key Numbers:

| Model | Method | Avg Pass Rate | Avg Win Rate |
|-------|--------|:---:|:---:|
| Vicuna/Alpaca | ReACT/DFSDT | **0.0%** | **0.0%** |
| ChatGPT | ReACT | 40.2% | — (baseline) |
| ChatGPT | DFSDT | 64.8% | 64.3% |
| **ToolLLaMA** | **DFSDT** | **66.7%** | **60.0%** |
| GPT-4 | DFSDT | 71.1% | 70.4% |

### Most impressive results in plain English:
- **Vicuna and Alpaca score literally 0%** — they can't use tools at all, proving current instruction tuning ignores this skill
- **ToolLLaMA (7B params) nearly matches ChatGPT** and beats Claude-2 and Text-Davinci-003
- **DFSDT boosts ChatGPT's pass rate from 40% → 65%** — the reasoning strategy alone is a huge win
- **Using the API retriever actually *improves* over ground-truth APIs** (67.3% vs 66.7% pass rate) because the retriever finds better alternatives
- On **OOD APIBench**, ToolLLaMA matches Gorilla (which was *specifically trained on APIBench*)

### Limitations admitted:
- Evaluation of tool learning is inherently difficult — even human experts disagree
- APIs on RapidAPI change over time, making reproducibility challenging
- DFSDT consumes more API calls (cost) than simple ReACT

---

## 🧩 6. KEY TERMS GLOSSARY

- **LLM (Large Language Model)** → An AI that understands and generates text (like ChatGPT, LLaMA)
- **API (Application Programming Interface)** → A way for programs to talk to services (e.g., "get weather for NYC")
- **RESTful API** → A standard format for web APIs using HTTP methods (GET, POST, etc.)
- **Tool Learning** → Teaching AI to use external tools/APIs to accomplish tasks
- **Instruction Tuning** → Training a model on (instruction, response) pairs so it follows directions better
- **SFT (Supervised Fine-Tuning)** → Training a model using labeled examples
- **ReACT** → A prompting method where the model alternates between *reasoning* (thinking) and *acting* (calling APIs)
- **CoT (Chain-of-Thought)** → Prompting the model to think step-by-step
- **DFSDT (Depth-First Search-based Decision Tree)** → A tree search where the model can backtrack and try alternative reasoning paths
- **Pass Rate** → Percentage of instructions the model successfully completes
- **Win Rate** → How often a model's solution is preferred over a baseline's
- **ToolBench** → The training dataset of 126K (instruction, solution path) pairs created in this paper
- **ToolEval** → The automatic evaluator using ChatGPT to judge tool-use quality
- **ToolLLaMA** → LLaMA-2 7B fine-tuned on ToolBench
- **RapidAPI** → A marketplace hosting thousands of real-world APIs
- **NDCG** → Normalized Discounted Cumulative Gain; a ranking quality metric for retrieval
- **API Retriever** → A model that finds the right APIs given a user instruction
- **Positional Interpolation** → A technique to extend a model's maximum input length
- **Contrastive Learning** → Training by pushing similar items together and dissimilar items apart in embedding space
- **OOD (Out-of-Distribution)** → Testing on data completely different from training data
- **AST Accuracy** → Abstract Syntax Tree accuracy; measures if the generated API call is structurally correct
- **Hallucination** → When the model makes up APIs or information that don't exist
- **Function Call** → A ChatGPT feature allowing structured tool/function invocation

---

## 🔗 7. HOW IT CONNECTS

### Intellectual Family Tree:
```
Self-Instruct (Wang et al., 2022)
    └── Instruction tuning via LLM-generated data
        └── ToolBench data construction

ReACT (Yao et al., 2022)
    └── Reasoning + Acting framework
        └── DFSDT extends this with backtracking

Tree-of-Thought (Yao et al., 2023) [concurrent work]
    └── Tree search for reasoning
        └── DFSDT adapts this for infinite decision spaces

Reflexion (Shinn et al., 2023)
    └── Learning from failures
        └── DFSDT generalizes this to multiple paths

Gorilla (Patil et al., 2023)
    └── LLM + API benchmark
        └── ToolLLM scales to 16K+ APIs with multi-tool support

Toolformer (Schick et al., 2023)
    └── Self-supervised tool learning
        └── ToolLLM uses supervised instruction tuning instead
```

### Who would use this?
- **AI developers** building LLM-powered agents that interact with real services
- **Researchers** studying tool use, planning, and decision-making in LLMs
- **Companies** wanting open-source alternatives to ChatGPT for tool-augmented applications

### Future work enabled:
- More advanced tree search / planning algorithms for LLM agents
- Scaling to even more APIs and cross-domain tool compositions
- Better evaluation frameworks for complex agent behaviors
- Multi-modal tool use (combining text APIs with image/video APIs)

---

## ⚖️ 8. CRITICAL ANALYSIS

### Hidden Assumptions:
- **ChatGPT is a reliable teacher** — the entire dataset quality depends on ChatGPT's own tool-use abilities, including its biases and errors
- **API documentation is sufficient** — assumes models can learn to use APIs purely from their descriptions (no code examples, tutorials, etc.)
- **APIs remain stable** — real-world APIs change, break, or get deprecated, but the model is trained on a static snapshot

### Weaknesses the authors DON'T mention:
- **Cost of data construction:** Creating 126K examples using ChatGPT (gpt-3.5-turbo-16k) with DFSDT likely cost **thousands of dollars** in API fees — they never disclose this
- **RapidAPI bias:** The APIs are all from one marketplace and may not represent enterprise APIs, internal APIs, or APIs requiring authentication flows
- **No safety analysis:** A model that can call 16K real-world APIs could cause real harm (spam APIs, financial transactions, data leaks) — no discussion of safety guardrails
- **Distillation legality:** Training on ChatGPT outputs may violate OpenAI's terms of service
- **API response compression** may lose critical information — they truncate to 1024 tokens
- **Only tested on LLaMA-2 7B** — no experiments with larger models or other architectures

### Is the evaluation fair?
- **Partially.** ToolEval using ChatGPT to evaluate is convenient but creates a **circular dependency** — ChatGPT generates the data AND judges the results
- The 87.1%/80.3% human agreement is decent but not stellar, especially for win rate
- **No error analysis** — what types of instructions fail? Which API categories are hardest?

### Would this work at scale in the real world?
- **Partially.** The API retriever + ToolLLaMA pipeline is promising, but:
  - Real APIs require authentication, rate limiting, error handling
  - APIs change constantly — the model needs continuous retraining or adaptation
  - 7B parameter model may be too slow for real-time applications requiring multiple API calls
  - Security/safety concerns are unaddressed

---

## 📝 9. MEMORY ANCHORS

### Memorable Metaphor:
> **ToolLLM is like building a "driving school" for AI** — instead of teaching with toy cars on a closed course (previous work), they built a massive training city with 16,000 real roads (APIs), hired an experienced driver (ChatGPT) to create lesson plans and demonstrate routes, gave the student driver a GPS that can reroute (DFSDT), and the graduate (ToolLLaMA) can now drive almost as well as the instructor.

### 3 Bullets That Capture 80%:
- 📦 **Built ToolBench**: 126K training examples spanning 16,464 real APIs, auto-generated by ChatGPT with multi-tool scenarios
- 🌳 **Invented DFSDT**: A tree-search reasoning strategy that lets LLMs backtrack and explore multiple paths, boosting pass rate from 35% → 64% over ReACT
- 🎓 **Trained ToolLLaMA**: A 7B open-source model that matches ChatGPT at tool use and generalizes to unseen APIs and out-of-distribution benchmarks

### Understanding Test Question:
> *"Why does DFSDT outperform ReACT, and why is DFS preferred over BFS in this context?"*

**Answer you should give:** DFSDT outperforms ReACT because ReACT follows a single linear chain — if it makes a mistake, errors propagate and it gets stuck. DFSDT builds a decision tree allowing backtracking and exploring alternative reasoning paths. DFS is preferred over BFS because we only need **one** valid solution path (not all solutions), so going deep along one branch first is more cost-efficient — BFS would expand all branches simultaneously, wasting API calls.

---

## 🗺️ 10. VISUAL MENTAL MAP

```
┌─────────────────────────────────────────────────────────────────┐
│                        PROBLEM                                  │
│  Open-source LLMs can't use tools; closed-source ones can       │
│  Prior datasets: few APIs, single-tool, no real execution       │
└──────────────────────────┬──────────────────────────────────────┘
                           ▼
┌─────────────────── DATA CONSTRUCTION ───────────────────────────┐
│                                                                 │
│  ① API Collection        ② Instruction Gen     ③ Solution Path │
│  ┌──────────────┐   ┌──────────────────┐   ┌─────────────────┐ │
│  │ RapidAPI Hub │──▶│ ChatGPT generates │──▶│ ChatGPT solves  │ │
│  │ 16,464 APIs  │   │ ~200K instructions│   │ using DFSDT 🌳  │ │
│  │ 49 categories│   │ Single + Multi    │   │ 126K solutions  │ │
│  └──────────────┘   └──────────────────┘   └─────────────────┘ │
│                                                                 │
│                    = ToolBench Dataset 📦                        │
└──────────────────────────┬──────────────────────────────────────┘
                           ▼
┌─────────────────── MODEL TRAINING ──────────────────────────────┐
│                                                                 │
│  ┌────────────┐    SFT     ┌──────────────┐                    │
│  │  LLaMA-2   │──────────▶│  ToolLLaMA   │                    │
│  │    7B      │           │  (7B, 8192   │                    │
│  └────────────┘           │   context)   │                    │
│                           └──────┬───────┘                    │
│  ┌────────────┐  contrastive    │                              │
│  │ BERT-base  │──────────▶ API Retriever                      │
│  └────────────┘                  │                              │
└──────────────────────────┬───────┘──────────────────────────────┘
                           ▼
┌─────────────────── INFERENCE PIPELINE ──────────────────────────┐
│                                                                 │
│  User Instruction                                               │
│       │                                                         │
│       ▼                                                         │
│  API Retriever ──▶ Top-5 APIs ──▶ ToolLLaMA + DFSDT            │
│                                        │                        │
│                                        ▼                        │
│                                   Final Answer                  │
└──────────────────────────┬──────────────────────────────────────┘
                           ▼
┌─────────────────── EVALUATION (ToolEval) ───────────────────────┐
│                                                                 │
│  Pass Rate: 66.7% (≈ ChatGPT 64.8%, close to GPT-4 71.1%)     │
│  Win Rate:  60.0% vs ChatGPT-ReACT baseline                    │
│  OOD:       Matches Gorilla on APIBench (never trained on it)   │
│  Vicuna/Alpaca: 0% (can't use tools at all!)                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🛠️ 11. IMPLEMENTATION SKETCH

### Pseudocode: DFSDT Core Algorithm (~15 lines)
```python
def dfsdt(instruction, apis, max_depth, max_children):
    root = Node(instruction, apis)
    stack = [root]
    
    while stack:
        node = stack[-1]
        if node.depth >= max_depth:
            stack.pop()  # backtrack
            continue
        
        # Generate next action (encourage diversity from siblings)
        action = llm.generate(
            node.history, 
            previous_siblings=node.parent.children if node.parent else []
        )
        
        if action.type == "finish_with_answer":
            return action.answer  # SUCCESS ✅
        elif action.type == "give_up":
            stack.pop()  # backtrack, try sibling
            if node.parent and len(node.parent.children) < max_children:
                new_child = Node(node.parent.history)  # expand new branch
                stack.append(new_child)
            continue
        else:
            # Execute real API call
            response = execute_api(action.api_name, action.params)
            child = Node(node.history + [(action, response)])
            node.children.append(child)
            stack.append(child)
    
    return "FAILED"  # exhausted search space
```

### Frameworks/Libraries Needed:
- **PyTorch** + **Hugging Face Transformers** (model training)
- **DeepSpeed** or **FSDP** (distributed training for 7B model)
- **Sentence-Transformers** (API retriever)
- **OpenAI API** (data generation with ChatGPT)
- **Requests** library (real API execution)
- **RapidAPI key** (access to APIs)

### Estimated Compute Cost:
- **Data generation:** ~$5,000-15,000 in OpenAI API fees (126K examples × DFSDT with multiple calls each — ~469K total API calls)
- **Model training:** LLaMA-2 7B fine-tuning for 2 epochs on 126K examples with 8192 seq length → ~**8× A100 GPUs for ~24-48 hours** (estimated ~$500-1000 on cloud)
- **API retriever training:** BERT-base, minimal compute (~1 GPU, few hours)
- **Total estimated reproduction cost:** ~$6,000-16,000+ (dominated by OpenAI API fees)
