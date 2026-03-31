[← Back to Table of Contents](./README.md)

# Chapter 7 — Planning & Reasoning

> *"The real challenge for agents isn't executing any single step — it's knowing which steps to take, in what order, and when to change course."*

## Why Planning Matters

An LLM can answer questions. An agent with tools can take actions. But **planning** is what separates a useful agent from a random one. Planning is the ability to:

- Break a complex goal into smaller steps
- Sequence those steps correctly
- Adapt when something goes wrong
- Know when the goal is achieved

## The Reasoning Landscape

<div class="diagram">
<div class="diagram-title">Reasoning Techniques for Agents</div>
<div class="layer-stack">
  <div class="layer purple">🌳 Tree of Thoughts <small>Explore multiple reasoning branches, evaluate and prune</small></div>
  <div class="layer blue">🔄 Reflexion <small>Reflect on past failures, adjust strategy, retry</small></div>
  <div class="layer accent">⚡ ReAct <small>Interleave reasoning (think) with acting (tool use)</small></div>
  <div class="layer orange">🔗 Chain-of-Thought <small>Step-by-step reasoning before answering</small></div>
  <div class="layer">💬 Direct Prompting <small>Just answer — no explicit reasoning</small></div>
</div>
</div>

## Chain-of-Thought (CoT)

The simplest reasoning technique. Ask the LLM to **think step by step** before answering.

```python
# Without CoT
prompt = "What is 27 * 43?"
# LLM might get it wrong

# With CoT
prompt = """What is 27 * 43? Think step by step.

Step 1: 27 * 40 = 1,080
Step 2: 27 * 3 = 81
Step 3: 1,080 + 81 = 1,161

The answer is 1,161."""
```

For agents, CoT is embedded in the system prompt:

```text
Before taking any action, think through your reasoning step by step:
1. What is the user's goal?
2. What information do I need?
3. What tools should I use?
4. What's my plan?

Then execute the plan one step at a time.
```

## ReAct: Reasoning + Acting

**ReAct** is the core reasoning pattern for tool-using agents. It interleaves **thoughts** (reasoning) with **actions** (tool calls) and **observations** (tool results).

<div class="diagram">
<div class="diagram-title">ReAct Trace</div>
<div class="flow">
  <div class="flow-node accent wide">💭 Thought: "I need to find the CEO of Tesla"</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node green wide">⚡ Action: web_search("CEO of Tesla 2026")</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node blue wide">👀 Observation: "Elon Musk is the CEO of Tesla"</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node accent wide">💭 Thought: "Now I need their net worth"</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node green wide">⚡ Action: web_search("Elon Musk net worth 2026")</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node blue wide">👀 Observation: "~$250 billion"</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node orange wide">✅ Answer: Formulates final response</div>
</div>
</div>

In production, you implement ReAct by including a "think" step in the system prompt:

```python
system_prompt = """You are a research assistant. For each step:

1. THINK: Explain your reasoning about what to do next
2. ACT: Call a tool if needed
3. OBSERVE: Analyze the tool's output
4. REPEAT until you can answer the user's question

Always show your thinking before acting."""
```

## Plan-and-Execute

For complex tasks, it's better to **plan first, then execute** — rather than figuring things out one step at a time.

<div class="diagram">
<div class="diagram-title">Plan-and-Execute Pattern</div>
<div class="flow-h">
  <div class="flow-node accent">📋 Plan</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node green">▶️ Step 1</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node green">▶️ Step 2</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node blue">🔍 Check</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node purple">🔄 Replan?</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node orange">✅ Done</div>
</div>
</div>

```python
def plan_and_execute(goal: str):
    # Step 1: Generate a plan
    plan = llm.generate(f"""
    Create a step-by-step plan to achieve this goal: {goal}
    
    Output as a numbered list. Each step should be specific and actionable.
    """)
    
    steps = parse_plan(plan)
    results = []
    
    for i, step in enumerate(steps):
        # Step 2: Execute each step
        result = agent.execute(step)
        results.append(result)
        
        # Step 3: Check progress
        evaluation = llm.generate(f"""
        Goal: {goal}
        Completed steps: {results}
        Remaining steps: {steps[i+1:]}
        
        Is the plan still on track? Should we modify remaining steps?
        """)
        
        if "replan" in evaluation.lower():
            steps = replan(goal, results, steps[i+1:])
    
    return synthesize_answer(goal, results)
```

## Reflexion: Learning from Mistakes

**Reflexion** adds a self-reflection loop. When the agent fails, it analyzes *why* and adjusts its approach.

<div class="diagram">
<div class="diagram-title">Reflexion Loop</div>
<div class="flow">
  <div class="flow-node accent wide">📋 Attempt the task</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node blue wide">🔍 Evaluate: did it work?</div>
  <div class="flow-arrow"></div>
  <div class="flow-node green wide">✅ Success → return result</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node orange wide">❌ Failure → reflect on what went wrong</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node purple wide">💡 Generate improved strategy</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node wide">🔄 Retry with new strategy</div>
