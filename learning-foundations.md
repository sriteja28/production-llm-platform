# AI/ML/LLM Foundations — Pre-Build Learning Path

A curated 3-4 week foundation for a senior systems engineer entering
AI infrastructure. Ruthlessly prioritized to build production-AI
engineering depth before writing any code.

This curriculum was assembled while building
[production-llm-platform](https://github.com/sriteja28/production-llm-platform)
— a multi-tenant LLM serving platform for reasoning-and-agent
workloads — but it stands alone for anyone entering this space.
Fork and adapt freely (MIT-licensed).

## Who this is for

You're a good fit if:

- You're an experienced backend / distributed-systems / SRE /
  platform engineer (5+ years) entering AI infrastructure as a new
  specialty.
- You don't need a from-scratch ML degree — you need the minimum
  viable foundation to read vLLM scheduler code, understand KV
  cache design, and reason about reasoning-model inference
  economics.
- You're targeting senior-IC roles in AI infrastructure (serving,
  scheduling, GPU economics) or AI engineering (production AI
  systems), and you need depth fluency, not a breadth survey.
- You learn by doing — every Tier 1 item has hands-on code, not
  just reading.

You're NOT a good fit if:

- You want a comprehensive ML curriculum (use Andrew Ng's Deep
  Learning Specialization or fast.ai instead).
- You're targeting AI/ML research roles (you'll need different math
  foundations).
- You're a junior engineer — this assumes 5+ years of production
  systems experience as the substrate.

## How to use this

1. **Day 0 first** — environment setup + a reasoning-model smoke
   test on Apple Silicon (or comparable local hardware). This makes
   the abstract concepts in Week 1 concrete.
2. **Tier 1** is the must-do 80% (~50 hours of work).
3. **Tier 2** is the next 15% (~15 hours).
4. **Tier 3** is reference material — read when relevant, not now.
5. **Self-test** at the end (12 questions). 10/12 passes; 6-7/12
   means repeat Weeks 2-3 before starting any build.
6. **Parallel community actions** — run these alongside the reading,
   not after. They compound while you learn.
7. **Fork and adapt.** If your target role, hardware (no Apple
   Silicon), or cloud provider differs, modify the commands and
   reading list. This is a starting point, not gospel.

Estimated time: ~50-70 hours at full-time pace, 3-4 weeks.

If you finish this curriculum and want to see what a senior-level
build using it looks like, the
[production-llm-platform](https://github.com/sriteja28/production-llm-platform)
repo contains the 12-milestone arc that takes the same person from
"foundations passed" to a flagship benchmark on cost-per-correct-
answer for reasoning + agent workloads.

---

## Operating Principles

- This is the **minimum viable foundation**, not a comprehensive curriculum
- Type out code yourself — do not just watch passively
- Skip anything not on this list unless you have a specific reason
- Target completion: 3-4 weeks at full-time pace, then start Milestone 1
- Total time investment: ~50-70 hours

---

## Day 0 — Environment setup and reasoning-model smoke test

Before starting Tier 1 reading, set up the dev environment you'll
use for the full 4-week curriculum and all 12 milestones. Plan ~90
minutes total. Doing this on Day 0 makes the abstract concepts in
Week 1 concrete by giving you a working reasoning model on your own
laptop.

### Setup checklist (60 min)

```bash
# 1. Verify Python 3.11+
python3 --version
# If you don't have 3.11+:
# brew install python@3.11

# 2. Create the project venv (kept outside the repo)
mkdir -p ~/.venv
python3.11 -m venv ~/.venv/llm-platform
source ~/.venv/llm-platform/bin/activate

# 3. Install MLX-LM (the local reasoning-model backend on M4 Max)
pip install --upgrade pip
pip install mlx-lm jupyter

# 4. Verify Apple Silicon Metal backend is active
python -c "import mlx.core as mx; print('Metal OK:', mx.metal.is_available())"
# Expected: Metal OK: True
```

If `Metal OK` is False, MLX won't use the GPU — debug before
continuing. Most common cause: running under x86 Rosetta. Verify
with `uname -m` (should be `arm64`).

### First reasoning model smoke test (30 min)

This is the "I generated a reasoning trace on my laptop" milestone.
Worth doing before any reading — it makes the abstract concepts
concrete.

```bash
# Downloads ~4.5 GB on first run
python -c "
from mlx_lm import load, generate
model, tokenizer = load('mlx-community/DeepSeek-R1-Distill-Llama-8B-4bit')

prompt = 'A factory makes 12 widgets per hour for 8 hours a day. How many widgets in a 5-day workweek? Think step by step.'
response = generate(model, tokenizer, prompt=prompt, max_tokens=2000, verbose=True)
print('---')
print(response)
"
```

What you'll see: the model emits a `<think>...</think>` block (the
reasoning trace), then the final answer. Note the ratio of think-
block tokens to final-answer tokens — that variable trace length is
the workload your platform's scheduler in Milestone 2 has to handle
gracefully.

