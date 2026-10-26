# Frontend Spec: Support Reports Dashboard (React)

You are the frontend agent. Build the support reports dashboard in our existing React portal.

Reference implementation for look, layout and interactions: `SupportDashboardV2.jsx` (mock data version). Keep its structure and behaviour, but replace the mock data with real API calls as described below. The API contract is defined in `BACKEND_SPEC.md` section 5; if the two ever disagree, the backend spec wins.

## 0. Ground rules

1. **No business logic in the frontend.** Do not calculate metrics, deltas, percentages, sentiment, sorting of server data, ages, or formatted durations. The backend returns display strings (`display`, `delta_display`, `age_display`, `label` fields), sentiments (`good`, `bad`, `neutral`) and bar widths (`bar_pct`, `load_pct`). Render exactly what you receive.
2. The only things the frontend owns: layout, interaction state, calling the API, loading, empty and error states, and accessibility.
3. Do not hard-code option lists (ranges, sources, members, categories, statuses). Load them from `GET /meta/filters`.
4. Use the portal's existing auth, router, HTTP client and design tokens where they exist. If there is no HTTP client, use `fetch` with the portal's auth header.
5. No new heavy dependencies. A small data-fetching helper (React Query or SWR) is fine if the portal already uses one; otherwise use the custom hook in section 3.

## 1. Page and route

- Route: `/support/reports`, visible to roles `viewer`, `agent`, `lead`, `admin`. Hide the nav item for other roles.
- Page title: "Support reports".
- Filter state lives in the URL query string (`?range=7d&source=all&q=&sort=&order=&quick=`), so views are shareable and survive refresh. Defaults: `range=7d`, `source=all`.

## 2. Component structure

```
src/features/support-dashboard/
  SupportDashboardPage.jsx     // page shell, reads filters from URL, composes panels
  api/supportApi.js            // all fetch calls (section 3)
  hooks/useApi.js              // loading/error/refetch helper
  components/
    TopBar.jsx                 // sticky header with segmented range and source controls
    Headline.jsx               // headline text and subtext
    KpiGrid.jsx / KpiCard.jsx  // 6 cards with sparkline, delta chip
    Sparkline.jsx              // small inline SVG from number[]
    TrendChart.jsx             // SVG line chart, hover crosshair and tooltip
    ReportsQueue.jsx           // search, sortable table, pagination
    ReportDrawer.jsx           // side panel with detail, timeline, assign, status
    CategoriesTable.jsx
    TeamTable.jsx
    Chip.jsx                   // good | bad | warn | neutral chips
    States.jsx                 // Skeleton, EmptyState, ErrorState
  support-dashboard.css        // tokens and styles (copy from the reference component)
```

Port the CSS variables, spacing and dark-mode rules from the reference component. Keep the design: sticky top bar, KPI cards with sparklines, tables for data, slide-in drawer.

## 3. API client

All calls go to `/api/v1/support`. Send the portal's auth header. Parse the error format `{ error: { code, message } }` and surface `message`.

| Function | Request | Used by |
|---|---|---|
| `getFilters()` | `GET /meta/filters` | TopBar, drawer assignee list, filters |
| `getSummary({range, source})` | `GET /dashboard/summary` | Headline, KpiGrid |
| `getTrend({range, source})` | `GET /dashboard/trend` | TrendChart |
| `getCategories({range, source})` | `GET /dashboard/categories` | CategoriesTable |
| `getTeam({range, source})` | `GET /dashboard/team` | TeamTable |
| `getReports({range, source, q, sort, order, page, page_size, quick_filter})` | `GET /reports` | ReportsQueue |
| `getReport(id)` | `GET /reports/{id}` | ReportDrawer |
| `assignReport(id, assigneeId)` | `PATCH /reports/{id}/assignment` | ReportDrawer |
| `updateStatus(id, status, note)` | `PATCH /reports/{id}/status` | ReportDrawer |
| `addComment(id, body)` | `POST /reports/{id}/comments` | ReportDrawer |
| `exportCsvUrl(params)` | `GET /reports/export.csv` | Export button |

`useApi(fn, deps)` returns `{ data, loading, error, refetch }`. It must cancel or ignore stale responses when filters change quickly (use `AbortController`).

Fire the summary, trend, categories, team and reports requests **in parallel** and let each panel load and fail independently. Never block the page on a single slow panel.

## 4. Data to UI mapping

### 4.1 Headline

| UI | Field |
|---|---|
| Large text | `summary.headline.text` |
| Muted line | `summary.headline.subtext` |

### 4.2 KPI cards (render in the order returned)

| UI | Field |
|---|---|
| Label | `kpi.label` |
| Big value | `kpi.display` |
| Sparkline | `kpi.spark` (8 numbers, scale to the card height; line colour from `kpi.sentiment`) |
| Chip text | `kpi.delta_display` followed by "vs previous" |
| Chip colour | `good` is green, `bad` is red, `neutral` is grey. If `delta_display` is null show "No prior data" in grey |

Clicking a card sets the `quick` filter for the queue where a mapping exists:

| Card id | Queue filter applied |
|---|---|
| `open` | `quick_filter` cleared (all open) |
| `sla` | `quick_filter=breached` |
| `reopen` | `quick_filter=reopened` |
| `frt`, `ttr`, `csat` | no filter, card only highlights |

Clicking the active card again clears the filter. Show the active filter as a removable chip above the queue.

### 4.3 Trend chart

