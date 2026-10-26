# First Flag Outreach Agent — Spec

**Owner:** Mark McInnis, First Flag Web Development (firstflagusa.com)
**Status:** Draft v1.1 — 2026-10-26 (build approach: D-013, D-014)
**Repo:** github.com/617mark/first-flag-marketing
**Local path:** `/Users/markmcinnis/Develop/first-flag-marketing`
**Sending address:** mark@firstflagusa.com

This spec describes the finished system. It is built incrementally for learning: see `ROADMAP.md` for the step order and `CLAUDE.md` for the step loop.

## 1. Purpose

Find local small businesses that handle a lot of customer inquiries and appointment scheduling, research each one, and draft a short, personal, honest outreach email from Mark. Mark reviews and approves every message before it goes out. Leads that reply go straight to Mark; the system never negotiates or responds on his behalf.

**Non-goals (v1):** SMS outreach, automated reply handling, CRM integration, multi-user support, cloud deployment.

## 2. Positioning

- Headline: *I help local businesses stop losing jobs to voicemail.*
- Subhead: *AI tools that answer, follow up, and book, so no customer slips through.*
- Proof: Loonen — AI system that reads incoming water-lab reports and automatically flags results exceeding safety limits (name/quote pending Loonen's permission).

## 3. System overview

Four pieces, each small:

| Piece | Type | Trigger | Job |
|---|---|---|---|
| Sourcing | CLI command (plain Python) | Manual, per trade | Pull businesses from Google Places across the territory grid, upsert into Postgres, apply suppression |
| Outreach graph | LangGraph, one thread per lead | Queued via CLI | Research → score → draft → validate → **Mark approves** → send |
| Follow-up job | CLI command + small LangGraph | Scheduled (daily) | Find sent leads past follow-up date with no reply; draft one bump; Mark approves; send |
| Reply watch | CLI command (plain Python) | Scheduled (every ~15 min) | Poll Gmail for replies in outreach threads; mark lead `replied`; stop follow-ups; notify Mark |

Postgres is the single source of truth for businesses, leads, messages, and suppression. LangGraph threads are short-lived working state.

## 4. Outreach graph

### 4.1 Flow

```
START → research → score ──(score < threshold)──→ park → END
                      │
                      └──(score ≥ threshold)──→ draft → validate ──(fail, retries left)──→ draft
                                                             │
                                                             └──(pass)──→ approve ⏸
                                                                            │
                                            ┌───────────────┬───────────────┼───────────────┐
                                         approve          edit            revise          reject
                                            │               │               │               │
                                          send ←──────────send           draft →…        reject → END
                                            │
                                           END
```

### 4.2 State

One lead per thread. State holds only what the nodes pass to each other; durable records live in Postgres.

```python
from typing import Literal, NotRequired, TypedDict

class Draft(TypedDict):
    subject: str
    custom_line: str        # the only LLM-written text in the body
    body: str               # full assembled email (template + custom_line)

class Review(TypedDict):
    action: Literal["approve", "edit", "revise", "reject"]
    body: NotRequired[str]  # for "edit": Mark's final text
    notes: NotRequired[str] # for "revise": guidance for a redraft

class OutreachState(TypedDict):
    lead_id: int
    research: NotRequired[dict]         # structured findings (see 4.3)
    score: NotRequired[int]             # 1–10
    score_reason: NotRequired[str]
    draft: NotRequired[Draft]
    validation_errors: NotRequired[list[str]]
    draft_attempts: NotRequired[int]
    review: NotRequired[Review]
```

### 4.3 Nodes

**research** (LLM + code)
- Load business from Postgres. Fetch homepage plus up to 3 likely pages (contact, booking, services) with `httpx`; extract text.
- LLM with structured output returns:
  - `booking_method`: phone_only | online_tool | form | unknown (+ tool name if visible)
  - `hours`, `offers_emergency_or_after_hours`
  - `emails_found` (from page text and `mailto:` links, verified by code, not LLM)
  - `owner_or_contact_name` (only if stated on the site)
  - `observations`: list of short, factual, quotable facts about the business
- Writes research JSON to the lead record.

**score** (rules + LLM)
- Hard rules (code): no email found → park; franchise/chain → park; already suppressed → park.
- LLM rates fit 1–10 with a one-line reason, using playbook "fit signals."
- Threshold lives in config (start at 7).

**park** (code) — set lead status `parked` with reason. End.

**draft** (LLM)
- Inputs: the trade's playbook, research observations, distance tier, Mark's revise notes (if any).
- Output: `subject` and `custom_line` only (1–2 sentences).
- Rule: the custom line may reference only facts in `research.observations`. No invented "I noticed…".
- Code assembles the full body from the fixed template (section 6).

**validate** (code)
- Body ≤ 90 words; custom line ≤ 45 words.
- Footer present verbatim (name, business, site, mailing address, opt-out line).
- No banned phrases (config list: "revolutionize," "leverage," "cutting-edge," exclamation points, etc.).
- Every quoted or specific claim traceable to an observation (LLM-assisted check allowed).
- On failure: return to draft with errors, max 2 retries; then pass to approve with errors shown.

**approve** (human interrupt)
- Calls `interrupt()` with the lead summary, research highlights, score reason, draft, and any validation warnings.
- Mark resumes in LangGraph Studio with a `Review` payload:
  - `{"action": "approve"}`
  - `{"action": "edit", "body": "…final text…"}`
  - `{"action": "revise", "notes": "mention their emergency line"}`
  - `{"action": "reject"}`

**send** (code)
- Re-check suppression and daily send cap immediately before sending.
- Send via Gmail API from mark@firstflagusa.com; store `gmail_message_id`, `gmail_thread_id`, `sent_at`, `followup_due_at`.
- If outside send window or over cap, set status `approved_queued`; a scheduled job sends it later.

**reject** (code) — set status `rejected`. End.

### 4.4 Edges

| From | Type | Logic |
|---|---|---|
| START → research | fixed | |
| research → score | fixed | |
| score → draft \| park | conditional | `score >= SCORE_THRESHOLD` and hard rules passed |
| draft → validate | fixed | |
| validate → draft \| approve | conditional | errors and `draft_attempts < 3` → draft |
| approve → send \| draft \| reject | conditional | on `review.action` (approve/edit → send) |
| send, park, reject → END | fixed | |

## 5. Playbooks

One YAML file per trade in `playbooks/`. Mark owns these; the LLM only chooses among and phrases them.

```yaml
trade: hvac
places_queries: ["hvac contractor", "heating and air conditioning"]
fit_signals:
  - phone-only booking
  - emergency or after-hours service advertised
  - single location, owner-operated
pains:
  - id: after_hours
    when: offers_emergency_or_after_hours or booking_method == phone_only
    offer: answer after-hours calls by text and get them booked before they call the next company
  - id: missed_calls
    when: booking_method == phone_only
    offer: text back missed calls within a minute so jobs don't go to whoever picks up first
```

Starting trades: HVAC, plumbing, electrical, nail salons, barbershops/hair salons, auto repair, pet grooming, landscaping/snow removal.

## 6. Email template

```
Subject: {subject}

Hi {name_or_there},

I'm Mark, a web developer {locality_phrase}. {custom_line}

I build tools like this for local businesses. Free estimate, references available, and I don't get paid until it's actually useful to you.

Reply here or text me at (617) 575-9799.

Mark McInnis
First Flag Web Development · firstflagusa.com
{mailing_address}
Not a fit? Just say so and I won't reach out again.
```

`locality_phrase` by distance from Bedford:

| Distance | Phrase |
|---|---|
| < 10 mi | "here in Bedford" |
| 10–25 mi | "a few towns over in Bedford" |
| > 25 mi or out of state | "based in Bedford, MA" |

**Follow-up (one only, ~5 business days later, same Gmail thread):**
```
Hi {name_or_there}, just bumping this in case it got buried. Happy to take a quick look at how you're handling calls and bookings, no charge. — Mark
```
(Footer repeated.)

## 7. Data model (Postgres)

**businesses** — `id`, `place_id` (unique), `name`, `trade`, `address`, `town`, `state`, `lat`, `lng`, `phone`, `website`, `distance_mi`, `places_fetched_at`, `created_at`

**leads** — `id`, `business_id` (unique), `status`, `score`, `score_reason`, `research` (JSONB), `park_reason`, `langgraph_thread_id`, `contact_email`, `contact_name`, `created_at`, `updated_at`

Lead status: `new` → `queued` → `awaiting_approval` → `approved_queued` | `sent` → `followed_up` → `replied` | `closed`; also `parked`, `rejected`, `opted_out`, `bounced`.

**messages** — `id`, `lead_id`, `kind` (initial | followup), `subject`, `body`, `gmail_message_id`, `gmail_thread_id`, `sent_at`

**suppression** — `id`, `email` or `domain`, `reason` (opt_out | bounce | manual | existing_client), `created_at`. Checked at sourcing, queueing, and send.

**send_log** — daily counts for cap enforcement (or derive from `messages`).

## 8. Sending rules

- From: mark@firstflagusa.com via Gmail API (Google Workspace).
- DNS: SPF, DKIM, DMARC (start `p=none`, tighten later) before the first send.
- Warm-up: 2 weeks of normal mail activity before outreach; then cap at 10/day, rising to 20/day if bounces and complaints stay low.
- Send window: Tue–Thu, 8–11 a.m. Eastern, prospect's local time.
- Stop conditions: bounce rate > 3% in a week, any spam-complaint signal, or Google Postmaster Tools reputation drop → pause and review.
- Opt-outs are processed within 24 hours (CAN-SPAM requires within 10 business days).

## 9. Reply watch

- Poll Gmail for new messages in any `gmail_thread_id` on file.
- On reply: status `replied`, cancel follow-up, notify Mark (email to his main inbox).
- Opt-out phrases ("remove," "not interested," "stop"): add to suppression, status `opted_out`. Mark can override.
- Text replies to Google Voice don't reach the system: `firstflag mark-replied <lead or email>` handles those.

## 10. Territory and sourcing

- Center: Bedford, MA. Radius: 45 miles (includes southern NH, central MA, South Shore).
- Grid of overlapping search circles across the radius, one Places query per trade per circle; dedupe by `place_id`.
- Filter: distance ≤ 45 mi, has website, not permanently closed.
- Store `place_id` long term; refresh other Places fields per Google Maps Platform caching terms.

## 11. Observability

- LangSmith tracing on every graph run (project `first-flag-outreach`).
- Tag runs with `trade` and `lead_id`.
- Later: LangSmith dataset of approved vs. rejected drafts, plus an LLM-judge evaluator for tone and grounding.

## 12. Tech stack

| Concern | Choice |
|---|---|
| Language | Python 3.12 |
| Package manager | uv |
| Agent framework | LangGraph, LangChain (Anthropic model integration) |
| Dev server + UI | LangGraph CLI (`langgraph dev`) + LangGraph Studio |
| Tracing/evals | LangSmith |
| DB | PostgreSQL (Postgres.app locally) |
| ORM + migrations | SQLAlchemy 2.0 + Alembic |
| HTTP + parsing | httpx, BeautifulSoup/trafilatura |
| Config | pydantic-settings + `.env` |
| CLI | Typer |
| Email | Gmail API (google-api-python-client) |
| Places | Google Places API (New) |
| Tests | pytest |

## 13. Repository layout

```
first-flag-marketing/
├── pyproject.toml            # dependencies + metadata (Python's package.json)
├── uv.lock                   # lockfile (package-lock.json)
├── .env.example              # required env vars, no secrets
├── langgraph.json            # tells `langgraph dev` / Studio where the graphs live
├── alembic.ini
├── CLAUDE.md                 # conventions for Claude Code
├── docs/
│   ├── SPEC.md
│   ├── DECISIONS.md
│   ├── ROADMAP.md
│   └── LEARNING_LOG.md       # one entry per completed step; where we left off
├── playbooks/                # one YAML per trade — Mark edits these
│   ├── hvac.yaml
│   └── plumbing.yaml
├── migrations/               # Alembic migration scripts
├── src/
│   └── firstflag/            # the importable package
│       ├── __init__.py       # marks the folder as a package
│       ├── config.py         # settings loaded from env
│       ├── cli.py            # `firstflag source|queue|followups|watch|mark-replied`
│       ├── db/
│       │   ├── models.py     # SQLAlchemy tables
│       │   ├── session.py    # engine + session factory
│       │   └── repo.py       # query helpers (get_lead, set_status, …)
│       ├── services/         # talks to the outside world, no LLM
│       │   ├── places.py
│       │   ├── web.py        # fetch + extract site text
│       │   └── gmail.py
│       ├── playbooks.py      # load + validate YAML
│       ├── templates.py      # email assembly + locality phrase
│       └── graphs/
│           ├── playground/   # toy graphs from roadmap stage B, kept as reference
│           ├── outreach/
│           │   ├── state.py  # OutreachState, Draft, Review
│           │   ├── nodes.py  # research, score, draft, validate, approve, send…
│           │   ├── edges.py  # routing functions for conditional edges
│           │   ├── prompts.py
│           │   └── graph.py  # builds and compiles the StateGraph → `graph`
│           └── followup/
│               ├── state.py
│               ├── nodes.py
│               └── graph.py
└── tests/
    ├── test_templates.py
    ├── test_validate.py
    └── test_edges.py
```

**How the pieces connect:** `state.py` defines the shape of the data; `nodes.py` holds functions that take state and return partial updates; `edges.py` holds functions that read state and return the next node's name; `graph.py` wires them together and exposes a compiled `graph`. `langgraph.json` points at that `graph` so Studio can load it.

```python
# graph.py (sketch)
from langgraph.graph import StateGraph, START, END
from .state import OutreachState
from . import nodes, edges

builder = StateGraph(OutreachState)
builder.add_node("research", nodes.research)
builder.add_node("score", nodes.score)
builder.add_node("park", nodes.park)
builder.add_node("draft", nodes.draft)
builder.add_node("validate", nodes.validate)
builder.add_node("approve", nodes.approve)
builder.add_node("send", nodes.send)
builder.add_node("reject", nodes.reject)

builder.add_edge(START, "research")
builder.add_edge("research", "score")
builder.add_conditional_edges("score", edges.after_score)       # → "draft" | "park"
builder.add_edge("draft", "validate")
builder.add_conditional_edges("validate", edges.after_validate) # → "draft" | "approve"
builder.add_conditional_edges("approve", edges.after_review)    # → "send" | "draft" | "reject"
for n in ("send", "park", "reject"):
    builder.add_edge(n, END)

graph = builder.compile()  # dev server supplies persistence; no checkpointer here
```

```python
# nodes.py (approve)
from langgraph.types import interrupt

def approve(state):
    review = interrupt({
        "lead_id": state["lead_id"],
        "draft": state["draft"],
        "score_reason": state.get("score_reason"),
        "warnings": state.get("validation_errors", []),
    })
    return {"review": review}
```

## 14. Runtime (local Mac)

- `langgraph dev` runs the graph server on localhost (port 2024) and opens Studio.
- `firstflag queue --trade hvac --limit 20` creates one thread per lead against the local server via the LangGraph SDK; each thread runs until it pauses at `approve`.
- Mark opens Studio, reviews paused threads, resumes each with a `Review` payload.
- `launchd` (or cron) runs `firstflag watch` every 15 minutes and `firstflag followups` plus queued sends each weekday morning.
