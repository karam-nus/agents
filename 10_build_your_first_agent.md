[← Back to Table of Contents](./README.md)

# Chapter 10 — Build Your First Agent

> *"The best way to learn agents is to build one."*

## What We're Building

A **Research Assistant Agent** that can:
1. Search the web for information
2. Read and summarize web pages
3. Write findings to a file
4. Answer follow-up questions based on its research

We'll build it **twice** — once with the **OpenAI Agents SDK** (simple) and once with **LangGraph** (flexible) — so you can compare.

## Prerequisites

```bash
# Create a project environment
mkdir my-first-agent && cd my-first-agent
python -m venv venv
source venv/bin/activate  # macOS/Linux
# venv\Scripts\activate   # Windows

# Install dependencies (pick your framework)
pip install openai-agents tavily-python   # For OpenAI Agents SDK
pip install langgraph langchain-openai tavily-python  # For LangGraph

# Set your API keys
export OPENAI_API_KEY="sk-..."
export TAVILY_API_KEY="tvly-..."
```

> **Tip**: Get a free Tavily API key at [tavily.com](https://tavily.com). For OpenAI, you need an API key from [platform.openai.com](https://platform.openai.com).

## Implementation 1: OpenAI Agents SDK

### Step 1: Define Tools

```python
# tools.py
import json
from datetime import datetime
from agents import function_tool
from tavily import TavilyClient
import os

tavily = TavilyClient(api_key=os.environ["TAVILY_API_KEY"])

@function_tool
def search_web(query: str) -> str:
    """Search the web for current information on a topic.
    Use this when you need up-to-date facts, news, or data.
    Returns a summary of the top search results."""
    results = tavily.search(query, max_results=5)
    output = []
    for r in results["results"]:
        output.append(f"**{r['title']}**\n{r['content']}\nSource: {r['url']}\n")
    return "\n---\n".join(output)

@function_tool
def save_to_file(filename: str, content: str) -> str:
    """Save research findings to a markdown file.
    Use this when the user asks you to save or export your findings."""
    # Sanitize filename
    safe_name = "".join(c for c in filename if c.isalnum() or c in "._- ")
    filepath = f"output/{safe_name}"
    os.makedirs("output", exist_ok=True)
    with open(filepath, "w") as f:
        f.write(f"# Research: {filename}\n")
        f.write(f"*Generated on {datetime.now().strftime('%Y-%m-%d %H:%M')}*\n\n")
        f.write(content)
    return f"Saved to {filepath}"

@function_tool
def get_current_date() -> str:
    """Get the current date and time. Use this when you need to know today's date."""
    return datetime.now().strftime("%Y-%m-%d %H:%M:%S")
```

### Step 2: Create the Agent

```python
# agent.py
from agents import Agent, Runner
from tools import search_web, save_to_file, get_current_date

research_agent = Agent(
    name="Research Assistant",
    instructions="""You are an expert research assistant. Your job is to help users 
    find accurate, up-to-date information on any topic.

    WORKFLOW:
    1. When given a research question, break it into specific search queries
    2. Search for information using the search_web tool
    3. Synthesize findings into a clear, well-organized answer
    4. Always cite your sources with URLs
    5. If asked to save findings, use the save_to_file tool

    GUIDELINES:
    - Be thorough — search multiple aspects of a topic
    - Be honest — if you can't find something, say so
    - Be concise — summarize, don't just paste raw results
    - Always verify claims with multiple sources when possible
    """,
    tools=[search_web, save_to_file, get_current_date],
    model="gpt-4o",
)

# Run interactively
if __name__ == "__main__":
    import asyncio
    
    async def main():
        print("🤖 Research Assistant ready! Type 'quit' to exit.\n")
        
        while True:
            user_input = input("You: ").strip()
            if user_input.lower() in ("quit", "exit", "q"):
                break
            
            result = await Runner.run(research_agent, user_input)
            print(f"\n🤖 Assistant: {result.final_output}\n")
    
    asyncio.run(main())
```

### Step 3: Run It

```bash
python agent.py
```

```
🤖 Research Assistant ready! Type 'quit' to exit.

You: What are the most promising quantum computing companies in 2026?

🤖 Assistant: Based on my research, here are the most promising quantum computing 
companies in 2026:

1. **IBM** - Released their 1,121-qubit Condor processor and Heron chip...
   Source: https://...

2. **Google (Alphabet)** - Their Willow chip achieved quantum error correction...
   Source: https://...

3. **IonQ** - Leading trapped-ion approach with commercial deployments...
   Source: https://...

[... detailed findings with sources ...]

You: Save that to a file called "quantum-companies-2026.md"

🤖 Assistant: Saved to output/quantum-companies-2026.md
```

## Implementation 2: LangGraph

The same agent, built with LangGraph for more control over the loop.

### Step 1: Define the State and Tools

```python
# langgraph_agent.py
import os
import json
from datetime import datetime
from typing import Annotated

from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langgraph.graph import StateGraph, START, END, MessagesState
from langgraph.prebuilt import ToolNode, tools_condition
from langgraph.checkpoint.memory import MemorySaver
from tavily import TavilyClient

tavily = TavilyClient(api_key=os.environ["TAVILY_API_KEY"])

@tool
def search_web(query: str) -> str:
    """Search the web for current information on a topic."""
    results = tavily.search(query, max_results=5)
    output = []
    for r in results["results"]:
        output.append(f"**{r['title']}**\n{r['content']}\nSource: {r['url']}\n")
    return "\n---\n".join(output)

@tool
def save_to_file(filename: str, content: str) -> str:
    """Save research findings to a markdown file."""
    safe_name = "".join(c for c in filename if c.isalnum() or c in "._- ")
    filepath = f"output/{safe_name}"
    os.makedirs("output", exist_ok=True)
    with open(filepath, "w") as f:
        f.write(f"# Research: {filename}\n")
        f.write(f"*Generated on {datetime.now().strftime('%Y-%m-%d %H:%M')}*\n\n")
        f.write(content)
    return f"Saved to {filepath}"

tools = [search_web, save_to_file]
```

### Step 2: Build the Graph

```python
# Continue in langgraph_agent.py

llm = ChatOpenAI(model="gpt-4o").bind_tools(tools)

SYSTEM_PROMPT = """You are an expert research assistant. 
Search for accurate, up-to-date information. Always cite sources with URLs.
Be thorough — search multiple aspects of a topic.
Synthesize findings into clear, organized answers."""

def agent_node(state: MessagesState):
    """The thinking node — calls the LLM."""
    messages = [{"role": "system", "content": SYSTEM_PROMPT}] + state["messages"]
    response = llm.invoke(messages)
    return {"messages": [response]}

# Build the graph
graph = StateGraph(MessagesState)

# Add nodes
graph.add_node("agent", agent_node)
graph.add_node("tools", ToolNode(tools))

# Add edges
graph.add_edge(START, "agent")
graph.add_conditional_edges("agent", tools_condition)  # → tools or END
graph.add_edge("tools", "agent")  # tools → back to agent

# Compile with memory (enables conversation history)
memory = MemorySaver()
app = graph.compile(checkpointer=memory)
```

### Step 3: Run It

```python
# Run interactively
if __name__ == "__main__":
    config = {"configurable": {"thread_id": "research-session-1"}}
    
    print("🤖 Research Assistant (LangGraph) ready! Type 'quit' to exit.\n")
    
    while True:
        user_input = input("You: ").strip()
        if user_input.lower() in ("quit", "exit", "q"):
            break
        
        result = app.invoke(
            {"messages": [("user", user_input)]},
            config=config,
        )
        
        last_message = result["messages"][-1]
        print(f"\n🤖 Assistant: {last_message.content}\n")
```

## Comparing the Two Implementations

<div class="diagram">
<div class="compare">
  <div class="compare-side left">
    <div class="compare-title">OpenAI Agents SDK</div>
    <ul>
      <li>~40 lines of code</li>
      <li>Handles the loop for you</li>
      <li>Simple mental model</li>
      <li>Quick to prototype</li>
      <li>Limited customization</li>
    </ul>
  </div>
  <div class="compare-side right">
    <div class="compare-title">LangGraph</div>
    <ul>
      <li>~60 lines of code</li>
      <li>You control the graph</li>
      <li>Visualizable state machine</li>
      <li>Built-in checkpointing</li>
      <li>Maximum flexibility</li>
    </ul>
  </div>
</div>
</div>

## Adding Guardrails

Both implementations should have safety guardrails:

```python
# Guardrail: limit the number of tool calls
MAX_TOOL_CALLS = 15

# OpenAI SDK: use max_turns
result = await Runner.run(research_agent, user_input, max_turns=MAX_TOOL_CALLS)

# LangGraph: use recursion_limit
result = app.invoke(
    {"messages": [("user", user_input)]},
    config={"configurable": {"thread_id": "1"}, "recursion_limit": MAX_TOOL_CALLS},
)
```

## Running Locally with Ollama

Want to run agents **without paying for API calls**? Use [Ollama](https://ollama.com) with a local model:

```bash
# Install Ollama (macOS)
brew install ollama

# Pull a capable model
ollama pull llama3.3

# Start the server
ollama serve
```

```python
# Use with LangGraph
from langchain_ollama import ChatOllama

llm = ChatOllama(model="llama3.3", temperature=0).bind_tools(tools)
# Everything else stays the same!
```

> **Note**: Local models are less reliable at function calling than GPT-4o or Claude. Expect more retries and formatting errors. Best for learning, not production.

## Exercises

1. **Add a new tool**: Create a `calculate` tool that evaluates math expressions
2. **Add memory**: Store research findings in a vector database so the agent remembers past sessions
3. **Add streaming**: Show the agent's thinking in real-time as it works
4. **Multi-agent**: Create a "reviewer" agent that fact-checks the researcher's output
5. **Deploy**: Wrap the agent in a FastAPI server with a simple chat UI

## What's Next

Now that you've built an agent hands-on, let's explore the **design patterns** that production agent systems use — patterns that will make your agents more robust, efficient, and maintainable.

**Next: [Chapter 11 — Agentic Design Patterns →](./11_agentic_design_patterns.md)**

---

[← Previous: Chapter 9 — Agentic Frameworks](./09_agentic_frameworks.md) · [Next: Chapter 11 — Agentic Design Patterns →](./11_agentic_design_patterns.md)

*Last updated: April 2026*
