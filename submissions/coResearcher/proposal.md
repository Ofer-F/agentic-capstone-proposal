# coResearcher

**One-liner:** A multi-agent research assistant that turns a researcher's brief into ranked, feasibility-scored research plans — for grad students, PIs, and R&D teams deciding what to actually pursue.
**Built by:** Ofer Feinstein  ·  **Repo (public preferred):** <repo URL>  ·  **Demo (video/live URL):** <live URL>  ·  **Try it:** `cd backend && pip install -e ".[dev]" && pytest` (runs the full agent loop offline), or open the live URL

---

## 1. The problem & who it's for  *(Product · Customers)*
Researchers waste weeks at the fuzzy front end: brainstorming questions, scanning literature, sketching methods, and guessing what's actually feasible with their data, time, and skills. A generic chatbot will happily invent a plan, but it won't ground it in real papers, won't score feasibility on a consistent rubric, and won't keep a human in control of the two decisions that matter (which questions to pursue, which plan to commit to). coResearcher is for a specific moment: a researcher who has a rough brief and their own constraints and needs a *ranked shortlist of defensible plans* backed by real citations — not prose.

## 2. What it does  *(Product · Ease of use)*
Three core flows:
1. **Brief → candidate questions.** User submits a brief (plus optional context and their own data/constraints). The Ideator generates diverse, tagged research questions with rationales. **HITL Gate 1:** the user picks which proceed.
2. **Questions → grounded plans.** For each selected question the system runs a literature review over OpenAlex/arXiv/Semantic Scholar, synthesizes methodology, and drafts a structured plan (objective, hypotheses, methods, data, timeline, risks, resources).
3. **Plans → ranked decision.** The Judge scores every plan against a weighted feasibility rubric and ranks them. **HITL Gate 2:** the user approves one plan, then one-click **exports it to Notion**.
The magic moment is what happens between the Ideator and the plan: a selected question is turned into a grounded, structured research plan — the system runs its own literature review over real papers, synthesizes a methodology, and drafts objective, hypotheses, methods, data, timeline, risks, and resources, with progress and running cost streaming over SSE the whole time.

## 3. The agentic core  *(Agentic depth)*
- **The loop / reasoning:** A LangGraph threads one research state through ideate → [gate] → literature review → methodology → research plan → judge → [gate]. The **literature agent** runs its own **plan → act → observe** sub-loop: it drafts 2–4 queries, searches, assesses coverage, and refines up to 2 iterations before selecting papers.
- **Tools / actions:** Literature search over OpenAlex, arXiv, Semantic Scholar, direct Postgres persistence, and **Notion page creation via an MCP subprocess**.
- **Autonomy:** Runs headless in a background task, pausing only at the two HITL gates. It self-terminates on any budget breach.
- **Multi-agent:** Five specialist agents (Ideator, Literature, Methodology, Plan, Judge) orchestrated by the graph, each pure async LLM logic with structured output.
- **Memory / state:** Full pause/resume across gates via a **Postgres LangGraph checkpointer**; every question, paper, plan, and score is persisted.

## 4. Architecture  *(Engineering excellence)*

```
Next.js UI  --POST brief / SSE-->  FastAPI  -->  LangGraph orchestrator
                                                   |
   Ideator -> [HITL: pick questions] -> Literature review (plan/act/observe)
     -> Methodology -> Research plan -> Judge (rubric scoring + rank)
                                                   |
   OpenAlex / arXiv / Semantic Scholar        Supabase (Postgres: data + checkpoints)
                                                   |
                              [HITL: approve plan] -> Notion export (MCP)
```

