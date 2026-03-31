[← Back to Table of Contents](./README.md)

# Chapter 11 — Agentic Design Patterns

> *"The difference between a demo agent and a production agent is design patterns."*

## Why Patterns Matter

Building a single agent is easy. Building a **reliable, maintainable, cost-effective** agent system is hard. Design patterns are proven solutions to common agent architecture problems. Master these and you can design any agentic system.

## The Six Core Patterns

<div class="diagram">
<div class="diagram-title">Agentic Design Patterns</div>
<div class="diagram-grid cols-3">
  <div class="diagram-card accent">
    <div class="card-icon">🔀</div>
    <div class="card-title">Router</div>
    <div class="card-desc">Route to specialized handlers</div>
  </div>
  <div class="diagram-card green">
    <div class="card-icon">👔</div>
    <div class="card-title">Orchestrator-Worker</div>
    <div class="card-desc">Manager delegates to workers</div>
  </div>
  <div class="diagram-card blue">
    <div class="card-icon">🔍</div>
    <div class="card-title">Evaluator-Optimizer</div>
    <div class="card-desc">Generate, evaluate, improve</div>
  </div>
  <div class="diagram-card purple">
    <div class="card-icon">⚡</div>
    <div class="card-title">Parallelization</div>
    <div class="card-desc">Run tasks concurrently</div>
  </div>
  <div class="diagram-card orange">
    <div class="card-icon">👤</div>
    <div class="card-title">Human-in-the-Loop</div>
    <div class="card-desc">Pause for human approval</div>
  </div>
  <div class="diagram-card pink">
    <div class="card-icon">🛡️</div>
    <div class="card-title">Guardrails</div>
    <div class="card-desc">Constrain agent behavior</div>
  </div>
</div>
</div>

## Pattern 1: Router

A **router** classifies the input and directs it to the right specialist. It's the simplest pattern and the foundation of customer support, coding assistants, and knowledge systems.

<div class="diagram">
<div class="diagram-title">Router Pattern</div>
<div class="flow">
  <div class="flow-node accent wide">📥 User Input</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node purple wide">🔀 Router: classify intent</div>
  <div class="flow-arrow"></div>
  <div class="flow-node wide" style="min-width: 100%;">
    <div style="display: flex; gap: 0.75rem; justify-content: center; flex-wrap: wrap;">
      <div class="diagram-card green" style="flex: 1; min-width: 120px;">
        <div class="card-title">Technical</div>
        <div class="card-desc">Coding agent</div>
      </div>
      <div class="diagram-card blue" style="flex: 1; min-width: 120px;">
        <div class="card-title">Creative</div>
        <div class="card-desc">Writing agent</div>
      </div>
      <div class="diagram-card orange" style="flex: 1; min-width: 120px;">
        <div class="card-title">Factual</div>
        <div class="card-desc">Research agent</div>
      </div>
    </div>
  </div>
</div>
</div>

```python
def router(user_input: str) -> str:
    """Classify intent and route to the right agent."""
    classification = llm.invoke(f"""
    Classify this user request into one category:
    - technical: coding, debugging, architecture questions
    - creative: writing, brainstorming, content creation  
    - factual: research, data, current events
    
    Request: {user_input}
    
    Respond with just the category name.
    """)
    
    agents = {
        "technical": coding_agent,
        "creative": writing_agent,
        "factual": research_agent,
    }
    
    return agents[classification.content.strip()](user_input)
```

**When to use**: Multiple distinct capabilities, high request volume, different SLAs per category.

## Pattern 2: Orchestrator-Worker

A central **orchestrator** breaks down complex tasks and delegates sub-tasks to specialized **workers**.

<div class="diagram">
<div class="diagram-title">Orchestrator-Worker Pattern</div>
<div class="flow">
  <div class="flow-node accent wide">📋 "Build me a landing page"</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node purple wide">🎯 Orchestrator: decomposes into subtasks</div>
  <div class="flow-arrow"></div>
  <div class="flow-node wide" style="min-width: 100%;">
    <div style="display: flex; gap: 0.75rem; justify-content: center; flex-wrap: wrap;">
      <div class="diagram-card green" style="flex: 1; min-width: 130px;">
        <div class="card-title">Worker 1</div>
        <div class="card-desc">Design the layout</div>
      </div>
      <div class="diagram-card blue" style="flex: 1; min-width: 130px;">
        <div class="card-title">Worker 2</div>
        <div class="card-desc">Write the HTML/CSS</div>
      </div>
      <div class="diagram-card orange" style="flex: 1; min-width: 130px;">
        <div class="card-title">Worker 3</div>
        <div class="card-desc">Write the copy</div>
      </div>
    </div>
  </div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node purple wide">🎯 Orchestrator: assembles final result</div>
