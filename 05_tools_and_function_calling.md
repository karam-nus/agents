[← Back to Table of Contents](./README.md)

# Chapter 5 — Tools & Function Calling

> *"Tools are what separate an agent from a chatbot. Without tools, an LLM can only talk. With tools, it can do."*

## What Is a Tool?

A **tool** is any function that an agent can invoke to interact with the outside world. It could be a web search, a database query, a code interpreter, or an API call. The LLM doesn't execute the tool itself — it **requests** a tool call, and your code executes it.

<div class="diagram">
<div class="diagram-title">How Function Calling Works</div>
<div class="flow">
  <div class="flow-node accent wide">🧠 LLM receives prompt + tool definitions</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node purple wide">💭 LLM decides: "I need to search the web"</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node blue wide">📤 LLM outputs structured JSON tool call</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node green wide">⚙️ Your code executes the function</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node orange wide">📥 Result sent back to LLM as observation</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node wide">🔄 LLM continues reasoning with the result</div>
</div>
</div>

## Defining Tools: The Schema

Tools are defined using JSON Schema. This tells the LLM what tools are available, what they do, and what parameters they accept.

### OpenAI Format

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "Get the current weather for a given city. Use this when the user asks about weather conditions.",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {
                        "type": "string",
                        "description": "The city name, e.g., 'San Francisco'"
                    },
                    "units": {
                        "type": "string",
                        "enum": ["celsius", "fahrenheit"],
                        "description": "Temperature units"
                    }
                },
                "required": ["city"]
            }
        }
    }
]
```

### Anthropic Format

```python
tools = [
    {
        "name": "get_weather",
        "description": "Get the current weather for a given city.",
        "input_schema": {
            "type": "object",
            "properties": {
                "city": {"type": "string", "description": "The city name"},
                "units": {"type": "string", "enum": ["celsius", "fahrenheit"]}
            },
            "required": ["city"]
        }
    }
]
```

> **Pro tip**: The `description` field is **critical**. The LLM uses it to decide *when* to call a tool. A vague description = unreliable tool use. Be specific about when the tool should be used and what it returns.

## Building a Tool Registry

In practice, you map tool names to actual Python functions:

```python
import json
import requests

# Define your tool functions
def get_weather(city: str, units: str = "celsius") -> str:
    """Fetch weather from a weather API."""
    resp = requests.get(
        "https://api.weatherapi.com/v1/current.json",
        params={"key": WEATHER_API_KEY, "q": city}
    )
    data = resp.json()
    temp = data["current"]["temp_c"] if units == "celsius" else data["current"]["temp_f"]
    return f"{city}: {temp}°{'C' if units == 'celsius' else 'F'}, {data['current']['condition']['text']}"

def web_search(query: str) -> str:
    """Search the web using Tavily."""
    from tavily import TavilyClient
    client = TavilyClient(api_key=TAVILY_API_KEY)
    results = client.search(query, max_results=3)
    return "\n".join(r["content"] for r in results["results"])

# Registry: maps function names to callables
TOOL_REGISTRY = {
    "get_weather": get_weather,
    "web_search": web_search,
}

def execute_tool_call(tool_call) -> str:
    """Execute a tool call from the LLM."""
    fn_name = tool_call.function.name
    fn_args = json.loads(tool_call.function.arguments)
    
    if fn_name not in TOOL_REGISTRY:
        return f"Error: Unknown tool '{fn_name}'"
    
    try:
        return TOOL_REGISTRY[fn_name](**fn_args)
    except Exception as e:
        return f"Error executing {fn_name}: {str(e)}"
```

## Parallel Tool Calls

Modern LLMs can request **multiple tool calls** in a single response. For example, "What's the weather in Tokyo and Paris?" triggers two parallel `get_weather` calls:

```python
# The LLM returns multiple tool_calls
message.tool_calls = [
    ToolCall(id="call_1", function=Function(name="get_weather", arguments='{"city":"Tokyo"}')),
    ToolCall(id="call_2", function=Function(name="get_weather", arguments='{"city":"Paris"}')),
]

# Execute all in parallel
import asyncio

async def execute_parallel(tool_calls):
    tasks = [execute_tool_async(tc) for tc in tool_calls]
    return await asyncio.gather(*tasks)
