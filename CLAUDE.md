# Mneme

> Named after the Greek muse of memory. An AI assistant that actually knows you — across sessions, across weeks, across every kind of work you do.

---

## What this is

A personal AI assistant with a 3-tier memory architecture that solves the core problem with every AI tool today: **you always start from zero.**

Mneme remembers you across sessions. Your coding preferences, your writing voice, your research conclusions, your past decisions. It retrieves the right memories at the right time so every conversation starts where you actually are — not at ground zero.

**The problem it solves:**
- Having to re-explain yourself every session
- Getting repetitive, beginner-level responses
- Losing context mid-conversation on long tasks
- AI tools that don't adapt to how you work

**Based on:** arXiv:2604.01670 — "Hierarchical Memory Orchestration for Personalized Persistent Agents" (Liu, Sun et al.)

---

## Memory architecture

### Tier 1 — working memory
- **What:** Last 15 messages of the current session
- **How:** Python list held in LangGraph state
- **Why:** Immediate conversational coherence
- **Lifetime:** Current session only, summarized on session end

### Tier 2 — episodic memory
- **What:** Compressed summaries of past conversations
- **How:** Stored as vector embeddings in Qdrant, retrieved by semantic similarity
- **Why:** "Last week you told me you hate meetings before 10am"
- **Lifetime:** Persistent, subject to forgetting curve decay

### Tier 3 — semantic memory
- **What:** Structured facts about the user as a knowledge graph
- **How:** Neo4j nodes and edges, extracted by LLM from conversations
- **Why:** "What stack does the user work in?" is a graph query, not a vector search
- **Lifetime:** Persistent, reinforced on repeated mention

---

## Mode-specific memory

The agent operates in 4 modes. What gets stored and retrieved depends on which mode is active.

| Mode | Remembers | Entity types |
|------|-----------|--------------|
| Coding | Stack, style, frameworks, past bugs | Language, Framework, Project, Bug, Pattern |
| Writing | Voice, audience, past drafts, style rules | Document, Audience, StyleRule, Topic |
| Research | Papers read, conclusions, open questions | Paper, Concept, Conclusion, Question |
| Thinking | Past decisions, mental models, reasoning | Decision, Reasoning, MentalModel, Problem |

**Build order:** coding mode first (week 2), then one mode per week after.

---

## Tech stack

| Layer | Technology | Why |
|-------|-----------|-----|
| LLM | `claude-sonnet-4-6` (Anthropic SDK) | Tool use for autonomous memory decisions |
| Entity extraction | `claude-haiku-4-5` | Cheaper model for background tasks |
| Agent framework | LangGraph | State graph pattern — clean, debuggable |
| Episodic memory | Qdrant (Docker) | Vector similarity search, local, free |
| Semantic memory | Neo4j community (Docker) | Knowledge graph, Cypher queries |
| Working memory | Python list in LangGraph state | Simple is correct here — no database needed |
| Embeddings | sentence-transformers `all-MiniLM-L6-v2` | Free, runs on CPU, no API calls |
| Background jobs | APScheduler | Nightly consolidation + session-end jobs |
| UI | Streamlit | Chat panel + memory inspector sidebar |
| Infra | Docker Compose | One-command startup for full stack |

---

## Project structure

```
mneme/
├── CLAUDE.md
├── README.md
├── .env                    # API keys — never commit
├── .gitignore
├── docker-compose.yml      # Qdrant + Neo4j + app
├── requirements.txt
│
├── agent/
│   ├── __init__.py
│   ├── loop.py             # LangGraph state graph — main agent loop
│   ├── tools.py            # Memory read/write tools Claude can call
│   └── modes.py            # Mode detection + mode-specific retrieval logic
│
├── memory/
│   ├── __init__.py
│   ├── working.py          # In-context buffer (Python list)
│   ├── episodic.py         # Qdrant read/write + embedding logic
│   ├── semantic.py         # Neo4j read/write + Cypher queries
│   └── consolidation.py    # APScheduler job — compression + entity extraction
│
├── ui/
│   └── app.py              # Streamlit chat UI + memory inspector
│
└── eval/
    └── recall.py           # Basic recall eval — feed known facts, measure accuracy
```

---

## Build phases