### Day 0 success criteria (must all be ✅ before Tier 1)

- [ ] Python 3.11+ venv with MLX-LM working on Metal
- [ ] Generated a reasoning trace from DeepSeek-R1-Distill-Llama-8B
      on M4 Max
- [ ] Modal account created with $50 credit (sign up at modal.com,
      run `pip install modal && modal token new`)
- [ ] vLLM Discord joined for lurking (https://discord.gg/jz7wjKhh6g)

---

## Tier 1 — MUST DO (covers ~80% of the foundation)

### Andrej Karpathy — Neural Networks: Zero to Hero (YouTube, free)

The single best AI/ML resource ever made for engineers. Karpathy explains
everything from first principles, builds it in PyTorch, and assumes
you're a smart engineer not an ML PhD.

Watch in this exact order:

1. **Micrograd** — backpropagation from scratch (~2 hrs)
2. **Makemore Part 1** — bigram language model (~2 hrs)
3. **Makemore Part 2** — MLP language model (~1.5 hrs)
4. **Makemore Part 3** — activations, gradients, batch norm (~2 hrs)
5. **Makemore Part 4** — backprop ninja (~2 hrs, can skim)
6. **Makemore Part 5** — building a WaveNet (~2 hrs)
7. **Let's build GPT** — building a transformer from scratch (~2 hrs) ★
8. **Let's build the GPT Tokenizer** — tokenization deep dive (~2 hrs) ★
9. **Let's reproduce GPT-2** — full pretraining pipeline (~4 hrs,
   ambitious)

★ = highest priority videos for AI infrastructure work

**Time investment:** 40-50 hours including hands-on work.
**Goal:** Be able to whiteboard the forward pass of a decoder-only
transformer, including attention, MLP, residuals, and layer norm, from
memory.

URL: https://www.youtube.com/@AndrejKarpathy

### 3Blue1Brown — Neural Networks series (YouTube, free)

Visual intuition for what neural networks actually do. Watch BEFORE
Karpathy's series — the visual mental models make everything click
faster when you hit the code.

Six videos, ~3 hours total:
1. But what is a neural network?
2. Gradient descent, how neural networks learn
3. What is backpropagation really doing?
4. Backpropagation calculus
5. But what is a GPT? Visual intro to transformers
6. Attention in transformers, visually explained

URL: https://www.youtube.com/@3blue1brown

### The Illustrated Transformer — Jay Alammar (blog post)

The single best written explanation of how transformers work. Read it
3 times over a week. Each read unlocks something new.

URL: https://jalammar.github.io/illustrated-transformer/

### The Illustrated GPT-2 — Jay Alammar (blog post)

Companion piece, specifically about decoder-only models (which is what
every modern LLM is).

URL: https://jalammar.github.io/illustrated-gpt2/

### DeepSeek-R1 paper

The defining 2025 paper on training reasoning models via RL on
chain-of-thought. Reasoning-and-agent inference economics is the
project's technical anchor — this paper is the prerequisite.