- Plot `points[].opened` and `points[].resolved` against `points[].label`.
- Hover shows the nearest point: `"{label}: {opened} opened, {resolved} resolved"`. Also support keyboard focus with left and right arrows, and give the SVG an `aria-label` and a visually hidden data table fallback.
- Draw the y-axis scale from the max value; this is presentation, not business logic.

### 4.4 Reports queue

Columns: ID (`public_id`), Issue (`title`), Customer (`customer_name`), Plan (`plan_label`), Open for (`age_display`, right aligned), SLA (chip from `sla_label`; red for `breached`, amber for `at_risk`, green otherwise), Owner (`assignee.name`; if `null` show an amber "Unassigned" chip).

- Sorting is **server side**. Clicking a header sets `sort` and `order` in the URL and refetches. Only these headers are sortable: ID, Issue, Customer, Plan, Open for, SLA, Owner. Map them to API sort keys `public_id`, `title`, `customer_name`, `plan_tier`, `age_seconds`, `sla_state`, `assignee`.
- Search is server side: debounce 300 ms, send as `q`, reset to page 1.
- Pagination: previous/next buttons and "Showing X to Y of `total`". Default `page_size=25`.
- Rows are focusable (`tabIndex=0`) and open the drawer on click or Enter. Put the report id in the URL (`?report=SUP-1042`) so the drawer is linkable and Back closes it.
- Export button: opens `exportCsvUrl` with the current filters. Show it only for roles `viewer`, `lead`, `admin`.

### 4.5 Report drawer

Fetch `getReport(id)` on open and show a skeleton while loading. Content:

| Section | Fields |
|---|---|
| Header | `public_id`, source label (Jira or Teams from `external.system`), link to `external.url` if present, Close button |
| Title and status | `title`, `status_label`, SLA chip |
| Description | `description` |
| Details list | Customer (`customer_name`, `plan_label`), Category, Open for (`age_display`), Reporter |
| Attachments | Thumbnails or links from `attachments[].url` (image preview for `image/*`) |
| Owner | Select populated from `filters.members`, current `assignee`. Disabled if `!allowed_actions.can_assign` |
| Actions | Buttons for each value in `allowed_actions.allowed_statuses`, e.g. "Mark resolved". Resolving opens a small inline note field (required) |
| Timeline | `timeline[]` rendered oldest to newest using `label`, `actor_name`, `at` formatted with the user's locale |
| CSAT | `csat.score` and `csat.comment` if not null |
| Comment box | Textarea and "Add comment" button, posts via `addComment` and appends the returned entry |

Behaviour: Esc and the scrim close the drawer and return focus to the originating row. Trap focus while open.

Mutations: use the **pessimistic** approach. Disable the control, call the API, and on success update the row in the queue from the returned list item and `refetch` summary, categories and team (their numbers changed). On failure keep the previous state and show the error message in an inline alert inside the drawer.

### 4.6 Categories and Team tables

| Table | Column to field |
|---|---|
| Categories | Category `name`; Share bar width `bar_pct` with count `report_count`; Hours to fix `avg_resolution_display`; CSAT chip `csat_display` coloured by `csat_sentiment` |
| Team | Member `name`; Open bar width `load_pct` coloured red when `load_sentiment = bad`, with count `open_count`; First reply `first_response_display`; SLA met `sla_met_display`; CSAT `csat_display` |

Show "-" for any display string that arrives empty or null.

## 5. States, errors and polish

| Situation | Behaviour |
|---|---|
| First load | Skeleton placeholders sized like the real content (no layout jump) |
| Filter change | Keep old data visible with reduced opacity until new data arrives; do not flash skeletons |
| Panel error | Replace only that panel with `ErrorState`: message from the API, "Try again" button calling `refetch` |
| 401 | Defer to the portal's existing logout or login redirect |
| 403 | Show "You do not have access to support reports" full page |
| No results in queue | "No reports match. Clear the search or change the source filter." with a Clear filters button |
| Empty dashboard (no data in range) | KPI cards show `-`, trend shows "No reports in this period" |
| Auto refresh | Refetch summary and queue every 60 seconds while the tab is visible (`document.visibilityState`); pause when hidden; show "Updated {time}" from `generated_at` with a manual refresh button |

## 6. Accessibility and responsiveness

- Everything operable by keyboard; visible focus rings; segmented controls use `aria-pressed`; drawer has `role="dialog"` and a label.
- Colour is never the only signal: SLA and sentiment chips always carry text.
- Respect `prefers-reduced-motion` and `prefers-color-scheme`.
- Works from 360px wide: KPI grid wraps, tables scroll horizontally inside their panel, drawer becomes full width.

## 7. Tests

1. Component tests with mocked API responses copied from the examples in `BACKEND_SPEC.md` section 5: each panel renders display strings verbatim, handles `null` deltas, `null` assignee and empty arrays.
2. Interaction tests: changing range or source triggers parallel refetches and updates the URL; sorting and search send the right query params; KPI click applies and clears `quick_filter`; drawer opens from `?report=`, Esc closes it, assign and status changes update the row.
3. Error tests: one failing endpoint shows an error in only its panel.
4. Run an accessibility check (axe) on the page and the open drawer.

## 8. Definition of done

- The page works against the real API with no mock data left in the code.
- No arithmetic on metric values appears anywhere in the feature (search the code for percentage or duration calculations; there should be none).
- Layout and interactions match `SupportDashboardV2.jsx`.
- Loading, empty and error states exist for every panel.
- Lighthouse accessibility score of 95 or higher on the page.
