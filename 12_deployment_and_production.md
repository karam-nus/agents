[← Back to Table of Contents](./README.md)

# Chapter 12 — Deployment & Production

> *"A demo is not a product. The gap between 'it works on my laptop' and 'it works for 10,000 users' is where most agent projects die."*

## The Production Gap

Your agent works great in development. Now ship it. Here's what that actually requires:

<div class="diagram">
<div class="diagram-title">Development vs. Production</div>
<div class="compare">
  <div class="compare-side left">
    <div class="compare-title">Development</div>
    <ul>
      <li>Single user (you)</li>
      <li>Best-case inputs</li>
      <li>Cost doesn't matter</li>
      <li>Errors → print() and fix</li>
      <li>No latency requirements</li>
    </ul>
  </div>
  <div class="compare-side right">
    <div class="compare-title">Production</div>
    <ul>
      <li>Thousands of concurrent users</li>
      <li>Adversarial and edge-case inputs</li>
      <li>Every token costs money</li>
      <li>Errors → graceful degradation + alerts</li>
      <li>Sub-second response time expected</li>
    </ul>
  </div>
</div>
</div>

## The Production Stack

<div class="diagram">
<div class="diagram-title">Agent Production Architecture</div>
<div class="layer-stack">
  <div class="layer purple">👤 User Interface <small>Chat UI, API client, Slack bot, CLI</small></div>
  <div class="layer blue">🌐 API Layer <small>FastAPI / Express.js — authentication, rate limiting, request routing</small></div>
  <div class="layer accent">🤖 Agent Runtime <small>LangGraph / OpenAI SDK — the agent loop, tool execution, state management</small></div>
  <div class="layer green">🔧 Tool Layer <small>MCP servers, API integrations, databases, search</small></div>
  <div class="layer orange">👁️ Observability <small>LangSmith, Arize Phoenix, Langfuse — tracing, logging, evaluation</small></div>
  <div class="layer">💾 Persistence <small>PostgreSQL, Redis, vector DB — conversation history, agent state, memory</small></div>
</div>
</div>

## Serving Agents via API

Wrap your agent in a FastAPI server:

```python
from fastapi import FastAPI, HTTPException
from fastapi.responses import StreamingResponse
from pydantic import BaseModel
import asyncio

app = FastAPI()

class ChatRequest(BaseModel):
    message: str
    thread_id: str = "default"

class ChatResponse(BaseModel):
    response: str
    tool_calls: list[dict] = []

@app.post("/chat", response_model=ChatResponse)
async def chat(req: ChatRequest):
    try:
        result = await agent.ainvoke(
            {"messages": [("user", req.message)]},
            config={"configurable": {"thread_id": req.thread_id}},
        )
        return ChatResponse(
            response=result["messages"][-1].content,
            tool_calls=[],
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/chat/stream")
async def chat_stream(req: ChatRequest):
    """Stream agent responses for real-time UX."""
    async def generate():
        async for event in agent.astream_events(
            {"messages": [("user", req.message)]},
            config={"configurable": {"thread_id": req.thread_id}},
            version="v2",
        ):
            if event["event"] == "on_chat_model_stream":
                chunk = event["data"]["chunk"]
                if chunk.content:
                    yield f"data: {chunk.content}\n\n"
        yield "data: [DONE]\n\n"
    
    return StreamingResponse(generate(), media_type="text/event-stream")
```

## Observability: Seeing What Your Agent Does

In production, you **must** be able to trace every decision your agent makes. Key tools:

