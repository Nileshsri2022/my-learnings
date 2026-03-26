

# InstructGPT: Training Language Models to Follow Instructions with Human Feedback

---

## 🎯 1. THE ONE-LINER
**They taught an AI to actually listen to what people want by having humans grade its answers, making a small model that people prefer over one 100x bigger.**

---

## 🤔 2. THE PROBLEM (Why does this paper exist?)

- **The problem:** Large language models (like GPT-3) are trained to predict the next word on the internet, NOT to follow your instructions. This means they often **make up facts, say toxic things, or completely ignore what you asked**.
- **Why should you care?** Imagine hiring a super-smart employee who memorized the entire internet but won't listen to you. You ask them to write a polite email, and they write a Wikipedia article instead. That's GPT-3. The **training objective** ("predict the next word") is fundamentally different from what users want ("help me with this task").
- **Analogy:** It's like training a dog by having it watch YouTube videos of dogs. It would learn a LOT about dogs—but it wouldn't know how to sit, stay, or come when called. You need to actually **train it to follow commands**.
- **Limitations of previous approaches:**
  - Simply making models **bigger** doesn't fix alignment—they just become bigger models that still don't follow instructions
  - Previous RLHF work (Stiennon et al., 2020) only focused on **summarization**, not general instruction-following
  - Fine-tuning on public NLP datasets (FLAN, T0) doesn't capture what real users actually want—only ~18% of real API usage is classification/QA, while ~57% is open-ended generation
  - Prompt engineering is fragile and unreliable

---

## 💡 3. THE KEY IDEA (The "Aha!" moment)

**Core insight:** Instead of training a model on "what comes next on the internet," train it on "what humans actually prefer as a response," using a **3-step recipe**: show examples → learn what's good → practice getting better.

**Cooking analogy:** 
- **Step 1 (SFT):** A chef watches a master chef prepare dishes (supervised learning from demonstrations)
- **Step 2 (Reward Model):** A food critic tastes multiple dishes and ranks them (learning human preferences)
- **Step 3 (PPO):** The chef keeps cooking, and the critic keeps scoring, so the chef gets better and better (reinforcement learning)

**The trick that makes it efficient:** Instead of having humans rank just 2 outputs at a time, they show 4-9 outputs and get a full ranking—generating **many pairwise comparisons from one labeling session**.

```
THE 3-STEP PIPELINE:
                                                    
Step 1: SFT          Step 2: Reward Model    Step 3: RL (PPO)
                                                    
[Prompt] ──→          [Prompt] ──→            [Prompt] ──→
[Human writes         [Model generates        [Model generates
 ideal answer]         4-9 answers]            an answer]
     │                     │                       │
     ▼                     ▼                       ▼
[Fine-tune           [Human ranks             [Reward Model
 GPT-3 on these]      answers]                 scores it]
                          │                       │
                          ▼                       ▼
                     [Train Reward            [PPO updates
                      Model to                 model to get
                      predict rankings]        higher scores]
                                                    
                     ◄── Steps 2 & 3 can iterate ──►
```

---

## 🏗️ 4. HOW IT WORKS (The Method - Layer by Layer)

### Step 1: Collect Demonstrations & Train SFT Model
- **WHAT:** Hire ~40 human labelers. Give them prompts (from API users + labeler-written). They write **ideal responses**. Fine-tune GPT-3 on these ~13K prompt-response pairs using standard supervised learning.
- **WHY:** Creates a starting point model that already understands the *format* of good responses. Without this, the model has no concept of what "following instructions" looks like.
- **HOW it connects:** This SFT model becomes the **starting point** for the next steps AND the **baseline** to prevent RL from going off the rails.
- **Details:** Trained 16 epochs, cosine LR decay, 0.2 dropout. Selected based on RM score (not validation loss).

### Step 2: Collect Comparisons & Train Reward Model (RM)
- **WHAT:** For each prompt, generate **4-9 model outputs**. Have labelers **rank** them from best to worst. Train a 6B parameter model to predict which output humans would prefer.
- **WHY:** It's **much easier for humans to rank outputs than write perfect ones**. This is the key leverage: comparisons are cheaper than demonstrations. The RM becomes an automated "human preference predictor."
- **HOW it connects:** The RM becomes the **scoring function** for Step 3. It takes (prompt, response) → scalar reward.
- **Key innovation:** Training on all K-choose-2 pairs from each prompt **as a single batch** prevents overfitting and is computationally efficient.
- **Loss function:** Bradley-Terry model: `loss = -log(σ(r(x, y_w) - r(x, y_l)))` — the reward difference between winner and loser should be large.

