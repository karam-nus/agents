[← Back to Table of Contents](./README.md)

# Chapter 3 — Anatomy of an Agent

> *"An agent is just an LLM that runs in a loop, using tools, and checking its work."*
> — Harrison Chase, creator of LangChain

## The Four Pillars

Every modern AI agent is built from four core components. Think of it like the human analogy: a brain, hands, memory, and a plan.

<div class="diagram">
<div class="diagram-title">The Four Pillars of an AI Agent</div>
<div class="diagram-grid cols-4">
  <div class="diagram-card accent">
    <div class="card-icon">🧠</div>
    <div class="card-title">LLM (Brain)</div>
    <div class="card-desc">Reasoning engine. Understands language, makes decisions, generates plans.</div>
  </div>
  <div class="diagram-card green">
    <div class="card-icon">🔧</div>
    <div class="card-title">Tools (Hands)</div>
    <div class="card-desc">Functions the agent can call — search, code execution, APIs, databases.</div>
  </div>
  <div class="diagram-card blue">
    <div class="card-icon">💾</div>
    <div class="card-title">Memory</div>
    <div class="card-desc">Short-term (conversation), long-term (vector store), episodic (past experiences).</div>
  </div>
  <div class="diagram-card purple">
    <div class="card-icon">📋</div>
    <div class="card-title">Planning</div>
    <div class="card-desc">Task decomposition, step sequencing, self-correction, goal tracking.</div>
  </div>
</div>
</div>

## Pillar 1: The LLM (Brain)

The LLM is the reasoning engine. It:

- **Understands** the user's goal from natural language
- **Decides** what to do next at each step
- **Generates** tool calls, code, or natural language responses
- **Evaluates** whether results are satisfactory

### Which LLM for Agents?

Not all LLMs are equal for agent tasks. Key requirements:

| Capability | Why It Matters |
|-----------|----------------|
| **Function calling** | Must reliably output structured JSON for tool invocations |
| **Instruction following** | Must follow system prompts precisely — agents live or die by this |
| **Long context** | Must handle growing conversation + tool results (32K+ tokens) |
| **Reasoning quality** | Multi-step problems require strong logical reasoning |

**Best choices for agents (2026)**: GPT-4o, Claude 3.5/4, Gemini 2.0, Llama 3.3 (local), Qwen 2.5

### The System Prompt Is Everything

For agents, the system prompt defines the agent's **identity, capabilities, and constraints**:

```text
You are a research assistant agent. You have access to the following tools:
- web_search: Search the internet for current information
- read_file: Read contents of a local file
- write_file: Write content to a local file

RULES:
1. Always search before answering factual questions
2. Never modify files without user confirmation
3. If unsure, ask the user for clarification
4. Cite your sources
```

## Pillar 2: Tools (Hands)

Tools give agents the ability to **interact with the world**. Without tools, an agent is just a chatbot.

<div class="diagram">
<div class="diagram-title">Common Agent Tools</div>
<div class="diagram-grid cols-3">
  <div class="diagram-card">
    <div class="card-icon">🔍</div>
    <div class="card-title">Web Search</div>
    <div class="card-desc">Google, Bing, Tavily — access to current information</div>
  </div>
  <div class="diagram-card">
    <div class="card-icon">💻</div>
    <div class="card-title">Code Execution</div>
    <div class="card-desc">Python sandbox, shell commands — compute and transform data</div>
  </div>
  <div class="diagram-card">
    <div class="card-icon">📁</div>
    <div class="card-title">File I/O</div>
    <div class="card-desc">Read, write, edit files on disk — persistent output</div>
  </div>
  <div class="diagram-card">
    <div class="card-icon">🌐</div>
    <div class="card-title">API Calls</div>
    <div class="card-desc">REST/GraphQL endpoints — interact with external services</div>
  </div>
  <div class="diagram-card">
    <div class="card-icon">🗄️</div>
    <div class="card-title">Database Queries</div>
    <div class="card-desc">SQL/NoSQL — read and write structured data</div>
  </div>
  <div class="diagram-card">
    <div class="card-icon">🖥️</div>
    <div class="card-title">Computer Use</div>
    <div class="card-desc">Click, type, screenshot — control GUI applications</div>
  </div>
</div>
</div>

A tool is defined as a **function schema** that tells the LLM what it can do:

```python
tools = [{
    "type": "function",
    "function": {
        "name": "web_search",
        "description": "Search the web for current information on a topic",
        "parameters": {
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "The search query"
                }
            },
            "required": ["query"]
        }
    }
}]
```

> **Deep dive**: [Chapter 5 — Tools & Function Calling](./05_tools_and_function_calling.md)

## Pillar 3: Memory

Agents need memory to maintain context, learn from past interactions, and access knowledge.

<div class="diagram">
<div class="diagram-title">Memory Architecture</div>
<div class="layer-stack">
  <div class="layer purple">🧠 Working Memory <small>Current context window — the LLM's "attention span" (32K–1M tokens)</small></div>
  <div class="layer blue">💬 Short-Term Memory <small>Conversation history — what's happened in this session</small></div>
  <div class="layer accent">📚 Long-Term Memory <small>Vector database — persistent knowledge, past experiences, user preferences</small></div>
  <div class="layer orange">🌍 External Knowledge <small>RAG — search over documents, APIs, databases at retrieval time</small></div>
</div>
</div>

The key challenge is **context window management** — as the conversation grows, you run out of space. Solutions:

- **Summarization**: Condense older messages into a summary
- **Sliding window**: Keep only the last N messages
- **RAG**: Store everything in a vector DB, retrieve only what's relevant
- **Hybrid**: Summarize old context + retrieve specific facts on demand

> **Deep dive**: [Chapter 6 — Memory Systems](./06_memory_systems.md)

## Pillar 4: Planning

Planning is how agents break complex goals into manageable steps.

### Plan-and-Execute Pattern

<div class="diagram">
<div class="diagram-title">Plan-and-Execute Pattern</div>
<div class="flow">
  <div class="flow-node accent wide">📋 Create plan from user goal</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node green wide">▶️ Execute step 1</div>
  <div class="flow-arrow"></div>
  <div class="flow-node green wide">▶️ Execute step 2</div>
  <div class="flow-arrow"></div>
  <div class="flow-node blue wide">🔍 Evaluate: on track?</div>
  <div class="flow-arrow"></div>
  <div class="flow-node purple wide">🔄 Replan if needed</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node orange wide">✅ Goal achieved</div>
</div>
</div>

```python
# Simplified plan-and-execute
plan = agent.create_plan("Write a blog post about quantum computing")
# plan = ["1. Research recent quantum computing breakthroughs",
#          "2. Outline the blog structure",
#          "3. Write the draft",
#          "4. Review and edit"]

for step in plan:
    result = agent.execute(step)
    if not agent.evaluate(result):
        plan = agent.replan(result, remaining_steps)
```

> **Deep dive**: [Chapter 7 — Planning & Reasoning](./07_planning_and_reasoning.md)

## How It All Fits Together

<div class="diagram">
<div class="diagram-title">Complete Agent Architecture</div>
<div class="flow">
  <div class="flow-node accent wide">👤 User sends a goal</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node purple wide">📋 Planning: decompose into steps</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node blue wide">🧠 LLM: reason about next action</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node green wide">🔧 Tools: execute the action</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node orange wide">💾 Memory: store result, update context</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node wide">🔄 Loop back to LLM or return result to user</div>
</div>
</div>

## Real-World Example: A Coding Agent

Let's trace how GitHub Copilot Workspace (a real agent) handles "Add dark mode to my app":

1. **Planning**: Reads codebase → identifies relevant files → creates a multi-step plan
2. **LLM reasoning**: "I need to modify the CSS variables and add a toggle component"
3. **Tool use**: Reads files, writes new code, runs the linter
4. **Memory**: Keeps track of which files were changed, what errors occurred
5. **Self-correction**: Linter reports an error → agent reads the error → fixes the code → reruns

This is the anatomy in action. Every agent, from a simple chatbot with search to a multi-agent coding system, builds on these four pillars.

## What's Next

Now let's look at the **engine** that drives all this — the agent loop. How does an agent decide when to think, when to act, and when to stop?

**Next: [Chapter 4 — The Agent Loop →](./04_the_agent_loop.md)**

---

[← Previous: Chapter 2 — History & Evolution](./02_history_and_evolution.md) · [Next: Chapter 4 — The Agent Loop →](./04_the_agent_loop.md)

*Last updated: April 2026*
