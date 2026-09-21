# Job Match Agent: LangGraph + MCP + Jev (structured decisions)

Date: 2026-09-22
Status: design agreed, pending review
Supersedes: [2026-09-13-lang-graph-demo-agent-design.md](2026-09-13-lang-graph-demo-agent-design.md)
(the "procurement" domain from the old spec was never implemented — the repo
had no code, only the spec itself; the project's direction changed to
job-matching)

## Goal

A personal tool: the agent reads several of the user's resumes (different
profiles — e.g. Fullstack and Backend), searches for vacancies across
multiple sources, scores how well each vacancy fits each resume via Jev, a
structured-decision model (`typesafe/jev-1.13`, OpenRouter), and shows a
ranked list in a UI without repeating vacancies already seen.

Out of scope for this spec (Phase 1): automatically submitting applications.
Phase 2 — an agent that fills in application forms on company sites itself
(not just LinkedIn Easy Apply) and indexes career pages — is deliberately
carved out: it's a separate, riskier subsystem (a real, irreversible action
on third-party sites) that deserves its own brainstorming session when its
turn comes. Phase 1's architecture is designed so the move to Phase 2 won't
require rework (see "Phase 2 compatibility").

## Decisions

| Decision | Choice | Why |
|---|---|---|
| Domain | Search and rank vacancies against the user's resumes | Replaces the unused "procurement" domain from the old spec |
| Matching | Jev (`typesafe/jev-1.13` on OpenRouter, System One API) | Structured decision model — a typed `Score` instead of free text, cheap input, free output |
| Resume parsing | A model the user picks from the OpenRouter list (the "Settings" screen) | Direct Anthropic API access (`ANTHROPIC_API_KEY`) is not allowed; everything goes through the already-paid `OPEN_ROUTER_KEY`, and the model choice belongs to the user, not hardcoded |
| Infrastructure stack | LangGraph + MCP + Jev client only | Temporal/pgvector/Langfuse/OTel from the old spec aren't needed for this task — dropped to avoid unused complexity |
| UI | Streamlit | A fast internal Python tool, no separate frontend |
| Job sources (Phase 1) | Modular adapters (`sources/`): AllJobs, Drushim. LinkedIn is deliberately excluded from Phase 1 | Starting with public Israeli portals that don't need authentication — simpler and lower ToS risk for the first pass; LinkedIn (needs a saved session) is added later via the same modular scheme, no graph changes |
| Source access | Playwright + LLM extraction (`web` role), no saved session — public search, no login needed | Neither AllJobs nor Drushim has an open API for job search; the user knowingly accepts the ToS risk for personal, non-commercial, low-volume use |
| Dedup storage | A JSON file (`seen_vacancies.json`) | Personal-scale data, no DB server needed |
| Resumes | Several named resumes, stored locally, parsing is cached | The user targets different roles (Fullstack/Backend, etc.) |
| Environment | venv via `uv`, Python 3.12, `pyproject.toml` | Same pattern as the old spec — it works, no reason to change it |

## Scenario

The user uploads several resumes to the UI once, each under its own name
("Fullstack", "Backend"). On the search screen they set a job title,
location, and enabled sources, then click "Search". The agent: parses (or
reads from cache) every active resume → pulls vacancies from the enabled
sources → for every vacancy not yet in the store, runs Jev against each
resume and keeps the best fit-score → filters out vacancies already seen →
shows a sorted table noting which resume produced the best match. The user
manually marks a vacancy as "applied" after actually applying — it no longer
shows up as new on the next search.

## Architecture

```
Streamlit UI (upload resumes / search / results table / mark-applied)
                      │
                      ▼
        ┌─────────────────────────────────────────┐
        │  LangGraph                                │
        │  load_resumes → search_jobs → match →     │
        │  filter_seen → rank → report              │
        └───┬──────────────┬───────────────┬────────┘
            │              │               │
            ▼              ▼               ▼
      resumes/ (cache)  sources/*        jev/client.py
      llm/client.py      (alljobs.py,     HTTP → OpenRouter
      (model from          drushim.py:     System One API
       "Settings")          Playwright +
                            LLM extraction,
                            no saved session;
                            linkedin.py later,
                            via MCP + saved
                            session)
                              │
                              ▼
                    seen_vacancies.json (dedup, lifecycle status)
```

