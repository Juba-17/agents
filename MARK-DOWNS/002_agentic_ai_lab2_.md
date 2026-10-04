# Agentic AI — Lab 2 Master Study Guide
### "The LLM Competition": Multi-Provider APIs, OpenAI-Compatible Endpoints, Parallelization, and Ranking with an LLM Judge

> **Source notebook:** `2_lab2.ipynb` (Week 1, Day 3)
> **What the notebook really teaches:** (1) one client library can talk to *many* LLM providers because they share a wire protocol; (2) sending the **same task to several models** and having **another model rank the results** is a reusable design pattern for improving quality. In agentic-design-pattern vocabulary this is **Parallelization (fan-out) + Evaluator (LLM-as-judge)**.

---

## Table of Contents

0. [How to Use This Guide](#0-how-to-use-this-guide)
1. [The Big Picture](#1-the-big-picture)
2. [Cell-by-Cell Map](#2-cell-by-cell-map)
3. [Part A: Setup, Multiple Keys, Safe Debugging](#3-part-a-setup-multiple-keys-safe-debugging)
4. [Part B: OpenAI-Compatible Endpoints (the Core Idea)](#4-part-b-openai-compatible-endpoints-the-core-idea)
5. [Part C: The Provider Landscape](#5-part-c-the-provider-landscape)
6. [Part D: Local Models with Ollama](#6-part-d-local-models-with-ollama)
7. [Part E: Detailed Code Walkthrough](#7-part-e-detailed-code-walkthrough)
8. [Part F: Design Patterns Used (and the Exercise)](#8-part-f-design-patterns-used-and-the-exercise)
9. [Part G: Evaluation Theory: Judges, Rankings, Math](#9-part-g-evaluation-theory-judges-rankings-math)
10. [Part H: Weaknesses and Hardened Version of the Code](#10-part-h-weaknesses-and-hardened-version-of-the-code)
11. [Part I: Exercise Solutions (Adding More Patterns)](#11-part-i-exercise-solutions-adding-more-patterns)
12. [Common Errors and Debugging](#12-common-errors-and-debugging)
13. [Comparison Tables](#13-comparison-tables)
14. [Exam Preparation](#14-exam-preparation)
15. [One-Page Cheat Sheet](#15-one-page-cheat-sheet)

---

## 0. How to Use This Guide

| Goal | Read |
|---|---|
| Understand the notebook quickly | §1, §2 |
| Understand *why* one client works for every provider | §4 |
| Re-implement from memory | §7, §11 |
| Answer "which patterns did this use?" | §8 |
| Understand the math of ranking/aggregation | §9 |
| Exam/interview prep | §14, §15 |

**Learning objectives.** After this guide you should be able to:

1. Explain what "OpenAI-compatible" means and configure `base_url` + `api_key` for any provider.
2. Run one prompt across many models, store results, and format them for a judge.
3. Write a judge prompt that returns **machine-parseable JSON**, and parse it safely.
4. Name the agentic patterns involved and extend the notebook with another pattern.
5. Explain the biases and statistical weaknesses of LLM-as-judge, and mitigate them (blind evaluation, multiple judges, Borda aggregation).
6. Spot the security and robustness traps hidden in the notebook (§10).

---

## 1. The Big Picture

### 1.1 What the notebook builds

```
                         ┌──► OpenAI  (gpt nano) ─────┐
                         ├──► Anthropic (Claude) ─────┤
 LLM generates           ├──► Gemini ─────────────────┤
 one hard question ──►   ├──► DeepSeek ───────────────┤──► answers[] ──► format ──► JUDGE (Grok) ──► JSON ranking
 (question)              ├──► Groq-hosted open model ─┤                                              │
                         ├──► OpenRouter-routed model ┤                                              ▼
                         └──► Ollama local models ────┘                                    map numbers → model names
```

Stages:

1. **Generate** a challenging, nuanced, short-answer question (with a model).
2. **Fan out** the question to many models from many providers (the "competitors").
3. **Collect** answers into parallel lists.
4. **Format** all answers into a single document (`together`), anonymized as "competitor 1, 2, ...".
5. **Judge**: a separate model ranks them and returns JSON.
6. **Decode** the ranking back to model names.

### 1.2 Why this matters for Agentic AI

| Idea | Where it reappears later |
|---|---|
| Provider-agnostic client | Frameworks (LangChain, LangGraph, LlamaIndex) wrap providers behind one interface |
| Fan-out then aggregate | Parallelization, ensembles, mixture-of-agents, self-consistency |
| LLM judges another | Evaluator–optimizer loops, agent evaluation, guardrails |
| Structured (JSON) output | Tool calling, function calling, agent state |
| Anonymized candidates | Reducing bias in any evaluation harness |

### 1.3 Glossary

| Term | Meaning |
|---|---|
| **Provider** | Company/service that hosts models behind an API (OpenAI, Anthropic, Google, DeepSeek, Groq, xAI, OpenRouter). |
| **Base URL** | Root URL of an API; the SDK appends paths like `/chat/completions`. |
| **OpenAI-compatible** | An API that accepts the same JSON request/response shape as OpenAI's Chat Completions. |
| **Aggregator / gateway** | A service (OpenRouter) that routes requests to many underlying providers with one key. |
| **Ollama** | Local runtime that serves open-weight models on your machine, exposing an OpenAI-compatible endpoint. |
| **Competitor** | In this notebook, one model's answer in the contest. |
| **Judge** | Model that evaluates/ranks competitors. |
| **Fan-out** | Sending one task to many workers/models. |
| **Reasoning effort** | A setting controlling how much hidden "thinking" a reasoning model does before answering. |

---

## 2. Cell-by-Cell Map

| # | Type | What it does | Concept |
|---|---|---|---|
| 1 | MD | Welcome; "Today we work with lots of models" | Goal: get comfortable with APIs |
| 2 | MD | Course philosophy (run it yourself *after* the lecture, add prints, make variations) | Active learning |
| 3 | Code | Imports: `os, json, load_dotenv, OpenAI, Markdown, display` | Toolkit (`json` is new vs Lab 1) |
| 4 | Code | `load_dotenv(override=True)` | Load all keys |
| 5 | Code | Read 7 env keys, print prefixes | Safe debugging of many keys |
| 6 | Code | Build `request` (prompt-writing prompt), `+=` the output constraint | Meta-prompting, string building |
| 7 | Code | `messages` | Inspect |
| 8 | Code | `openai = OpenAI()`; call `gpt-5.4-mini` → `question` | Generate the contest question |
| 9 | MD | "Calling LLMs from multiple providers" | OpenAI-compatible idea |
| 10 | Code | Constants: 7 base URLs | Endpoint registry |
| 11 | Code | Build 7 `OpenAI(api_key=..., base_url=...)` clients | One class, many backends |
| 12 | Code | `competitors = []`, `answers = []`, `messages = [question]` | Result stores + new prompt |
| 13 | Code | `def record(model_name, answer)` | Helper: store + display |
| 14 | Code | OpenAI nano call with `reasoning_effort="none"` | Provider-specific parameter |
| 15 | Code | Anthropic (`claude-sonnet-4-6`) via compat endpoint | Provider 2 |
| 16 | Code | Gemini (`gemini-3.1-flash-lite`) | Provider 3 |
| 17 | Code | DeepSeek (`deepseek-v4-flash`) | Provider 4 |
| 18 | Code | Groq (`openai/gpt-oss-120b`), manual append | Provider 5 (no `record`) |
| 19 | Code | OpenRouter (`moonshotai/kimi-k2.6`) | Provider 6 (gateway) |
| 20 | MD | Ollama explainer and commands | Local inference |
| 21 | MD | Warning: model size, avoid `:cloud` tags | Hardware limits |
| 22 | Code | `!ollama pull llama3.2` | Shell magic in Jupyter |
| 23 | Code | `requests.get('http://localhost:11434')` | Health check |
| 24 | Code | `GET /v1/models` and print ids | List local models |
| 25–27 | Code | Ollama calls: `llama3.2:1b`, `gpt-oss:latest`, `gemma4:latest` | Local competitors |
| 28 | Code | Print count, names, answers | Sanity check |
| 29 | Code | `zip(competitors, answers)` loop | Parallel iteration |
| 30 | Code | `enumerate` loop building `together` | Anonymized formatting |
| 31 | Code | `print(together)` | Inspect |
| 32 | Code | Build the `judge` prompt (f-string with `{{ }}`) | Judge prompt + JSON format |
| 33 | Code | `print(judge)`; `judge_messages = [...]` | Inspect and wrap |
| 34 | MD | "And now for Grok!" | Choose the judge |
| 35 | Code | Grok call → `results` string | Judging |
| 36 | Code | `json.loads` → ranks → map to names | Decode ranking |
| 37 | MD | Exercise: which patterns? add another | Pattern identification |
| 38 | MD | Commercial implications | Multi-model + evaluation for accuracy-critical work |

> **Model names:** the notebook names specific models (e.g. `gpt-5.4-nano`, `claude-sonnet-4-6`, `grok-4.3`). I can't verify these, and names change frequently. The *pattern* is what matters; if a call fails with "model not found," check the provider's current model list.

---

## 3. Part A: Setup, Multiple Keys, Safe Debugging

### 3.1 Imports

```python
import os, json
from dotenv import load_dotenv
from openai import OpenAI
from IPython.display import Markdown, display
```

New compared with Lab 1: **`json`** (parse the judge's JSON reply).

### 3.2 The key-prefix check, and why prefix lengths differ

```python
openai_api_key    = os.getenv('OPENAI_API_KEY')
anthropic_api_key = os.getenv('ANTHROPIC_API_KEY')
google_api_key    = os.getenv('GOOGLE_API_KEY')
deepseek_api_key  = os.getenv('DEEPSEEK_API_KEY')
groq_api_key      = os.getenv('GROQ_API_KEY')
grok_api_key      = os.getenv('GROK_API_KEY')
openrouter_api_key= os.getenv('OPENROUTER_API_KEY')
```

- `os.getenv` returns `None` for missing variables; `if key:` is false for `None`/empty.
- The slices (`[:8]`, `[:7]`, `[:2]`, `[:3]`, `[:4]`, `[:6]`) print only the **start** of each key. Different lengths exist because providers' keys have different standard prefixes, so you can eyeball that the right kind of key loaded without exposing it.
- Only OpenAI's key is required; the rest are labelled optional. You do **not** need every provider.
- Naming gotcha: **Groq** (the inference company) and **Grok** (xAI's model) are different things with confusingly similar names, and so are their env vars (`GROQ_API_KEY` vs `GROK_API_KEY`). Mixing them up gives authentication errors.

### 3.3 Secrets hygiene (extended)

1. `.env` stays out of version control (`.gitignore`).
2. Print prefixes only; notebook outputs are stored inside the `.ipynb` file.
3. One key per provider; rotate on suspicion of leak.
4. Spending limits on provider dashboards are your safety net against a leaked key or a runaway loop.

---

## 4. Part B: OpenAI-Compatible Endpoints (the Core Idea)

### 4.1 What is actually on the wire

The OpenAI Python SDK is a convenience wrapper over HTTP. A call like

```python
client.chat.completions.create(model="X", messages=[{"role":"user","content":"Hi"}])
```

becomes

```
POST {base_url}/chat/completions
Authorization: Bearer {api_key}
Content-Type: application/json

{"model": "X", "messages": [{"role": "user", "content": "Hi"}]}
```

and the server replies with JSON shaped like

```json
{"choices": [{"index": 0, "message": {"role": "assistant", "content": "..."}, "finish_reason": "stop"}],
 "usage": {"prompt_tokens": 9, "completion_tokens": 12, "total_tokens": 21}}
```

**Key insight:** if another server accepts that same request and returns that same response shape, the *same client class* works. You only change **where** it sends the request (`base_url`) and **who you are** (`api_key`).

### 4.2 The pattern in the notebook

```python
ANTHROPIC_BASE_URL  = "https://api.anthropic.com/v1/"
DEEPSEEK_BASE_URL   = "https://api.deepseek.com/v1"
GEMINI_BASE_URL     = "https://generativelanguage.googleapis.com/v1beta/openai/"
GROQ_BASE_URL       = "https://api.groq.com/openai/v1"
GROK_BASE_URL       = "https://api.x.ai/v1"
OPENROUTER_BASE_URL = "https://openrouter.ai/api/v1"
OLLAMA_BASE_URL     = "http://localhost:11434/v1"

anthropic = OpenAI(api_key=anthropic_api_key, base_url=ANTHROPIC_BASE_URL)
...
ollama    = OpenAI(base_url=OLLAMA_BASE_URL, api_key="ollama")
```

Each variable (`anthropic`, `gemini`, ...) is an **`OpenAI` object**, an instance of the same class pointed at a different server. The variable names are only labels you chose; the class never changes.

```
          same class OpenAI
   ┌──────────┬──────────┬──────────┬──────────┐
 base_url:  api.openai   api.anthropic  api.x.ai   localhost:11434
 api_key :  OPENAI key   ANTHROPIC key  GROK key   "ollama" (dummy)
```

**Why Ollama gets `api_key="ollama"`:** the SDK requires *some* key string, but a local server doesn't check it, so any placeholder works.

### 4.3 Caveats of compatibility layers

| Caveat | Explanation |
|---|---|
| Not 100% identical | Compat layers support the common core (messages, model, basic params) but may ignore or reject advanced/provider-specific features. |
| Native SDK may be better | For provider-specific features (for example, Anthropic's extended thinking, prompt caching, or tool details), the provider's own SDK exposes more. Compat endpoints are mainly a convenience for quick comparison and migration. Verify in the provider's docs. |
| Parameter differences | Notebook example: `reasoning_effort="none"` is passed to OpenAI only; other providers may not understand it. |
| Model names are provider-specific | `"openai/gpt-oss-120b"` (Groq naming) vs `"gpt-oss:latest"` (Ollama naming) refer to the same family under different hosts' conventions. |

### 4.4 The `reasoning_effort` parameter

```python
openai.chat.completions.create(model=model_name, messages=messages, reasoning_effort="none")
```

Reasoning models spend extra hidden "thinking" tokens before answering. `reasoning_effort` trades **quality/latency/cost** (levels listed in the notebook comment: none, low, medium, high, xhigh). Setting `"none"` on a small model makes it fast and cheap, which is appropriate for a quick, short answer; higher effort suits hard multi-step problems. Hidden reasoning tokens are typically billed as output tokens, so effort directly affects cost.

---

## 5. Part C: The Provider Landscape

| Provider/service | What it is | Notebook role |
|---|---|---|
| **OpenAI** | Original API; the reference format | Question generator + competitor |
| **Anthropic** | Claude models | Competitor (via compat endpoint) |
| **Google (Gemini)** | Gemini models | Competitor (via OpenAI-compat path) |
| **DeepSeek** | Open-weight-oriented lab, low-cost API | Competitor |
| **Groq** | **Inference hardware company** that serves open models very fast | Hosts an open-weight model |
| **xAI (Grok)** | Grok models | **Judge** |
| **OpenRouter** | **Gateway**: one key, many providers/models | Routes to a third-party model |
| **Ollama** | **Local** runtime for open-weight models | Three local competitors |

**Closed vs open-weight, hosted vs local:**

| | Hosted API | Local (Ollama) |
|---|---|---|
| Cost | Per token | Hardware + electricity |
| Privacy | Data leaves your machine | Stays local |
| Quality ceiling | Frontier models | Limited by your RAM/GPU |
| Latency | Network + queue | No network, but slower hardware |
| Setup | API key | Install, download weights |

Interesting detail: the *same open-weight model family* can appear via Groq (hosted, fast hardware) and via Ollama (your laptop, usually smaller or quantized). Same weights, different serving.

---

## 6. Part D: Local Models with Ollama

### 6.1 What Ollama does

Runs a **local web service** (default port **11434**) that loads open-weight models and exposes an OpenAI-compatible `/v1` API. Under the hood it uses optimized native (C++) inference code, often with **quantized** weights to fit consumer hardware.

### 6.2 Commands in the notebook

| Command | Purpose |
|---|---|
| `ollama serve` | Start the server (terminal) |
| `ollama pull <model>` | Download a model |
| `ollama ls` | List downloaded models |
| `ollama rm <model>` | Delete a model |
| `!ollama pull llama3.2` | The `!` prefix runs a **shell command from a notebook cell** |

### 6.3 Health checks

```python
import requests
requests.get('http://localhost:11434').content            # expect "Ollama is running"
models = requests.get('http://localhost:11434/v1/models').json()
for model in models.get("data"):
    print(model.get("id"))
```

- First call: a ping; if the server is down you get a connection error.
- Second: the **models list endpoint** (also part of the OpenAI API shape). `.json()` parses the body to a dict; `models.get("data")` is a list of model records; each has an `"id"`.
- **Use the printed ids as `model_name`.** A mismatch (for example `llama3.2` vs `llama3.2:1b` tags) gives "model not found."

### 6.4 Sizing rule

Model memory ≈ parameters × bytes per parameter. For a model with $N$ parameters at $b$ bits per weight:

$$\text{Memory} \approx \frac{N \cdot b}{8}\ \text{bytes} \;(+\text{ overhead for KV cache, activations})$$

**Worked example.** A 1-billion-parameter model at 16 bits: $\frac{10^9 \times 16}{8} = 2\times10^9$ bytes ≈ 2 GB. At 4-bit quantization: ≈ 0.5 GB. A 120B model at 4 bits ≈ 60 GB, far too large for typical laptops. This is why the notebook warns: prefer models around 3 GB or smaller, and avoid tags ending in `:cloud`, which mean the model runs in a cloud service rather than on your machine.

---

## 7. Part E: Detailed Code Walkthrough

### 7.1 Meta-prompting: the model writes the test question

```python
request = """
Please come up with a challenging, nuanced question with a succinct answer,
that I can ask a number of LLMs to evaluate their intelligence.
Not a mathematical puzzle, but more of a thought-provoking question that requires intelligent insight.
Include in your question that the answer must be short.
"""
request += "Answer only with the question, no explanation."
messages = [{"role": "user", "content": request}]
```

Why each sentence exists:

| Phrase | Purpose |
|---|---|
| "challenging, nuanced" | Separates strong from weak models |
| "succinct answer" / "answer must be short" | Keeps outputs comparable and cheap; short answers are easier to judge |
| "Not a mathematical puzzle" | Avoids a question with a single checkable answer, so the judge evaluates *insight and argument* instead |
| "Answer only with the question" | Clean output as data for the next step |

`request += "..."` is **string concatenation** (`a += b` means `a = a + b`). Python strings are immutable, so this creates a new string and rebinds the name.

This is **meta-prompting**: using an LLM to generate the inputs (prompts, test sets, rubrics) for other LLM work. Related to synthetic evaluation data.

### 7.2 Generate the question

```python
openai = OpenAI()
response = openai.chat.completions.create(model="gpt-5.4-mini", messages=messages)
question = response.choices[0].message.content
display(Markdown(question))
```

(The client instance is again named `openai`; prefer `client`/`openai_client`.)

### 7.3 Shared containers and the `record` helper

```python
competitors = []
answers = []
messages = [{"role": "user", "content": question}]

def record(model_name, answer):
    competitors.append(model_name)
    answers.append(answer)
    display(Markdown(answer))
```

- `competitors` and `answers` are **parallel lists**: index $i$ in one corresponds to index $i$ in the other. The whole judging step depends on this alignment. (A list of dicts or a dict `{model: answer}` is more robust.)
- `messages` is **rebuilt** with the new `question`. The same `messages` list is then reused for every provider: every model sees *exactly the same input*, which is what makes the comparison fair (a controlled experiment, with the model as the only varying factor).
- `record` is a **function with side effects**: it mutates the global lists and displays output. It reads and mutates globals without reassigning them, so no `global` keyword is needed (`append` mutates in place). Reassigning (`answers = []` inside the function) would instead need `global`.
- Its purpose is **DRY** (don't repeat yourself): every competitor cell would otherwise repeat three lines.

### 7.4 The per-provider call pattern

Every competitor cell is the same four lines:

```python
model_name = "<model id>"
response = <client>.chat.completions.create(model=model_name, messages=messages)
answer = response.choices[0].message.content
record(model_name, answer)
```

Only the **client** and **model name** change. This regularity is the payoff of compatibility, and it suggests a loop (see §10).

Inconsistency to notice: the Groq and two Ollama cells do **not** call `record`; they inline `display(...)`, `competitors.append(...)`, `answers.append(...)`. Behavior is the same, but it's duplication; a future edit to `record` would not apply there.

### 7.5 Inspecting results: `zip` and `enumerate`

```python
for competitor, answer in zip(competitors, answers):
    print(f"Competitor: {competitor}\n\n{answer}")
```

`zip(a, b)` pairs elements positionally and stops at the shorter list. This is the standard way to walk parallel lists.

```python
together = ""
for index, answer in enumerate(answers):
    together += f"# Response from competitor {index+1}\n\n"
    together += answer + "\n\n"
```

`enumerate(seq)` yields `(index, item)` with `index` starting at **0**; the code uses `index+1` so humans (and the judge) see **1-based** labels.

**Critical design choice: anonymization.** The labels are "competitor 1, 2, 3...", not model names. This is **blind evaluation**: the judge cannot favor a brand it likes, nor penalize one it dislikes. The decoding step later restores names from positions.

Resulting text looks like:

```
# Response from competitor 1

<answer 1>

# Response from competitor 2

<answer 2>
...
```

### 7.6 The judge prompt

```python
judge = f"""You are judging a competition between {len(competitors)} competitors.
Each model has been given this question:

{question}

Your job is to evaluate each response for clarity and strength of argument, and rank them in order of best to worst.
Respond with JSON, and only JSON, with the following format:
{{"results": ["best competitor number", "second best competitor number", "third best competitor number", ...]}}

Here are the responses from each competitor:

{together}

Now respond with the JSON with the ranked order of the competitors, nothing else. Do not include markdown formatting or code blocks."""
```

Dissection:

| Element | Why |
|---|---|
| Role statement ("You are judging...") | Sets task framing |
| `{len(competitors)}` | Tells the judge how many to rank, so it doesn't drop or invent entries |
| `{question}` | The judge must know what was asked |
| Explicit criterion: "clarity and strength of argument" | A **rubric**; without it the judge invents its own criteria |
| "Respond with JSON, and only JSON" + format spec | Makes output machine-parseable |
| `{{ ... }}` | **Escaping braces in f-strings.** In an f-string, `{x}` is substituted, so a literal brace must be doubled: `{{` renders `{`, `}}` renders `}`. Single braces here would raise a `NameError`/`SyntaxError`. |
| Instruction repeated at the end | **Recency**: models weigh the end of the prompt heavily; repeating the format rule there reduces format violations |
| "Do not include markdown formatting or code blocks" | Models love wrapping JSON in triple backticks, which breaks `json.loads` |

### 7.7 Judge call, parse, and decode

```python
model_name = "grok-4.3"
response = grok.chat.completions.create(model=model_name, messages=judge_messages)
results = response.choices[0].message.content
print(results)

results_dict = json.loads(results)
ranks = results_dict["results"]
for index, result in enumerate(ranks):
    competitor = competitors[int(result)-1]
    print(f"Rank {index+1}: {competitor}")
```

Step by step with an example. Suppose 4 competitors and the judge returns the **string** `'{"results": ["3", "1", "4", "2"]}'`:

1. `json.loads(results)` converts the string to the dict `{"results": ["3","1","4","2"]}`.
2. `ranks = ["3","1","4","2"]` is best-to-worst competitor numbers.
3. `enumerate(ranks)` gives `(0,"3"), (1,"1"), (2,"4"), (3,"2")`.
4. `int(result)` converts `"3"` to `3`; `-1` converts the **1-based label to a 0-based list index** (2).
5. `competitors[2]` is the third model recorded. Printed as `Rank 1: <that model>` (`index+1` converts 0-based rank position back to 1-based).

**Off-by-one is the central hazard.** Labels start at 1 (humans), Python lists start at 0 (code); the `-1` bridges them.

Why `int(...)`: the format string shows numbers in quotes, so the judge returns strings; indexing needs integers.

---

## 8. Part F: Design Patterns Used (and the Exercise)

> **Exercise in the notebook:** "Which pattern(s) did this use? Try updating this to add another Agentic design pattern."

### 8.1 Pattern identification (model answer)

| Pattern | Where in the notebook |
|---|---|
| **Parallelization (sectioning/voting style)** | Same question sent to many models; independent calls that could run simultaneously |
| **Evaluator / LLM-as-judge** | Grok ranks the answers |
| **Prompt chaining** | Question generation → fan-out → formatting → judging (each output feeds the next) |
| **Meta-prompting / synthetic task generation** | A model writes the question for the others |
| **Model diversity / ensemble** | Multiple providers, closed/open, hosted/local |
| **Blind evaluation (bias mitigation)** | Anonymous "competitor N" labels |
| **Structured output** | JSON-only judge reply parsed with `json.loads` |

It is **not** an autonomous agent: the control flow is fixed by the programmer; no tools, no loops, no LLM-chosen next step. It is a **workflow**.

### 8.2 The catalog of common agentic design patterns (for extension)

| Pattern | Idea | Typical use |
|---|---|---|
| **Prompt chaining** | Fixed sequence of calls | Decompose tasks |
| **Routing** | A classifier picks which model/prompt handles an input | Cost/quality dispatch |
| **Parallelization** | Run subtasks or repeats concurrently; aggregate | Speed, confidence, coverage |
| **Orchestrator–workers** | An LLM plans subtasks dynamically and delegates | Open-ended tasks |
| **Evaluator–optimizer (reflection)** | Generate → critique → revise loop | Quality improvement |
| **Tool use** | Model calls functions/APIs | Grounding, actions |
| **Planning** | Decompose goal into steps first | Long tasks |
| **Multi-agent collaboration** | Several specialized agents interact | Complex systems |

Ideas for "add another pattern" are solved in §11: (a) **Reflection** (winner critiques-and-improves), (b) **Synthesis/aggregation** (mixture-of-agents), (c) **multiple judges + Borda** (robust evaluation), (d) **Routing** (judge picks a specialist), (e) true **parallel** execution.

---

## 9. Part G: Evaluation Theory: Judges, Rankings, Math

### 9.1 Why multi-model comparison improves quality

If each model independently produces a correct answer with probability $p$, then the chance that **at least one** of $n$ independent attempts is correct (the "best-of-$n$ / pass@$n$" view) is

$$P(\text{at least one correct}) = 1-(1-p)^n$$

**Worked example:** $p=0.6$, $n=5$: $1-0.4^5 = 1-0.01024 = 0.98976$. But this only helps if a **selector** (the judge) can *identify* the correct one. If the judge picks correctly with probability $q$ when a correct answer exists, expected success is roughly $q\,[1-(1-p)^n]$ (a simplification). **Quality of the selector bounds the benefit.**

Independence is optimistic; different models share training data and similar failure modes, so errors are **correlated**, and real gains are smaller than the formula suggests. Diverse providers/architectures reduce correlation.

### 9.2 Majority voting (self-consistency style)

For $n$ independent voters each correct with probability $p>0.5$, majority-vote accuracy (odd $n$) is

$$P_{\text{maj}} = \sum_{k=\lceil n/2\rceil}^{n}\binom{n}{k}p^k(1-p)^{n-k}$$

**Worked example:** $n=3$, $p=0.7$: $\binom{3}{2}(0.7)^2(0.3)+\binom{3}{3}(0.7)^3 = 3(0.49)(0.3)+0.343 = 0.441+0.343 = 0.784$. Voting lifts 70% to 78.4%. If $p<0.5$, voting makes things **worse**.

This applies to questions with a **discrete answer**. The notebook's open-ended question has no single right answer, so a judge ranking replaces voting.

### 9.3 Biases of LLM judges

| Bias | Description | Mitigation |
|---|---|---|
| **Position bias** | Candidates earlier (or later) in the prompt favored | Shuffle order across several judge runs |
| **Verbosity bias** | Longer/more elaborate answers favored | Rubric penalizing padding; question demands short answers (the notebook does this) |
| **Self-preference** | Judge favors text like its own | Use a judge **not among the competitors** (Grok isn't a competitor here, good) |
| **Brand/identity bias** | Favors names it knows | **Anonymize** (done) |
| **Sycophancy / style over substance** | Prefers confident, polished tone | Rubric on correctness/insight; reference answers |
| **Non-determinism** | Same prompt, different ranking | Several runs/judges; temperature low |
| **Format fragility** | Invalid JSON | Validation + retry |

**The notebook's "most truth-seeking" remark is marketing, not evidence of judge accuracy.** Pick a judge by measured agreement with humans or ground truth, not slogans.

### 9.4 Aggregating several rankings: Borda count

With $m$ judges and $n$ competitors, a competitor ranked at position $r$ (1 = best) earns $n - r$ points; sum across judges.

**Worked example.** $n=3$ competitors (A, B, C), $m=3$ judges:

| Judge | Ranking | A | B | C |
|---|---|---|---|---|
| J1 | A > B > C | 2 | 1 | 0 |
| J2 | B > A > C | 1 | 2 | 0 |
| J3 | A > C > B | 2 | 0 | 1 |
| **Total** | | **5** | **3** | **1** |

Final order: A, B, C. Borda uses full ranking information and is more robust than trusting one judge.

### 9.5 Agreement between rankings: Kendall's tau

To quantify how much two rankings agree:

$$\tau = \frac{C - D}{\binom{n}{2}}$$

where $C$ = number of concordant pairs (same relative order in both), $D$ = discordant pairs. $\tau=1$ identical, $-1$ reversed, $0$ unrelated.

**Worked example.** Ranking X: A>B>C>D. Ranking Y: A>C>B>D. Pairs (6 total): (A,B) concordant, (A,C) concordant, (A,D) concordant, (B,C) **discordant** (X: B before C, Y: C before B), (B,D) concordant, (C,D) concordant. $C=5, D=1$: $\tau = (5-1)/6 \approx 0.667$.

Low $\tau$ between repeated runs of the same judge means the judge is **unreliable**; low $\tau$ between a judge and humans means it's **invalid**.

### 9.6 Statistical caution: one question is an anecdote

A ranking on **one** question says almost nothing about overall model quality. Sound evaluation needs many diverse prompts, repeated runs, and aggregate statistics (win rate with confidence intervals). Win-rate standard error for $N$ comparisons with win probability $\hat p$:

$$SE \approx \sqrt{\frac{\hat p(1-\hat p)}{N}}$$

Example: $\hat p = 0.6$, $N=25$: $SE=\sqrt{0.24/25}\approx 0.098$, so the 95% interval is about $0.6 \pm 0.19$, which easily includes 0.5 (no real difference).

### 9.7 Cost of the experiment

Total cost $\approx \sum_{i=1}^{n} c_i + c_{\text{judge}}$, where each competitor $c_i$ depends on the question size, answer size, and price. The judge's **input** grows with $n \times$ (average answer length), so judging many verbose answers costs more. Hence "answer must be short."

---

## 10. Part H: Weaknesses and Hardened Version of the Code

### 10.1 Issues in the notebook (read these carefully)

| # | Issue | Why it matters | Fix |
|---|---|---|---|
| 1 | **Missing key silently becomes `None`** in `OpenAI(api_key=None, base_url=...)` | In the official SDK, `api_key=None` falls back to the `OPENAI_API_KEY` environment variable. If, say, `ANTHROPIC_API_KEY` is unset, the client could send your **OpenAI key to another provider's server**. | Only create a client when its key exists (see code below) |
| 2 | `json.loads` on raw output | Fails on code fences, extra text, or trailing commentary | Extract the JSON object, validate, retry |
| 3 | No validation of ranks | Judge may return duplicates, out-of-range numbers, or names instead of numbers; `int(result)` then crashes or `competitors[...]` raises `IndexError` | Validate it's a permutation of `1..n` |
| 4 | Sequential calls | Slow; total time = sum of latencies | Run in parallel threads/async |
| 5 | No error handling | One failed provider aborts the cell; later cells depend on earlier ones | `try/except`, skip failed models |
| 6 | `message.content` may be `None` or empty | Concatenating `None` raises `TypeError` | Check and substitute a placeholder |
| 7 | Parallel lists | Fragile alignment | Use a dict or list of records |
| 8 | Single judge, single question, single run | Noisy and biased (§9) | Multiple judges, shuffled orders, Borda |
| 9 | Inconsistent `record` use | Duplication | Use one helper everywhere |
| 10 | Global mutable state in a notebook | Re-running cells **duplicates** entries in `competitors`/`answers` (e.g., rerun a model cell twice → it appears twice and the count/labels shift) | Reset lists in one cell; or make calls idempotent |
| 11 | Judge sees `together` with heading syntax (`#`) inside answers that may also contain headings | Could confuse boundaries between responses | Use unambiguous delimiters (e.g., `<response id="1">...</response>`) |
| 12 | Reasoning/thinking models may return different fields | Some servers return reasoning separately or inline | Inspect the raw response when output looks odd |

### 10.2 Hardened implementation

```python
import os, json, random, re
from concurrent.futures import ThreadPoolExecutor, as_completed
from openai import OpenAI

# 1. Registry: one place describing every competitor
PROVIDERS = {
    "openai":    dict(base_url=None,                               key_env="OPENAI_API_KEY"),
    "anthropic": dict(base_url="https://api.anthropic.com/v1/",    key_env="ANTHROPIC_API_KEY"),
    "deepseek":  dict(base_url="https://api.deepseek.com/v1",      key_env="DEEPSEEK_API_KEY"),
    "ollama":    dict(base_url="http://localhost:11434/v1",        key_env=None),   # no key needed
}

def make_client(name):
    cfg = PROVIDERS[name]
    if cfg["key_env"] is None:
        return OpenAI(base_url=cfg["base_url"], api_key="ollama")
    key = os.getenv(cfg["key_env"])
    if not key:
        return None                       # <-- never construct a client without its own key
    return OpenAI(api_key=key, base_url=cfg["base_url"]) if cfg["base_url"] else OpenAI(api_key=key)

CONTESTANTS = [                           # (provider, model)
    ("openai", "gpt-5.4-nano"),
    ("anthropic", "claude-sonnet-4-6"),
    ("deepseek", "deepseek-v4-flash"),
    ("ollama", "llama3.2:1b"),
]

def ask(client, model, messages, **kw):
    r = client.chat.completions.create(model=model, messages=messages, **kw)
    return (r.choices[0].message.content or "").strip()

# 2. Parallel fan-out with per-model error handling
def run_contest(question):
    messages = [{"role": "user", "content": question}]
    results = {}
    def job(provider, model):
        client = make_client(provider)
        if client is None:
            raise RuntimeError(f"no key for {provider}")
        return ask(client, model, messages)

    with ThreadPoolExecutor(max_workers=len(CONTESTANTS)) as pool:
        futures = {pool.submit(job, p, m): m for p, m in CONTESTANTS}
        for fut in as_completed(futures):
            model = futures[fut]
            try:
                results[model] = fut.result()
            except Exception as e:
                print(f"[skip] {model}: {e}")
    return results                        # dict: model -> answer (robust alignment)

# 3. Blind, shuffled judge prompt with unambiguous delimiters
def build_judge_prompt(question, answers_by_model, rng):
    models = list(answers_by_model)
    rng.shuffle(models)                   # kills position bias across runs
    body = "\n".join(
        f'<response id="{i+1}">\n{answers_by_model[m]}\n</response>'
        for i, m in enumerate(models)
    )
    prompt = f"""You are judging {len(models)} responses to this question:

{question}

Rank them best to worst on insight, correctness, and clarity. Ignore length.
Return ONLY JSON: {{"results": ["<id>", "<id>", ...]}} containing every id exactly once.

{body}"""
    return prompt, models                 # `models[i]` is the model behind id i+1

# 4. Robust parse + validation
def parse_ranking(text, n):
    m = re.search(r"\{.*\}", text, re.DOTALL)         # tolerate fences/extra prose
    data = json.loads(m.group(0))
    ids = [int(x) for x in data["results"]]
    if sorted(ids) != list(range(1, n + 1)):
        raise ValueError(f"bad permutation: {ids}")
    return ids
```

Design notes: the **registry** removes copy-paste; **never building a client without its own key** closes the key-leak path; the **dict** replaces fragile parallel lists; **shuffling + a per-run seed/`rng`** reduces position bias; **regex extraction + permutation validation** handles messy output.

---

## 11. Part I: Exercise Solutions (Adding More Patterns)

All snippets assume the hardened helpers from §10.2 (`ask`, `make_client`, `run_contest`, `build_judge_prompt`, `parse_ranking`).

### 11.1 Pattern A: Multiple judges + Borda aggregation (robust evaluation)

```python
import random
JUDGES = [("openai", "gpt-5.4-mini"), ("anthropic", "claude-sonnet-4-6")]  # ideally not competitors

def borda(question, answers_by_model, judges=JUDGES, runs_per_judge=2):
    n = len(answers_by_model)
    score = {m: 0 for m in answers_by_model}
    for (prov, jmodel) in judges:
        client = make_client(prov)
        for r in range(runs_per_judge):
            rng = random.Random(r)                             # different order per run
            prompt, order = build_judge_prompt(question, answers_by_model, rng)
            try:
                ids = parse_ranking(ask(client, jmodel, [{"role":"user","content":prompt}]), n)
            except Exception as e:
                print("judge failed, skipping run:", e); continue
            for position, id_ in enumerate(ids):               # position 0 = best
                score[order[id_ - 1]] += n - (position + 1)    # Borda points
    return sorted(score.items(), key=lambda kv: -kv[1])
```

Patterns added: **ensemble of evaluators**, **order randomization**, **aggregation**.

### 11.2 Pattern B: Reflection (evaluator–optimizer loop)

```python
def reflect(question, draft, model_provider="openai", model="gpt-5.4-mini", rounds=2):
    client = make_client(model_provider)
    for _ in range(rounds):
        critique = ask(client, model, [{"role":"user","content":
            f"Question:\n{question}\n\nAnswer:\n{draft}\n\nCritique this answer: list concrete weaknesses only."}])
        draft = ask(client, model, [{"role":"user","content":
            f"Question:\n{question}\n\nPrevious answer:\n{draft}\n\nCritique:\n{critique}\n\n"
            "Write an improved short answer. Output only the answer."}])
    return draft
```

Apply `reflect` to the **winner** and check whether judges now prefer the refined version over the original (an A/B test of the pattern itself).

### 11.3 Pattern C: Synthesis / mixture-of-agents (aggregator)

```python
def synthesize(question, answers_by_model, provider="openai", model="gpt-5.4"):
    client = make_client(provider)
    body = "\n\n".join(f"<candidate>\n{a}\n</candidate>" for a in answers_by_model.values())
    prompt = (f"Question:\n{question}\n\nHere are candidate answers from different models:\n{body}\n\n"
              "Write one final short answer that keeps the best insights and fixes errors. Output only the answer.")
    return ask(client, model, [{"role":"user","content":prompt}])
```

This follows the **fan-out → aggregate** structure: instead of *selecting* one answer, it *combines* them. Risk: the synthesizer may repeat a shared error (correlated mistakes).

### 11.4 Pattern D: Routing

```python
def route(task):
    label = ask(make_client("openai"), "gpt-5.4-nano", [{"role":"user","content":
        f"Classify this task as exactly one word: 'code', 'reasoning', or 'chat'.\n\nTask: {task}"}]).lower()
    table = {"code": ("deepseek","deepseek-v4-flash"),
             "reasoning": ("anthropic","claude-sonnet-4-6"),
             "chat": ("ollama","llama3.2:1b")}
    prov, model = table.get(label, table["chat"])
    return ask(make_client(prov), model, [{"role":"user","content":task}])
```

Cheap model classifies; the right specialist answers. This is **routing** and a form of model tiering (Lab 1).

### 11.5 Putting it together

```
question ──► run_contest (parallel) ──► answers_by_model
                                          ├─► borda (multi-judge) ──► ranking
                                          ├─► reflect(winner)     ──► improved winner
                                          └─► synthesize(all)     ──► merged answer
                                  then judge: original winner vs reflected vs synthesized
```

---

## 12. Common Errors and Debugging

| Symptom | Likely cause | Fix |
|---|---|---|
| `AuthenticationError (401)` for one provider | Wrong/missing key, or **Groq vs Grok** variable mix-up | Check the exact env var and prefix output |
| `NotFoundError` / "model not found" | Model name wrong for that provider, or not available to your plan | Use the provider's exact id; list models |
| `404` on Anthropic/Gemini URL | Wrong base URL (missing `/v1/`, or `/openai/` path for Gemini) | Copy the notebook constants exactly |
| `ConnectionError` to `localhost:11434` | Ollama not running | Run `ollama serve`; open `http://localhost:11434` |
| Ollama "model not found" | Not pulled, or tag mismatch (`llama3.2` vs `llama3.2:1b`) | `ollama ls`; use printed ids |
| Ollama very slow or crashes | Model too large for RAM | Use ≤ ~3 GB models; avoid `:cloud` |
| `json.JSONDecodeError` | Judge wrapped output in code fences/prose | Strip/extract JSON; retry |
| `KeyError: 'results'` | Judge used a different key | Validate keys; tighten prompt |
| `IndexError` at `competitors[int(result)-1]` | Judge returned a number out of range, or 0 | Validate permutation |
| Wrong model shown at a rank | Re-ran cells, **duplicated entries** in lists, shifting positions | Reset `competitors`/`answers` and re-run all competitor cells once |
| `NameError: grok` / `ollama` | Client-creation cell not run | Run cells in order |
| `TypeError: can only concatenate str (not "NoneType")` | `content` was `None` | Use `or ""` fallback |
| Parameter rejected (`reasoning_effort`) | Provider doesn't support it | Only pass it to OpenAI |
| Judge always prefers competitor 1 | Position bias | Shuffle; multiple runs |
| Unexpected bill | Large judge prompt, high reasoning effort, loops | Set dashboard limits; keep answers short |

---

## 13. Comparison Tables

### 13.1 Lab 1 vs Lab 2

| | Lab 1 | Lab 2 |
|---|---|---|
| Providers | 1 (OpenAI) | 7 + local |
| Pattern | Prompt chaining | Parallelization + evaluator |
| Judge output | Free text verdict | **Structured JSON ranking** |
| Evaluation | Single answer, correct/incorrect | Comparative ranking |
| Bias controls | Frames answer as possibly wrong | Anonymized labels, judge outside the contestants |
| New Python | f-strings, `os.getenv` | `zip`, `enumerate`, `json`, `{{}}`, `+=`, helper functions, `!` shell, `requests` |

### 13.2 Ranking vs scoring vs pairwise

| Method | Pros | Cons |
|---|---|---|
| **Ranking** (notebook) | One call; relative order | Hard to see magnitude; sensitive to position; scale with $n$ |
| **Scoring** (1–10 each) | Absolute scale, easy to average | Score inflation; judges cluster at 8–9 |
| **Pairwise comparison** | Most reliable per judgment | $\binom{n}{2}$ calls |

### 13.3 Hosted API vs gateway vs local

| | Direct provider | OpenRouter (gateway) | Ollama (local) |
|---|---|---|---|
| Keys | One per provider | One for many | None |
| Model breadth | That provider's | Very wide | Open weights you download |
| Extra hop/fees | None | Yes (routing layer) | None |
| Privacy | Provider sees data | Gateway + provider see data | Fully local |

### 13.4 `zip` vs `enumerate`

| | `zip(a, b)` | `enumerate(a)` |
|---|---|---|
| Yields | Pairs `(a_i, b_i)` | `(i, a_i)` |
| Use when | Walking parallel sequences | Need index/numbering |
| Stops at | Shorter sequence | End of sequence |

---

## 14. Exam Preparation

### 14.1 Conceptual questions with model answers

**Q1. What does "OpenAI-compatible" mean, and why can one `OpenAI` class call Anthropic, Gemini, or Ollama?**
The provider's server accepts the same HTTP request format (`POST /chat/completions` with `model` and `messages`) and returns the same response shape. The SDK just sends HTTP, so changing `base_url` and `api_key` retargets it.

**Q2. Why does the Ollama client use `api_key="ollama"`?**
The SDK requires a non-empty key, but a local server doesn't authenticate, so any placeholder works.

**Q3. Why is the same `messages` list sent to every model?**
So all models face an identical input, making the comparison a controlled experiment where only the model varies.

**Q4. Why does the judge see "competitor 1, 2, ..." instead of model names?**
Blind evaluation removes brand/identity bias. Names are restored afterward by index.

**Q5. Explain the `-1` in `competitors[int(result)-1]`.**
Labels given to the judge are 1-based; Python lists are 0-based. Also `int()` converts the string ids returned in JSON to integers.

**Q6. Why are braces doubled in the judge f-string?**
In f-strings `{}` triggers substitution; `{{` and `}}` produce literal braces so the JSON format example survives.

**Q7. Why instruct "Respond with JSON only, no code blocks"?**
The output is parsed by `json.loads`; prose or markdown fences cause `JSONDecodeError`. It's a structured-output constraint.

**Q8. Which agentic patterns does the notebook use? Is it an agent?**
Parallelization (fan-out), evaluator/LLM-as-judge, prompt chaining, meta-prompting, model ensemble, and blind evaluation. It is a **workflow**, not an agent: fixed control flow, no tools, no LLM-chosen actions.

**Q9. What is a flaw of using one LLM judge on one question?**
Judge biases (position, verbosity, self-preference), non-determinism, and a sample size of one make the ranking an anecdote; there's also no ground truth for an open-ended question.

**Q10. Why is it good that Grok is not one of the competitors?**
It avoids self-preference bias, where a judge favors its own style/outputs.

**Q11. What is the difference between Groq and Grok?**
Groq is an inference company serving (mostly open) models on specialized hardware; Grok is xAI's model family. Different keys, different base URLs.

**Q12. What security issue arises if a provider's API key is missing and you pass `api_key=None`?**
The SDK may fall back to the `OPENAI_API_KEY` environment variable and send that key to the other provider's endpoint. Create clients only when their own key exists.

**Q13. Why does re-running a competitor cell corrupt the results?**
`record` appends to global lists; re-running adds duplicates, so counts and the index mapping from judge labels to models shift.

**Q14. What does `reasoning_effort` do and what's the trade-off?**
It controls hidden reasoning depth for reasoning models: more effort generally improves hard-problem quality but increases latency and cost (reasoning tokens are billed).

**Q15. When does multi-model fan-out help most?**
When errors are weakly correlated across models and a good selector/aggregator exists; it matters for accuracy-critical tasks (the notebook's "commercial implications"). It costs more and adds latency.

### 14.2 Calculation practice

**P1.** Each of 4 independent models is correct with probability 0.5 on a task. Probability at least one is correct?
$1-0.5^4 = 0.9375$.

**P2.** Three independent voters, each 80% accurate. Majority-vote accuracy?
$3(0.8^2)(0.2)+0.8^3 = 0.384+0.512 = 0.896$.

**P3.** Borda with $n=4$: Judge 1 ranks D>A>B>C; Judge 2 ranks A>D>C>B. Find totals.
Points $=4-\text{position}$: J1: D=3, A=2, B=1, C=0. J2: A=3, D=2, C=1, B=0. Totals: A=5, D=5, B=1, C=1 (tie A/D; tie B/C).

**P4.** Kendall's tau between X: A>B>C and Y: C>B>A.
All 3 pairs discordant: $\tau=(0-3)/3=-1$.

**P5.** A 7B-parameter model at 4-bit quantization: approximate weight memory?
$7\times10^9\times4/8 = 3.5\times10^9$ bytes ≈ 3.5 GB (plus overhead).

**P6.** Win rate 0.7 over $N=49$ comparisons. Standard error and approximate 95% interval?
$SE=\sqrt{0.21/49}\approx0.0655$; interval ≈ $0.7\pm0.128$.

### 14.3 Code practice

1. Replace the seven near-identical competitor cells with a loop over a list of `(client, model)` pairs; skip providers without keys.
2. Convert parallel lists to a single `dict` and update the decoding step accordingly.
3. Make `parse_ranking` robust to code fences and validate a full permutation.
4. Run the judging three times with shuffled order and aggregate with Borda; print Kendall's tau between the runs.
5. Run all competitors concurrently with `ThreadPoolExecutor` and measure the speedup vs sequential.
6. Add a rubric with weighted criteria (insight 50%, clarity 30%, brevity 20%) and have the judge return per-criterion scores as JSON.
7. Add a `usage`-based cost tracker per provider.

### 14.4 Spot the bug

```python
judge = f"""Rank them. Respond like {"results": ["1", "2"]}"""     # Bug 1
ranks = json.loads(results)["results"]
for i, r in enumerate(ranks):
    print(competitors[r])                                            # Bug 2
for c, a in zip(competitors, answers):
    together += a                                                    # Bug 3
```

- Bug 1: unescaped braces in an f-string (also the inner quotes clash with the triple-quoted f-string expression); must be `{{"results": [...]}}`.
- Bug 2: `r` is a **string** and 1-based; need `competitors[int(r)-1]`.
- Bug 3: `together` is never initialized in this snippet, and without labels/delimiters the judge cannot tell where one answer ends (and `a` could be `None`).

---

## 15. One-Page Cheat Sheet

```python
# Many providers, one class
client = OpenAI(api_key=KEY, base_url=BASE_URL)           # Ollama: api_key="ollama"
r = client.chat.completions.create(model=M, messages=msgs)
text = r.choices[0].message.content

# Contest
answers[model] = text                                      # same messages to every model

# Blind formatting (1-based labels)
together = "".join(f"# Response from competitor {i+1}\n\n{a}\n\n" for i, a in enumerate(answers_list))

# Judge returns JSON only; escape braces in f-strings: {{ }}
ranks = json.loads(results)["results"]
names = [competitors[int(x) - 1] for x in ranks]           # 1-based label -> 0-based index
```

**Ten facts to remember**

1. **OpenAI-compatible** = same HTTP JSON contract; swap `base_url` + `api_key`, keep the code.
2. Ollama = local server at `localhost:11434`; dummy key; mind model size (memory ≈ params × bits / 8).
3. Same `messages` to every model = fair comparison.
4. Anonymize candidates for the judge (blind evaluation); decode by index afterward.
5. Judge must return **JSON only**; validate before trusting (permutation of `1..n`).
6. Escape literal braces in f-strings with `{{ }}`; labels are 1-based, lists 0-based (`-1`).
7. Patterns used: **parallelization + evaluator (LLM-as-judge)**, plus chaining, meta-prompting, ensemble; it's a workflow, not an agent.
8. Judge biases: position, verbosity, self-preference, brand, sycophancy; mitigate by shuffling, multiple judges, rubric, outside-the-contest judge.
9. One question and one judge is an anecdote; use many prompts, aggregate (Borda), measure agreement (Kendall's tau).
10. Never create a client with `api_key=None`; avoid re-running cells that append to global lists; Groq ≠ Grok.

**Where this goes next:** structured outputs and tool calling → the agent loop → memory/RAG → orchestration frameworks → evaluation and observability. Lab 2 gave you the *evaluation harness* mindset that you will reuse to test every agent you build.
