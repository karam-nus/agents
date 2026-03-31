[← Back to Table of Contents](./README.md)

# Chapter 8 — Multi-Agent Systems

> *"One agent is useful. Multiple agents collaborating is transformative."*
> — Andrew Ng

## Why Multiple Agents?

A single agent has limits: it can get confused with too many tools, struggles with very complex tasks, and has one perspective. **Multi-agent systems** solve this by having specialized agents collaborate — the same way a team of humans is more effective than one person doing everything.

<div class="diagram">
<div class="diagram-title">Single Agent vs. Multi-Agent</div>
<div class="compare">
  <div class="compare-side left">
    <div class="compare-title">One Agent Does Everything</div>
    <ul>
      <li>20+ tools → confused tool selection</li>
      <li>One system prompt tries to cover all roles</li>
      <li>Context window fills with everything</li>
      <li>Hard to debug — one monolithic loop</li>
      <li>No checks and balances</li>
    </ul>
  </div>
  <div class="compare-side right">
    <div class="compare-title">Specialized Agents Collaborate</div>
    <ul>
      <li>Each agent has 2–5 focused tools</li>
      <li>Each has a clear, specific role</li>
      <li>Context stays relevant per agent</li>
      <li>Debug each agent independently</li>
      <li>Agents can review each other's work</li>
    </ul>
  </div>
</div>
</div>

## Multi-Agent Architectures

### 1. Supervisor Pattern

A central **supervisor agent** coordinates worker agents. It decides who works on what and synthesizes their output.

<div class="diagram">
<div class="diagram-title">Supervisor Pattern</div>
<div class="flow">
  <div class="flow-node accent wide">👤 User request</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node purple wide">🎯 Supervisor Agent <small>Delegates tasks, coordinates, synthesizes</small></div>
  <div class="flow-arrow"></div>
  <div class="flow-node wide" style="min-width: 100%;">
    <div style="display: flex; gap: 0.75rem; justify-content: center; flex-wrap: wrap;">
      <div class="diagram-card green" style="flex: 1; min-width: 110px;">
        <div class="card-icon">🔍</div>
        <div class="card-title">Researcher</div>
        <div class="card-desc">Web search, data gathering</div>
      </div>
      <div class="diagram-card blue" style="flex: 1; min-width: 110px;">
        <div class="card-icon">💻</div>
        <div class="card-title">Coder</div>
        <div class="card-desc">Write and execute code</div>
      </div>
      <div class="diagram-card orange" style="flex: 1; min-width: 110px;">
        <div class="card-icon">✍️</div>
        <div class="card-title">Writer</div>
        <div class="card-desc">Draft reports and docs</div>
      </div>
      <div class="diagram-card" style="flex: 1; min-width: 110px;">
        <div class="card-icon">🔍</div>
        <div class="card-title">Reviewer</div>
        <div class="card-desc">Quality check output</div>
      </div>
    </div>
  </div>
</div>
</div>

```python
# Supervisor pattern in LangGraph
from langgraph.graph import StateGraph, START, END

def supervisor(state):
    """Decides which agent to route to next."""
    response = llm.invoke(f"""
    You are a supervisor managing these agents: researcher, coder, writer.
    
    Current task: {state["task"]}
    Progress so far: {state["results"]}
    
    Which agent should work next? Or should we finish?
    Respond with: researcher, coder, writer, or FINISH
    """)
    return {"next_agent": response.content.strip().lower()}

def researcher(state):
    """Research agent with web search tools."""
    result = research_agent.invoke(state["task"])
    return {"results": state["results"] + [result]}

# Build the graph
graph = StateGraph(AgentState)
graph.add_node("supervisor", supervisor)
graph.add_node("researcher", researcher)
graph.add_node("coder", coder)
graph.add_node("writer", writer)

graph.add_edge(START, "supervisor")
graph.add_conditional_edges("supervisor", route_to_agent)
```

### 2. Swarm Pattern

Agents **hand off to each other** directly — no central coordinator. Each agent decides when to transfer control to another.

<div class="diagram">
<div class="diagram-title">Swarm Pattern</div>
<div class="cycle">
  <div class="cycle-step" style="border-color: var(--accent); color: var(--accent);">🤖 Triage Agent</div>
  <div class="cycle-arrow"></div>
  <div class="cycle-step" style="border-color: var(--green); color: var(--green);">🔧 Tech Support</div>
  <div class="cycle-arrow"></div>
  <div class="cycle-step" style="border-color: var(--blue); color: var(--blue);">💰 Billing Agent</div>
  <div class="cycle-arrow"></div>
  <div class="cycle-step" style="border-color: var(--purple); color: var(--purple);">📦 Returns Agent</div>