```

## The Model Context Protocol (MCP)

**MCP** is an open standard (created by Anthropic, adopted widely) that standardizes how agents connect to tools. Think of it as "USB for AI tools" — a universal plug that lets any agent use any tool server.

<div class="diagram">
<div class="diagram-title">MCP Architecture</div>
<div class="flow-h">
  <div class="flow-node accent">🤖 Agent (MCP Client)</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node purple">🔌 MCP Protocol</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node green">🔧 Tool Server (MCP Server)</div>
</div>
</div>

### Why MCP Matters

| Without MCP | With MCP |
|------------|----------|
| Every agent integrates each tool differently | Standard protocol for all tools |
| Tool code lives inside the agent | Tools are separate servers, reusable across agents |
| Adding a new tool = changing agent code | Adding a new tool = connecting to a new server |
| No discovery mechanism | Agents can discover available tools at runtime |

### MCP in Practice

```python
# An MCP server exposes tools as a standardized service
from mcp.server import Server, Tool

server = Server("weather-tools")

@server.tool("get_weather")
async def get_weather(city: str, units: str = "celsius") -> str:
    """Get current weather for a city."""
    # ... implementation ...
    return f"{city}: 72°F, Sunny"

@server.tool("get_forecast")
async def get_forecast(city: str, days: int = 5) -> str:
    """Get weather forecast for a city."""
    # ... implementation ...
    return f"{days}-day forecast for {city}: ..."

# Run the server
server.run(transport="stdio")
```

```python
# An MCP client (your agent) connects to the server
from mcp.client import Client

async with Client("weather-tools", transport="stdio") as client:
    # Discover available tools
    tools = await client.list_tools()
    
    # Call a tool
    result = await client.call_tool("get_weather", {"city": "Tokyo"})
```

## Tool Safety and Sandboxing

> **⚠️ Critical**: Tools execute real actions. A code execution tool can run `rm -rf /`. A file write tool can overwrite production configs. Safety is not optional.

### Safety Principles

<div class="diagram">
<div class="diagram-grid cols-2">
  <div class="diagram-card accent">
    <div class="card-icon">🔒</div>
    <div class="card-title">Least Privilege</div>
    <div class="card-desc">Tools should have the minimum permissions needed. A search tool doesn't need file system access.</div>
  </div>
  <div class="diagram-card green">
    <div class="card-icon">📦</div>
    <div class="card-title">Sandboxing</div>
    <div class="card-desc">Code execution should run in a container or VM. Never on the host system directly.</div>
  </div>
  <div class="diagram-card blue">
    <div class="card-icon">✅</div>
    <div class="card-title">Confirmation</div>
    <div class="card-desc">Destructive actions (delete, write, send) require human approval before execution.</div>
  </div>
  <div class="diagram-card orange">
    <div class="card-icon">📋</div>
    <div class="card-title">Audit Logging</div>
    <div class="card-desc">Every tool call should be logged — who called what, with which arguments, and what happened.</div>
  </div>
</div>
</div>

```python
# Example: confirmation gate for dangerous tools
DANGEROUS_TOOLS = {"delete_file", "execute_shell", "send_email"}

def execute_with_safety(tool_call):
    fn_name = tool_call.function.name
    fn_args = json.loads(tool_call.function.arguments)
    
    if fn_name in DANGEROUS_TOOLS:
        print(f"⚠️  Agent wants to call: {fn_name}({fn_args})")
        if input("Allow? (y/n): ").lower() != "y":
            return "Tool call denied by user."
    
    return TOOL_REGISTRY[fn_name](**fn_args)
```

## Building Great Tools: Best Practices

1. **Clear descriptions**: Tell the LLM exactly when to use the tool and what it returns
2. **Structured output**: Return structured data, not raw HTML or binary blobs
3. **Error messages**: Return helpful errors the LLM can understand and recover from
4. **Idempotent when possible**: Calling a tool twice with the same args should be safe
5. **Bounded output**: Cap response size — don't return 10MB of data into the context window

## What's Next

Tools give agents hands to interact with the world. But without memory, every interaction starts from scratch. Let's explore how agents remember.

**Next: [Chapter 6 — Memory Systems →](./06_memory_systems.md)**

---

[← Previous: Chapter 4 — The Agent Loop](./04_the_agent_loop.md) · [Next: Chapter 6 — Memory Systems →](./06_memory_systems.md)

*Last updated: April 2026*
