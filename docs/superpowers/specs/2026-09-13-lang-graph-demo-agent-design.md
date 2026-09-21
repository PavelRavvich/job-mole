# Demo agent: LangGraph + Temporal + MCP + pgvector + Langfuse + OTel

Date: 2026-09-13
Status: **superseded** — never implemented (the repo had no code when the
project's direction changed), replaced by
[2026-09-22-job-match-agent-design.md](2026-09-22-job-match-agent-design.md).
Kept here as a historical record.

## Goal

A learning project. The point was not to build a product but to get
hands-on with seven technologies and see how they fit together. Hence the
main design-quality criterion: every layer must be runnable and testable on
its own, and each technology's role must not be for show — if a component
could be removed without breaking anything, it doesn't belong in the demo.

Out of scope: scaling, fault tolerance beyond what Temporal gives out of the
box, authentication, multi-tenancy, production deployment.

## Decisions

| Decision | Choice | Why |
|---|---|---|
| Domain | Synthetic: processing purchase requests | The domain itself doesn't matter; this one gives every technology a natural role |
| LLM | Claude (`claude-sonnet-5`) via `langchain-anthropic` | The project's primary provider |
| Embeddings | OpenAI `text-embedding-3-small`, 1536 dim | Anthropic has no embeddings API; local models pull in ~500MB of torch |
| Infrastructure | Docker Compose | One way to manage all the ready-made services, nothing left lying around on the host |
| Code | Local, a single venv via `uv` | A fast edit→run loop, debuggable from the IDE |
| LangGraph ↔ Temporal | Hybrid: a graph node = an activity | Durability where it actually matters, without describing the topology twice |
| Entry point | `Makefile` | Available everywhere, needs no install |

## Scenario

The user submits a request: "need 3 PyCharm licenses for the platform
team." The agent finds relevant purchasing-policy items and similar past
decisions in memory, plans tool calls, checks the department's budget, and
creates the order. If the amount exceeds $500, the process stops and waits
for a human decision — for however long it takes, surviving a worker
restart. After the decision, the agent records the outcome in memory, and
the next similar request will find it in search.

The approval threshold ($500) is the only business constant, and it lives
in config.

## Architecture

```
        CLI (submit / approve / reject / run-local / memory)
                      │
                      ▼
        ┌─────────────────────────────┐
        │  Temporal Workflow           │   durable state, signals
        │  PurchaseRequestWorkflow     │   waits for a human
        └──────────┬───────────────────┘
                   │ activities
        ┌──────────▼──────────────────┐
        │  Agent nodes (pure funcs)   │◄── LangGraph uses the same
        │  recall → plan → act →      │    functions in local mode
        │  finalize                   │
        └───┬──────────┬──────────┬───┘
            │          │          │
            ▼          ▼          ▼
      pgvector     Claude     MCP server (stdio)
      (memory)                search_catalog
                              check_budget
                              create_purchase_order
                                    │
                                    ▼
                              Postgres (business data)

  Observability (cross-cutting): Langfuse ← LLM traces
                                 Prometheus ← metrics via OTel → Grafana
```

### Key decision: nodes as reusable functions

Agent nodes are pure functions of the form `(State) -> dict` (a partial
state update). They know nothing about LangGraph or Temporal. On top of
them sit two independent run modes:

- **Local** (`agent/graph.py`): nodes are assembled into a `StateGraph`, the
  graph runs in a single process. Fast, debuggable from the IDE, no
  Temporal required.
- **Durable** (`temporal/workflow.py`): each node is wrapped in an activity,
  the workflow calls them in sequence and knows how to pause between `act`
  and `finalize`.

That's the whole point of the chosen hybrid. The learning value is direct:
the same agent is visible in two modes, and the difference between "just a
graph" and a "durable graph" stops being an abstraction and becomes two
commands in a terminal.

The constraint that shapes this boundary: the Temporal Python SDK enforces
a strict sandbox for workflow code — determinism is mandatory there, and
imports of LangGraph, HTTP clients, and LLM SDKs are forbidden. So all the
"real" work lives in activities, and the workflow contains only
orchestration and waiting on a signal.

## Components

### 1. Environment

One venv for everything, managed by `uv`. Python 3.12 (installed).
Dependencies and scripts live in `pyproject.toml`, no `requirements.txt`.

Packages by group: agent (`langgraph`, `langchain-anthropic`,
`langchain-openai`, `langchain-mcp-adapters`), MCP (`mcp`), durability
(`temporalio`), memory (`psycopg[binary]`, `pgvector`), observability
(`langfuse`, `opentelemetry-sdk`, `opentelemetry-exporter-prometheus`,
`opentelemetry-exporter-otlp`), glue (`pydantic-settings`, `typer`), dev
(`pytest`, `pytest-asyncio`, `ruff`).

Versions aren't pinned in the spec — resolved at install time, locked in
`uv.lock`.

`config.py` is the single place environment gets read (`pydantic-settings`).
No `os.environ` anywhere else in the code: this keeps configuration
inspectable and testable.

### 2. Postgres + pgvector — agent memory

Image `pgvector/pgvector:pg17`, port 5432. The schema is applied via
migration SQL when the container starts (`sql/` is mounted into
`docker-entrypoint-initdb.d`).

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE memory (
  id         bigserial PRIMARY KEY,
  kind       text NOT NULL,              -- 'policy' | 'past_decision'
  content    text NOT NULL,
  metadata   jsonb NOT NULL DEFAULT '{}',
  embedding  vector(1536) NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ON memory USING hnsw (embedding vector_cosine_ops);

CREATE TABLE catalog (
  sku text PRIMARY KEY, name text NOT NULL, unit_price numeric NOT NULL);

CREATE TABLE budgets (
  department text PRIMARY KEY, limit_usd numeric NOT NULL, spent_usd numeric NOT NULL DEFAULT 0);

CREATE TABLE purchase_orders (
  id bigserial PRIMARY KEY, department text NOT NULL, sku text NOT NULL,
  qty int NOT NULL, total_usd numeric NOT NULL,
  status text NOT NULL,                  -- 'created' | 'rejected'
  workflow_id text, created_at timestamptz NOT NULL DEFAULT now());
```

The `memory/` module exposes three operations: `embed(text)`,
`search(query, kind, limit)` (cosine distance), `remember(kind, content,
metadata)`. It knows nothing about the agent and is tested on its own.

Seed data: 8-10 policy items and 3-4 past decisions — enough for search to
return something meaningful, and little enough to read by eye.

### 3. MCP server — the project's own tool

A separate process, stdio transport, the official Python SDK (`mcp`). Three
tools:

| Tool | Action | Side effect |
|---|---|---|
| `search_catalog(query)` | Search catalog items | no |
| `check_budget(department)` | Department's remaining budget | no |
| `create_purchase_order(department, sku, qty)` | Create an order, deduct from budget | **yes** |

The server is detachable: it can be wired into Claude Code or any MCP
client and poked by hand, without running the agent at all. That's a
separate, checkable deliverable for this stage, not an implementation
detail.

The agent connects as an MCP client via `MultiServerMCPClient` with the
config `{"transport": "stdio", "command": ..., "args": [...]}`, and gets
tools via `get_tools()` — they arrive as regular LangChain `BaseTool`
objects and get bound to the model.

### 4. LangGraph — the agent

State (`TypedDict`): the original request, context retrieved from memory,
message history, the planned tool call, its result, the decision, an
`needs_approval` flag.

Nodes:

- **`recall`** — embeds the request, runs two pgvector searches (policy +
  past decisions), puts the result into state.
- **`plan`** — Claude gets the request and context, responds with a tool
  call.
- **`act`** — executes the call via MCP. This is also where the total is
  computed and `needs_approval` is set if it's above the threshold.
- **`finalize`** — writes the decision as text and records the outcome in
  memory.

Edges are linear: `recall → plan → act → finalize`. There's exactly one
branch, and it lives outside the graph — in Temporal. In local mode the
approval step is skipped (with a warning in the output), because there's
nowhere to wait for a signal — and that's exactly the difference between
the two modes the demo is meant to show.

### 5. Temporal — durable workflow and human approval

One container: the `temporalio/temporal` image in `server start-dev` mode —
the server on 7233 and the web UI on 8233 in a single process, state in
sqlite. No separate containers needed for the DB or the UI.

```python
@workflow.defn
class PurchaseRequestWorkflow:
    @workflow.run
    async def run(self, req: PurchaseRequest) -> Decision: ...

    @workflow.signal
    def approve(self, note: str) -> None: ...

    @workflow.signal
    def reject(self, note: str) -> None: ...

    @workflow.query
    def status(self) -> str: ...
```

Execution flow: the workflow calls activities `recall` → `plan` → `act`.
If `act` returned `needs_approval`, the workflow does
`await workflow.wait_condition(lambda: self._decision is not None)` and
freezes. The worker process can be killed at this point — the state lives
on the Temporal server. After an `approve`/`reject` signal, `finalize`
runs, with a different outcome on rejection.

The retry policy on activities is the default, except for `act`: it has a
side effect, so `maximum_attempts=1`, and idempotency is ensured by writing
`workflow_id` into `purchase_orders`.

`worker.py` registers the workflow and the activities. It's the one that
gets stopped and restarted to see durability in action.

### 6. Langfuse — tracing and eval

The official self-hosted compose: `langfuse-web`, `langfuse-worker`, its
own `postgres`, `clickhouse`, `redis`, `minio`. Langfuse's DB is separate
from ours — mixing them in a demo would be harmful, the boundary is clearer
this way. UI on 3000.

Integration: the `CallbackHandler` from `langfuse.langchain` is passed via
`config={"callbacks": [handler]}` when the graph is invoked. Keys and
`LANGFUSE_BASE_URL` (`http://localhost:3000`) are read from the
environment. `flush()` is called at the end of the process — otherwise a
short-lived CLI process would exit before the background send completes.

The Langfuse v3 SDK is built on OpenTelemetry, so our own spans and the LLM
call spans land in one tree — no separate bridge needs to be written.

Eval, at demo scale: one dataset of 5-6 requests, a run over it, and an
LLM-as-judge check of "was the right tool chosen." The goal is to see the
mechanics, not to build a quality-evaluation system.

### 7. OpenTelemetry + Prometheus + Grafana — metrics

No collector needed: the app's OTel SDK exposes a Prometheus endpoint
itself (`opentelemetry-exporter-prometheus`) on `:9464/metrics`, and
Prometheus scrapes it. One fewer moving part, and OTel's role is preserved.

Metrics:

| Metric | Type | Labels |
|---|---|---|
| `agent_runs_total` | counter | `outcome` = approved / rejected / auto |
| `agent_node_duration_seconds` | histogram | `node` |
| `llm_tokens_total` | counter | `type` = input / output |
| `mcp_tool_calls_total` | counter | `tool`, `status` |
| `approvals_pending` | gauge | — |

Grafana (port 3001, since 3000 is taken by Langfuse) comes up with
provisioning: a Prometheus datasource and one four-panel dashboard — runs
by outcome, node latency, token spend, pending requests. The dashboard
lives in the repo as JSON rather than being configured by hand, otherwise
it would be lost whenever the container is recreated.

Worth keeping in mind, the division of labor between Langfuse and
Prometheus: Prometheus answers "how much and how fast," Langfuse answers
"what exactly did the model respond in this specific run, and what did it
cost."

## Port map

| Service | Port |
|---|---|
| Postgres (ours) | 5432 |
| Temporal gRPC | 7233 |
| Temporal Web UI | 8233 |
| Langfuse UI | 3000 |
| Grafana | 3001 |
| Prometheus | 9090 |
| App metrics | 9464 |

## Repository layout

```
├── pyproject.toml
├── .env.example
├── Makefile
├── docker/
│   ├── docker-compose.yml
│   ├── prometheus/prometheus.yml
│   └── grafana/provisioning/{datasources,dashboards}/
├── sql/
│   ├── 001_schema.sql
│   └── 002_seed.sql
├── src/demo_agent/
│   ├── config.py
│   ├── memory/          # embeddings, search, writes
│   ├── mcp_server/      # our own MCP server
│   ├── agent/           # state.py, nodes.py, graph.py
│   ├── temporal/        # workflow.py, activities.py, worker.py
│   ├── obs/             # otel.py, langfuse.py
│   └── cli.py
└── tests/
```

The boundaries are deliberately strict: `memory` knows nothing about the
agent, `mcp_server` runs without Temporal, `agent/nodes.py` doesn't know
it's being wrapped in a workflow. This isn't aesthetics — it's what lets
the project be built in stages and each layer be checked apart from the
rest.

## Commands

```
make up          # bring up all infrastructure
make down        # stop it
make seed        # load seed data and compute embeddings
make worker      # run the Temporal worker
make run         # a local graph run, no Temporal
make submit      # submit a request via Temporal
make approve     # send an approval signal
make memory      # search the agent's memory
make test / lint
```

Underneath `make` sits a single `typer` CLI, installed into the venv as
`demo` (`demo submit`, `demo memory search "..."`, etc.). The Makefile is a
thin wrapper with convenient defaults; everything is also available
directly via `demo`.

## Stages

Every stage ends with something runnable and visible. A stage isn't
considered done until its check passes.

| # | Stage | Check |
|---|---|---|
| 0 | Skeleton, venv, config, compose with all infra | `make up`, all UIs open |
| 1 | pgvector + memory + seed | `demo memory search "licenses"` returns something meaningful |
| 2 | MCP server | tools can be poked by hand, independent of the agent |
| 3 | LangGraph agent, local | `make run` takes a request through end-to-end |
| 4 | Temporal + approval | `make submit` pauses, the worker restarts, `make approve` carries it through |
| 5 | Langfuse | the run's trace tree is visible in the UI |
| 6 | OTel + Prometheus + Grafana | the dashboard fills with data |

The order isn't arbitrary: 1-3 produce a working agent, 4 adds durability,
5-6 add observability on top of something already working. If the project
has to be cut short, stopping at any boundary still leaves something whole.

## Testing

- **Unit** — agent nodes with a stubbed LLM and a stubbed MCP client; memory
  functions against a running Postgres. Fast, run constantly.
- **Integration** — the whole MCP server (spawn the process, call a tool);
  the workflow via `WorkflowEnvironment` from `temporalio.testing`,
  including verifying that a signal carries a stuck run through to
  completion.
- Tests require infrastructure to be up (`make up`) — for a learning
  project, that's more honest than mocking everything.

TDD: a behavior test is written before the implementation.

## Risks

| Risk | Response |
|---|---|
| The Langfuse stack is heavy (6 containers) | Stage 5 comes late; if Docker can't handle it, Langfuse is disabled via a profile, everything else still works |
| The Temporal sandbox breaks imports | The "the workflow imports nothing" boundary is baked into the design from the start |
| Token spend while debugging | Model and temperature live in config; the LLM is stubbed for tests |
| Port 3000 is claimed by both Langfuse and Grafana | Grafana moved to 3001, recorded in the port map |