### Week 1 — working memory + basic agent loop
- Get Claude responding in a loop with last N messages in context
- Implement LangGraph state graph
- Learn tool use — Claude deciding when to save vs just respond
- CLI only, no UI
- **Goal:** coherent conversation within a session

### Week 2 — episodic memory (Qdrant) + coding mode
- On session end: summarize → embed → store in Qdrant
- On session start: retrieve top-3 relevant past episodes
- Implement coding mode entity schema
- **Goal:** close app, reopen, ask "what did we talk about?" — it knows

### Week 3 — semantic memory (Neo4j) + consolidation job
- Entity extraction prompt → structured facts → Neo4j
- APScheduler consolidation job (session end + nightly)
- Forgetting curve — decay scores on unreinforced memories
- **Goal:** "what do you know about me?" returns a populated graph

### Week 4 — UI + polish + eval
- Streamlit chat panel + memory inspector sidebar (pyvis graph visualization)
- Docker Compose full stack
- Basic recall eval (100 known facts → measure accuracy)
- Deploy to Fly.io
- **Goal:** one-command setup, live demo, shareable link

---

## Papers to read

| Paper | arXiv | What to read | Why |
|-------|-------|-------------|-----|
| CoALA — Cognitive Architectures for Language Agents | 2309.02427 | Abstract + section 3 | Vocabulary and theory behind the architecture |
| Hierarchical Memory Orchestration (this project) | 2604.01670 | Sections 1–3 | The architecture you're implementing |
| MemGPT: Towards LLMs as Operating Systems | 2310.08560 | Abstract + section 2 | Best mental model for context window as RAM |
| LLM-as-a-Judge — Zheng et al. | 2306.05685 | Sections 1–2 | For the eval layer in week 4 |

---

## Learning resources

| Resource | URL | When |
|----------|-----|------|
| Anthropic docs | docs.anthropic.com | Day 1 — tool use + structured outputs |
| Anthropic cookbook | github.com/anthropics/anthropic-cookbook | Day 2 — working agent code examples |
| LangGraph docs | langchain-ai.github.io/langgraph | Day 6 — concepts + get started |
| Qdrant docs | qdrant.tech/documentation | Day 4 — quick start + concepts |
| Neo4j sandbox | sandbox.neo4j.com | Day 5 — blank sandbox tutorial |
| LangGraph crash course — Greg Kamradt | YouTube | Day 6 — after reading docs |
| Vector databases explained — Fireship | YouTube | Day 3 — before Qdrant quickstart |

---

## Setup

```bash
# 1. clone and create venv
git clone https://github.com/you/mneme
cd mneme
python -m venv .venv && source .venv/bin/activate

# 2. install dependencies
pip install anthropic langgraph qdrant-client sentence-transformers \
            neo4j streamlit apscheduler python-dotenv

# 3. add API key
echo "ANTHROPIC_API_KEY=your_key_here" > .env

# 4. start databases
docker-compose up -d

# 5. run
streamlit run ui/app.py
```

---

## Cost estimate

- Anthropic API budget for full build: **~$15–25**
- Use `claude-haiku-4-5` for entity extraction and summarization
- Use `claude-sonnet-4-6` for user-facing responses only
- Use local sentence-transformers for embeddings (free, no API calls)

---

## Eval (week 4)

Feed 100 known facts about a test user. Ask questions that require retrieving them. Measure:
- **Recall accuracy** — does it return the right fact?
- **Retrieval latency** — how long does it take?
- **Tier attribution** — did it come from episodic or semantic?

Target: >85% recall accuracy, <200ms p95 retrieval latency.

---

## Resume bullets (fill in real numbers after building)

**Lead:** "Built production-grade hierarchical memory agent (Mneme) with 3-tier architecture — in-context buffer, Qdrant episodic store, Neo4j knowledge graph — persisting accurate user context across sessions with **X% recall accuracy** and **Yms** p95 retrieval latency."

**Infra:** "Designed nightly memory consolidation pipeline using APScheduler — compresses episodic summaries, extracts structured entities into Neo4j, applies forgetting curve decay; reduced context window usage by **X%** vs naive full-history injection."

**Agent:** "Implemented LangGraph state graph with Claude tool use for autonomous memory read/write decisions across 4 work modes (coding, writing, research, thinking)."

**Product:** "Shipped memory inspector UI visualizing live knowledge graph updates in real time; deployed full stack via Docker Compose to Fly.io with one-command setup."
