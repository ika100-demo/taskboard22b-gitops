---
plan_id: 001-persistent-tasks-in-postgres-with-a-web
spec: 001-persistent-tasks-in-postgres-with-a-web
feature: Persistent tasks in Postgres with a web UI to list and create them
gitops_app: ika100-demo/taskboard22b-gitops
status: draft
repos:
  - id: taskboard22b-gitops
    shape: gitops-app
    summary: Postgres database for the Task API, 2 Task API instances in dev, web UI wired to the Task API
    acs: [AC-001.1, AC-001.2, AC-001.3, AC-001.4]
    order: before-merge
    gitops:
      - {resource: postgres, type: database}
      - {uses: postgres, component: taskboard22b-api}
      - {replicas: 2, component: taskboard22b-api, instance: dev}
      - {env: {TASK_API_URL: 'http://taskboard22b-api'}, component: taskboard22b-web}
    depends_on: []
    done: false
  - id: taskboard22b-api
    shape: service-python
    summary: Persistent tasks in Postgres
    acs: [AC-001.1, AC-001.2, AC-001.6, AC-001.8]
    criteria:
      - "Given tasks were created through `POST /tasks` and the database is reachable through `DATABASE_URL`, when the service is restarted (a new process against the same database), then `GET /tasks` returns `200` with every one of those tasks, each with the same `id`, `title`, `due_date` and `created_at` it was created with, ordered by creation time oldest first (system AC-001.1) (replaces AC-001.6)"
      - "Given an empty database without the service's tables, when the service starts, then it creates its schema itself and `GET /tasks` returns `200` with `[]` (system AC-001.1)"
      - "Given two instances of the service use the same database, when a task is created through `POST /tasks` on one instance, then `GET /tasks` on the other instance contains that task with the same `id`, `title`, `due_date` and `created_at` (system AC-001.2)"
      - "Given the database holds 10,000 tasks, when clients send valid `POST /tasks` requests, also concurrently through two instances, then each answers `507` with `{\"detail\": \"Task limit of 10000 reached\"}` and the task count stays at 10,000 (system AC-001.8, AC-001.2)"
      - "Given the database is unreachable, when a client sends `GET /tasks` or a valid `POST /tasks`, then the service answers `503` with `{\"detail\": \"Tasks are temporarily unavailable\"}` within 6 seconds, no task is created, and `GET /ready` answers `503` while `GET /health` stays `200` (system AC-001.6)"
      - "Given the database was unreachable and becomes reachable again, when a client sends `GET /ready`, `GET /tasks` and a valid `POST /tasks`, then they answer `200`, `200` and `201` without a restart of the service (system AC-001.6)"
    arguments: |
      Withdraw assumption A8 of docs/specs/001-create-and-list-tasks-in-memory (one instance, each holding its own tasks): tasks now live in the shared Postgres database.
      Connection comes from DATABASE_URL (and PG*), injected by the gitops entry; never a default credential in code.
      The request/response contract of spec 001 (fields, 422 body, 201, 507, ordering) stays unchanged; only the 503 and /ready behaviour are new.
      Ordering oldest first must stay stable for equal created_at (insertion order, e.g. a sequence column).
    depends_on: []
    done: false
  - id: taskboard22b-web
    shape: web-nextjs
    summary: Task list and create form on the start page
    acs: [AC-001.3, AC-001.4, AC-001.5, AC-001.7, AC-001.8]
    criteria:
      - "Given the Task API's `GET /tasks` returns tasks, when a user opens the start page `/`, then the page lists every returned task in the order the Task API returned it (oldest first), each showing its `title` and its `due_date` as `YYYY-MM-DD`, or the text \"No due date\" when `due_date` is `null`; given `GET /tasks` returns `[]`, the page shows the text \"No tasks yet\" (system AC-001.3)"
      - "Given the start page is open, when the user enters the title \"Buy milk\", picks the due date 2026-12-24 in the date field and submits the create form, then the web sends `POST /tasks` with `{\"title\": \"Buy milk\", \"due_date\": \"2026-12-24\"}` to the Task API at `TASK_API_URL`, and on `201` the returned task appears at the end of the list without a page reload and the form is emptied (system AC-001.4)"
      - "Given the start page is open, when the user submits a valid title without picking a due date, then the web sends `due_date` `null` and on `201` the new task appears in the list with the text \"No due date\" (system AC-001.4, AC-001.3)"
      - "Given the start page is open, when the user submits the create form with an empty title, a title of only whitespace, or a title longer than 200 characters after trimming, then the page shows \"Title must be 1 to 200 characters\" next to the title field, keeps the entered values and sends no `POST /tasks`; a title of 1 or 200 characters after trimming is sent trimmed, and a `422` from the Task API naming `title` shows the same message (system AC-001.5)"
      - "Given the Task API is unreachable, does not answer `GET /tasks` within 5 seconds, or answers `503` or any status the contract does not list, when a user opens the start page, then the page shows \"Tasks cannot be loaded right now. Please try again later.\" instead of the list (system AC-001.7)"
      - "Given the start page is open, when the user submits a valid task and the Task API is unreachable, does not answer within 5 seconds, or answers `503` or any status the contract does not list, then the page shows \"The task could not be saved. Please try again.\", keeps the entered values and leaves the list unchanged (system AC-001.7)"
      - "Given the start page is open, when the user submits a valid task and the Task API answers `POST /tasks` with `507`, then the page shows \"The task limit of 10,000 is reached.\", keeps the entered values and leaves the list unchanged (system AC-001.8)"
    arguments: |
      The Task API is called from the Next.js server (server component for the list, a server action or route handler for create), never from the browser, so TASK_API_URL can stay cluster-internal and no CORS is needed.
      The page text is English; the start page `/` is the task page.
    depends_on: []
    done: false
