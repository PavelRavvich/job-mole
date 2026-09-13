# Демо-агент: LangGraph + Temporal + MCP + pgvector + Langfuse + OTel

Дата: 2026-09-13
Статус: дизайн согласован, ожидает ревью

## Цель

Учебный проект. Задача — не построить продукт, а вживую пощупать семь технологий
и увидеть, как они стыкуются друг с другом. Отсюда главный критерий качества
дизайна: каждый слой должен запускаться и проверяться отдельно от остальных, а
роль каждой технологии должна быть неотыгранной — если компонент можно выбросить
и ничего не сломается, он в демо лишний.

Не входит в цель: масштабирование, отказоустойчивость сверх той, что даёт
Temporal из коробки, аутентификация, мультитенантность, продовый деплой.

## Принятые решения

| Решение | Выбор | Почему |
|---|---|---|
| Домен | Синтетический: обработка заявок на закупку | Домен неважен сам по себе; этот даёт естественную роль каждой технологии |
| LLM | Claude (`claude-sonnet-5`) через `langchain-anthropic` | Основной провайдер проекта |
| Эмбеддинги | OpenAI `text-embedding-3-small`, 1536 dim | У Anthropic нет embeddings API; локальные модели тянут ~500МБ torch |
| Инфраструктура | Docker Compose | Один способ управления всеми готовыми сервисами, ничего не оседает на хосте |
| Код | Локально, один venv через `uv` | Быстрый цикл правка→запуск, отладка из IDE |
| LangGraph ↔ Temporal | Гибрид: узел графа = activity | Durability там, где она осмысленна, без двойного описания топологии |
| Точка входа | `Makefile` | Есть везде, не требует установки |

## Сценарий

Пользователь подаёт заявку: «нужны 3 лицензии PyCharm для отдела платформы».
Агент находит в памяти релевантные пункты закупочной политики и похожие прошлые
решения, планирует вызовы инструментов, проверяет бюджет отдела, создаёт заказ.
Если сумма превышает 500 USD, процесс останавливается и ждёт решения человека —
сколько угодно долго, переживая перезапуск worker'а. После решения агент
записывает исход в память, и следующая похожая заявка найдёт его в поиске.

Порог аппрува (500 USD) — единственная бизнес-константа, вынесена в конфиг.

## Архитектура

```
        CLI (submit / approve / reject / run-local / memory)
                      │
                      ▼
        ┌─────────────────────────────┐
        │  Temporal Workflow          │   durable-состояние, сигналы
        │  PurchaseRequestWorkflow    │   ожидание человека
        └──────────┬──────────────────┘
                   │ activities
        ┌──────────▼──────────────────┐
        │  Узлы агента (чистые ф-ции) │◄── те же функции использует
        │  recall → plan → act →      │    LangGraph в локальном режиме
        │  finalize                   │
        └───┬──────────┬──────────┬───┘
            │          │          │
            ▼          ▼          ▼
      pgvector     Claude     MCP-сервер (stdio)
      (память)                search_catalog
                              check_budget
                              create_purchase_order
                                    │
                                    ▼
                              Postgres (бизнес-данные)

  Наблюдаемость (сквозная): Langfuse ← трейсы LLM
                            Prometheus ← метрики через OTel → Grafana
```

### Ключевое решение: узлы как переиспользуемые функции

Узлы агента — чистые функции вида `(State) -> dict` (частичное обновление
состояния). Они не знают ни про LangGraph, ни про Temporal. Поверх них два
независимых режима запуска:

- **Локальный** (`agent/graph.py`): узлы собраны в `StateGraph`, граф исполняется
  в одном процессе. Быстро, отлаживается из IDE, не требует Temporal.
- **Durable** (`temporal/workflow.py`): каждый узел обёрнут в activity, workflow
  вызывает их последовательно и умеет вставать на паузу между `act` и `finalize`.

Это и есть суть выбранного гибрида. Учебная ценность прямая: один и тот же агент
виден в двух режимах, и разница между «просто граф» и «durable-граф» становится
не абстракцией, а двумя командами в терминале.

Ограничение, формирующее эту границу: у Temporal Python SDK строгая песочница для
workflow-кода — детерминизм там обязателен, а импорты LangGraph, HTTP-клиентов и
LLM SDK запрещены. Поэтому вся «настоящая» работа живёт в activities, а workflow
содержит только оркестрацию и ожидание сигнала.

## Компоненты

### 1. Окружение

Один venv на всё, управляется `uv`. Python 3.12 (установлен). Зависимости и
скрипты — в `pyproject.toml`, без `requirements.txt`.

Пакеты по группам: агент (`langgraph`, `langchain-anthropic`, `langchain-openai`,
`langchain-mcp-adapters`), MCP (`mcp`), durability (`temporalio`), память
(`psycopg[binary]`, `pgvector`), наблюдаемость (`langfuse`,
`opentelemetry-sdk`, `opentelemetry-exporter-prometheus`,
`opentelemetry-exporter-otlp`), обвязка (`pydantic-settings`, `typer`), dev
(`pytest`, `pytest-asyncio`, `ruff`).

