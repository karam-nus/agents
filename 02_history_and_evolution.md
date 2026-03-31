[← Back to Table of Contents](./README.md)

# Chapter 2 — History & Evolution

> *"The best way to predict the future is to invent it."*
> — Alan Kay

## How We Got Here

The idea of machines that reason and act isn't new. What's new is that LLMs finally made it *work*. Here's the journey from early AI to today's autonomous agents.

<div class="diagram">
<div class="diagram-title">Timeline of Agent Evolution</div>
<div class="timeline">
  <div class="timeline-item">
    <div class="timeline-year">1966</div>
    <div class="timeline-title">ELIZA</div>
    <div class="timeline-desc">Joseph Weizenbaum's pattern-matching chatbot. No reasoning — just regex. But it showed people <em>want</em> to interact with machines conversationally.</div>
  </div>
  <div class="timeline-item">
    <div class="timeline-year">1980s</div>
    <div class="timeline-title">Expert Systems</div>
    <div class="timeline-desc">Hand-coded if/then rules (MYCIN, DENDRAL). Agents before LLMs — brittle, expensive to maintain, couldn't generalize.</div>
  </div>
  <div class="timeline-item">
    <div class="timeline-year">1996</div>
    <div class="timeline-title">BDI Architecture</div>
    <div class="timeline-desc">Belief-Desire-Intention model formalized how agents reason. Influenced robotics and game AI for decades.</div>
  </div>
  <div class="timeline-item">
    <div class="timeline-year">2017</div>
    <div class="timeline-title">Transformer Architecture</div>
    <div class="timeline-desc">"Attention Is All You Need" — the foundation for every modern LLM and, by extension, every modern agent.</div>
  </div>
  <div class="timeline-item">
    <div class="timeline-year">2022 — Oct</div>
    <div class="timeline-title">ReAct Paper</div>
    <div class="timeline-desc">Yao et al. showed LLMs can interleave <strong>reasoning</strong> (thinking) with <strong>acting</strong> (tool use). The birth of modern agents.</div>
  </div>
  <div class="timeline-item">
    <div class="timeline-year">2023 — Feb</div>
    <div class="timeline-title">Toolformer</div>
    <div class="timeline-desc">Meta showed LLMs can learn to use tools autonomously — deciding when and how to call APIs from training data.</div>
  </div>
  <div class="timeline-item">
    <div class="timeline-year">2023 — Mar</div>
    <div class="timeline-title">GPT-4 + Function Calling</div>
    <div class="timeline-desc">OpenAI released GPT-4 with native function calling. This made building agents accessible to any developer.</div>
  </div>
  <div class="timeline-item">
    <div class="timeline-year">2023 — Apr</div>
    <div class="timeline-title">AutoGPT / BabyAGI</div>
    <div class="timeline-desc">Open-source fully autonomous agents. They showed the dream (and the limitations) of self-directed AI.</div>
  </div>
  <div class="timeline-item">
    <div class="timeline-year">2023 — May</div>
    <div class="timeline-title">Voyager</div>
    <div class="timeline-desc">NVIDIA's agent that <em>wrote its own code</em> to explore Minecraft — learning new skills by writing and saving programs.</div>
  </div>
  <div class="timeline-item">
    <div class="timeline-year">2023 — Oct</div>
    <div class="timeline-title">Generative Agents (Stanford)</div>
    <div class="timeline-desc">25 AI agents living in a simulated town, forming relationships, planning parties. Memory + reflection = emergent behavior.</div>
  </div>
  <div class="timeline-item">
    <div class="timeline-year">2024</div>
    <div class="timeline-title">Framework Explosion</div>
    <div class="timeline-desc">LangGraph, CrewAI, AutoGen, OpenAI Agents SDK, Anthropic's tool use — agents went from research to production.</div>
  </div>
  <div class="timeline-item">
    <div class="timeline-year">2025</div>
    <div class="timeline-title">MCP & Computer Use</div>
    <div class="timeline-desc">Model Context Protocol standardized tool integration. Anthropic's computer use let agents control desktops. Agents became mainstream.</div>
  </div>
  <div class="timeline-item">
    <div class="timeline-year">2026</div>
    <div class="timeline-title">Agentic Enterprise</div>
    <div class="timeline-desc">Agents in production at scale — customer support, coding, research, operations. The "agent developer" role emerges.</div>
  </div>
