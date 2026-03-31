[← Back to Table of Contents](./README.md)

# Chapter 6 — Memory Systems

> *"Without memory, there is no learning. Without learning, there is no intelligence."*

## Why Agents Need Memory

An LLM by itself is **stateless** — each API call is independent, with no knowledge of previous calls. Agents need memory systems to:

- **Maintain conversation context** across turns
- **Learn from past mistakes** and successes
- **Access large knowledge bases** that don't fit in the context window
- **Remember user preferences** across sessions
- **Build up working state** during complex multi-step tasks

## The Memory Hierarchy

<div class="diagram">
<div class="diagram-title">Agent Memory Architecture</div>
<div class="layer-stack">
  <div class="layer purple">🧠 Working Memory (Context Window) <small>32K–1M tokens — the LLM's "RAM". Everything it can see right now.</small></div>
  <div class="layer blue">💬 Short-Term Memory <small>Current conversation history. Persists within a session.</small></div>
  <div class="layer accent">📚 Long-Term Memory (Vector Store) <small>Persistent facts, past conversations, learned knowledge. Survives across sessions.</small></div>
  <div class="layer orange">🌍 External Memory (RAG) <small>Documents, databases, APIs — retrieved on demand at query time.</small></div>
</div>
</div>

## Working Memory: The Context Window

The context window is the most immediate form of memory. Everything the LLM can "see" right now lives here:

- System prompt
- Conversation history
- Tool definitions
- Tool call results
- Retrieved documents

### The Context Window Problem

As an agent runs, the context window fills up. A typical agent interaction:

| Content | Token Estimate |
|---------|---------------|
| System prompt | ~500 tokens |
| Tool definitions (10 tools) | ~2,000 tokens |
| User message | ~100 tokens |
| Per loop iteration (LLM response + tool result) | ~1,000–5,000 tokens |
| After 10 iterations | ~15,000–55,000 tokens |

With a 128K context window, you have room. But at **$2.50/M input tokens**, those tokens get expensive fast.

### Context Management Strategies

```python
class ContextManager:
    def __init__(self, max_tokens: int = 100_000):
        self.max_tokens = max_tokens
        self.messages = []
    
    def add(self, message: dict):
        self.messages.append(message)
        self._trim_if_needed()
    
    def _trim_if_needed(self):
        """Sliding window: keep system prompt + recent messages."""
        while self._estimate_tokens() > self.max_tokens:
            # Never remove system prompt (index 0) or last 5 messages
            if len(self.messages) > 6:
                self.messages.pop(1)  # Remove oldest non-system message
    
    def _summarize_old_context(self):
        """Alternative: summarize old messages instead of dropping them."""
        old = self.messages[1:-5]  # Everything except system + recent
        summary = llm.summarize(old)
        self.messages = [
            self.messages[0],  # system prompt
            {"role": "system", "content": f"Previous context summary: {summary}"},
            *self.messages[-5:],  # recent messages
        ]
```

## Short-Term Memory: Conversation History

Short-term memory is the **conversation buffer** — the growing list of messages exchanged between the user, the agent, and tools within a single session.

<div class="diagram">
<div class="diagram-title">Short-Term Memory Flow</div>
<div class="flow">
  <div class="flow-node accent wide">👤 User: "Find quarterly revenue for Apple"</div>
  <div class="flow-arrow"></div>
  <div class="flow-node purple wide">🤖 Agent: Calls web_search("Apple Q4 2025 revenue")</div>
  <div class="flow-arrow"></div>
  <div class="flow-node green wide">🔧 Tool result: "$94.9B in Q4 2025"</div>
  <div class="flow-arrow"></div>
  <div class="flow-node blue wide">🤖 Agent: "Apple reported $94.9B in Q4 2025"</div>
  <div class="flow-arrow"></div>
  <div class="flow-node orange wide">👤 User: "How does that compare to last year?"</div>
  <div class="flow-arrow"></div>
  <div class="flow-node wide">🤖 Agent recalls the context and searches for Q4 2024 data</div>
</div>
</div>

The agent understands "that" and "last year" because the previous turns are in short-term memory.

## Long-Term Memory: Vector Stores

For knowledge that persists **across sessions**, agents use vector databases. The pattern:

1. **Store**: Convert text into embeddings and save them
2. **Retrieve**: When needed, find relevant memories by semantic similarity
3. **Inject**: Add retrieved memories into the context window

```python
from openai import OpenAI
import chromadb

client = OpenAI()
db = chromadb.Client()
collection = db.create_collection("agent_memory")

def store_memory(text: str, metadata: dict = None):
    """Store a fact in long-term memory."""
    embedding = client.embeddings.create(
        model="text-embedding-3-small",
        input=text
    ).data[0].embedding
    
    collection.add(
        ids=[str(hash(text))],
        embeddings=[embedding],
        documents=[text],
        metadatas=[metadata or {}],
    )

def recall_memories(query: str, k: int = 5) -> list[str]:
    """Retrieve relevant memories."""
    embedding = client.embeddings.create(
        model="text-embedding-3-small",
        input=query
    ).data[0].embedding
    
    results = collection.query(
        query_embeddings=[embedding],
        n_results=k,
    )
    return results["documents"][0]
```

