# Decisions Log

Format: decision, why, consequences. Newest at the bottom. Change a decision by adding a new entry that supersedes the old one.

---

### D-001 · Email only for cold outreach; no cold SMS — 2026-09-29
**Why:** TCPA treats automated marketing texts to cell phones as requiring prior written consent, even for business owners; per-message penalties. Carriers require opt-in proof for A2P 10DLC registration. CAN-SPAM permits cold B2B email with honest headers, a physical address, and an opt-out.
**Consequences:** Text only after a prospect replies and invites it. Google Voice (617-575-9799) is a manual reply channel only, never automated.

### D-002 · Human approves every message — 2026-09-29
**Why:** The pitch is "a real local person," and the approval step makes that literally true. It also protects domain reputation.
**Consequences:** Throughput is capped by Mark's review time. That's fine at 10–20/day.

### D-003 · The LLM writes only the subject and the custom line — 2026-09-29
**Why:** Keeps Mark's voice, phone number, and legally required footer fixed and consistent.
**Consequences:** Template lives in code (`templates.py`); validator enforces word limits and footer presence.

### D-004 · Custom lines may reference only recorded research observations — 2026-09-29
**Why:** Invented "I noticed…" claims break trust instantly and undermine the personal-outreach premise.
**Consequences:** Research node produces a structured `observations` list; draft and validate nodes check against it.

### D-005 · Per-trade playbooks (YAML) own the offers — 2026-09-29
**Why:** Mark controls what he promises; the LLM matches and phrases.
**Consequences:** Adding a trade = adding a YAML file, no code change.

### D-006 · PostgreSQL + SQLAlchemy 2.0 + Alembic, local via Postgres.app — 2026-09-29
**Why:** Industry-standard Python pairing; Mark prefers Postgres.
**Consequences:** Postgres is the source of truth for all lead status.

### D-007 · LangGraph Studio is the approval UI — 2026-09-29
**Why:** Mark's choice; it teaches the graph model directly by showing state, nodes, and interrupts.
**Consequences:** Graph runs under `langgraph dev`, which manages its own thread persistence, so graphs compile without a custom checkpointer. Long waits (follow-ups) live in Postgres + scheduled jobs, not sleeping threads. If approval in Studio gets tedious at volume, add a small local review page later (see D-011 placeholder).

### D-008 · Sourcing and dedupe are CLI jobs, not graph nodes — 2026-09-29
**Why:** They operate on hundreds of businesses in batch; the graph is per lead.
**Consequences:** Outreach graph starts at `research`. `firstflag source` and `firstflag queue` bridge DB and graph.

### D-009 · Send from firstflagusa.com directly — 2026-09-29
**Why:** Mark's choice; simpler, and volume is low and hand-approved.
**Consequences:** Replies and outreach share one domain reputation. Guardrails: SPF/DKIM/DMARC before first send; 2-week warm-up; cap 10/day rising to 20; pause on bounce > 3% or complaint signals; monitor Google Postmaster Tools.

### D-010 · Territory: 45-mile radius from Bedford, MA — 2026-09-29
**Why:** Includes southern NH, central MA, South Shore.
**Consequences:** Places sourcing uses a grid of overlapping circles, deduped by `place_id`. Email locality phrase scales with distance.

### D-011 · Reply handling is manual; system only detects and stops follow-ups — 2026-09-29
**Why:** Low expected volume; every lead gets Mark's personal attention.
**Consequences:** Reply watch polls Gmail, marks `replied`, notifies Mark, auto-suppresses clear opt-outs. `mark-replied` CLI covers Google Voice texts.

### D-012 · One follow-up only, ~5 business days after the first email — 2026-09-29
**Why:** Short and respectful fits the personal tone. Default; revisit after first 50 sends.

### D-013 · Build for learning: small visible steps, Mark chooses who runs each — 2026-10-26
**Why:** Mark is learning uv, Python, Postgres, LangGraph, LangSmith, and Studio, and wants to understand how they fit together, not just have working code.
**Consequences:** `ROADMAP.md` is split into small steps (A1, A2, …), each with a feature, a concept, and where to see it. Claude Code follows the step loop in `CLAUDE.md`: orient, ask whether Claude, Mark, or a split runs the step, build, show where to look in code/Studio/LangSmith/Postgres, then check and pause. Every step gets an entry in `docs/LEARNING_LOG.md` and its own commit. Slower build; much better understanding.

### D-014 · Toy graphs first, then a walking skeleton — 2026-10-26
**Why:** Learn each LangGraph concept (state, nodes, edges, loops, interrupts, LLM calls) in isolation before it's tangled with real APIs. Then wire the full outreach graph with stub nodes so the whole flow is visible in Studio early.
**Consequences:** Playground graphs in `graphs/playground/` stay as reference. Real work replaces one `# STUB` node at a time, so each change shows up against the complete picture. The roadmap reorders accordingly: environment → LangGraph fundamentals → skeleton → database → real nodes.

---

## Open items

- [ ] Sending address is mark@firstflagusa.com; confirm it's a Google Workspace mailbox (needed for the Gmail API) and start the warm-up
- [ ] Mailing address for the footer (PO box or virtual mailbox)
- [ ] Loonen's permission to be named; ask for a 2-sentence quote
- [ ] Write first three playbooks (HVAC, plumbing, nail salons)
- [ ] Pricing anchor for conversations (not needed in emails)
- [ ] Website v1 live before first send
- [ ] Review Google Maps Platform terms for caching Places data
- [ ] Quick check with a lawyer on CAN-SPAM compliance of the template