- **Components & data flow:** Next.js UI → FastAPI → LangGraph orchestrator → OpenAI + literature tools + Supabase Postgres (domain data + checkpoints) → Notion via MCP. Responsibilities are layered so LLM logic stays pure: graph nodes wire agents to state, agents hold prompts only, and injectable dependencies make the LLM and search calls overridable.
- **Robustness:** Budget guards short-circuit any run to a terminal capped state; a clear status lifecycle covers capped and error cases; the API uses a consistent error envelope; cost/token/tool-call usage is tracked live; health and readiness checks expose DB status.
- **Tests:** **54 tests** across the suite, runnable offline. Injectable dependencies and test doubles let the *entire* agent loop, both HITL gates, ranking, and budget capping run with no API keys. A passing test suite is the strongest single proof.

**Agent architecture.** The heart of coResearcher is a **LangGraph state machine** that threads a single research state through processing nodes and two interrupt gates:

```
START -> ideator -> [gate: pick questions] -> literature_review
      -> methodology -> research_plan -> judge -> [gate: approve plan] -> END
```

Responsibilities are split across layers so the LLM logic stays pure and testable:

| Layer            | Role                                                             |
| ---------------- | --------------------------------------------------------------- |
| Graph nodes      | Wire agents to state, persist to DB, enforce budgets, run gates. |
| Agents           | Pure async LLM logic: prompts + structured output.              |
| LLM helper       | OpenAI call + parsing + cost tracking.                          |
| Injectable deps  | LLM and search calls are overridable (real vs. test doubles).   |
| Tools            | Literature search over OpenAlex, arXiv, Semantic Scholar.       |
| Guards           | Budget caps + prompt-injection wrapping of untrusted text.      |
| Execution        | Background streaming, pub/sub that feeds the SSE stream.         |
| Checkpointing    | Postgres saver enabling pause/resume across HITL gates.         |

The five specialist agents: **Ideator** generates diverse, tagged candidate questions (runs hot for breadth); **Literature review** runs a **plan → act → observe** loop per question, drafting 2–4 queries, searching, assessing coverage, and refining up to 2 iterations before selecting papers; **Methodology** synthesizes candidate methods, datasets, and gaps; **Research plan** drafts a structured plan (objective, hypotheses, methods, data, risks, resources); **Judge** scores each plan 1–5 against the weighted feasibility rubric — data availability (0.25), methodological soundness (0.25), scope/time realism (0.20), novelty (0.15), resource/skill fit (0.15) — and ranks them.

## 5. Safety & control  *(Safety & control)*
- **Human-in-the-loop on every meaningful decision:** the graph cannot proceed past question selection or plan approval without explicit human input (LangGraph interrupts). Notion export requires an *already-approved* plan and is a reversible create.
- **Hard budget caps** enforced before every node: tool calls (40), tokens (400k), cost ($5), wall-clock (15m), plus question/paper counts — breaching any caps the run and ends it.
- **Prompt-injection handling:** all external abstracts/tool output are wrapped in untrusted-data blocks with system guidance to treat them as data only; nested delimiters are defanged so a malicious document can't close the block early. Example malicious abstract content we neutralize and ignore:
- **Data & secrets:** Supabase row-level security is on for every table with **no permissive policies** (service-role backend only); secrets stay in environment variables (gitignored). No high-harm autonomous actions: the agent never spends money beyond capped LLM calls, never messages third parties, and takes no unattended irreversible action.

## 6. Engineering highlights  *(Engineering excellence)*
- **Testable-by-design DI:** injectable dependencies swap real LLM/search for test doubles without touching graph wiring, so the whole system is verifiable offline.
- **Durable HITL via Postgres checkpointing:** exact pause/resume across gates, including custom serialization.
- **Live cost governance:** token/USD/tool-call accounting flows into state so caps are enforced *mid-run*, not after.
- **Streaming pub/sub:** a run manager bridges background streaming to SSE events (node, cost, interrupt, completed) for real-time UI.

## 7. Hardest problem solved  *(Complexity & difficulty)*
Making a long multi-agent run *pausable, resumable, and budget-bounded* at once. Async LangGraph interrupts plus a Postgres checkpointer let a run halt at a gate, survive a disconnect, and resume from the exact node once the client responds — while budget guards can still terminate it cleanly. Proven by the offline test suite's full-run and budget-breach tests.

