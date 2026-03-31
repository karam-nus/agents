[← Back to Table of Contents](./README.md)

# Chapter 1 — What Are Agents?

> *"An agent is anything that can be viewed as perceiving its environment through sensors and acting upon that environment through actuators."*
> — Stuart Russell & Peter Norvig, *Artificial Intelligence: A Modern Approach*

## The Big Idea

An **AI agent** is a system that uses a large language model (LLM) as its reasoning engine to **autonomously decide what actions to take** in order to accomplish a goal. Unlike a chatbot that simply responds to prompts, an agent can observe, reason, act, and iterate — without human intervention at each step.

<div class="diagram">
<div class="diagram-title">Chatbot vs. Agent</div>
<div class="compare">
  <div class="compare-side left">
    <div class="compare-title">💬 Traditional LLM (Chatbot)</div>
    <ul>
      <li>Receives a prompt</li>
      <li>Generates a response</li>
      <li>Done — waits for next prompt</li>
      <li>No memory across turns (unless managed)</li>
      <li>Cannot take actions in the world</li>
    </ul>
  </div>
  <div class="compare-side right">
    <div class="compare-title">🤖 AI Agent</div>
    <ul>
      <li>Receives a goal</li>
      <li>Plans steps to achieve the goal</li>
      <li>Uses tools (search, code, APIs)</li>
      <li>Observes results and adapts</li>
      <li>Loops until the goal is met</li>
    </ul>
  </div>
</div>
</div>

## The Spectrum of Agency

Not every system is either a plain LLM or a fully autonomous agent. There's a spectrum:

<div class="diagram">
<div class="diagram-title">The Agency Spectrum</div>
<div class="layer-stack">
  <div class="layer purple">🌟 Fully Autonomous Agent <small>Self-directed goal pursuit, minimal human oversight</small></div>
  <div class="layer blue">🔄 Agentic Workflow <small>Multi-step reasoning with tool use, human checkpoint gates</small></div>
  <div class="layer accent">🔧 Augmented LLM <small>LLM + function calling / RAG — single tool use per turn</small></div>
  <div class="layer orange">📝 Prompted LLM <small>Chain-of-thought, few-shot — still just text in, text out</small></div>
  <div class="layer">💬 Basic LLM <small>Single prompt → single completion</small></div>
</div>
</div>

Most **production systems today** operate in the middle — **agentic workflows** — rather than at the fully autonomous extreme. The sweet spot is giving the LLM enough autonomy to be useful while keeping humans in the loop for critical decisions.

## When to Use Agents (and When Not To)

| Use Agents When… | Stick with Plain LLMs When… |
|---|---|
| The task requires **multiple steps** and tool use | The task is a single-turn Q&A |
| The output depends on **external information** (APIs, databases, web) | All information is in the prompt/context |
| You need **adaptive behavior** — the next step depends on what happened | The pipeline is deterministic and fixed |
| The task involves **creating + validating** (write code → run tests → fix) | Simple generation (summarize, translate) |
| There are too many possible paths to hardcode | A simple `if/else` pipeline works |

## A Minimal Agent in Code

Here's the simplest possible agent pattern in Python — a loop that thinks and acts:

```python
from openai import OpenAI

client = OpenAI()
tools = [...]  # tool definitions

messages = [{"role": "user", "content": "Find the weather in Tokyo and suggest what to wear"}]

# The agent loop
while True:
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=messages,
        tools=tools,
    )
    
    message = response.choices[0].message
    messages.append(message)
    
    # If the model wants to use a tool → execute it and continue
    if message.tool_calls:
        for tool_call in message.tool_calls:
            result = execute_tool(tool_call)  # your function
            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": result,
            })
    else:
        # No tool calls → the agent is done
        print(message.content)
        break
```

That's it. The essence of every agent is this **loop**: call the LLM → if it wants to act, execute the action → feed the result back → repeat. Everything else (memory, planning, multi-agent orchestration) builds on this foundation.

## Key Terminology

| Term | Definition |
|------|-----------|
| **Agent** | A system that uses an LLM to autonomously decide actions toward a goal |
| **Tool** | An external function the agent can invoke (search, calculator, API call) |
| **Agent Loop** | The observe → think → act cycle that drives agent behavior |
| **Agentic Workflow** | A multi-step pipeline where the LLM makes routing/action decisions |
| **Function Calling** | The LLM's ability to output structured tool invocations instead of text |
| **Grounding** | Connecting the agent to real-world data sources (search, databases) |
| **Guardrails** | Safety constraints that limit what an agent can do |

## What's Next

Now that you know what agents are and when to use them, let's look at how we got here — the research and ideas that led to the current agent paradigm.

**Next: [Chapter 2 — History & Evolution →](./02_history_and_evolution.md)**

---

*Last updated: April 2026*