Версии не пиним в спеке — резолвятся при установке, фиксируются в `uv.lock`.

`config.py` — единственная точка чтения окружения (`pydantic-settings`). Никаких
`os.environ` в остальном коде: это делает конфигурацию обозримой и тестируемой.

### 2. Postgres + pgvector — память агента

Образ `pgvector/pgvector:pg17`, порт 5432. Схема применяется миграционным SQL при
старте контейнера (`sql/` монтируется в `docker-entrypoint-initdb.d`).

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

Модуль `memory/` даёт три операции: `embed(text)`, `search(query, kind, limit)`
(косинусное расстояние), `remember(kind, content, metadata)`. Он не знает про
агента и тестируется отдельно.

Сид-данные: 8-10 пунктов политики и 3-4 прошлых решения — достаточно, чтобы поиск
возвращал осмысленное, и мало, чтобы читать глазами.

### 3. MCP-сервер — свой инструмент

Отдельный процесс, транспорт stdio, официальный Python SDK (`mcp`). Три
инструмента:

| Инструмент | Действие | Побочный эффект |
|---|---|---|
| `search_catalog(query)` | Поиск позиций в каталоге | нет |
| `check_budget(department)` | Остаток бюджета отдела | нет |
| `create_purchase_order(department, sku, qty)` | Создание заказа, списание бюджета | **да** |

Сервер отчуждаем: его можно подключить к Claude Code или любому MCP-клиенту и
подёргать руками, не запуская агента вообще. Это отдельный проверяемый результат
этапа, а не деталь реализации.

Агент подключается как MCP-клиент через `MultiServerMCPClient` с конфигурацией
`{"transport": "stdio", "command": ..., "args": [...]}`, инструменты получает
через `get_tools()` — они приходят как обычные LangChain `BaseTool` и биндятся к
модели.

### 4. LangGraph — агент

Состояние (`TypedDict`): исходная заявка, найденный контекст из памяти, история
сообщений, запланированный вызов инструмента, результат вызова, решение, флаг
`needs_approval`.

Узлы:

- **`recall`** — эмбеддинг заявки, два поиска в pgvector (политика + прошлые
  решения), результат кладётся в состояние.
- **`plan`** — Claude получает заявку и контекст, отвечает вызовом инструмента.
- **`act`** — исполнение вызова через MCP. Здесь же вычисляется сумма и
  выставляется `needs_approval`, если она выше порога.
- **`finalize`** — формулировка решения текстом и запись исхода в память.

Рёбра линейные: `recall → plan → act → finalize`. Ветвление одно и живёт снаружи
графа — в Temporal. В локальном режиме аппрув пропускается (с предупреждением в
выводе), потому что ждать сигнал негде — и это ровно та разница между режимами,
которую демо должно показать.

### 5. Temporal — durable workflow и human-approval

Один контейнер: образ `temporalio/temporal` в режиме `server start-dev` — сервер
на 7233 и Web UI на 8233 в одном процессе, состояние в sqlite. Отдельные
контейнеры под БД и UI не нужны.

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

Ход выполнения: workflow вызывает activities `recall` → `plan` → `act`. Если
`act` вернул `needs_approval`, workflow делает
`await workflow.wait_condition(lambda: self._decision is not None)` и замирает.
Процесс worker'а в этот момент можно убить — состояние живёт на сервере Temporal.
После сигнала `approve`/`reject` выполняется `finalize`, при отказе — с иным
исходом.

Политика ретраев на activities — дефолтная, кроме `act`: у него побочный эффект,
поэтому `maximum_attempts=1`, а идемпотентность обеспечивается тем, что
`workflow_id` пишется в `purchase_orders`.

`worker.py` регистрирует workflow и activities. Именно его останавливают и
запускают заново, чтобы увидеть durability своими глазами.

### 6. Langfuse — трейсинг и eval

Официальный self-hosted compose: `langfuse-web`, `langfuse-worker`, собственный
`postgres`, `clickhouse`, `redis`, `minio`. Своя БД у Langfuse отдельная от нашей
— смешивать их в демо вредно, границу видно чётче. UI на 3000.

Интеграция: `CallbackHandler` из `langfuse.langchain` передаётся в
`config={"callbacks": [handler]}` при вызове графа. Ключи и `LANGFUSE_BASE_URL`
(`http://localhost:3000`) читаются из окружения. `flush()` вызывается в конце
процесса — иначе короткоживущий CLI-процесс завершится раньше фоновой отправки.

Langfuse v3 SDK построен на OpenTelemetry, поэтому наши собственные спаны и спаны
LLM-вызовов попадают в одно дерево — отдельного моста писать не нужно.

Eval в объёме демо: один датасет из 5-6 заявок, прогон по нему и LLM-as-judge
оценка «правильный ли инструмент выбран». Задача — увидеть механику, а не
построить систему оценки качества.

### 7. OpenTelemetry + Prometheus + Grafana — метрики

