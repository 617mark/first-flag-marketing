# CLAUDE.md

Context and working agreement for Claude Code in this repo.

- **Project path:** `/Users/markmcinnis/Develop/first-flag-marketing`
- **Owner:** Mark McInnis (mark@firstflagusa.com), First Flag Web Development
- **Read first, every session:** `docs/LEARNING_LOG.md` (where we left off), then `docs/ROADMAP.md` (the step list), `docs/SPEC.md` (what we're building), `docs/DECISIONS.md` (why)

## The point of this repo

Mark is building this system **to learn** uv, Python, PostgreSQL, SQLAlchemy/Alembic, LangGraph, LangSmith, and LangGraph Studio, and to see how they fit together. Understanding matters more than speed. A working feature Mark doesn't understand counts as unfinished.

Mark is an experienced developer from a JavaScript/Express/React background and is new to Python. Explain each Python idiom the first time it appears, and compare it to the JS equivalent when that helps:

| Python | JS equivalent |
|---|---|
| `pyproject.toml` | `package.json` |
| `uv.lock` | `package-lock.json` |
| `.venv/` | `node_modules/` (plus its own Python interpreter) |
| `uv add httpx` | `npm install httpx` |
| `uv run firstflag …` | `npx …` / `npm run …` |
| `TypedDict` | TypeScript `interface` |
| Pydantic model | Zod schema |
| SQLAlchemy + Alembic | Sequelize/Knex + migrations |

## The step loop (follow this for every roadmap step)

Work **one roadmap step at a time**. Each step adds one small, working, visible feature. Never chain steps.

**1. Orient.** Name the step (e.g., "B2 — Wire the skeleton graph"), what it builds, and the one new concept it teaches. Keep it to a few sentences.

**2. Ask how to run it.** Use AskUserQuestion with these options:
   - **Claude does it:** you write the files and run the commands, then walk Mark through what changed.
   - **Mark does it:** you give exact commands and complete file contents (never diffs or fragments); Mark types or pastes them and reports the output.
   - **Split:** you write the code; Mark runs the commands.

   If Mark has set a default for the current stage ("you do all of stage C"), use it and skip the question, but still stop at step 5.

**3. Build.** Do the work the chosen way. Stay inside the step's scope. If something outside scope needs changing, stop and ask.

**4. Show.** Tell Mark exactly where to look. Include every view that applies:
   - **Code:** `path:line` references and what to notice in each.
   - **Run:** the command and the expected output.
   - **Studio:** which graph, which node, what to click, what to see in the state panel.
   - **LangSmith:** project `first-flag-outreach`, which trace, what to expand.
   - **Database:** the `psql` query that shows the rows.

**5. Check and pause.** Give one check Mark can run himself (change a value and predict the result, or break something on purpose and read the error). Append an entry to `docs/LEARNING_LOG.md` using the template in that file. Then stop and ask: continue, repeat with a variation, or dig deeper? **Do not start the next step until Mark says go.**

When something fails, explain the error in plain terms before fixing it. Errors are learning material.

## Conventions

- Python 3.12, managed with `uv`. Add deps with `uv add`, never `pip install`.
- Add a dependency in the step that first needs it, not ahead of time.
- Package code lives in `src/firstflag/`. Type hints everywhere.
- Graph code: `state.py` (state TypedDicts), `nodes.py` (state in → partial update out), `edges.py` (routing functions), `graph.py` (wiring; exports `graph`).
- Stub nodes from the walking skeleton are commented `# STUB` until replaced; grep for `STUB` shows what's left.
- Nodes stay thin; external calls go in `services/`, DB access in `db/repo.py`.
- Graphs compile without a checkpointer; the `langgraph dev` server supplies persistence.
- Postgres is the source of truth for lead status.
- Every LLM call gets a LangSmith run name and tags (`trade`, `lead_id`).
- Output complete files, not diffs or partial edits.
- One git commit per completed step, message prefixed with the step ID (e.g., `B2: wire skeleton graph`).

## Hard rules

- Never send email except through the `send` node, with a suppression re-check and the daily cap.
- `DRY_RUN=true` is the default and redirects every send to mark@firstflagusa.com. Only Mark flips it.
- The LLM writes only `subject` and `custom_line`. The template and footer live in `templates.py`.
- Never automate sending through Google Voice.
- Never commit `.env`, API keys, or OAuth tokens.
- Log any design change as a new entry in `docs/DECISIONS.md`, and ask Mark before changing `SPEC.md` or `ROADMAP.md`.