</div>
</div>

```python
# Swarm pattern (OpenAI Agents SDK style)
from agents import Agent, handoff

triage_agent = Agent(
    name="Triage",
    instructions="Route the customer to the right department.",
    handoffs=[
        handoff(tech_support_agent, "Technical issues"),
        handoff(billing_agent, "Billing questions"),
        handoff(returns_agent, "Returns and refunds"),
    ]
)

tech_support_agent = Agent(
    name="Tech Support",
    instructions="Help users with technical problems.",
    tools=[search_docs, check_status],
    handoffs=[handoff(triage_agent, "Not a tech issue")],
)
```

### 3. Debate Pattern

Two agents **argue** opposing positions. A judge agent evaluates and picks the best answer. Great for reducing hallucination and improving accuracy.

<div class="diagram">
<div class="diagram-title">Debate Pattern</div>
<div class="flow">
  <div class="flow-node accent wide">❓ Question / Task</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node wide" style="min-width: 100%;">
    <div style="display: flex; gap: 1rem; justify-content: center; flex-wrap: wrap;">
      <div class="diagram-card green" style="flex: 1; min-width: 150px;">
        <div class="card-icon">🟢</div>
        <div class="card-title">Agent A (Pro)</div>
        <div class="card-desc">Argues for position X</div>
      </div>
      <div class="diagram-card orange" style="flex: 1; min-width: 150px;">
        <div class="card-icon">🔴</div>
        <div class="card-title">Agent B (Con)</div>
        <div class="card-desc">Argues against X</div>
      </div>
    </div>
  </div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node purple wide">⚖️ Judge Agent: evaluates both arguments</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node blue wide">📋 Final verdict with reasoning</div>
</div>
</div>

### 4. Pipeline Pattern

Agents execute in **sequence**, each processing and transforming the output of the previous one.

<div class="diagram">
<div class="diagram-title">Pipeline Pattern</div>
<div class="flow-h">
  <div class="flow-node accent">📝 Draft Agent</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node green">🔍 Review Agent</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node blue">✨ Polish Agent</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node purple">✅ QA Agent</div>
</div>
</div>

## Communication Between Agents

Agents communicate through **messages** — structured data passed between them.

| Method | How It Works | Example |
|--------|-------------|---------|
| **Shared state** | All agents read/write to a common state object | LangGraph's state dict |
| **Message passing** | Agents send messages directly to each other | AutoGen conversations |
| **Handoffs** | One agent transfers the entire conversation to another | OpenAI Swarm SDK |
| **Blackboard** | Shared workspace anyone can write to | Research boards, shared docs |

## When to Use Multi-Agent

| Scenario | Single Agent | Multi-Agent |
|----------|-------------|-------------|
| Simple Q&A with search | ✅ | Overkill |
| Customer support routing | ❌ | ✅ Swarm pattern |
| Research report with code + writing | ❌ | ✅ Supervisor pattern |
| Code review | ❌ | ✅ Pipeline pattern |
| Complex decision making | ❌ | ✅ Debate pattern |

## Real-World Multi-Agent Systems

| Project | Architecture | What It Does |
|---------|-------------|-------------|
| [ChatDev](https://github.com/OpenBMB/ChatDev) | Pipeline | Agents role-play as CEO, CTO, programmer, tester to build software |
| [MetaGPT](https://github.com/geekan/MetaGPT) | Supervisor | Multi-agent framework for complex software projects |
| [AutoGen](https://github.com/microsoft/autogen) | Message passing | Microsoft's framework for multi-agent conversations |
| [CrewAI](https://github.com/crewAIInc/crewAI) | Task-based | Agents with roles, tools, and tasks — accessible multi-agent |

## What's Next

Now that you understand agent architectures from single to multi-agent, let's survey the frameworks and SDKs that make building agents practical.

**Next: [Chapter 9 — Agentic Frameworks & SDKs →](./09_agentic_frameworks.md)**

---

[← Previous: Chapter 7 — Planning & Reasoning](./07_planning_and_reasoning.md) · [Next: Chapter 9 — Agentic Frameworks & SDKs →](./09_agentic_frameworks.md)

*Last updated: April 2026*