### Step 3: Optimize Policy with PPO
- **WHAT:** Use the RM as a reward signal. Fine-tune the SFT model using Proximal Policy Optimization (PPO) to generate responses that score high on the RM.
- **WHY:** This is where the model actually **learns to produce better outputs**, not just mimic demonstrations. RL allows exploration beyond the demonstration data.
- **Key safeguard:** Add a **KL penalty** between the PPO policy and the SFT model, preventing the model from "gaming" the reward model by producing weird outputs that score high but aren't actually good.
- **PPO-ptx variant:** Mix in some pretraining data during PPO to prevent **forgetting** general capabilities (the "alignment tax" fix).

**The PPO-ptx objective:**
```
objective = E[reward(x,y) - β·KL(πRL || πSFT)] + γ·E[log πRL(pretraining data)]
               ↑ maximize reward    ↑ stay close to SFT    ↑ don't forget language skills
```

---

## 📊 5. THE PROOF (Results & Experiments)

### Benchmarks/Datasets:
- **Main evaluation:** Human preference ratings on held-out API prompts
- **Safety:** TruthfulQA, RealToxicityPrompts, Winogender, CrowS-Pairs
- **NLP capability:** SQuAD, DROP, HellaSwag, WMT translation, SST, RTE, WSC, CNN/DM, TLDR

### Key Numbers:

| Metric | Result |
|--------|--------|
| **1.3B InstructGPT vs 175B GPT-3** | **1.3B InstructGPT preferred** (despite 100x fewer params!) |
| 175B InstructGPT vs 175B GPT-3 | Preferred **85 ± 3%** of the time |
| 175B InstructGPT vs few-shot GPT-3 | Preferred **71 ± 4%** of the time |
| TruthfulQA (truthful + informative) | InstructGPT **~2x better** than GPT-3 |
| Hallucination rate (closed-domain) | InstructGPT **21%** vs GPT-3 **41%** |
| Toxic output reduction (with respectful prompt) | **~25% fewer** toxic outputs |
| InstructGPT vs FLAN | Preferred **78 ± 4%** of the time |
| InstructGPT vs T0 | Preferred **79 ± 4%** of the time |
| Training cost (SFT 175B) | **4.9 petaflop/s-days** (vs 3,640 for GPT-3 pretraining) |
| Training cost (PPO-ptx 175B) | **60 petaflop/s-days** (~1.6% of pretraining cost) |

### Most Impressive Result (Plain English):
**A model with 1.3 billion parameters, after RLHF training, was preferred by humans over a vanilla model with 175 billion parameters.** The alignment procedure made a small model more useful than one 100x larger.

### Admitted Failure Cases / Limitations:
- Still **makes up facts** and **fails to follow complex multi-constraint instructions**
- **Assumes false premises** when given trick questions (e.g., "Why is it important to eat socks after meditating?")
- **Over-hedges** on simple questions
- **Does NOT improve bias** (Winogender, CrowS-Pairs scores similar to GPT-3)
- **Performance regressions** on some NLP benchmarks (DROP, SQuAD, translation) — partially mitigated by PPO-ptx
- When explicitly prompted to be toxic, InstructGPT produces **MORE toxic** content than GPT-3

---

## 🧩 6. KEY TERMS GLOSSARY

- **Alignment** → Making an AI system do what humans actually want, not just what its training objective says
- **RLHF (Reinforcement Learning from Human Feedback)** → Training method where human preferences serve as the reward signal
- **SFT (Supervised Fine-Tuning)** → Standard training where model learns from input-output examples
- **Reward Model (RM)** → A model that learns to predict which outputs humans would prefer
- **PPO (Proximal Policy Optimization)** → An RL algorithm that updates the model's behavior in small, stable steps
- **PPO-ptx** → PPO variant that mixes in pretraining data to prevent capability loss
- **KL Divergence / KL Penalty** → A measure of how different two probability distributions are; used to keep the RL model from drifting too far from SFT
- **Alignment Tax** → Performance loss on other tasks caused by alignment training
- **Hallucination** → When a model makes up information that isn't true or isn't in the input
- **Labeler/Contractor** → Human workers hired to write demonstrations and rank model outputs
- **Bandit Environment** → RL setup where each prompt is a one-step episode (no multi-turn interaction)
- **Misaligned** → When a model's training objective differs from what users actually want
- **Likert Scale** → A rating scale (1-7 here) used to evaluate quality
- **PII** → Personally Identifiable Information (filtered from training data)
- **Inter-annotator Agreement** → How often human labelers agree with each other (~73%)
- **Few-shot Prompting** → Giving a model examples of desired behavior in the input prompt

