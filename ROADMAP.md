# Build Roadmap

Small steps, each one working and visible before the next begins. Claude Code follows the step loop in `CLAUDE.md` for every step: orient → ask who runs it → build → show where to look → check and pause.

**How to read each step**
- **Builds:** the one feature added
- **Learn:** the concept it teaches
- **See it:** where to look — 📄 code · ▶️ terminal · 🎛 Studio · 🔍 LangSmith · 🗄 Postgres

**The shape of the plan.** Stages A–B teach the tools in isolation with toy code. Stage C builds a **walking skeleton**: the full outreach graph with every node stubbed, so the whole flow is visible in Studio on day two. Stages D–K replace one stub at a time with the real thing. Whatever you're working on, you can always see where it sits in the full picture.

**Start the domain warm-up now** (see J1). It needs two weeks and runs in parallel with everything else.

---

## Stage A — Python environment and Postgres

### A1 · Create the uv project
- **Builds:** `uv init`, Python 3.12 pinned, `.venv`, `pyproject.toml`, a `hello.py` that prints.
- **Learn:** what uv manages (Python versions, the venv, dependencies, the lockfile) and how it maps to npm.
- **See it:** 📄 `pyproject.toml`, `.python-version` · ▶️ `uv run python hello.py`, `uv run python -c "import sys; print(sys.executable)"` (shows Python running from `.venv`)

### A2 · Package layout and a CLI
- **Builds:** `src/firstflag/` package with `__init__.py`, a `cli.py` using Typer with one `hello` command, and a `[project.scripts]` entry so `firstflag` is a command.
- **Learn:** packages vs. modules, `__init__.py`, imports, the `src/` layout, entry points, `if __name__ == "__main__"`.
- **See it:** 📄 `src/firstflag/cli.py`, `[project.scripts]` in `pyproject.toml` · ▶️ `uv run firstflag hello Mark`

### A3 · Config and secrets
- **Builds:** `.env`, `.env.example`, `.gitignore`, `config.py` using pydantic-settings; `firstflag config` prints settings with secrets masked.
- **Learn:** typed settings, where env vars come from, why `.env` never gets committed.
- **See it:** 📄 `src/firstflag/config.py` · ▶️ `uv run firstflag config`

### A4 · Postgres running and reachable
- **Builds:** Postgres.app installed, `firstflag` database created, `firstflag db-ping` connects from Python and prints the server version.
- **Learn:** connection URLs, the `psql` client, the SQLAlchemy engine.
- **See it:** 🗄 `psql firstflag -c "select version();"` · ▶️ `uv run firstflag db-ping`

---

## Stage B — LangGraph fundamentals (toy graphs)

Toy code lives in `src/firstflag/graphs/playground/` and stays as reference.

### B1 · One-node graph, run from Python
- **Builds:** a state with one field, one node, START → node → END, invoked from a script.
- **Learn:** state as a `TypedDict`, a node as a plain function, `StateGraph`, `compile()`, `invoke()`.
- **See it:** 📄 `graphs/playground/hello_graph.py` · ▶️ `uv run python -m firstflag.graphs.playground.hello_graph`

### B2 · Two nodes and partial updates
- **Builds:** a second node; each returns only the keys it changes; add a list field with a reducer (`Annotated[list, operator.add]`).
- **Learn:** how LangGraph merges partial updates into state; overwrite vs. append reducers.
- **See it:** ▶️ print state after each node with `graph.stream(..., stream_mode="updates")`

### B3 · Open it in LangGraph Studio
- **Builds:** `langgraph.json` pointing at the playground graph; `uv run langgraph dev`.
- **Learn:** what the dev server does (API, thread persistence, Studio connection), threads vs. runs.
- **See it:** 🎛 the graph drawing, then submit an input, watch nodes light up, click each node to see the state it produced, open the thread history

### B4 · Turn on LangSmith
- **Builds:** `LANGSMITH_API_KEY`, `LANGSMITH_TRACING=true`, project `first-flag-outreach`.
- **Learn:** traces, runs, and spans; how Studio and LangSmith relate (Studio = run and inspect one graph live; LangSmith = history of every run).
- **See it:** 🔍 the trace from a Studio run, expanded node by node

