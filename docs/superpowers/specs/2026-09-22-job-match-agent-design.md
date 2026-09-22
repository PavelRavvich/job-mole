# JobMole: LangGraph + JobsPipe + Jev (structured decisions)

Date: 2026-09-22
Status: design agreed, pending review

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
| Domain | Search and rank vacancies against the user's resumes | The project's actual purpose |
| Matching | Jev (`typesafe/jev-1.13` on OpenRouter, System One API) | Structured decision model — a typed `Score` instead of free text, cheap input, free output |
| Resume parsing | A model the user picks from the OpenRouter list (the "Settings" screen) | Direct Anthropic API access (`ANTHROPIC_API_KEY`) is not allowed; everything goes through the already-paid `OPEN_ROUTER_KEY`, and the model choice belongs to the user, not hardcoded |
| Infrastructure stack | LangGraph + Jev client only, no MCP | Keep it minimal — no Temporal/pgvector/Langfuse/OTel; MCP dropped too, since it was only in the stack to wrap browser-automation sources, and Phase 1 no longer scrapes anything (see below) |
| UI | Streamlit | A fast internal Python tool, no separate frontend |
| Job sources (Phase 1) | JobsPipe API (`jobspipe.dev`) — one aggregator over 30+ ATS platforms and job boards (Greenhouse, Lever, Workday, Comeet, SmartRecruiters, Indeed, LinkedIn, Glassdoor, ZipRecruiter, YC, and more) | Verified live via its own docs (`docs.jobspipe.dev`) — a real, paid API with a clean REST contract, not scraping; also resolves the earlier LinkedIn-access problem for free, since LinkedIn postings arrive through JobsPipe's own aggregation rather than our own session-scraping |
| Source access | `POST https://api.jobspipe.dev/v1/jobs/search`, Bearer token (`JOBSPIPE_API_KEY`), filtered by `country_code` (IL) and `job_title_or` | Normalized JSON response, already includes `apply_url` per posting — no HTML parsing or LLM extraction needed for search; no ToS/scraping risk, since it's a paid API used as intended |
| Dedup storage | A JSON file (`seen_vacancies.json`) | Personal-scale data, no DB server needed |
| Resumes | Several named resumes, stored locally, parsing is cached | The user targets different roles (Fullstack/Backend, etc.) |
| Environment | venv via `uv`, Python 3.12, `pyproject.toml` | A standard, low-friction setup for a local Python tool |

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
      resumes/ (cache)  sources/          jev/client.py
      llm/client.py     jobspipe.py:       HTTP → OpenRouter
      (model from        HTTP → JobsPipe   System One API
       "Settings")        API, Bearer
                          token
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

Phase 1 ships one implementation, **`sources/jobspipe.py`** — a plain HTTP
client (`httpx`) over `POST https://api.jobspipe.dev/v1/jobs/search` with
`Authorization: Bearer {JOBSPIPE_API_KEY}`. Maps the UI's job-title chips to
`job_title_or`, restricts to `country_code: "IL"`, and reads `apply_url`
straight from the response (useful later — see "Phase 2 compatibility").
No browser automation, no LLM extraction, no ToS risk: it's a paid API used
through its documented contract.

The `JobSource` protocol stays in place for a reason beyond this one
implementation: JobsPipe's coverage is ATS platforms and major boards
(Greenhouse, Lever, Workday, Comeet, Indeed, LinkedIn, Glassdoor,
ZipRecruiter, YC) — it does **not** include Israeli portals like AllJobs or
Drushim (checked directly against its docs). If that coverage gap turns out
to matter in practice, a direct-scrape source for one of those can be added
later as a new `sources/*.py` module without touching `agent/graph.py` — the
graph works off the list of sources from config, not hardcoded names.

### LLM client — two roles, both chosen in "Settings"

`llm/client.py` is a plain HTTP client (`httpx`) over
`POST https://openrouter.ai/api/v1/chat/completions` with `Authorization:
Bearer {OPEN_ROUTER_KEY}`. No model is hardcoded for either role: the model
id is read from `settings.json`, written there by the "Settings" screen. The
client is parameterized by role, not just by model — there are two roles:

- **`resume`** — parses a resume into a structured profile (`load_resumes`).
- **`web`** — reserved for Phase 2: reading and filling out an application
  form on a company's site. **Not used anywhere in Phase 1** — now that
  vacancy search goes through the JobsPipe API (structured JSON, no HTML to
  parse), there's no Phase 1 caller for this role. It's kept in the client's
  interface and in the "Settings" screen now anyway, so Phase 2 doesn't need
  a UI change to introduce it later.

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

A plain HTTP client (`jev/client.py`, `httpx`) over
`POST https://openrouter.ai/api/v1/systemone`, called directly from the
`match` node, the same way the LLM client is called from `load_resumes` —
just another call to an external model, not a tool with a side effect.

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
Apply) will be able to reuse: the vacancy's canonical key
`(source, external_id)`, the `apply_url` JobsPipe already returns per
posting (no extra extraction step needed to find where to apply), the
`matched_resume` field (which resume to attach), the open status-lifecycle
in the store (an `applied` state will appear, and possibly intermediate
`applying`/`failed` ones), and the `web` role in `llm/client.py` — reserved
but unused in Phase 1 — which will read and fill out the application form
found at `apply_url`. None of this is implemented in Phase 1 — it's only
reserved in the schema and in the LLM client's interface, so no data
migration or role rework is needed later.

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
├── .env.example        # OPEN_ROUTER_KEY, JOBSPIPE_API_KEY
├── Makefile             # make ui / make test / make lint — a thin wrapper
├── resumes/             # local resume files + parse cache; in .gitignore
├── settings.json         # {"resume_model": "...", "web_model": "..."}; in .gitignore
├── seen_vacancies.json    # dedup store; in .gitignore
├── src/job_mole/
│   ├── config.py         # pydantic-settings, the single place environment is read
│   ├── llm/               # client.py (chat completions via OpenRouter), models.py (model list)
│   ├── resume/             # parse.py, cache.py
│   ├── sources/             # base.py (Protocol), jobspipe.py
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

