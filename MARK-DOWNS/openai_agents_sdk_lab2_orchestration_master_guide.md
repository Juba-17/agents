# Master Study Guide: OpenAI Agents SDK, Lab 2 (Orchestration: Code, Tools, Handoffs)

**Source:** `2_lab2.ipynb` (Week 2, Day 2)
**Approach:** theory first; every code block from the notebook is included in order, explained by the idea and design reasoning behind it, not by Python syntax.

> **Labels:** **[Notebook]** = taught by the notebook · **Additional Explanation** = background I added · **Textbook Insight** = standard AI/agent concept · **Research Connection** = named papers or frameworks.

---

## Table of Contents

**Part A: Theory**
1. [What the Lab Teaches](#1-what-the-lab-teaches)
2. [Orchestration: The Central Concept](#2-orchestration-the-central-concept)
3. [Workflow Patterns Used in the Lab](#3-workflow-patterns-used-in-the-lab)
4. [Agents as Tools vs Handoffs](#4-agents-as-tools-vs-handoffs)
5. [Concurrency, Cost and Latency](#5-concurrency-cost-and-latency)

**Part B: The Notebook, Code Block by Block**
6. [Part 1: Email Setup](#6-part-1-email-setup)
7. [Fallback Strategy: Push Notifications](#7-fallback-strategy-push-notifications)
8. [Part 2: Orchestrating by Code](#8-part-2-orchestrating-by-code)
9. [Adding a Tool to the Pipeline](#9-adding-a-tool-to-the-pipeline)
10. [Part 3a: Orchestrating by LLM via Tools](#10-part-3a-orchestrating-by-llm-via-tools)
11. [Part 3b: Orchestrating by LLM via Handoffs](#11-part-3b-orchestrating-by-llm-via-handoffs)

**Part C: Depth**
12. [Worked Execution Traces](#12-worked-execution-traces)
13. [Choosing an Orchestration Style](#13-choosing-an-orchestration-style)
14. [Weaknesses and Lessons](#14-weaknesses-and-lessons)
15. [Improved Version](#15-improved-version)
16. [Frameworks and Research](#16-frameworks-and-research)
17. [Self-Assessment](#17-self-assessment)
18. [Glossary](#18-glossary)
19. [One-Page Summary](#19-one-page-summary)

---

# Part A: Theory

## 1. What the Lab Teaches

Lab 1 built single agents. Lab 2 builds a **system of agents** that cooperate on a business task: producing and sending the best cold sales email for a fictional SOC2-compliance company, ComplAI.

| Part | Topic | Core idea |
|---|---|---|
| 1 | Email setup | A real-world action (sending email) with a safe fallback |
| 2 | Orchestration **by code** | You write the control flow; agents are components |
| 3a | Orchestration **by LLM, via tools** | A manager agent calls other agents like tools |
| 3b | Orchestration **by LLM, via handoffs** | An agent passes control to another agent |

**Core lesson:** the interesting design question in multi-agent systems is **who decides what happens next**: your code, or an LLM.

## 2. Orchestration: The Central Concept

**Orchestration** means coordinating several agents and tools so that together they complete a task.

| | **By code** | **By LLM** |
|---|---|---|
| **Who controls the flow** | Your program (fixed steps, loops, branches) | A manager LLM decides at runtime |
| **Predictability** | High, deterministic structure | Lower, varies run to run |
| **Power / flexibility** | Limited to what you anticipated | Can adapt, replan, handle surprises |
| **Debugging** | Easier: you know the path | Harder: read traces to see what it chose |
| **Cost control** | Easy to bound | Harder to bound |
| **Failure mode** | Rigid when reality differs from plan | Skips steps, wrong tool, loops |

The notebook's own summary: *by code is more predictable and deterministic; by LLMs is more powerful.*

**Textbook Insight.** This is a position on a spectrum of **autonomy**:

```
single LLM call  →  fixed workflow (code)  →  LLM chooses tools  →  LLM delegates to agents  →  fully autonomous
(no agency)         (predictable)              (some autonomy)       (more autonomy)             (least predictable)
```

**Practical rule (Research Connection):** Anthropic's "Building Effective Agents" guidance recommends starting with the **simplest** solution and adding autonomy only when needed, because every added degree of freedom costs reliability and cost.

## 3. Workflow Patterns Used in the Lab

Each section of the notebook is an instance of a named pattern.

| Pattern | What it is | Where in the lab |
|---|---|---|
| **Parallelization** | Run independent tasks at the same time | Three sales agents writing emails together |
| **Ensemble / best-of-N** | Generate several candidates, keep the best | Three different styles, pick one |
| **LLM-as-judge (evaluator)** | An LLM scores or selects among outputs | The sales picker agent |
| **Prompt-chaining (pipeline)** | Output of one step becomes input to the next | Drafts → picker → sender |
| **Orchestrator-workers** | A manager decomposes and delegates to workers | The Sales Manager and writer tools |
| **Handoff / delegation** | One agent transfers control to another | Sales Manager → Sales Sender |
| **Fallback** | Switch to a backup when the primary fails | Email → push notification |
| **Persona prompting** | The same task, different style instructions | Professional / humorous / executive |

**Why ensembles help:** one sample from an LLM is a random draw. Several samples with *different* instructions increase diversity, and a judge selects the best, which typically beats a single attempt. The cost is multiplied calls.

**Why the judge is imperfect:** an LLM judge may favor its own style, longer text, or the first option (position bias). It is a heuristic for "most likely to get a response", not a measurement of actual response rates.

## 4. Agents as Tools vs Handoffs

Both let one agent use another. The difference is **who holds control afterwards**.

| | **Agent as tool** | **Handoff** |
|---|---|---|
| Control flow | A → B → **back to A** | A → B (**control passes across**) |
| Who writes the final answer | The calling agent (A) | The receiving agent (B) |
| Mental model | Calling a function / consulting a specialist | Transferring a customer to another department |
| Conversation visibility | The sub-agent typically sees only the instruction given to it | The new agent takes over with the conversation context |
| Best for | Planning agent that gathers parts, then combines them | Triage and routing, specialists that finish the job |
| Reliability (per the notebook author) | More reliable | "Unreliable"; the author is "not a fan" |

**Additional Explanation.** In the SDK, a handoff is implemented **as a tool** behind the scenes (the model sees something like a "transfer to this agent" tool). So at the model level both are tool calls. The difference is what the *runtime* does after the call: for an agent-tool, run the sub-agent and return its output to the caller; for a handoff, switch the active agent.

**Why handoffs feel unreliable:** the manager must *decide* to transfer, and the decision depends on prompt wording and the model. Small models in particular may skip the transfer or never call the tools in the right order. The notebook notes that forcing tool use was needed to make it work.

## 5. Concurrency, Cost and Latency

**Parallel runs.** The three writers are independent, so running them concurrently makes total time roughly the **slowest** of the three, not the sum:

$$T_{\text{parallel}} \approx \max(t_1, t_2, t_3), \qquad T_{\text{sequential}} = t_1 + t_2 + t_3$$

**Cost does not shrink.** Tokens are still paid for each call:

$$\text{cost} \approx \sum_i c_i$$

So parallelism buys **latency**, not savings. Adding a picker adds one more call whose input contains all drafts.

**Why concurrency suits this workload:** LLM calls spend most of their time waiting on the network, so they can overlap in a single program. This is "I/O-bound" concurrency.

**Additional Explanation.** Total model calls for the lab's designs:

| Design | Model calls (approx.) |
|---|---|
| By code: 3 writers + picker | 4 |
| By code: 3 writers + sender (with tool) | 4 or 5 (the sender's tool call is followed by a final reply) |
| LLM manager with tools | 3 writers + several manager turns (about 6) |
| Manager with handoff | writers + manager turns + sender turns (about 7) |

LLM orchestration generally costs more calls than a fixed pipeline because the manager must reason at every step.

---

# Part B: The Notebook, Code Block by Block

## 6. Part 1: Email Setup

The notebook warns that proper email delivery (SendGrid, Resend) requires owning a domain and DNS verification, which is heavy. Instead it uses **SMTP** with your own account. It also notes the email is **optional**: the point is the agent work, and a push notification can replace it.

**Concepts in the setup instructions**

| Concept | Meaning |
|---|---|
| **SMTP server** | The mail server that accepts outgoing mail (each provider has an address) |
| **App password** | A separate, revocable password just for this program, so your real password is never used. Requires two-step verification on some providers |
| **`.env` file** | Holds the three secrets (server, password, address) outside the code |
| **Trade-off** | Simple and free, but you can only send, not receive (a hard challenge at the end asks you to handle replies with a proper provider) |

**Security principle:** never put credentials in code or notebooks you share. Use environment variables and revocable app-specific passwords.

```python
from dotenv import load_dotenv
import requests
from agents import Agent, Runner, trace, function_tool, ModelSettings
from agents.extensions.visualization import draw_graph
from openai.types.responses import ResponseTextDeltaEvent
import os
import asyncio
import smtplib
from email.message import EmailMessage
load_dotenv(override=True)
MODEL_NAME = "gpt-5.4-mini"
```

**Idea:** gather everything needed for the whole lab. New compared with Lab 1:

| Import | Purpose |
|---|---|
| `ModelSettings` | Lets you control model behavior per agent (used to **force tool use**) |
| `draw_graph` | Draws a diagram of an agent and its tools/handoffs (a visual map of the architecture) |
| `asyncio` | Runs several agents concurrently |
| `smtplib`, `EmailMessage` | Build and send email through an SMTP server |
| `MODEL_NAME` constant | One place to change the model for every agent (avoids repetition) |

```python
EMAIL_ADDRESS = os.getenv("EMAIL_ADDRESS")
EMAIL_SMTP_SERVER = os.getenv("EMAIL_SMTP_SERVER")
EMAIL_APP_PASSWORD = os.getenv("EMAIL_APP_PASSWORD")

if EMAIL_ADDRESS:
    print("Email address is set")
else:
    print("Email address is not set")

if EMAIL_SMTP_SERVER:
    print("SMTP server is set")
else:
    print("SMTP server is not set")

if EMAIL_APP_PASSWORD:
    print("App password is set")
else:
    print("App password is not set")

USE_EMAIL = EMAIL_ADDRESS and EMAIL_SMTP_SERVER and EMAIL_APP_PASSWORD

if USE_EMAIL:
    print("Email is set up and we will try using it")
else:
    print("Email is not set up; we will send push notifications instead")
```

**Idea: configuration-driven feature flag with graceful degradation.** The block checks which settings exist, **reports presence without revealing the values**, and sets a single switch `USE_EMAIL` that is true only if all three are present. The rest of the program asks that one switch instead of re-checking. This is the first half of the **fallback strategy**.

```python
# Here we go

def send_email(subject, text_body, html_body):
    msg = EmailMessage()
    msg["From"] = EMAIL_ADDRESS
    msg["To"] = EMAIL_ADDRESS
    msg["Subject"] = subject
    msg.set_content(text_body)
    msg.add_alternative(html_body, subtype="html")

    with smtplib.SMTP(EMAIL_SMTP_SERVER, 587) as server:
        server.starttls()
        server.login(EMAIL_ADDRESS, EMAIL_APP_PASSWORD)
        server.send_message(msg)
```

**Idea: the real-world actuator.** Concepts embedded here:

1. **Multipart email.** The message carries a plain-text body **and** an HTML alternative; the recipient's mail client shows whichever it supports. This is why the tool later takes both a text and an HTML body.
2. **Sender and recipient are the same address.** The lab emails *yourself*, which makes it safe to experiment (no real prospects get contacted).
3. **Secure connection.** The program connects to the server, upgrades the connection to **encrypted** (TLS), authenticates with the app password, then sends. Port 587 is the standard for this "start encryption after connecting" flow.

```python
send_email("Testing testing 123", "Fingers crossed..", "<html><body><strong>Fingers</strong> crossed..</body></html>")
```

**Idea: test the deterministic actuator before adding any agent**, the same principle as in Labs 0 and 1. If this fails, no amount of prompting will fix it.

```python
# USE_EMAIL = False
```

**Idea: a manual override.** If the test failed, uncomment this to force the fallback. A deliberate escape hatch that keeps the lab usable when email is impossible (for example a work account with SMTP disabled).

## 7. Fallback Strategy: Push Notifications

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

```python
def push(message):
    print(f"Push: {message}")
    payload = {"user": pushover_user, "token": pushover_token, "message": message}
    requests.post(pushover_url, data=payload)
```

**Idea:** the same Pushover setup from Lab 1, reused as the **backup channel**. A format sanity check on the keys, and the simple send function.

```python
def send_message(subject, text_body, html_body):
    if USE_EMAIL:
        send_email(subject, text_body, html_body)
    else:
        push(f"Subject: {subject}\n\n{text_body}")
```

**Idea: a facade (single interface) over multiple channels.** Callers (the agent tool later) use one function and never care which channel is active. Push messages have no HTML or subject field, so the function folds the subject into the text and drops the HTML. This **separates "what to send" from "how to deliver it"**, so channels can be swapped without touching agents.

```python
send_message("Big news", "Communications are a go!", "<html><body>Communications are a <strong>go!</strong></body></html>")
```

**Idea:** test the facade end to end. After this, "everything should work", whichever channel is active.

## 8. Part 2: Orchestrating by Code

The notebook states the two models of orchestration (by code, by LLM) and links the SDK's multi-agent guide. It starts with **code**.

```python
intro = """
You are a sales agent working for ComplAI, 
a company that provides a SaaS tool for ensuring SOC2 compliance and preparing for audits, powered by AI.
You write emails.
"""

instructions1 = intro + "Your email style is professional, serious, with gravitas and credibility."
instructions2 = intro + "Your email style is witty, engaging, and humorous."
instructions3 = intro + "Your email style is concise, to the point, in the style of a busy senior executive."
```

**Idea: shared context plus a varying style.** The shared `intro` fixes **who the agent works for and what it does**; each variant appends only a **persona**. This is **prompt composition**: change one dimension (tone) while holding everything else constant, which creates deliberate diversity for the ensemble and makes comparisons fair. SOC2 is a compliance framework for service organizations' data-security controls, which is the fictional product's domain.

```python
sales_agent1 = Agent(name="Professional Sales Agent", instructions=instructions1, model=MODEL_NAME)
sales_agent2 = Agent(name="Humorous Sales Agent", instructions=instructions2, model=MODEL_NAME)
sales_agent3 = Agent(name="Executive Sales Agent", instructions=instructions3, model=MODEL_NAME)
```

**Idea:** three agent *configurations* (recall: an agent is just configuration). Same model, different instructions. Descriptive **names** matter: they appear in traces and in the manager's tool list.

```python
result = Runner.run_streamed(sales_agent1, input="Write a cold sales email")
async for event in result.stream_events():
    if event.type == "raw_response_event" and isinstance(event.data, ResponseTextDeltaEvent):
        print(event.data.delta, end="", flush=True)
```

**Idea:** a first run of one writer using the streaming pattern from Lab 1, to see the style. Notice the input is deliberately minimal ("Write a cold sales email"): the **instructions carry the behavior**, the input is just the trigger.

```python
message = "Write a cold sales email"

with trace("Parallel cold emails"):
    results = await asyncio.gather(
        Runner.run(sales_agent1, message),
        Runner.run(sales_agent2, message),
        Runner.run(sales_agent3, message),
    )

outputs = [result.final_output for result in results]

for output in outputs:
    print(output + "\n\n")
```

**Idea: parallelization, controlled by code.** Three independent runs start together and the program waits until **all** finish. Key points:

- The code, not an LLM, decides that all three run, and when.
- Results come back **in the same order** as the calls, so you know which agent wrote which.
- The **named trace groups** the three runs into one workflow, so the dashboard shows them as one story instead of three unrelated runs.
- Total time is about the slowest writer, not the sum (§5).

```python
decision = """
You pick the best cold sales email from the given options.
Imagine you are a customer and pick the one you are most likely to respond to.
Do not give an explanation; reply with the selected email only.
"""

sales_picker = Agent(name="Sales_picker", instructions=decision, model=MODEL_NAME)
```

**Idea: the LLM-as-judge.** Three design choices:

1. **Role-play for evaluation criteria:** "imagine you are a customer" gives the judge a concrete standard (likelihood to respond) instead of an abstract "best".
2. **No explanation, selected email only:** the output is meant to be consumed by the next step in a pipeline, so it must be clean. (Trade-off: you lose the reasoning, which would help debugging and judge calibration.)
3. **No tools:** the picker only *chooses*; it does not act.

```python
message = "Write a cold sales email"

with trace("Sales selection workflow"):
    results = await asyncio.gather(
        Runner.run(sales_agent1, message),
        Runner.run(sales_agent2, message),
        Runner.run(sales_agent3, message),
    )
    outputs = [result.final_output for result in results]

    emails = "Cold sales emails:\n\n" + "\n\nEmail:\n\n".join(outputs)

    best = await Runner.run(sales_picker, emails)

    print(f"Best sales email:\n{best.final_output}")
```

**Idea: a complete code-orchestrated pipeline (fan-out, then fan-in).**

```
          ┌─ writer 1 ─┐
input ───►├─ writer 2 ─┼──► combine into one text ──► picker ──► best email
          └─ writer 3 ─┘
        (parallel, fan-out)      (code)               (sequential)
```

- **Fan-out:** the same task goes to three agents at once.
- **Combine:** plain code formats the three outputs into **one input** for the next agent. Passing text from one agent to another is the simplest way to connect agents. (Agents do not share memory; only what you put in the input.)
- **Fan-in:** the picker sees all candidates together and selects.
- All of it sits inside **one named trace**, so the whole workflow is observable.

The notebook then points you to the trace dashboard: look for three parallel model calls followed by one.

## 9. Adding a Tool to the Pipeline

```python
@function_tool
def send_email_tool(subject: str, text_body: str, html_body: str) -> str:
    """
    Send out an email with the given subject and body to all sales prospects
    
    Args:
        subject: The subject of the email
        text_body: The body of the email as plain text
        html_body: The HTML body of the email
    """
    send_message(subject, text_body, html_body)
    return "Email sent successfully"
```

**Idea: wrap the facade as a tool the model can call.** New detail beyond Lab 1: the docstring now has an **Args section**, and the SDK reads it to produce a **description for each parameter**. So the docstring supplies:

| Docstring part | Becomes |
|---|---|
| Summary line | Tool description |
| `Args:` entries | Per-parameter descriptions in the schema |
| Type hints | Parameter types |

**Design points**

- The tool wraps `send_message`, so the **fallback** works transparently.
- It returns a fixed success string regardless of what happened (a weakness; see §14).
- The docstring says "to all sales prospects", though the lab sends only to yourself. The model reads this text, so wording that does not match reality can mislead it.
- The model must supply **three** pieces of content: a subject, a plain-text body and an HTML body. The model writes the HTML itself.

```python
send_email_tool.params_json_schema
```

**Idea:** inspect the schema the SDK generated. This is the "boilerplate JSON" you hand-wrote in Week 1. Always inspect it: it is exactly what the model sees.

```python
decision = """
You pick the best cold sales email from the given options.
Imagine you are a customer and pick the one you are most likely to respond to.
Then use your tool to send the email.
"""

require_tool = ModelSettings(tool_choice="required")

sales_sender = Agent(name="Sales Sender", instructions=decision, model=MODEL_NAME, tools=[send_email_tool], model_settings=require_tool)
```

**Idea: merge judging and acting into one agent, and force the action.** Two new concepts:

1. **Picker vs sender.** The earlier picker only *returned* text. This agent both *chooses* and *acts* (sends). Whether to separate "decide" from "do" is a design decision: separation gives clearer traces and testability; merging saves a call.
2. **Forced tool use.** By default the model *chooses* whether to call a tool. `tool_choice="required"` removes that choice: the model **must** call some tool. It guarantees the action happens and compensates for small models that sometimes just reply with text. The risk: if forced forever, a model could keep calling tools with no end. The SDK includes a mechanism to reset the forcing after a tool call to avoid such loops (check the docs for the exact setting).

```python
message = "Write a cold sales email"

with trace("Sales selection workflow with sending"):
    results = await asyncio.gather(
        Runner.run(sales_agent1, message),
        Runner.run(sales_agent2, message),
        Runner.run(sales_agent3, message),
    )
    outputs = [result.final_output for result in results]

    emails = "Cold sales emails:\n\n" + "\n\nEmail:\n\n".join(outputs)

    response = await Runner.run(sales_sender, emails)

    print(f"Final response:\n{response.final_output}")
```

**Idea: the full code-orchestrated workflow with a real-world effect.** Same fan-out, combine, fan-in, but the final agent **sends** the email. Your code still controls the structure; the LLM controls only *which draft wins* and *how the send call is filled in*. This is the **most predictable** form: the sequence is guaranteed even though content is generated.

The notebook's remark: traces are "a great way to debug", and smaller models "might require more time and experimentation".

## 10. Part 3a: Orchestrating by LLM via Tools

**Concept:** instead of *your code* deciding to call the three writers, a **manager LLM** decides. The simplest mechanism is to present each agent to the manager **as a tool**. The notebook says this fits flows of the shape A → B → A and the classic **planning agent**.

```python
description = "Use this tool to write a sales email. In the input, just instruct it to write a sales email."

tool1 = sales_agent1.as_tool(tool_name="sales_email_writer_1", tool_description=description)
tool1
```

**Idea: an agent becomes a callable tool.** One method wraps an entire agent (with its own instructions and model) as a function-like tool that takes a text instruction and returns the agent's final output. Two pieces of metadata the manager reads:

| Field | Role |
|---|---|
| `tool_name` | How the manager refers to it (and how it appears in traces) |
| `tool_description` | Tells the manager **when and how** to use it. Here it also tells the manager to keep the input simple |

This is a powerful abstraction: **agents compose like functions**, so hierarchies of any depth are possible.

```python
tool1 = sales_agent1.as_tool(tool_name="sales_email_writer_1", tool_description=description)
tool2 = sales_agent2.as_tool(tool_name="sales_email_writer_2", tool_description=description)
tool3 = sales_agent3.as_tool(tool_name="sales_email_writer_3", tool_description=description)

tools = [tool1, tool2, tool3, send_email_tool]

tools
```

**Idea: the manager's toolbox mixes agents and ordinary functions.** The model sees them identically: three writers and one sender. It cannot tell (and need not know) that some tools are agents and one is a plain function. That uniform interface is the point.

```python
instructions = """
You are a Sales Manager at ComplAI. Your goal is to find the single best cold sales email using the sales_writer tools.
"""

task = """
Follow these steps:

1. Generate Drafts: Use each of the three sales_email_writer tools to generate different email drafts.
Just instruct each to write a sales email; no further details are needed.
Do not proceed until all three drafts are ready, one from each tool.
 
2. Evaluate and Select: Review the drafts and choose the single best email using your judgment of which one is most effective.
 
3. Use your tool to send the best email (and only the best email) to the user. Only send 1 email.
"""

sales_manager = Agent(name="Sales Manager", instructions=instructions, tools=tools, model=MODEL_NAME)
```

**Idea: the planning agent, with the plan in the prompt.** Note two layers of text:

| Text | Role |
|---|---|
| `instructions` | Persona and goal of the manager (the *what*) |
| `task` | A **numbered procedure** the manager must follow (the *how*), delivered as the user input |

Why this matters: in code orchestration the order is enforced by *program structure*; here it is enforced only by *prompt wording*. So the prompt must be explicit:

- "Do not proceed until all three drafts are ready" guards against skipping or partial work.
- "Only send 1 email" and "only the best email" are **guardrails** against a harmful or wasteful side effect (sending multiple emails).
- "Just instruct each to write a sales email; no further details" prevents the manager from over-engineering sub-instructions, which would change what the writers do.

Notice the manager **both selects and sends**, with no separate picker: the manager's own judgment is the evaluator.

```python
draw_graph(sales_manager)
```

**Idea: visualize the architecture.** The diagram shows the manager and its four tools as a graph. Useful for documentation, review and spotting mistakes (a missing tool, an unintended connection) before running anything.

```python
with trace("Sales manager"):
    result = await Runner.run(sales_manager, task)
```

**Idea:** run the manager. Everything else is now decided by the model: calling the three writers (perhaps in parallel), comparing, and calling the sender. Then check the trace and your email (and the spam folder, because this is, as the notebook jokes, "basically a spam message").

**Reliability warning (from the notebook):** unreliable results are "par for the course" with agentic AI, especially LLM orchestration. The remedy is **iterating on prompts**, especially with small models. This is an engineering reality: LLM-orchestrated behavior is *tuned*, not *specified*.

## 11. Part 3b: Orchestrating by LLM via Handoffs

*(The notebook's own heading repeats "3a" here; this is the second variant, 3b.)*

The notebook's position: handoffs seem unreliable and are not used consistently by other frameworks, and the SDK implements them with tools anyway. The definition:

> A handoff is a way for an agent to **delegate** to another agent, **passing control** to it.

```
Agent as tool:  A → B → A   (control returns)
Handoff:        A → B       (control passes across)
```

```python
instructions = """
You are a Sales Manager at ComplAI. You get your sales team to draft emails, then send them all to a sales picker.
"""

task = """
Follow these steps:

1. Generate Drafts: Use each of the three sales_email_writer tools to generate different email drafts.
Just instruct each to write a sales email; no further details are needed.
Do not proceed until all three drafts are ready, one from each tool.
 
2. Handoff to the sales sender to choose and send the best email.
"""

tools = [tool1, tool2, tool3]
handoffs = [sales_sender]

sales_manager = Agent(name="Sales Manager", instructions=instructions, tools=tools, handoffs=handoffs, model=MODEL_NAME)
```

**Idea: a hybrid architecture.** The manager now has two mechanisms:

| Mechanism | Used for | Why |
|---|---|---|
| **Tools** (the three writers) | Gathering drafts | The manager needs the results **back** to carry on (A → B → A) |
| **Handoff** (the sender) | Choosing and sending | Once drafts exist, the work is **finished by another agent** (A → B); the manager need not continue |

Notes:

- The manager has **no email-sending tool** anymore; sending is the sender agent's job. This follows **separation of responsibilities and least privilege**: only the sender can cause the side effect.
- The sender is the same `sales_sender` from §9, which already has `tool_choice="required"`. This is why the notebook says forced tool use was needed to make handoffs work.
- When the handoff happens, the sender receives the **conversation so far**, including the drafts the manager gathered. That is how the drafts reach it, rather than via an explicit string like in the code pipeline.
- The handoff target is described to the model by its **name**. Giving the agent a short **description of when to hand off to it** makes the decision clearer (not done in the notebook).

```python
draw_graph(sales_manager)
```

**Idea:** the graph now shows the *handoff* edge from the manager to the sender in addition to its tools, which visually distinguishes "calls" from "transfers".

```python
with trace("Sales manager"):
    result = await Runner.run(sales_manager, task)
```

**Idea:** run it. After the handoff, the final answer in the result comes from the **sender**, not the manager (control passed across). Note the notebook's reliability caveat: iterate on prompts, or use a bigger model.

**The closing exercise:** extend the orchestration (for example, refine the chosen email in another iteration, using both code and LLM orchestration). The **hard challenge** is to move to a professional email provider and **handle user replies**, so the sales agent continues the conversation. That adds *inbound events*, *state per conversation* and *long-running workflows*, which is where agent systems become genuinely complex. The commercial note: the same architecture can automate **any business process** driven by conversations and tools.

---

# Part C: Depth

## 12. Worked Execution Traces

### 12.1 Code-orchestrated pipeline with sending (§9)

| Step | Action | Who decides |
|---|---|---|
| 1 | Three writer runs start concurrently | **Code** |
| 2 | Each writer makes one model call and returns an email | Model |
| 3 | Code joins the three drafts into one text | **Code** |
| 4 | Sender run: model reads the drafts, picks one, requests `send_email_tool` with subject, text and HTML | Model |
| 5 | SDK runs the tool; message sent; "Email sent successfully" returned | SDK |
| 6 | Model writes a short final reply | Model |

Model calls: 3 + 2 = 5. Predictable shape.

### 12.2 LLM manager with tools (§10)

| Step | Action | Who decides |
|---|---|---|
| 1 | Manager reads the task; requests the three writer tools (often in one turn, in parallel) | Manager LLM |
| 2 | Each writer tool runs its own agent (3 model calls) and returns an email | Sub-agents |
| 3 | Manager receives three drafts as tool results; reasons about which is best | Manager LLM |
| 4 | Manager requests the send tool with the winner | Manager LLM |
| 5 | Tool runs; result returned | SDK |
| 6 | Manager gives a final reply | Manager LLM |

Model calls: manager 3 + writers 3 = 6. Conversation items for the manager: task, 3 tool requests, 3 outputs, 1 send request, 1 send output, final reply = 10.

### 12.3 Manager with handoff (§11)

Same first three steps as 12.2, then:

| Step | Action |
|---|---|
| 4 | Manager requests the **handoff** to the sender |
| 5 | The runtime switches the active agent to **Sales Sender** |
| 6 | Sender (forced to use a tool) requests `send_email_tool`; it runs |
| 7 | Sender writes the final reply, and **that** is the run's final output |

**Where things go wrong (diagnose in traces):** the manager writes an email itself instead of calling the writers; it calls fewer than three; it never hands off; the sender replies in text without sending (hence the forced tool choice); the manager sends more than one email.

## 13. Choosing an Orchestration Style

| Situation | Prefer | Reason |
|---|---|---|
| Steps are known and fixed | **Code** | Reliable, cheap, debuggable |
| Need parallel branches with a guaranteed join | **Code** | `asyncio`-style fan-out and fan-in |
| The sequence depends on intermediate results | **LLM via tools** | The planner adapts |
| One specialist should take over and finish the job (triage, routing) | **Handoff** | Control transfers; the specialist owns the answer |
| The side effect is costly or irreversible | **Code** (or LLM plus human approval) | Predictability |
| Prototype to explore what is possible | **LLM** | Fastest to try, least code |
| Production at scale, tight budget | **Code, with LLM only where needed** | Bounded cost and latency |

**Rule of thumb:** use LLM orchestration for the *decisions that need judgment* and code for everything else. A common architecture is code for the skeleton, LLMs at the joints.

## 14. Weaknesses and Lessons

| Weakness | Why it matters | Lesson |
|---|---|---|
| Tool returns "Email sent successfully" regardless of outcome | A failed send looks successful; the model reports false success | *Return real outcomes; surface errors* |
| No error handling or timeout around SMTP | A server problem can hang or crash the run | *Bound and guard external calls* |
| Fallback chosen only at startup | If email fails mid-run, no automatic switch to push | *Fallback at call time* |
| Docstring claims sending "to all sales prospects" while sending to oneself | The model acts on misleading text; real deployment could surprise | *Descriptions must match behavior* |
| The agent writes the subject and HTML freely | Possible HTML injection, odd formatting, misleading content | *Validate or sanitize generated content* |
| Cold sales emails are a spam-prone use case | Compliance (anti-spam laws, consent, unsubscribe) and reputation | *Consider legal and ethical constraints before automating outreach* |
| Judge selects with no explanation | Hard to audit or calibrate | *Log reasons during development* |
| LLM judge biases (position, length, self-preference) | Selection may not track real effectiveness | *Randomize order; validate against outcomes* |
| Manager prompt is the only enforcement of order | Skipped steps, extra emails | *Add checks in code or force tools; verify results* |
| "Only send 1 email" enforced by wording only | A misbehaving model may send several | *Enforce hard limits in code (one-shot tool, counters)* |
| Handoff target has no description | The model has little guidance on when to transfer | *Describe each handoff target's purpose* |
| Forced tool choice | Can force an inappropriate or repeated call | *Use sparingly and understand the SDK's reset behavior* |
| Cost grows with number of agents and turns | Three writers + judge + manager turns add up | *Count calls; use smaller models for simple roles* |
| Small-model reliability | Orchestration may fail intermittently | *Iterate on prompts; test repeatedly; consider larger models for managers* |
| Duplicated agent runs in several cells | Easy to drift out of sync | *Factor shared pipelines into functions* |
| Section heading typo (3a repeated) | Minor confusion | *Proofread* |

**Prompt-injection note.** **Research Connection.** If any agent's output is passed on as input to another (drafts into the picker, writers into the manager), a malicious or malformed output could carry instructions. In this lab all content is self-generated, so risk is low; in systems that ingest external text (replies from prospects), treat all such content as **data**, never as instructions.

## 15. Improved Version

Same design (code skeleton, LLM judgment where needed) with the §14 fixes. The *changes* are the lesson.

```python
import os, asyncio, smtplib
import requests
from email.message import EmailMessage
from dotenv import load_dotenv
from agents import Agent, Runner, trace, function_tool, ModelSettings

load_dotenv(override=True)
MODEL_NAME = "gpt-5.4-mini"

# ---------- Delivery with call-time fallback ----------
def _send_email(subject, text_body, html_body):
    msg = EmailMessage()
    msg["From"] = msg["To"] = os.environ["EMAIL_ADDRESS"]
    msg["Subject"] = subject
    msg.set_content(text_body)
    msg.add_alternative(html_body, subtype="html")
    with smtplib.SMTP(os.environ["EMAIL_SMTP_SERVER"], 587, timeout=20) as s:   # bounded
        s.starttls()
        s.login(os.environ["EMAIL_ADDRESS"], os.environ["EMAIL_APP_PASSWORD"])
        s.send_message(msg)

def _push(text):
    requests.post("https://api.pushover.net/1/messages.json", timeout=10,
                  data={"user": os.environ["PUSHOVER_USER"],
                        "token": os.environ["PUSHOVER_TOKEN"], "message": text}).raise_for_status()

def send_message(subject, text_body, html_body) -> str:
    try:
        _send_email(subject, text_body, html_body)
        return "Sent by email."
    except Exception as e:                                   # fallback at call time
        try:
            _push(f"Subject: {subject}\n\n{text_body}")
            return f"Email failed ({type(e).__name__}); sent as push instead."
        except Exception as e2:
            return f"FAILED to deliver by email or push ({type(e2).__name__})."

# ---------- One-shot send tool (hard limit enforced in code) ----------
_sent = {"count": 0}

@function_tool
def send_email_tool(subject: str, text_body: str, html_body: str) -> str:
    """
    Send ONE sales email to the test inbox. Can be used only once per run.

    Args:
        subject: The subject of the email
        text_body: The body of the email as plain text
        html_body: The HTML body of the email
    """
    if _sent["count"] >= 1:
        return "Refused: an email was already sent in this run."
    _sent["count"] += 1
    return send_message(subject, text_body, html_body)         # real outcome returned

# ---------- Writers, picker, sender ----------
intro = "You are a sales agent for ComplAI (AI SaaS for SOC2 compliance). You write emails."
styles = {
    "Professional": "professional, serious, with gravitas",
    "Humorous": "witty, engaging, humorous",
    "Executive": "concise, to the point, like a busy senior executive",
}
writers = [Agent(name=f"{n} Sales Agent", instructions=f"{intro} Style: {s}.", model=MODEL_NAME)
           for n, s in styles.items()]

picker = Agent(
    name="Sales Sender",
    instructions=("Pick the email a customer would most likely respond to, then send it with your tool. "
                  "Briefly state which style you chose and why."),     # keep the reason
    model=MODEL_NAME,
    tools=[send_email_tool],
    model_settings=ModelSettings(tool_choice="required"),
    handoff_description="Chooses the best email draft and sends it.",
)

# ---------- Code skeleton, LLM judgment at the joint ----------
async def run_pipeline(message="Write a cold sales email"):
    _sent["count"] = 0
    with trace("Improved sales workflow"):
        results = await asyncio.gather(*(Runner.run(w, message, max_turns=3) for w in writers))
        drafts = [r.final_output for r in results]
        combined = "\n\n".join(f"Email {i+1}:\n{d}" for i, d in enumerate(drafts))
        final = await Runner.run(picker, "Cold sales emails:\n\n" + combined, max_turns=4)
    return final.final_output
```

| Change | Principle |
|---|---|
| Timeouts on email and push | Bound external calls |
| Fallback decided at call time | Resilience to mid-run failures |
| Tool returns the real outcome | Honest feedback to the model |
| One-shot send enforced in code | Hard limits belong in code, not in prompts |
| Judge states its reason | Auditability and judge calibration |
| `handoff_description` | Clear delegation guidance (if used as a handoff target) |
| Turn limits on every run | Bounded autonomy |
| Pipeline in a function | Reuse; no copy-pasted cells |
| Code skeleton with LLM only at the choice | "Code for structure, LLM for judgment" |

## 16. Frameworks and Research

| Concept in this lab | Equivalents elsewhere |
|---|---|
| Parallel agent runs with a join | LangGraph parallel branches; AutoGen group chats; plain `asyncio` |
| LLM judge / picker | Evaluator-optimizer loops; LLM-as-judge evaluation |
| Manager with agents as tools | Supervisor pattern (LangGraph supervisor); CrewAI hierarchical process |
| Handoffs | Swarm-style handoffs (OpenAI's earlier Swarm experiment); LangGraph "command" transfers; AutoGen speaker transfer |
| `draw_graph` | LangGraph graph visualization |
| Forced tool use | `tool_choice` option in most LLM APIs |
| Traces of multi-agent flows | LangSmith, Langfuse, OpenTelemetry |

**Research Connection.** Several named patterns map onto the lab:

| Source | Pattern | In the lab |
|---|---|---|
| *Building Effective Agents* (Anthropic) | Parallelization, orchestrator-workers, evaluator-optimizer, routing | Writers in parallel; manager with writer tools; the picker; the handoff |
| **Self-consistency** (Wang et al., 2022) | Sample several answers, then select | Three drafts, one winner |
| **LLM-as-judge** (Zheng et al., 2023) | Use an LLM to rate outputs; known biases | The picker |
| **Multi-agent debate / MetaGPT / CAMEL** | Role-specialized agents collaborating | Role-style writers and manager |

**Also in the SDK, beyond this lab:** guardrails (validate inputs and outputs), structured outputs (force a typed response, which would make the picker return an index instead of free text), and agent lifecycle hooks.

## 17. Self-Assessment

**Questions**

1. Define orchestration, and contrast orchestrating by code with orchestrating by LLM.
2. Why is the email sent to the user's own address in the lab, and what does this teach about experimenting with side effects?
3. What does the `USE_EMAIL` flag achieve, and what two mechanisms (startup and manual) control it?
4. Why does `send_message` exist instead of calling `send_email` directly from the tool?
5. Why do the three sales agents share an `intro` and differ only in one sentence?
6. What does running the three writers with `asyncio.gather` buy, and what does it not buy?
7. In the pipeline, how does information get from the writers to the picker?
8. Why does the picker's instruction say "reply with the selected email only"? What is the downside?
9. What does `tool_choice="required"` do, and what risk does it carry?
10. Why does the SDK read the docstring's Args section?
11. Compare an agent-as-tool with a handoff in terms of control flow and who writes the final answer.
12. Why does the handoff version remove the send tool from the manager?
13. Why must the manager's `task` text say "Do not proceed until all three drafts are ready" and "Only send 1 email"?
14. What does `draw_graph` show and why is it useful?
15. How many model calls occur in the LLM-manager-with-tools design (one call per writer, one manager turn per step)?
16. Name three failure modes you would look for in a trace of an LLM-orchestrated run.
17. How would you enforce "send only one email" robustly?
18. When would you choose code orchestration over LLM orchestration?

**Answer key**

1. Orchestration coordinates multiple agents and tools toward a task. By code: your program fixes the flow (predictable, cheap, debuggable, rigid). By LLM: a manager model decides the flow at runtime (flexible, more capable, less predictable and costlier).
2. It makes experimentation safe: no real prospects are contacted. Side-effecting tools should be tested against yourself or a sandbox first.
3. It switches the whole program between email and push without changing any agent. Set automatically at startup from whether all three settings exist; manually overridable by setting it to false.
4. It is a facade: the tool does not care about the channel, and the fallback works transparently. Channels can change without touching agents.
5. Shared context keeps everything constant except tone, so differences in output come from the persona. This creates controlled diversity for the ensemble.
6. It reduces total time to about the slowest writer instead of the sum. It does not reduce cost; tokens are still paid for each call.
7. Through plain code: the drafts are formatted into one text that becomes the picker's input. Agents share nothing automatically.
8. The output is consumed by a later step, so it must be clean. The downside is that reasoning is lost, which makes auditing and debugging the judge harder.
9. It forces the model to call a tool instead of choosing. It guarantees the action but risks forcing an inappropriate or repeated call; the SDK has a reset mechanism to avoid endless loops.
10. To generate a description for each parameter in the schema, so the model knows what to put in each argument.
11. Tool: A → B → back to A; A writes the final answer. Handoff: A → B; control passes and B writes the final answer.
12. Separation of responsibility and least privilege: only the sender can cause the side effect, and the manager's job ends with delegating.
13. In LLM orchestration the order and limits are enforced only by prompt wording; these sentences prevent skipped steps and repeated side effects.
14. A graph of the manager, its tools and its handoffs. It lets you review the architecture and spot mistakes before running.
15. Three writer calls plus about three manager turns (request the tools, decide and send, final reply) = about 6.
16. For example: the manager writing an email itself rather than calling the writers; calling fewer than three writers; never handing off or never sending; sending several emails; reporting success after a failed send.
17. In code: a one-shot tool or counter that refuses further sends, not just a prompt instruction.
18. When the steps are known, the cost or side effect must be tightly bounded, or you need guaranteed structure such as a parallel fan-out and join.

## 18. Glossary

| Term | Meaning |
|---|---|
| **Orchestration** | Coordinating agents and tools to accomplish a task |
| **Orchestration by code** | The program fixes the control flow; agents are components |
| **Orchestration by LLM** | A manager model decides the flow at runtime |
| **Fan-out / fan-in** | Start several tasks together / combine their results into one |
| **Parallelization** | Running independent tasks concurrently |
| **Ensemble / best-of-N** | Generate several candidates and choose one |
| **LLM-as-judge** | Using a model to evaluate or select among outputs |
| **Persona prompting** | Instructing a model to adopt a style or role |
| **Planning agent / manager** | An agent that decomposes a task and delegates to others |
| **Agent as tool** | An agent wrapped so another agent can call it like a function |
| **Handoff** | Delegating to another agent and passing control to it |
| **Forced tool choice** | A setting that requires the model to call a tool |
| **Function tool** | A decorated function exposed to the model, schema auto-generated |
| **Facade** | A single simple interface hiding several implementations |
| **Fallback** | A backup behavior used when the primary one is unavailable |
| **Feature flag** | A switch that enables or disables behavior from configuration |
| **SMTP** | Protocol for sending email through a mail server |
| **TLS / starttls** | Encryption of a connection |
| **App password** | A revocable password dedicated to one application |
| **Multipart email** | An email carrying both plain-text and HTML versions |
| **Trace** | A recorded timeline of a workflow's model and tool calls |
| **Guardrail (prompt)** | An instruction constraining a risky behavior |
| **Position bias** | A judge's tendency to favor an option because of its place in the list |
| **SOC2** | A security and compliance framework for service organizations |

## 19. One-Page Summary

1. Lab 2 moves from single agents to **systems of agents**; the key question is **who decides the next step**: code or an LLM.
2. **By code**: predictable, cheap, debuggable, rigid. **By LLM**: flexible and powerful, less predictable, costlier. Start simple; add autonomy only when needed.
3. The code pipeline is **fan-out (three writers in parallel) → combine → fan-in (picker or sender)**. Parallelism cuts latency (about the slowest writer), not cost.
4. Prompt composition (shared intro plus one varying sentence) gives controlled diversity; an **LLM judge** selects the best, with known biases.
5. A **facade** (`send_message`) hides the delivery channel; **fallback** from email to push keeps the lab running.
6. The SDK turns a function into a tool using the name, the docstring (including its Args section) and type hints. Inspect the schema it generates.
7. **Forced tool choice** guarantees the action but must be used with care.
8. **Agents as tools** let a manager call specialists and get results back (A → B → A); this suits planning agents.
9. **Handoffs** transfer control (A → B) and the receiving agent writes the final answer; implemented as tools under the hood and less reliable in practice.
10. In LLM orchestration, **order and limits live in prompt text**, so prompts must be explicit and critical limits should also be enforced in code.
11. **Traces and `draw_graph`** are the primary debugging and design tools.
12. Weaknesses: false success messages, no timeouts, fallback only at startup, misleading docstring, prompt-only guardrails, judge biases, spam and compliance risk. Fixes: honest outcomes, bounded calls, call-time fallback, code-enforced limits, turn limits.
13. Best practice: **code for the skeleton, LLM for judgment at the joints.**