gitops_pin: []
---

## Decomposition

Three repos, one level: the gitops entry lands first (`before-merge`), then the Task API and the web UI build in parallel against the contract below. The Task API owns persistence, the shared view of 2 instances, the `503` and its recovery and the cap of 10,000 (A2, A7). The web UI owns the start page, its form, its own title check (A5) and the mapping of every Task API answer to the page texts. The gitops repo owns the database, the dev replica count AC-001.2 states and the wiring of the web UI to the Task API (A4: only the Task API uses the database). No library: the two repos share no code, only the HTTP contract. No repo is missing; no `/component:new` is needed.

## taskboard22b-gitops — Postgres database for the Task API, 2 Task API instances in dev, web UI wired to the Task API

**Shape:** `gitops-app` · **Criteria:** AC-001.1, AC-001.2, AC-001.3, AC-001.4 · **Depends on:** []

Declares the `postgres` database resource (CloudNativePG, one per environment, A9) and lets `taskboard22b-api` use it, which injects `DATABASE_URL` and `PG*` (AC-001.1). Sets `taskboard22b-api` to 2 replicas in dev so that AC-001.2 holds where the system runs today; staging and prod get their replica count when they are promoted. Wires `taskboard22b-web` to the Task API through `TASK_API_URL=http://taskboard22b-api` (the Service listens on port 80 in the same namespace), so the start page can list and create tasks (AC-001.3, AC-001.4). The Task API's existing readiness probe `/ready` takes an instance out of the Service while the database is down (AC-001.6); no probe change is needed.

## taskboard22b-api — Persistent tasks in Postgres

**Shape:** `service-python` · **Criteria:** AC-001.1, AC-001.2, AC-001.6, AC-001.8 · **Depends on:** []

Replaces the in-memory store of its spec 001 with a Postgres store behind the existing `TaskStore` protocol, so that tasks survive restarts (AC-001.1) and both instances list the same tasks (AC-001.2). Its spec supersedes the old AC-001.6 ("empty after restart") and withdraws its assumption A8. It answers `503` and reports not ready while the database is unreachable and recovers by itself (AC-001.6, A7). It keeps the cap of 10,000 with `507`, now counted in the database and safe across instances (AC-001.8, A2). Tasks held in memory at the switch are not carried over (A3).

## taskboard22b-web — Task list and create form on the start page

**Shape:** `web-nextjs` · **Criteria:** AC-001.3, AC-001.4, AC-001.5, AC-001.7, AC-001.8 · **Depends on:** []

Turns the start page `/` into the task page: the list from `GET /tasks` (AC-001.3), a create form whose new task appears without a reload (AC-001.4, A6), the title check before sending (AC-001.5, A5) and the messages for an unavailable Task API (AC-001.7) and the reached cap (AC-001.8). It reads tasks only through the Task API (A4).

## Contract

The Task API contract of `taskboard22b-api` spec 001 (its `design.md`) stays as it is (A1); this plan adds the `503` answers and the `/ready` behaviour. Base URL for the web UI: the env var `TASK_API_URL` (`http://taskboard22b-api` in the cluster). No authentication on any endpoint (A10).

**Task object**

```json
{"id": "3f2b8c1e-…", "title": "Buy milk", "due_date": "2026-12-24", "created_at": "2026-11-20T10:15:30.123456Z"}
```

| Field | Type | Rule |
|---|---|---|
| `id` | string | server-assigned UUID, opaque; stays the same across restarts |
| `title` | string | stored trimmed, 1 to 200 characters after trimming |
| `due_date` | string `YYYY-MM-DD` or `null` | calendar date, past dates allowed |
| `created_at` | string | ISO 8601 UTC with `Z`, set by the server at creation, unchanged across restarts |