### Local data and cold start

`.gitignore` covers everything that's personal or generated, none of it
ships in the repo: `resumes/`, `settings.json`, `seen_vacancies.json`, plus
`.env` itself (only `.env.example` is tracked).

A fresh clone must still run cleanly, with none of these files present:
`store/` and `resume/cache.py` create `resumes/`, `settings.json`, and
`seen_vacancies.json` on first access if missing (empty state — no
resumes, no saved model choice, no seen vacancies), rather than crashing
on a missing file or directory. The "Settings" screen already surfaces the
no-model-chosen case as a UI error (see "Risks"), so a cold start's only
visible effect is that both role dropdowns start unset.

## Error handling

- JobsPipe: an API error, rate limit, or an empty free-tier credit balance
  — the node logs it and returns an empty list, without failing the whole
  run (resumes and matching still proceed on whatever was already fetched).
- Jev: retry once on a network error; on a persistent failure the vacancy is
  marked `fit_score: null` and stays in the list (visible as unscored,
  rather than silently disappearing).
- An unreadable resume file — an explicit error in the UI on upload; we
  don't proceed further.

## Testing

- **Unit**: `resume/parse.py` with fixture text and a stubbed LLM client
  (`resume` role); `sources/jobspipe.py` with a stubbed HTTP response
  (fixture JSON matching JobsPipe's documented schema); `agent/nodes.py::match`
  with a stubbed Jev client (the API is obscure and hasn't been verified live
  at writing time — tests don't rely on a real response); dedup/lifecycle
  logic in `store/seen_vacancies.py` — pure functions, tested without the
  network.
- **Manual/integration**: one real call to JobsPipe against its free tier,
  to confirm the live response still matches the fixture shape used in unit
  tests — not run in CI, checked manually as needed.
- TDD: a test for the node's behavior is written before the implementation.

## Risks

| Risk | Response |
|---|---|
| JobsPipe doesn't cover AllJobs/Drushim and isn't guaranteed to surface every Israeli hi-tech vacancy — only what's published through the ATS platforms/boards it covers (though many Israeli startups do sit on Greenhouse/Lever/Comeet) | The `JobSource` protocol is already set up to add another source later if this coverage gap turns out to matter in practice |
| The JobsPipe API hasn't been verified live under a real key at spec-writing time — only against its docs | Manually check a real response (curl/httpx script) before implementing `sources/jobspipe.py`; the client is stubbed at the HTTP boundary in tests |
| The Jev `Score` API hasn't been verified live, response format is uncertain | Before implementing `match` — manually check a real API response; Jev is stubbed at the client boundary in tests |
| JobsPipe's free tier is capped (1,000 jobs/month) | Fine at personal scale (occasional runs, sane per-search limits); paid tiers start at $49/month if more is needed |
| N resumes × M vacancies = N×M Jev calls per run | Jev's input is cheap ($0.042/M tokens), output is free; at personal scale (a handful of resumes, tens of vacancies per run) this isn't a concern |
| The resume cache goes stale unnoticed | Invalidated by file content hash and by model id — not by date, so both editing a resume and changing the model in "Settings" always re-parse |
| The user hasn't picked a model (`resume`/`web`) yet, or `/models` is unreachable | The "Settings" screen is a mandatory first step before parsing resumes or searching for jobs; an explicit, role-specific UI error if a model isn't chosen, instead of a silent fallback to something hardcoded |

## Stages

| # | Stage | Check |
|---|---|---|
| 0 | Skeleton, venv, config, `.gitignore`, Streamlit "hello world" | `make ui` opens an empty screen on a fresh clone — no `resumes/`, `settings.json`, or `seen_vacancies.json` present yet, nothing crashes |
| 1 | "Settings" screen + OpenRouter model list | Both dropdowns (`resume`, `web`) show real models, the choice is saved to `settings.json` |
| 2 | Resume parsing + cache, multiple profiles | The resumes screen shows a structured profile for an uploaded file |
| 3 | JobsPipe source client | Vacancies can be pulled from `sources/jobspipe.py` manually, independent of the agent |
| 4 | Jev client + matching logic | On fixtures (a stubbed Jev), a correct aggregated fit-score is computed |
| 5 | The full graph, wired into the UI | A search in Streamlit runs end-to-end, the table fills in |
| 6 | Dedup + Mark applied | A repeat search doesn't show vacancies already seen |

## API usage note

Phase 1 has no scraping and no ToS risk of that kind anymore: JobsPipe is
used as a paying customer, through its documented API contract, not against
the terms of the sites it aggregates. The only remaining real-world-action
risk in this project is Phase 2's actual form submission on company sites —
already called out in "Phase 2 compatibility" and to be designed carefully
when that phase is brainstormed.
