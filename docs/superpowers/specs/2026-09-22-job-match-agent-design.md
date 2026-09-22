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
require rework (see "Phase 2 compatibility"). The full Pipeline/apply UI has
already been mocked up end to end (see "UI (Streamlit)" and "Phase 1 vs
Phase 2 UI scope") — Phase 1 ships that UI, but the controls that would
actually submit an application stay inert until Phase 2 is built.

## Decisions

| Decision | Choice | Why |
|---|---|---|
| Domain | Search and rank vacancies against the user's resumes | The project's actual purpose |
| Matching | Jev (`typesafe/jev-1.13` on OpenRouter, System One API), model chosen in the Models tab ("Classification"), scoped to `typesafe/*` only | Structured decision model — a typed `Score` instead of free text, cheap input, free output; user-selectable like the other two roles, not hardcoded, but restricted to the `typesafe/*` namespace since Jev needs the System One endpoint, not the general chat-model list |
| Resume parsing | A model the user picks from the OpenRouter list, in the Models tab ("Generation (resumes)") | Direct Anthropic API access (`ANTHROPIC_API_KEY`) is not allowed; everything goes through the already-paid `OPEN_ROUTER_KEY`, and the model choice belongs to the user, not hardcoded |
| Infrastructure stack | LangGraph + Jev client only, no MCP | Keep it minimal — no Temporal/pgvector/Langfuse/OTel; MCP dropped too, since it was only in the stack to wrap browser-automation sources, and Phase 1 no longer scrapes anything (see below) |
| UI | Streamlit | A fast internal Python tool, no separate frontend |
| Job sources (Phase 1) | JobsPipe API (`jobspipe.dev`) — one aggregator over 30+ ATS platforms and job boards (Greenhouse, Lever, Workday, Comeet, SmartRecruiters, Indeed, LinkedIn, Glassdoor, ZipRecruiter, YC, and more) | Verified live via its own docs (`docs.jobspipe.dev`) — a real, paid API with a clean REST contract, not scraping; also resolves the earlier LinkedIn-access problem for free, since LinkedIn postings arrive through JobsPipe's own aggregation rather than our own session-scraping |
| Source access | `POST https://api.jobspipe.dev/v1/jobs/search`, Bearer token (`JOBS_PIPE_KEY`) | Normalized JSON response, already includes `apply_url` per posting — no HTML parsing or LLM extraction needed for search; no ToS/scraping risk, since it's a paid API used as intended |
| Search-tab filters → JobsPipe params | Job titles → `job_title_or`; Location → `city_or`; Seniority → `job_seniority_or` (`entry_level`/`mid_level`/`senior`/`director`/`executive`); Format → `work_arrangement_or` (`remote`/`hybrid`/`onsite`); always `job_country_code_or: ["IL"]` | Verified against JobsPipe's filter reference doc (`docs.jobspipe.dev/api-reference/filters`) — all four UI filters map to real, documented server-side filters, so filtering happens before the (paid) Jev calls, not after |
| Min score filter | Applied client-side, after Jev has scored the batch — not sent to JobsPipe | The score doesn't exist until Jev runs, so it can't be a search-time filter; it only hides already-fetched rows below the threshold |
| Filter presets ("Profile") | Named combinations of the above filters, saved or overwritten from a small popup on the Search tab, stored in `settings.json` | Repeat searches (e.g. "Backend, Israel") without re-entering the same filters every time |
| Provider credentials | Models tab (OpenRouter) and Integrations tab (JobsPipe) both offer a Direct value / Env var toggle, defaulting to Env var; Direct value writes the raw key into `settings.json` | One consistent key-entry pattern for both providers; env var stays the default so nothing sensitive is typed into the UI unless the user opts in; `settings.json` is already gitignored either way |
| Dedup storage | A JSON file (`seen_vacancies.json`) | Personal-scale data, no DB server needed |
| Resumes | Several named resumes, stored locally, parsing is cached | The user targets different roles (Fullstack/Backend, etc.) |
| Resume upload | Two explicit steps in the Resumes tab: "Choose file" only picks and names the file, a separate "Upload" button (disabled until a file is chosen) starts parsing | Prevents parsing from starting on a file the user hasn't finished naming/hasn't committed to yet |
| Resume file types | `.pdf`, `.doc`, `.docx`, `.md`, `.txt` | Covers the common resume formats, not just PDF |
| Environment | venv via `uv`, Python 3.12, `pyproject.toml` | A standard, low-friction setup for a local Python tool |

## Scenario

The user uploads several resumes to the UI once, each under its own name
("Fullstack", "Backend") — picking a file, naming it, then clicking
"Upload" starts parsing. On the Search tab they optionally load a saved
filter Profile (or start from scratch), set job titles (a tag list),
location, seniority, work format, and a minimum score, leave JobsPipe as
the (only, for now) enabled source, and click "Search". The agent: parses
(or reads from cache) every active resume → pulls vacancies from JobsPipe
using the mapped filters → for every vacancy not yet in the store, runs Jev
against each resume and keeps the best score → filters out vacancies
already seen → shows a list sorted by score, each row tagged with a
colored badge naming which resume produced the best match, and a link
(first ~30 characters of the URL, opens in a new tab). The user can narrow
the list further with the Min score field (client-side, no re-fetch), then
selects some vacancies (individually or via "Select all") and adds them to
the queue — either per-row or in bulk ("Add selected to queue"). Queued
vacancies show up in the Pipeline tab's "In queue" list; the user manually
marks one "Applied" once they've actually applied outside the app — it
moves to the Applied list and no longer shows as new on a repeat search.
Models (Classification/Generation/Browser control) and provider keys
(OpenRouter, JobsPipe) are one-time setup in the Models and Integrations
tabs; the Bio tab is optional free-form prep (main bio text + answers to
common application questions) that Phase 1 stores but nothing yet reads —
it's there ahead of Phase 2.

## Architecture

```
Streamlit UI (Search / Pipeline / Resumes / Integrations / Models / Bio)
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
       Models tab)        API, Bearer      (model from
                          token             Models tab)
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
  file changed, calls the model selected in the Models tab again and
  updates the cache (the cache stores the model id alongside the profile —
  changing the model invalidates the cache).
- **`search_jobs`** — calls JobsPipe's `sources/jobspipe.py` with the
  Search tab's filters mapped to JobsPipe's own field names (see
  "Search-tab filters → JobsPipe params" in Decisions), and normalizes the
  response into one `VacancyRaw` list. Still built behind the `JobSource`
  protocol so a second source can be added later without touching this
  node's shape.
- **`match`** — for every vacancy not already in `seen_vacancies.json`: one
  Jev call per active resume with four `Score` questions (skills/stack fit,
  seniority level, domain/industry, location/work format), aggregated into
  an overall score; keeps the maximum across resumes and records which
  resume produced the best result (`matched_resume`).
- **`filter_seen`** — drops vacancies already present in the store.
- **`rank`** — sorts by score.
- **`report`** — returns the result to the UI and appends new vacancies to
  `seen_vacancies.json` with status `matched`. The Min score field and the
  Search tab's "New results" list operate on this already-ranked result —
  no separate node for it.

### Job sources — a modular interface

```python
class JobSource(Protocol):
    def search(self, query: str, location: str, limit: int) -> list[VacancyRaw]: ...
```

Phase 1 ships one implementation, **`sources/jobspipe.py`** — a plain HTTP
client (`httpx`) over `POST https://api.jobspipe.dev/v1/jobs/search` with
`Authorization: Bearer {JOBS_PIPE_KEY}`. Maps the Search tab's fields to
JobsPipe's documented filters: job-title chips → `job_title_or`, Location →
`city_or`, Seniority → `job_seniority_or`, Format → `work_arrangement_or`,
always `job_country_code_or: ["IL"]`. Reads `apply_url` straight from the
response (useful later — see "Phase 2 compatibility"). No browser
automation, no LLM extraction, no ToS risk: it's a paid API used through
its documented contract.

The `JobSource` protocol stays in place for a reason beyond this one
implementation: JobsPipe's coverage is ATS platforms and major boards
(Greenhouse, Lever, Workday, Comeet, Indeed, LinkedIn, Glassdoor,
ZipRecruiter, YC) — it does **not** include Israeli portals like AllJobs or
Drushim (checked directly against its docs). If that coverage gap turns out
to matter in practice, a direct-scrape source for one of those can be added
later as a new `sources/*.py` module without touching `agent/graph.py` — the
graph works off the list of sources from config, not hardcoded names. The
Search tab's "Source" row already anticipates this: it shows JobsPipe as a
checked, disabled checkbox with a tooltip ("More sources can be added
later") rather than a hardcoded single value.

### LLM client — two chat roles, plus Jev's own model choice, all set in the Models tab

`llm/client.py` is a plain HTTP client (`httpx`) over
`POST https://openrouter.ai/api/v1/chat/completions` with `Authorization:
Bearer {OPEN_ROUTER_KEY}`. No model is hardcoded for either role: the model
id is read from `settings.json`, written there by the Models tab. The
client is parameterized by role, not just by model — there are two roles:

- **`resume`** — parses a resume into a structured profile (`load_resumes`).
  Shown in the Models tab as "Generation (resumes)".
- **`web`** — reserved for Phase 2: reading and filling out an application
  form on a company's site. **Not used anywhere in Phase 1** — now that
  vacancy search goes through the JobsPipe API (structured JSON, no HTML to
  parse), there's no Phase 1 caller for this role. It's kept in the client's
  interface and in the Models tab now anyway (shown as "Browser control
  (web)"), so Phase 2 doesn't need a UI change to introduce it later.

Each role has its own independent model choice in the Models tab (two
searchable dropdowns) — the user deliberately picks a "smart and cheap"
model for `web` (higher token volume there — whole pages) and whatever
model they like for `resume`. For tests, a role can point at any model —
the concrete choice isn't baked into the code; it's made by the user in the
UI, not by the developer in the spec.

The model lists for these two dropdowns come from `GET
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
just another call to an external model, not a tool with a side effect. The
model id ("Classification" in the Models tab) is also read from
`settings.json`, defaulting to `typesafe/jev-1.13`; its dropdown is scoped
to the `typesafe/*` namespace only (currently `typesafe/jev-1.13` and
`typesafe/jev-latest`) rather than the general OpenRouter model list, since
Jev is a separate decisions API, not a chat-completions model.

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
  "source": "jobspipe",
  "external_id": "...",
  "url": "...",
  "title": "...",
  "company": "...",
  "score": 0.82,
  "matched_resume": "Backend",
  "status": "matched",
  "seen_at": "2026-09-22T10:00:00Z"
}
```

The dedup key is `(source, external_id)`, falling back to a normalized
`url` if a source has no id of its own. `status` is an open enum, only
part of which is written in Phase 1:

- `matched` — written by `report` when a new vacancy is first seen.
- `queued` — written when the user adds a vacancy to the queue from the
  Search tab (per-row or "Add selected to queue"); shown in Pipeline →
  In queue.
- `applied` — written manually through a per-row action in Pipeline →
  In queue ("Mark applied"), never automatically; shown in Pipeline →
  Applied.
- `applying`, `failed` — **Phase 2 only**, reserved. Written by the future
  apply-agent while a batch submission is in flight or after it fails a
  blocking application question; not produced by anything in Phase 1 (see
  "Phase 1 vs Phase 2 UI scope").

### Phase 2 compatibility (not implemented now)

The Pipeline tab's full design — In queue / Applied / Failed sub-tabs, a
Manual-confirm / Auto mode toggle, a "Run batch" action, a
review-before-submitting popup, and, on Failed, inline per-question
answering with an "Add to Bio" action that appends the answer to the Bio
tab as-is (no LLM recompute at save time) — is already fully mocked up.
None of it is implemented in Phase 1; see "Phase 1 vs Phase 2 UI scope"
for exactly which controls are wired up now versus inert placeholders.

A future apply-agent (filling in forms on company sites, not just Easy
Apply) will be able to reuse: the vacancy's canonical key
`(source, external_id)`, the `apply_url` JobsPipe already returns per
posting (no extra extraction step needed to find where to apply), the
`matched_resume` field (which resume to attach), the `applying`/`failed`
statuses already reserved in the dedup schema above, the Bio tab's stored
answers (main text + a growing list of question/answer pairs, some
pre-filled by the user ahead of time, some appended later from a Failed
vacancy's blocking questions), and the `web` role in `llm/client.py` —
reserved but unused in Phase 1 — which will read and fill out the
application form found at `apply_url`. None of this is implemented in
Phase 1 — it's only reserved in the schema, in the LLM client's interface,
and in the UI's inert controls, so no data migration or role rework is
needed later.

## UI (Streamlit)

Six tabs: **Search**, **Pipeline**, **Resumes**, **Integrations**,
**Models**, **Bio**. All six are visible in Phase 1; "Phase 1 vs Phase 2 UI
scope" below says exactly which controls do something yet.

- **Search** — filter row: a "Profile" dropdown of saved filter presets
  (plus a save icon that opens a small popup to add a new preset or
  overwrite the selected one by name), a job-titles tag input, a Location
  text field, Seniority and Format dropdowns (Any/Junior/Mid/Senior,
  Any/Remote/Hybrid/Onsite), a numeric Min score field, a "Search" button,
  and — right-aligned on the same row — a checked, disabled "Source:
  JobsPipe" checkbox (tooltip: "More sources can be added later"). Below a
  divider, "New results" lists vacancies sorted by score descending, each
  row showing: title + a star/score badge, company + a colored badge
  naming the matched resume (color is a deterministic hash of the resume
  name, so the same resume always gets the same color), a link (first
  ~30 characters of the URL, opens in a new tab), and a per-row icon button
  ("Add to queue", tooltipped) alongside a "Select all" + bulk "Add
  selected to queue" action above the list. Min score filters this list
  client-side as it's typed, no re-fetch.
- **Pipeline** — segmented sub-tabs "In queue · N", "Applied · N",
  "Failed · N". In queue and Applied share the Search tab's row styling
  (score badge, resume badge, link). In queue additionally has "Select
  all", a Manual-confirm/Auto mode toggle, and a "Run batch" button;
  Applied rows show the link first, then "applied Xd ago"; Failed groups
  each vacancy with its blocking application questions, each with an
  inline answer field and an "Add to Bio" button, plus a "Re-add to queue"
  action.
- **Resumes** — "Choose file" (`.pdf`, `.doc`, `.docx`, `.md`, `.txt`) only
  picks and displays a filename; a Name field; a separate "Upload" button,
  disabled until a file is chosen, that starts parsing. Below, a list of
  existing resumes (name, source filename, cache status) each with a
  delete icon.
- **Integrations** — one row per data provider (JobsPipe today); a
  provider's row states what it powers, followed by its API key section
  (Direct value / Env var toggle, defaulting to Env var
  `JOBS_PIPE_KEY`), and a shared Edit/Save/Cancel at the bottom.
- **Models** — three rows (Classification / Generation (resumes) /
  Browser control (web)), each a searchable model dropdown with an inline
  explanation of what the role is used for; Classification's dropdown is
  scoped to `typesafe/*` only. Below, the OpenRouter API key section
  (same Direct value / Env var pattern, default Env var
  `OPEN_ROUTER_KEY`). One shared Edit/Save/Cancel at the bottom covers all
  three model rows and the key together.
- **Bio** — a free-text "Main bio" field (context given on every
  application), a list of ~10 suggested common application questions the
  user can pre-answer, and an "Answered" list of question/answer pairs
  already saved — every row has independent Edit/Save/Cancel (Cancel on an
  empty, never-saved row just clears the input and stays in edit mode,
  rather than closing it). A footnote clarifies that inline answers given
  on a Failed vacancy (Phase 2) land in this same Answered list.

Both API-key sections (Models, Integrations) write a Direct-value key to
`settings.json`, which is already in `.gitignore`, same as `.env` — nothing
typed into either field is ever committed regardless of which mode is
chosen.

There's no separate CLI — the Streamlit app (`streamlit run`) is the only
entry point for the user.

### Phase 1 vs Phase 2 UI scope

All six tabs ship in Phase 1, but not every control inside them is backed
by real logic yet:

| Area | Phase 1 (real) | Phase 2 (inert placeholder in Phase 1) |
|---|---|---|
| Search | Filters, JobsPipe search, scoring, Min score, saved Profiles, "Add to queue" (per-row and bulk) | — |
| Pipeline → In queue | List, Select all, a manual "Mark applied" action per row | Manual-confirm/Auto mode toggle, "Run batch" |
| Pipeline → Applied | List, populated only by the manual "Mark applied" action | — |
| Pipeline → Failed | Sub-tab exists, empty in Phase 1 (nothing produces `failed` yet) | The entire tab's content: inline question answering, "Add to Bio", "Re-add to queue" |
| Review-before-submit popup | — | The whole popup — never opens in Phase 1, since nothing calls `runBatch` |
| Resumes / Integrations / Models | Fully functional | — |
| Bio | Fully functional as a data-entry screen | Being *read* by an apply-agent; the "answers reused automatically on future applications" behavior |

Phase 2 controls are visually present (per the mockup) so introducing them
later needs no layout rework, but they either do nothing or are hidden
behind a "coming soon" state — implementation detail left to Phase 2's own
design pass.

## Components and repository layout

```
├── pyproject.toml
├── .env.example        # OPEN_ROUTER_KEY, JOBS_PIPE_KEY
├── Makefile             # make ui / make test / make lint — a thin wrapper
├── resumes/             # local resume files + parse cache; in .gitignore
├── settings.json         # models (resume/web/jev), provider keys (if Direct value),
│                         # saved search Profiles, Bio; in .gitignore
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
│       └── app.py                  # Streamlit entrypoint (tabs: search, pipeline,
│                                    # resumes, integrations, models, bio)
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
resumes, no saved model choice, no saved Profiles, no Bio, no seen
vacancies), rather than crashing on a missing file or directory. The
Models tab already surfaces the no-model-chosen case as a UI error (see
"Risks"), so a cold start's only visible effect is that all three model
dropdowns start unset and Integrations/Models show their key sections
empty.

## Error handling

- JobsPipe: an API error, rate limit, or an empty free-tier credit balance
  — the node logs it and returns an empty list, without failing the whole
  run (resumes and matching still proceed on whatever was already fetched).
- Jev: retry once on a network error; on a persistent failure the vacancy is
  marked `score: null` and stays in the list (visible as unscored, rather
  than silently disappearing).
- An unreadable resume file — an explicit error in the UI on upload; we
  don't proceed further.

## Testing

- **Unit**: `resume/parse.py` with fixture text and a stubbed LLM client
  (`resume` role); `sources/jobspipe.py` with a stubbed HTTP response
  (fixture JSON matching JobsPipe's documented schema, including the
  `job_title_or`/`city_or`/`job_seniority_or`/`work_arrangement_or` filter
  mapping); `agent/nodes.py::match` with a stubbed Jev client (the API is
  obscure and hasn't been verified live at writing time — tests don't rely
  on a real response); dedup/lifecycle logic in `store/seen_vacancies.py`
  (including the `matched → queued → applied` transitions) — pure
  functions, tested without the network.
- **Manual/integration**: one real call to JobsPipe against its free tier,
  to confirm the live response and filter behavior still match the fixture
  shape used in unit tests — not run in CI, checked manually as needed.
- TDD: a test for the node's behavior is written before the implementation.

## Risks

| Risk | Response |
|---|---|
| JobsPipe doesn't cover AllJobs/Drushim and isn't guaranteed to surface every Israeli hi-tech vacancy — only what's published through the ATS platforms/boards it covers (though many Israeli startups do sit on Greenhouse/Lever/Comeet) | The `JobSource` protocol is already set up to add another source later if this coverage gap turns out to matter in practice |
| The JobsPipe API hasn't been verified live under a real key at spec-writing time — filter field names (`job_title_or`, `city_or`, `job_seniority_or`, `work_arrangement_or`, `job_country_code_or`) are confirmed against its docs only | Manually check a real response (curl/httpx script) before implementing `sources/jobspipe.py`; the client is stubbed at the HTTP boundary in tests |
| The Jev `Score` API hasn't been verified live, response format is uncertain | Before implementing `match` — manually check a real API response; Jev is stubbed at the client boundary in tests |
| JobsPipe's free tier is capped (1,000 jobs/month) | Fine at personal scale (occasional runs, sane per-search limits); paid tiers start at $49/month if more is needed |
| N resumes × M vacancies = N×M Jev calls per run | Jev's input is cheap ($0.042/M tokens), output is free; at personal scale (a handful of resumes, tens of vacancies per run) this isn't a concern |
| The resume cache goes stale unnoticed | Invalidated by file content hash and by model id — not by date, so both editing a resume and changing the model in the Models tab always re-parse |
| The user hasn't picked a model (`resume`/`web`/`jev`) yet, or `/models` is unreachable | The Models tab is a mandatory first step before parsing resumes, searching for jobs, or scoring; an explicit, role-specific UI error if a model isn't chosen, instead of a silent fallback to something hardcoded |
| Shipping the full Pipeline/Bio UI in Phase 1 (Manual-confirm/Auto, Failed, Add to Bio) while the apply-agent behind it doesn't exist yet could read as broken or misleading | "Phase 1 vs Phase 2 UI scope" documents exactly which controls are inert; Phase 2 controls are either disabled or clearly marked, never silently no-op |

## Stages

| # | Stage | Check |
|---|---|---|
| 0 | Skeleton, venv, config, `.gitignore`, Streamlit "hello world" | `make ui` opens an empty screen on a fresh clone — no `resumes/`, `settings.json`, or `seen_vacancies.json` present yet, nothing crashes |
| 1 | Models tab (three model dropdowns, including `typesafe/*`-scoped Classification) + Integrations tab (JobsPipe key) + OpenRouter key, both with Direct value/Env var | All three dropdowns show real models, both key sections work in either mode, everything lands in `settings.json` |
| 2 | Resume parsing + cache, multiple profiles, two-step upload | The Resumes tab shows a structured profile for an uploaded file; Upload stays disabled until a file is chosen |
| 3 | JobsPipe source client | Vacancies can be pulled from `sources/jobspipe.py` manually, independent of the agent, with filters mapped per "Search-tab filters → JobsPipe params" |
| 4 | Jev client + matching logic | On fixtures (a stubbed Jev), a correct aggregated score is computed using the model id from `settings.json` |
| 5 | The full graph, wired into the Search tab | A search in Streamlit runs end-to-end; results are sorted by score, tagged with a colored resume badge, and Min score filters them client-side |
| 6 | Dedup + queue + manual "Mark applied", filter Profiles, Bio tab | Adding a result to the queue moves it to Pipeline → In queue; marking it applied moves it to Applied; a repeat search doesn't show vacancies already seen; a saved Profile reloads its filters; Bio's Edit/Save/Cancel persists to `settings.json` |

## API usage note

Phase 1 has no scraping and no ToS risk of that kind anymore: JobsPipe is
used as a paying customer, through its documented API contract, not against
the terms of the sites it aggregates. The only remaining real-world-action
risk in this project is Phase 2's actual form submission on company sites —
already called out in "Phase 2 compatibility" and to be designed carefully
when that phase is brainstormed.