Read Section 2 (training pipeline) and Section 3 (R1-Zero RL setup)
deeply. Section 4 (R1-Distill variants) explains why we use
DeepSeek-R1-Distill-Llama-8B for local dev — these are the models
that fit on an M4 Max and are MLX-LM compatible.

Title: "DeepSeek-R1: Incentivizing Reasoning Capability in LLMs via
Reinforcement Learning"
URL: https://arxiv.org/abs/2501.12948

### Qwen3 thinking mode documentation

Qwen3 introduced switchable thinking mode (via `/think` and
`/no_think` prompt directives). You'll need to understand this for M1
benchmark fairness (mixing thinking-on and thinking-off requests
realistically) and for M2 budget enforcement (capping thinking
tokens per tier).

URL: https://qwen.readthedocs.io/en/latest/inference/chat.html
(skim sections on thinking mode and `enable_thinking`)

---

## Tier 2 — SHOULD DO (the next 15%)

### Karpathy — State of GPT (YouTube talk, ~40 min)

The cleanest mental model of how modern LLMs are trained: pretraining →
SFT → RLHF. You'll reference this framework constantly in interviews.

Search "Andrej Karpathy State of GPT" on YouTube.

### Simon Boehm — How to Optimize a CUDA Matmul Kernel (blog post)

The single best resource for understanding GPU performance from a
software engineer's perspective. Don't try to write CUDA kernels — just
read this to understand what makes them fast or slow. Read it three
times.

URL: https://siboehm.com/articles/22/CUDA-MMM

### PagedAttention paper (vLLM original paper)

Read after Karpathy's "Let's build GPT." This paper introduces how vLLM
solves KV cache fragmentation. Foundational reading for AI infra roles.
Section 3 (PagedAttention) and Section 4 (system design) are the
critical parts.

Title: "Efficient Memory Management for Large Language Model Serving
with PagedAttention"
URL: https://arxiv.org/abs/2309.06180

### FlashAttention paper

Read for *intuition only* — what makes attention slow, why memory access
patterns matter. Skip the math; the high-level message about IO-aware
algorithms is what matters.

Title: "FlashAttention: Fast and Memory-Efficient Exact Attention with
IO-Awareness"
URL: https://arxiv.org/abs/2205.14135

### Hugging Face NLP Course (free, online)

Skim chapters 1-3 only. Not deep — geared toward fine-tuning more than
infrastructure. But you should know what tokenizers, datasets, and the
Transformers library look like in practice.

URL: https://huggingface.co/learn/nlp-course

### Anthropic — Building Effective Agents

The canonical guide to agent system design. Defines the vocabulary
(workflows, agents, tools, augmented LLMs) used in modern agent
implementations.

URL: https://www.anthropic.com/research/building-effective-agents

### OWASP Top 10 for LLM Applications

Industry standard for LLM security categories. Required vocabulary
for any production LLM work — prompt injection, model DoS, training
data poisoning, output leakage, etc.

URL: https://owasp.org/www-project-top-10-for-large-language-model-applications/

### vLLM reasoning_outputs documentation

How vLLM serves reasoning models in production: `--reasoning-parser`
flag, supported parsers (`deepseek_r1`, `qwen3`, `granite`), thinking
budget control, streaming reasoning vs final-answer separation, and
the constraint that reasoning is **online-serving only** (chat
completions endpoint, not raw `/v1/completions`).

This determines how the M1 benchmark harness has to be structured —
the harness must use the OpenAI-compatible chat client, not raw
generate calls.

URL: https://docs.vllm.ai/en/latest/features/reasoning_outputs/

### DeepSeek DualPath paper (Feb 2026)

The 2026 paper that establishes KV-cache I/O — not compute — as the
bottleneck in long-horizon agent workloads. Foundational reading for
M6 (agent orchestration) and M3 (KV cache hierarchy). The DualPath
storage-to-decode loading approach is the architectural pattern M6
will fold in.

