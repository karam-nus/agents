[← Back to Table of Contents](./README.md)

# Chapter 4 — The Agent Loop (Heartbeat)

> *"The agent loop is to AI agents what the event loop is to JavaScript — the engine that makes everything work."*

## The Heartbeat

Every agent is driven by a **loop** — a repeated cycle of observing, thinking, and acting. This loop is the agent's *heartbeat*. As long as it beats, the agent is alive and working. When it stops, the agent has either achieved its goal or given up.

<div class="diagram">
<div class="diagram-title">The Agent Heartbeat</div>
<div class="cycle">
  <div class="cycle-step" style="border-color: var(--accent); color: var(--accent);">👀 Observe</div>
  <div class="cycle-arrow"></div>
  <div class="cycle-step" style="border-color: var(--purple); color: var(--purple);">🤔 Think</div>
  <div class="cycle-arrow"></div>
  <div class="cycle-step" style="border-color: var(--green); color: var(--green);">⚡ Act</div>
  <div class="cycle-arrow"></div>
  <div class="cycle-step" style="border-color: var(--blue); color: var(--blue);">🔄 Repeat</div>
</div>
</div>

| Phase | What Happens | Example |
|-------|-------------|---------|
| **Observe** | Agent receives input — user message, tool result, environment state | "Search returned 5 articles about quantum computing" |
| **Think** | LLM reasons about what to do next — possibly using chain-of-thought | "I should read the top 3 articles and extract key facts" |
| **Act** | Agent either calls a tool or returns a final answer | Calls `read_url(url)` for each article |
| **Repeat** | Tool results feed back as new observations | Loop continues until goal is met |

## The ReAct Pattern

The most influential formalization of the agent loop is **ReAct** (Reasoning + Acting). Here's how a single ReAct trace looks:

```
User: What's the population of the city that hosted the 2024 Olympics?

Thought: I need to find which city hosted the 2024 Olympics first.
Action: web_search("2024 Olympics host city")
Observation: The 2024 Summer Olympics were held in Paris, France.

Thought: Now I need to find the population of Paris.
Action: web_search("Paris France population 2024")
Observation: The population of Paris is approximately 2.1 million (city proper).

Thought: I have the answer. Paris hosted the 2024 Olympics and has ~2.1M people.
Answer: The 2024 Olympics were hosted by Paris, France, which has a population 
        of approximately 2.1 million (city proper) or 12.2 million (metro area).
```

Each **Thought → Action → Observation** cycle is one heartbeat.

## Implementation: The Core Loop

Here's a production-quality agent loop:

```python
import json
from openai import OpenAI

client = OpenAI()

def agent_loop(user_message: str, tools: list, tool_registry: dict, max_iterations: int = 10):
    """The core agent loop — the heartbeat."""
    
    messages = [
        {"role": "system", "content": "You are a helpful research assistant."},
        {"role": "user", "content": user_message},
    ]
    
    for i in range(max_iterations):
        # THINK: Ask the LLM what to do
        response = client.chat.completions.create(
            model="gpt-4o",
            messages=messages,
            tools=tools,
        )
        
        assistant_msg = response.choices[0].message
        messages.append(assistant_msg)
        
        # DECIDE: Does the agent want to act or is it done?
        if not assistant_msg.tool_calls:
            # No tool calls — agent is returning final answer
            return assistant_msg.content
        
        # ACT: Execute each tool call
        for tool_call in assistant_msg.tool_calls:
            fn_name = tool_call.function.name
            fn_args = json.loads(tool_call.function.arguments)
            
            # Execute the tool
            result = tool_registry[fn_name](**fn_args)
            
            # OBSERVE: Feed result back
            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": str(result),
            })
    
    return "Max iterations reached — could not complete the task."
```

## Loop Control: When to Stop

One of the hardest problems in agent design is **knowing when to stop**. An uncontrolled loop is expensive and potentially dangerous.

