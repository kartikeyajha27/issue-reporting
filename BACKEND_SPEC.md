# Backend Spec: Support Reports Dashboard

You are the backend agent. Build the data layer, metric engine, APIs and background jobs for the support reports dashboard of our AI-agent-building platform.

## 0. Ground rules

1. **The backend does all processing.** The frontend only calls APIs and renders. Every metric, delta, sentiment (good/bad/neutral), formatted display string, percentage for a bar and sort order must be computed here and returned in the response.
2. Reuse what already exists: the report form (title, details, screenshot), routing to Teams (internal users) or Jira (external users), and the assignment API. Extend them; do not rebuild them.
3. Database: PostgreSQL 14+. Store all timestamps as `timestamptz` in UTC. Bucket by day in a configurable timezone (`SUPPORT_TZ` env var).
4. All endpoints live under `/api/v1/support`, return JSON, and require authentication.
5. Match the response shapes in section 5 exactly. The frontend is built against them.

## 1. Roles and access

| Role | Can do |
|---|---|
| `viewer` (executives) | Read all dashboard and report endpoints. No mutations |
| `agent` | Read all. Update status and assignment of reports assigned to them |
| `lead` | Everything an agent can, plus assign or reassign any report |
| `admin` | Everything, plus manage SLA policies and categories |

Return `403` with the error format in section 6 when a role is not allowed. Return `allowed_actions` on report detail so the UI can hide buttons.

## 2. Database schema

### 2.1 Enums

```sql
CREATE TYPE report_source   AS ENUM ('internal', 'external');
CREATE TYPE report_status   AS ENUM ('new', 'in_progress', 'waiting_on_customer', 'resolved', 'closed');
CREATE TYPE report_priority AS ENUM ('low', 'medium', 'high', 'critical');
CREATE TYPE plan_tier       AS ENUM ('internal', 'free', 'pro', 'enterprise');
CREATE TYPE sla_state       AS ENUM ('on_track', 'at_risk', 'breached', 'met', 'missed');
CREATE TYPE member_role     AS ENUM ('viewer', 'agent', 'lead', 'admin');
CREATE TYPE actor_type      AS ENUM ('user', 'support_member', 'system', 'jira', 'teams');
```

### 2.2 `support_members`

| Column | Type | Notes |
|---|---|---|
| id | uuid PK | default `gen_random_uuid()` |
| user_id | uuid UNIQUE | FK to the existing platform users table |
| name | text NOT NULL | |
| email | citext UNIQUE NOT NULL | |
| role | member_role NOT NULL | default `agent` |
| capacity | int NOT NULL | max comfortable open reports, default 15 |
| is_active | boolean NOT NULL | default true |
| created_at | timestamptz | default `now()` |

### 2.3 `report_categories`

| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| slug | text UNIQUE NOT NULL | e.g. `deployment`, `agent_behavior`, `integration`, `billing`, `ui_bug`, `other` |
| name | text NOT NULL | display name |
| is_active | boolean | default true |

Seed the six slugs above.

### 2.4 `sla_policies`

| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| plan_tier | plan_tier NOT NULL | |
| priority | report_priority NOT NULL | |
| first_response_minutes | int NOT NULL | |
| resolution_minutes | int NOT NULL | |
| UNIQUE (plan_tier, priority) | | |

Seed defaults (editable by admin):

| Plan | Critical (first / resolve) | High | Medium | Low |
|---|---|---|---|---|
| enterprise | 30m / 4h | 1h / 8h | 4h / 24h | 8h / 72h |
| pro | 1h / 8h | 4h / 24h | 8h / 48h | 24h / 96h |
| free | 8h / 48h | 24h / 72h | 48h / 120h | 72h / 168h |
| internal | 4h / 24h | 8h / 48h | 24h / 72h | 48h / 120h |

### 2.5 `reports` (extend your existing table or create it)

| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| seq | bigint GENERATED ALWAYS AS IDENTITY | basis for `public_id` |
| public_id | text UNIQUE NOT NULL | `'SUP-' \|\| seq`, e.g. `SUP-1042` |
| title | text NOT NULL | max 200 chars |
| description | text NOT NULL | |
| category_id | uuid FK report_categories | nullable until triaged |
| priority | report_priority NOT NULL | default `medium` |
| status | report_status NOT NULL | default `new` |
| source | report_source NOT NULL | internal goes to Teams, external goes to Jira |
| reporter_user_id | uuid | FK to platform users |
| reporter_name | text | denormalised |
| reporter_email | text | |
| customer_org_id | uuid | FK to platform organisations, null for internal |
| customer_name | text | denormalised; use `Internal` for internal reports |
| plan_tier | plan_tier NOT NULL | snapshot at creation time |
| agent_id | uuid | the customer's AI agent involved, if any |
| external_system | text | `jira` or `teams` |
| external_ref | text | Jira key or Teams message id |
| external_url | text | |
| assignee_id | uuid FK support_members | null means unassigned |
| created_at | timestamptz NOT NULL | default `now()` |
| assigned_at | timestamptz | set on first assignment |
| first_response_at | timestamptz | see 4.1 |
| first_response_due_at | timestamptz NOT NULL | `created_at` + policy |
| resolution_due_at | timestamptz NOT NULL | `created_at` + policy + paused time |
| resolved_at | timestamptz | set when status becomes `resolved`; cleared on reopen |
| closed_at | timestamptz | |
| reopened_count | int NOT NULL | default 0 |
| last_reopened_at | timestamptz | |
| sla_state | sla_state NOT NULL | maintained by job, see 4.2 |
| sla_paused_at | timestamptz | set while `waiting_on_customer` |
| sla_paused_total_seconds | int NOT NULL | default 0 |
| resolution_note | text | |
| updated_at | timestamptz NOT NULL | default `now()` |

UNIQUE (`external_system`, `external_ref`) WHERE `external_ref` IS NOT NULL.

Indexes:

```sql
CREATE INDEX idx_reports_created        ON reports (created_at);
CREATE INDEX idx_reports_resolved       ON reports (resolved_at) WHERE resolved_at IS NOT NULL;
CREATE INDEX idx_reports_status_sla     ON reports (status, sla_state);
CREATE INDEX idx_reports_assignee_open  ON reports (assignee_id) WHERE status IN ('new','in_progress','waiting_on_customer');
CREATE INDEX idx_reports_source_created ON reports (source, created_at);
CREATE INDEX idx_reports_category       ON reports (category_id, created_at);
CREATE INDEX idx_reports_search         ON reports USING gin (to_tsvector('simple', title || ' ' || coalesce(customer_name,'') || ' ' || public_id));
```

### 2.6 `report_attachments`

| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| report_id | uuid FK reports ON DELETE CASCADE | |
| filename | text NOT NULL | |
| storage_key | text NOT NULL | object-store key; never return it directly |
| mime_type | text NOT NULL | |
| size_bytes | bigint NOT NULL | |
| created_at | timestamptz | |

Serve attachments through short-lived signed URLs in API responses.

### 2.7 `report_events` (the audit trail and detail timeline)

| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| report_id | uuid FK reports ON DELETE CASCADE | |
| event_type | text NOT NULL | `created`, `assigned`, `status_changed`, `first_response`, `comment`, `reopened`, `sla_at_risk`, `sla_breached`, `csat_received`, `synced_from_jira` |
| actor_type | actor_type NOT NULL | |
| actor_id | uuid | null for system |
| from_value | text | |
| to_value | text | |
| note | text | |
| created_at | timestamptz NOT NULL | default `now()` |

Index on (`report_id`, `created_at`). Every mutation must write an event in the same transaction.

### 2.8 `csat_responses`

| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| report_id | uuid UNIQUE FK reports | one response per report |
| token | text UNIQUE NOT NULL | random 32+ bytes, URL-safe |
| score | smallint | CHECK 1 to 5, null until answered |
| comment | text | |
| sent_at | timestamptz NOT NULL | |
| responded_at | timestamptz | |
| expires_at | timestamptz NOT NULL | `sent_at` + 14 days |

### 2.9 `support_daily_metrics` (rollup, phase 2)

| Column | Type | Notes |
|---|---|---|
| metric_date | date | in `SUPPORT_TZ` |
| source | report_source | |
| category_id | uuid | null means all categories |
| opened_count | int | |
| resolved_count | int | |
| reopened_count | int | |
| sla_met_count | int | |
| csat_sum | int | |
| csat_count | int | |
| open_at_end_of_day | int | |
| PRIMARY KEY | (metric_date, source, category_id) | |

Phase 1 computes everything live from `reports` with the indexes above and a 60 second cache. Move trend and sparkline queries to this table when volume demands it.

## 3. Metric definitions

Let `[start, end)` be the selected range and `[pstart, pend)` the immediately preceding range of equal length. All metrics honour the `source` filter (`all`, `internal`, `external`).