</div>
</div>

```python
def reflexion_loop(task: str, max_retries: int = 3):
    reflections = []
    
    for attempt in range(max_retries):
        # Attempt the task (with past reflections as context)
        result = agent.execute(task, past_reflections=reflections)
        
        # Evaluate
        success = evaluate(result, task)
        if success:
            return result
        
        # Reflect on the failure
        reflection = llm.generate(f"""
        Task: {task}
        Attempt #{attempt + 1} failed.
        Result: {result}
        
        What went wrong? What should I do differently next time?
        Be specific and actionable.
        """)
        
        reflections.append(reflection)
    
    return "Failed after maximum retries."
```

## Tree of Thoughts (ToT)

For problems with multiple valid approaches, **Tree of Thoughts** explores different reasoning paths and picks the best one.

<div class="diagram">
<div class="diagram-title">Tree of Thoughts</div>
<div class="flow">
  <div class="flow-node accent wide">🌱 Problem: "Design a REST API for a todo app"</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node wide" style="min-width: 100%;">
    <div style="display: flex; gap: 1rem; justify-content: center; flex-wrap: wrap;">
      <div class="diagram-card green" style="flex: 1; min-width: 140px;">
        <div class="card-title">Path A</div>
        <div class="card-desc">RESTful with CRUD endpoints</div>
      </div>
      <div class="diagram-card blue" style="flex: 1; min-width: 140px;">
        <div class="card-title">Path B</div>
        <div class="card-desc">GraphQL with single endpoint</div>
      </div>
      <div class="diagram-card purple" style="flex: 1; min-width: 140px;">
        <div class="card-title">Path C</div>
        <div class="card-desc">Event-driven with WebSocket</div>
      </div>
    </div>
  </div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node orange wide">🏆 Evaluate each → Pick best → Expand</div>
</div>
</div>

## Task Decomposition

Breaking complex tasks into sub-tasks is a planning superpower:

```python
def decompose_task(complex_task: str) -> list[str]:
    return llm.generate(f"""
    Break this complex task into 3-7 simple, independent sub-tasks:
    
    Task: {complex_task}
    
    Rules:
    - Each sub-task should be completable with a single tool call
    - Sub-tasks should be ordered by dependency
    - Include a final "verify" sub-task
    
    Output as a JSON array of strings.
    """)

# Example
subtasks = decompose_task("Create a data analysis report on Q4 sales")
# ["Search for Q4 sales data files",
#  "Load and clean the sales data",
#  "Calculate key metrics (total revenue, growth rate, top products)", 
#  "Generate visualizations (bar chart, trend line)",
#  "Write the report narrative",
#  "Format as PDF with charts embedded",
#  "Verify all numbers match the source data"]
```

## Choosing the Right Reasoning Strategy

| Strategy | Best For | Overhead | Reliability |
|----------|---------|----------|-------------|
| **Direct** | Simple, single-step tasks | None | Low |
| **CoT** | Math, logic, step-by-step problems | Low | Medium |
| **ReAct** | Tasks requiring tool use | Medium | High |
| **Plan-and-Execute** | Complex multi-step projects | Medium | High |
| **Reflexion** | Tasks where failure is likely/acceptable | High | Very High |
| **Tree of Thoughts** | Creative/design tasks with multiple valid solutions | Very High | Highest |

<div class="diagram">
<div class="diagram-title">Decision Guide: Which Reasoning Strategy?</div>
<div class="flow">
  <div class="flow-node accent wide">How complex is the task?</div>
  <div class="flow-arrow"></div>
  <div class="flow-node wide" style="min-width: 100%;">
    <div style="display: flex; gap: 1rem; justify-content: center; flex-wrap: wrap;">
      <div class="diagram-card" style="flex: 1; min-width: 120px;">
        <div class="card-title">Simple</div>
        <div class="card-desc">→ CoT or Direct</div>
      </div>
      <div class="diagram-card accent" style="flex: 1; min-width: 120px;">
        <div class="card-title">Medium</div>
        <div class="card-desc">→ ReAct</div>
      </div>
      <div class="diagram-card blue" style="flex: 1; min-width: 120px;">
        <div class="card-title">Complex</div>
        <div class="card-desc">→ Plan-and-Execute</div>
      </div>
      <div class="diagram-card purple" style="flex: 1; min-width: 120px;">
        <div class="card-title">Error-prone</div>
        <div class="card-desc">→ Reflexion</div>
      </div>
    </div>
  </div>
</div>
</div>

## What's Next

So far we've covered single agents. But what happens when you need **multiple agents** working together? That's multi-agent systems.

**Next: [Chapter 8 — Multi-Agent Systems →](./08_multi_agent_systems.md)**

---

[← Previous: Chapter 6 — Memory Systems](./06_memory_systems.md) · [Next: Chapter 8 — Multi-Agent Systems →](./08_multi_agent_systems.md)

*Last updated: April 2026*