<div class="diagram">
<div class="diagram-title">Loop Termination Conditions</div>
<div class="diagram-grid cols-2">
  <div class="diagram-card green">
    <div class="card-icon">✅</div>
    <div class="card-title">Happy Path: Goal Met</div>
    <div class="card-desc">LLM returns a final answer without requesting tool calls — the natural exit</div>
  </div>
  <div class="diagram-card orange">
    <div class="card-icon">🔢</div>
    <div class="card-title">Max Iterations</div>
    <div class="card-desc">Hard limit on loop cycles (e.g., 10, 25, 50). Prevents runaway agents</div>
  </div>
  <div class="diagram-card blue">
    <div class="card-icon">💰</div>
    <div class="card-title">Budget Limit</div>
    <div class="card-desc">Cap on token spend or API cost. The agent must finish within budget</div>
  </div>
  <div class="diagram-card purple">
    <div class="card-icon">⏱️</div>
    <div class="card-title">Timeout</div>
    <div class="card-desc">Wall-clock time limit. Critical for user-facing applications</div>
  </div>
  <div class="diagram-card red">
    <div class="card-icon">🛑</div>
    <div class="card-title">Error Threshold</div>
    <div class="card-desc">Too many consecutive failures. Something is fundamentally wrong</div>
  </div>
  <div class="diagram-card accent">
    <div class="card-icon">👤</div>
    <div class="card-title">Human Interrupt</div>
    <div class="card-desc">User cancels or a checkpoint requires human approval</div>
  </div>
</div>
</div>

## Synchronous vs. Asynchronous Loops

<div class="diagram">
<div class="diagram-title">Loop Execution Models</div>
<div class="compare">
  <div class="compare-side left">
    <div class="compare-title">Synchronous (Blocking)</div>
    <ul>
      <li>User waits while agent runs</li>
      <li>Simple to implement</li>
      <li>Good for quick tasks (< 30 seconds)</li>
      <li>Used in: chatbots, CLI tools</li>
    </ul>
  </div>
  <div class="compare-side right">
    <div class="compare-title">Asynchronous (Background)</div>
    <ul>
      <li>Agent runs in background</li>
      <li>User can check progress</li>
      <li>Required for long tasks (minutes/hours)</li>
      <li>Used in: Devin, coding agents, research agents</li>
    </ul>
  </div>
</div>
</div>

## Streaming the Loop

Modern agents **stream** their heartbeat so users can watch the agent think in real time:

```python
async def streaming_agent_loop(user_message: str):
    """Agent loop with real-time streaming."""
    messages = [{"role": "user", "content": user_message}]
    
    while True:
        # Stream the LLM's response
        stream = client.chat.completions.create(
            model="gpt-4o",
            messages=messages,
            tools=tools,
            stream=True,
        )
        
        # Yield tokens as they arrive
        full_response = ""
        tool_calls = []
        
        for chunk in stream:
            delta = chunk.choices[0].delta
            if delta.content:
                yield {"type": "thinking", "content": delta.content}
                full_response += delta.content
            if delta.tool_calls:
                tool_calls.extend(delta.tool_calls)
        
        if not tool_calls:
            yield {"type": "answer", "content": full_response}
            break
        
        # Execute tools and show the user
        for tc in tool_calls:
            yield {"type": "tool_call", "name": tc.function.name}
            result = execute_tool(tc)
            yield {"type": "tool_result", "content": result}
```

## The Loop in Different Frameworks

Every framework implements the agent loop differently, but the concept is identical:

| Framework | Loop Implementation |
|-----------|-------------------|
| **OpenAI Agents SDK** | `Runner.run()` — handles the loop internally, max_turns parameter |
| **LangGraph** | State machine with edges — the loop is a graph cycle |
| **CrewAI** | Task-based — agents loop on their assigned task until completion |
| **AutoGen** | Message-passing — agents loop by sending messages to each other |
| **Raw Python** | `while True` + break conditions — you control everything |

## Common Loop Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| **Infinite loop** | No max iterations, agent loops forever | Always set `max_iterations` |
| **No error handling** | Tool failure crashes the loop | Catch exceptions, let LLM retry |
| **Context explosion** | Messages grow until context window is full | Summarize old messages, use sliding window |
| **Tool call loops** | Agent calls the same tool with the same args repeatedly | Detect repeated calls, force diversification |
| **No observability** | Can't debug what the agent did | Log every Thought/Action/Observation |

## What's Next

The agent loop drives the agent, but **tools** are what give it real-world power. Let's deep-dive into how tools work, how function calling is implemented, and how to build your own.

**Next: [Chapter 5 — Tools & Function Calling →](./05_tools_and_function_calling.md)**

---

[← Previous: Chapter 3 — Anatomy of an Agent](./03_anatomy_of_an_agent.md) · [Next: Chapter 5 — Tools & Function Calling →](./05_tools_and_function_calling.md)

*Last updated: April 2026*