| KPI id | Label | Definition | Delta type | Good direction |
|---|---|---|---|---|
| `open` | Open reports | Reports with `created_at <= T` and (`resolved_at` is null or `resolved_at > T`) at `T = end`. Previous value uses `T = pend` | percent | down |
| `frt` | First response (median) | Median of `first_response_at - created_at` for reports created in range with a first response | percent | down |
| `ttr` | Resolution (median) | Median of `resolved_at - created_at` minus paused time, for reports resolved in range | percent | down |
| `sla` | SLA met | Reports resolved in range with `resolved_at <= resolution_due_at`, divided by all reports resolved in range | points | up |
| `reopen` | Reopen rate | Reports reopened in range divided by reports resolved in range | points | down |
| `csat` | CSAT | Mean `score` of responses with `responded_at` in range | absolute | up |

Computation notes:

```sql
-- median first response (seconds)
SELECT percentile_cont(0.5) WITHIN GROUP (ORDER BY extract(epoch FROM first_response_at - created_at))
FROM reports
WHERE created_at >= :start AND created_at < :end AND first_response_at IS NOT NULL
  AND (:source = 'all' OR source = :source::report_source);

-- median resolution (seconds, excluding paused time)
SELECT percentile_cont(0.5) WITHIN GROUP (ORDER BY extract(epoch FROM resolved_at - created_at) - sla_paused_total_seconds)
FROM reports
WHERE resolved_at >= :start AND resolved_at < :end
  AND (:source = 'all' OR source = :source::report_source);

-- SLA met
SELECT 100.0 * count(*) FILTER (WHERE resolved_at <= resolution_due_at) / NULLIF(count(*), 0)
FROM reports WHERE resolved_at >= :start AND resolved_at < :end;
```

### 3.1 Delta, sentiment and display

- `percent` delta: `(current - previous) / previous * 100`, rounded to 1 decimal. If `previous` is 0 or null, return `null`.
- `points` delta (rates): `current - previous`, 1 decimal.
- `absolute` delta (CSAT): `current - previous`, 1 decimal.
- `sentiment`: `neutral` if the absolute change is under 0.5 percent (or 0.1 points); otherwise `good` if the change moves in the good direction, else `bad`.
- `delta_display`: signed string with unit, e.g. `-8%`, `+2.1 pts`, `+0.2`.
- Duration display: under 1 hour `45m`; under 24 hours `1h 12m`; otherwise `2d 4h`.
- `spark`: split `[start, end)` into 8 equal buckets and compute the same metric per bucket (for `open`, the snapshot at each bucket end). Return 8 numbers; use the previous bucket value when a bucket has no data.

### 3.2 Headline

- `urgent_count`: open reports where `sla_state IN ('at_risk','breached')`.
- `unassigned_count`: open reports with `assignee_id IS NULL`.
- `headline.text`: `"{urgent_count} reports need action today"`, or `"No reports need urgent action"` when zero.
- `subtext`: `"{unassigned_count} have no owner."` followed by one sentence naming KPIs with `sentiment = good` ("Response times are improving") and `bad` ("resolution time and reopen rate are drifting up"). Build it from templates keyed by KPI id; omit a clause if none apply.

## 4. Business rules and background jobs

### 4.1 Lifecycle

| Event | Effects |
|---|---|
| Create | Set `status = new`, snapshot `plan_tier`, compute both due dates from `sla_policies`, `sla_state = on_track`, write `created` event, send to Teams (internal) or create Jira issue (external) as today, store `external_ref` |
| First support action | The first of: a support member's comment, or moving status from `new` to anything else. Set `first_response_at`, write `first_response` event. Never overwrite |
| Assign | Set `assignee_id`, set `assigned_at` if null, write `assigned` event with from and to member names. Assigning from `new` does not count as a response |
| `waiting_on_customer` | Set `sla_paused_at = now()`. On leaving that status, add the elapsed seconds to `sla_paused_total_seconds`, push `resolution_due_at` forward by the same amount, clear `sla_paused_at` |
| Resolve | Set `resolved_at`, set `sla_state` to `met` or `missed`, create the `csat_responses` row, send the CSAT email with a link `/csat/{token}` |
| Reopen (status leaves `resolved` or `closed`) | Increment `reopened_count`, set `last_reopened_at`, clear `resolved_at`, recompute `sla_state`, write `reopened` event |
| Close | Allowed from `resolved` only. Set `closed_at` |

Allowed transitions: `new` to `in_progress`; `in_progress` to `waiting_on_customer` or `resolved`; `waiting_on_customer` to `in_progress` or `resolved`; `resolved` to `closed` or `in_progress` (reopen). Anything else returns `409`.

### 4.2 SLA state job (every 5 minutes, and on every status change)

For open reports (not paused):