Both external model calls — `llm/client.py` (resume parsing) and
`jev/client.py` (matching) — go through the same `OPEN_ROUTER_KEY`; there is
no separate `ANTHROPIC_API_KEY` in this project and there won't be one.

### Graph nodes

Nodes are pure functions `(State) -> dict`, same as in the original design:
they know nothing about Streamlit and are easy to test with a stubbed
LLM/Jev/sources.

- **`load_resumes`** — reads the list of active resumes from `resumes/`; for
  each one, either takes the cached structured profile (skills, years of
  experience, seniority, domains, location/format preferences), or, if the
  file changed, calls the model selected on the "Settings" screen again and
  updates the cache (the cache stores the model id alongside the profile —
  changing the model in Settings invalidates the cache).
- **`search_jobs`** — iterates over the enabled sources in `sources/`, calls
  `search(query, location, limit)` on each, and merges the results into one
  `VacancyRaw` list (a common shape regardless of source).
- **`match`** — for every vacancy not already in `seen_vacancies.json`: one
  Jev call per active resume with four `Score` questions (skills/stack fit,
  seniority level, domain/industry, location/work format), aggregated into an
  overall fit-score; keeps the maximum across resumes and records which
  resume produced the best result (`matched_resume`).
- **`filter_seen`** — drops vacancies already present in the store.
- **`rank`** — sorts by fit-score.
- **`report`** — returns the result to the UI and appends new vacancies to
  `seen_vacancies.json` with status `matched`.

### Job sources — a modular interface

```python
class JobSource(Protocol):
    def search(self, query: str, location: str, limit: int) -> list[VacancyRaw]: ...
```

Phase 1 ships two implementations, both plain Python modules calling
headless Playwright directly (no MCP wrapper — see below for why), plus LLM
extraction (`web` role) to turn raw HTML into vacancy structure instead of
hardcoded CSS selectors (this keeps the source working through minor site
layout changes):

- **`sources/alljobs.py`**, **`sources/drushim.py`** — public search, no
  saved session: neither portal requires login to browse vacancies.
  **Assumption, not verified live at spec-writing time** — if search does
  turn out to require authentication in practice, the same `storage_state`
  mechanism used for LinkedIn (below) gets wired in.

**LinkedIn is deliberately out of Phase 1.** It will be added later as
`sources/linkedin.py`, following the same `JobSource` interface but,
unlike AllJobs/Drushim, wrapped as an MCP tool with `storage_state` (cookies)
for a saved browser session stored on disk outside the repo (not committed;
the path lives in `.env`) — LinkedIn requires authentication for job search,
which is why it needs the isolated, stateful MCP process rather than a plain
module call. The architecture (the `JobSource` protocol, the graph) already
accounts for this — adding it later won't require changes to
`agent/graph.py`.

Further sources are added as new modules in `sources/` without touching
`agent/graph.py` — the graph works off the list of sources from config, not
hardcoded names.

### LLM client — two roles, both chosen in "Settings"

`llm/client.py` is a plain HTTP client (`httpx`) over
`POST https://openrouter.ai/api/v1/chat/completions` with `Authorization:
Bearer {OPEN_ROUTER_KEY}`. No model is hardcoded for either role: the model
id is read from `settings.json`, written there by the "Settings" screen. The
client is parameterized by role, not just by model — there are two roles:

- **`resume`** — parses a resume into a structured profile (`load_resumes`).
- **`web`** — extracts structure from raw web pages: right now, parsing
  vacancies in `sources/alljobs.py` and `sources/drushim.py` (and, later,
  `sources/linkedin.py`) before they're written to the store; in Phase 2 the
  same role gets reused to read the application form on a company's site
  (not implemented now, but the role is already shared so a third one isn't
  needed later).

Each role has its own independent model choice in "Settings" (two
dropdowns) — the user deliberately picks a "smart and cheap" model for
`web` (higher token volume there — whole pages) and whatever model they
like for `resume`. For tests, a role can point at any model — the concrete
choice isn't baked into the code; it's made by the user in the UI, not by
the developer in the spec.

