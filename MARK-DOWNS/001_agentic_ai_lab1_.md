# Agentic AI — Lab 1 Master Study Guide
### "Your First LLM API Calls": Environment, Chat Completions, Model Chaining, and LLM-as-Judge

> **Source notebook:** `1_lab1.ipynb` (Agentic AI course, Lab 1)
> **What the notebook really teaches:** how to talk to an LLM programmatically, and how to chain several LLM calls so that the output of one becomes the input of the next. That second idea, *composition of LLM calls*, is the seed from which every workflow and agent architecture grows.

---

## Table of Contents

0. [How to Use This Guide](#0-how-to-use-this-guide)
1. [The Big Picture](#1-the-big-picture)
2. [Cell-by-Cell Map of the Notebook](#2-cell-by-cell-map-of-the-notebook)
3. [Part A: Environment and Secrets](#3-part-a-environment-and-secrets)
4. [Part B: LLM Fundamentals You Need First](#4-part-b-llm-fundamentals-you-need-first)
5. [Part C: The Chat Completions API](#5-part-c-the-chat-completions-api)
6. [Part D: Detailed Code Walkthrough](#6-part-d-detailed-code-walkthrough)
7. [Part E: Chaining, Model Tiering, and LLM-as-Judge](#7-part-e-chaining-model-tiering-and-llm-as-judge)
8. [Part F: From Chained Calls to Agentic Systems](#8-part-f-from-chained-calls-to-agentic-systems)
9. [Part G: The Exercise, Fully Solved](#9-part-g-the-exercise-fully-solved)
10. [Part H: Common Errors and Debugging](#10-part-h-common-errors-and-debugging)
11. [Part I: Extensions Beyond the Notebook](#11-part-i-extensions-beyond-the-notebook)
12. [Comparison Tables](#12-comparison-tables)
13. [Exam Preparation](#13-exam-preparation)
14. [One-Page Cheat Sheet](#14-one-page-cheat-sheet)

---

## 0. How to Use This Guide

| Goal | Read |
|---|---|
| Understand the notebook quickly | §1, §2 |
| Fix your setup | §3, §10 |
| Build deep understanding of what happens inside the API call | §4, §5 |
| Be able to re-implement the notebook from memory | §6, §9 |
| Understand why this matters for agents | §7, §8 |
| Prepare for exams or interviews | §13, §14 |

**Learning objectives.** After this guide you should be able to:

1. Explain what an LLM does when you call it (token prediction, sampling).
2. Explain why chat APIs are **stateless** and what that implies for memory.
3. Write the full pipeline *load key → create client → build messages → call → read result* without looking.
4. Build a multi-call chain (generator → solver → judge) and explain each design choice.
5. Distinguish a **workflow** from an **agent**, and place this notebook on that spectrum.
6. Debug the classic failures (`False` from `load_dotenv`, `NameError`, missing key, wrong kernel).

---

## 1. The Big Picture

### 1.1 What is being built

The notebook performs three escalating steps:

```
Step 1: "Tell me a fun fact"
        user ──► LLM(nano) ──► text                          (a single call)

Step 2: Chain of three calls
        LLM(mini): "write a hard IQ question"   ──► question
        LLM(mini): answer(question)             ──► answer
        LLM(full): judge(question, answer)      ──► verdict

Step 3: Exercise (yours)
        LLM: pick a business area  ──► area
        LLM: describe a pain point in area ──► pain point
        LLM: propose an agentic solution   ──► solution
```

### 1.2 Why this is the foundation of Agentic AI

An agent is, at its core, **a program that repeatedly calls an LLM and acts on the results**. Before you can build loops, tools, memory, or multiple agents, you must master the atomic operation:

```
messages (list of dicts)  ──►  LLM  ──►  response  ──►  extract text  ──►  use it
```

Lab 1 teaches that atomic operation plus the simplest composition: feeding output into the next input. Everything later (tool calling, ReAct, LangGraph, multi-agent systems) is this primitive plus control flow.

### 1.3 Key ideas introduced (glossary)

| Term | Meaning |
|---|---|
| **LLM** | A neural network (Transformer) trained to predict the next token. |
| **API** | A web interface through which your code sends a request and receives a response. |
| **API key** | A secret string identifying and billing your account. |
| **Environment variable** | A named value stored in the OS process environment, used to keep secrets out of code. |
| **`.env` file** | A text file of `KEY=value` lines loaded into environment variables by `python-dotenv`. |
| **Message** | A dict with `role` and `content`. |
| **Role** | Who is speaking: `system`, `user`, `assistant` (later also `tool`). |
| **Completion** | The model's generated continuation. |
| **Chaining** | Using one call's output as part of the next call's input. |
| **LLM-as-judge** | Using an LLM to evaluate another LLM's output. |
| **Model tiering** | Using cheap models for easy sub-tasks and stronger models for hard ones. |

---

## 2. Cell-by-Cell Map of the Notebook

| # | Cell type | What it does | Concept |
|---|---|---|---|
| 1 | Markdown | Title and "Are you ready?" banner | Pre-flight checklist (setup, README, guides) |
| 2 | Markdown | "Live resource" banner | The course repo changes; code may differ from videos |
| 3 | Markdown | Kernel selection instructions | Choose the `.venv` Python 3.12 kernel |
| 4 | Code | `from dotenv import load_dotenv` | Import the `.env` loader |
| 5 | Code | `load_dotenv(override=True)` | Load keys into environment, returns `True`/`False` |
| 6 | Markdown | "Did that output False?" | Troubleshooting `.env` |
| 7 | Markdown | Final reminders | Pointers to guides 4, 6, 9 |
| 8 | Code | `os.getenv('OPENAI_API_KEY')` and print prefix | Verify the key exists without leaking it |
| 9 | Code | `from openai import OpenAI` | Import the SDK client class |
| 10 | Code | `openai = OpenAI()` | Create the client instance |
| 11 | Code | `messages = [{"role": "user", "content": "Tell me a fun fact"}]` | Build the input |
| 12 | Code | `messages` | Inspect it |
| 13 | Code | `chat.completions.create(model="gpt-5.4-nano", ...)` | **First API call** |
| 14 | Code | Define `question` prompt, wrap in `messages` | Prompt for a question |
| 15 | Code | Call with `gpt-5.4-mini`, store `question` | **Call 1 of chain**: generate a question |
| 16 | Code | `messages = [{"role": "user", "content": question}]` | Feed output as new input |
| 17 | Code | Call with `gpt-5.4-mini`, store `answer` | **Call 2**: answer the question |
| 18 | Code | `display(Markdown(answer))` | Render markdown nicely in Jupyter |
| 19 | Code | Build an f-string `message` containing question and answer | Prompt assembly (templating) |
| 20 | Code | Call with `gpt-5.4` | **Call 3**: judge the answer |
| 21 | Markdown | Congratulations | Wrap-up |
| 22 | Markdown | Exercise description | Three-call business-idea chain |
| 23 | Code | Skeleton with blanks (`response =`, `business_area = response.`) | Your task; intentionally incomplete |

> **Note on model names.** The notebook uses a tiered family of names (`nano`, `mini`, and the full model). Treat the exact names as whatever your provider currently offers; the *pattern* (small → medium → large) is what matters. Names change over time, so check your provider's model list if a call fails with "model not found."

---

## 3. Part A: Environment and Secrets

### 3.1 Why a virtual environment and a kernel

A **virtual environment** (`.venv`) is an isolated folder containing its own Python interpreter and installed packages. It prevents version conflicts between projects.

A **Jupyter kernel** is the process that actually executes your notebook cells. If the notebook's kernel is not the `.venv` interpreter, then `import openai` or `import dotenv` will fail even though you "installed" them, because you installed into a different Python.

```
Notebook (.ipynb)  ──►  Kernel (a Python process)  ──►  Interpreter in .venv  ──►  site-packages (openai, dotenv, ...)
```

**Rule of thumb:** `ModuleNotFoundError` in a notebook is usually a *wrong kernel* problem before it is a *missing package* problem.

### 3.2 Environment variables

An environment variable is a key-value pair attached to a running process. Programs inherit them from their parent.

```python
import os
os.getenv("OPENAI_API_KEY")        # returns the string, or None if absent
os.environ["OPENAI_API_KEY"]       # returns the string, raises KeyError if absent
```

**Why not hard-code the key?**

| Hard-coded in source | Environment variable |
|---|---|
| Leaks when you push to GitHub | Stays in a local file that is git-ignored |
| Must edit code to rotate | Change one file |
| Same code cannot run in dev vs prod | Same code, different environments |

### 3.3 `python-dotenv` and `.env`

A `.env` file looks like this (no quotes or spaces needed):

```
OPENAI_API_KEY=sk-...
```

```python
from dotenv import load_dotenv
load_dotenv(override=True)
```

- `load_dotenv()` searches for a `.env` file (starting from the current directory and walking upward), parses it, and copies each pair into `os.environ`.
- **Return value:** `True` if it found a file and loaded at least one variable; `False` otherwise. That is why the notebook says "if this returns false, see the next cell."
- **`override=True`:** if a variable with the same name already exists in the OS environment, the `.env` value **wins**. With the default `override=False`, an existing OS variable would silently win, a classic source of "why is it using my old key?" bugs.

**Why `False` happens (checklist from the notebook):**

1. You did not **save** the `.env` file after pasting the key.
2. The file is not named exactly `.env` (for example `.env.txt` on Windows).
3. The file is not in the project root where the search finds it.

### 3.4 The key-check cell

```python
import os
openai_api_key = os.getenv('OPENAI_API_KEY')

if openai_api_key:
    print(f"OpenAI API Key exists and begins {openai_api_key[:8]}")
else:
    print("OpenAI API Key not set - please head to the troubleshooting guide in the setup folder")
```

Details worth understanding:

- `if openai_api_key:` is true for any non-empty string, false for `None` and `""`.
- Printing only `[:8]` (the first 8 characters) verifies the key without exposing it. **Never print or commit a full key.** Notebook outputs get saved into the `.ipynb` file and can leak.
- Leading/trailing whitespace or quotes copied into `.env` cause authentication errors later; this check helps catch a wrong prefix.

---

## 4. Part B: LLM Fundamentals You Need First

You do not need the full math of Transformers to run this notebook, but you do need the right mental model to understand its behavior.

### 4.1 What an LLM is, mathematically

An LLM models the probability of a token sequence **autoregressively**:

$$P(x_1, x_2, \dots, x_T) = \prod_{t=1}^{T} P(x_t \mid x_1, \dots, x_{t-1})$$

Generation repeats: given the prefix, compute a distribution over the vocabulary, pick a token, append it, repeat until a stop token or a length limit.

### 4.2 Tokens

Models do not read characters or words; they read **tokens** (sub-word pieces produced by a tokenizer such as BPE). Roughly, one token is about 4 English characters or 0.75 words (a heuristic, not a rule).

Why you care:

- **Cost** is billed per input token and per output token.
- **Context window** is the maximum number of tokens (prompt + output) the model can handle at once.
- Non-English text and code often use more tokens per word.

**Cost model:**

$$\text{Cost} = \frac{N_{in}}{10^6}\,p_{in} + \frac{N_{out}}{10^6}\,p_{out}$$

where $N_{in}, N_{out}$ are input/output token counts and $p_{in}, p_{out}$ are the per-million-token prices. Output tokens typically cost more than input tokens.

**Worked example.** Suppose a call has 500 input tokens and 300 output tokens, with prices of \$0.10 per million input and \$0.40 per million output (illustrative numbers only):

$$\text{Cost} = \frac{500}{10^6}(0.10) + \frac{300}{10^6}(0.40) = 0.00005 + 0.00012 = \$0.00017$$

Roughly 6,000 such calls cost one dollar. That is why the notebook calls the small model "incredibly cheap," and why *model tiering* matters at scale.

### 4.3 From logits to a sampled token

At each step the network outputs a vector of **logits** $z \in \mathbb{R}^{|V|}$, one per vocabulary item. Probabilities come from softmax with **temperature** $\tau$:

$$p_i = \frac{\exp(z_i/\tau)}{\sum_{j}\exp(z_j/\tau)}$$

**Worked numeric example.** Three candidate tokens with logits $z = (2.0,\ 1.0,\ 0.1)$:

| $\tau$ | $p_1$ | $p_2$ | $p_3$ | Behavior |
|---|---|---|---|---|
| 1.0 | 0.659 | 0.242 | 0.099 | Balanced sampling |
| 0.5 | 0.826 | 0.112 | 0.062 | Sharper, more deterministic |
| 2.0 | 0.502 | 0.304 | 0.194 | Flatter, more random |

Computation for $\tau=1$: $e^{2.0}=7.389$, $e^{1.0}=2.718$, $e^{0.1}=1.105$; sum $=11.212$; so $p_1 = 7.389/11.212 = 0.659$, $p_2 = 0.242$, $p_3 = 0.099$.

Intuition: $\tau \to 0$ approaches greedy decoding (always the argmax); $\tau \to \infty$ approaches uniform sampling. This is why **running the same prompt twice can give different answers**, and why the notebook's "hard IQ question" differs on each run.

Other decoding controls: **top-p (nucleus)** samples from the smallest set of tokens whose cumulative probability exceeds $p$; **top-k** restricts to the $k$ most likely tokens; **max tokens** caps output length. (Some newer model families restrict which sampling parameters you may set; consult the provider docs.)

### 4.4 Attention in one paragraph (conceptual background)

Inside the Transformer, each token builds a context-aware representation by attending to earlier tokens:

$$\text{Attention}(Q,K,V) = \text{softmax}\!\left(\frac{QK^\top}{\sqrt{d_k}} + M\right)V$$

where $M$ is a **causal mask** (entries $-\infty$ above the diagonal) so a token cannot see the future. The $\sqrt{d_k}$ scaling keeps dot products from growing with dimension, which would push softmax into saturated, near-one-hot regions with tiny gradients. Because each layer attends over the whole prompt, **everything you put in `messages` is "visible" to the model at every generated token**, which is exactly why passing the question and the answer in one prompt (the judge cell) works.

### 4.5 How chat models are trained (why roles exist)

1. **Pretraining:** next-token prediction on huge text corpora.
2. **Supervised fine-tuning (SFT):** learn to follow instructions from example conversations.
3. **Preference tuning (RLHF / DPO-style):** align outputs with human preferences (helpful, harmless, honest).

Chat models are trained on conversations serialized with special role markers. That is why the API takes a list of role-tagged messages rather than raw text: the server converts them into the model's internal chat format before running the network.

---

## 5. Part C: The Chat Completions API

### 5.1 Anatomy of a request

```python
response = openai.chat.completions.create(
    model="gpt-5.4-mini",
    messages=[{"role": "user", "content": "Tell me a fun fact"}],
)
```

```
client            resource    sub-resource   method
openai    .       chat    .   completions .  create(...)
```

Required inputs: `model` (which network) and `messages` (the conversation so far).

### 5.2 Message roles

| Role | Purpose | In this notebook? |
|---|---|---|
| `system` | Sets behavior, persona, rules, constraints | Not used (but you should add it, see §11) |
| `user` | The human (or your program) asking | **Yes**, every call |
| `assistant` | Prior model replies | Not used (needed for multi-turn memory) |
| `tool` | Results of tool calls | Not used (Lab on tool use) |

### 5.3 Statelessness (the most important API fact)

**The API remembers nothing between calls.** Each call is independent; the model only "knows" what is inside `messages` right now.

```
Call 1: messages=[user: "My name is Rabie"]     ──► "Nice to meet you, Rabie!"
Call 2: messages=[user: "What's my name?"]      ──► "I don't know."   (fresh call, no history)

Call 2 (fixed): messages=[
    user:      "My name is Rabie",
    assistant: "Nice to meet you, Rabie!",
    user:      "What's my name?"
] ──► "Rabie."
```

This is why in the notebook, after producing `question`, the code **rebuilds** `messages` with that question. Chat UIs feel like they have memory only because the application re-sends the whole transcript every turn. **Memory in agents is, at bottom, deciding what to put back into `messages`.** This single fact motivates context management, summarization, vector memory, and RAG later.

### 5.4 Anatomy of the response

`response` is an object (not a string). The relevant path:

```python
response.choices[0].message.content
```

| Piece | Meaning |
|---|---|
| `response.choices` | List of candidate completions (usually length 1; `n=` can request more) |
| `choices[0]` | First candidate |
| `.message` | The assistant message object (`role="assistant"`, `content=...`) |
| `.content` | The generated text (a `str`, can be `None` if the model made a tool call) |

Other useful fields: `response.usage` (token counts for cost tracking), `choices[0].finish_reason` (`"stop"` normal end, `"length"` truncated by max tokens, `"tool_calls"`, and so on), and `response.model`.

### 5.5 OpenAI-compatible endpoints

The notebook says: *"Even for other LLM providers like Gemini, you still use this OpenAI import."* Many providers (and local servers such as Ollama) expose an **OpenAI-compatible HTTP API**, so you only change the base URL and key:

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:11434/v1",   # e.g. Ollama (needs no real key)
    api_key="ollama",                        # placeholder
)
response = client.chat.completions.create(model="llama3.2", messages=messages)
```

Why this works: the SDK is a thin wrapper that sends `POST {base_url}/chat/completions` with a JSON body. If another server speaks the same JSON protocol, the same client works. This de-facto standard is why frameworks can swap providers easily. (Exact base URLs and model names are provider-specific; see the course's AI APIs guide.)

---

## 6. Part D: Detailed Code Walkthrough

### 6.1 Imports and client

```python
from dotenv import load_dotenv     # function that loads .env
load_dotenv(override=True)         # returns True/False
import os
from openai import OpenAI          # the client class
openai = OpenAI()                  # instance; reads OPENAI_API_KEY from the environment
```

Key points:

- `OpenAI()` with no arguments automatically reads `OPENAI_API_KEY` from the environment. That is why `load_dotenv` must run **before** this line.
- **Class vs instance:** `OpenAI` is the class (blueprint); `openai` is an object created from it. Notice the instance is named `openai`, the same as the package. This works because the code imported the *class*, not the module, but it can confuse readers. In your own code prefer `client = OpenAI()`.
- Order of cells matters in notebooks. A `NameError` almost always means *you skipped a cell*, or restarted the kernel and didn't re-run earlier cells.

### 6.2 First call

```python
messages = [{"role": "user", "content": "Tell me a fun fact"}]
response = openai.chat.completions.create(model="gpt-5.4-nano", messages=messages)
print(response.choices[0].message.content)
```

Data-flow:

```
messages ──► create() ──HTTP──► provider ──► response object ──► .choices[0].message.content ──► print
```

Why the nano model here: the task is trivial; a small, cheap, fast model suffices.

### 6.3 Generate a question (chain link 1)

```python
question = "Please propose a hard, challenging question to assess someone's IQ. Respond only with the question."
messages = [{"role": "user", "content": question}]
response = openai.chat.completions.create(model="gpt-5.4-mini", messages=messages)
question = response.choices[0].message.content
print(question)
```

Prompt-engineering details:

- **"Respond only with the question."** is an *output-format constraint*. Without it the model tends to add chatter ("Sure! Here's a question:"), and that chatter would pollute the next step's input. In chains, clean intermediate outputs are essential because the output is *data* for the next step, not text for a human.
- **Variable reuse:** `question` first holds the *prompt*, then is overwritten with the *model's generated question*. Overwriting is legal but a readability trap; prefer distinct names (`question_prompt`, `question`).

### 6.4 Answer the question (chain link 2)

```python
messages = [{"role": "user", "content": question}]
response = openai.chat.completions.create(model="gpt-5.4-mini", messages=messages)
answer = response.choices[0].message.content
print(answer)
```

This is **function composition**: `answer = LLM₂(LLM₁(prompt))`. Notice that the second call has *no memory* of the first; the only link is that your Python code pasted the first output into the second input.

### 6.5 Display as Markdown

```python
from IPython.display import Markdown, display
display(Markdown(answer))
```

LLMs often return Markdown (headings, bold, lists, LaTeX). `print` shows raw symbols; `display(Markdown(...))` renders them in the notebook. Display only; it does not change the data.

### 6.6 Assemble the judge prompt

```python
message = f"""
Here is a question:
{question}

And here is a possible answer that might be correct or incorrect:
{answer}

Please evaluate if the answer is correct or incorrect.
"""
```

- An **f-string** substitutes `{question}` and `{answer}` into a template: *prompt templating*.
- The triple quotes allow multi-line strings.
- The phrase "might be correct or incorrect" frames the answer as **untrusted**, which reduces the model's tendency to simply agree (sycophancy).
- Naming trap: this variable is `message` (singular, a **string**); the next one is `messages` (plural, a **list of dicts**). Confusing them is a common bug.

### 6.7 The judge call (chain link 3)

```python
messages = [{"role": "user", "content": message}]
response = openai.chat.completions.create(model="gpt-5.4", messages=messages)
print(response.choices[0].message.content)
```

The strongest model is used here. Rationale: **verification is the step where mistakes are costly**, and a judge should generally be at least as capable as the model being judged.

### 6.8 Full pipeline diagram

```
                ┌──────────────────────────────┐
 prompt_q ────► │ LLM  (mini)  "make question" │──► question
                └──────────────────────────────┘
                                │
                ┌───────────────▼──────────────┐
                │ LLM  (mini)  "answer it"     │──► answer
                └───────────────┬──────────────┘
                                │   (question, answer)
                ┌───────────────▼──────────────┐
                │ LLM  (full)  "judge it"      │──► verdict
                └──────────────────────────────┘
```

---

## 7. Part E: Chaining, Model Tiering, and LLM-as-Judge

### 7.1 Prompt chaining (a named workflow pattern)

**Prompt chaining** decomposes a task into a fixed sequence of LLM calls, where each step processes the previous output. It trades latency for accuracy and controllability: each call has an easier job and you can inspect or validate intermediate results.

| Benefit | Why |
|---|---|
| Simpler prompts | One goal per call |
| Debuggability | Print/inspect each intermediate output |
| Programmatic gates | Insert checks between steps |
| Per-step model choice | Cheap vs strong models |

Cost: more calls means more latency and tokens; errors can **compound**. If each step is correct with probability $p$, an $n$-step chain is correct with probability about $p^n$ (assuming independence). Example: $p=0.9, n=3 \Rightarrow 0.729$. That is why validation gates between steps matter.

### 7.2 Model tiering

Match model strength to subtask difficulty:

| Subtask | Suitable tier | Reason |
|---|---|---|
| Trivial text, formatting, classification | nano / small | Cheap, fast |
| Generation, summarization | mini / medium | Good quality/cost |
| Judging, hard reasoning, planning | full / large | Errors costly |

This is the same idea as using a small router model plus large specialist models in production systems.

### 7.3 LLM-as-judge

Using an LLM to grade outputs is the basis of scalable evaluation (and later, agent evaluation). Understand both power and pitfalls:

**Known biases/limitations:**

| Bias | Description | Mitigation |
|---|---|---|
| Self-preference | Judges favor text resembling their own style | Use a different model family as judge |
| Verbosity bias | Longer answers rated higher | Instruct to ignore length; use rubrics |
| Position bias | In pairwise comparisons, first/last option favored | Randomize/swap order |
| Sycophancy | Agrees with a confident-sounding answer | Frame answer as possibly wrong (as the notebook does) |
| Unreliable on hard math | Judge may itself be wrong | Use tools/code execution, reference answers |

**Important teaching point in this notebook:** the question is generated by an LLM and has **no ground-truth answer**. So the "judge" verdict is only an opinion of a stronger model, not verified truth. For a true correctness check you would use a known-answer benchmark, executable verification (code, calculator), or multiple independent judges (majority vote).

**Better judge prompt (rubric style):**

```text
You are a strict grader. Question:
{question}

Candidate answer:
{answer}

1. Solve the question yourself first, step by step.
2. Compare your solution with the candidate.
3. Output a JSON object: {"verdict": "correct"|"incorrect", "reason": "..."}.
```

Asking the judge to solve first reduces the chance it anchors on the candidate's reasoning.

---

## 8. Part F: From Chained Calls to Agentic Systems

### 8.1 Workflow vs agent

| Property | **Workflow** (this notebook) | **Agent** |
|---|---|---|
| Control flow | Fixed in code by the developer | Chosen dynamically by the LLM |
| Number of steps | Known in advance (3) | Unknown; loops until goal met |
| Tools | None | Calls tools/functions |
| Memory/state | Variables you manage | Managed state across steps |
| Predictability | High | Lower; needs guardrails |
| Typical use | Well-defined pipelines | Open-ended tasks |

**The notebook is a workflow, not yet an agent.** It is the right first rung of the ladder:

```
single call  →  chain (workflow)  →  routing/parallel  →  tool use loop (agent)  →  multi-agent
   Lab 1          Lab 1 (this)         next labs             ReAct, function calling      orchestration
```

### 8.2 The agent loop (what you are building toward)

```
        ┌──────────────────────────────────────────┐
        ▼                                          │
   observe (messages + tool results)               │
        ▼                                          │
   LLM decides: answer OR call tool ──tool call──► execute tool ──result──┘
        ▼
     final answer
```

Every box in this loop is a thing you already did in Lab 1 (build `messages`, call the LLM, read the response); the new parts are *tool execution* and *looping*.

### 8.3 Patterns visible in this lab

| Lab 1 element | Agentic pattern it foreshadows |
|---|---|
| Generate question then answer | **Prompt chaining** |
| Strong model judges weaker model | **Evaluator / critic** (evaluator-optimizer loop, reflection) |
| Output of call becomes input of next | **State passing** between nodes (LangGraph state) |
| Business area → pain point → solution | **Sequential task decomposition / planning** |

---

## 9. Part G: The Exercise, Fully Solved

**Task (from the notebook):** (1) ask an LLM to pick a business area worth exploring for an agentic AI opportunity; (2) ask it to present a pain point in that industry; (3) a third call proposes the agentic AI solution.

The skeleton in the notebook is **intentionally incomplete** (`response =` with nothing on the right is a `SyntaxError`). Here is a complete, cleaner solution, with a helper function to remove repetition:

```python
from dotenv import load_dotenv
from openai import OpenAI
from IPython.display import Markdown, display

load_dotenv(override=True)
client = OpenAI()

def ask(prompt: str, model: str = "gpt-5.4-mini") -> str:
    """Send one user message and return the assistant's text."""
    response = client.chat.completions.create(
        model=model,
        messages=[{"role": "user", "content": prompt}],
    )
    return response.choices[0].message.content

# Call 1: pick a business area
business_area = ask(
    "Pick one business area that might be worth exploring for an Agentic AI "
    "opportunity. Respond only with the name of the area and one short sentence of context."
)

# Call 2: pain point in that area
pain_point = ask(
    f"Business area: {business_area}\n\n"
    "Describe one specific, challenging pain point in this industry that is ripe for "
    "an agentic solution. Respond only with the pain point."
)

# Call 3: propose the agentic solution
solution = ask(
    f"Business area: {business_area}\n"
    f"Pain point: {pain_point}\n\n"
    "Propose an Agentic AI solution: what the agent does, what tools it uses, "
    "what could go wrong, and how to evaluate it.",
    model="gpt-5.4",          # stronger model for the creative/design step
)

display(Markdown(solution))
```

**Design notes:**

- The `ask()` helper encapsulates *build messages → call → extract text*. This is the first step of building your own tiny agent framework.
- Each later prompt **includes earlier outputs** (state passing). Call 3 receives both the area and the pain point so it has full context.
- Output constraints ("Respond only with...") keep intermediate results clean.
- The exercise says "have 3 third LLM call" (a typo for "a third").

**Extension challenge:** add a fourth call where a judge scores the proposed solution for feasibility (1–10) with a justification, then loop: if the score is below 7, regenerate with the critique as feedback. That is a basic **evaluator-optimizer loop**.

---

## 10. Part H: Common Errors and Debugging

| Symptom | Likely cause | Fix |
|---|---|---|
| `load_dotenv` returns `False` | `.env` unsaved, misnamed, or in the wrong folder | Save; rename to exactly `.env`; place in project root |
| "API Key not set" printed | Key variable missing/misspelled, or `load_dotenv` not run | Check `OPENAI_API_KEY=...` name; re-run the load cell |
| `ModuleNotFoundError: dotenv` / `openai` | Wrong kernel or package not installed in `.venv` | Select the `.venv` kernel; install packages into it |
| `NameError: name 'question' is not defined` | A cell was skipped, or kernel restarted | Run cells top to bottom |
| `AuthenticationError (401)` | Wrong or expired key, stray spaces/quotes | Re-copy key; no quotes/spaces in `.env` |
| `RateLimitError (429)` | Too many requests or no credit | Add credit; retry with backoff |
| `NotFoundError` for model | Model name not available to your account | Use a model from your provider's list |
| `SyntaxError` in the final cell | Exercise skeleton is incomplete by design | Fill in the blanks (see §9) |
| Output differs each run | Sampling randomness (temperature) | Expected; lower temperature for stability |
| Kernel list has no `.venv` | Editor can't find the environment | Set the virtual-environments folder path in editor settings |
| Judge says "correct" to a wrong answer | No ground truth; judge bias | Use rubric, reference answer, code verification |

**Debugging mindset for LLM programs:** print every intermediate variable (`question`, `answer`, the assembled `message`), because in a chain the bug is almost always an *intermediate output* that is not what the next step expects.

---

## 11. Part I: Extensions Beyond the Notebook

### 11.1 System prompt

```python
messages = [
    {"role": "system", "content": "You are a rigorous exam-question writer. Output only the question."},
    {"role": "user",   "content": "Write a hard probability question."},
]
```

System messages set stable behavior, which is more reliable than repeating instructions in each user message.

### 11.2 Multi-turn memory

```python
history = [{"role": "system", "content": "You are a helpful tutor."}]

def chat(user_text):
    history.append({"role": "user", "content": user_text})
    r = client.chat.completions.create(model="gpt-5.4-mini", messages=history)
    reply = r.choices[0].message.content
    history.append({"role": "assistant", "content": reply})   # <-- this is "memory"
    return reply
```

The `history` list *is* the memory. Its length grows each turn, which costs tokens and eventually hits the context limit. Fixes: truncation, summarization, retrieval (RAG).

### 11.3 Track usage and cost

```python
u = response.usage
print(u.prompt_tokens, u.completion_tokens, u.total_tokens)
```

### 11.4 Robustness: retries and timeouts

```python
import time
from openai import RateLimitError, APIError

def ask_safe(prompt, retries=3):
    for i in range(retries):
        try:
            return ask(prompt)
        except (RateLimitError, APIError):
            time.sleep(2 ** i)          # exponential backoff: 1s, 2s, 4s
    raise RuntimeError("LLM call failed after retries")
```

### 11.5 Structured output

Ask for JSON (or use the provider's structured-output / JSON-schema feature) so that the next step can *parse* the result instead of reading free text. Parsing is far more reliable than string guessing and is how tool calling works internally.

### 11.6 Streaming

Passing `stream=True` returns tokens incrementally, giving a better user experience for long answers.

### 11.7 Security reminders

- Never commit `.env`; add it to `.gitignore`.
- Never print full keys; rotate any key that was exposed.
- Treat model outputs as untrusted input when they feed into code, SQL, or shell commands (this becomes critical with tool-using agents and prompt injection).

---

## 12. Comparison Tables

### 12.1 Notebook calls compared

| Call | Model tier | Input | Output | Purpose |
|---|---|---|---|---|
| Fun fact | nano | "Tell me a fun fact" | A fact | Verify the setup works |
| Make question | mini | Instruction prompt | A question | Generate data |
| Answer | mini | The question | An answer | Solve |
| Judge | full | Question + answer | Evaluation | Verify |

### 12.2 `os.getenv` vs `os.environ[...]`

| | `os.getenv("K")` | `os.environ["K"]` |
|---|---|---|
| Missing key | Returns `None` | Raises `KeyError` |
| Default value | `os.getenv("K", "x")` | `os.environ.get("K", "x")` |
| Use when | Absence is acceptable and you will check | Absence is a hard error |

### 12.3 `print` vs `display(Markdown(...))`

| | `print` | `display(Markdown(...))` |
|---|---|---|
| Output | Raw text | Rendered formatting |
| Works outside Jupyter | Yes | Needs IPython |
| Changes data | No | No |

### 12.4 Single call vs chain vs agent

| | Single call | Chain | Agent |
|---|---|---|---|
| LLM calls | 1 | Fixed n | Variable |
| Who decides next step | N/A | Programmer | LLM |
| Tools | No | Optional | Yes |
| Predictability | Highest | High | Lower |
| Cost/latency | Lowest | Medium | Highest |

---

## 13. Exam Preparation

### 13.1 Conceptual questions with model answers

**Q1. Why are chat completion APIs stateless, and what does that imply for building a chatbot?**
Each request is processed independently; the server stores no conversation. The application must re-send the full relevant history in `messages` every turn. Consequently "memory" is a client-side responsibility and is limited by the context window and cost, which motivates summarization and retrieval.

**Q2. Why does `load_dotenv(override=True)` matter?**
It makes values in `.env` take precedence over variables already set in the OS environment, preventing stale or conflicting keys from silently being used.

**Q3. Why must `load_dotenv()` run before `OpenAI()`?**
`OpenAI()` reads `OPENAI_API_KEY` from `os.environ` at construction time; if the variable is not loaded yet, creation fails or authentication errors occur.

**Q4. What does `response.choices[0].message.content` navigate?**
The response holds a list of candidate completions; take the first, its assistant message object, and the text field of that message.

**Q5. Why use a stronger model for the judge step?**
Evaluation errors are the most costly, and a judge should be at least as capable as the generator, while cheap models handle easier steps. This is model tiering.

**Q6. Why add "Respond only with the question"?**
The output is consumed by the next step as data; extra chatter would contaminate the next prompt. It is an output-format constraint for pipeline cleanliness.

**Q7. Is the notebook's chain an agent? Justify.**
No. The control flow (three calls in fixed order) is hard-coded by the programmer; there are no tools, no loop, and the LLM does not decide the next action. It is a prompt-chaining workflow.

**Q8. Why can the same prompt produce different answers?**
Generation samples from a temperature-scaled softmax over tokens; unless temperature is 0 (or decoding is greedy), outputs vary. Even then, some serving systems are not perfectly deterministic.

**Q9. What is a weakness of the judge step in this notebook?**
There is no ground truth: both question and answer are LLM-generated, so the verdict is only another model's opinion and may be wrong or biased (self-preference, verbosity, sycophancy).

**Q10. Why is printing only the first 8 characters of the key good practice?**
It confirms the key loaded and has the right prefix without exposing the secret in the notebook output, which is saved to disk and may be shared.

**Q11. How can the same OpenAI client work with Gemini or Ollama?**
Those services expose OpenAI-compatible HTTP endpoints; you set `base_url` and `api_key` accordingly and keep the same `chat.completions.create` interface.

**Q12. What causes a `NameError` in notebooks and how do you fix it?**
Using a variable that was never defined in the current kernel session, typically from skipping a cell or restarting the kernel. Run the cells in order.

### 13.2 Calculation practice

**P1.** A call uses 2,000 input tokens and 500 output tokens. Prices: \$0.20/M input, \$0.80/M output. Cost?
Solution: $\frac{2000}{10^6}(0.20)+\frac{500}{10^6}(0.80) = 0.0004+0.0004 = \$0.0008$.

**P2.** Logits $(3, 1, 0)$ at $\tau=1$. Find the probability of the first token.
Solution: $e^3=20.086,\ e^1=2.718,\ e^0=1$; sum $=23.804$; $p_1 = 20.086/23.804 \approx 0.844$.

**P3.** A 4-step chain where each step is 95% reliable. End-to-end reliability (independent)?
Solution: $0.95^4 \approx 0.8145$.

**P4.** What happens to the distribution as $\tau \to 0$ and $\tau \to \infty$?
Solution: $\tau\to0$: mass concentrates on the largest logit (greedy/argmax). $\tau\to\infty$: all probabilities approach $1/|V|$ (uniform).

### 13.3 Code practice

1. Rewrite the notebook's judge section so that the judge returns strict JSON `{"verdict": ..., "reason": ...}` and parse it with `json.loads`.
2. Add a `system` message to the question-generation call so that it always returns exactly one question with no preamble.
3. Extend §9 with a loop: regenerate the solution up to 3 times until a judge scores it at least 7/10.
4. Replace the OpenAI client with an OpenAI-compatible local endpoint (for example Ollama) by changing only `base_url`, `api_key`, and `model`.
5. Implement `ask()` returning both text and token usage, and compute the cost per call.

### 13.4 Short-answer "spot the bug"

```python
messages = {"role": "user", "content": "Hi"}          # Bug 1
response = openai.chat.completions.create(messages=messages)   # Bug 2
print(response.choices.message.content)                # Bug 3
```

- Bug 1: `messages` must be a **list** of dicts, not a single dict.
- Bug 2: the required `model` argument is missing.
- Bug 3: `choices` is a list; you need `choices[0]`.

---

## 14. One-Page Cheat Sheet

```python
# --- SETUP ---
from dotenv import load_dotenv
import os
from openai import OpenAI
load_dotenv(override=True)                 # True if .env found
assert os.getenv("OPENAI_API_KEY"), "key missing"
client = OpenAI()                          # reads key from env

# --- ONE CALL ---
messages = [{"role": "user", "content": "..."}]
r = client.chat.completions.create(model="<model>", messages=messages)
text = r.choices[0].message.content

# --- CHAIN ---
q = ask("Write a question. Respond only with the question.")
a = ask(q)
v = ask(f"Question:\n{q}\n\nAnswer (may be wrong):\n{a}\n\nEvaluate it.", model="<strong model>")
```

**Ten facts to remember**

1. LLM = autoregressive next-token predictor; outputs are sampled, so they vary.
2. Chat APIs are **stateless**; you re-send history. Memory = what you put in `messages`.
3. `messages` is a **list of `{role, content}` dicts**; roles: `system`, `user`, `assistant`, `tool`.
4. Text lives at `response.choices[0].message.content`.
5. `load_dotenv()` must run before `OpenAI()`; it returns `True`/`False`.
6. `override=True` makes `.env` beat existing OS variables.
7. Never print or commit a full API key; show only a prefix.
8. Wrong kernel ≈ `ModuleNotFoundError`; skipped cell ≈ `NameError`.
9. Chaining = output of call *n* becomes input of call *n+1*; constrain output formats; validate between steps.
10. Cheap models for easy steps, strong models for judging; LLM judges have biases and no ground truth.

**Where this goes next:** tool/function calling → ReAct loops → memory and RAG → LangGraph/multi-agent orchestration → evaluation and observability. Every one of them is built from the primitive you mastered here: **messages in, text out, code decides what happens next.**