Skim Section 3 (the KV-I/O analysis) and Section 4 (the dual-path
mechanism). Don't try to reproduce; read for the architectural
intuition.

URL: https://arxiv.org/abs/2602.21548

---

## Tier 3 — REFERENCE MATERIAL (read when relevant, not now)

### Attention Is All You Need (the original transformer paper)

Skim only. Read Section 3 for the architecture. The illustrated blog
posts above are clearer than the paper for first exposure.

URL: https://arxiv.org/abs/1706.03762

### Language Models are Few-Shot Learners (GPT-3 paper)

Skim. Important historically. Don't memorize.

URL: https://arxiv.org/abs/2005.14165

### Training language models to follow instructions (InstructGPT/RLHF)

Skim. Understand the concept of RLHF, don't memorize details.

URL: https://arxiv.org/abs/2203.02155

### Anthropic published research

Read 2-3 papers that interest you, especially on interpretability and
Constitutional AI.

URL: https://www.anthropic.com/research

### Multimodal AI vocabulary

You don't need depth here, just vocabulary. Skim:
- One LayoutLM paper abstract (document AI)
- One blog post on vision-language models (e.g., LLaVA)
- One blog post on voice AI architecture (e.g., Deepgram Nova or
  Whisper)

Enough to discuss in interviews if it comes up.

### Stanford CS25 — Transformers United (YouTube)

Pick 4-5 lectures based on topics you want depth on. Graduate-level
but accessible. Particularly useful: original transformer lecture, GPT
lecture, inference optimization lecture.

URL: https://web.stanford.edu/class/cs25/

---

## What to DELIBERATELY SKIP

These are common recommendations that are wrong for your specific goal
and background. Skip them.

- **Andrew Ng's Deep Learning Specialization (Coursera)** — too slow,
  too foundational, designed for people without your engineering
  background. Will bore you and waste 3-4 weeks.
- **Fast.ai courses** — excellent for ML engineers wanting to do model
  training. Wrong audience for AI infrastructure.
- **Anything called "ML for Beginners" or "AI Bootcamp"** — wrong
  altitude for a senior IC.
- **Mathematics-heavy resources (Bishop's textbook, Goodfellow's Deep
  Learning book)** — necessary for ML research, unnecessary for
  infrastructure work.
- **Most YouTube AI tutorials** — almost all are surface-level wrappers
  on OpenAI APIs. Karpathy is the exception that proves the rule.
- **Computer vision deep specialization** — unless your target role
  specifically requires it (most AI infra/AI engineer roles do not).

---

## 4-Week Schedule (Full-Time Pace)

### Week 1: Visual + Conceptual Foundation
- 3Blue1Brown Neural Networks series (3 hrs)
- Karpathy Micrograd + Makemore Parts 1-2 (5-6 hrs hands-on)
- Read Illustrated Transformer twice (2 hrs)
- **Goal end of week:** understand what a neural network does and how
  backprop works at a code level

### Week 2: Transformers Deep
- Karpathy Makemore Parts 3-5 (5-6 hrs)
- Karpathy "Let's build GPT" with full code-along (4-5 hrs)
- Read Illustrated GPT-2 (1 hr)
- Karpathy "Let's build the GPT Tokenizer" (3 hrs)
- **Goal end of week:** can implement attention from scratch in PyTorch
  in an afternoon

### Week 3: Inference Internals + Production Reality
- Karpathy "Let's reproduce GPT-2" (skim or full, 2-4 hrs)
- Karpathy "State of GPT" talk (1 hr)
- PagedAttention paper, deep read (2-3 hrs)
- FlashAttention paper for intuition (1 hr)
- Simon Boehm CUDA matmul blog post — read 3 times (3-4 hrs)
- Anthropic "Building effective agents" blog post (1 hr)
- OWASP Top 10 for LLM Applications (1 hr)
- **Goal end of week:** understand why inference is memory-bound, what
  KV cache costs in VRAM, why batching matters, agent system anatomy