</div>
</div>

**When to use**: Complex tasks with multiple sub-skills, where each sub-task benefits from a focused specialist.

## Pattern 3: Evaluator-Optimizer

Generate a draft → evaluate its quality → improve it. Repeat until quality threshold is met. This is how coding agents fix bugs and how writing agents polish prose.

<div class="diagram">
<div class="diagram-title">Evaluator-Optimizer Loop</div>
<div class="flow">
  <div class="flow-node green wide">✏️ Generator: produce output</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node blue wide">🔍 Evaluator: score quality (0-10)</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node accent wide">Score ≥ 8? → Done ✅ | Score < 8? → ↓</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node orange wide">💡 Feedback: specific improvement suggestions</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node purple wide">🔄 Generator: revise using feedback</div>
</div>
</div>

```python
def evaluator_optimizer(task: str, max_rounds: int = 3, threshold: float = 8.0):
    output = generator.invoke(task)
    
    for round in range(max_rounds):
        # Evaluate
        evaluation = evaluator.invoke(f"""
        Task: {task}
        Output: {output}
        
        Rate the quality from 0-10 and provide specific feedback for improvement.
        Format: SCORE: X\nFEEDBACK: ...
        """)
        
        score = parse_score(evaluation)
        if score >= threshold:
            return output
        
        feedback = parse_feedback(evaluation)
        
        # Optimize
        output = generator.invoke(f"""
        Original task: {task}
        Previous output: {output}
        Feedback: {feedback}
        
        Improve the output based on the feedback.
        """)
    
    return output
```

**When to use**: Code generation (write → test → fix), content creation, data analysis reports.

## Pattern 4: Parallelization

Run independent tasks **simultaneously** to reduce latency. Two flavors:

<div class="diagram">
<div class="compare">
  <div class="compare-side left">
    <div class="compare-title">Sectioning (Split task)</div>
    <ul>
      <li>One task, split into independent parts</li>
      <li>"Research 3 companies" → 3 parallel searches</li>
      <li>Results are combined at the end</li>
      <li>Linear speedup with # of sections</li>
    </ul>
  </div>
  <div class="compare-side right">
    <div class="compare-title">Voting (Redundancy)</div>
    <ul>
      <li>Same task, run multiple times</li>
      <li>"Is this code secure?" → 3 agents vote</li>
      <li>Majority wins or disagreements flagged</li>
      <li>Higher reliability, higher cost</li>
    </ul>
  </div>
</div>
</div>

```python
import asyncio

async def parallel_research(topics: list[str]):
    """Research multiple topics in parallel."""
    tasks = [research_agent.ainvoke(topic) for topic in topics]
    results = await asyncio.gather(*tasks)
    return combine_results(results)

# Sectioning: 3x faster than sequential
results = await parallel_research([
    "Apple Q4 2025 earnings",
    "Google Q4 2025 earnings",
    "Microsoft Q4 2025 earnings",
])
```

## Pattern 5: Human-in-the-Loop

Pause agent execution at **critical checkpoints** for human approval. Essential for actions with real consequences.

<div class="diagram">
<div class="diagram-title">Human-in-the-Loop Pattern</div>
<div class="flow">
  <div class="flow-node accent wide">🤖 Agent: plan generated</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node orange wide">⏸️ Checkpoint: "I want to send an email to 500 customers"</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node blue wide">👤 Human reviews and approves/rejects</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node green wide">✅ Approved → Continue | ❌ Rejected → Replan</div>
</div>
</div>

```python
# LangGraph human-in-the-loop
from langgraph.graph import StateGraph

def should_continue(state):
    """Check if the next action requires human approval."""
    last_action = state["pending_action"]
    
    REQUIRES_APPROVAL = ["send_email", "delete_data", "deploy", "purchase"]
    
    if last_action["tool"] in REQUIRES_APPROVAL:
        return "human_review"
    return "execute"

graph.add_conditional_edges("plan", should_continue, {
    "human_review": "wait_for_approval",
    "execute": "execute_action",
})
```

