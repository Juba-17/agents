# Master Study Guide: OpenAI Agents SDK, Lab 1 (Agent, Runner, Tracing, Streaming, Tools, Memory)

**Source:** `1_lab1.ipynb` (Week 2, Day 1)
**Approach:** theory first; every code block from the notebook is included in order, explained by the idea and design reasoning behind it, not by Python syntax.

> **Labels:** **[Notebook]** = taught by the notebook · **Additional Explanation** = background I added · **Textbook Insight** = standard AI/agent concept · **Research Connection** = named papers or frameworks.

---

## Table of Contents

**Part A: Theory**
1. [What the Lab Teaches](#1-what-the-lab-teaches)
2. [Why Agent Frameworks Exist](#2-why-agent-frameworks-exist)
3. [The Six Building Blocks of the SDK](#3-the-six-building-blocks-of-the-sdk)
4. [Observability, Streaming, Tools and Memory: Theory](#4-observability-streaming-tools-and-memory-theory)

**Part B: The Notebook, Code Block by Block**
5. [Setup and Imports](#5-setup-and-imports)
6. [Part 1: Agent and Runner](#6-part-1-agent-and-runner)
7. [Observability with Traces](#7-observability-with-traces)
8. [Streaming](#8-streaming)
9. [Part 2: Adding a Tool](#9-part-2-adding-a-tool)
10. [Part 3: Memory](#10-part-3-memory)

**Part C: Depth**
11. [Worked Execution Trace](#11-worked-execution-trace)
12. [Mapping to the Week 1 Hand-Built Loop](#12-mapping-to-the-week-1-hand-built-loop)
13. [Weaknesses and Lessons](#13-weaknesses-and-lessons)
14. [Improved Version](#14-improved-version)
15. [Frameworks and Research](#15-frameworks-and-research)
16. [Self-Assessment](#16-self-assessment)
17. [Glossary](#17-glossary)
18. [One-Page Summary](#18-one-page-summary)

---

# Part A: Theory

## 1. What the Lab Teaches

The lab introduces the **OpenAI Agents SDK**, a deliberately lightweight framework, in three parts:

| Part | Topic | Core idea |
|---|---|---|
| 1 | Agent and Agent Loop | An agent is a configuration; a runner executes the loop. Add tracing and streaming |
| 2 | Tools | A normal function becomes a tool automatically |
| 3 | Memory | A conversation survives across separate runs through a session |

**Core lesson:** everything you built by hand in Week 1 (the loop, tool schemas, the dispatcher, the message list) is **packaged** by the SDK. The concepts are unchanged; the boilerplate disappears.

## 2. Why Agent Frameworks Exist

**Additional Explanation.** Building agents from scratch repeats the same plumbing every time:

| Plumbing | Hand-built in Week 1 | Packaged by the SDK |
|---|---|---|
| Loop that calls the model until no tools are requested | `while` loop | Runner |
| Tool descriptions the model can read | Hand-written JSON schema | Generated from the function |
| Mapping tool names to functions and running them | Dispatcher | Handled internally |
| Keeping and resending history | Manual message list | Session |
| Seeing what happened | Custom printing | Tracing |
| Showing output as it arrives | Not built | Streaming |

**Trade-off of any framework:** less boilerplate and faster development versus less visibility into what happens inside. This is why understanding the underlying loop (Week 1) still matters: when something misbehaves, you debug the loop, not the framework.

**Research Connection.** Frameworks differ in weight. *Lightweight* ones (OpenAI Agents SDK, similar in spirit to others) offer a few primitives. *Graph-based* ones (LangGraph) let you draw explicit state machines. *Conversation-based* ones (AutoGen) model agents as chatting participants. The notebook's closing claim, that lightweight frameworks are all very similar, is true because they all wrap the same loop.

## 3. The Six Building Blocks of the SDK

| Block | What it is | Conceptual role |
|---|---|---|
| **Agent** | Name + instructions + model (+ tools) | The *policy and its configuration*: who the agent is and what it can do |
| **Runner** | Executes an agent on an input | The *agent loop*: ask model, run tools, repeat, stop |
| **Result** | Output object from a run | Holds the final answer and the full record of what happened |
| **Trace** | A recorded timeline of a run | *Observability*: every model call and tool call |
| **Function tool** | A decorated Python function | The *actuator*, with its schema auto-generated |
| **Session** | A stored conversation history | *Memory across runs* |

**Key separation:** an **Agent** is passive (just configuration). A **Runner** is active (does the work). The same agent can be run many times, with different inputs, sessions or traces.

**Textbook Insight.** This mirrors the classical split between an agent's *specification* (policy, goals, capabilities) and the *execution engine* that carries out the perceive-decide-act cycle.

## 4. Observability, Streaming, Tools and Memory: Theory

### 4.1 Memory: the central conceptual point of Part 3

LLMs are stateless: they remember only what is in the input. The SDK's own framing:

- **Within** one run, history is maintained (the loop keeps appending messages).
- **Between** runs, nothing carries over: each run is a **fresh start**.

So "memory" always means **resending earlier messages**. There are two ways to do it:

| Approach | Who manages history | Pros | Cons |
|---|---|---|---|
| **Manual** (pass the previous history plus the new message yourself) | You | Total control; transparent; works anywhere | You write and maintain the bookkeeping |
| **Session** (SDK stores and replays history) | The SDK | Almost no code; can persist to disk | Less control; history grows unboundedly (cost, context limit) |

**Additional Explanation.** Both approaches are *short-term conversational memory*. They are different from **long-term memory** (facts stored in a database and retrieved on demand, as in RAG or memory systems), which is not in this lab.

### 4.2 Observability: why traces matter

An agent makes decisions you cannot predict. A **trace** records each step (model calls, tool calls, timing, inputs, outputs) so you can answer "why did it do that?" and "where did the time and money go?". Observability is a prerequisite for debugging, evaluation and trust. **Research Connection:** LangSmith, Langfuse and OpenTelemetry-based tooling serve the same purpose.

### 4.3 Streaming: why it matters

An LLM generates **token by token**. Without streaming you wait for the whole answer; with streaming you display tokens as they arrive. This improves **perceived latency** (time to first token) without changing total generation time. It is essential for chat interfaces.

### 4.4 Tools from functions: the automatic schema idea

Week 1 required a hand-written schema describing each tool. The SDK derives it from the function itself:

| Part of the function | Becomes |
|---|---|
| Function name | Tool name |
| Docstring | Tool description (what the model reads to decide when to use it) |
| Type hints | Parameter schema |

**Design lesson:** the docstring is no longer just documentation for programmers; it is **prompt text for the model**. Writing it well is part of prompt engineering.

---

# Part B: The Notebook, Code Block by Block

## 5. Setup and Imports

```python
# The imports

import os
import requests
from dotenv import load_dotenv
from openai.types.responses import ResponseTextDeltaEvent
from agents import Agent, Runner, trace, function_tool, SQLiteSession
load_dotenv(override=True)
```

**Idea:** gather the capabilities used across the lab.

| Import | Purpose |
|---|---|
| `os`, `load_dotenv` | Read secrets (API keys) from the environment, keeping them out of code |
| `requests` | Call an external web API (the push-notification service) |
| `ResponseTextDeltaEvent` | The event type that carries a *small piece of generated text*, used for streaming |
| `Agent, Runner, trace, function_tool, SQLiteSession` | The SDK's main primitives (§3) |

**Packaging pitfall (from the notebook):** the package is installed as `openai-agents` but imported as `agents`. Installing a package literally named `agents` gives an unrelated, older reinforcement-learning library. Lesson: **package name and import name can differ; verify before installing.**

## 6. Part 1: Agent and Runner

```python
# Make an agent with name, instructions, model

agent = Agent(name="Jokester", instructions="You are a joke teller", model="gpt-5.4-mini")
```

**Idea:** an agent is a **configuration**, not a running process. Nothing is called yet. The three fields:

| Field | Role |
|---|---|
| `name` | Identifier, shown in traces (essential once you have several agents) |
| `instructions` | The system prompt: the agent's policy and persona |
| `model` | Which LLM powers it. A small, cheap model is chosen for a simple task |

```python
# Run the joke with Runner.run(agent, prompt)

result = await Runner.run(agent, "Tell a joke about Autonomous AI Agents")
```

**Idea: the Runner is the agent loop.** One call performs the whole cycle from Week 1: send input to the model, execute any requested tools, feed results back, repeat, stop when the model gives a final answer. Here there are no tools, so the loop does a single model call, which is why the notebook calls it "basically an LLM call".

The call is **asynchronous**: running an agent means waiting on network responses, and asynchronous execution lets a program do other work (or run many agents) during that wait. This is a design choice that becomes important for multi-agent systems.

```python
# Here is the final output

print(result.final_output)
```

**Idea:** the result object separates **the answer** (`final_output`) from **the full record**. Most of the time you want just the answer.

```python
# Here is the detail of the LLM calls

result.to_input_list()
```

**Idea: transparency.** This returns the whole conversation as a list of messages (the user input plus what the agent produced, including any tool calls and outputs). It is the same kind of message list you built by hand in Week 1, now generated for you. It matters for two reasons: (1) it lets you **inspect** what happened, and (2) it is the key to **manual memory** (§10).

## 7. Observability with Traces

```python
with trace("Telling a joke"):
    result = await Runner.run(agent, "Tell a joke about Autonomous AI Agents")
print(result.final_output)
```

**Idea: grouping runs under a named trace.** Everything executed inside the block is recorded as one named workflow. Without an explicit trace, each run still records its own trace; the named block is useful to **group several runs** (for example a multi-agent workflow) into one coherent story and to label it clearly.

The notebook then directs you to the OpenAI traces dashboard to look at the recorded steps. Practice reading traces: it is the fastest way to build intuition for what agents actually do (how many model calls, which tool, what arguments, how long).

**Additional Explanation.** What a trace typically shows: the agent run as the top-level item, each model generation as a child, each tool call as a child with its inputs and outputs, plus timings. For multi-agent systems, handoffs between agents also appear.

## 8. Streaming

```python
# Streaming

result = Runner.run_streamed(agent, input="Please tell me 5 jokes about AI Agents.")
async for event in result.stream_events():
    if event.type == "raw_response_event" and isinstance(event.data, ResponseTextDeltaEvent):
        print(event.data.delta, end="", flush=True)
```

**Idea: consume the run as a stream of events instead of waiting for the end.** Three concepts:

1. **A streamed run** returns immediately and emits events as the run progresses.
2. **Events have types.** The SDK emits many kinds of events (raw model output, tool calls, agent changes). The code **filters** for only one kind.
3. **A "text delta" event** carries just the newest few tokens. Printing each delta as it arrives produces the typing effect. Printing without line breaks, and flushing immediately, makes the text appear live rather than in a lump.

**Why this design:** a stream of typed events lets a user interface react to different things differently (show text, show a "searching..." spinner when a tool runs, show which agent is speaking). The notebook shows only the simplest use.

## 9. Part 2: Adding a Tool

```python
pushover_user = os.getenv("PUSHOVER_USER")
pushover_token = os.getenv("PUSHOVER_TOKEN")
pushover_url = "https://api.pushover.net/1/messages.json"

if pushover_user:
    if pushover_user.startswith("u"):
        print("Pushover user found and looks good")
    else:
        print("Pushover user found but doesn't start with u")
else:
    print("Pushover user not found")

if pushover_token:
    if pushover_token.startswith("a"):
        print("Pushover token found and looks good")
    else:
        print("Pushover token found but doesn't start with a")
else:
    print("Pushover token not found")
```

**Idea: validate configuration before using it.** The tool depends on an external service (Pushover) that needs credentials. This block checks (1) are the secrets present and (2) do they look right (that service's user keys start with "u", tokens with "a"). It is a **cheap sanity check** that saves confusing failures later. Note it only checks a *format hint*, not validity; the real test is the next step. It also never prints the secrets themselves, which is good practice.

```python
# Remember this?

def push(message):
    print(f"Push: {message}")
    payload = {"user": pushover_user, "token": pushover_token, "message": message}
    requests.post(pushover_url, data=payload)
```

**Idea: an ordinary function with a real-world side effect.** This is the "actuator" from §3 of the earlier guide: code that does something outside the model. It prints for the developer and sends an HTTP request to the notification service. Nothing about it is agentic yet.

```python
push("HEY!!")
```

**Idea: test the deterministic part first** (same principle as testing the checklist tools by hand in Week 1). If this does not reach your phone, no agent can fix it.

```python
push
```

**Idea:** inspect the plain function. This is a baseline for the next comparison: it is just a function object, with no knowledge of how a model should use it.

```python
# Now this:

@function_tool
def push_tool(message: str) -> str:
    """ Send the given message to the user as a push notification """
    payload = {"user": pushover_user, "token": pushover_token, "message": message}
    result = requests.post(pushover_url, data=payload).status_code
    return f"Push sent with API status code {result}"
```

**Idea: one decorator turns a function into an agent tool.** The central lesson of Part 2. Compare with Week 1:

| Week 1 | Here |
|---|---|
| Hand-written JSON schema | Generated from the type hints |
| Hand-written description | Taken from the docstring |
| Dispatcher looking up functions by name | Handled by the SDK |

Design choices worth noting:

- **The docstring is the model's instructions for the tool.** "Send the given message to the user as a push notification" tells the model when and why to use it.
- **Type hints define the contract:** one text argument, and a text return.
- **The return value is a message for the model**, not for a human. It reports the API's status code so the model learns whether the push succeeded. Returning outcomes the model can read is the "errors and results are observations" principle from Week 1.
- The `print` from the plain version is gone: observability now comes from **tracing**.

```python
push_tool
```

```python
push_tool.description
```

**Idea: the decorated object is no longer the original function; it is a tool object** carrying metadata. Inspecting `description` shows exactly what text the model will be given, which confirms the docstring became the tool description. **Practice:** always inspect the generated description and schema; they are what the model actually sees.

```python
notifier = Agent(name="Notifier", model="gpt-5.4-mini", instructions="You notify the user upon request", tools=[push_tool])
```

**Idea: capabilities are attached to an agent by listing tools.** The agent's behavior now has two parts: its instructions (when to act) and its tools (what it can do). The model decides at runtime whether to call the tool, based on the request. The instructions here are minimal because the tool description carries most of the guidance.

```python
with trace("Pizza has arrived"):
    result = await Runner.run(notifier, "Notify the user that the pizza is here")

print(result.final_output)
```

**Idea: the full tool-using agent loop, now packaged.** Behind this one call the Runner performs the loop of Week 1: the model sees the request and the tool, decides to use the tool, the SDK runs it, the result goes back, the model writes a final confirmation. The named trace lets you inspect this in the dashboard (§11 walks through it).

## 10. Part 3: Memory

The notebook opens by stating the key fact: **within one run, history is maintained; each run starts fresh.**

```python
agent = Agent(name="Assistant", model="gpt-5.4-mini")
```

**Idea:** a plain agent with no instructions and no tools, so that memory is the only thing under test.

```python
response = await Runner.run(agent, "Hi there. My name is Ed.")
print(response.final_output)
```

```python
response = await Runner.run(agent, "What's my name?")
print(response.final_output)
```

**Idea: demonstrating statelessness.** The second run is a separate call with no knowledge of the first, so the model cannot know the name. This experiment *proves* the concept from §4.1 rather than just stating it. Expect the model to say it does not know.

### Approach 1: manual memory

```python
response = await Runner.run(agent, "Hi there. My name is Ed.")
print(response.final_output)
```

```python
response.to_input_list()
```

```python
next_input = response.to_input_list() + [{"role": "user", "content": "What's my name?"}]
next_input
```

```python
response = await Runner.run(agent, next_input)
print(response.final_output)
```

**Idea: memory is just the history, resent.** Step by step in concepts:

1. Run the first message and get the result.
2. Extract the **whole conversation so far** from the result.
3. **Append** the new user message to it.
4. Give that longer history as the input to the next run.

Now the model sees "my name is Ed" earlier in its input and can answer. Two details: the input to a run can be either a simple string **or** a list of messages (the full conversation); and the message format is the same role-and-content pairs used in Week 1.

**Value of this approach:** nothing is hidden, so you can edit, trim, summarize or store the history however you like. **Cost:** you do the bookkeeping, and the history keeps growing.

### Approach 2: SDK session

```python
# This is created in-memory
# For an on-disk memory, use SQLiteSession("12345", "memory.db")

session = SQLiteSession("12346")
```

```python
response = await Runner.run(agent, "Hi there. My name is Ed.", session=session)
print(response.final_output)
```

```python
response = await Runner.run(agent, "What's my name?", session=session)
print(response.final_output)
```

**Idea: let the framework do the bookkeeping.** A session is a store of conversation history, identified by an **id**. When a run receives a session it (1) **loads** the stored history, (2) adds it before the new input, (3) runs, and (4) **saves** the new messages back. The same two questions that failed earlier now succeed.

Concepts to retain:

| Concept | Meaning |
|---|---|
| **Session id** | Names one conversation. Different ids are separate conversations (one per user or thread). Reusing an id continues that conversation |
| **SQLite** | A lightweight database, stored in a file or in memory |
| **In-memory vs on-disk** | Without a file name the history lives only while the program runs. Giving a file name makes it **persist across restarts** |
| **Pluggable memory** | SQLite is one backend; the session idea generalizes to other stores (databases, caches) |

**Additional Explanation.** A session replays history, so it does not escape the cost problem: every message of the conversation is resent each turn, so tokens and cost grow with conversation length, and a very long conversation eventually exceeds the model's context window. Real systems add **trimming, summarization, or retrieval of only relevant memories**.

**Closing exercise from the notebook:** rebuild a Week 1 project (the digital twin or the checklist loop) with this SDK. The point is to feel how much code vanishes while the design stays the same.

---

# Part C: Depth

## 11. Worked Execution Trace

**Scenario:** the "Pizza has arrived" run with the notifier agent.

| Step | What happens | Who acts |
|---|---|---|
| 1 | Runner sends the instructions, the user message, and the tool's description to the model | Runner |
| 2 | Model decides a notification is needed and **emits a tool request**: push_tool with message "the pizza is here" (wording varies) | Model |
| 3 | The SDK runs the function, which calls the external API and gets a status code (typically 200 for success) | SDK / your function |
| 4 | The tool's return text ("Push sent with API status code 200") is sent back to the model | Runner |
| 5 | Model writes a final reply confirming the user was notified | Model |
| 6 | Runner sees no more tool requests and stops; the result holds the final output | Runner |

**Items in the conversation record** after this run: (1) the user message, (2) the model's tool request, (3) the tool's output, (4) the model's final reply. So the conversation list has four entries, and the trace shows two model calls (steps 1 to 2 and steps 4 to 5) plus one tool call.

## 12. Mapping to the Week 1 Hand-Built Loop

| Week 1 (by hand) | This lab (SDK) | Same concept? |
|---|---|---|
| System prompt string | `instructions` on an Agent | Yes |
| Tool JSON schema | Generated by the decorator | Yes, automated |
| Dispatcher | Internal to the Runner | Yes, hidden |
| The `while` loop | `Runner.run` | Yes |
| Message list you maintain | `to_input_list()` or a Session | Yes, managed |
| Print statements for progress | Traces and streaming | Yes, more systematic |
| Checklist (planning) tool | Not included; you could add it as a function tool | Idea still applies |

**Takeaway:** a framework is a convenience layer over the same loop. If you understand the loop, you can predict what a framework does and debug it when it misbehaves.

## 13. Weaknesses and Lessons

| Weakness | Why it matters | Lesson |
|---|---|---|
| Tool returns only a status code and ignores failures | A failed push (bad credentials, network error) may still read as success to the model | *Return clear success or error messages the model can act on* |
| No timeout on the HTTP request | A hung service can block the whole agent | *Always bound external calls* |
| Secret check is only a prefix test | Gives false confidence | *Validate with a real call, not a format guess* |
| Credentials read at import time into globals | Easy to leak or misconfigure; hard to test | *Centralize configuration; never print secrets* |
| The agent can send notifications with any text | A model (or prompt injection) could spam or leak information | *Constrain side-effecting tools; consider approval* |
| Sessions grow without limit | Rising cost, context overflow | *Trim, summarize or retrieve selectively* |
| Session id is a hard-coded string | Different users could collide on one id and see each other's history | *Use unique ids per user or conversation* |
| In-memory session loses data on restart | Surprising "amnesia" | *Use on-disk storage when persistence matters* |
| Streaming handles only text deltas | A UI would miss tool and agent events | *Use the other event types for richer interfaces* |
| Reliance on tracing to a vendor dashboard | Data leaves your environment | *Know where trace data goes; consider privacy and compliance* |
| No limit on loop turns shown | Runaway loops cost money | *Configure a maximum number of turns* |
| Plain-text docstring is the only tool guidance | A vague docstring leads to wrong tool use | *Write descriptions as carefully as prompts* |

**Prompt-injection note.** **Research Connection.** An agent with a side-effecting tool (sending messages) is exposed if it also reads untrusted content: the content may instruct it to send something harmful. Defenses: restrict what tools can do, validate arguments, require human approval for risky actions.

## 14. Improved Version

The design is unchanged; the code adds the fixes from §13. The *changes* are the lesson.

```python
import os
import requests
from dotenv import load_dotenv
from agents import Agent, Runner, trace, function_tool, SQLiteSession

load_dotenv(override=True)

PUSHOVER_USER = os.environ["PUSHOVER_USER"]      # fail fast if missing
PUSHOVER_TOKEN = os.environ["PUSHOVER_TOKEN"]
PUSHOVER_URL = "https://api.pushover.net/1/messages.json"


@function_tool
def push_tool(message: str) -> str:
    """Send a short push notification to the user. Use only when the user explicitly asks to be notified."""
    try:
        r = requests.post(
            PUSHOVER_URL,
            data={"user": PUSHOVER_USER, "token": PUSHOVER_TOKEN, "message": message[:500]},
            timeout=10,                              # bounded external call
        )
        if r.ok:
            return "Push sent successfully."
        return f"Push FAILED with status {r.status_code}. Do not claim it was sent."
    except requests.RequestException as e:           # errors become observations
        return f"Push FAILED: {type(e).__name__}. Do not claim it was sent."


notifier = Agent(
    name="Notifier",
    model="gpt-5.4-mini",
    instructions="You notify the user upon request. If the tool reports failure, say so honestly.",
    tools=[push_tool],
)

session = SQLiteSession("user-42-thread-1", "memory.db")   # persistent, unique id

async def chat(user_message: str) -> str:
    with trace("Notifier chat"):
        result = await Runner.run(notifier, user_message, session=session, max_turns=5)
    return result.final_output
```

| Change | Principle |
|---|---|
| Fail fast on missing secrets | Configuration errors surface immediately |
| Timeout on the request | Bound external calls |
| Explicit success and failure messages | Errors are observations the model can act on |
| Message length cap | Constrain side effects |
| Docstring says when to use the tool | Tool descriptions are prompts |
| Unique, persistent session | Isolation between users and survival across restarts |
| Turn limit on the run | Bounded autonomy |
| Instructions mention honest failure reporting | Policy that complements the tool's behavior |

## 15. Frameworks and Research

| Concept in this lab | Equivalents elsewhere |
|---|---|
| Agent (name, instructions, model, tools) | LangChain / LangGraph agents; AutoGen conversable agents; LlamaIndex agents |
| Runner | Agent executor; LangGraph graph invocation |
| Function tool from a decorated function | LangChain `@tool`; auto-schema from type hints (Pydantic) |
| Trace | LangSmith, Langfuse, OpenTelemetry spans |
| Streaming events | Event streams in most frameworks and in the model APIs |
| Session | Chat message history stores; LangGraph checkpointers |
| Tool protocol across vendors | **MCP (Model Context Protocol)** standardizes how tools are described and served |

**Beyond this lab (features the SDK also offers, worth reading in the docs):** **handoffs** (an agent delegates to another agent), **agents as tools** (call one agent from another like a function), **guardrails** (validate inputs or outputs), **structured outputs** (force a typed response), and **hosted tools** (such as web search). These build the multi-agent patterns of later labs.

**Research Connection.** The loop inside the Runner is a **ReAct**-style cycle (reason, act with a tool, observe, repeat). Session memory is **short-term memory** in agent-memory taxonomies; long-term memory (vector stores, learned facts) is the next layer.

## 16. Self-Assessment

**Questions**

1. Why is an Agent described as a configuration, and what does the Runner add?
2. What does one `Runner.run` call do when the agent has no tools? With a tool?
3. Why does the second run in the "My name is Ed" demo fail to know the name?
4. Explain the manual memory approach in four steps.
5. What does a session do on each run? What does the session id control?
6. Why does a session not solve the cost-growth problem?
7. What three parts of a decorated function become parts of the tool the model sees?
8. Why should a tool return text describing its outcome rather than nothing?
9. What is the benefit of grouping runs under a named trace?
10. What is streaming good for, and why does it not shorten total generation time?
11. Why does the streaming code filter events by type?
12. Why test the plain function before turning it into a tool?
13. What is the difference between short-term and long-term memory?
14. Why can installing the package named `agents` be a mistake?
15. List two risks of giving an agent a notification tool, and a mitigation for each.
16. How many conversation items exist after the "Pizza has arrived" run?

**Answer key**

1. It only specifies name, instructions, model and tools. The Runner executes the loop: model call, tool execution, repeat, stop.
2. With no tools, one model call. With a tool: model requests the tool, SDK runs it, result returned, model produces the final reply (two model calls, one tool call).
3. Each run is a fresh start and the model is stateless; the earlier message was never in the second input.
4. Run and get the result; extract the full conversation; append the new user message; pass that longer list as the next input.
5. It loads the stored history before the run and saves the new messages after. The id names the conversation: same id continues it; a different id is a separate one.
6. The full history is still resent every turn, so tokens grow and may exceed the context window. You need trimming, summarization or selective retrieval.
7. The function name (tool name), the docstring (description), and the type hints (parameter schema).
8. The model learns whether the action succeeded and can report honestly or retry; silence leaves it guessing.
9. It groups several runs into one named workflow, making multi-step or multi-agent flows readable in the dashboard.
10. Users see text sooner (lower perceived latency). Tokens are still generated at the same speed, so total time is the same.
11. The stream carries many kinds of events; only text-delta events contain displayable text.
12. A broken function cannot be fixed by an agent; separating deterministic from probabilistic parts makes failures diagnosable.
13. Short-term memory is the recent conversation resent each turn. Long-term memory stores facts persistently and retrieves relevant ones on demand.
14. That name belongs to an unrelated older reinforcement-learning library; the right package is `openai-agents`.
15. Spam or data leakage (including via prompt injection): restrict content, validate arguments, require approval. Silent failure reported as success: return explicit failure messages and instruct honest reporting.
16. Four: user message, tool request, tool output, final reply.

## 17. Glossary

| Term | Meaning |
|---|---|
| **Agent** | A configuration of name, instructions, model and tools |
| **Runner** | The component that executes the agent loop |
| **Agent loop** | Model call → tool execution → append results → repeat until final answer |
| **Final output** | The agent's last answer from a run |
| **Input list** | The conversation as a list of role-and-content messages |
| **Trace** | A recorded timeline of a run (model calls, tool calls, timings) |
| **Observability** | Ability to see and diagnose what an agent did |
| **Streaming** | Delivering output incrementally as it is generated |
| **Event / delta** | A typed streamed item / the newest small piece of generated text |
| **Function tool** | A function exposed to the model, with schema and description generated automatically |
| **Docstring** | The function's description text; here it doubles as the tool description |
| **Session** | A stored conversation history that is loaded and saved on each run |
| **SQLite** | Lightweight database used as the session store |
| **Short-term memory** | Recent conversation history resent each turn |
| **Long-term memory** | Persistent knowledge stored externally and retrieved when relevant |
| **Stateless** | Keeping nothing between calls |
| **Handoff** | One agent delegating a task to another agent |
| **Guardrail** | A check on inputs or outputs of an agent |
| **Pushover** | The external notification service used as the example tool |
| **MCP** | Model Context Protocol, a standard for describing and serving tools |

## 18. One-Page Summary

1. The SDK packages the Week 1 agent loop. Concepts are unchanged; boilerplate vanishes.
2. **Agent** = configuration (name, instructions, model, tools). **Runner** = the loop.
3. A run returns a **result**: the final answer plus the full record (`to_input_list`).
4. **Traces** record every step for debugging, cost analysis and trust. Named traces group multi-step workflows.
5. **Streaming** delivers events as they happen; text deltas give the typing effect and lower perceived latency.
6. A **decorator** turns a function into a tool: name, docstring and type hints become the schema the model reads. The docstring is prompt text.
7. Tool results are messages to the model: return clear success and failure outcomes.
8. **Memory** is resending history. Each run is a fresh start. Two ways: manual (extend the input list yourself) or a **session** (SDK stores and replays history, optionally on disk).
9. Sessions still grow in cost and context; long-term memory is a separate, later concept.
10. Risks: unbounded sessions, session id collisions, unbounded loops, side-effecting tools exposed to prompt injection, vendor-hosted traces.
11. Fixes: bounded calls, explicit errors, constrained tools, unique persistent sessions, turn limits.
12. Frameworks look alike because they all wrap the same loop. Knowing it lets you debug any of them.
