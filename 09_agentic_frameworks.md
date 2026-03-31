[← Back to Table of Contents](./README.md)

# Chapter 9 — Agentic Frameworks & SDKs

> *"Don't build the plumbing — build the agent. Frameworks handle the rest."*

## The Framework Landscape

You don't need to build agent loops, tool registries, and memory systems from scratch. Frameworks handle the infrastructure so you can focus on what your agent does. Here's the 2026 landscape:

<div class="diagram">
<div class="diagram-title">Framework Comparison Map</div>
<div class="diagram-grid cols-2">
  <div class="diagram-card accent">
    <div class="card-icon">🦜</div>
    <div class="card-title">LangGraph</div>
    <div class="card-desc">Graph-based orchestration. Maximum flexibility. Industry standard for complex workflows.</div>
  </div>
  <div class="diagram-card green">
    <div class="card-icon">🤖</div>
    <div class="card-title">OpenAI Agents SDK</div>
    <div class="card-desc">Simple, opinionated. Best for OpenAI models. Great for starting out.</div>
  </div>
  <div class="diagram-card blue">
    <div class="card-icon">👥</div>
    <div class="card-title">CrewAI</div>
    <div class="card-desc">Role-based multi-agent. Easy to conceptualize. Business-friendly API.</div>
  </div>
  <div class="diagram-card purple">
    <div class="card-icon">💬</div>
    <div class="card-title">AutoGen</div>
    <div class="card-desc">Microsoft's conversational agents. Message-passing architecture. Strong multi-agent.</div>
  </div>
  <div class="diagram-card orange">
    <div class="card-icon">🤗</div>
    <div class="card-title">Smolagents</div>
    <div class="card-desc">Hugging Face's lightweight approach. Code-first, minimal abstraction.</div>
  </div>
  <div class="diagram-card">
    <div class="card-icon">▲</div>
    <div class="card-title">Vercel AI SDK</div>
    <div class="card-desc">TypeScript-first. Best for web/Next.js. Streaming-native.</div>
  </div>
</div>
</div>

## Head-to-Head Comparison

| Feature | LangGraph | OpenAI SDK | CrewAI | AutoGen | Smolagents |
|---------|-----------|------------|--------|---------|------------|
| **Language** | Python/JS | Python | Python | Python | Python |
| **Learning curve** | Steep | Easy | Easy | Medium | Easy |
| **Multi-agent** | Excellent | Good (handoffs) | Excellent | Excellent | Basic |
| **Model support** | Any LLM | OpenAI only | Any LLM | Any LLM | Any LLM |
| **Streaming** | Yes | Yes | Limited | Yes | Limited |
| **State management** | Built-in (graph state) | Basic | Task-based | Conversation | Minimal |
| **Human-in-the-loop** | First-class | Guardrails | Callbacks | Built-in | Manual |
| **Production-ready** | Yes | Yes | Growing | Yes | Experimental |
| **Best for** | Complex workflows | Quick prototypes | Business teams | Research | Learning |

## Framework 1: LangGraph

The most flexible framework. Agents are defined as **state machines** (graphs) where nodes are functions and edges define the flow.

```python
from langgraph.graph import StateGraph, START, END, MessagesState
from langgraph.prebuilt import ToolNode, tools_condition
from langchain_openai import ChatOpenAI

# Define tools
def search(query: str) -> str:
    """Search the web for information."""
    return tavily_client.search(query)

def calculator(expression: str) -> str:
    """Evaluate a math expression."""
    return str(eval(expression))  # simplified

tools = [search, calculator]
llm = ChatOpenAI(model="gpt-4o").bind_tools(tools)

# Define the agent node
def agent(state: MessagesState):
    return {"messages": [llm.invoke(state["messages"])]}

# Build the graph
graph = StateGraph(MessagesState)
graph.add_node("agent", agent)
graph.add_node("tools", ToolNode(tools))

graph.add_edge(START, "agent")
graph.add_conditional_edges("agent", tools_condition)  # agent → tools or END
graph.add_edge("tools", "agent")  # tools → back to agent

app = graph.compile()

# Run it
result = app.invoke({"messages": [("user", "What is 25 * 73?")]})
```

### Why LangGraph?

- **State machines are visual** — you can see the agent's control flow as a graph
- **Checkpointing** — save and resume agent state (critical for long-running agents)
- **Human-in-the-loop** — pause the graph, ask the user, resume
- **Subgraphs** — compose complex agents from smaller agents

## Framework 2: OpenAI Agents SDK

The simplest path from zero to working agent. Opinionated, focused on OpenAI models.

```python
from agents import Agent, Runner, function_tool

@function_tool
def search_web(query: str) -> str:
    """Search the web for current information."""
    return tavily_client.search(query)

@function_tool
def read_file(path: str) -> str:
    """Read the contents of a file."""
    with open(path) as f:
        return f.read()

agent = Agent(
    name="Research Assistant",
    instructions="""You are a helpful research assistant. 
    Search the web for factual answers. Always cite your sources.""",
    tools=[search_web, read_file],
    model="gpt-4o",
)

# Run synchronously
result = Runner.run_sync(agent, "What are the latest advances in quantum computing?")
print(result.final_output)
```