## 8. Potential & MOAT  *(Potential · MOAT)*
Buyers: researchers in the academia or in industrial R&D teams that fund feasibility triage. The moat is the *workflow* — a rubric-scored, citation-grounded, human-gated pipeline that produces auditable ranked plans — plus accumulated per-domain feasibility judgments and integrations (literature sources today, lab data/Notion/reference managers next). Next milestone: a deployed multi-user instance with saved projects and a calibrated feasibility rubric validated against real project outcomes.

## 9. Built across the fellowship  *(context only)*
- [x] **Agent harness (WS1)** — LangGraph state machine, injectable dependencies, structured-output LLM helper.
- [x] **Skills & product packaging (WS2)** — Next.js streaming UI: brief form, live timeline, two gates, results.
- [x] **MCP server / tools & security (WS3)** — Notion export over MCP subprocess; literature tools; untrusted-input wrapping.
- [x] **Autonomous agent (WS4)** — headless background run loop, budget caps, HITL gates, SSE observability, live cost tracking.
- [x] **Cross-agent / sub-agents (WS5)** — five specialist agents orchestrated by the graph, with a plan/act/observe sub-loop in the literature agent.

## 10. Evidence index  *(curate, don't dump)*
- **Runnable test:** `cd backend && pip install -e ".[dev]" && pytest` — **demonstrates** the full agent loop, both HITL gates, ranking, and budget capping offline (54 tests).
- **No hosted demo URL is pasted here — by design.** The app is demoed over a temporary Cloudflare quick-tunnel, which assigns a **new random `*.trycloudflare.com` hostname on every run**, so any link written into this doc would go stale and 404. Instead, the repo has everything needed to run the full end-to-end flow (brief → questions → gate → plans → gate → Notion export) locally.
  - *Why not a one-click Vercel deploy:* Vercel can host the Next.js **frontend**, but not this **backend**. The backend is intentionally stateful/long-lived in ways serverless functions don't support: it drives **background LangGraph runs up to 15 minutes**, streams progress over a **persistent SSE connection** fed by **in-process pub/sub** (which needs the stream and the run on the same live process), holds a **long-lived Postgres connection pool + LangGraph checkpointer**, and spawns a **Notion MCP subprocess**. Vercel's serverless model (short execution limits, no durable background tasks, no guaranteed instance affinity for a stream, no subprocesses) breaks all four. A production deploy therefore splits it: frontend on Vercel + backend on an always-on host (Render / Railway / Fly.io).
- **Run it on your own machine (instructions in the repo):** clone the repo and run the two services locally with your own keys — no hosting needed, it always works. Requires **Python 3.14** (needed for the async HITL `interrupt()`), **Node 18+**, a **Supabase Postgres** project, and an **OpenAI API key**. Notion export is optional (set `NOTION_TOKEN` + `NOTION_PARENT_PAGE_ID` to enable it). Quick start:

```bash
# 1. Secrets: copy the template, then fill in OPENAI_API_KEY + the Supabase values
cp .env.example .env

# 2. Database: apply the schema to your Supabase Postgres
psql "$SUPABASE_DB_URL" -f supabase/migrations/0001_init.sql

# 3. Backend (FastAPI + LangGraph) on :8000
cd backend && python3.14 -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"
uvicorn app.main:app --port 8000        # verify: http://localhost:8000/health

# 4. Frontend (Next.js) on :3000 — in a second terminal
cd frontend && cp .env.example .env.local   # NEXT_PUBLIC_API_BASE_URL defaults to http://localhost:8000
npm install && npm run dev              # open http://localhost:3000
```

- **Repo:** https://github.com/Ofer-F/coResearcher — the LangGraph orchestrator, graph nodes, budget guards, and the five agents.