- `breached`: now is past `resolution_due_at`, or past `first_response_due_at` with no `first_response_at`.
- `at_risk`: less than 25 percent of the total SLA window remains.
- `on_track`: otherwise.

Write `sla_at_risk` and `sla_breached` events once per transition. Resolved reports are `met` or `missed` and never change again.

### 4.3 Other jobs

| Job | Schedule | Purpose |
|---|---|---|
| Jira sync | Webhook plus a 15 minute reconciliation sweep | Map Jira status, assignee and comments back into `reports` and `report_events`. Be idempotent using `external_ref` |
| CSAT reminder | Daily | Re-send once after 3 days if unanswered |
| Rollup | Hourly (phase 2) | Upsert `support_daily_metrics` for today and yesterday |
| Cache invalidation | On every mutation | Drop dashboard cache keys |

## 5. API contract

Base path `/api/v1/support`. Common query params for dashboard endpoints: `range` (`7d`, `14d`, `30d`, `90d`; default `7d`) and `source` (`all`, `internal`, `external`; default `all`). Reject unknown values with `422`.

### 5.1 `GET /dashboard/summary`

```json
{
  "range": "7d",
  "source": "all",
  "generated_at": "2026-10-27T09:30:00Z",
  "headline": { "urgent_count": 3, "unassigned_count": 2, "text": "3 reports need action today",
                "subtext": "2 have no owner. Response times are improving, but resolution time and reopen rate are drifting up." },
  "kpis": [
    { "id": "open", "label": "Open reports", "value": 42, "display": "42",
      "delta_value": -8.0, "delta_display": "-8%", "sentiment": "good",
      "spark": [58, 55, 57, 52, 49, 47, 45, 42] }
  ]
}
```

Return all six KPIs in this order: `open`, `frt`, `ttr`, `sla`, `reopen`, `csat`. `value` is numeric (seconds for durations, percent for rates).

### 5.2 `GET /dashboard/trend`

```json
{ "granularity": "day",
  "points": [ { "date": "2026-10-21", "label": "Oct 21", "opened": 34, "resolved": 30 } ] }
```

Daily points for ranges up to 30d. For `90d` use weekly points (`granularity: "week"`, label like `Oct 14 - 20`). Include zero-count days.

### 5.3 `GET /dashboard/categories`

```json
{ "items": [ { "id": "uuid", "name": "Deployment", "report_count": 64, "bar_pct": 100,
               "avg_resolution_hours": 12.4, "avg_resolution_display": "12.4h",
               "csat": 3.9, "csat_display": "3.9", "csat_sentiment": "bad" } ] }
```

Sort by `report_count` descending. `bar_pct` is count divided by the largest count times 100. `csat_sentiment`: below 4.0 `bad`, 4.0 to 4.39 `neutral`, 4.4 and above `good`. Return `csat: null` and `"-"` when there are no responses.

### 5.4 `GET /dashboard/team`

```json
{ "items": [ { "id": "uuid", "name": "Asha R.", "open_count": 14, "load_pct": 93, "load_sentiment": "bad",
               "resolved_count": 61, "first_response_display": "48m",
               "sla_met_display": "97%", "csat_display": "4.6" } ] }
```

`load_pct` is `open_count / capacity * 100`, capped at 100 for the bar; `load_sentiment` is `bad` above 80, otherwise `neutral`. Active members only. Sort by `open_count` descending.

### 5.5 `GET /reports`

Query params: `range`, `source`, `q` (search over id, title, customer), `status` (comma list; default all open statuses), `quick_filter` (`urgent`, `unassigned`, `reopened`, `breached`), `assignee_id`, `category_id`, `sort` (`public_id`, `title`, `customer_name`, `plan_tier`, `age_seconds`, `sla_state`, `assignee`), `order` (`asc`, `desc`; default `desc`), `page` (default 1), `page_size` (default 25, max 100). Whitelist sort fields; never interpolate raw input into SQL.

```json
{
  "items": [
    { "id": "uuid", "public_id": "SUP-1042", "title": "Deployment stuck at 'Building'",
      "customer_name": "Acme Corp", "plan_tier": "enterprise", "plan_label": "Enterprise",
      "source": "external", "category": { "id": "uuid", "name": "Deployment" },
      "status": "in_progress", "status_label": "In progress", "priority": "high",
      "age_seconds": 273600, "age_display": "3d 4h",
      "sla_state": "breached", "sla_label": "Breached",
      "assignee": { "id": "uuid", "name": "Vikram S." }, "created_at": "2026-10-24T05:10:00Z" }
  ],
  "page": 1, "page_size": 25, "total": 42
}
```

