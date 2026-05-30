# Agentic RAG Chatbot — Hospital System

An AI-powered chatbot that answers natural-language questions about a hospital system by intelligently routing between **semantic search over patient reviews** and **structured graph queries over hospital data**. Built with LangChain, Neo4j, OpenAI, FastAPI, Streamlit, and Docker.

---

## Overview

Hospital administrators constantly need answers from their data — *"What was our total billing to Cigna last year?"*, *"Are patients complaining about the nursing staff?"*, *"How many emergency visits did we have in 2023?"* — but they don't know SQL, can't write graph queries, and don't want to wait for an analyst.

This chatbot bridges that gap. It accepts plain-English questions and uses a LangChain agent to decide, per question, whether the answer requires:

- **Semantic search** over free-text patient reviews (for subjective questions about experiences, feelings, or feedback), or
- **Structured graph queries** over hospital data (for objective questions about counts, aggregations, billing, or relationships)

The agent makes this decision automatically — no manual classification, no separate endpoints. The user just asks; the system figures out how to answer.

---

## Architecture

```
                    ┌─────────────────────┐
                    │   Streamlit Chat    │
                    │      Frontend       │
                    └──────────┬──────────┘
                               │ HTTP
                    ┌──────────▼──────────┐
                    │   FastAPI Backend   │
                    │  (async, retry)     │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │   LangChain Agent   │
                    │  (OpenAI Functions) │
                    └──────────┬──────────┘
                               │
                    ┌──────────┴──────────┐
                    │                     │
         ┌──────────▼─────────┐  ┌────────▼──────────┐
         │  Experiences Tool  │  │    Graph Tool     │
         │  (Vector Chain)    │  │  (Cypher Chain)   │
         └──────────┬─────────┘  └────────┬──────────┘
                    │                     │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │   Neo4j AuraDB      │
                    │  (Graph + Vectors)  │
                    └─────────────────────┘
```

The agent receives a question, reads the description of each available tool, and decides which to invoke. Each tool returns a natural-language answer that the agent relays to the user — along with an explanation of how the answer was derived.

---

## Tech Stack

| Layer | Technology |
|---|---|
| LLM orchestration | LangChain |
| Language models | OpenAI GPT (configurable per chain) |
| Graph database | Neo4j AuraDB |
| Vector search | Neo4j native vector index |
| Embeddings | OpenAI `text-embedding-ada-002` |
| Backend API | FastAPI (async) |
| Frontend | Streamlit |
| Orchestration | Docker Compose |

---

## How It Works

### The Two Chains

**Experiences Chain (Vector RAG)**

Patient reviews are embedded once and stored as a property on `Review` nodes in Neo4j, with a native vector index for similarity search. When a user asks something subjective like *"Are patients satisfied with the nursing staff at Castaneda-Hardy?"*, this chain:

1. Embeds the question with the same OpenAI embedding model.
2. Retrieves the top-12 most semantically similar reviews from Neo4j's vector index.
3. Stuffs the reviews into a prompt with anti-hallucination guardrails.
4. Generates a grounded natural-language answer.

The embedded text concatenates the review body with physician name, patient name, and hospital name — so semantic search works for queries that mention specific entities, not just review content.

**Graph Chain (Text-to-Cypher RAG)**

For objective questions like *"Which physician has billed the most to Cigna?"*, the system uses `GraphCypherQAChain` to translate natural language into Cypher queries. This runs in two LLM passes:

1. **Cypher generation** — given the user's question and the Neo4j schema, generate a Cypher query. The prompt includes few-shot examples, schema-aware instructions, valid category values, and guardrails against destructive queries.
2. **Answer generation** — given the question and the query results, write a human-readable response that grounds itself only in the returned data.

Two separate LLMs can be configured for these passes via environment variables — a stronger model for query generation (where precision matters), a cheaper one for answer composition.

### The Agent

A LangChain agent built with `create_openai_functions_agent` reads tool descriptions and chooses which to call per question. Tool descriptions explicitly define both positive scope (*"useful for...")* and negative scope (*"not useful for..."*) to reduce routing ambiguity.

