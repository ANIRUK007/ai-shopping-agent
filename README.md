# AI Shopping Agent

Ask for a product in plain English — *"Find the best laptop for AI development under ₹70,000"* —
and an LLM agent researches the live web, extracts real products, ranks them against your
budget and the specs that matter, and recommends the top picks with prices and source links.

## How it works

```
browser  ──POST /api/search──▶  FastAPI  ──▶  Gemini agent loop (function calling)
                                                 │
                                    ┌────────────┼────────────┐
                                    ▼            ▼            ▼
                              web_search   extract_products  compare_products
                              (Tavily)     (Gemini JSON      (deterministic
                                            extraction)       Python scoring)
                                                 │
                              SQLite  ◀──────────┘  query + answer + products + tool trace
```

The agent decides *what* to search, *how many times*, and *which specs matter* for the stated
use case. It runs up to 8 reasoning steps; each tool call is recorded and shown in the UI, so
the agentic workflow is visible rather than a black box.

### The three tools

| Tool | Implementation | Why |
|---|---|---|
| `web_search(query)` | Tavily API | Live public web. The model calls it 2-4× with different angles. |
| `extract_products()` | Gemini with a JSON `response_schema` | Turns messy search text into `{name, price_inr, key_specs, source_url}`. |
| `compare_products(budget, specs)` | Plain Python (`compare` in `app.py`) | Ranking is deterministic and testable — the LLM picks the criteria, not the maths. |

Scoring: in-budget items score `50 + 20·(price/budget)` (spending more of the budget usually buys
better hardware), over-budget items are penalised proportionally to the overshoot, and each
matched must-have spec adds 15. Duplicates are merged on a normalised product name, preferring
the row that actually carries a price.

## Run it

```bash
python3 -m venv .venv && .venv/bin/pip install -r requirements.txt
cp .env.example .env        # then fill in the two keys
.venv/bin/uvicorn app:app --reload --port 8000
```

Open http://localhost:8000

Keys — both have free tiers:
- `GEMINI_API_KEY` — https://aistudio.google.com/apikey
- `TAVILY_API_KEY` — https://app.tavily.com (1000 searches/month free)

## Test

```bash
.venv/bin/python test_agent.py
```

Covers dedupe and the ranking logic (budget fit, over-budget penalty, spec matching, no-budget case).

## API

| Endpoint | Purpose |
|---|---|
| `POST /api/search` `{"query": "..."}` | Run the agent. Returns `{id, answer, products, trace}`. |
| `GET /api/history` | Last 25 searches. |
| `GET /api/history/{id}` | Full stored result, including the tool trace. |

## Storage

SQLite (`shopping.db`, created on first run) — one `searches` row per request holding the query,
the final recommendation, the ranked products and the agent's tool trace.

## Deliberate scope limits

- Search results are Tavily snippets, not per-retailer scrapes — prices are indicative, not live checkout prices.
- Synchronous request (20-60s per search). A job queue + polling is the next step if this ever gets traffic.
- No auth, no rate limiting — it's a local MVP.