### B5 · A conditional edge
- **Builds:** a router function that reads state and returns the next node's name; two branches.
- **Learn:** `add_conditional_edges`, routing functions, why routing logic belongs in `edges.py`.
- **See it:** 🎛 run twice with inputs that take different branches; compare the paths in Studio

### B6 · A loop
- **Builds:** a node that retries until a counter or condition is met.
- **Learn:** cycles in a graph, guarding against infinite loops, recursion limits.
- **See it:** 🎛 watch the node repeat; 🔍 the repeated spans in the trace

### B7 · Pause for a human
- **Builds:** a node that calls `interrupt()` with a payload; resume it from Studio with a value.
- **Learn:** interrupts, why they need persistence, `Command(resume=…)`. This is the mechanism behind every approval later.
- **See it:** 🎛 the paused thread, the interrupt payload, the resume box; the state after resuming

### B8 · First LLM call
- **Builds:** one node that calls Claude through LangChain's `ChatAnthropic`, plus a second that uses `with_structured_output` and a Pydantic model.
- **Learn:** LangChain chat models, messages, structured output, model config.
- **See it:** 🔍 the LLM span: prompt, response, tokens, latency, cost · 🎛 the structured object in state

---

## Stage C — Walking skeleton of the outreach graph

### C1 · Outreach state
- **Builds:** `graphs/outreach/state.py` with `OutreachState`, `Draft`, `Review` (spec §4.2).
- **Learn:** `NotRequired`, `Literal`, nested TypedDicts; designing state as the contract between nodes.
- **See it:** 📄 `graphs/outreach/state.py`

### C2 · Every node as a stub, fully wired
- **Builds:** `nodes.py` with all eight nodes returning hard-coded data (`# STUB`); `edges.py` with real routing; `graph.py` wiring it all; added to `langgraph.json`.
- **Learn:** the file split (state / nodes / edges / graph) and why it scales.
- **See it:** 🎛 the full outreach graph; run a fake lead end to end, including the approval interrupt; make it hit `park` by changing the stubbed score

### C3 · Tests for the routing
- **Builds:** pytest tests for `after_score`, `after_validate`, `after_review`.
- **Learn:** pytest basics, fixtures, testing graph logic without running the graph.
- **See it:** ▶️ `uv run pytest -v`

From here on, `grep -rn STUB src/` shows what's left to build.

---

## Stage D — The database

### D1 · First models
- **Builds:** SQLAlchemy 2.0 models for `businesses` and `leads` in `db/models.py`; `db/session.py`.
- **Learn:** declarative models, `Mapped[...]` types, sessions, the unit of work.
- **See it:** 📄 `db/models.py`

### D2 · First migration
- **Builds:** Alembic initialized; autogenerated migration; `alembic upgrade head`.
- **Learn:** how migrations are generated, reviewed, applied, and rolled back.
- **See it:** 📄 `migrations/versions/…` · 🗄 `\d businesses`, `\d leads`

### D3 · Seed and query
- **Builds:** `db/repo.py` helpers (`create_lead`, `get_lead`, `set_status`); `firstflag seed-fake` inserts one fake business and lead.
- **Learn:** queries with `select()`, commits, JSONB columns.
- **See it:** 🗄 `select * from leads;`

### D4 · The skeleton reads and writes the DB
- **Builds:** the stubbed `research` node loads its business from Postgres; `park`, `reject`, and `send` stubs write status.
- **Learn:** where DB access lives relative to nodes; Postgres as truth vs. graph state as working memory.
- **See it:** 🎛 run the seeded lead · 🗄 watch `leads.status` change

### D5 · Remaining tables
- **Builds:** `messages` and `suppression` models + migration.
- **See it:** 🗄 `\dt`

---

## Stage E — Sourcing real businesses

### E1 · One Places query
- **Builds:** `services/places.py`; `firstflag places-test "hvac contractor" --near Bedford` prints results.
- **Learn:** calling an external API with httpx, API keys, response models.
- **See it:** ▶️ printed results