</div>
</div>

## The Key Papers

If you want to go deep, these are the foundational papers that shaped modern agents:

| Paper | Year | Key Contribution |
|-------|------|-----------------|
| [ReAct: Synergizing Reasoning and Acting](https://arxiv.org/abs/2210.15153) | 2022 | Interleaving thought + action traces — the core agent pattern |
| [Toolformer](https://arxiv.org/abs/2302.04761) | 2023 | LLMs learning to use tools from self-supervised data |
| [Reflexion](https://arxiv.org/abs/2303.11366) | 2023 | Agents that learn from their own mistakes through self-reflection |
| [Voyager](https://arxiv.org/abs/2305.16291) | 2023 | Lifelong learning agent that writes and stores its own code |
| [Generative Agents](https://arxiv.org/abs/2304.03442) | 2023 | Memory + reflection → emergent social behavior |
| [Chain-of-Thought Prompting](https://arxiv.org/abs/2201.11903) | 2022 | Step-by-step reasoning dramatically improves LLM performance |
| [Tree of Thoughts](https://arxiv.org/abs/2305.10601) | 2023 | Exploring multiple reasoning paths, not just one chain |

## The Three Breakthroughs That Enabled Modern Agents

<div class="diagram">
<div class="diagram-title">Why Agents Work Now</div>
<div class="diagram-grid cols-3">
  <div class="diagram-card accent">
    <div class="card-icon">🧠</div>
    <div class="card-title">Capable LLMs</div>
    <div class="card-desc">GPT-4, Claude, Gemini — models smart enough to reason over multi-step problems and follow complex instructions</div>
  </div>
  <div class="diagram-card green">
    <div class="card-icon">🔧</div>
    <div class="card-title">Function Calling</div>
    <div class="card-desc">Native support for structured tool invocation — the LLM can reliably output JSON to call functions</div>
  </div>
  <div class="diagram-card blue">
    <div class="card-icon">💾</div>
    <div class="card-title">Long Context + RAG</div>
    <div class="card-desc">128K–1M token windows + vector search = agents can reason over large knowledge bases</div>
  </div>
</div>
</div>

## Lessons from the Early Autonomous Agent Era

AutoGPT (2023) was one of the fastest-growing GitHub repos in history. But it also taught us hard lessons:

1. **Unbounded autonomy fails** — agents would loop forever, spending money on API calls without making progress
2. **Planning > execution** — agents good at individual steps but bad at long-horizon planning
3. **Hallucination is deadly for agents** — a chatbot hallucination is annoying; an agent hallucination triggers real-world actions
4. **Human-in-the-loop is essential** — the best agent systems keep humans at critical checkpoints

These lessons shaped the move from "fully autonomous" to "agentic workflows" — controlled autonomy with human oversight.

## What Changed: The Modern Agent Paradigm

<div class="diagram">
<div class="diagram-title">2023 vs. 2026: How Agent Design Evolved</div>
<div class="compare">
  <div class="compare-side left">
    <div class="compare-title">Early 2023: The Dream</div>
    <ul>
      <li>Fully autonomous — "set and forget"</li>
      <li>One giant agent does everything</li>
      <li>Free-form planning with no guardrails</li>
      <li>LLM decides everything</li>
      <li>Cool demos, unreliable in practice</li>
    </ul>
  </div>
  <div class="compare-side right">
    <div class="compare-title">2026: The Reality</div>
    <ul>
      <li>Controlled autonomy with human checkpoints</li>
      <li>Specialized agents collaborating</li>
      <li>Structured workflows + constrained actions</li>
      <li>LLM handles reasoning; code handles control flow</li>
      <li>Production-grade with observability</li>
    </ul>
  </div>
</div>
</div>

## What's Next

With this historical context, let's zoom into the architecture of a modern agent — what components it's made of and how they fit together.

**Next: [Chapter 3 — Anatomy of an Agent →](./03_anatomy_of_an_agent.md)**

---

[← Previous: Chapter 1 — What Are Agents?](./01_what_are_agents.md) · [Next: Chapter 3 — Anatomy of an Agent →](./03_anatomy_of_an_agent.md)

*Last updated: April 2026*