**`GET /tasks`** — no query parameters.

| Status | When | Body |
|---|---|---|
| `200` | database reachable | JSON array of task objects, oldest first, unpaged; `[]` when none |
| `503` | database unreachable or the query did not finish in time | `{"detail": "Tasks are temporarily unavailable"}` |

**`POST /tasks`** — body `application/json` `{"title": string, "due_date"?: string | null}`; unknown fields ignored. The web UI always sends both fields, `title` trimmed, `due_date` `null` when none was picked.

| Status | When | Body |
|---|---|---|
| `201` | valid input, fewer than 10,000 tasks stored | the task object |
| `422` | invalid `title` or `due_date` (unchanged) | FastAPI standard validation body, see Errors |
| `503` | database unreachable or the statement did not finish in time; nothing stored | `{"detail": "Tasks are temporarily unavailable"}` |
| `507` | 10,000 tasks already stored; nothing stored | `{"detail": "Task limit of 10000 reached"}` |

**`GET /ready`** — `200` `{"status": "ready"}` when a `SELECT 1` on the database succeeds; `503` `{"status": "not ready"}` otherwise. **`GET /health`** stays `200` regardless of the database, so the database being down never restarts the pod.

No events.

### Rules

No shared rules: the one shared input rule (title 1 to 200 characters after trimming, A5) cannot be checked by the rule engine, which needs a rejection body carrying the rule id, while the spec keeps the Task API's FastAPI `422` body unchanged (A1, non-goals); both repos test the rule through their own criteria instead.

The rule in words, identical in both repos: `title` is required, is trimmed of leading and trailing whitespace, and must then be 1 to 200 characters long (200 accepted, 201 rejected, empty and whitespace-only rejected). `due_date` is a real calendar date `YYYY-MM-DD` or `null`; the web UI enforces it by its date field.

### Errors

```yaml
errors:
  status: 422
  body:
    list: detail
    field: loc
    rule: type
    message: msg
  unlisted:
    message: "The task could not be saved. Please try again."
```

The `422` body is FastAPI's: `{"detail": [{"type": str, "loc": ["body", "<field>"], "msg": str, ...}]}`; the field is the last element of `loc` (`title` or `due_date`). `503` and `507` carry a single `{"detail": "<text>"}` object. How the web UI treats each answer:

| Call | Answer | Page shows |
|---|---|---|
| `GET /tasks` | `200` | the list, or "No tasks yet" for `[]` |
| `GET /tasks` | `503`, any other status, a body that is not an array, connection error, timeout | "Tasks cannot be loaded right now. Please try again later." instead of the list |
| `POST /tasks` | `201` | the returned task appended to the list, form emptied |
| `POST /tasks` | `422` naming `title` | "Title must be 1 to 200 characters" next to the title field, values kept |
| `POST /tasks` | `507` | "The task limit of 10,000 is reached.", values kept |
| `POST /tasks` | `503`, `422` naming anything else, any other status, connection error, timeout | "The task could not be saved. Please try again.", values kept |

### Timeouts

- **Web UI → Task API**, `GET /tasks` and `POST /tasks`: 5 seconds per call (connect plus response), from the Next.js server. No automatic retry (a retried `POST` could create a task twice); the user retries by reloading or submitting again. Down or slow: treated as unavailable, see Errors.
- **Task API → Postgres**: connect timeout 2 seconds, statement timeout 3 seconds, so the Task API answers `503` within 6 seconds when the database is down or slow, before the web UI's 5-second timeout in the usual case of a refused connection. The connection pool drops broken connections and reconnects on the next request, so the Task API recovers with no restart (A7).
- **`GET /ready`**: the database check gives up after 1 second and answers `503`; Kubernetes then removes the instance from the Service until the check succeeds again, and `/health` keeps the pod alive.

## gitops-app PR

The gitops entry lands before the component PRs merge: it adds `resources.postgres` (type database), `uses: [postgres]` and `replicas: 2` in dev on `taskboard22b-api`, and `TASK_API_URL` on `taskboard22b-web`; `devbox run render` regenerates the overlays and adds the CloudNativePG operator under `operators/` (applied once per cluster). No `gitops_pin` entries: both components run only in dev, and dev is pinned by `/system:ship` in its `compose/<spec>` PR; staging and prod follow by promotion.

After Argo reconciles dev, verify: the `taskboard22b-postgres` cluster is healthy and its `-app` Secret exists; 2 `taskboard22b-api` pods are ready with `DATABASE_URL` set; a task created through `POST /tasks` is listed by `GET /tasks` after the Task API pods restart (`/instance:verify dev` smoke); `http://taskboard22b-web.taskboard22b-dev.localhost:8088/` lists the tasks and creates one.
