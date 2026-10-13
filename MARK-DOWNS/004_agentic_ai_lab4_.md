# Agentic AI — Lab 4 Master Study Guide
### "The Digital Twin, Productionized": Real-World Tools (Pushover), Tool Dispatch, Modular Code, and Deployment

> **Source notebook:** `4_lab4.ipynb` (first big project: the Digital Twin)
> **What the notebook really teaches:** (1) how an agent's tools can touch the **real world** (a push notification to your phone); (2) how to **dispatch** tool calls cleanly (manual `if` vs a dictionary lookup via `globals()`), and why the lookup must be **protected**; (3) how to turn a notebook into **modules**; (4) how to **deploy** the app and manage **secrets**.
> Lab 3 built the agent loop. Lab 4 makes it *useful, organized, and shippable*.

---

## Table of Contents

0. [How to Use This Guide](#0-how-to-use-this-guide)
1. [The Big Picture](#1-the-big-picture)
2. [Cell-by-Cell Map](#2-cell-by-cell-map)
3. [Part A: Pushover and HTTP APIs](#3-part-a-pushover-and-http-apis)
4. [Part B: Two Real Tools and Their Schemas](#4-part-b-two-real-tools-and-their-schemas)
5. [Part C: Tool Dispatch: `if` vs `globals()` vs Registry](#5-part-c-tool-dispatch-if-vs-globals-vs-registry)
6. [Part D: The Upgraded System Prompt and `chat()`](#6-part-d-the-upgraded-system-prompt-and-chat)
7. [Part E: From Notebook to Modules](#7-part-e-from-notebook-to-modules)
8. [Part F: Deployment and Secrets](#8-part-f-deployment-and-secrets)
9. [Part G: Production Concerns and Math](#9-part-g-production-concerns-and-math)
10. [Part H: Weaknesses and a Hardened Version](#10-part-h-weaknesses-and-a-hardened-version)
11. [Part I: Exercise Solutions](#11-part-i-exercise-solutions)
12. [Common Errors and Debugging](#12-common-errors-and-debugging)
13. [Comparison Tables](#13-comparison-tables)
14. [Exam Preparation](#14-exam-preparation)
15. [One-Page Cheat Sheet](#15-one-page-cheat-sheet)

---

## 0. How to Use This Guide

| Goal | Read |
|---|---|
| Understand the notebook quickly | §1, §2 |
| Understand tool dispatch and its security risk | §5, §10 |
| Structure and ship your own app | §7, §8 |
| Re-implement everything from memory | §4, §5, §6, §7 |
| Exam/interview prep | §14, §15 |

**Learning objectives.** After this guide you should be able to:

1. Call a REST API with `requests.post`, and explain `data=` vs `json=`.
2. Design tool functions and schemas, including **optional parameters with defaults**.
3. Compare three dispatch strategies and explain why `globals()` dispatch is dangerous without an allowlist.
4. Explain exactly how Lab 4's `chat()` differs from Lab 3's, and why it is better.
5. Split a notebook into `context.py`, `tools.py`, `app.py` (separation of concerns).
6. Describe how to deploy a Gradio app and where secrets must (and must not) live.
7. Name and fix the robustness and security gaps in the notebook.

---

## 1. The Big Picture

### 1.1 What is built

```
 Visitor ◄──► Gradio ChatInterface ◄──► chat() ──► LLM (system prompt + 2 tools)
                                          │
                       finish_reason == "tool_calls" ?
                                          │ yes
                                          ▼
                              handle_tool_calls(tool_calls)
                                  │                    │
                    record_user_details()    record_unknown_question()
                                  └──────┬─────────────┘
                                         ▼
                                   push(message)
                                         │  HTTP POST
                                         ▼
                                  Pushover API ──► your phone 📱
```

The twin now **acts in the real world**: when a visitor shares contact details, or asks something the twin cannot answer, **you get a phone notification**.

### 1.2 What changed from Lab 3

| Lab 3 | Lab 4 |
|---|---|
| One tool writing to `emails.txt` | **Two tools** sending push notifications |
| Tool logic inline in `chat()` | Separate `handle_tool_calls()` function |
| Hard-coded handling for the email tool | **Name-based dispatch** (`if/elif`, then `globals()`) |
| Notebook only | **Modules** + deployment |
| Tool use implied by prompt | Prompt **explicitly instructs** when to use each tool |
| Local only | Deployable (HF Spaces / Render / fly.io) |

### 1.3 Glossary

| Term | Meaning |
|---|---|
| **Pushover** | Service that sends push notifications to your devices through a simple HTTP API. |
| **User key / App token** | Pushover's two credentials: *who* receives (user) and *which application* sends (token). |
| **REST API** | Web API using HTTP verbs (GET/POST) and URLs. |
| **`requests`** | Python library for HTTP calls. |
| **Dispatch** | Choosing which function to run based on a name. |
| **`globals()`** | Built-in returning a dict of the current module's global names to objects. |
| **Allowlist / registry** | An explicit dict of permitted tool names to functions. |
| **Separation of concerns** | Each module has one job. |
| **`uv`** | Fast Python package/project manager (`uv run app.py` runs the script in the project environment). |
| **Secret** | A credential stored in the platform's environment, never in code or git. |
| **PaaS** | Platform-as-a-service (HF Spaces, Render, fly.io) that runs your app for you. |

---

## 2. Cell-by-Cell Map

| # | Type | What it does | Concept |
|---|---|---|---|
| 1 | MD | Pushover setup steps; note that free HF Spaces hosting changed | Account, keys, `.env` |
| 2 | Code | Imports add `requests` (and `os`) | HTTP |
| 3 | Code | `load_dotenv`; `openai = OpenAI()` | Setup |
| 4 | Code | Read `PUSHOVER_USER`/`TOKEN`; check prefixes `u` / `a` | Credential sanity check |
| 5 | Code | `def push(message)`: POST to Pushover | The real-world side effect |
| 6 | Code | `push("HEY!!")` | Test your phone |
| 7 | Code | `record_user_details(email, name=..., notes=...)` | Tool 1 |
| 8 | Code | `record_unknown_question(question)` | Tool 2 |
| 9–10 | Code | Two JSON schemas | Tool descriptions |
| 11–12 | Code | `tools = [...]`; inspect | Tool list |
| 13 | Code | `handle_tool_calls_with_manual_if` | Dispatch by `if/elif` |
| 14 | MD | "Using Python built-in `globals()`" | Dynamic lookup idea |
| 15 | Code | `globals()["record_unknown_question"]("...")` | Calling by string name |
| 16 | Code | `handle_tool_calls` using `globals().get(tool_name)` | Dispatch without `if` |
| 17 | Code | Load PDF + summary | Context (as Lab 3) |
| 18 | Code | New `system_prompt` with tool instructions | Prompt engineering |
| 19 | Code | `chat()` using `handle_tool_calls` | Cleaner agent loop |
| 20 | Code | `gr.ChatInterface(chat).launch(inbrowser=True)` | UI |
| 21 | MD | Turning into modules (`context.py`, `tools.py`, `app.py`, `styles.py`); `uv run app.py` | Code organization |
| 22 | MD | "STOP THE PRESS": free HF Spaces deployment no longer available; Render alternative; fly.io twin | Deployment options |
| 23 | MD | Original HF Spaces instructions | `gradio deploy`, tokens, secrets |
| 24 | MD | Exercise ideas | Extensions |
| 25 | MD | Commercial implications | Domain-expert assistants that can act |

> **About deployment notes.** The notebook states that free Gradio hosting on HuggingFace Spaces stopped on a certain date and points to a `RENDER_INSTRUCTIONS.md` file. I can't verify platform policy or see that file; hosting offers change often, so treat §8 as the **general principles** that hold on any platform, and follow the current instructions from your course repo or the platform's docs for exact steps.
> Model names (`gpt-5.4-mini`) are used as written and I can't verify them.

---

## 3. Part A: Pushover and HTTP APIs

### 3.1 What Pushover does and why it was chosen

Pushover turns an HTTP request into a phone notification. It's ideal for agents because:

- One POST request, no complex SDK.
- Immediate, human-visible feedback that a tool actually did something.
- Free tier for personal use.

It illustrates the general pattern: **an agent tool = a Python function that calls some external API.** Replace Pushover with Slack, email, a CRM, or a database and the agent architecture is unchanged.

### 3.2 Credentials: two keys, two roles

```
PUSHOVER_USER   = identifies the recipient (starts with "u")
PUSHOVER_TOKEN  = identifies the application/sender (starts with "a")
```

```python
pushover_user  = os.getenv("PUSHOVER_USER")
pushover_token = os.getenv("PUSHOVER_TOKEN")
pushover_url   = "https://api.pushover.net/1/messages.json"
```

The check cell prints whether each exists and has the expected first letter:

```python
if pushover_user:
    if pushover_user.startswith("u"):
        print("Pushover user found and looks good")
    else:
        print("Pushover user found but doesn't start with u")
else:
    print("Pushover user not found")
```

This is a **three-way check** (missing / wrong shape / OK) using nested conditions. It catches the most common mistake: **swapping the two values**. Note it confirms only the *shape*; it does not prove the credentials are valid.

### 3.3 The `push` function

```python
def push(message):
    print(f"Push: {message}")
    payload = {"user": pushover_user, "token": pushover_token, "message": message}
    requests.post(pushover_url, data=payload)
```

Dissection:

| Line | Meaning |
|---|---|
| `print(...)` | Local log, so you can see attempts even if the push fails |
| `payload = {...}` | The three fields the API expects |
| `requests.post(url, data=payload)` | Sends an HTTP POST with the dict **form-encoded** (`application/x-www-form-urlencoded`) |

**`data=` vs `json=`:**

| | `data=dict` | `json=dict` |
|---|---|---|
| Body encoding | Form-encoded `a=1&b=2` | JSON `{"a":1,"b":2}` |
| `Content-Type` | `application/x-www-form-urlencoded` | `application/json` |
| Use when | API expects form fields (Pushover) | API expects JSON (most modern APIs, OpenAI-style) |

**What the notebook's `push` does *not* do:**

- It ignores the **response**: no `status_code` check, so a failed push (bad token, network issue) looks identical to success.
- No **timeout**: `requests` waits indefinitely by default; a hung network stalls the chat turn.
- No **exception handling**: a connection error raises and breaks the tool call.

A robust version:

```python
def push(message: str) -> bool:
    print(f"Push: {message}")
    try:
        r = requests.post(pushover_url,
                          data={"user": pushover_user, "token": pushover_token, "message": message},
                          timeout=10)
        r.raise_for_status()          # raises on 4xx/5xx
        return True
    except requests.RequestException as e:
        print(f"Push failed: {e}")
        return False
```

### 3.4 Test the tool before the LLM

`push("HEY!!")` is the same discipline as Lab 3: **test each tool in isolation**. If the phone buzzes, the plumbing works, and any later failure is the agent's logic, not the API.

---

## 4. Part B: Two Real Tools and Their Schemas

### 4.1 The functions

```python
def record_user_details(email, name="Name not provided", notes="not provided"):
    push(f"Recording interest from {name} with email {email} and notes {notes}")
    return "OK"

def record_unknown_question(question):
    push(f"Recording {question} asked that I couldn't answer")
    return "OK"
```

Design observations:

- **Default arguments** (`name=...`, `notes=...`) make those parameters **optional** at the Python level.
- Both return `"OK"`: the string that becomes the **tool result** the model sees.
- The "recording" is just a push notification; a real system might also write to a database. A push is **ephemeral**; if you miss it, the data is gone. (Exercise idea: persist to SQL, §11.2.)

### 4.2 The schemas

```python
record_user_details_json = {
    "name": "record_user_details",
    "description": "Use this tool to record that a user is interested in being in touch and provided an email address",
    "parameters": {
        "type": "object",
        "properties": {
            "email": {"type": "string", "description": "The email address of this user"},
            "name":  {"type": "string", "description": "The user's name, if they provided it"},
            "notes": {"type": "string", "description": "Any additional info about the conversation that's worth recording to give context"}
        },
        "required": ["email"],
        "additionalProperties": False
    }
}

record_unknown_question_json = {
    "name": "record_unknown_question",
    "description": "Always use this tool to record any question that couldn't be answered as you didn't know the answer",
    "parameters": {
        "type": "object",
        "properties": {"question": {"type": "string", "description": "The question that couldn't be answered"}},
        "required": ["question"],
        "additionalProperties": False
    }
}

tools = [{"type": "function", "function": record_user_details_json},
         {"type": "function", "function": record_unknown_question_json}]
```

**Matching schema to function** (a source of bugs; check every time):

| Schema | Python signature | Match? |
|---|---|---|
| `required: ["email"]` | `email` has no default | ✅ required both sides |
| `name`, `notes` not required | defaults provided | ✅ optional both sides |
| Property names `email`, `name`, `notes` | parameter names identical | ✅ **must be identical**, because the call is `fn(**arguments)` |
| `additionalProperties: False` | Python would raise `TypeError` on unexpected keyword | consistent |

If you add a property to the schema but not the function (or rename one), `fn(**arguments)` raises `TypeError: unexpected keyword argument`.

### 4.3 Descriptions are prompts

| Description wording | Effect |
|---|---|
| "Use this tool to record that a user is interested in being in touch and provided an email address" | Condition: user wants contact **and** gave an email |
| "**Always** use this tool to record any question that couldn't be answered..." | Strong imperative; pushes the model to call it whenever it can't answer |
| `notes`: "...worth recording to give context" | Lets the model add useful context for you |

The system prompt **and** the schema both mention when to use the tools. Redundancy reinforces behavior, but the two must **agree**; contradictions confuse the model.

---

## 5. Part C: Tool Dispatch: `if` vs `globals()` vs Registry

**Dispatch** = mapping the string `tool_call.function.name` to the right Python function.

### 5.1 Strategy 1: manual `if/elif`

```python
def handle_tool_calls_with_manual_if(tool_calls):
    results = []
    for tool_call in tool_calls:
        tool_name = tool_call.function.name
        arguments = json.loads(tool_call.function.arguments)
        print(f"Tool called: {tool_name}", flush=True)

        if tool_name == "record_user_details":
            result = record_user_details(**arguments)
        elif tool_name == "record_unknown_question":
            result = record_unknown_question(**arguments)

        results.append({"role": "tool", "content": json.dumps(result), "tool_call_id": tool_call.id})
    return results
```

Key syntax: **`**arguments`** (dict unpacking) turns `{"email": "a@b.com", "name": "Sam"}` into `record_user_details(email="a@b.com", name="Sam")`.

**Latent bug:** there is **no `else`**. If the model returns a tool name that matches neither branch, `result` is either **undefined** (first iteration ⇒ `UnboundLocalError`) or **stale from the previous loop iteration** (it would silently send the previous tool's result under the wrong `tool_call_id`). Always add an `else` that returns an error string.

### 5.2 `json.dumps(result)`

The tool message's `content` must be a **string**. `json.dumps("OK")` yields the string `"\"OK\""` (with literal quotes). That's valid, but for plain strings you could pass the string directly. `json.dumps` becomes *useful* when a tool returns a dict/list (for example `{"status": "ok", "id": 17}`), because it serializes structure for the model to read.

### 5.3 Strategy 2: `globals()` lookup

```python
globals()["record_unknown_question"]("this is a really hard question")
```

`globals()` returns a dict `{name: object}` of everything defined at module level (functions, variables, imports). Indexing it by a string name retrieves the function object, and `(...)` calls it.

```python
def handle_tool_calls(tool_calls):
    results = []
    for tool_call in tool_calls:
        tool_name = tool_call.function.name
        arguments = json.loads(tool_call.function.arguments)
        print(f"Tool called: {tool_name}", flush=True)
        tool = globals().get(tool_name)
        result = tool(**arguments) if tool else "No tool found"
        results.append({"role": "tool", "content": json.dumps(result), "tool_call_id": tool_call.id})
    return results
```

Walkthrough:

1. `globals().get(tool_name)` returns the function, or `None` if the name isn't defined (`.get` avoids `KeyError`).
2. `tool(**arguments) if tool else "No tool found"` is a **conditional expression**: call the tool if it exists, otherwise produce a fallback string. This fixes the missing-`else` bug.
3. A tool message is appended for **every** call (protocol rule from Lab 3).

**Advantage:** adding a tool = define the function + its schema. **No edits to the dispatcher** (open/closed principle).

### 5.4 ⚠️ The security problem (the notebook hints at it)

> "Sidenote: for sure when we deploy, we will use this in a more protected way.."

`globals()` contains **everything** in the module: `push`, `chat`, `load_dotenv`, `PdfReader`, `OpenAI`, helper functions, and anything else you define later, not only your intended tools. The tool **name** comes from model output, which can be steered by a malicious visitor (prompt injection). So a visitor might induce the model to request calls to functions you never meant to expose:

| Name requested | Possible harm |
|---|---|
| `push` | Spam your phone with arbitrary text (quota/notification abuse), since `push(message=...)` is callable with attacker-chosen text |
| `PdfReader` | Open arbitrary file paths with attacker-chosen arguments |
| `load_dotenv` | Reload/override environment variables from a path chosen via arguments |
| any future function that deletes, sends, or spends | Much worse |

**Principle: never let model output select arbitrary code.** Use an **allowlist (registry)**.

### 5.5 Strategy 3: explicit registry (recommended)

```python
TOOL_FUNCTIONS = {
    "record_user_details": record_user_details,
    "record_unknown_question": record_unknown_question,
}

def handle_tool_calls(tool_calls):
    results = []
    for tc in tool_calls:
        name = tc.function.name
        try:
            args = json.loads(tc.function.arguments or "{}")
            fn = TOOL_FUNCTIONS.get(name)
            output = fn(**args) if fn else f"Error: unknown tool '{name}'."
        except TypeError as e:                       # wrong/missing/extra arguments
            output = f"Error: bad arguments for {name}: {e}"
        except Exception as e:
            output = f"Error while running {name}: {e}"
        results.append({"role": "tool", "tool_call_id": tc.id, "content": json.dumps(output)})
    return results
```

You can even **generate** the `tools` list and the registry from one decorator so schema and function never drift apart (a pattern frameworks use).

### 5.6 Comparison

| | `if/elif` | `globals()` | Registry dict |
|---|---|---|---|
| Add a tool | Edit dispatcher | Just define function | Add one dict entry |
| Safety | Safe (explicit) | **Unsafe** (exposes all globals) | **Safe** (allowlist) |
| Unknown name | Bug unless `else` | Handled by `.get` | Handled by `.get` |
| Readability | Clear, verbose | Compact, "magic" | Clear, compact |
| Used in production | Small apps | Demos only | **Yes** |

---

## 6. Part D: The Upgraded System Prompt and `chat()`

### 6.1 New rules added to the system prompt

```
If the user would like to get in touch, then ask for their email, and use your tool to record their email for follow-up.

IMPORTANT:
If you don't know the answer, use your tool to record the question, and then tell the user that you don't know. Never make up an answer.
```

Compare with Lab 3's "If you don't know, say so": Lab 4 turns "don't know" into an **actionable workflow**: *record it, then admit it*. This implements a **feedback loop**: the owner learns which questions the context lacks and can improve `summary.txt` / LinkedIn data. It's the cheapest form of **continuous improvement for an agent**.

Prompt-engineering points:

| Instruction | Technique |
|---|---|
| "ask for their email, and use your tool" | **Conditional workflow** tied to a tool |
| "use your tool to record the question, **and then** tell the user" | **Ordered steps** (act, then respond) |
| Reduced scope sentence ("Only answer questions related to career...") | Tighter guardrail than Lab 3 |
| `IMPORTANT:` | Emphasis for the highest-priority rule |

### 6.2 The cleaner `chat()`

```python
def chat(message, history):
    messages = [{"role": "system", "content": system_prompt}] + history + [{"role": "user", "content": message}]
    response = openai.chat.completions.create(model="gpt-5.4-mini", messages=messages, tools=tools)
    while response.choices[0].finish_reason == "tool_calls":
        message = response.choices[0].message
        tool_calls = message.tool_calls
        results = handle_tool_calls(tool_calls)
        messages.append(message)
        messages.extend(results)
        response = openai.chat.completions.create(model="gpt-5.4-mini", messages=messages, tools=tools)
    return response.choices[0].message.content
```

Differences from Lab 3's `chat()`:

| Aspect | Lab 3 | Lab 4 |
|---|---|---|
| Tool execution | Inline `for` with hard-coded email extraction | Delegated to `handle_tool_calls()` |
| Appending results | One `append` per call inside the loop | `messages.extend(results)` (list of tool messages) |
| Tools supported | One | Any number |
| Readability | Loop mixes orchestration and tool logic | **Orchestration only** |

`append` adds **one element**; `extend` adds **all elements of a list**. Using `append(results)` would nest a list inside the list, which is a bug.

**Order matters:** `messages.append(message)` (assistant with `tool_calls`) must come **before** `messages.extend(results)` (tool messages). The notebook calls `handle_tool_calls` first (so the tools run), but appends in the correct protocol order.

The `while` is still **unbounded** (no max steps); see §10.

### 6.3 Message flow example

Visitor: *"Can you tell me your favorite programming language? Also, email me at sam@example.com, I'm Sam."*

```
system, user
  → assistant: tool_calls [ record_unknown_question(question="favorite programming language"),
                            record_user_details(email="sam@example.com", name="Sam") ]   ← parallel calls
  → tool (id1): "\"OK\""
  → tool (id2): "\"OK\""
  → assistant: "I don't have that information, but I've noted your email, Sam..."
```

Your phone buzzes twice. Both calls arrive in **one assistant message**, which is why the `for` loop from Lab 3 mattered.

---

## 7. Part E: From Notebook to Modules

### 7.1 Why leave the notebook

| Notebook | Modules |
|---|---|
| Great for **experiments** and explanation | Needed for **deployment** (a platform runs a script, not a notebook) |
| Hidden state, cell-order bugs | Clear import graph, reproducible |
| Hard to test and reuse | Unit-testable functions |
| One giant namespace | Separation of concerns |

The notebook's advice: **experiment in the notebook first, then refactor into modules.**

### 7.2 The four modules

| File | Responsibility |
|---|---|
| `context.py` | Load static data (PDF, summary) and **construct the system prompt** |
| `tools.py` | Tool functions, **their JSON schemas**, the `tools` list, and the dispatcher |
| `app.py` | Gradio app and the OpenAI call (`chat()`), the entry point |
| `styles.py` | Visual styling for Gradio (per the notebook, written by an AI coding assistant) |

### 7.3 Dependency direction

```
app.py ──imports──► context.py   (system_prompt)
   │
   └─────imports──► tools.py     (tools, handle_tool_calls)
   └─────imports──► styles.py    (CSS/theme)

context.py and tools.py do NOT import app.py   (no circular imports)
```

Rule: **dependencies point inward toward the entry point**; low-level modules know nothing about the app.

### 7.4 A sketch of the modules

```python
# context.py
from pypdf import PdfReader

def load_text() -> tuple[str, str]:
    reader = PdfReader("twin/linkedin.pdf")
    linkedin = "\n".join(p.extract_text() or "" for p in reader.pages)
    with open("twin/summary.txt", "r", encoding="utf-8") as f:
        summary = f.read()
    return linkedin, summary

def build_system_prompt() -> str:
    linkedin, summary = load_text()
    return f"""# Your role ... {summary} ... {linkedin} ... # Rules ..."""

system_prompt = build_system_prompt()
```

```python
# tools.py
import json, os, requests

PUSH_URL = "https://api.pushover.net/1/messages.json"

def push(message): ...                      # as §3.3 (robust version)
def record_user_details(email, name="Name not provided", notes="not provided"): ...
def record_unknown_question(question): ...

record_user_details_json = {...}
record_unknown_question_json = {...}
tools = [{"type": "function", "function": record_user_details_json},
         {"type": "function", "function": record_unknown_question_json}]

TOOL_FUNCTIONS = {"record_user_details": record_user_details,
                  "record_unknown_question": record_unknown_question}

def handle_tool_calls(tool_calls): ...      # registry version, §5.5
```

```python
# app.py
from dotenv import load_dotenv
from openai import OpenAI
import gradio as gr
from context import system_prompt
from tools import tools, handle_tool_calls

load_dotenv(override=True)
client = OpenAI()

def chat(message, history):
    history = [{"role": h["role"], "content": h["content"]} for h in history]
    messages = [{"role": "system", "content": system_prompt}] + history + [{"role": "user", "content": message}]
    for _ in range(5):                                     # bounded agent loop
        resp = client.chat.completions.create(model="gpt-5.4-mini", messages=messages, tools=tools)
        choice = resp.choices[0]
        if choice.finish_reason != "tool_calls":
            return choice.message.content
        messages.append(choice.message)
        messages.extend(handle_tool_calls(choice.message.tool_calls))
    return "Sorry, something went wrong. Please try again."

if __name__ == "__main__":
    gr.ChatInterface(chat).launch()
```

`if __name__ == "__main__":` runs the launch **only when the file is executed directly**, not when imported (for tests, for example).

### 7.5 Running with `uv`

```
cd 1_foundations
cd twin
uv run app.py
```

`uv run` executes the script inside the project's managed virtual environment (installing dependencies from the project file if needed). It replaces manually activating `.venv` and running `python app.py`.

**Path caveat:** the code opens `twin/linkedin.pdf` with a **relative path**. It resolves against the **current working directory**, so the notebook (run from one folder) and `uv run app.py` (run from `twin/`) may need *different* relative paths. Robust fix:

```python
from pathlib import Path
BASE = Path(__file__).parent
linkedin_path = BASE / "linkedin.pdf"
```

---

## 8. Part F: Deployment and Secrets

### 8.1 What deployment actually means

Your laptop is not a server. Deployment = running the app on a **always-on machine** with a **public URL**, with the right **secrets** and **start command**.

```
git push / deploy tool ──► platform builds environment ──► runs `python app.py` ──► public HTTPS URL
                                  ▲
                      secrets injected as environment variables
```

### 8.2 The notebook's options (as stated)

| Option | Notes (per the notebook) |
|---|---|
| **HuggingFace Spaces** | Original course method (`gradio deploy`); the notebook says free hosting for Gradio apps is no longer available, paid still possible |
| **Render** | Free alternative; step-by-step in the repo's `RENDER_INSTRUCTIONS.md` |
| **fly.io** | The instructor's own twin runs there at very low cost |

### 8.3 HuggingFace Spaces flow (original instructions, explained)

1. **Account + write-permission token.** `uvx hf auth login --token ...` authenticates the CLI; `uvx hf auth whoami` verifies. (`uvx` runs a tool in a temporary environment.) The token needs **write** permission because deployment *pushes files* to your Space repo.
2. **`HF_TOKEN` in `.env`** for later use.
3. **`uv run gradio deploy`** from the app directory: pick a name, the entry file (`app.py`), hardware (`cpu-basic` is enough since the heavy computation happens at OpenAI, not locally), and decline secrets and GitHub Actions at this step.
4. **Secrets** in the Space settings: `OPENAI_API_KEY`, `PUSHOVER_USER`, `PUSHOVER_TOKEN` (use **New Secret**, not *Variable*; variables are visible, secrets are not). Restart the Space.
5. Delete any pre-existing `README.md` in the app folder: the deploy tool generates one (conflict otherwise).

### 8.4 Platform-agnostic principles (apply to Render, fly.io, anything)

| Concern | Rule |
|---|---|
| **Secrets** | Set as environment variables in the **platform dashboard**. **Never** commit `.env`. Add `.env` to `.gitignore`. A committed key must be **rotated**, even if you delete the commit. |
| **Start command** | Something like `python app.py` (or `uv run app.py`). |
| **Network binding** | Cloud containers must listen on `0.0.0.0` and often on a port given by an env var. For Gradio: `launch(server_name="0.0.0.0", server_port=int(os.getenv("PORT", 7860)))`. Binding to `127.0.0.1` makes the app unreachable from outside the container. |
| **Dependencies** | Provide a `requirements.txt` / `pyproject.toml` so the platform can install them. |
| **Data files** | `twin/linkedin.pdf` and `summary.txt` must be **included in the deployment** (and be OK to be public-ish, see §10 on PII). |
| **Free-tier behavior** | Free instances may **sleep** after inactivity ⇒ **cold start** delay on the first visit. |
| **Ephemeral disk** | Writing to local files (like Lab 3's `emails.txt`) may be lost on restart. This is **why Lab 4 uses push notifications** (or should use a database). |
| **Logging** | Use the platform's log viewer: your `print` statements (with `flush=True`) appear there. |
| **Embedding** | Gradio apps can be embedded via an iframe/web component, so the twin can live on your personal site. |

### 8.5 Troubleshooting mindset

1. Read the **platform logs** first.
2. Add temporary prints around key loading (prefix only, never the whole secret).
3. Confirm secrets' **names match exactly** what the code reads (`os.getenv("PUSHOVER_USER")`).
4. Redeploy or restart after changing secrets (they're read at process start).

---

## 9. Part G: Production Concerns and Math

### 9.1 Notification volume and alert fatigue

Let $V$ be visitors per day, $p_e$ the fraction that leave an email, and $p_u$ the average number of "unknown question" pushes per visitor. Expected pushes per day:

$$E[\text{pushes}] = V\,(p_e + p_u)$$

**Worked example.** $V=200$, $p_e=0.02$, $p_u=0.3$: $200\,(0.02+0.30)=64$ pushes/day. Most are low-value unknown-question notifications; the valuable ones (emails) are ~4. **Signal drowns in noise**, which is why you'd batch unknown questions into a daily digest or a database and keep push only for high-value events.

### 9.2 Precision of the "unknown question" tool

Suppose of the model's unknown-question calls, a fraction are genuine gaps. Define precision:

$$\text{Precision} = \frac{TP}{TP + FP}$$

If in a test of 50 calls, 30 were real gaps (TP) and 20 were answerable-from-context but the model gave up (FP): $\text{Precision} = 30/50 = 0.6$. Low precision ⇒ noisy notifications **and** a sign the model is under-using its context.

### 9.3 Cost with the abuse ceiling

If a public bot allows each visitor session at most $T$ turns and each call is capped at $m$ output tokens, the worst-case output tokens per session is bounded by $T\,m\,(1+r)$ where $r$ is the max tool rounds per turn. Without a cap, a single abusive client can run unbounded. **Always set**: max turns, `max_tokens`, a loop cap, and a provider spending limit.

Session input cost grows quadratically with history (from Lab 3):

$$\sum_{t=1}^{T} N_{in}(t) = T\,S + u\,T + (u+a)\,\frac{T(T-1)}{2}$$

so truncating or summarizing history also bounds cost.

### 9.4 Reliability of the push step

If the Pushover call succeeds with probability $q=0.99$, then over $n$ notifications the probability that **all** arrive is $q^n$. For $n=64$: $0.99^{64}\approx 0.526$. Roughly **half the days** you lose at least one notification, so important events (emails) must be **persisted** (database/log) *and* pushed, with retries.

### 9.5 Latency

Each tool round adds a model call, and each call re-sends the full prompt. A turn with $r$ tool rounds takes roughly $(r+1)\cdot L_{\text{LLM}} + r\cdot L_{\text{tool}}$. A slow, un-timed-out `push` directly increases user-visible latency, which is another reason for `timeout=10` (or running pushes **asynchronously** in the background).

---

## 10. Part H: Weaknesses and a Hardened Version

### 10.1 Issues

| # | Issue | Why it matters | Fix |
|---|---|---|---|
| 1 | **`globals()` dispatch** | Model output can invoke any global callable | Allowlist registry |
| 2 | **Manual-`if` version has no `else`** | `UnboundLocalError` or stale result under the wrong id | `else` branch with error string |
| 3 | **`push()` ignores errors and has no timeout** | Silent failures; hangs | `timeout`, `raise_for_status`, `try/except` |
| 4 | **Tool failure crashes the chat** | Visitor sees an error page | Catch errors, return text to the model |
| 5 | **`fn(**arguments)` with unexpected/missing args** | `TypeError` | Validate against schema, catch `TypeError` |
| 6 | **No input validation** (email shape, length) | Garbage/spam pushes; **injection into the push text** | Regex, max lengths, strip control characters |
| 7 | **Visitor data sent to a third party** (Pushover) | Names, emails, and questions leave your system | Disclose purpose; minimize data; privacy policy |
| 8 | **Unbounded `while` loop** | Runaway cost | Max steps |
| 9 | **Prompt injection** (visitor text; also *document* text such as the PDF) | Could trigger tool spam or leak the prompt | Delimit data (`<linkedin>` tags), evaluator, least privilege |
| 10 | **Public bot, no rate limiting** | Cost abuse; notification flooding | Per-IP/session limits, daily budget |
| 11 | **Prompt contains the whole résumé** | Anyone can ask for it verbatim | Include only publishable facts |
| 12 | **Relative file paths** | Break when launched from another directory | `Path(__file__).parent` |
| 13 | **Ephemeral storage on free tiers** | Local files lost on restart | Database / external store |
| 14 | **No logging/observability** | Can't see failures or misuse | Structured logs, tracing |
| 15 | **No history whitelist** (other providers) | 400 errors with extra keys | Strip to `role`/`content` |
| 16 | **Push sent synchronously** | Slows replies | Background thread/queue |
| 17 | **Duplicate notifications** | Model may call the same tool twice | Idempotency/dedupe |
| 18 | **`json.dumps` of plain strings** | Quoted strings; harmless but confusing | Return strings directly or dicts |

### 10.2 Hardened `tools.py`

```python
import json, os, re, requests
from threading import Thread

PUSH_URL = "https://api.pushover.net/1/messages.json"
EMAIL_RE = re.compile(r"^[^@\s]+@[^@\s]+\.[^@\s]+$")

def _clean(text: str, limit: int = 300) -> str:
    return re.sub(r"[\r\n\t]+", " ", str(text))[:limit]      # strip control chars, bound length

def push(message: str) -> None:
    def _send():
        try:
            r = requests.post(PUSH_URL, timeout=10, data={
                "user": os.getenv("PUSHOVER_USER"),
                "token": os.getenv("PUSHOVER_TOKEN"),
                "message": message})
            r.raise_for_status()
        except requests.RequestException as e:
            print(f"Push failed: {e}", flush=True)
    Thread(target=_send, daemon=True).start()                # non-blocking

def record_user_details(email, name="Name not provided", notes="not provided"):
    if not EMAIL_RE.match(email or ""):
        return {"status": "error", "message": "Invalid email address; ask the user to re-enter it."}
    push(f"Interest from {_clean(name, 80)} <{_clean(email, 120)}> notes: {_clean(notes)}")
    return {"status": "ok"}

def record_unknown_question(question):
    push(f"Unanswered question: {_clean(question)}")
    return {"status": "ok"}

TOOL_FUNCTIONS = {"record_user_details": record_user_details,
                  "record_unknown_question": record_unknown_question}

def handle_tool_calls(tool_calls):
    results = []
    for tc in tool_calls:
        name = tc.function.name
        try:
            args = json.loads(tc.function.arguments or "{}")
            fn = TOOL_FUNCTIONS.get(name)
            output = fn(**args) if fn else {"status": "error", "message": f"unknown tool {name}"}
        except Exception as e:
            output = {"status": "error", "message": f"{type(e).__name__}: {e}"}
        results.append({"role": "tool", "tool_call_id": tc.id, "content": json.dumps(output)})
    return results
```

Notes: returning **structured results** (`{"status": ...}`) lets the model react correctly (for example, ask for a corrected email). Errors are **returned**, not raised. `daemon=True` threads don't block shutdown. In a serverless environment, background threads may be killed after the response, so prefer a short-timeout synchronous call there.

---

## 11. Part I: Exercise Solutions

The notebook's exercises: **(1) deploy it**; **(2) improve the resources / add a knowledge base (RAG)**; **(3) add more tools (e.g., a SQL Q&A database)**; **(4) bring in the evaluator and other patterns**; **(5) Telegram integration**.

### 11.1 Exercise 1: Deployment checklist

- [ ] Replace `linkedin.pdf` and `summary.txt` with *your* public-safe data.
- [ ] `.env` is in `.gitignore`; no keys in code.
- [ ] `launch(server_name="0.0.0.0", server_port=int(os.getenv("PORT", 7860)))` for cloud.
- [ ] Dependencies declared (`pyproject.toml` / `requirements.txt`).
- [ ] Secrets set in the platform: `OPENAI_API_KEY`, `PUSHOVER_USER`, `PUSHOVER_TOKEN`.
- [ ] Provider spending limit set.
- [ ] Test: normal question, unknown question (phone buzzes), email submission (phone buzzes).
- [ ] Check logs for errors; share the URL.

### 11.2 Exercise 3: SQL-backed Q&A tools (persistent memory for the twin)

Two tools: **look up** known Q&A, and **save** new answered/unanswered questions. Use parameterized queries (never string-format SQL).

```python
import sqlite3

DB = "twin_qa.db"

def _conn():
    c = sqlite3.connect(DB)
    c.execute("""CREATE TABLE IF NOT EXISTS qa (
                   id INTEGER PRIMARY KEY AUTOINCREMENT,
                   question TEXT NOT NULL,
                   answer TEXT,
                   asked_count INTEGER DEFAULT 1)""")
    return c

def lookup_qa(keyword: str):
    with _conn() as c:
        rows = c.execute(
            "SELECT question, answer FROM qa WHERE answer IS NOT NULL AND question LIKE ? LIMIT 3",
            (f"%{keyword}%",)).fetchall()                    # ? placeholder: prevents SQL injection
    return {"matches": [{"question": q, "answer": a} for q, a in rows]}

def log_question(question: str):
    with _conn() as c:
        row = c.execute("SELECT id FROM qa WHERE question = ?", (question,)).fetchone()
        if row:
            c.execute("UPDATE qa SET asked_count = asked_count + 1 WHERE id = ?", (row[0],))
        else:
            c.execute("INSERT INTO qa (question) VALUES (?)", (question,))
    return {"status": "ok"}

TOOL_FUNCTIONS.update({"lookup_qa": lookup_qa, "log_question": log_question})
```

Add matching schemas (`name`, `description`: "Search previously answered questions by keyword", `parameters` with a `keyword` string) to `tools`. Because dispatch uses the registry, **the agent loop is unchanged**.

Why parameterized SQL: `LIKE ?` with a tuple sends the value separately from the SQL text, so input like `'; DROP TABLE qa; --` is treated as data, not code. **Never** build SQL with f-strings from model or user text.

Design benefit: you can later review `asked_count` to see the most common unanswered questions and then **improve `summary.txt`**: a closed improvement loop. Also, on ephemeral hosting, use a hosted database instead of a local SQLite file.

### 11.3 Exercise 2: A tiny RAG knowledge base about you

When your knowledge grows beyond a few pages, stop stuffing and retrieve.

```python
import numpy as np

def chunk(text, size=800, overlap=100):
    out, i = [], 0
    while i < len(text):
        out.append(text[i:i+size]); i += size - overlap
    return out

def embed(texts):
    r = client.embeddings.create(model="text-embedding-3-small", input=texts)   # check your provider's embedding model name
    return np.array([d.embedding for d in r.data])

docs   = chunk(linkedin + "\n" + summary + "\n" + extra_notes)
doc_vecs = embed(docs)
doc_vecs /= np.linalg.norm(doc_vecs, axis=1, keepdims=True)    # normalize once

def retrieve(query, k=3):
    q = embed([query])[0]; q /= np.linalg.norm(q)
    scores = doc_vecs @ q                                      # cosine similarity (unit vectors)
    return [docs[i] for i in np.argsort(-scores)[:k]]
```

Cosine similarity between vectors $a$ and $b$:

$$\cos(a,b) = \frac{a\cdot b}{\lVert a\rVert\,\lVert b\rVert}$$

Normalizing vectors to unit length makes cosine similarity equal to a plain dot product, so one matrix-vector multiply scores every chunk.

**Worked example.** $a=(1,2)$, $b=(2,1)$: $a\cdot b=4$; $\lVert a\rVert=\lVert b\rVert=\sqrt5$; $\cos=4/5=0.8$.

Integrate either as a **pre-step** (retrieve, then inject the top-$k$ chunks into the prompt) or as a **tool** (`search_knowledge_base(query)`) so the model decides when to retrieve (**agentic RAG**). Token savings: with $S_{\text{full}}=3000$ tokens stuffed versus about $3\times200=600$ retrieved, you pay roughly 5× less context per call.

### 11.4 Exercise 4: Add the evaluator and other patterns

- Reuse the **evaluator-optimizer** from Lab 3 §11.1 around `chat()`. Put evaluation **after** the tool loop and only on the final reply.
- **Routing**: a cheap classifier decides if the message is about career, contact, or off-topic, and picks a response path.
- **Reflection for unknown questions**: before calling `record_unknown_question`, do a second pass that re-searches the knowledge base (reduces false "unknown" pushes, improving precision from §9.2).
- **Guardrail on input**: block obvious injection attempts and over-long messages before the LLM sees them.

### 11.5 Exercise 5: Telegram integration (outline)

Idea: let the owner reply from Telegram and have the reply appear in the visitor's chat (human-in-the-loop handoff).

1. Create a bot via Telegram's BotFather to obtain a bot token; store it as a secret.
2. Outbound: your `push()` becomes a Telegram `sendMessage` call to your chat id (the bot HTTP API takes the token in the URL and a JSON body).
3. Inbound: poll the bot's updates endpoint (or use a webhook) for your replies.
4. Maintain a mapping `session_id → telegram thread`, and surface owner replies in the Gradio chat (for example, polling every few seconds).

Treat details as provider-specific; check Telegram's current Bot API documentation. This pattern is **human-in-the-loop**: the agent escalates when it cannot answer.

---

## 12. Common Errors and Debugging

| Symptom | Likely cause | Fix |
|---|---|---|
| "Pushover user found but doesn't start with u" | Swapped user key and app token | Swap the `.env` values |
| `push` runs but no notification | Invalid keys/token; ignored `status_code` | Print `r.status_code`/`r.text`; check Pushover app is installed on the device |
| `requests.exceptions.ConnectionError`/hang | Network or no timeout | `timeout=10`, retry |
| `UnboundLocalError: result` | Manual-`if` dispatcher with an unrecognized tool name, no `else` | Add `else` |
| Tool result attributed to the wrong call | Stale `result` variable reused | `else` branch; assign per call |
| `TypeError: unexpected keyword argument 'x'` | Schema property names differ from function parameters | Make them identical |
| `TypeError: missing 1 required positional argument` | Model omitted a required field | Mark `required`; catch `TypeError` and return an error to the model |
| `"No tool found"` returned | Tool name not defined/registered | Check the name in schema, function, and registry |
| `json.JSONDecodeError` | Malformed arguments | `try/except`, return error text |
| 400 error: tool message ids | Missing tool message for a call, or wrong order | Append assistant message first, one tool message per id |
| `ModuleNotFoundError` after refactor | Running from the wrong directory or wrong environment | Use `uv run` from the right folder; check imports |
| `FileNotFoundError: twin/linkedin.pdf` in `app.py` | Relative path differs from notebook | `Path(__file__).parent` |
| Works locally, blank/unreachable when deployed | Bound to localhost; wrong port | `server_name="0.0.0.0"` and `PORT` env var |
| Deployed app errors on first request | Missing secrets on the platform | Add secrets; restart |
| App slow on first visit | Free-tier cold start | Expected; upgrade or keep-alive |
| Local `emails.txt`/SQLite data vanishes | Ephemeral disk | Use an external database |
| Hit rate limit/bill spike | Public bot, no limits | Rate limit, spending cap, loop cap |
| Too many notifications | Unknown-question tool fires constantly | Improve context; batch; dedupe |

---

## 13. Comparison Tables

### 13.1 Lab 3 vs Lab 4

| | Lab 3 | Lab 4 |
|---|---|---|
| Tool effect | Append to a local file | Push to your phone (external API) |
| Tools | 1 | 2 (+ designed for many) |
| Dispatch | None (hard-coded) | `if/elif` → `globals()` → (registry) |
| Structure | One notebook | Modules + entry point |
| Deployment | None | HF Spaces / Render / fly.io |
| Prompt | Don't know → say so | Don't know → **record, then say so** |

### 13.2 `data=` vs `json=` in `requests`

| | `data=` | `json=` |
|---|---|---|
| Encoding | Form | JSON |
| Typical | Pushover, classic web forms | Modern JSON APIs |

### 13.3 `append` vs `extend`

| | `lst.append(x)` | `lst.extend(xs)` |
|---|---|---|
| Adds | One element `x` | Every element of `xs` |
| `append([1,2])` result | `[..., [1,2]]` (nested) | `extend([1,2])` → `[..., 1, 2]` |
| In this lab | Add the assistant `tool_calls` message | Add the list of tool results |

### 13.4 Where to deploy (conceptual)

| | HF Spaces | Render | fly.io | Your laptop |
|---|---|---|---|---|
| Cost (per the notebook) | No longer free for Gradio apps | Free alternative | Very low cost | Free |
| Always on | Yes | Free tiers may sleep | Yes | No |
| Secrets UI | Settings → Secrets | Environment variables | `fly secrets` | `.env` |
| Public URL | Yes | Yes | Yes | No (unless tunneled) |

### 13.5 Tool-effect durability

| Effect | Durable? | Visible? | Good for |
|---|---|---|---|
| `print` | No | Logs only | Debugging |
| Push notification | No | Immediate | Alerts |
| Local file | Not on ephemeral hosts | No | Local dev |
| Database | **Yes** | Query | Records of truth |

---

## 14. Exam Preparation

### 14.1 Conceptual questions with model answers

**Q1. What is the purpose of Pushover in this lab?**
It gives the agent a real-world side effect: when the twin records a contact or an unanswerable question, an HTTP POST triggers a phone notification, demonstrating that tools can act outside the chat.

**Q2. Why two credentials (user and token)?**
The user key identifies the recipient; the application token identifies the sending application. Both are required in the payload.

**Q3. Why `data=payload` in `requests.post` for Pushover?**
The API expects form-encoded fields; `data=` sends `application/x-www-form-urlencoded`, whereas `json=` would send a JSON body.

**Q4. Why do optional tool parameters need defaults in Python and to be absent from `required` in the schema?**
The model may omit optional fields; the function must still run. Schema and signature must agree, otherwise you get missing-argument or unexpected-keyword `TypeError`s.

**Q5. What does `**arguments` do?**
Unpacks a dict into keyword arguments, so `{"email": "a@b.c"}` becomes `fn(email="a@b.c")`. Keys must match the parameter names.

**Q6. How does `globals()` dispatch work, and why is it dangerous?**
`globals()` is a dict of module-level names; `.get(tool_name)` looks up a function by string. Since the name originates from model output (steerable by visitors), it can invoke any global callable (like `push` or `PdfReader`) with attacker-influenced arguments. Use an explicit allowlist registry.

**Q7. What bug exists in the manual `if/elif` dispatcher?**
No `else`: an unknown tool name leaves `result` undefined (error) or stale from a previous iteration, attaching the wrong result to a `tool_call_id`.

**Q8. Why must every tool call get a tool message even for unknown tools?**
The API requires a tool message for each `tool_call_id`. Return an error string rather than skipping.

**Q9. Why does the system prompt say to record the unknown question *and then* tell the user?**
It defines an ordered workflow: act first (so the owner learns about the gap) then respond honestly. It also builds a feedback loop for improving the knowledge base.

**Q10. Why are tool descriptions called prompts?**
The model chooses tools based only on their names and descriptions; wording such as "Always use this tool..." directly shapes behavior.

**Q11. Why refactor into modules, and what goes where?**
Deployment platforms run scripts, and modules give clarity, testability and separation of concerns: `context.py` builds the prompt, `tools.py` holds tools and dispatch, `app.py` the UI and LLM loop, `styles.py` the look.

**Q12. What does `if __name__ == "__main__":` protect against?**
Launching the web app when the module is merely imported (for tests or other modules).

**Q13. Why shouldn't secrets be committed, and where should they live in deployment?**
Git history persists and repos get shared or scraped; leaked keys must be rotated. Store them as platform secrets (environment variables) injected at runtime.

**Q14. Why bind to `0.0.0.0` when deploying?**
Containers need to accept connections from outside; `127.0.0.1` only accepts local connections inside the container, making the app unreachable.

**Q15. Why are push notifications a weak record of leads, and what's better?**
They're ephemeral and delivery isn't guaranteed (and free-tier disks are ephemeral too). Persist to a database and push as an alert on top.

**Q16. Name three risks of this public twin.**
Prompt injection leading to tool abuse or prompt leakage, cost/notification abuse without rate limits, and exposure of personal data (the résumé in the prompt; visitor data sent to third parties).

### 14.2 Calculation practice

**P1.** 500 visitors/day; 3% leave an email; each visitor triggers on average 0.2 unknown-question pushes. Expected pushes per day?
$500\,(0.03+0.20)=115$.

**P2.** 40 notifications/day, each delivered with probability 0.98 independently. Probability all arrive?
$0.98^{40}\approx 0.446$.

**P3.** Of 80 unknown-question calls, 48 were genuine gaps. Precision?
$48/80=0.6$.

**P4.** Vectors $a=(3,4)$, $b=(4,3)$. Cosine similarity?
$a\cdot b=24$, $\lVert a\rVert=\lVert b\rVert=5$, so $24/25=0.96$.

**P5.** Full-document prompt = 4,000 tokens; RAG sends 3 chunks of 250 tokens. Context reduction factor?
$4000/750\approx 5.33\times$.

**P6.** Worst-case output tokens for a session capped at 10 turns, `max_tokens=300`, up to 3 tool rounds per turn (4 model calls)?
Upper bound $10\times300\times4 = 12{,}000$ output tokens.

### 14.3 Code practice

1. Replace `globals()` dispatch with a registry and add a decorator `@tool(schema)` that registers function and schema together.
2. Make `push()` retry twice with exponential backoff and a 10-second timeout.
3. Add an input guard that rejects messages over 2,000 characters and strips control characters.
4. Add a SQLite persistence layer; push only for new emails (dedupe by address).
5. Add a daily digest function that sends a single push summarizing unknown questions.
6. Implement per-session rate limiting (for example, max 20 messages per hour) inside `chat()`.
7. Refactor your notebook into the four modules and run with `uv run app.py`; fix the relative paths with `pathlib`.
8. Add `launch(server_name="0.0.0.0", server_port=int(os.getenv("PORT", 7860)))` and test inside a Docker container.

### 14.4 Spot the bug

```python
def handle_tool_calls(tool_calls):
    results = []
    for tc in tool_calls:
        args = json.loads(tc.function.arguments)
        fn = globals().get(tc.function.name)                      # Bug 1
        result = fn(args)                                         # Bug 2
    results.append({"role": "tool", "content": result})           # Bug 3
    return results

def push(message):
    requests.post(pushover_url, json={"user": pushover_user, "token": pushover_token, "message": message})   # Bug 4
```

- Bug 1: `globals()` dispatch exposes every global; use an allowlist.
- Bug 2: should be `fn(**args)`; passing the dict as one positional argument breaks the signature.
- Bug 3: `results.append` is outside the loop (only the last call gets a result), `content` may not be a string, and `tool_call_id` is missing.
- Bug 4: Pushover expects form data (`data=`), and there is no timeout or status check.

---

## 15. One-Page Cheat Sheet

```python
# Real-world tool
def push(msg):
    r = requests.post(URL, data={"user": U, "token": T, "message": msg}, timeout=10)
    r.raise_for_status()

# Tool + schema (names must match; optional params get defaults)
def record_user_details(email, name="Name not provided", notes="not provided"): ...
schema = {"name": "...", "description": "...when to use...",
          "parameters": {"type": "object", "properties": {...},
                         "required": ["email"], "additionalProperties": False}}
tools = [{"type": "function", "function": schema}]

# Safe dispatch (allowlist, NOT globals())
TOOL_FUNCTIONS = {"record_user_details": record_user_details, ...}
def handle_tool_calls(tcs):
    out = []
    for tc in tcs:
        fn = TOOL_FUNCTIONS.get(tc.function.name)
        try:    res = fn(**json.loads(tc.function.arguments)) if fn else "unknown tool"
        except Exception as e: res = f"error: {e}"
        out.append({"role": "tool", "tool_call_id": tc.id, "content": json.dumps(res)})
    return out

# Loop (bounded)
for _ in range(5):
    r = client.chat.completions.create(model=M, messages=messages, tools=tools)
    c = r.choices[0]
    if c.finish_reason != "tool_calls": return c.message.content
    messages.append(c.message)                      # assistant first
    messages.extend(handle_tool_calls(c.message.tool_calls))   # then all tool results

# Deploy: secrets as env vars; launch(server_name="0.0.0.0", server_port=int(os.getenv("PORT", 7860)))
```

**Ten facts to remember**

1. An agent tool = **Python function + schema**; Pushover shows tools can have real-world effects.
2. Pushover needs **user key** (`u...`) and **app token** (`a...`); use `data=` (form) for its POST.
3. Schema property names must **exactly match** function parameters because of `fn(**arguments)`.
4. Descriptions and the system prompt **jointly** steer when tools fire; keep them consistent.
5. Dispatch options: `if/elif` (needs `else`) → `globals()` (convenient, **unsafe**) → **allowlist registry** (use this).
6. Model output must never choose arbitrary code; validate arguments and sanitize pushed text.
7. Every tool call needs a tool message with the matching id; `append` the assistant message, then `extend` the results.
8. "Don't know → **record, then admit**" creates a feedback loop to improve the agent's knowledge.
9. Modules: `context.py` (prompt), `tools.py` (tools), `app.py` (UI + loop); run with `uv run app.py`; use `pathlib` paths.
10. Deploy with secrets as env vars (never in git), bind `0.0.0.0`, expect cold starts and ephemeral disks; persist leads to a database, add rate limits and spending caps.

**Where this goes next:** knowledge bases and RAG, richer tools and databases, evaluators and guardrails, then frameworks (LangGraph and others) and multi-agent systems. You have now built the full stack by hand: **prompt → loop → tools → dispatch → modules → deployment.**