### Handoffs with OpenAI SDK

```python
from agents import Agent, handoff

math_agent = Agent(
    name="Math Tutor",
    instructions="Help with math problems. Show your work step by step.",
    tools=[calculator],
)

writing_agent = Agent(
    name="Writing Coach",
    instructions="Help improve writing. Give specific, actionable feedback.",
)

triage_agent = Agent(
    name="Triage",
    instructions="Determine if the user needs math help or writing help.",
    handoffs=[
        handoff(math_agent, "Math questions or calculations"),
        handoff(writing_agent, "Writing or grammar questions"),
    ],
)

result = Runner.run_sync(triage_agent, "Can you help me solve x^2 + 5x + 6 = 0?")
# → Hands off to math_agent
```

## Framework 3: CrewAI

Role-based multi-agent. You define agents as team members with roles, goals, and backstories.

```python
from crewai import Agent, Task, Crew

# Define agents with roles
researcher = Agent(
    role="Senior Research Analyst",
    goal="Find comprehensive, accurate information on the topic",
    backstory="You're a veteran analyst with 20 years of experience.",
    tools=[search_tool, scrape_tool],
    llm="gpt-4o",
)

writer = Agent(
    role="Technical Writer",
    goal="Create clear, engaging content from research findings",
    backstory="You write for top tech publications.",
    llm="gpt-4o",
)

# Define tasks
research_task = Task(
    description="Research the latest developments in AI agents for 2026",
    agent=researcher,
    expected_output="Detailed research notes with sources",
)

writing_task = Task(
    description="Write a blog post based on the research",
    agent=writer,
    expected_output="A polished 1000-word blog post",
    context=[research_task],  # depends on research
)

# Create and run the crew
crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, writing_task],
    verbose=True,
)

result = crew.kickoff()
```

## Framework 4: AutoGen

Microsoft's framework for multi-agent conversations. Agents communicate by sending messages to each other.

```python
from autogen import ConversableAgent

assistant = ConversableAgent(
    name="assistant",
    system_message="You are a helpful AI assistant.",
    llm_config={"model": "gpt-4o"},
)

user_proxy = ConversableAgent(
    name="user_proxy",
    human_input_mode="NEVER",
    code_execution_config={"work_dir": "coding"},
)

# Agents chat with each other
user_proxy.initiate_chat(
    assistant,
    message="Write a Python function to find prime numbers using the Sieve of Eratosthenes.",
)
```

## Framework 5: Smolagents (Hugging Face)

Minimal, code-first. Agents write Python code instead of making tool calls — the code *is* the action.

```python
from smolagents import CodeAgent, HfApiModel, tool

@tool
def get_weather(city: str) -> str:
    """Get the current weather for a city."""
    # ... implementation
    return f"{city}: 72°F, Sunny"

agent = CodeAgent(
    tools=[get_weather],
    model=HfApiModel("Qwen/Qwen2.5-72B-Instruct"),
)

result = agent.run("What's the weather like in Tokyo?")
```

## Choosing a Framework

<div class="diagram">
<div class="diagram-title">Framework Decision Tree</div>
<div class="flow">
  <div class="flow-node accent wide">What's your priority?</div>
  <div class="flow-arrow"></div>
  <div class="flow-node wide" style="min-width: 100%;">
    <div style="display: flex; gap: 0.75rem; justify-content: center; flex-wrap: wrap;">
      <div class="diagram-card green" style="flex: 1; min-width: 130px;">
        <div class="card-title">Speed to MVP</div>
        <div class="card-desc">→ OpenAI Agents SDK</div>
      </div>
      <div class="diagram-card accent" style="flex: 1; min-width: 130px;">
        <div class="card-title">Max Flexibility</div>
        <div class="card-desc">→ LangGraph</div>
      </div>
      <div class="diagram-card blue" style="flex: 1; min-width: 130px;">
        <div class="card-title">Multi-Agent</div>
        <div class="card-desc">→ CrewAI or AutoGen</div>
      </div>
      <div class="diagram-card purple" style="flex: 1; min-width: 130px;">
        <div class="card-title">Learning</div>
        <div class="card-desc">→ Smolagents</div>
      </div>
    </div>
  </div>
</div>
</div>

## What's Next

You've surveyed the frameworks. Now let's get hands-on — in the next chapter, you'll build a working research assistant agent from scratch using two frameworks side by side.

**Next: [Chapter 10 — Build Your First Agent →](./10_build_your_first_agent.md)**

---

[← Previous: Chapter 8 — Multi-Agent Systems](./08_multi_agent_systems.md) · [Next: Chapter 10 — Build Your First Agent →](./10_build_your_first_agent.md)

*Last updated: April 2026*