| Tool | What It Does | Best For |
|------|-------------|----------|
| [LangSmith](https://smith.langchain.com) | Full trace visualization, evaluation, datasets | LangGraph users |
| [Langfuse](https://langfuse.com) | Open-source tracing, prompt management | Self-hosted / privacy |
| [Arize Phoenix](https://phoenix.arize.com) | Traces + evaluation + embeddings analysis | ML teams |
| [Braintrust](https://braintrust.dev) | Evaluation, logging, prompt playground | Evaluation-focused |

### Adding Tracing (LangSmith)

```python
# Just set environment variables — LangGraph auto-traces
export LANGCHAIN_TRACING_V2=true
export LANGCHAIN_API_KEY="ls__..."
export LANGCHAIN_PROJECT="my-research-agent"

# Every agent run now shows in LangSmith:
# - Full message trace (user → LLM → tool → LLM → ...)
# - Token counts and costs per step
# - Latency breakdown
# - Tool inputs/outputs
```

### Custom Logging

```python
import logging
import time

logger = logging.getLogger("agent")

class AgentLogger:
    def on_llm_start(self, prompt, **kwargs):
        logger.info(f"LLM call started | model={kwargs.get('model')}")
        self.start_time = time.time()
    
    def on_llm_end(self, response, **kwargs):
        duration = time.time() - self.start_time
        tokens = response.usage
        logger.info(
            f"LLM call completed | "
            f"duration={duration:.2f}s | "
            f"input_tokens={tokens.prompt_tokens} | "
            f"output_tokens={tokens.completion_tokens}"
        )
    
    def on_tool_start(self, tool_name, tool_input, **kwargs):
        logger.info(f"Tool call: {tool_name}({tool_input})")
    
    def on_tool_error(self, error, **kwargs):
        logger.error(f"Tool error: {error}")
```

## Cost Management

Agents can be **expensive**. A complex task might make 10–20 LLM calls with large context windows.

### Cost Estimation

| Model | Input (per 1M tokens) | Output (per 1M tokens) | Typical agent task |
|-------|----------------------|------------------------|-------------------|
| GPT-4o | $2.50 | $10.00 | ~$0.05–$0.50 |
| GPT-4o-mini | $0.15 | $0.60 | ~$0.005–$0.05 |
| Claude 3.5 Sonnet | $3.00 | $15.00 | ~$0.06–$0.60 |
| Llama 3.3 (local) | Free | Free | Electricity only |

### Cost Control Strategies

```python
# 1. Use cheaper models for simple routing/classification
router_llm = ChatOpenAI(model="gpt-4o-mini")  # Cheap for classification
worker_llm = ChatOpenAI(model="gpt-4o")        # Powerful for complex work

# 2. Set token budgets
MAX_TOKENS_PER_REQUEST = 50_000
MAX_COST_PER_REQUEST = 1.00  # dollars

# 3. Cache tool results
from functools import lru_cache

@lru_cache(maxsize=1000)
def cached_search(query: str) -> str:
    return tavily.search(query)

# 4. Summarize long tool outputs before injecting into context
def truncate_tool_output(output: str, max_chars: int = 2000) -> str:
    if len(output) > max_chars:
        return output[:max_chars] + "\n...[truncated]"
    return output
```

## Error Handling

Agents fail in unique ways. Plan for all of them:

<div class="diagram">
<div class="diagram-grid cols-2">
  <div class="diagram-card accent">
    <div class="card-icon">🔌</div>
    <div class="card-title">API Errors</div>
    <div class="card-desc">Rate limits, timeouts, model downtime. → Retry with exponential backoff</div>
  </div>
  <div class="diagram-card green">
    <div class="card-icon">🔧</div>
    <div class="card-title">Tool Failures</div>
    <div class="card-desc">External API returns error. → Let the LLM know and try an alternative</div>
  </div>
  <div class="diagram-card blue">
    <div class="card-icon">🔄</div>
    <div class="card-title">Infinite Loops</div>
    <div class="card-desc">Agent repeats the same action. → Detect and break with max iterations</div>
  </div>
  <div class="diagram-card orange">
    <div class="card-icon">🤯</div>
    <div class="card-title">Hallucination</div>
    <div class="card-desc">Agent invents facts or fake tool names. → Validate outputs, structured responses</div>
  </div>
</div>
</div>

```python
import tenacity

@tenacity.retry(
    stop=tenacity.stop_after_attempt(3),
    wait=tenacity.wait_exponential(multiplier=1, min=1, max=10),
    retry=tenacity.retry_if_exception_type((RateLimitError, TimeoutError)),
)
async def call_llm_with_retry(messages, tools):
    return await client.chat.completions.create(
        model="gpt-4o",
        messages=messages,
        tools=tools,
        timeout=30,
    )
```

## Deployment Options

| Option | Complexity | Best For |
|--------|-----------|----------|
| **FastAPI + Docker** | Medium | Custom deployments, full control |
| **LangServe** | Low | LangGraph apps, quick deployment |
| **Modal / Fly.io** | Low | Serverless, auto-scaling |
| **AWS Lambda + API Gateway** | Medium | Event-driven, pay-per-use |
| **Kubernetes** | High | Enterprise, multi-region, high availability |

### Docker Deployment

```dockerfile
FROM python:3.12-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```yaml
# docker-compose.yml
services:
  agent:
    build: .
    ports:
      - "8000:8000"
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - TAVILY_API_KEY=${TAVILY_API_KEY}
    restart: unless-stopped
```

## Safety in Production

<div class="diagram">
<div class="diagram-title">Production Safety Checklist</div>
<div class="diagram-grid cols-2">
  <div class="diagram-card accent">
    <div class="card-icon">🔐</div>
    <div class="card-title">Auth & Authorization</div>
    <div class="card-desc">API keys, JWT tokens, role-based access. Never expose raw LLM access.</div>
  </div>
  <div class="diagram-card green">
    <div class="card-icon">🚦</div>
    <div class="card-title">Rate Limiting</div>
    <div class="card-desc">Per-user and global limits. Prevent abuse and cost spikes.</div>
  </div>
  <div class="diagram-card blue">
    <div class="card-icon">🧹</div>
    <div class="card-title">Input Sanitization</div>
    <div class="card-desc">Detect prompt injection. Validate all user inputs. Block PII leaks.</div>
  </div>
  <div class="diagram-card orange">
    <div class="card-icon">📊</div>
    <div class="card-title">Monitoring & Alerts</div>
    <div class="card-desc">Track error rates, latency, cost per user. Alert on anomalies.</div>
  </div>
</div>
</div>

## What's Next

You know how to deploy. Now let's explore the **open source ecosystem** — the projects, tools, and communities that make the agent world tick.

**Next: [Chapter 13 — Open Source Landscape →](./13_open_source_landscape.md)**

---

[← Previous: Chapter 11 — Agentic Design Patterns](./11_agentic_design_patterns.md) · [Next: Chapter 13 — Open Source Landscape →](./13_open_source_landscape.md)

*Last updated: April 2026*