The model lists for the dropdowns come from `GET
https://openrouter.ai/api/v1/models`, filtered down to models that support
plain chat-completions with text input/output (embedding models,
image-only models, and decision models like Jev itself are excluded — Jev
needs its own endpoint and doesn't belong in a general chat-model list).
The list is cached for the Streamlit session so `/models` isn't hit on
every screen re-render.

### Jev client

Not an MCP tool — a plain HTTP client (`jev/client.py`, `httpx`) over
`POST https://openrouter.ai/api/v1/systemone`, called directly from the
`match` node, the same way the LLM client is called from `load_resumes`.
Reason not to wrap it in MCP: it's just another call to an external model,
like the LLM client call, not a tool with a side effect.

**Important (not verified live at spec-writing time):** the `Score`
question type is documented by name only, with no response examples.
Before implementing the `match` node — manually check a real API response
(curl/httpx script) against a couple of examples to pin down the actual
`answers.score` format. If it doesn't match the expected shape (a 0–10
number, as assumed), the aggregation logic gets adjusted accordingly.

### Dedup storage

`seen_vacancies.json` — a list of records:

```json
{
  "source": "alljobs",
  "external_id": "...",
  "url": "...",
  "title": "...",
  "company": "...",
  "fit_score": 0.82,
  "matched_resume": "Backend",
  "status": "matched",
  "seen_at": "2026-09-22T10:00:00Z"
}
```

The dedup key is `(source, external_id)`, falling back to a normalized
`url` if a source has no id of its own. `status` is an open enum: `matched`
(shown to the user) is the only value written now; `applied` is written
manually through a separate UI action ("Mark applied"), never
automatically.

### Phase 2 compatibility (not implemented now)

A future apply-agent (filling in forms on company sites, not just Easy
Apply, plus indexing career pages) will be able to reuse: the vacancy's
canonical key `(source, external_id)`, the `matched_resume` field (which
resume to attach), the open status-lifecycle in the store (an `applied`
state will appear, and possibly intermediate `applying`/`failed` ones), and
the `web` role in `llm/client.py` — the same "smart and cheap" model choice
that reads vacancy pages today will read and fill out application forms.
None of this is implemented in Phase 1 — it's only reserved in the schema
and in the LLM client's interface, so no data migration or role rework is
needed later.

## UI (Streamlit)

- **Resumes screen**: list of uploaded resumes (name, date, cache status),
  upload a new file under a name, delete.
- **Search screen**: job title, location, source checkboxes, a "Search"
  button — runs the graph synchronously (`st.spinner`), result: a table
  (`st.dataframe`) with columns: vacancy, company, fit-score, matched
  resume, link, a "Mark applied" button on each row.
- **Settings screen**: two independent dropdowns listing OpenRouter models
  (see "LLM client — two roles") — one for the `resume` role, one for the
  `web` role. The choice is saved to `settings.json`; changing the `resume`
  model invalidates the cache of parsed resumes.

There's no separate CLI — the Streamlit app (`streamlit run`) is the only
entry point for the user.

## Components and repository layout

```
├── pyproject.toml
├── .env.example        # OPEN_ROUTER_KEY, LINKEDIN_SESSION_PATH
├── Makefile             # make ui / make test / make lint — a thin wrapper
├── resumes/             # local resume files + parse cache; in .gitignore
├── settings.json         # {"resume_model": "...", "web_model": "..."}; in .gitignore
├── src/job_match_agent/
│   ├── config.py         # pydantic-settings, the single place environment is read
│   ├── llm/               # client.py (chat completions via OpenRouter), models.py (model list)
│   ├── resume/             # parse.py, cache.py
│   ├── sources/             # base.py (Protocol), alljobs.py, drushim.py (linkedin.py — later)
│   ├── jev/                  # client.py — System One API client
│   ├── agent/                  # state.py, nodes.py, graph.py
│   ├── store/                    # seen_vacancies.py, settings.py — JSON read/write
│   └── ui/
│       └── app.py                  # Streamlit entrypoint (screens: resumes, search, settings)
└── tests/
```