### What to Store in Long-Term Memory

| Memory Type | Example | Why |
|-------------|---------|-----|
| **User preferences** | "User prefers Python over JavaScript" | Personalization |
| **Past task outcomes** | "Last time I searched for X, the best source was Y" | Learning |
| **Extracted facts** | "Company X was founded in 2019" | Knowledge accumulation |
| **Conversation summaries** | "In our last session, we planned the API architecture" | Cross-session continuity |
| **Error lessons** | "Using endpoint X requires auth header Y" | Self-improvement |

## Episodic vs. Semantic Memory

Inspired by human cognition, agent memory can be split into:

<div class="diagram">
<div class="compare">
  <div class="compare-side left">
    <div class="compare-title">📖 Episodic Memory</div>
    <ul>
      <li>"What happened" — specific events and experiences</li>
      <li>"Last time you asked about stocks, you wanted tech sector"</li>
      <li>Stores conversation transcripts, task logs</li>
      <li>Good for: personalization, learning from mistakes</li>
    </ul>
  </div>
  <div class="compare-side right">
    <div class="compare-title">🧠 Semantic Memory</div>
    <ul>
      <li>"What I know" — general facts and knowledge</li>
      <li>"Apple is a tech company headquartered in Cupertino"</li>
      <li>Stores extracted facts, documentation chunks</li>
      <li>Good for: answering questions, reasoning about domains</li>
    </ul>
  </div>
</div>
</div>

## RAG: Memory From External Sources

**Retrieval-Augmented Generation** lets agents access vast knowledge bases without storing everything in the context window:

<div class="diagram">
<div class="diagram-title">RAG Pipeline</div>
<div class="flow-h">
  <div class="flow-node accent">📄 Documents</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node purple">✂️ Chunk</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node blue">🔢 Embed</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node green">💾 Vector DB</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node orange">🔍 Retrieve</div>
  <div class="flow-arrow accent"></div>
  <div class="flow-node pink">🧠 Generate</div>
</div>
</div>

```python
# RAG as an agent tool
def search_knowledge_base(query: str) -> str:
    """Search the company knowledge base for relevant information."""
    # 1. Embed the query
    query_embedding = embed(query)
    
    # 2. Search the vector database
    results = vector_db.search(query_embedding, top_k=5)
    
    # 3. Format results for the LLM
    context = "\n\n".join([
        f"[Source: {r.metadata['source']}]\n{r.text}" 
        for r in results
    ])
    
    return context
```

## Memory in Practice: The Mem0 Pattern

[Mem0](https://github.com/mem0ai/mem0) popularized a pattern for agent memory that auto-extracts and stores important facts:

```python
from mem0 import Memory

memory = Memory()

# After each conversation, auto-extract and store memories
memory.add("I prefer dark mode and vim keybindings", user_id="user_123")
memory.add("My project uses FastAPI with PostgreSQL", user_id="user_123")

# Before generating a response, retrieve relevant memories
relevant = memory.search("What framework should I use?", user_id="user_123")
# Returns: ["My project uses FastAPI with PostgreSQL"]
```

## Memory Architecture Summary

<div class="diagram">
<div class="diagram-grid cols-2">
  <div class="diagram-card accent">
    <div class="card-icon">⚡</div>
    <div class="card-title">Fast Path</div>
    <div class="card-desc">Context window → immediate access, limited size, expensive per token</div>
  </div>
  <div class="diagram-card blue">
    <div class="card-icon">🔍</div>
    <div class="card-title">Retrieval Path</div>
    <div class="card-desc">Vector DB → unlimited size, semantic search, cheap storage, small latency cost</div>
  </div>
  <div class="diagram-card green">
    <div class="card-icon">📝</div>
    <div class="card-title">Write Path</div>
    <div class="card-desc">Auto-extract facts from conversations → embed → store in vector DB for future use</div>
  </div>
  <div class="diagram-card purple">
    <div class="card-icon">🗑️</div>
    <div class="card-title">Forget Path</div>
    <div class="card-desc">Summarize old context → evict from working memory → keep in long-term if important</div>
  </div>
</div>
</div>

## What's Next

Memory gives agents knowledge. But an agent also needs to *reason* — to plan, decompose problems, and correct its own mistakes. That's planning.

**Next: [Chapter 7 — Planning & Reasoning →](./07_planning_and_reasoning.md)**

---

[← Previous: Chapter 5 — Tools & Function Calling](./05_tools_and_function_calling.md) · [Next: Chapter 7 — Planning & Reasoning →](./07_planning_and_reasoning.md)

*Last updated: April 2026*