### E2 · Save to the database
- **Builds:** upsert by `place_id`, distance from Bedford, `new` lead per business.
- **Learn:** upserts (`on_conflict_do_update`), idempotent jobs.
- **See it:** 🗄 rows; rerun and confirm no duplicates

### E3 · Cover the 45-mile territory
- **Builds:** grid of search circles; `firstflag source --trade hvac`.
- **See it:** ▶️ counts per circle · 🗄 `select town, count(*) from businesses group by town;`

### E4 · Queue leads into the graph
- **Builds:** `firstflag queue --trade hvac --limit 5` starts one thread per lead via the LangGraph SDK.
- **Learn:** the LangGraph SDK client, threads from code vs. from Studio.
- **See it:** 🎛 five new threads, each paused at approval (stubs still in place)

---

## Stage F — Research node (first real LLM node)

### F1 · Fetch site text
- **Builds:** `services/web.py` fetches the homepage + likely contact/booking pages and extracts text; emails pulled by regex and `mailto:` links.
- **See it:** ▶️ `firstflag fetch-site <url>`

### F2 · Structured research
- **Builds:** prompt + Pydantic `Research` model; replaces the `research` stub.
- **See it:** 🎛 the `research` object in state · 🔍 the prompt and structured response · 🗄 `leads.research` JSONB

---

## Stage G — Score node

### G1 · Hard rules
- **Builds:** no email / chain / suppressed → park, in plain code.
### G2 · LLM fit rating
- **Builds:** 1–10 score with a reason; threshold from config; replaces the stub.
- **See it:** 🎛 leads splitting between `draft` and `park` · 🔍 compare a high and low score's reasoning

---

## Stage H — Drafting

### H1 · Email template
- **Builds:** `templates.py` assembly and locality phrase; tests.
- **See it:** ▶️ `uv run pytest tests/test_templates.py`

### H2 · Playbooks
- **Builds:** `playbooks/hvac.yaml`, loader with Pydantic validation.
- **Learn:** validating config files; why the offers live in YAML.

### H3 · Draft node
- **Builds:** subject + custom line from playbook + observations; replaces the stub.
- **See it:** 🎛 drafts in state · 🔍 the draft prompt with research injected

### H4 · Validate node and the retry loop
- **Builds:** word limits, footer, banned phrases, grounding check; replaces the stub.
- **See it:** 🎛 a draft failing validation and looping back · 🔍 both attempts

---

## Stage I — Real approval

### I1 · Approval payload and the four actions
- **Builds:** real `approve` node and `after_review` routing; approve / edit / revise / reject.
- **See it:** 🎛 resume a lead each way; confirm revise notes reach the next draft · 🗄 statuses

---

## Stage J — Sending

### J1 · Domain setup (start during Stage A)
- Google Workspace mailbox mark@firstflagusa.com; SPF, DKIM, DMARC; Google Postmaster Tools; two weeks of normal mail.

### J2 · Gmail API, dry run
- **Builds:** OAuth for one user; `services/gmail.py`; `firstflag send-test` mails yourself.
### J3 · Send node
- **Builds:** suppression re-check, daily cap, send window, `DRY_RUN` redirect; replaces the stub.
- **See it:** your inbox · 🗄 `messages` rows with Gmail IDs
### J4 · First five real sends
- Flip `DRY_RUN=false` yourself.

---

## Stage K — Replies and follow-ups

### K1 · Reply watch
- **Builds:** `firstflag watch` polls threads, marks `replied`, notifies you, suppresses opt-outs; `firstflag mark-replied`.
### K2 · Follow-up graph
- **Builds:** `graphs/followup/` (draft bump → approve → send); `firstflag followups`.
- **See it:** 🎛 second graph in Studio
### K3 · Scheduling
- **Builds:** `launchd` jobs for watch, follow-ups, queued sends.

---

## Stage L — Evals and tuning (ongoing)
- LangSmith dataset from approved vs. edited/rejected drafts.
- LLM-judge evaluators for tone, length, grounding.
- Reply rate by trade and offer; retire weak offers.

---

## Parallel track (not code)
- Website v1 on firstflagusa.com: headline, subhead, Loonen story, your photo, phone, email.
- Mailing address for the footer, Loonen's permission and quote, playbook content, pricing anchor.