Коллектор не нужен: OTel SDK в приложении сам экспонирует Prometheus-эндпоинт
(`opentelemetry-exporter-prometheus`) на `:9464/metrics`, Prometheus его
скрейпит. На один движущийся компонент меньше, а роль OTel сохраняется.

Метрики:

| Метрика | Тип | Метки |
|---|---|---|
| `agent_runs_total` | counter | `outcome` = approved / rejected / auto |
| `agent_node_duration_seconds` | histogram | `node` |
| `llm_tokens_total` | counter | `type` = input / output |
| `mcp_tool_calls_total` | counter | `tool`, `status` |
| `approvals_pending` | gauge | — |

Grafana (порт 3001, т.к. 3000 занят Langfuse) поднимается с provisioning:
datasource Prometheus и один дашборд из четырёх панелей — прогоны по исходам,
латентность узлов, расход токенов, заявки в ожидании. Дашборд лежит в репозитории
как JSON, а не настраивается руками, иначе он потеряется при пересоздании
контейнера.

Разделение ответственности между Langfuse и Prometheus стоит держать в голове:
Prometheus отвечает на вопрос «сколько и как быстро», Langfuse — «что именно
модель ответила в конкретном прогоне и сколько это стоило».

## Карта портов

| Сервис | Порт |
|---|---|
| Postgres (наш) | 5432 |
| Temporal gRPC | 7233 |
| Temporal Web UI | 8233 |
| Langfuse UI | 3000 |
| Grafana | 3001 |
| Prometheus | 9090 |
| Метрики приложения | 9464 |

## Структура репозитория

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
│   ├── memory/          # эмбеддинги, поиск, запись
│   ├── mcp_server/      # свой MCP-сервер
│   ├── agent/           # state.py, nodes.py, graph.py
│   ├── temporal/        # workflow.py, activities.py, worker.py
│   ├── obs/             # otel.py, langfuse.py
│   └── cli.py
└── tests/
```

Границы намеренно жёсткие: `memory` не знает про агента, `mcp_server`
запускается без Temporal, `agent/nodes.py` не знает, что его оборачивают в
workflow. Это не эстетика — это то, что позволяет собирать проект этапами и
проверять каждый слой в отрыве от остальных.

## Команды

```
make up          # поднять всю инфраструктуру
make down        # остановить
make seed        # залить сид-данные и посчитать эмбеддинги
make worker      # запустить Temporal worker
make run         # локальный прогон графа без Temporal
make submit      # подать заявку через Temporal
make approve     # послать сигнал одобрения
make memory      # поиск по памяти агента
make test / lint
```

Под `make` лежит один CLI на `typer`, ставящийся в venv как `demo`
(`demo submit`, `demo memory search "..."` и т.д.). Makefile — тонкая обёртка с
удобными дефолтами; всё то же доступно напрямую через `demo`.

## Этапы

Каждый этап заканчивается результатом, который можно запустить и увидеть. Этап не
считается закрытым, пока его проверка не пройдена.

| # | Этап | Проверка |
|---|---|---|
| 0 | Скелет, venv, конфиг, compose со всей инфрой | `make up`, все UI открываются |
| 1 | pgvector + память + сид | `demo memory search "лицензии"` возвращает осмысленное |
| 2 | MCP-сервер | инструменты дёргаются вручную, в отрыве от агента |
| 3 | LangGraph-агент локально | `make run` проходит заявку end-to-end |
| 4 | Temporal + approval | `make submit` встаёт на паузу, worker перезапускается, `make approve` доводит до конца |
| 5 | Langfuse | дерево трейса прогона видно в UI |
| 6 | OTel + Prometheus + Grafana | дашборд заполняется данными |

Порядок не произвольный: 1-3 дают работающего агента, 4 добавляет durability,
5-6 — наблюдаемость поверх уже работающего. Если проект придётся свернуть раньше
срока, обрыв на любой границе оставляет что-то целое.

## Тестирование

- **Unit** — узлы агента с подменённым LLM и подменённым MCP-клиентом; функции
  памяти против поднятого Postgres. Быстрые, гоняются постоянно.
- **Интеграционные** — MCP-сервер целиком (запуск процесса, вызов инструмента);
  workflow через `WorkflowEnvironment` из `temporalio.testing`, включая проверку
  того, что сигнал доводит до конца зависший прогон.
- Тесты требуют поднятой инфраструктуры (`make up`) — для учебного проекта это
  честнее, чем моки поверх всего.

Разработка по TDD: тест на поведение пишется до реализации.

## Риски

| Риск | Реакция |
|---|---|
| Langfuse-стек тяжёлый (6 контейнеров) | Этап 5 идёт поздно; если Docker не тянет, Langfuse отключается через профиль, остальное работает |
| Песочница Temporal ломает импорты | Граница «workflow ничего не импортирует» заложена в дизайн с самого начала |
| Расход токенов при отладке | Модель и температура в конфиге; для тестов LLM подменяется |
| Порт 3000 занят у Langfuse и Grafana | Grafana вынесена на 3001, зафиксировано в карте портов |