The agent runs inside an `AgentExecutor` with `return_intermediate_steps=True`, so every response includes the reasoning trace — visible to users in the UI as a collapsible "How was this generated?" panel.

---

## The Data Model

The hospital system is modeled as a graph in Neo4j with six node types and six relationship types:

**Nodes:** `Hospital`, `Physician`, `Patient`, `Visit`, `Payer`, `Review`

**Relationships:**

```
(Patient) -[HAS]-> (Visit) -[AT]-> (Hospital)
                     |               |
                     | -[COVERED_BY {service_date, billing_amount}]-> (Payer)
                     |
                     | -[WRITES]-> (Review)
                     |
              (Physician) -[TREATS]-> (Visit)
                     ^
                     |
              [EMPLOYS]
                     |
              (Hospital)
```

`Visit` acts as the central fact node connecting almost everything — analogous to a fact table in dimensional modeling. The `COVERED_BY` relationship carries `billing_amount` and `service_date` as properties, demonstrating one advantage of graph data modeling: facts that belong to relationships, not entities, can be stored there naturally.

**Synthetic dataset** loaded from CSV:
- 30 hospitals across multiple states
- 500 physicians
- ~10,000 patients
- ~10,000 visits with admission types, diagnoses, billing amounts, and statuses
- ~1,000 patient reviews
- 5 insurance payers (Cigna, Blue Cross, UnitedHealthcare, Medicaid, Aetna)

---

## Example Questions

The system handles a wide range of question types, routed automatically to the right chain:

**Graph Chain handles:**
- *"How many emergency visits were there in 2023?"*
- *"Which physician has the lowest average visit duration?"*
- *"What was the total billing amount charged to Cigna in 2023?"*
- *"Which state had the largest percent increase in Medicaid visits from 2022 to 2023?"*
- *"For visits that are not missing chief complaints, what percentage have reviews?"*
- *"How much was billed for patient 789's stay?"*

**Experiences Chain handles:**
- *"Are patients satisfied with the nursing staff?"*
- *"At which hospitals are patients complaining about billing and insurance issues?"*
- *"Have any patients complained about noise or cleanliness?"*
- *"What are patients saying about communication with doctors?"*

---

## Project Structure

```
langchain-rag-app/
├── chatbot_api/                # FastAPI service — the core deliverable
│   ├── src/
│   │   ├── agents/             # LangChain agent definition
│   │   ├── chains/             # Vector chain + Cypher chain
│   │   ├── models/             # Pydantic request/response schemas
│   │   ├── utils/              # Async retry decorator
│   │   └── main.py             # FastAPI entry point
│   ├── Dockerfile
│   └── pyproject.toml
│
├── chatbot_frontend/           # Streamlit chat UI
│   ├── src/main.py
│   ├── Dockerfile
│   └── pyproject.toml
│
├── hospital_neo4j_etl/         # One-shot ETL: CSV → Neo4j
│   ├── src/hospital_bulk_csv_write.py
│   ├── Dockerfile
│   └── pyproject.toml
│
├── data/                       # Source CSVs (hospitals, physicians, etc.)
├── tests/                      # Sync vs async benchmark scripts
├── docker-compose.yml
└── .env                        # API keys, Neo4j credentials, model config
```

---

## Getting Started

### Prerequisites