---

## 🔗 7. HOW IT CONNECTS

### Intellectual Family Tree:
```
Christiano et al. 2017          Radford et al. 2019 (GPT-2)
(RLHF for Atari/robots)         Brown et al. 2020 (GPT-3)
         │                              │
         ▼                              ▼
Ziegler et al. 2019 ──────────────────────►
(RLHF for text style)                    │
         │                              │
         ▼                              │
Stiennon et al. 2020 ─────────────────────►
(RLHF for summarization)                │
         │                              │
         ▼                              ▼
    ┌──────────────────────────────────────┐
    │    InstructGPT (THIS PAPER)          │
    │    RLHF for general instructions     │
    └──────────────────────────────────────┘
         │
         ▼
    ChatGPT, GPT-4, Claude, Llama-2-Chat
    (all use variants of RLHF)
```

### Related Contemporary Work:
- **FLAN** (Wei et al., 2021) — Instruction-tuning on NLP tasks (different approach: no human feedback, just task instructions)
- **Anthropic's work** (Askell et al., 2021) — Language assistants as alignment testbed (concurrent work)
- **WebGPT** (Nakano et al., 2021) — RLHF for web-browsing QA

### Who Would Use This & For What:
- **AI companies** building chatbots and assistants (this IS the recipe behind ChatGPT)
- **Alignment researchers** studying how to make AI safe
- **Anyone deploying LLMs** in production who needs reliable instruction-following