**When to use**: Any action that costs money, sends communications, modifies production data, or is irreversible.

## Pattern 6: Guardrails

Constrain what the agent can do — **before** it acts, not after.

<div class="diagram">
<div class="diagram-grid cols-2">
  <div class="diagram-card accent">
    <div class="card-icon">🔒</div>
    <div class="card-title">Input Guardrails</div>
    <div class="card-desc">Validate user input before the agent sees it. Block prompt injection, off-topic requests, PII exposure.</div>
  </div>
  <div class="diagram-card green">
    <div class="card-icon">🛡️</div>
    <div class="card-title">Output Guardrails</div>
    <div class="card-desc">Validate agent output before the user sees it. Block harmful content, hallucinated URLs, leaked secrets.</div>
  </div>
  <div class="diagram-card blue">
    <div class="card-icon">🔧</div>
    <div class="card-title">Tool Guardrails</div>
    <div class="card-desc">Restrict which tools can be called, with what arguments, and how often. Rate limits, allowlists.</div>
  </div>
  <div class="diagram-card orange">
    <div class="card-icon">💰</div>
    <div class="card-title">Budget Guardrails</div>
    <div class="card-desc">Cap total token usage, API costs, and execution time. Prevent runaway agents.</div>
  </div>
</div>
</div>

```python
# OpenAI Agents SDK guardrails
from agents import Agent, GuardrailFunctionOutput, input_guardrail

@input_guardrail  
async def block_off_topic(ctx, agent, input):
    """Reject requests that aren't related to research."""
    result = await llm.invoke(f"""
    Is this request related to research, analysis, or information gathering?
    Request: {input}
    Respond with YES or NO.
    """)
    
    if "NO" in result.content:
        return GuardrailFunctionOutput(
            output_info="I can only help with research-related questions.",
            tripwire_triggered=True,
        )

agent = Agent(
    name="Researcher",
    instructions="...",
    input_guardrails=[block_off_topic],
)
```

## Pattern Combinations

Real systems combine multiple patterns:

| System | Patterns Used |
|--------|-------------|
| **Customer support bot** | Router + Swarm (handoffs) + Human-in-the-loop |
| **Coding assistant** | Evaluator-Optimizer (write → test → fix) + Guardrails |
| **Research pipeline** | Orchestrator-Worker + Parallelization + Evaluator |
| **Content platform** | Router + Parallelization + Human-in-the-loop |

## Decision Framework

<div class="diagram">
<div class="diagram-title">Which Pattern Do You Need?</div>
<div class="flow">
  <div class="flow-node accent wide">What's your challenge?</div>
  <div class="flow-arrow"></div>
  <div class="flow-node wide" style="min-width: 100%;">
    <div style="display: flex; gap: 0.75rem; justify-content: center; flex-wrap: wrap;">
      <div class="diagram-card" style="flex: 1; min-width: 140px;">
        <div class="card-title">Many request types</div>
        <div class="card-desc">→ Router</div>
      </div>
      <div class="diagram-card accent" style="flex: 1; min-width: 140px;">
        <div class="card-title">Complex task</div>
        <div class="card-desc">→ Orchestrator-Worker</div>
      </div>
      <div class="diagram-card green" style="flex: 1; min-width: 140px;">
        <div class="card-title">Quality matters</div>
        <div class="card-desc">→ Evaluator-Optimizer</div>
      </div>
      <div class="diagram-card blue" style="flex: 1; min-width: 140px;">
        <div class="card-title">Slow response</div>
        <div class="card-desc">→ Parallelization</div>
      </div>
    </div>
  </div>
</div>
</div>

## What's Next

Patterns tell you *how* to build. But how do you make agents reliable and observable in **production**? That's deployment.

**Next: [Chapter 12 — Deployment & Production →](./12_deployment_and_production.md)**

---

[← Previous: Chapter 10 — Build Your First Agent](./10_build_your_first_agent.md) · [Next: Chapter 12 — Deployment & Production →](./12_deployment_and_production.md)

*Last updated: April 2026*