- Docker and Docker Compose
- A Neo4j AuraDB instance (free tier works) — get one at [neo4j.com/cloud/aura-free](https://neo4j.com/cloud/aura-free/)
- An OpenAI API key

### Setup

1. **Clone the repository:**
   ```bash
   git clone <repo-url>
   cd langchain-rag-app
   ```

2. **Create a `.env` file in the project root:**
   ```env
   # OpenAI
   OPENAI_API_KEY=sk-...

   # Neo4j
   NEO4J_URI=neo4j+s://xxxxx.databases.neo4j.io
   NEO4J_USERNAME=neo4j
   NEO4J_PASSWORD=your-password

   # Model selection (mix-and-match for cost/quality)
   HOSPITAL_AGENT_MODEL=gpt-3.5-turbo-1106
   HOSPITAL_CYPHER_MODEL=gpt-3.5-turbo-0125
   HOSPITAL_QA_MODEL=gpt-3.5-turbo-0125

   # Data sources (CSV URLs)
   HOSPITALS_CSV_PATH=https://raw.githubusercontent.com/.../hospitals.csv
   PAYERS_CSV_PATH=https://raw.githubusercontent.com/.../payers.csv
   PHYSICIANS_CSV_PATH=https://raw.githubusercontent.com/.../physicians.csv
   PATIENTS_CSV_PATH=https://raw.githubusercontent.com/.../patients.csv
   VISITS_CSV_PATH=https://raw.githubusercontent.com/.../visits.csv
   REVIEWS_CSV_PATH=https://raw.githubusercontent.com/.../reviews.csv

   # Frontend → backend URL (Docker internal DNS)
   CHATBOT_URL=http://chatbot_api:8000/hospital-rag-agent
   ```

3. **Build and start the stack:**
   ```bash
   docker-compose up --build
   ```

   This launches three services:
   - `hospital_neo4j_etl` — loads CSV data into Neo4j (runs once, then exits)
   - `chatbot_api` — FastAPI backend on port 8000
   - `chatbot_frontend` — Streamlit UI on port 8501

4. **Open the chat UI:**
   Visit [http://localhost:8501](http://localhost:8501) and start asking questions.

5. **Or hit the API directly:**
   ```bash
   curl -X POST http://localhost:8000/hospital-rag-agent \
     -H "Content-Type: application/json" \
     -d '{"text": "Which physician has billed the most to Cigna?"}'
   ```

---

## Key Design Decisions

**Why a graph database?**
LLMs generate Cypher more reliably than SQL with complex joins. Graph schemas — nodes, relationships, properties — map cleanly to natural-language question structure. Neo4j also stores vectors natively, so structured and unstructured retrieval share one database.

**Why an agent instead of a router?**
Routing decisions are themselves natural-language understanding tasks. An LLM-based agent handles ambiguous phrasing, mixed intents, and extends to new tools with prompt changes alone. The tradeoff — slightly more latency and less deterministic routing — is acceptable for the use case.

**Why two LLMs in the Cypher chain?**
Cypher generation is high-stakes (one wrong relationship direction returns silently wrong data) and benefits from a stronger model. Answer composition is low-stakes (summarizing already-correct data) and runs fine on a cheaper model. Splitting them lets you optimize the cost-quality tradeoff per task via environment variables.

**Why FastAPI async?**
LLM calls are I/O-bound and slow (5–20 seconds each). Async FastAPI handles other requests during the wait, dramatically improving throughput under concurrent load. Test scripts in `tests/` benchmark sync vs async — async processes 14 questions in ~45 seconds vs ~180 seconds sequentially.

**Why expose intermediate steps?**
Trust. Agentic systems feel like black boxes; surfacing the agent's tool calls and the generated Cypher in the UI makes the system auditable. Users can verify *how* an answer was derived, not just *what* the answer was.

---

## Known Limitations

This is a learning project that demonstrates the full RAG stack end-to-end. It is not production-grade. Specific limitations:

- **No conversation memory** — every query is stateless; follow-ups like "tell me more about that" don't work.
- **No authentication** — anyone with the URL has full data access.
- **Cypher correctness is best-effort** — `validate_cypher=True` catches syntax errors, but semantically wrong queries return wrong answers confidently.
- **No evaluation harness** — prompt changes are validated by inspection, not by automated regression tests.
- **No response streaming** — users wait for full generation before seeing output.
- **Pure semantic retrieval** — no hybrid (keyword + vector) search, no metadata pre-filtering on the vector chain.

A production rebuild would prioritize: an eval harness first, then a query verification step, then memory, then streaming, then auth.

---