`assignee` is `null` when unassigned. `age_seconds` is time since creation for open reports (excluding paused time) and creation to resolution for resolved ones. Default order when `sort` is absent: SLA severity (breached, at risk, on track) then age descending.

### 5.6 `GET /reports/{id}`

Accepts the uuid or the `public_id`. Returns everything in the list item plus:

```json
{
  "description": "Build logs show a registry timeout.",
  "reporter": { "name": "Jane D.", "email": "jane@acme.com" },
  "external": { "system": "jira", "key": "PLAT-381", "url": "https://..." },
  "attachments": [ { "id": "uuid", "filename": "screenshot.png", "mime_type": "image/png", "url": "signed-url", "expires_at": "..." } ],
  "timeline": [ { "id": "uuid", "type": "assigned", "label": "Assigned to Vikram S.", "actor_name": "Asha R.", "at": "2026-10-24T06:00:00Z" } ],
  "csat": { "score": 4, "comment": "Quick fix" },
  "allowed_actions": { "can_assign": true, "can_change_status": true, "allowed_statuses": ["waiting_on_customer", "resolved"] }
}
```

`csat` is `null` until answered. Timeline is oldest first and `label` is fully composed server-side.

### 5.7 `PATCH /reports/{id}/assignment`

Body: `{ "assignee_id": "uuid" | null }`. Validates the member is active. Returns the updated list item (same shape as in 5.5). Also push the assignment to Jira or Teams as the existing assignment API does.

### 5.8 `PATCH /reports/{id}/status`

Body: `{ "status": "resolved", "resolution_note": "optional, required for resolved" }`. Enforces the transition table in 4.1. Returns the updated list item.

### 5.9 `POST /reports/{id}/comments`

Body: `{ "body": "text" }`. Writes a `comment` event (and sets `first_response_at` if it is the first support action). Returns the new timeline entry.

### 5.10 `GET /meta/filters`

```json
{ "ranges": [ {"value":"7d","label":"7 days"}, {"value":"14d","label":"14 days"} ],
  "sources": [ {"value":"all","label":"All"}, {"value":"external","label":"Customers (Jira)"}, {"value":"internal","label":"Internal (Teams)"} ],
  "members": [ {"id":"uuid","name":"Asha R."} ],
  "categories": [ {"id":"uuid","name":"Deployment"} ],
  "statuses": [ {"value":"new","label":"New"} ] }
```

### 5.11 `GET /reports/export.csv`

Same filters as 5.5, no pagination, streamed. Restricted to `lead`, `admin` and `viewer`.

### 5.12 CSAT (public, token-authenticated, no login)

- `GET /api/v1/csat/{token}` returns `{ "report_public_id": "SUP-1042", "title": "...", "already_answered": false, "expired": false }`.
- `POST /api/v1/csat/{token}` with `{ "score": 1-5, "comment": "optional" }`. Returns `409` if already answered and `410` if expired. Writes a `csat_received` event.

### 5.13 Existing report creation endpoint

Extend it to accept optional `category_slug` and `priority`, and to fill every field in 2.5. Existing clients must keep working: default `priority = medium`, and default `category` from `other` until triaged.

## 6. Conventions

- Error format: `{ "error": { "code": "invalid_transition", "message": "Cannot move from closed to new", "details": {} } }`. Status codes: `400`, `401`, `403`, `404`, `409`, `410`, `422`.
- Dashboard GET endpoints are cached 60 seconds per (`range`, `source`, role-independent key). Add `Cache-Control: private, max-age=30`.
- Rate limit the CSAT endpoints. Validate and sanitise all text; limit `comment` to 2,000 chars.
- Log every mutation with actor and request id.
- Performance targets: each dashboard endpoint under 300 ms at p95 with 500k reports; `GET /reports` under 200 ms.

## 7. Testing and acceptance

1. Unit tests for every KPI with fixed fixtures, including empty ranges (return `null` deltas, `"-"` displays, never errors).
2. Tests for SLA transitions including paused time, reopen and the 25 percent at-risk threshold.
3. Tests for each role's permissions on every mutation.
4. Idempotency test: replaying the same Jira webhook twice changes nothing.
5. Seed script generating about 2,000 reports across 90 days, all plans, sources and categories, with CSAT on roughly 60 percent of resolved ones.
6. Done when: all endpoints in section 5 return the documented shapes, a mutation is reflected in `/dashboard/summary` immediately (cache invalidated), and the frontend in `FRONTEND_SPEC.md` works against the API without client-side calculation.