### Future Work Enabled:
- **ChatGPT** (directly descended from this work)
- **Constitutional AI** (Anthropic's extension replacing human labelers with AI feedback)
- **DPO** (Direct Preference Optimization — removing the RM step entirely)
- **RLHF at scale** for multimodal models (GPT-4V, etc.)

---

## ⚖️ 8. CRITICAL ANALYSIS

### Hidden Assumptions:
- **Assumes labeler preferences = user preferences = "good"** — but labelers aren't the end users, and labeler demographics are narrow (mostly young, US/SE Asia, English-speaking)
- **Assumes prompts submitted to the API represent general use** — but early API users were biased toward OpenAI's network
- **Assumes a single reward model can capture diverse human values** — but inter-labeler agreement is only ~73%

### Weaknesses the Authors DON'T Emphasize:
- **Only ~40 labelers** — extremely small and non-representative group shaping the model's behavior for millions of users
- **Helpfulness prioritized over safety in training** (they switched to truthfulness/harmlessness only for evaluation — inconsistent)
- The model becomes **MORE dangerous** when given explicit instructions to be harmful (it's better at following ALL instructions, including bad ones)
- **Most comparisons labeled by only 1 contractor** — no reliability check
- **English-centric** (~96%+ English data) but deployed globally
- The **reward model is a proxy** — optimizing it too hard leads to "reward hacking" (Goodhart's Law)

### Is the Evaluation Fair?
- **Somewhat circular:** The prompts come from InstructGPT users, naturally favoring instruction-following models over GPT-3. They acknowledge this and also test on GPT-3 prompts (results still hold).
- **Human evaluation is the gold standard** — they use it extensively, which is good
- **Automatic metrics on NLP benchmarks** provide useful grounding but don't capture real-world usage

### Would This Work at Scale?
- **YES** — It already did. ChatGPT is essentially a scaled version of this approach.
- **Cost is remarkably low** — training is <2% of pretraining cost
- **BUT:** The labeler bottleneck limits iteration speed, and labeler quality/bias remains a fundamental concern
- Ongoing data collection is needed as usage patterns evolve

---

## 📝 9. MEMORY ANCHORS

### Memorable Metaphor:
**InstructGPT is like putting a smart but unruly employee through corporate training.** The employee (GPT-3) already has enormous knowledge (pretraining), but needs to learn to actually listen to the boss (SFT), get feedback from a manager (RM), and practice until they consistently perform well (PPO). The result: a junior employee who's better at their job than a senior employee who never got trained.

### 3 Bullet Points (80% of the Paper):
- **A 3-step process (SFT → Reward Model → PPO) makes language models follow instructions**, dramatically improving helpfulness, truthfulness, and reducing (but not eliminating) toxicity
- **Human preferences beat scale:** A 1.3B RLHF model is preferred over a 175B vanilla model, at <2% of pretraining cost
- **The alignment tax is real but manageable:** Mixing pretraining data into PPO training (PPO-ptx) mostly preserves capabilities on standard benchmarks

### Comprehension Test Question:
> *Why does simply training on larger datasets or making models bigger NOT solve the alignment problem, and how does the 3-step RLHF process address this fundamental gap?*

---

## 🗺️ 10. VISUAL MENTAL MAP

```
PROBLEM                          METHOD                              RESULT
═══════                         ═══════                             ═══════

GPT-3 trained to               STEP 1: SFT                         
"predict next word"  ──►        ┌─────────────┐                     
                                │ Labelers     │                     
But users want                  │ write ideal  │──► Fine-tune        
"follow my                      │ responses    │    GPT-3            
 instructions"                  │ (~13K demos) │                     
                                └─────────────┘                     
     │                               │                              
     │ MISALIGNMENT!                 ▼                              
     │                          STEP 2: RM                          
     │                          ┌─────────────┐                     
     │                          │ Generate 4-9 │                     
     │                          │ outputs, have│──► Train 6B         
     │                          │ humans rank  │    reward model     
     │                          │ (~33K prompts│                     
     │                          └─────────────┘                     
     │                               │                              
     │                               ▼                              
     │                          STEP 3: PPO                         
     │                          ┌─────────────┐                     
     │                          │ RM scores    │    ┌──────────────┐
     │                          │ model outputs│──►│ InstructGPT  │
     │                          │ PPO updates  │    │              │
     │                          │ + KL penalty │    │ 1.3B beats   │
     │                          │ + pretrain   │    │ 175B GPT-3!  │
     │                          │   data mix   │    │              │
     │                          └─────────────┘    │ 85% win rate │
     │                                              │ 2x truthful  │
     ▼                                              │ 25% less     │
  "Making models                                    │ toxic        │
   bigger doesn't                                   └──────────────┘
   fix this"                                              │
                                                          ▼
                                                    LIMITATIONS:
                                                    • Still hallucinates
                                                    • Follows harmful instructions
                                                    • Doesn't fix bias
                                                    • Small labeler pool
                                                    • Performance regressions
```

---

## 🛠️ 11. IMPLEMENTATION SKETCH

### Pseudocode (~20 lines):
```python
# STEP 1: Supervised Fine-Tuning
sft_model = finetune(gpt3, demonstration_data, epochs=16, dropout=0.2)

# STEP 2: Reward Model Training
rm_model = init_from(sft_model, remove_unembedding=True, add_scalar_head=True)
for batch in comparison_data:
    prompt, rankings = batch  # K outputs ranked per prompt
    for (y_w, y_l) in all_pairs(rankings):  # K-choose-2 pairs
        r_w = rm_model(prompt, y_w)
        r_l = rm_model(prompt, y_l)
        loss += -log(sigmoid(r_w - r_l))
    loss /= num_pairs
    update(rm_model, loss)

# STEP 3: PPO with Pretraining Mix
policy = copy(sft_model)  # Initialize from SFT
for episode in range(256_000):
    prompt = sample(ppo_prompts)
    response = policy.generate(prompt)
    reward = rm_model(prompt, response)
    kl_penalty = β * KL(policy(response|prompt), sft_model(response|prompt))
    total_reward = reward - kl_penalty
    ppo_update(policy, total_reward)
    
    # PPO-ptx: mix in pretraining
    pretrain_batch = sample(pretrain_data)
    pretrain_loss = -γ * log_likelihood(policy, pretrain_batch)
    update(policy, pretrain_loss)
```

### Frameworks/Libraries Needed:
- **PyTorch** (model implementation)
- **Custom PPO implementation** (or trl library for modern reproduction)
- **GPT-3 architecture** (transformer with BPE tokenizer)
- **Human annotation platform** (custom web interface — Upwork/ScaleAI for labelers)
- **Perspective API** (for toxicity evaluation)

### Estimated Compute to Reproduce:
- **SFT 175B:** ~4.9 petaflop/s-days
- **PPO-ptx 175B:** ~60 petaflop/s-days
- **Total:** ~65 petaflop/s-days ≈ **~$100K-$500K** in cloud compute (2022 prices)
- **Human labeling costs:** ~40 contractors over months → estimated **$200K-$500K**
- **Compare:** GPT-3 pretraining was 3,640 petaflop/s-days → **RLHF is <2% of pretraining cost**