Boundaries are strict, same as in the original design: `resume/`,
`sources/`, `jev/`, `llm/`, `store/` know nothing about LangGraph;
`agent/nodes.py` knows nothing about Streamlit.

## Error handling

- Sources (AllJobs/Drushim): a scraping failure (layout changed, captcha,
  rate limit) — the node logs it and returns an empty list for that source,
  without failing the whole run (other sources and resumes are still
  processed).
- Jev: retry once on a network error; on a persistent failure the vacancy is
  marked `fit_score: null` and stays in the list (visible as unscored,
  rather than silently disappearing).
- An unreadable resume file — an explicit error in the UI on upload; we
  don't proceed further.

## Testing

- **Unit**: `resume/parse.py` with fixture text and a stubbed LLM client
  (`resume` role); LLM extraction of a vacancy from raw HTML inside each
  source (`alljobs.py`, `drushim.py`) — with fixture HTML and a stubbed LLM
  client (`web` role), separate from the Playwright scraping itself;
  `agent/nodes.py::match` with a stubbed Jev client (the API is obscure and
  hasn't been verified live at writing time — tests don't rely on a real
  response); dedup/lifecycle logic in `store/seen_vacancies.py` — pure
  functions, tested without the network.
- **Manual/integration**: the actual Playwright scraping for AllJobs/Drushim
  — not run in CI; page-structure freshness is checked manually as needed.
- TDD: a test for the node's behavior is written before the implementation.

## Risks

| Risk | Response |
|---|---|
| AllJobs/Drushim ToS likely prohibit automated data collection (the exact wording for these two hasn't been checked at spec-writing time) | The user knowingly accepts the risk for personal, low-volume use |
| The Jev `Score` API hasn't been verified live, response format is uncertain | Before implementing `match` — manually check a real API response; Jev is stubbed at the client boundary in tests |
| AllJobs/Drushim layout fragility on site updates | `search_jobs` doesn't fail entirely if one source breaks — vacancies simply aren't found from it that run; LLM extraction (`web` role) is more resilient to small changes than CSS selectors, but not immune |
| N resumes × M vacancies = N×M Jev calls per run | Jev's input is cheap ($0.042/M tokens), output is free; at personal scale (a handful of resumes, tens of vacancies per run) this isn't a concern |
| The resume cache goes stale unnoticed | Invalidated by file content hash and by model id — not by date, so both editing a resume and changing the model in "Settings" always re-parse |
| The user hasn't picked a model (`resume`/`web`) yet, or `/models` is unreachable | The "Settings" screen is a mandatory first step before parsing resumes or searching for jobs; an explicit, role-specific UI error if a model isn't chosen, instead of a silent fallback to something hardcoded |

## Stages

| # | Stage | Check |
|---|---|---|
| 0 | Skeleton, venv, config, Streamlit "hello world" | `make ui` opens an empty screen |
| 1 | "Settings" screen + OpenRouter model list | Both dropdowns (`resume`, `web`) show real models, the choice is saved to `settings.json` |
| 2 | Resume parsing + cache, multiple profiles | The resumes screen shows a structured profile for an uploaded file |
| 3 | AllJobs + Drushim sources (Playwright + LLM extraction) | Vacancies can be pulled from each source manually, independent of the agent |
| 4 | Jev client + matching logic | On fixtures (a stubbed Jev), a correct aggregated fit-score is computed |
| 5 | The full graph, wired into the UI | A search in Streamlit runs end-to-end, the table fills in |
| 6 | Dedup + Mark applied | A repeat search doesn't show vacancies already seen |

## Legal/ToS note

Automated scraping of AllJobs and Drushim likely violates their terms of
use (a typical clause for job portals — the exact ToS text for these two
sites hasn't been checked at spec-writing time). The decision was made
knowingly by the user, for personal, non-commercial use at low request
volume. Adding LinkedIn later (outside Phase 1) carries a similar but
higher risk, because it requires a personal session — scraping through
Playwright with that session directly violates the LinkedIn User Agreement.