### Week 4: Hands-On End-to-End (reasoning-aware)
- Hugging Face NLP Course chapters 1-3 (3-4 hrs)
- Run DeepSeek-R1-Distill-Llama-8B (4-bit MLX) end-to-end on M4 Max
  via mlx_lm — generate reasoning traces, observe trace length
  variance across questions (3 hrs)
- Run the same model with vLLM `--reasoning-parser deepseek_r1` on
  rented A100, benchmark thinking vs non-thinking throughput
  (2 hrs, ~$10 cloud cost). Use the OpenAI-compatible chat
  completions endpoint (not raw `/completions` — vLLM reasoning is
  online-only).
- Read the DualPath paper Section 3 for KV-I/O intuition (1 hr)
- Write a 500-1000 word notes file (NOT yet a public blog post —
  see `_private/docs/publishing-plan.md`) explaining what you learned
  about reasoning trace variance and why it breaks naive batching
  (2-3 hrs)
- **Goal end of week:** ready to start Milestone 1

---

## Self-Test Before Starting Milestone 1

You're ready to start when you can answer these questions without
looking anything up:

1. What does the forward pass of a decoder-only transformer compute,
   layer by layer?
2. What is a KV cache, why does it exist, and what does it cost in VRAM?
3. Why is LLM inference memory-bound, not compute-bound?
4. What is tokenization, why does it matter, and how do BPE tokenizers
   work?
5. What's the difference between pretraining, supervised fine-tuning,
   and RLHF?
6. What does PagedAttention solve and why is it called "paged"?
7. Why does batching dramatically increase GPU utilization?
8. What's the difference between greedy decoding, sampling, and what
   does temperature control?
9. What's the basic anatomy of an LLM agent (loop, tools, state,
   termination)?
10. What's prompt injection and why is "input sanitization alone"
    insufficient?
11. What's the difference between training infra and inference infra,
    and which one are you targeting and why?
12. How do reasoning models (o1/o3/extended thinking) differ from
    standard inference economically?

If you can answer 10/12 confidently, you're ready. If you can only
answer 6-7/12, repeat Week 2 and Week 3 before starting Milestone 1.

---

## Parallel community actions (start Week 1, not after Milestone 12)

These are visibility multipliers — they compound while you learn,
and at the senior-IC ($400K+) hire tier the project alone is
necessary but not sufficient. Run these in parallel with foundations
and the milestone build:

- **vLLM Discord:** lurk 10-15 min/day in `#general`, `#help`, and
  `#announcements`. By Week 3 you should know the regulars and the
  active topics. Don't post yet — observe and learn the terminology.
  By Milestone 2 (~week 6-7), start asking one well-formed question
  per week and answering easier ones. By M4-M6, be a recognizable
  handle.
- **Modal account active from Day 0** so the CLI and credit are
  ready when M1 cloud sessions start.
- **Identify one upstream-contribution candidate during Tier 2
  reading** (vLLM, SGLang, MLX-LM, or llm-d). Anything counts:
  documentation typo, error-message clarification, missing example,
  reproducible bug report. File during M1-M2.

What to **NOT do during foundations:**
- No Hashnode publication setup yet (gates on M1 readiness)
- No blog posts (first post is the M1 report)
- No applying to roles yet (apply at the M4 gate per the strategy
  in `_private/docs/job-hunt-positioning.md`)

---

## Cost Estimate

- Karpathy videos: free
- 3Blue1Brown videos: free
- Blog posts and papers: free
- Hugging Face NLP Course: free
- GPU rentals for Week 4 hands-on: ~$5-15 on Modal (small model, single
  GPU, ~3-4 hours of runtime)

**Total foundation cost: under $20 in cloud credits.**

---

## Honest Note on Pacing

If after Week 2 you feel shaky on transformers and attention, take an
extra week. Building the platform on confused fundamentals is worse than
delaying the build by a week. The signal that you're ready isn't "I
understand everything perfectly" — it's "I can read vLLM's scheduler
code and follow what it's doing without getting lost." When you hit
that bar, start Milestone 1.
