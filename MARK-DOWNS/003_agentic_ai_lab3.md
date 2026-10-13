# Agentic AI — Lab 3 Master Study Guide
### "The Digital Twin": System Prompts, Conversation History, Gradio Chat, Tool Calling, and a Hand-Built Agent Loop

> **Source notebook:** `3_lab3.ipynb` (Week 1, Day 4)
> **What the notebook really teaches:** (1) how to ground an LLM in *your own documents* by putting them in the system prompt; (2) how a chat UI creates the **illusion of memory** by re-sending history; (3) how **tool calling** works at the protocol level; (4) how to turn tool calling into an **agent loop** (`if` → `while` + `for`) without any framework.
> This is the lab where the course moves from *chatbots* to *agents*.

---

## Table of Contents

0. [How to Use This Guide](#0-how-to-use-this-guide)
1. [The Big Picture](#1-the-big-picture)
2. [Cell-by-Cell Map](#2-cell-by-cell-map)
3. [Part A: Building the Knowledge Context (PDF + Summary)](#3-part-a-building-the-knowledge-context-pdf--summary)
4. [Part B: System Prompt, History, and the Illusion of Memory](#4-part-b-system-prompt-history-and-the-illusion-of-memory)
5. [Part C: Prompt Engineering the Digital Twin](#5-part-c-prompt-engineering-the-digital-twin)
6. [Part D: The `chat()` Function and Gradio](#6-part-d-the-chat-function-and-gradio)
7. [Part E: Tool Calling from First Principles](#7-part-e-tool-calling-from-first-principles)
8. [Part F: From `if` to `while`: The Agent Loop](#8-part-f-from-if-to-while-the-agent-loop)
9. [Part G: Cost, Context, and Scaling Math](#9-part-g-cost-context-and-scaling-math)
10. [Part H: Weaknesses, Security, and a Hardened Agent](#10-part-h-weaknesses-security-and-a-hardened-agent)
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
| Understand the tool-calling protocol (most exam-relevant) | §7, §8 |
| Re-implement the whole lab from memory | §6, §7, §8, §11 |
| Understand risks of deploying such a bot | §10 |
| Exam/interview prep | §14, §15 |

**Learning objectives.** After this guide you should be able to:

1. Explain the roles of the system prompt, conversation history, and why chat is stateless.
2. Build a grounded persona bot by injecting documents into a system prompt, and explain its limits versus RAG.
3. Write a JSON-Schema tool description and explain every field.
4. Trace, message by message, what happens during a tool call (including `tool_call_id` and `finish_reason`).
5. Explain why replacing `if` with `while` (and adding a `for`) turns a tool-using chatbot into an **agent loop**.
6. Identify security/robustness problems (prompt injection, PII, infinite loops, unvalidated tool input) and fix them.

---

## 1. The Big Picture

### 1.1 What is built

```
 twin/linkedin.pdf ──pypdf──► linkedin (str) ─┐
                                              ├──► system_prompt (f-string) ──┐
 twin/summary.txt ───open()──► summary (str) ─┘                               │
                                                                              ▼
 Visitor ◄──► Gradio ChatInterface ◄──► chat(message, history) ──► LLM  ◄── tools=[record_email_tool]
                                                      │                          │
                                                      │   finish_reason == "tool_calls"?
                                                      │             yes ▼
                                                      └── run Python function ── append tool result ── call LLM again (loop)
```

A **digital twin**: a website chatbot that represents a person, answers questions about their career using supplied documents, refuses to invent facts, steers back to professional topics, and **records visitor emails via a tool**.

### 1.2 The four conceptual jumps in this lab

| # | Jump | New capability |
|---|---|---|
| 1 | Plain call → **system prompt + documents** | Grounded persona |
| 2 | Single call → **chat with history** | Conversation (illusion of memory) |
| 3 | Text only → **tool calling** | The model can *request actions* |
| 4 | `if` → **`while` loop** | Multi-step autonomous behavior (agent loop) |

### 1.3 Glossary

| Term | Meaning |
|---|---|
| **Digital twin** | An AI that represents a specific person/business to visitors. |
| **System prompt** | Instruction message (role `system`) that sets behavior and context for the whole conversation. |
| **Conversation history** | The ordered list of earlier messages re-sent each turn. |
| **Context stuffing** | Pasting documents directly into the prompt (here: LinkedIn text + summary). |
| **Gradio** | Python library that builds web UIs quickly; `ChatInterface` makes a chat page from one function. |
| **pypdf** | Library that extracts text from PDFs. |
| **Tool / function calling** | Mechanism where the model returns a *structured request* to run a function, and you run it. |
| **Tool schema** | JSON (JSON Schema) description of a tool's name, purpose, and parameters. |
| **`finish_reason`** | Why generation stopped: `"stop"`, `"length"`, `"tool_calls"`, ... |
| **`tool_call_id`** | Identifier linking a tool *result* to the tool *request* it answers. |
| **Agent loop** | Repeat: call model → if it requests tools, run them and feed results back → until it gives a final answer. |
| **Abstraction layer** | What agent frameworks provide: they hide this plumbing. |

---

## 2. Cell-by-Cell Map

| # | Type | What it does | Concept |
|---|---|---|---|
| 1 | MD | Intro; replace `twin/linkedin.pdf` and `twin/summary.txt` with your own | Personalization data |
| 2 | MD | Gradio and PyPDF note | Package selection |
| 3 | Code | Imports: `dotenv, OpenAI, PdfReader, Markdown/display, gradio as gr, json` | Toolkit |
| 4 | Code | `load_dotenv(override=True)`; `openai = OpenAI()` | Setup |
| 5 | Code | `PdfReader("twin/linkedin.pdf")`, loop pages, `extract_text()`, concatenate | PDF to string |
| 6 | Code | `print(linkedin)` | Inspect extraction |
| 7 | Code | `open("twin/summary.txt")`, `read()` | Text file to string |
| 8 | Code | `print(summary)` | Inspect |
| 9 | MD | Sidebar: system prompt, history, illusion of memory | Refresher |
| 10–17 | Code | Four mini-experiments: generic assistant; snarky assistant; "What's my name?" with no history; same with history | Demonstrate **statelessness** and persona |
| 18 | MD | "Back to the main plot" | Transition |
| 19 | Code | `system_prompt = f"""..."""` with `{summary}` and `{linkedin}` | Digital-twin prompt |
| 20 | Code | `display(Markdown(system_prompt))` | Inspect |
| 21–22 | Code | `messages = [system, user]`; call `gpt-5.4-mini` | Test the persona |
| 23 | Code | `def chat(message, history)` v1 (no tools) | Chat function |
| 24 | Code | `chat("Please summarize who you are", [])` | Manual test |
| 25 | MD | Note: non-OpenAI models may need history cleaning | Compatibility |
| 26 | Code | `gr.ChatInterface(chat).launch(inbrowser=True)` | Web UI |
| 27 | MD | "And now, TOOLS!" | Transition |
| 28–29 | Code | `record_email_tool(email)`; test call | The Python function |
| 30–32 | Code | `record_email_tool_json` schema; `tools = [{"type":"function","function":...}]` | Tool description |
| 33 | MD | "Step 2: tool calling is an `if` statement" | Key insight |
| 34 | Code | `chat()` v2: `if finish_reason == "tool_calls"` | Single tool call |
| 35 | Code | Launch Gradio again | Test |
| 36 | MD | "Step 3: first agent loop" | `while` + `for` |
| 37 | Code | `chat()` v3 with `while` and `for` | **Agent loop** |
| 38 | Code | Launch Gradio | Test |
| 39 | MD | Congratulations | Wrap-up |
| 40 | MD | Exercise: (1) evaluator LLM for work-only replies; (2) adapt to your business | Extensions |

> **Model names:** the notebook uses `gpt-5.4-nano` for toy tests and `gpt-5.4-mini` for the twin. I can't verify the names and they change; use whichever tool-calling-capable model your provider offers.

---

## 3. Part A: Building the Knowledge Context (PDF + Summary)

### 3.1 Extracting PDF text

```python
reader = PdfReader("twin/linkedin.pdf")
linkedin = ""
for page in reader.pages:
    text = page.extract_text()
    if text:
        linkedin += text
```

Line by line:

| Code | Meaning |
|---|---|
| `PdfReader(path)` | Opens the PDF and parses its structure. |
| `reader.pages` | A list-like of page objects. |
| `page.extract_text()` | Pulls the text layer from one page. Returns an empty string or `None`-like for pages without a text layer. |
| `if text:` | Skip empty/missing results (guards against concatenating `None`). |
| `linkedin += text` | Accumulate into one big string. |

**Limits to know:**

- Works only for PDFs with a **text layer**. Scanned/image-only PDFs return nothing; you need OCR.
- Layout is flattened: columns, tables, and bullet structure can come out jumbled. Always `print(linkedin)` and check (the notebook does this).
- Pages are concatenated without a separator; adding `"\n"` between pages avoids words gluing together (`linkedin += text + "\n"`).

### 3.2 Reading the summary file

```python
with open("twin/summary.txt", "r", encoding="utf-8") as f:
    summary = f.read()
```

- `with` is a **context manager**: it guarantees the file closes even on error.
- `"r"` = read mode; `encoding="utf-8"` prevents platform-dependent decoding bugs (important on Windows with non-ASCII names).
- `f.read()` returns the whole file as one string.

### 3.3 Why two sources?

| Source | Provides |
|---|---|
| `linkedin.pdf` | Structured career facts: roles, dates, education, skills |
| `summary.txt` | Voice, personality, emphasis, things not on LinkedIn |

Combining a **factual record** with a **hand-written narrative** gives accurate content *and* the right tone.

### 3.4 This is "poor man's RAG" (context stuffing)

Instead of retrieving relevant chunks, the notebook puts **all** the text in the prompt every time.

| | Context stuffing (this lab) | RAG (later labs) |
|---|---|---|
| Mechanism | Paste everything into the system prompt | Embed documents, retrieve top-$k$ chunks per query |
| Works when | Documents are small (a few pages) | Corpus is large |
| Cost per call | Pays for **all** doc tokens every turn | Pays only for retrieved chunks |
| Accuracy risk | Model may ignore details buried in long text | Retriever may miss the relevant chunk |
| Complexity | Minimal | Vector DB, chunking, embeddings |

For a single résumé (~1–3k tokens) stuffing is the right engineering choice: simplest solution that works.

---

## 4. Part B: System Prompt, History, and the Illusion of Memory

### 4.1 The three refresher concepts (from the sidebar)

1. **System prompt**: sets overall context and behavior.
2. **Conversation history**: the complete conversation so far.
3. **Illusion of memory**: every API call is stateless; you pass the whole conversation each time.

### 4.2 The four mini-experiments, explained

**Experiment 1: generic assistant.**

```python
messages = [
  {"role": "system", "content": "You are a helpful assistant"},
  {"role": "user",   "content": "Hi, my name is Ed"}]
```
Expect a polite greeting.

**Experiment 2: change only the system prompt.**
`"You are a snarky, witty assistant"` ⇒ same user message, noticeably different tone. This shows the system prompt **steers style and behavior** without changing the user's words.

**Experiment 3: ask "What's my name?" in a fresh call.**

```python
messages = [
  {"role": "system", "content": "You are a snarky, witty assistant"},
  {"role": "user",   "content": "What's my name?"}]
```
The model **cannot know** (it never saw "Ed"). It will say it doesn't know, or joke. This is **statelessness** demonstrated.

**Experiment 4: include the earlier turns.**

```python
messages = [
  {"role": "system",    "content": "You are a snarky, witty assistant"},
  {"role": "user",      "content": "Hi, my name is Ed"},
  {"role": "assistant", "content": "Well hi there, Ed. It's nice to meet you."},
  {"role": "user",      "content": "What's my name?"}]
```
Now it answers "Ed": because "Ed" is literally in the input text. **Memory = text you re-send.**

```
Turn 1 request:  [system, user1]                           → assistant1
Turn 2 request:  [system, user1, assistant1, user2]        → assistant2
Turn 3 request:  [system, user1, assistant1, user2, assistant2, user3] → assistant3
```

### 4.3 Message roles in this lab

| Role | Written by | Purpose |
|---|---|---|
| `system` | Developer | Persona, rules, documents |
| `user` | Visitor | Questions |
| `assistant` | Model (or you, replaying history) | Replies; may carry `tool_calls` |
| `tool` | Your code | Tool results, tied to a `tool_call_id` (new in this lab) |

Because the model is a function of the whole text, **the model can be "tricked" into believing a fabricated assistant message**; you control history. Anything a user can inject into history (or into the system prompt via data) is a security consideration (see §10).

---

## 5. Part C: Prompt Engineering the Digital Twin

### 5.1 The system prompt, annotated

```python
system_prompt = f"""

# Your role

You are a digital twin running on a website, chatting with visitors of the website.
You represent the person who's website you are on.
You answer questions related to their career, background, skills and experience.

Here are the details of the person you are representing:

{summary}

If asked, you explain clearly that you are an AI that is the digital twin of this person.

# Context

Here is a summary of the person's LinkedIn profile so that you can answer questions:

{linkedin}

# Rules

Engage with the user. Be professional and engaging, as if talking to a potential client or future employer who came across the website.
Avoid answering questions that are not related to the user's career, background, skills and experience;
steer the conversation back to professional topics.

Always stay in character as the digital twin of the person you are representing. Represent the person.

IMPORTANT: If you don't know the answer, say so. Never make up an answer.
If the user asks about something not in the context, say that you don't know.
"""
```

| Section | Technique | Why it works |
|---|---|---|
| `# Your role` | **Role prompting** | Establishes identity and audience |
| `{summary}`, `{linkedin}` | **Grounding / context injection** | Gives facts so the model needn't guess |
| "If asked, you explain... you are an AI" | **Transparency/honesty rule** | Prevents deceiving visitors about talking to a human |
| `# Rules` engage / professional | **Tone specification** | Matches the audience (employers, clients) |
| "Avoid answering questions not related..." | **Scope restriction / guardrail** | Prevents off-topic use (cost abuse, embarrassing content) |
| "Never make up an answer" | **Anti-hallucination instruction** | Reduces fabricated claims; **not a guarantee** |
| `IMPORTANT:` + repeat | **Emphasis and redundancy** | Models weigh explicit, repeated, capitalized constraints more |
| Markdown headers (`#`) | **Structure** | Helps the model separate role/context/rules |

### 5.2 Structuring tips used here

- **Order:** role → persona facts → context → rules. Putting **rules last** exploits recency.
- **Delimiters:** headers separate instructions from data. A stronger practice is wrapping documents in tags (`<linkedin>...</linkedin>`) so that instructions inside the document aren't confused with yours (reduces prompt-injection risk).
- **f-string:** `{summary}` and `{linkedin}` are substituted at definition time. If you later edit `summary`, you must **rebuild** `system_prompt` (re-run that cell), because the string was already formatted.

### 5.3 What the instructions cannot guarantee

Prompt rules are **probabilistic**, not enforced. A determined user can sometimes jailbreak scope restrictions, and the model can still hallucinate. That is why the exercise adds a **second LLM call** as an evaluator (§11.1), and why high-stakes systems add programmatic checks.

---

## 6. Part D: The `chat()` Function and Gradio

### 6.1 Version 1: no tools

```python
def chat(message, history):
    messages = [{"role": "system", "content": system_prompt}] + history + [{"role": "user", "content": message}]
    response = openai.chat.completions.create(model="gpt-5.4-mini", messages=messages)
    return response.choices[0].message.content
```

**The key line**, read as list concatenation:

```
[ system ]  +  history  +  [ new user message ]
   1 item      N items          1 item
```

- Python `+` on lists creates a **new list** (does not mutate `history`).
- `history` is supplied by Gradio and contains **prior turns** as dicts with `role` and `content`.
- The new `message` is wrapped as a `user` message and placed last.
- The function returns a **string**; Gradio displays it as the assistant's reply and appends both turns to `history` for next time.

```python
chat("Please summarize who you are", [])     # manual test: empty history = first turn
```

### 6.2 Gradio `ChatInterface`

```python
gr.ChatInterface(chat).launch(inbrowser=True)
```

- `ChatInterface(fn)` builds a complete chat UI. It calls `fn(message, history)` each time the user submits.
- **Gradio owns the conversation state in the browser session**; your function stays stateless. That is a clean separation: UI keeps history, function maps (message, history) → reply.
- `.launch()` starts a local web server; `inbrowser=True` opens a tab.
- **Security:** `launch()` serves on your machine only. `share=True` would create a public link: don't do that with a bot holding personal data and a paid API key.

### 6.3 The history-format note

The notebook warns: for non-OpenAI models you may need

```python
history = [{"role": h["role"], "content": h["content"]} for h in history]
```

Why: Gradio's history dicts can carry **extra keys** (such as `metadata` or `options`). OpenAI's endpoint tolerates them, but some compatible providers reject unknown fields with a 400-type error. The comprehension **whitelists only `role` and `content`**. (Details of Gradio's history format depend on version; the principle is: send providers only the fields they expect.)

### 6.4 Statelessness across the stack

```
Browser (Gradio)  : holds history list           ← the "memory"
chat()            : stateless function           ← rebuilds messages every call
OpenAI API        : stateless                    ← knows only what is in messages
```

---

## 7. Part E: Tool Calling from First Principles

### 7.1 The big idea

An LLM can only output text. **Tool calling** lets it output a *structured request* ("please run function X with arguments Y"), which **your code** executes, then you send the result back so the model can continue. **The model never runs anything itself.**

```
You → LLM : "here are tools you may request: [schema]"
LLM → You : "I want to call record_email_tool(email='a@b.com')"      (finish_reason="tool_calls")
You       : run the real Python function
You → LLM : "result of that call (id=call_123): Email recorded"
LLM → You : "Thanks! I've noted your email..."                      (finish_reason="stop")
```

The notebook's blunt framing: **"Tool calling is an `if` statement."** Under the hood it is JSON in, JSON out, plus branching in your code. Frameworks hide this boilerplate.

### 7.2 The Python function (the actual tool)

```python
def record_email_tool(email):
    print(f"Tool called to record an email: {email}")
    with open("emails.txt", "a", encoding="utf-8") as f:
        f.write(email + "\n")
    return "Email received"
```

- `"a"` = **append** mode (creates the file if absent, never overwrites).
- It has a **side effect** (writes to disk) and returns a **string** (what you may report back to the model).
- Test it directly first (`record_email_tool("test@testy.com")`): always unit-test a tool **before** wiring it to an LLM, so failures are localized.

### 7.3 Describing the tool: the JSON schema

```python
record_email_tool_json = {
    "name": "record_email_tool",
    "description": "Use this tool to record that a user provided their email address",
    "parameters": {
        "type": "object",
        "properties": {
            "email": {"type": "string", "description": "The email address of this user"}
        },
        "required": ["email"],
        "additionalProperties": False
    }
}
tools = [{"type": "function", "function": record_email_tool_json}]
```

Field-by-field:

| Field | Role | Why it matters |
|---|---|---|
| `type: "function"` (outer) | Declares the tool kind | Wrapper required by the Chat Completions API |
| `name` | Identifier the model returns when calling | **Must match** how you dispatch |
| `description` | Natural language: **when to use it** | The *only* thing the model uses to decide *whether/when* to call; it is prompt engineering |
| `parameters` | A JSON Schema object for the arguments | Constrains argument structure |
| `properties` | Each argument with `type` and `description` | The per-argument description guides how the model fills it |
| `required` | Which arguments must be present | Prevents missing args |
| `additionalProperties: False` | Disallow extra keys | Tight validation |

`tools` is a **list**: you can offer many tools. The model sees the schemas (they are added to the prompt internally) and may choose none, one, or several per turn.

**Quality of the `description` is critical.** Vague descriptions cause missed calls or misuse. Include *when* to call, *when not to*, and argument formats.

### 7.4 The response when the model wants a tool

When the model decides to call a tool, the response looks like (simplified):

```python
response.choices[0].finish_reason            # "tool_calls"
response.choices[0].message.content          # often None (no text yet)
response.choices[0].message.tool_calls       # list of requested calls
    [0].id                                   # "call_abc123"
    [0].function.name                        # "record_email_tool"
    [0].function.arguments                   # '{"email": "ed@example.com"}'   <-- a JSON STRING
```

**Crucial detail:** `function.arguments` is a **string containing JSON**, not a dict. Hence:

```python
email = json.loads(tool_call.function.arguments).get("email")
```

`json.loads` converts the string to a dict; `.get("email")` retrieves the value (returns `None` if missing rather than raising `KeyError`).

### 7.5 Version 2: `if` (single tool call)

```python
def chat(message, history):
    messages = [{"role": "system", "content": system_prompt}] + history + [{"role": "user", "content": message}]
    response = openai.chat.completions.create(model="gpt-5.4-mini", messages=messages, tools=tools)

    if response.choices[0].finish_reason == "tool_calls":
        message = response.choices[0].message
        tool_call = message.tool_calls[0]
        email = json.loads(tool_call.function.arguments).get("email")
        record_email_tool(email)
        messages.append(message)
        messages.append({"role": "tool", "content": "Email recorded", "tool_call_id": tool_call.id})
        response = openai.chat.completions.create(model="gpt-5.4-mini", messages=messages, tools=tools)

    return response.choices[0].message.content
```

**The message sequence after a tool call** (this is the protocol; memorize it):

```
messages = [
  {role: system,    content: system_prompt},
  ...history...,
  {role: user,      content: "sure, my email is ed@example.com"},
  {role: assistant, content: None, tool_calls: [{id: "call_abc123", function: {name, arguments}}]},   ← append(message)
  {role: tool,      tool_call_id: "call_abc123", content: "Email recorded"}                           ← append(tool result)
]
→ second LLM call → final assistant text
```

**Rules of the protocol:**

1. The **assistant message that contains `tool_calls`** must be appended to `messages` *as is*.
2. For **every** `tool_call` id there must be a following **`tool` message with the same `tool_call_id`**. Missing or mismatched ids cause an API error (400).
3. Then call the model **again** with the extended list; it now sees the result and writes the user-facing reply.

Why the second call? The first response was a *request*, not an answer. Without the second call, the visitor would receive `None`.

**Variable shadowing:** inside the `if`, `message = response.choices[0].message` **reassigns the function parameter** `message` (the user's string) to the assistant message object. It works only because the user string isn't needed afterwards; a clearer name is `assistant_message`.

**Flaws of version 2:**

| Flaw | Consequence |
|---|---|
| Only `tool_calls[0]` handled | If the model requests 2+ calls, extras get no result ⇒ next API call errors |
| Single round only | If the model wants another tool after seeing the result, the `if` doesn't repeat |
| Hard-coded to the email tool | No dispatch by `name`; adding a tool means editing the branch |
| Result string is hard-coded ("Email recorded") | Doesn't reflect real success/failure |

### 7.6 Parallel tool calls

Modern models can return **multiple** `tool_calls` in one assistant message (for example, record an email *and* look something up). All must be answered by separate `tool` messages before the next model call.

---

## 8. Part F: From `if` to `while`: The Agent Loop

### 8.1 Version 3

```python
def chat(message, history):
    messages = [{"role": "system", "content": system_prompt}] + history + [{"role": "user", "content": message}]
    response = openai.chat.completions.create(model="gpt-5.4-mini", messages=messages, tools=tools)

    while response.choices[0].finish_reason == "tool_calls":
        message = response.choices[0].message
        messages.append(message)
        for tool_call in message.tool_calls:
            email = json.loads(tool_call.function.arguments).get("email")
            record_email_tool(email)
            messages.append({"role": "tool", "content": "Email recorded", "tool_call_id": tool_call.id})
        response = openai.chat.completions.create(model="gpt-5.4-mini", messages=messages, tools=tools)

    return response.choices[0].message.content
```

Two changes, two purposes:

| Change | Fixes |
|---|---|
| `if` → **`while`** | Keep going as long as the model keeps requesting tools (multi-step reasoning/acting) |
| Add **`for tool_call in message.tool_calls`** | Handle **every** tool call in one assistant turn |

### 8.2 Why this is an agent loop

```
              ┌──────────────────────────────────────────────┐
              ▼                                              │
   call LLM with (messages, tools)                           │
              │                                              │
     finish_reason == "tool_calls" ? ── yes ─► for each call: │
              │                          run tool, append     │
              no                         result message ──────┘
              ▼
     return final text
```

This is the core of every agent framework: **model decides → code acts → result observed → model decides again**. It's the same skeleton as ReAct (Reason + Act): the model "reasons" (implicitly), requests an action, sees an observation, and repeats. The **LLM controls the flow** (how many iterations, whether to use a tool), which is what separates an **agent** from a Lab 1/2 **workflow**.

### 8.3 Workflow vs agent (placing Labs 1–3)

| | Lab 1/2 workflows | Lab 3 agent loop |
|---|---|---|
| Who decides the next step | Programmer | **LLM** (via `tool_calls`) |
| Number of LLM calls | Fixed | **Variable** |
| Uses tools | No | **Yes** |
| Termination | After the last fixed step | When model stops requesting tools |
| Risk | Low | Loops, wrong tool use, side effects |

### 8.4 Termination and reliability math

The loop ends when the model stops requesting tools, but nothing guarantees that. Suppose that at each iteration the model requests another tool with probability $q$ (independent). The expected number of LLM calls is geometric:

$$E[\text{calls}] = \frac{1}{1-q}\quad(\text{for } q<1)$$

**Worked example:** $q=0.5 \Rightarrow 2$ calls on average; $q=0.9 \Rightarrow 10$ calls. If $q$ were 1 (a model stuck calling a tool repeatedly), the loop never ends ⇒ always add a **max-iterations guard** (§10).

Reliability of a multi-step run: if each step succeeds with probability $p$, then $k$ steps succeed with about $p^k$. For $p=0.95, k=5$: $0.95^5\approx0.774$. **Errors compound across steps**, a core challenge of agents that motivates validation and evaluation.

### 8.5 The notebook's own message: frameworks abstract this

> "In practice, you'll likely never type this again yourself. We are shielded from these if statements by the Agent Framework."

True, and **valuable to understand anyway**: when a framework misbehaves (a missing `tool_call_id`, an infinite loop, a malformed argument), knowing the protocol lets you debug it. This is the "see the ingredients" point.

---

## 9. Part G: Cost, Context, and Scaling Math

### 9.1 The system prompt is paid for every turn

Let the system prompt (including résumé) have $S$ tokens. Each turn adds a user message of $u$ tokens and an assistant reply of $a$ tokens. Input tokens at turn $t$ (no tools):

$$N_{in}(t) = S + (t-1)(u+a) + u$$

Total input tokens over $T$ turns:

$$\sum_{t=1}^{T} N_{in}(t) = T\,S + u\,T + (u+a)\frac{T(T-1)}{2}$$

So total cost grows **quadratically** in conversation length because history is re-sent.

**Worked example.** $S=3000$, $u=50$, $a=150$ ($u+a=200$), $T=10$:

$$10(3000) + 50(10) + 200\cdot\frac{10\cdot9}{2} = 30{,}000 + 500 + 9{,}000 = 39{,}500 \text{ input tokens}$$

At an illustrative \$0.40 per million input tokens, that's about \$0.0158 for the whole conversation. Cheap, but note that **most tokens are the repeated system prompt** (30,000 of 39,500). At scale, **prompt caching** (provider feature that discounts repeated prefixes) is the standard fix. Always put the **static** content (system prompt) at the **start** so the prefix is cacheable.

### 9.2 Tool calls add extra LLM calls

Each tool round adds a full model call that re-sends everything. A turn with $r$ tool rounds makes $r+1$ calls. With history size $H$ tokens per call, input cost for that turn is about $(r+1)H$.

### 9.3 Context window limit

If $S + \text{history} + \text{output} >$ the model's context window, the call fails or truncates. Mitigations: trim old turns, summarize, or retrieve instead of stuffing.

---

## 10. Part H: Weaknesses, Security, and a Hardened Agent

### 10.1 Issues in the notebook's code

| # | Issue | Why it matters | Fix |
|---|---|---|---|
| 1 | **No max-iteration guard** in `while` | A confused model could loop and burn money | Cap iterations (e.g., 5) |
| 2 | **Hard-coded tool handling** (`.get("email")` for any tool) | Adding a second tool breaks it; wrong arguments passed | **Dispatch by `tool_call.function.name`** via a dict |
| 3 | **Hard-coded result** `"Email recorded"` | Model told success even if the write failed | Return the real result/error string |
| 4 | **No email validation** | Garbage, injection, or spam written to disk; model may hallucinate an address | Validate with a regex/`email-validator` |
| 5 | **No `try/except` around tool and `json.loads`** | Malformed JSON or I/O error crashes the chat | Catch errors, return error text to the model |
| 6 | **Prompt injection** via visitor text (and any document content) | A visitor can say "ignore your rules"; a PDF could contain instructions | Delimit data, add output checks, keep tools low-privilege |
| 7 | **PII exposure** | The whole résumé/summary is in the prompt; a visitor can ask the bot to print it verbatim | Include only what you're willing to make public |
| 8 | **Abuse/cost** | Public bot ⇒ anyone can drive up your API bill | Rate limits, max tokens, spending caps |
| 9 | **Reused/shadowed variable names** (`message`, `messages`) | Subtle bugs | Distinct names |
| 10 | **Unbounded file writes** | Flooding `emails.txt` | Dedupe, rate limit |
| 11 | **Tool messages not persisted in Gradio history** | Next turn doesn't "remember" that a tool was called, only the final text | Acceptable here; track state yourself if needed |
| 12 | **Hallucination despite rules** | The bot may still invent facts | Evaluator pass (exercise 1), quote-from-context checks |
| 13 | `launch(share=True)` risk (not used, but tempting) | Public URL to a paid, PII-holding bot | Keep local, or deploy with auth/limits |
| 14 | **Privacy of the emails collected** | Visitors' emails are personal data | Disclose purpose; store securely (GDPR-style concerns) |

### 10.2 A hardened, general agent loop

```python
import json, re

EMAIL_RE = re.compile(r"^[^@\s]+@[^@\s]+\.[^@\s]+$")

def record_email_tool(email: str) -> str:
    if not EMAIL_RE.match(email or ""):
        return "Error: that does not look like a valid email address."
    with open("emails.txt", "a", encoding="utf-8") as f:
        f.write(email.strip() + "\n")
    return "Email recorded successfully."

# Registry: tool name -> real Python function
TOOL_REGISTRY = {"record_email_tool": record_email_tool}

def handle_tool_calls(tool_calls):
    results = []
    for tc in tool_calls:
        name = tc.function.name
        try:
            args = json.loads(tc.function.arguments or "{}")
            fn = TOOL_REGISTRY[name]
            output = fn(**args)                       # dispatch by name with keyword args
        except KeyError:
            output = f"Error: unknown tool '{name}'."
        except Exception as e:                        # bad JSON, wrong args, I/O errors
            output = f"Error while running {name}: {e}"
        results.append({"role": "tool", "tool_call_id": tc.id, "content": output})
    return results                                    # one tool message per call: protocol rule #2

MAX_STEPS = 5

def chat(message, history):
    history = [{"role": h["role"], "content": h["content"]} for h in history]   # whitelist keys
    messages = [{"role": "system", "content": system_prompt}] + history + [{"role": "user", "content": message}]

    for _ in range(MAX_STEPS):
        response = openai.chat.completions.create(model="gpt-5.4-mini", messages=messages, tools=tools)
        choice = response.choices[0]
        if choice.finish_reason != "tool_calls":
            return choice.message.content
        messages.append(choice.message)                       # assistant message with tool_calls
        messages.extend(handle_tool_calls(choice.message.tool_calls))

    return "Sorry, I couldn't complete that request. Please try again."
```

What changed: **registry dispatch** (scales to many tools), **errors returned to the model** (it can recover or apologize accurately), **validation**, **bounded loop**, **history whitelist**, **honest tool results**.

Design principle: *return errors as tool output instead of raising*. The model can read "Error: invalid email" and ask the visitor to retype it, a self-correcting loop.

---

## 11. Part I: Exercise Solutions

### 11.1 Exercise 1: Evaluator pass ("strictly related to work only")

Pattern: **Evaluator–optimizer / output guardrail**. Generate a reply, have a second LLM call judge it, and regenerate with feedback if it fails.

```python
from pydantic import BaseModel

class Evaluation(BaseModel):
    is_acceptable: bool
    feedback: str

evaluator_system_prompt = f"""You are an evaluator for a digital twin chatbot.
The twin must discuss ONLY the represented person's career, background, skills and experience,
must not invent facts beyond the context, and must stay professional.

Context about the person:
{summary}

{linkedin}

Decide whether the latest reply is acceptable. Reject if it is off-topic, invents facts not in the
context, or breaks character. Give short, actionable feedback."""

def evaluate(reply, message, history):
    convo = "\n".join(f"{h['role']}: {h['content']}" for h in history)
    user_prompt = (f"Conversation so far:\n{convo}\n\nLatest user message:\n{message}\n\n"
                   f"Latest reply to evaluate:\n{reply}\n\nIs this acceptable?")
    r = openai.beta.chat.completions.parse(      # structured output (SDK parse helper)
        model="gpt-5.4-mini",
        messages=[{"role":"system","content":evaluator_system_prompt},
                  {"role":"user","content":user_prompt}],
        response_format=Evaluation)
    return r.choices[0].message.parsed

def rerun(reply, message, history, feedback):
    amended = (system_prompt +
               f"\n\n## Previous answer rejected\nYou tried: {reply}\nIt was rejected because: {feedback}\n"
               "Write a better answer that follows the rules.")
    msgs = [{"role":"system","content":amended}] + history + [{"role":"user","content":message}]
    return openai.chat.completions.create(model="gpt-5.4-mini", messages=msgs).choices[0].message.content

def chat_with_eval(message, history):
    reply = chat(message, history)               # the tool-using loop from §10.2
    for _ in range(2):                           # bounded retries
        verdict = evaluate(reply, message, history)
        if verdict.is_acceptable:
            return reply
        reply = rerun(reply, message, history, verdict.feedback)
    return "I can only discuss my professional background. What would you like to know about my work?"
```

Notes:

- **Structured output** (a Pydantic schema) guarantees parseable `is_acceptable` / `feedback`, avoiding the fragile JSON handling in Lab 2. (The exact SDK helper name/location can vary between versions; if `beta.chat.completions.parse` isn't available, use the JSON-only prompt approach from Lab 2 plus validation.)
- Use a **separate, ideally different/stronger model** as evaluator to limit self-preference bias.
- **Bounded retries** with a safe fallback prevent loops and cost spikes.
- Trade-off: roughly **2× latency and cost** per reply for added safety. Consider evaluating only when a cheap classifier flags risk.
- Edge: evaluating a reply that merely says "I don't know" should pass; make the rubric explicit that refusals and redirects are acceptable.

**Calculation:** if the base reply is acceptable with probability $p=0.9$ and the evaluator is perfect, a loop with up to 2 retries passes with probability $1-(1-p)^3=0.999$ (assuming independent retries). Real evaluators err, so the benefit is smaller.

### 11.2 Exercise 2: Adapt to your business

Steps:

1. Replace `twin/linkedin.pdf` with your business material (brochure, FAQ, pricing PDF) and rewrite `summary.txt` (services, tone, contact policy).
2. Edit the system prompt role to "an assistant for **{business}**", scope: services, pricing policy, process.
3. Keep `record_email_tool`, and **improve its description** (e.g., "Use this when a visitor wants to be contacted or asks for a quote and has given an email address; do not call it without an explicit email").
4. Add a second tool to see why name-based dispatch matters:

```python
def record_unknown_question(question: str) -> str:
    with open("unknown_questions.txt", "a", encoding="utf-8") as f:
        f.write(question.strip() + "\n")
    return "Question recorded for follow-up."

record_unknown_question_json = {
    "name": "record_unknown_question",
    "description": "Record any question you could not answer from the provided context, so the owner can follow up",
    "parameters": {"type": "object",
                   "properties": {"question": {"type": "string", "description": "The unanswered question"}},
                   "required": ["question"], "additionalProperties": False}}

tools = [{"type":"function","function":record_email_tool_json},
         {"type":"function","function":record_unknown_question_json}]
TOOL_REGISTRY["record_unknown_question"] = record_unknown_question
```

With the registry from §10.2, **no change to the loop is required**: that is the payoff of dispatch-by-name. Also update the system prompt: "If you cannot answer, call `record_unknown_question`, then tell the visitor you've noted it."

5. Test cases: normal question; off-topic question; visitor gives an email; visitor gives a *malformed* email; visitor asks for the system prompt; visitor attempts "ignore your instructions."

---

## 12. Common Errors and Debugging

| Symptom | Likely cause | Fix |
|---|---|---|
| `FileNotFoundError: twin/linkedin.pdf` | Wrong working directory or file missing | Check `os.getcwd()`; correct relative path |
| `linkedin` is empty | Image-only/scanned PDF | OCR, or export a text-based PDF |
| Garbled/merged text | Multi-column layout | Clean up text, or hand-write the key facts into `summary.txt` |
| Bot answers with old info after you edit `summary` | `system_prompt` f-string not rebuilt | Re-run the prompt cell |
| API 400: "tool_call_id ... did not have response messages" | Assistant `tool_calls` not followed by a `tool` message for every id | Answer **every** tool call (use the `for`) |
| API 400: tool message without preceding tool_calls | Appended `tool` message without appending the assistant message first | Append `message` **before** tool results |
| `TypeError`/400 about message content `None` | Tried to pass `content=None` badly or serialized the message object wrongly | Append the SDK message object as in the notebook |
| `json.JSONDecodeError` on arguments | Model produced malformed JSON | Wrap in `try/except`, return error to model |
| `NameError: tools is not defined` | Skipped the tools cell | Run cells in order |
| Tool never called | Weak `description`, or model not tool-capable | Improve description; confirm model supports tools |
| Tool called with a made-up email | No validation; prompt lacks "only if given" | Validate; refine description |
| Infinite/long loops, big bill | No iteration cap | `MAX_STEPS` |
| Gradio history key error with other providers | Extra keys in history dicts | Whitelist `role`/`content` |
| Bot reveals the résumé verbatim | Expected behavior of grounding | Keep only public info in the prompt |
| `emails.txt` not where expected | Relative path = notebook's working directory | Use an absolute path or print `os.getcwd()` |
| Gradio port busy | Previous interface still running | Stop the old cell or change the port |

---

## 13. Comparison Tables

### 13.1 `if` vs `while` vs bounded `for`

| | `if` (v2) | `while` + `for` (v3) | bounded `for _ in range(N)` (hardened) |
|---|---|---|---|
| Rounds of tool use | 1 | Unlimited | ≤ N |
| Multiple calls per round | No (index 0 only) | Yes | Yes |
| Runaway risk | None | **Yes** | Contained |
| Production-ready | No | No | Closer |

### 13.2 Chatbot vs tool-using assistant vs agent

| | Chatbot (v1) | Tool-using assistant (v2) | Agent loop (v3+) |
|---|---|---|---|
| Can act on the world | No | Once per turn | Repeatedly |
| LLM calls per user turn | 1 | 1–2 | 1–N |
| Autonomy | None | Low | Higher |
| Failure surface | Hallucination | + tool misuse | + loops, compounding errors |

### 13.3 Where state lives

| State | Lives in | Persists? |
|---|---|---|
| Conversation history | Gradio session (browser/server memory) | Until the page/session ends |
| System prompt | Python variable | Until the kernel restarts |
| Collected emails | `emails.txt` | Permanent (disk) |
| Model's "knowledge" of the user | Nowhere between calls | Only what is re-sent |

### 13.4 Function calling vs prompt-based "tool use"

| | Native function calling | Parse tool requests from free text |
|---|---|---|
| Output | Structured `tool_calls` | Regex/JSON from prose |
| Reliability | High | Fragile |
| Schema enforcement | Yes (JSON Schema) | Manual |
| Provider support | Most major APIs | Anything |

### 13.5 Hand-built loop vs framework

| | Hand-built (Lab 3) | Framework (LangGraph, etc.) |
|---|---|---|
| Control/transparency | Full | Abstracted |
| Boilerplate | You write it | Provided |
| Features | Minimal | State, memory, retries, tracing |
| Learning value | **Understand the protocol** | Productivity |

---

## 14. Exam Preparation

### 14.1 Conceptual questions with model answers

**Q1. Why does "What's my name?" fail in a fresh call but work when the earlier turns are included?**
The API is stateless; the model only sees the text in `messages`. Including the earlier user/assistant turns puts "Ed" in the input, so the model can answer. Chat "memory" is just re-sent history.

**Q2. What does the system prompt do, and why is it re-sent every turn?**
It sets persona, rules, and context. Because the API is stateless, it must be included in every request or the behavior is lost.

**Q3. Why put the LinkedIn text and summary into the system prompt rather than the user message?**
The system prompt is the designated place for standing context and rules that apply to the whole conversation; models are trained to treat it as higher-priority instructions. It also keeps the history clean.

**Q4. What is context stuffing, and when is it better than RAG?**
Pasting all source text into the prompt. It's better for small corpora (a résumé) because it's simple and avoids retrieval errors; RAG is needed when the corpus is too big or costly to send each turn.

**Q5. Explain `[system] + history + [user]`.**
List concatenation building the full message list: one system message, all prior turns from Gradio, and the new user message. It produces a new list without mutating `history`.

**Q6. Why must a tool be described with JSON Schema?**
The model needs a precise, machine-readable contract of the tool's name, purpose, and argument structure to decide when to call it and to produce valid arguments; the `description` fields act as prompts guiding usage.

**Q7. Why is `function.arguments` parsed with `json.loads`?**
It is a JSON-formatted **string**, not a dict; `json.loads` converts it so you can read fields.

**Q8. Describe the full message sequence for one tool call.**
`system`, history, `user` → model returns an `assistant` message with `tool_calls` (finish_reason `tool_calls`) → you append that assistant message → run the tool → append a `tool` message with the matching `tool_call_id` → call the model again → final `assistant` text (finish_reason `stop`).

**Q9. What happens if you omit the `tool` message for a tool call?**
The next API call errors (400) because every `tool_call` id requires a corresponding tool result message.

**Q10. Why does the second LLM call exist after running the tool?**
The first response is a request, not a user-facing answer. The model needs the tool result to compose the final reply.

**Q11. What two changes turn version 2 into an agent loop, and why?**
`if` → `while` (continue while the model requests tools, enabling multi-step actions) and adding a `for` over `message.tool_calls` (handle multiple parallel tool calls). Control flow is now decided dynamically by the model.

**Q12. Why is the notebook's loop unsafe for production?**
No iteration cap, hard-coded handling for one tool, no validation or error handling, success message hard-coded, and exposure to prompt injection and cost abuse.

**Q13. How does a framework relate to this lab's code?**
It implements exactly this plumbing (schemas, dispatch, appending messages, looping) behind an abstraction. Understanding it helps debug the framework.

**Q14. Why does "never make up an answer" not guarantee honesty?**
Prompt instructions are probabilistic; the model can still hallucinate or be jailbroken. Add evaluators, grounding checks, and tests.

**Q15. How does the evaluator exercise fit design patterns?**
It's **evaluator–optimizer / reflection** (generate → judge → regenerate with feedback) and acts as an output guardrail; costs extra latency/tokens.

### 14.2 Calculation practice

**P1.** $S=2000$, $u=40$, $a=160$, $T=6$ turns. Total input tokens?
$6(2000)+40(6)+200\cdot\frac{6\cdot5}{2} = 12000+240+3000 = 15{,}240$.

**P2.** A turn needs 2 tool rounds with a 4,000-token context each call. Approximate input tokens for the turn?
$(r+1)H = 3\times4000 = 12{,}000$ (history grows slightly with tool messages, so slightly more).

**P3.** The model requests another tool each iteration with probability 0.8 (independent). Expected number of LLM calls?
$1/(1-0.8)=5$.

**P4.** A 4-step agent run where each step is 90% reliable. Success probability?
$0.9^4=0.6561$.

**P5.** Evaluator loop: base acceptability 0.7, up to 2 retries, perfect evaluator, independent retries. Probability of an accepted answer?
$1-(0.3)^3 = 1-0.027=0.973$.

### 14.3 Code practice

1. Add `record_unknown_question` as a second tool with name-based dispatch; no change to the loop.
2. Add a `MAX_STEPS` guard and test it by creating a tool whose result always prompts another call.
3. Write `validate_email` and have the tool return errors the model can act on.
4. Implement the evaluator exercise with a Pydantic schema; log how often replies are rejected.
5. Add per-session logging of `response.usage` to measure cost per conversation.
6. Wrap the system prompt documents in XML tags and test a prompt-injection attempt hidden in `summary.txt`.
7. Rewrite the hand-built loop using a different provider via `base_url` (Lab 2) and verify tool calling still works.

### 14.4 Spot the bug

```python
def chat(message, history):
    messages = [system_prompt] + history + [message]                          # Bug 1
    response = openai.chat.completions.create(model="gpt-5.4-mini", messages=messages, tools=tools)
    if response.choices[0].finish_reason == "tool_calls":
        call = response.choices[0].message.tool_calls[0]
        args = call.function.arguments["email"]                              # Bug 2
        record_email_tool(args)
        messages.append({"role": "tool", "content": "ok"})                   # Bug 3
        response = openai.chat.completions.create(model="gpt-5.4-mini", messages=messages, tools=tools)
    return response.choices[0].message
```

- Bug 1: `messages` must be a list of **dicts** with `role`/`content`; here the system prompt and user message are raw strings.
- Bug 2: `arguments` is a **JSON string**; needs `json.loads(...)["email"]`.
- Bug 3: tool message lacks `tool_call_id`, **and** the assistant message with `tool_calls` was never appended first.
- (Bonus) The function returns the **message object**, not `.content` text; Gradio expects a string.

---

## 15. One-Page Cheat Sheet

```python
# Grounding
linkedin = "".join(p.extract_text() or "" for p in PdfReader(path).pages)
summary  = open(path, encoding="utf-8").read()
system_prompt = f"...{summary}...{linkedin}...rules..."

# Chat = stateless function + history from the UI
messages = [{"role":"system","content":system_prompt}] + history + [{"role":"user","content":message}]
gr.ChatInterface(chat).launch(inbrowser=True)

# Tool schema
tools = [{"type":"function","function":{
    "name":"fn", "description":"when to use it",
    "parameters":{"type":"object","properties":{"x":{"type":"string","description":"..."}},
                  "required":["x"], "additionalProperties":False}}}]

# Agent loop
resp = client.chat.completions.create(model=M, messages=messages, tools=tools)
while resp.choices[0].finish_reason == "tool_calls":
    msg = resp.choices[0].message
    messages.append(msg)                                           # 1. assistant tool_calls message
    for tc in msg.tool_calls:
        args = json.loads(tc.function.arguments)                   # 2. arguments is a JSON string
        out  = REGISTRY[tc.function.name](**args)                  # 3. dispatch by name
        messages.append({"role":"tool","tool_call_id":tc.id,"content":out})   # 4. result tied to id
    resp = client.chat.completions.create(model=M, messages=messages, tools=tools)
final_text = resp.choices[0].message.content
```

**Ten facts to remember**

1. The API is **stateless**; memory = re-sent history; Gradio holds the history, `chat()` stays stateless.
2. The **system prompt** carries persona, rules, and documents; rebuild the f-string after editing its inputs.
3. **Context stuffing** suits small documents; RAG suits large corpora.
4. Prompt rules are **probabilistic**; add evaluators and validation for safety.
5. A tool = **Python function + JSON-Schema description**; the `description` is a prompt.
6. The model only **requests** a tool (`finish_reason == "tool_calls"`); **your code** runs it.
7. `function.arguments` is a **JSON string** ⇒ `json.loads`.
8. Protocol: append the assistant `tool_calls` message, then **one `tool` message per call** with the matching `tool_call_id`, then call the model again.
9. `if` → `while` + `for` = **agent loop** (model decides how many steps); always add a **max-step guard**.
10. Total input cost grows **quadratically** with turns (history re-sent) and the static system prompt is paid every call ⇒ prompt caching; frameworks hide this plumbing but you should know it.

**Where this goes next:** multiple tools and structured outputs → memory and RAG → framework-based orchestration (LangGraph and similar) → multi-agent systems → evaluation, observability, and guardrails. The loop in §8 is the engine inside all of them.
