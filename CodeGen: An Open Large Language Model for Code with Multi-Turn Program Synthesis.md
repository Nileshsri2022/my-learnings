

# CodeGen: An Open Large Language Model for Code with Multi-Turn Program Synthesis

---

## 🎯 1. THE ONE-LINER
**Salesforce built a family of open-source AI models (up to 16 billion parameters) that can write computer programs by having a back-and-forth conversation with you, where you describe what you want step-by-step in plain English.**

---

## 🤔 2. THE PROBLEM (Why does this paper exist?)

- **The real problem:** Writing code is hard. Automating it ("program synthesis") has been called the "holy grail of computer science." Previous AI models that could do this well (like OpenAI's Codex) were **closed-source** — no one outside those companies could study, improve, or use them freely.

- **Why should anyone care?** Imagine you want to build a house but the only architect who can help you works behind a locked door, and you can't see their blueprints or learn from them. That's what the AI coding world looked like — the best models were hidden. CodeGen **opens the door**.

- **Limitations of previous approaches:**
  - **Closed access:** Codex (behind GitHub Copilot) was not publicly available for researchers to study
  - **Single-turn only:** Previous models expected you to describe the **entire** program in one big prompt — like trying to explain an entire recipe in one sentence
  - **Search space problem:** The space of all possible programs is astronomically large; earlier methods used restricted languages to keep it manageable, limiting generality
  - **Intent specification is hard:** Describing exactly what you want a program to do in one shot is genuinely difficult

---

## 💡 3. THE KEY IDEA (The "Aha!" moment)

**Core insight:** Instead of asking the AI to write an entire program from one big description, **break the task into small steps** — like a conversation. Each step is easier for the model to understand, and it builds the program piece by piece.

### Everyday Analogy: **The IKEA Instruction Approach**
- **Old way (single-turn):** "Build me a bookshelf." → Here's the entire bookshelf (probably wrong)
- **New way (multi-turn):** 
  - "First, lay out the side panels" → ✅ Done
  - "Now attach the shelves with screws" → ✅ Done  
  - "Finally, mount it to the wall" → ✅ Done

**Second key insight:** Code on GitHub already has this pattern! Programmers write **comments explaining what the next block of code does**, then write the code. This interleaved comment-code pattern acts as **weak but free training data** for multi-turn code generation — no special labeling needed.

### How it works step-by-step (like explaining to a friend):
```
Turn 1: Human says "Import regex and define an email pattern"
        → Model writes: import re; pattern = re.compile(...)
        
Turn 2: Human says "Search for email in this text"
        → Model sees Turn 1's code + this new prompt
        → Model writes: match = pattern.search(text)
        
Turn 3: Human says "Extract just the username part"
        → Model sees ALL previous code + this prompt
        → Model writes: username = match[:match.find("@")]
```

---

## 🏗️ 4. HOW IT WORKS (The Method - Layer by Layer)

### Step 1: Train on Natural Language (CODEGEN-NL)
- **WHAT:** Train a standard transformer language model on **ThePile** — 825 GB of English text (books, Wikipedia, GitHub, etc.)
- **WHY:** The model first needs to understand English well before it can understand programming instructions
- **HOW it connects:** This becomes the foundation; the model learns language patterns and some code (7.6% of ThePile is code)

### Step 2: Fine-tune on Multi-Language Code (CODEGEN-MULTI)
- **WHAT:** Take CODEGEN-NL and continue training on **BigQuery** — code in 6 languages (C, C++, Go, Java, JavaScript, Python)
- **WHY:** The model needs to understand programming concepts across languages — loops, functions, data structures
- **HOW it connects:** Initializing from NL model lets it transfer natural language understanding to code comprehension

### Step 3: Specialize on Python (CODEGEN-MONO)
- **WHAT:** Take CODEGEN-MULTI and continue training on **BigPython** — 217 GB of Python code from GitHub
- **WHY:** To maximize Python code generation quality (since benchmarks test Python)
- **HOW it connects:** Each stage **progressively narrows** from general language → multi-language code → Python-specific expertise

### Step 4: Multi-Turn Generation at Inference
- **WHAT:** At test time, the user provides a problem broken into multiple natural language prompts. After each prompt, the model generates code. The **entire history** (all previous prompts + generated code) is concatenated and fed as context for the next turn.
- **WHY:** Breaking a complex specification into pieces reduces the search space and makes each sub-problem easier
- **HOW it connects:** The final generated program is the concatenation of all code from all turns, which is executed and tested

### Step 5: Evaluate with MTPB
- **WHAT:** A new benchmark (115 problems) where each problem is split into 3+ turns of natural language prompts with test cases
- **WHY:** No benchmark existed for multi-turn code generation; needed to measure whether multi-turn actually helps
- **HOW it connects:** Provides quantitative evidence that multi-turn > single-turn

---

## 📊 5. THE PROOF (Results & Experiments)

### Benchmarks Used:
- **HumanEval** (164 Python problems, single-turn) — from OpenAI
- **MTPB** (115 multi-turn Python problems) — **new**, created by the authors
- **MBPP** (Mostly Basic Python Problems) — additional validation

### Key Numbers (HumanEval):

| Model | pass@1 | pass@100 |
|-------|--------|----------|
| Codex 12B | 28.81% | 72.31% |
| **CODEGEN-MONO 16.1B** | **29.28%** | **75.00%** |
| CODEGEN-MONO 6.1B | 26.13% | 65.82% |
| GPT-J 6B | 11.62% | 27.74% |

### Key Numbers (MTPB — multi-turn):

| Model | 350M | 2.7B | 6.1B | 16.1B |
|-------|------|------|------|-------|
| CODEGEN-MONO | 16.98% | 38.72% | 43.52% | **47.34%** |
| code-davinci-002 | - | - | - | **59.86%** |

### Multi-turn vs Single-turn (Table 4 — the **most important result**):
- Multi-turn pass rate: **47.34%** (16.1B)
- Single-turn pass rate: **38.74%** (16.1B)
- **~10 percentage point improvement** just by breaking prompts into steps
- Multi-turn prompts also had **lower perplexity** (8.05 vs 10.25), confirming the model "understands" multi-turn better

### Most impressive result in plain English:
**CODEGEN-MONO 16.1B, which is fully open-source, matched or beat OpenAI's Codex 12B on HumanEval** — despite Codex being closed-source and from a much larger organization.

### Admitted Limitations:
- Still significantly behind the latest Codex variants (code-davinci-002: 47.0% pass@1 vs 29.28%)
- MTPB is only 115 problems (relatively small)
- Larger models sometimes become **inflexible** — taking prompts too literally (e.g., initializing `num = 100` as integer instead of string `"100"`)
- Generated code may contain **vulnerabilities** and **biases** from training data

---

## 🧩 6. KEY TERMS GLOSSARY

**Program Synthesis** → Automatically generating a computer program from a description of what it should do

**Autoregressive Language Model** → A model that generates text one token at a time, where each token depends on all previous tokens (like finishing someone's sentence)

**pass@k** → The probability that at least 1 out of k generated code samples passes all test cases

**Multi-Turn Program Synthesis** → Breaking a programming task into multiple steps, generating code for each step sequentially

**MTPB (Multi-Turn Programming Benchmark)** → A new benchmark of 115 problems designed to test multi-turn code generation

**HumanEval** → OpenAI's benchmark of 164 Python programming problems for testing code generation

**ThePile** → An 825 GB English text dataset used for language model training

**BigQuery** → A dataset of code in 6 programming languages from Google's public dataset

**BigPython** → A 217 GB dataset of Python code from GitHub

**Nucleus Sampling (top-p)** → A text generation strategy that samples from the smallest set of tokens whose cumulative probability exceeds p

**Perplexity** → A measure of how "surprised" a model is by text; lower = better understanding

**Scaling Laws** → The observation that model performance improves predictably as model size and data size increase

**Transformer** → The dominant neural network architecture using attention mechanisms to process sequences

**Rotary Position Embedding (RoPE)** → A method to encode position information in transformer models using rotation matrices

**BPE (Byte-Pair Encoding)** → A tokenization method that breaks text into subword units

**TPU-v4** → Google's custom hardware accelerator for training large AI models

**JAXFORMER** → The custom training library built by the authors for efficient TPU training

**Functional Correctness** → Judging code quality by whether it actually produces the right output, not just whether it "looks right"

**Emergence** → Capabilities that appear in large models that weren't explicitly trained for

---

## 🔗 7. HOW IT CONNECTS

### Intellectual Family Tree:
```
GPT-2/GPT-3 (Brown et al., 2020)
    ├── Codex (Chen et al., 2021) — closed-source code LLM
    ├── GPT-Neo/GPT-J (Black et al., 2021) — open-source text LLMs
    ├── AlphaCode (Li et al., 2022) — competition-level code gen
    └── CODEGEN (this paper) — open-source code LLM + multi-turn

CodeBERT/CuBERT → code understanding (not generation)
Chain-of-Thought (Wei et al., 2022) → breaking reasoning into steps
```

### Related Papers to Compare:
1. **Codex (Chen et al., 2021):** Closed-source, single-turn focused, 12B parameters. CodeGen matches its performance with open weights and adds multi-turn.
2. **AlphaCode (Li et al., 2022):** Focused on competition-level problems with massive sampling (millions of candidates). CodeGen is more practical with fewer samples.
3. **InCoder (Fried et al., 2022):** Open-source, supports code infilling (not just left-to-right), but doesn't explore multi-turn.

### Who would use this:
- **Researchers** studying code generation and program synthesis
- **Developers** building AI coding assistants
- **Companies** wanting an open-source alternative to Codex
- **Educators** exploring AI-assisted programming

### Future work enabled:
- Better multi-turn interaction paradigms for coding
- Instruction-tuned code models (CodeGen2, etc.)
- Combining multi-turn with test-based feedback
- Studying emergent capabilities in code LLMs

---

## ⚖️ 8. CRITICAL ANALYSIS

### Hidden Assumptions:
- **Assumption that interleaved comment-code patterns in GitHub are sufficient** supervision for multi-turn synthesis — but most GitHub comments are low quality
- **Assumes users can decompose problems** into good sub-steps — this is itself a non-trivial skill
- **Sequential training (NL → Multi → Mono)** assumes this ordering is optimal — no ablation on alternatives

### Weaknesses Authors DON'T Mention:
- **MTPB is author-created and small** (115 problems by 4 authors) — potential bias in problem selection and difficulty
- **No comparison with human decomposition quality** — the multi-turn prompts were written by experts; regular users might not decompose problems as effectively
- **Context window limitation** (2,048 tokens) severely limits how many turns and how much code can accumulate
- **No error recovery mechanism** — if the model generates wrong code at turn 2, all subsequent turns are doomed
- **Training data contamination** is not thoroughly discussed — some test problems might be similar to training data

### Is the Evaluation Fair?
- HumanEval comparison is fair — same benchmark and metrics as Codex
- MTPB comparison is **less fair** — the benchmark was designed by the same team, and the Codex models weren't specifically designed for multi-turn use
- Temperature selection (picking best across 3 temperatures) is standard but can be seen as optimistic

### Would This Work at Scale in the Real World?
- **Partially.** The models are open-source and usable, but:
  - Real programming tasks are much more complex than 115-problem benchmarks
  - 16.1B parameters still requires significant GPU resources for inference
  - Multi-turn requires a good user interface and user skill in decomposition
  - No handling of external dependencies, file systems, or real-world codebases

---

## 📝 9. MEMORY ANCHORS

### Memorable Metaphor:
**CodeGen is like a sous chef who can follow recipes step by step.** Instead of shouting the entire meal order at once ("Make me a 3-course dinner!"), you tell them one dish at a time ("First, make the soup" → "Now the salad" → "Finally the steak"). They cook better because they can focus on one thing at a time — and they remember what they already made.

### 3 Bullets That Capture 80%:
- 📦 **Open-source family of code LLMs** (350M to 16.1B params) trained sequentially on natural language → multi-language code → Python, **matching closed-source Codex** on HumanEval
- 🔄 **Multi-turn program synthesis** breaks complex coding tasks into conversational steps, **improving pass rates by ~10 percentage points** over single-turn
- 📊 **New benchmark (MTPB)** with 115 multi-turn problems shows that both **bigger models and more code data** systematically improve multi-turn coding ability

### Comprehension Test Question:
> *Why does multi-turn factorization improve program synthesis, and under what conditions does it NOT help?*

**Answer:** Multi-turn factorization reduces perplexity (the model understands each sub-step better than one long specification), effectively reducing the search space. However, for **easy problems with large models**, multi-turn provides no benefit (and can even slightly hurt) — because the model already understands the problem well enough in a single turn.

---

## 🗺️ 10. VISUAL MENTAL MAP

```
PROBLEM                          METHOD                              RESULT
═══════                          ══════                              ══════

Code generation is hard     ──►  SEQUENTIAL TRAINING:                
  • Huge search space             ┌─────────────┐                    
  • Hard to specify intent        │  ThePile     │ (825GB English)   
  • Best models closed-source     │  → NL model  │                   
                                  └──────┬──────┘                    
                                         │ initialize               
                                  ┌──────▼──────┐                    
                                  │  BigQuery   │ (6 languages)      
                                  │  → MULTI    │                    HumanEval:
                                  └──────┬──────┘                    CODEGEN-MONO 16.1B
                                         │ initialize               pass@1 = 29.28%
                                  ┌──────▼──────┐                    (≈ Codex 12B: 28.81%)
                                  │  BigPython  │ (Python only)      
                                  │  → MONO     │                    
                                  └──────┬──────┘                    
                                         │                           
                              ┌──────────▼──────────┐                
                              │  MULTI-TURN IDEA:   │               MTPB:
                              │                     │               Multi-turn: 47.34%
                              │  Human: "Step 1..." │               Single-turn: 38.74%
                              │  Model: [code 1]    │               (+10 pp improvement!)
                              │  Human: "Step 2..." │                
                              │  Model: [code 2]    │               
                              │  ...                │               Scaling works:
                              │  Execute & test     │               Bigger model = better
                              └─────────────────────┘               More code data = better
                                         │                           
                              ┌──────────▼──────────┐                
                              │  NEW BENCHMARK:     │               Open Source:
                              │  MTPB (115 problems)│               ✅ Models released
                              │  • 3+ turns each    │               ✅ Training lib released
                              │  • Expert-written   │               ✅ Benchmark released
                              │  • 5 test cases each│                
                              └─────────────────────┘                
```

---

## 🛠️ 11. IMPLEMENTATION SKETCH

### Pseudocode of Multi-Turn Generation:
```python
def multi_turn_synthesis(model, prompts, test_inputs, test_outputs):
    """Core multi-turn program synthesis loop"""
    
    for test_input in test_inputs:
        context = "# Import libraries.\nimport numpy as np\n"
        generated_code_parts = []
        
        for prompt in prompts:
            # Fill in test inputs into prompt template
            filled_prompt = prompt.format(**test_input)
            
            # Prepend as comment
            full_input = context + "# " + filled_prompt + "\n"
            
            # Generate code with nucleus sampling
            generated = model.sample(
                full_input, 
                top_p=0.95, 
                temperature=0.8,
                stop_tokens=["\n#", "\n\n"]  # stop at next comment
            )
            
            # Append to context for next turn
            context = full_input + generated + "\n"
            generated_code_parts.append(generated)
        
        # Execute concatenated program
        full_program = "\n".join(generated_code_parts)
        actual_output = execute_in_sandbox(full_program)
        
        # Check functional correctness
        if type_relaxed_equal(actual_output, test_outputs):
            return "PASS"
    
    return "FAIL"
```

### Frameworks/Libraries Needed:
- **JAX** + **Flax** (for TPU training — original implementation)
- **PyTorch** / **Hugging Face Transformers** (for using released checkpoints)
- **JAXFORMER** (custom library, open-sourced by authors)
- **TPU-v4** or **NVIDIA A100 GPUs** for training

### Estimated Compute Cost:
- **Training 16.1B model:** ~Hundreds of thousands of TPU-v4 hours (estimated $500K–$1M+ in cloud compute)
- **Inference:** ~1 A100 GPU for the 16.1B model (with model parallelism), smaller models fit on consumer GPUs
- **Reproducing MTPB evaluation:** A few hundred GPU-hours (40 samples × 5 test cases × 115 problems × multiple models)
