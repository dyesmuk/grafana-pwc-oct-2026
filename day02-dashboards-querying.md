# Day 2 – Dashboards & Querying

**Previous:** [Day 1 – Observability, Grafana Setup & Data Sources](day01-observability-setup-datasources.md) | **Next:** [Day 3 – Dynamic Dashboards & Alerting](day03-dynamic-dashboards-alerting.md)

---

## Contents

1. [Learning Goals](#learning-goals)
2. [Concepts](#concepts)
   - 2.1 Dashboard Structure and Time Range
   - 2.2 Visualizations
   - 2.3 Units, Thresholds, Value Mappings and Overrides
   - 2.4 Dashboard Versions and the JSON Model
   - 2.5 PromQL: Rate, Aggregations and Percentiles
   - 2.6 SQL in Grafana: Macros
   - 2.7 LogQL Basics
   - 2.8 Explore and the Drilldown Apps
   - 2.9 Dashboard Design Guidelines
3. [Labs](#labs)
   - Lab 1: Query warm-up in Explore (PromQL, SQL, LogQL)
   - Lab 2: Build a Windows host monitoring dashboard
   - Lab 3: Build an application (RED) dashboard from PromQL, SQL and log queries
   - Lab 4: Dashboard versions, JSON model and Drilldown apps
4. [Exercises](#exercises)
5. [Troubleshooting](#troubleshooting)
6. [Quiz](#quiz)
7. [Summary & Cheat Sheet](#summary--cheat-sheet)
8. [Answers](#answers)

---

## Learning Goals

By the end of Day 2, you will be able to:

- Build dashboards with panels, rows and a sensible layout, and control the time range
- Choose the right visualization for each kind of data
- Use units, thresholds, value mappings and overrides to make panels readable
- Use dashboard versions to compare and restore changes, and read a dashboard's JSON model
- Write PromQL queries with `rate`, aggregations and `histogram_quantile`
- Write time-aware SQL queries with Grafana macros
- Write LogQL queries to filter, parse and count logs
- Investigate data with Explore and the Drilldown apps
- Build a host (USE) dashboard and an application (RED) dashboard

> **Before you start:** run `C:\grafana-lab\start-lab.ps1 -WithLoad`. All labs today need steady traffic from the load generator.

---

## Concepts

### 2.1 Dashboard Structure and Time Range

**Building blocks:**

| Element | What it is |
|---|---|
| **Dashboard** | A page of panels with a shared time range. Identified by a unique **UID** in its URL. Stored in a **folder**. |
| **Panel** | One visualization. It has one or more **queries**, a **visualization type**, and **options**. |
| **Row** | A horizontal group of panels, with a title. Rows can be **collapsed** to hide panels until needed. |
| **Grid layout** | Dashboards use a **24-column** grid. A panel's position and size (`x`, `y`, `w`, `h`) are stored as `gridPos`. |
| **Time picker** | Top right. Sets the time range for every panel, unless a panel overrides it. |
| **Refresh picker** | Next to the time picker. Sets auto-refresh (e.g. every 30s). |

**Anatomy of a panel:**

```
┌─ Panel ────────────────────────────────────────────────┐
│  Queries          what data to fetch (PromQL/SQL/LogQL) │
│  Transformations  reshape the result (Day 3)            │
│  Visualization    how to draw it (Time series, Stat…)   │
│  Field config     units, min/max, thresholds, mappings  │
│  Overrides        exceptions for specific series        │
└─────────────────────────────────────────────────────────┘
```

**Time range syntax:**

| Expression | Meaning |
|---|---|
| `now-1h` to `now` | Last 1 hour |
| `now-24h` to `now` | Last 24 hours |
| `now/d` to `now` | Today so far |
| `now-1d/d` to `now-1d/d` | Yesterday (whole day) |
| `now/w` to `now` | This week so far |
| `now-7d` to `now` | Last 7 days |

`/d`, `/w`, `/M` round down to the start of the day, week or month.

**Panel-level time overrides** (panel editor → **Query options**):

| Option | Example | Effect |
|---|---|---|
| **Relative time** | `7d` | This panel always shows the last 7 days, whatever the dashboard time range |
| **Time shift** | `1d` | This panel shows the same range, one day earlier (compare with yesterday) |
| **Min interval** | `1m` | The smallest step Grafana may use for this query |
| **Max data points** | `500` | Upper limit on points per series; controls the step on wide ranges |

**Built-in interval variables:**

| Variable | Meaning | Typical use |
|---|---|---|
| `$__interval` | Step Grafana picks for the current time range and panel width | SQL `$__timeGroup`, LogQL ranges |
| `$__rate_interval` | A safe window for `rate()`: at least 4× the scrape interval | Every PromQL `rate()` / `increase()` in a dashboard |
| `$__range` | The whole dashboard time range (e.g. `1h`) | "Total over the selected period" |

> **Rule of thumb:** in dashboards, write `rate(x[$__rate_interval])`, not `rate(x[5m])`. It adapts to the time range and never becomes too short for the scrape interval. This is why we set the scrape interval on the Prometheus data source on Day 1.

### 2.2 Visualizations

| Visualization | Best for | Example in our labs |
|---|---|---|
| **Time series** | Values changing over time | Requests/sec per route, CPU % over time |
| **Stat** | One big number, optionally with a sparkline | Current error %, uptime |
| **Gauge** | One value against a known min/max | Memory used % (0–100) |
| **Bar gauge** | Several values against the same scale | Disk used % per volume |
| **Bar chart** | Comparing categories (non-time X axis) | Revenue per region |
| **Table** | Detailed rows and columns | Latest orders |
| **Pie chart** | Share of a total (few slices) | Share of HTTP status codes |
| **Heatmap** | Distribution over time | Latency distribution from histogram buckets |
| **State timeline** | Discrete states over time | Target up/down, service running/stopped |
| **Status history** | Periodic status per item, in a grid | Similar to state timeline, for many items |
| **Logs** | Log lines | Error and warning logs of orders-api |
| **Text** | Notes, instructions, links (Markdown/HTML) | Dashboard description, runbook links |

**Choosing a visualization:**

```
Is it text (log lines)?                ──► Logs
Is it a single current value?          ──► Stat  (or Gauge if min/max are meaningful)
Is it a value over time?               ──► Time series
Is it categories compared?             ──► Bar chart / Bar gauge
Is it a share of a whole (≤ 6 parts)?  ──► Pie chart (donut)
Is it a distribution over time?        ──► Heatmap
Is it on/off or named states?          ──► State timeline
Does the reader need exact details?    ──► Table
```

> **Tip:** Pie charts are hard to read with many slices. For more than about 6 categories, use a bar chart or bar gauge.

### 2.3 Units, Thresholds, Value Mappings and Overrides

These settings live in the panel editor's options pane (right side). The **All** tab holds settings for every field. The **Overrides** tab holds exceptions.

**Units:** always set them. `0.253` means nothing; `253 ms` does.

| Data | Unit to choose | Note |
|---|---|---|
| Percentage 0–100 | Misc → **Percent (0-100)** | Our CPU % queries |
| Ratio 0.0–1.0 | Misc → **Percent (0.0-1.0)** | Shows 0.25 as 25% |
| Bytes | Data → **bytes(IEC)** | 1 KiB = 1024 bytes |
| Bytes per second | Data rate → **bytes/sec(IEC)** | Disk and network throughput |
| Seconds | Time → **seconds (s)** | Auto-scales: 0.25 s shows as 250 ms, 90000 s as 1.04 day |
| Requests per second | Throughput → **requests/sec (rps)** | Request rate |
| Plain count | Misc → **short** | Adds K, M suffixes |
| Currency | Currency → e.g. **Indian Rupee (₹)** | Revenue |

Related settings in **Standard options**: **Min**, **Max**, **Decimals**, **Display name**, **Color scheme**.

**Thresholds:** turn numbers into colours.

- **Absolute** thresholds use real values, e.g. `Base = green`, `80 = orange`, `90 = red`.
- **Percentage** thresholds are relative to Min/Max.
- **Show thresholds** (Time series option) draws them as lines or filled regions on the graph.

**Value mappings:** replace values with text and colour.

| Mapping type | Example |
|---|---|
| **Value** | `1` → "UP" (green), `0` → "DOWN" (red) |
| **Range** | `0–50` → "Low" |
| **Regex** | `PAYMENT_.*` → "Payment issue" |
| **Special** | `null` / `NaN` → "No traffic" |

**Overrides:** change settings for **some** fields only. You pick fields with a matcher, then add properties.

| Matcher | Example use |
|---|---|
| **Fields with name** | Make the `errors` series red |
| **Fields with name matching regex** | All series matching `5..` get a red colour |
| **Fields with type** | All number fields get 2 decimals |
| **Fields returned by query** | Query B (p99) drawn as a dashed line |

Common override properties:
- Color scheme (fixed colour)
- Axis placement (right axis)
- **Graph styles → Transform → Negative Y** (mirror a series below the axis, e.g. disk writes under disk reads)
- Line style (dashed)
- Unit

### 2.4 Dashboard Versions and the JSON Model

**Versions:**
- Every **Save** creates a new version.
- **Dashboard settings → Versions** lists them with the author, date and your save message.
- You can:
  - **Compare** any two versions (a diff, with a JSON view);
  - **Restore** an older version. The restore itself becomes a new version, so nothing is lost.

> **Tip:** Always write a short save message ("Added p99 latency panel"). It makes the version list useful.

**JSON model:** every dashboard is a JSON document. **Dashboard settings → JSON Model** shows it. Key parts:

```json
{
  "uid": "a1b2c3d4",
  "title": "orders-api – Service Overview",
  "tags": ["orders-api", "red"],
  "time": { "from": "now-1h", "to": "now" },
  "refresh": "30s",
  "schemaVersion": 41,
  "version": 7,
  "templating": { "list": [] },
  "panels": [
    {
      "type": "timeseries",
      "title": "Requests / sec by route",
      "gridPos": { "x": 0, "y": 0, "w": 12, "h": 8 },
      "datasource": { "type": "prometheus", "uid": "..." },
      "targets": [
        { "refId": "A", "expr": "sum by (route) (rate(http_requests_total[$__rate_interval]))", "legendFormat": "{{route}}" }
      ],
      "fieldConfig": {
        "defaults": { "unit": "reqps", "thresholds": { "steps": [] } },
        "overrides": []
      },
      "options": { "legend": { "displayMode": "table", "placement": "bottom" } }
    }
  ]
}
```

| Key | Meaning |
|---|---|
| `uid` | Stable identifier, used in URLs and links |
| `version` | Increments on every save |
| `panels[].gridPos` | Position and size on the 24-column grid |
| `panels[].targets` | The queries (`expr` for PromQL, `rawSql` for SQL) |
| `fieldConfig.defaults` | Units, thresholds, mappings for all fields |
| `fieldConfig.overrides` | The Overrides tab |
| `templating` | Variables (Day 3) |

The JSON model is what you **export**, **import**, keep in **Git**, and **provision** from files (Day 5).

### 2.5 PromQL: Rate, Aggregations and Percentiles

**Selectors:** pick series by metric name and labels.

```promql
http_requests_total                                         # all series of this metric
http_requests_total{job="orders-api"}                       # exact match
http_requests_total{status=~"5.."}                          # regex match (5xx)
http_requests_total{route!="/health"}                       # not equal
http_requests_total{method="GET", status!~"2.."}            # several matchers (AND)
```

| Matcher | Meaning |
|---|---|
| `=` | Equals |
| `!=` | Not equal |
| `=~` | Matches regex (anchored: `5..` means the whole value) |
| `!~` | Does not match regex |

**Instant vector vs range vector:**
- `http_requests_total` is an **instant vector**: one value per series, at each evaluation time.
- `http_requests_total[5m]` is a **range vector**: all samples of the last 5 minutes per series. You can't graph it directly. You pass it to a function like `rate()`.

**Counters need `rate()`:** a raw counter only goes up, which isn't useful to graph. You want its **speed**.

| Function | Returns | Use for |
|---|---|---|
| `rate(c[w])` | Per-second average rate over window `w` | Dashboards and alerts (smooth) |
| `irate(c[w])` | Per-second rate from the **last two** samples | Very spiky, detailed graphs (rarely needed) |
| `increase(c[w])` | Total increase over `w` (= `rate × seconds`) | "How many in the last hour" |

`rate()` and `increase()` handle **counter resets** (e.g. app restarts) automatically.

**Aggregation operators:** combine series.

```promql
sum(rate(http_requests_total[5m]))                          # total req/s, all series
sum by (route) (rate(http_requests_total[5m]))              # req/s per route
sum without (instance) (rate(http_requests_total[5m]))      # drop one label, keep the rest
avg by (mode) (rate(windows_cpu_time_total[5m]))            # average across cores per mode
max by (volume) (windows_logical_disk_free_bytes)           # per volume
count(up == 1)                                              # number of targets up
topk(3, sum by (route) (rate(http_requests_total[5m])))     # 3 busiest routes
```

> **Order matters:** **rate first, then sum**: `sum(rate(x[5m]))`. Never `rate(sum(x))`, which breaks counter-reset handling.

**Arithmetic and ratios:**

```promql
# Error percentage (5xx as % of all requests)
100 * sum(rate(http_requests_total{status=~"5.."}[5m]))
    / sum(rate(http_requests_total[5m]))

# Memory used %
100 * (1 - windows_memory_available_bytes / windows_memory_physical_total_bytes)
```

> ⚠️ **Watch out, empty numerator:** if there have been **no** 5xx responses since the app started, the 5xx series doesn't exist yet. The whole ratio then returns *No data* instead of 0. Fix it with `or vector(0)`:
> ```promql
> 100 * (sum(rate(http_requests_total{status=~"5.."}[5m])) or vector(0))
>     / sum(rate(http_requests_total[5m]))
> ```

**Percentiles from histograms:** a histogram metric such as `http_request_duration_seconds` is exposed as three families:

```
http_request_duration_seconds_bucket{le="0.1"}   # cumulative count of requests ≤ 0.1 s
http_request_duration_seconds_bucket{le="0.25"}  # … ≤ 0.25 s
http_request_duration_seconds_bucket{le="+Inf"}  # all requests
http_request_duration_seconds_sum                # total seconds spent
http_request_duration_seconds_count              # number of requests
```

```promql
# p95 latency across all routes
histogram_quantile(0.95,
  sum by (le) (rate(http_request_duration_seconds_bucket[5m])))

# p95 latency per route (keep 'le' AND the label you want)
histogram_quantile(0.95,
  sum by (le, route) (rate(http_request_duration_seconds_bucket[5m])))

# Average latency
sum(rate(http_request_duration_seconds_sum[5m]))
  / sum(rate(http_request_duration_seconds_count[5m]))
```

Things to know about `histogram_quantile`:
- Always keep `le` in the `by (...)` clause.
- The result is an **estimate**, interpolated within buckets. Accuracy depends on the bucket boundaries; our app's buckets are 10 ms to 5 s.
- With no traffic in the window, the result is `NaN` (shown as *No data*).

**Comparing with the past:**

```promql
sum(rate(http_requests_total[5m]))                          # now
sum(rate(http_requests_total[5m] offset 1h))                # 1 hour ago
```

### 2.6 SQL in Grafana: Macros

SQL data sources (PostgreSQL, MySQL, MS SQL) don't know Grafana's time range by themselves. **Macros** insert it into your SQL before it runs.

| Macro | Expands to (PostgreSQL) | Use |
|---|---|---|
| `$__timeFilter(col)` | `col BETWEEN '2026-10-05T09:00:00Z' AND '2026-10-05T10:00:00Z'` | Limit rows to the dashboard time range. **Use in every query.** |
| `$__timeGroup(col, $__interval)` | `floor(extract(epoch from col)/60)*60` (for a 1m interval) | Bucket rows into time intervals |
| `$__timeGroupAlias(col, $__interval)` | Same, plus `AS "time"` | Same, and names the column `time` |
| `$__timeGroupAlias(col, $__interval, 0)` | Same, and fills empty buckets with 0 | Continuous lines without gaps |
| `$__timeFrom()` / `$__timeTo()` | Start / end of the time range | Custom conditions |
| `$__unixEpochFilter(col)` | Filter on a Unix-seconds column | Tables that store epoch numbers |

**Query formats:**

| Format | Grafana expects | Use for |
|---|---|---|
| **Time series** | A column named `time`, one or more number columns, optional string columns (become series names). Sorted by time. | Time series, heatmap |
| **Table** | Any columns | Table, bar chart, stat, pie chart |

**Example 1: orders per interval, one series per status (Time series format).**

```sql
SELECT
  $__timeGroupAlias(created_at, $__interval),
  status,
  count(*) AS orders
FROM orders
WHERE $__timeFilter(created_at)
GROUP BY 1, 2
ORDER BY 1;
```

**Example 2: revenue per region (Table format).**

```sql
SELECT region, round(sum(amount), 2) AS revenue
FROM orders
WHERE $__timeFilter(created_at)
GROUP BY region
ORDER BY revenue DESC;
```

> **Tip:** To see the SQL after macro expansion, open **Query inspector → Query** tab and click **Refresh**. This is the fastest way to debug SQL panels.

> **Performance:** `$__timeFilter` on an **indexed** time column is essential for large tables. Our `orders` table has an index on `created_at`.

### 2.7 LogQL Basics

A LogQL query always starts with a **stream selector** (labels in `{}`), followed by an optional **pipeline**.

```
{job="orders-api"}  |= "order"  | json  | status >= 500  | line_format "{{.route}} {{.status}}"
└─ stream selector ┘ └ line filter┘ └parser┘ └─ label filter ─┘ └──────── formatting ──────────┘
```

**1. Stream selector:** uses indexed labels only; fast.

```logql
{job="orders-api"}
{service_name="orders-api"}
{job=~"orders-api|payments"}
```

**2. Line filters:** plain text search on the raw line; fast. Put them **early**.

| Filter | Meaning | Example |
|---|---|---|
| `\|=` | Line contains | `{job="orders-api"} \|= "order created"` |
| `!=` | Line does not contain | `{job="orders-api"} != "/health"` |
| `\|~` | Line matches regex | `{job="orders-api"} \|~ "PAYMENT_FAILED\|invalid"` |
| `!~` | Line does not match regex | `{job="orders-api"} !~ "GET /api/products"` |

**3. Parsers:** turn the line into labels you can filter on.

| Parser | For | Example |
|---|---|---|
| `\| json` | JSON logs (our app) | Extracts `level`, `route`, `status`, `duration_ms`, … |
| `\| logfmt` | `key=value` logs (Grafana's own log) | Extracts `level`, `logger`, `msg`, … |
| `\| pattern "<pattern>"` | Fixed text layouts | `\| pattern "<ip> - - <_> \"<method> <path> <_>\" <status>"` |
| `\| regexp "<re>"` | Anything else | Named groups become labels |

**4. Label filters:** after a parser.

```logql
{job="orders-api"} | json | level="error"
{job="orders-api"} | json | status >= 400 and route="/api/orders"
{job="orders-api"} | json | duration_ms > 1000
```

**5. Formatting:** changes what is displayed.

```logql
{job="orders-api"} | json | line_format "{{.method}} {{.route}} → {{.status}} in {{.duration_ms}} ms"
```

**Metric queries:** turn logs into numbers (for time series panels and alerts).

```logql
# Log lines per second, per level
sum by (level) (rate({job="orders-api"} | json [5m]))

# Number of error lines in each interval
sum(count_over_time({job="orders-api"} | json | level="error" [$__interval]))

# p95 request duration from logs, per route
quantile_over_time(0.95,
  {job="orders-api"} | json | unwrap duration_ms | __error__="" [5m]) by (route)
```

- `unwrap` uses a numeric label as the sample value.
- `__error__=""` drops lines that failed parsing or unwrapping. Here that's lines without `duration_ms`, such as "order created".

> **Performance:** Loki reads every line matched by the stream selector in the time range. Narrow selectors, short time ranges and early line filters keep queries fast. **Never** use high-cardinality values (user IDs, order IDs) as stream labels.

### 2.8 Explore and the Drilldown Apps

**Explore** is for ad-hoc investigation, without building a dashboard.

| Feature | How | Why |
|---|---|---|
| **Code / Builder** | Toggle in the query row | Builder helps you learn; Code is faster once you know the language |
| **Split view** | **Split** button | Compare two queries or data sources side by side (synced time) |
| **Query history** | **Query history** button | Re-run or star earlier queries |
| **Query inspector** | **Query inspector** button | See the executed query, raw data, timing and response size |
| **Add to dashboard** | **Add → Add to dashboard** | Turn a good query into a panel |
| **Share link** | **Share** | Send a colleague the exact query and time range |

**Drilldown apps** give **query-less** exploration. You click through the data instead of writing queries. Open them from **Menu → Drilldown**.

| App | Data | What it shows |
|---|---|---|
| **Metrics** | Prometheus | Browse all metrics as small graphs; break a metric down by any label; find related metrics |
| **Logs** | Loki | Services with their log volume; logs split by level; detected **fields**; recurring **patterns** |
| **Traces** | Tempo | Rate, errors and duration of spans; slowest traces (Day 4) |

> **Note:** Logs Drilldown needs `volume_enabled: true` and the pattern ingester in Loki. We enabled both in `loki-config.yaml` on Day 1.

### 2.9 Dashboard Design Guidelines

1. **One purpose per dashboard.** "Is orders-api healthy?" is a good purpose; "Everything about everything" is not.
2. **Most important information top-left.** Put the key Stat panels (RED) in the first row.
3. **Overview first, detail below.** Summary stats → trends → breakdowns → logs and tables.
4. **Consistent colours:** green = good, orange = warning, red = bad. Errors are always red.
5. **Always set units**, and set Min to 0 where negative values make no sense.
6. **Limit series per panel.** More than about 10 lines makes a graph unreadable; use `topk` or a table.
7. **Use rows** to group, and collapse the detail rows.
8. **Name panels as questions or answers:** "Error % (5xx)" rather than "Panel 7".
9. **Add a description** to each non-obvious panel. It appears as an ⓘ tooltip.
10. **Avoid expensive queries** on dashboards that auto-refresh: long ranges × short intervals × many series.

---

## Labs

> **Before you start:**
> - Run `C:\grafana-lab\start-lab.ps1 -WithLoad` and wait for all ports to report `OK`.
> - All dashboards you build today go into the **`Lab Dashboards`** folder you created on Day 1.
> - **Save often** (`Ctrl+S`), with a short save message.

### Lab 1: Query warm-up in Explore (PromQL, SQL, LogQL)

**Goal:** Build each query step by step in Explore, so you understand what every part does before you use it in a dashboard.

**Time:** about 45 minutes

#### Step 1.1: PromQL, from raw counter to percentile

1. Open **Explore**, select `Prometheus`, switch to **Code**, and set the time range to **Last 30 minutes**.
2. Run each query in order. After each one, read the result and the legend before moving on.

| # | Query | What you should notice |
|---|---|---|
| 1 | `http_requests_total` | Many series, one per `method`/`route`/`status` combination. The lines only go **up**: that's a counter. |
| 2 | `http_requests_total{route="/api/orders", method="POST"}` | Fewer series: only order creation, split by status (201, 400). |
| 3 | `rate(http_requests_total{route="/api/orders", method="POST"}[5m])` | Now the lines are flat-ish: requests **per second**. |
| 4 | `sum(rate(http_requests_total[5m]))` | One line: total req/s of the app (about 5 with the load generator). |
| 5 | `sum by (route) (rate(http_requests_total[5m]))` | One line per route. |
| 6 | `sum by (status) (rate(http_requests_total[5m]))` | One line per status code: 200, 201, 400, 404. No 5xx yet. |
| 7 | `topk(2, sum by (route) (rate(http_requests_total[5m])))` | Only the 2 busiest routes. |
| 8 | `http_request_duration_seconds_bucket{route="/api/slow"}` | Histogram buckets: one series per `le` value. |
| 9 | `histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))` | One line: p95 latency in seconds, for all routes. |
| 10 | `histogram_quantile(0.95, sum by (le, route) (rate(http_request_duration_seconds_bucket[5m])))` | p95 per route. `/api/slow` is far above the rest (around 2.4 s). |

3. Now make the legend readable:
   - Re-run query 5.
   - Under the query, open **Options → Legend**, choose **Custom**, and enter `{{route}}`.

✅ **Expected:** legend entries show just the route, e.g. `/api/orders`, instead of the full label set.

4. Try the empty-numerator problem from Concepts 2.5:

```promql
100 * sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))
```

✅ **Expected:** *No data*. There have been no 5xx responses yet.

   Now run the fixed version:

```promql
100 * (sum(rate(http_requests_total{status=~"5.."}[5m])) or vector(0)) / sum(rate(http_requests_total[5m]))
```

✅ **Expected:** a flat line at 0.

#### Step 1.2: PromQL on host metrics

Run these and note what each returns:

| # | Query | Returns |
|---|---|---|
| 1 | `rate(windows_cpu_time_total{mode="idle"}[2m])` | Idle fraction per core (0–1) |
| 2 | `100 - (avg(rate(windows_cpu_time_total{mode="idle"}[2m])) * 100)` | Overall CPU busy % |
| 3 | `windows_memory_available_bytes / 1024 / 1024 / 1024` | Available memory in GiB |
| 4 | `windows_logical_disk_free_bytes` | Free bytes per `volume` (e.g. `C:`, plus hidden system volumes) |
| 5 | `sum by (nic) (rate(windows_net_bytes_received_total[2m]))` | Received bytes/sec per network interface |
| 6 | `time() - windows_system_boot_time_timestamp` | Uptime in seconds |

> **Tip:** Unsure which metrics exist? In Code mode, start typing `windows_` and use auto-complete. Or use the **Metrics browser** button.

#### Step 1.3: SQL with macros

1. Select `PostgreSQL-Orders`, switch to **Code**, and set the time range to **Last 7 days**.
2. Set **Format** to **Time series** and run:

```sql
SELECT
  $__timeGroupAlias(created_at, $__interval),
  count(*) AS orders
FROM orders
WHERE $__timeFilter(created_at)
GROUP BY 1
ORDER BY 1;
```

✅ **Expected:** a graph of orders per interval over 7 days.

3. Click **Query inspector → Query** and **Refresh**. Read the executed SQL:
   - `$__timeFilter` became a `BETWEEN '…' AND '…'` condition.
   - `$__timeGroupAlias` became `floor(extract(epoch from created_at)/…)*… AS "time"`.
4. Change the time range to **Last 6 hours** and re-inspect. Both the range and the grouping interval change automatically.
5. Add a series per status. Change the query to:

```sql
SELECT
  $__timeGroupAlias(created_at, $__interval),
  status,
  count(*) AS orders
FROM orders
WHERE $__timeFilter(created_at)
GROUP BY 1, 2
ORDER BY 1;
```

✅ **Expected:** two lines, `PAID` and `PAYMENT_FAILED`.

6. Switch **Format** to **Table** and run:

```sql
SELECT product, count(*) AS orders, round(sum(amount), 2) AS revenue
FROM orders
WHERE $__timeFilter(created_at)
GROUP BY product
ORDER BY revenue DESC;
```

✅ **Expected:** a table with 5 products.

#### Step 1.4: LogQL, from lines to metrics

1. Select `Loki`, switch to **Code**, and set the time range to **Last 15 minutes**.
2. Run each query in order:

| # | Query | What you should notice |
|---|---|---|
| 1 | `{job="orders-api"}` | Raw JSON lines |
| 2 | `{job="orders-api"} \|= "order created"` | Only order-creation lines |
| 3 | `{job="orders-api"} \| json` | Expand a line: every JSON key is now a label |
| 4 | `{job="orders-api"} \| json \| level="warn"` | 400/404 requests (logged as `warn`) |
| 5 | `{job="orders-api"} \| json \| duration_ms > 1000` | Slow requests only (mostly `/api/slow`) |
| 6 | `{job="orders-api"} \| json \| route="/api/orders" \| line_format "{{.method}} {{.status}} {{.duration_ms}}ms"` | Compact, readable lines |
| 7 | `sum by (level) (count_over_time({job="orders-api"} \| json [1m]))` | A graph: lines per minute, per level |
| 8 | `quantile_over_time(0.95, {job="orders-api"} \| json \| unwrap duration_ms \| __error__="" [5m]) by (route)` | p95 duration per route, calculated from **logs** |

3. Compare query 8 with PromQL query 10 from Step 1.1. The values should be similar, since both measure the same requests.

> **Discussion:** If both give the same answer, which would you use on a dashboard?
> - **Metrics:** cheaper and faster, the right choice for dashboards and alerts.
> - **Logs:** the fallback when an app has no metrics, and the way to see the *individual* slow requests.

---

### Lab 2: Build a Windows host monitoring dashboard

**Goal:** Build a USE-style host dashboard from scratch, using Stat, Gauge, Bar gauge, Time series and State timeline panels, with proper units, thresholds, mappings and overrides.

**Time:** about 90 minutes

**What you'll build:**

```
┌──────────────────────────── Windows Host ─────────────────────────────┐
│ Row: Overview                                                         │
│ [Uptime][CPU %][Memory % gauge][Disk used % bar gauge ][Processes]    │
│ Row: CPU                                                              │
│ [CPU % by mode – stacked      ][Processor queue length             ]  │
│ Row: Memory                                                           │
│ [Memory used vs available     ][Committed vs commit limit          ]  │
│ Row: Disk                                                             │
│ [Disk read / write (neg-Y)    ][Disk queue                         ]  │
│ Row: Network & Services                                               │
│ [Network in / out (neg-Y)     ][Service state – state timeline     ]  │
└───────────────────────────────────────────────────────────────────────┘
```

#### Step 2.1: Create the dashboard

1. Go to **Dashboards → New → New dashboard**.
2. Click **Settings** (top toolbar) and set:
   - Title: `Windows Host`
   - Tags: `windows`, `host`, `use`
   - Folder: `Lab Dashboards`
3. Click **Save dashboard**, with the message `Created`.
4. Set the time picker to **Last 1 hour** and the refresh picker to **30s**. Save again.

#### Step 2.2: Row "Overview": Stat, Gauge and Bar gauge panels

1. Click **Add → Row** and set its title to `Overview`.
2. Create the panels in the table below. For each one:
   - Click **Add → Visualization** and select `Prometheus`.
   - Switch the query to **Code** and enter the query.
   - Choose the visualization from the picker at the top right.
   - Set the options, then click **Back to dashboard**.

| Title | Visualization | Query | Options |
|---|---|---|---|
| **Uptime** | Stat | `time() - windows_system_boot_time_timestamp` | Unit: **seconds (s)**. Color mode: **None**. Graph mode: **None**. |
| **CPU %** | Stat | `100 - (avg(rate(windows_cpu_time_total{mode="idle"}[$__rate_interval])) * 100)` | Unit: **Percent (0-100)**. Decimals: 1. Thresholds: Base green, 70 orange, 90 red. Color mode: **Background gradient**. Graph mode: **Area**. |
| **Memory used %** | Gauge | `100 * (1 - windows_memory_available_bytes / windows_memory_physical_total_bytes)` | Unit: **Percent (0-100)**. Min 0, Max 100. Thresholds: Base green, 80 orange, 90 red. |
| **Disk used %** | Bar gauge | `100 * (1 - windows_logical_disk_free_bytes{volume!~"HarddiskVolume.*"} / windows_logical_disk_size_bytes{volume!~"HarddiskVolume.*"})` | Legend (query options): `{{volume}}`. Unit: **Percent (0-100)**. Min 0, Max 100. Thresholds: Base green, 80 orange, 90 red. Display mode: **Gradient**. Orientation: **Horizontal**. |
| **Processes** | Stat | `windows_system_processes` | Unit: **short**. Color mode: **None**. |

3. Arrange the five panels in one line under the row, by dragging their headers and bottom-right corners:
   - Uptime and Processes narrow (w = 3)
   - CPU % and Memory w = 5
   - Disk w = 8

✅ **Expected:** the uptime shows as a readable duration (e.g. `2.3 day`), the CPU stat changes colour with load, and the bar gauge shows one bar per real drive (e.g. `C:`).

> **Why `volume!~"HarddiskVolume.*"`?** Windows has hidden system and recovery volumes. They appear with names like `HarddiskVolume1` and are rarely useful on a dashboard.

#### Step 2.3: Row "CPU": stacked time series

1. **Add → Row** titled `CPU`.
2. Panel **CPU % by mode** (Time series):

```promql
100 * sum by (mode) (rate(windows_cpu_time_total{mode!="idle"}[$__rate_interval]))
    / scalar(count(windows_cpu_time_total{mode="idle"}))
```

   - Legend: `{{mode}}`
   - Unit: **Percent (0-100)**. Min 0.
   - **Graph styles → Stack series: Normal**. Fill opacity: 30.
   - Legend (options pane): Mode **Table**, Placement **Right**, Values **Mean**, **Max**.

> **How this query works:**
> - `count(windows_cpu_time_total{mode="idle"})` counts one idle series per logical core, which gives the number of cores.
> - Dividing the summed time by the number of cores gives % of the **whole** machine.
> - Stacked, the modes add up to total CPU %.

3. Panel **Processor queue length** (Time series):
   - Query: `windows_system_processor_queue_length`
   - Legend: `queue`. Unit **short**. Min 0.
   - Thresholds: Base green, 10 red.
   - **Show thresholds: As lines (dashed)**.
   - Description (Panel options): `Threads waiting for CPU. Sustained values above ~2 per core indicate CPU saturation (USE: S).`

#### Step 2.4: Row "Memory": two queries in one panel

1. **Add → Row** titled `Memory`.
2. Panel **Memory used vs available** (Time series), with two queries. Click **+ Add query** for query B.

| Query | Expression | Legend |
|---|---|---|
| A | `windows_memory_physical_total_bytes - windows_memory_available_bytes` | `used` |
| B | `windows_memory_available_bytes` | `available` |

   - Unit: **bytes(IEC)**. Min 0.
   - Stack series: **Normal**. Fill opacity 40.
   - Use **Overrides**:
     - Add field override **Fields with name** → `used`, property **Color scheme → Single color → red** (or orange).
     - Add another for `available` → **green**.

✅ **Expected:** the stack height equals your total RAM. The red area is what's in use.

3. Panel **Committed vs commit limit** (Time series):

| Query | Expression | Legend |
|---|---|---|
| A | `windows_memory_committed_bytes` | `committed` |
| B | `windows_memory_commit_limit` | `commit limit` |

   - Unit **bytes(IEC)**.
   - Override `commit limit` → **Line style: Dash**, **Fill opacity: 0**, colour **red**.

#### Step 2.5: Row "Disk": negative-Y override

1. **Add → Row** titled `Disk`.
2. Panel **Disk read / write** (Time series):

| Query | Expression | Legend |
|---|---|---|
| A | `sum by (volume) (rate(windows_logical_disk_read_bytes_total{volume!~"HarddiskVolume.*"}[$__rate_interval]))` | `{{volume}} read` |
| B | `sum by (volume) (rate(windows_logical_disk_write_bytes_total{volume!~"HarddiskVolume.*"}[$__rate_interval]))` | `{{volume}} write` |

   - Unit: **bytes/sec(IEC)**.
   - Override **Fields returned by query → B** → **Graph styles → Transform → Negative Y**.

✅ **Expected:** reads above the axis, writes mirrored below it, so both are easy to compare.

3. Panel **Disk queue** (Time series):
   - Query: `windows_logical_disk_requests_queued{volume!~"HarddiskVolume.*"}`
   - Legend `{{volume}}`. Unit **short**. Min 0.

#### Step 2.6: Row "Network & Services": state timeline with value mappings

1. **Add → Row** titled `Network & Services`.
2. Panel **Network in / out** (Time series):

| Query | Expression | Legend |
|---|---|---|
| A | `sum by (nic) (rate(windows_net_bytes_received_total[$__rate_interval])) > 0` | `{{nic}} in` |
| B | `sum by (nic) (rate(windows_net_bytes_sent_total[$__rate_interval])) > 0` | `{{nic}} out` |

   - Unit **bytes/sec(IEC)**.
   - Override query B → **Transform → Negative Y**.
   - The `> 0` filter hides idle virtual adapters.

3. Panel **Service state** (State timeline):

```promql
windows_service_state{name=~"windows_exporter|postgresql.*", state="running"}
```

   - Legend: `{{name}}`
   - **Value mappings:**
     - `1` → Display text `Running`, colour **green**
     - `0` → Display text `Stopped`, colour **red**
   - Show values: **Never**. Row height 0.8.

✅ **Expected:** one green bar per service.

4. Test it:
   - Open PowerShell **as Administrator** and run `Stop-Service windows_exporter`.
   - Wait 1–2 minutes, then run `Start-Service windows_exporter`.

✅ **Expected:** a **gap** in all host panels while the exporter is down. Prometheus couldn't scrape it, so there is no data at all. This shows why "no data" must be monitored too. Day 3 alerting covers this.

> **Note:** Stopping windows_exporter stops *all* host metrics, including its own service state. To see a red **Stopped** state in this panel, you'd stop a *different* monitored service. Don't stop PostgreSQL now, or orders-api will start failing.

#### Step 2.7: Finish and save

1. **Collapse** the `Disk` and `Network & Services` rows, so the dashboard opens with the overview first.
2. Save with the message `Host dashboard complete`.
3. Export the dashboard JSON (**Export → Export as JSON → Download file**) and save it as `C:\grafana-lab\dashboards\windows-host.json`.

---

### Lab 3: Build an application (RED) dashboard from PromQL, SQL and log queries

**Goal:** Build a service dashboard for orders-api that combines metrics (RED), business data (SQL) and logs, then prove it works by injecting faults.

**Time:** about 90 minutes

**What you'll build:**

```
┌──────────────────────── orders-api – Service Overview ────────────────────────┐
│ Row: RED                                                                      │
│ [Req/s][Error % (5xx)][p95 latency][In-flight gauge][Targets up – timeline ]  │
│ [Requests/sec by route                ][Responses by status class          ]  │
│ [Latency p50 / p95 / p99              ][Latency heatmap                    ]  │
│ [Status codes – pie                   ]                                       │
│ Row: Business (SQL)                                                           │
│ [Orders by status – time series       ][Revenue by region – bar chart      ]  │
│ [Payment failure %][Latest orders – table                                  ]  │
│ Row: Logs                                                                     │
│ [Log volume by level                  ][Warnings & errors – logs panel     ]  │
└───────────────────────────────────────────────────────────────────────────────┘
```

#### Step 3.1: Create the dashboard

1. **Dashboards → New → New dashboard**.
2. **Settings**:
   - Title `orders-api – Service Overview`
   - Tags `orders-api`, `red`
   - Folder `Lab Dashboards`
3. Time range **Last 1 hour**, refresh **30s**. Save.

#### Step 3.2: Row "RED": key numbers

1. **Add → Row** titled `RED`.
2. Add these five panels:

| Title | Visualization | Query (Prometheus) | Options |
|---|---|---|---|
| **Request rate** | Stat | `sum(rate(http_requests_total{job="orders-api"}[$__rate_interval]))` | Unit **requests/sec (rps)**. Decimals 2. Color mode **None**. Graph mode **Area**. |
| **Error % (5xx)** | Stat | `100 * (sum(rate(http_requests_total{job="orders-api", status=~"5.."}[$__rate_interval])) or vector(0)) / sum(rate(http_requests_total{job="orders-api"}[$__rate_interval]))` | Unit **Percent (0-100)**. Decimals 1. Thresholds: Base green, 1 orange, 5 red. Color mode **Background**. |
| **p95 latency** | Stat | `histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket{job="orders-api", route!="/api/slow"}[$__rate_interval])))` | Unit **seconds (s)**. Thresholds: Base green, 0.3 orange, 1 red. Color mode **Background**. |
| **In-flight requests** | Gauge | `sum(http_requests_in_flight{job="orders-api"})` | Unit **short**. Min 0, Max 20. Thresholds: Base green, 10 orange, 15 red. |
| **Targets up** | State timeline | `up` | Legend `{{job}}`. Value mappings: `1` → `UP` (green), `0` → `DOWN` (red). Show values **Never**. |

> **Why exclude `/api/slow` from p95?** It is deliberately slow (0.5–2.5 s). Including it would hide real regressions in the normal endpoints. Excluding known outliers is a common, documented dashboard decision. Say so in the panel **Description**.

#### Step 3.3: Row "RED": trends

Add these four panels below the stats:

**Panel "Requests/sec by route"** (Time series):
- Query: `sum by (route, method) (rate(http_requests_total{job="orders-api"}[$__rate_interval]))`
- Legend: `{{method}} {{route}}`
- Unit **requests/sec (rps)**. Min 0.
- Legend mode **Table**, placement **Right**, values **Mean**, **Last**.

**Panel "Responses by status class"** (Time series). Three queries:

| Query | Expression | Legend |
|---|---|---|
| A | `sum(rate(http_requests_total{job="orders-api", status=~"2.."}[$__rate_interval]))` | `2xx` |
| B | `sum(rate(http_requests_total{job="orders-api", status=~"4.."}[$__rate_interval]))` | `4xx` |
| C | `sum(rate(http_requests_total{job="orders-api", status=~"5.."}[$__rate_interval]))` | `5xx` |

- Unit **requests/sec (rps)**. Stack **Normal**. Fill opacity 30.
- Overrides by name: `2xx` → green, `4xx` → orange, `5xx` → red.

**Panel "Latency p50 / p95 / p99"** (Time series). Three queries:

| Query | Expression | Legend |
|---|---|---|
| A | `histogram_quantile(0.50, sum by (le) (rate(http_request_duration_seconds_bucket{job="orders-api", route!="/api/slow"}[$__rate_interval])))` | `p50` |
| B | `histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket{job="orders-api", route!="/api/slow"}[$__rate_interval])))` | `p95` |
| C | `histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{job="orders-api", route!="/api/slow"}[$__rate_interval])))` | `p99` |

- Unit **seconds (s)**.
- Thresholds: Base transparent, 1 red. **Show thresholds: As lines (dashed)**.
- Override query C → **Line style: Dash**.

**Panel "Latency heatmap"** (Heatmap):
- Query: `sum by (le) (increase(http_request_duration_seconds_bucket{job="orders-api"}[$__rate_interval]))`
- In the query options, set **Format** to **Heatmap**.
- Heatmap options: **Y axis → Unit: seconds (s)**. Color scheme: e.g. **Oranges**.

✅ **Expected:** most requests in the lowest bands. A separate band near 1–2.5 s comes from `/api/slow`, which is included here on purpose.

**Panel "Status codes"** (Pie chart):
- Query: `sum by (status) (increase(http_requests_total{job="orders-api"}[$__range]))`
- In the query options, set **Type** to **Instant**.
- Legend `{{status}}`.
- Pie chart type **Donut**. Legend values **Percent**. Value mappings are optional.

> **Why Instant and `$__range`?** A pie chart needs **one** number per slice: the total over the whole selected time range. A range query would return many points per series, and the pie would have to reduce them.

#### Step 3.4: Row "Business (SQL)"

1. **Add → Row** titled `Business (SQL)`.
2. For every panel in this row:
   - Select `PostgreSQL-Orders` and switch to **Code**.
   - In **Query options**, set **Relative time** to `7d`. This row always shows the last 7 days, while the RED row follows the dashboard time range.

**Panel "Orders by status"** (Time series, Format **Time series**):

```sql
SELECT
  $__timeGroupAlias(created_at, $__interval),
  status,
  count(*) AS orders
FROM orders
WHERE $__timeFilter(created_at)
GROUP BY 1, 2
ORDER BY 1;
```

- **Graph styles → Style: Bars**. Stack **Normal**.
- Overrides: `PAID` → green, `PAYMENT_FAILED` → red.

> **Note:** The series may be named `orders PAID` rather than `PAID`, depending on the Grafana version. Match whatever name appears in the legend.

**Panel "Revenue by region"** (Bar chart, Format **Table**):

```sql
SELECT region, round(sum(amount), 2) AS revenue
FROM orders
WHERE $__timeFilter(created_at)
GROUP BY region
ORDER BY revenue DESC;
```

- Unit **Indian Rupee (₹)** or **short**. Show values **Always**.

**Panel "Payment failure %"** (Stat, Format **Table**):

```sql
SELECT round(100.0 * count(*) FILTER (WHERE status = 'PAYMENT_FAILED') / NULLIF(count(*), 0), 2) AS failed_pct
FROM orders
WHERE $__timeFilter(created_at);
```

- Unit **Percent (0-100)**. Thresholds: Base green, 12 orange, 15 red.

**Panel "Latest orders"** (Table, Format **Table**). For this panel only, set **Relative time** to `1h` instead of `7d`:

```sql
SELECT created_at AS "time", id, customer, product, region, quantity, amount, status
FROM orders
WHERE $__timeFilter(created_at)
ORDER BY created_at DESC
LIMIT 20;
```

- **Value mappings:**
  - `PAID` → text `Paid`, green
  - `PAYMENT_FAILED` → text `Payment failed`, red
- Override **Fields with name → status** → **Cell options → Cell type: Colored background**.
- Override **amount** → Unit **Indian Rupee (₹)**, Decimals 2.

✅ **Expected:** the 20 newest orders, with paid ones green and failed ones red, updating every 30 seconds.

#### Step 3.5: Row "Logs"

1. **Add → Row** titled `Logs`.
2. Panel **Log volume by level** (Time series, data source `Loki`):

```logql
sum by (level) (count_over_time({job="orders-api"} | json [$__interval]))
```

   - Legend `{{level}}`.
   - Style **Bars**, Stack **Normal**.
   - Overrides: `info` → green, `warn` → orange, `error` → red.

3. Panel **Warnings & errors** (Logs, data source `Loki`):

```logql
{job="orders-api"} | json | level=~"warn|error"
```

   - Logs options: **Time** on, **Wrap lines** on, **Enable log details** on, **Order: Newest first**.
4. Save with the message `RED dashboard complete`.

#### Step 3.6: Prove the dashboard works (fault injection)

A dashboard is only useful if it shows problems clearly. Let's create some.

1. **Inject errors.** Set a 20% error rate:

```powershell
Invoke-RestMethod -Method Post http://localhost:8080/admin/chaos -ContentType 'application/json' -Body '{"errorRate":0.2}'
```

   Wait 2–3 minutes, refreshing the dashboard.

✅ **Expected:**
- **Error % (5xx)** turns **red**, at about 20%.
- A red `5xx` band appears in *Responses by status class*.
- A `500` slice appears in the pie.
- The log volume shows `error` bars.
- *Warnings & errors* shows `request failed` lines.

2. **Inject latency.** Keep the errors, and add 800 ms latency:

```powershell
Invoke-RestMethod -Method Post http://localhost:8080/admin/chaos -ContentType 'application/json' -Body '{"errorRate":0.2,"latencyMs":800}'
```

✅ **Expected:**
- **p95 latency** turns red, rising above 0.8 s.
- The heatmap shows a new band around 1 s.
- **In-flight requests** rises.

3. **Reset:**

```powershell
Invoke-RestMethod -Method Post http://localhost:8080/admin/chaos -ContentType 'application/json' -Body '{"errorRate":0,"latencyMs":0}'
```

✅ **Expected:** within a few minutes, the stats return to green.

4. **Tell the story.** Zoom into the incident by dragging across it on any time series panel. Every panel zooms to the same range, and you can read the whole incident from one screen: errors, latency, affected routes and the matching log lines.

---

### Lab 4: Dashboard versions, JSON model and Drilldown apps

**Goal:** Use the version history to compare and restore, read and edit the JSON model, and explore data with the Drilldown apps.

**Time:** about 40 minutes

#### Step 4.1: Compare and restore versions

1. Open `orders-api – Service Overview` and click **Edit**.
2. Make a deliberate mistake:
   - Change the **Error % (5xx)** thresholds to Base green, 50 red.
   - Delete the **Status codes** pie chart.
3. Save with the message `Experiment – wrong thresholds`.
4. Open **Settings → Versions**.
5. Tick the two newest versions and click **Compare versions**.
6. Read the summary, then click **View JSON diff**.

✅ **Expected:** the diff shows the changed threshold `value` and the removed pie panel.

7. **Restore** the version before the experiment and confirm.

✅ **Expected:**
- The dashboard is back to normal.
- The version list has a **new** version, "Restored from version N".
- The bad version is still in the history.

#### Step 4.2: Read and edit the JSON model

1. Open **Settings → JSON Model**. Find:
   - the dashboard `uid` (also in the browser URL);
   - the `panels` array: find the panel with `"title": "Request rate"`, and note its `gridPos` and `targets[0].expr`;
   - `"refresh": "30s"` and the `time` section.
2. Edit the JSON directly:
   - Change the `title` of the **Request rate** panel to `Request rate (all routes)`.
   - Click **Save changes**, then save the dashboard.

✅ **Expected:** the panel title is updated.

> ⚠️ **Watch out:** Invalid JSON (a missing comma or quote) is rejected. Make small edits, and prefer the UI for anything complex.

3. Export the dashboard JSON and save it as `C:\grafana-lab\dashboards\orders-api-overview.json`. You'll use these files on Day 5 for provisioning and version control.

#### Step 4.3: Import a copy

1. **Dashboards → New → Import** → **Upload dashboard JSON file** → select `orders-api-overview.json`.
2. Grafana warns that a dashboard with the same UID exists. Change the **Name** to `orders-api – Copy`, click **Change uid** to generate a new one, and choose folder `Lab Dashboards`.
3. **Import**.

✅ **Expected:** a second, independent copy.

4. Delete the copy afterwards: **Settings → Delete dashboard**.

#### Step 4.4: Metrics Drilldown

1. Open **Menu → Drilldown → Metrics** and select data source `Prometheus`.
2. In the search box, type `http_`.

✅ **Expected:** small graphs for each `http_*` metric, with no queries written.

3. Select `http_requests_total` and open the **Breakdown** tab. Break it down by `route`, then by `status`.
4. Open **Related metrics** and look at what else is available for this job.

#### Step 4.5: Logs Drilldown

1. Open **Menu → Drilldown → Logs**.

✅ **Expected:** `orders-api` appears as a service with its log volume.

2. Open it and explore the tabs:
   - **Logs:** the raw lines, filterable by level.
   - **Labels:** the distribution of `job`, `detected_level`, and so on.
   - **Fields:** fields detected from the JSON, e.g. `route`, `status`, `duration_ms`. Click `route` to see the volume per route.
   - **Patterns:** recurring message patterns, e.g. `request completed`, `order created`. Useful for spotting a *new* pattern during an incident.
3. Inject errors again (`{"errorRate":0.2}`), wait 2 minutes, and find the errors from Logs Drilldown **without typing any query**. Then reset the fault.

#### Step 4.6: From Explore to dashboard

1. In **Explore** (`Prometheus`), run:

```promql
topk(3, sum by (route) (rate(http_requests_total{job="orders-api"}[5m])))
```

2. Click **Add → Add to dashboard**, choose **Existing dashboard** → `orders-api – Service Overview`, and click **Open dashboard**.
3. Give the new panel the title `Top 3 routes`, place it in the `RED` row, and save.

---

## Exercises

Try these without step-by-step help. Hints are in the [Answers](#answers) section.

**Exercise 1: Error % per route.** Add a time series panel to the RED dashboard showing the 5xx error percentage **per route**. Test it with the `errorRate` fault.

**Exercise 2: Compare with an hour ago.** Add a panel showing the request rate now and the request rate one hour ago, as two lines. Do it two ways:
- (a) with PromQL `offset`;
- (b) with the panel's **Query options → Time shift**.

What is the difference between the two approaches?

**Exercise 3: Average order value.** Using SQL, build a time series of the **average order value** per interval, one line per product, for the last 7 days. Use `$__timeGroupAlias` and `$__timeFilter`.

**Exercise 4: Throughput vs saturation.** On the host dashboard, add a panel that puts **CPU %** (left axis, 0–100) and **processor queue length** (right axis) in the same graph. Hint: overrides.

**Exercise 5: Better status pie.** Change the *Status codes* pie so that it shows status **classes** (2xx, 4xx, 5xx) instead of individual codes, with fixed colours.

**Exercise 6: Slow requests from logs.** Using only LogQL, build a table panel that lists requests slower than 1.5 s in the last 15 minutes, showing time, route and duration.

**Exercise 7: Dashboard review.** Swap dashboards with a colleague, or review your own against the 10 guidelines in Concepts 2.9. Write down three improvements and make them.

---

## Troubleshooting

| # | Symptom | Likely cause | Fix |
|---|---|---|---|
| 1 | Panel shows *No data* | Time range has no data, wrong metric/label name, or the load generator isn't running | Run the query in Explore; check the time range; check `up`; start `load.js` (`start-lab.ps1 -WithLoad`) |
| 2 | Error % panel shows *No data* instead of 0 | No 5xx series exists yet (empty numerator) | Wrap the numerator in `( … or vector(0))` |
| 3 | `histogram_quantile` returns *No data* / `NaN` | No requests in the window, or `le` dropped by the aggregation | Check traffic; make sure it's `sum by (le)` or `sum by (le, …)` |
| 4 | `rate()` graph is empty or jagged | Range window shorter than 2 scrape intervals | Use `$__rate_interval`; set the scrape interval on the data source (Day 1) |
| 5 | Graph shows a constantly rising line | Counter graphed without `rate()` | Wrap in `rate(…[$__rate_interval])` |
| 6 | Values look 100× too big or small | Wrong unit: Percent (0-100) vs Percent (0.0-1.0) | Match the unit to what the query returns |
| 7 | Stat shows an odd number (not the latest) | **Calculation** set to Mean (or similar) | Value options → Calculation → **Last \*** |
| 8 | SQL time series: *Data does not have a time field* | No column named `time`, or Format is Table | Use `$__timeGroupAlias(…)` or `AS "time"`; set Format to **Time series** |
| 9 | SQL time series looks scrambled or zig-zag | Rows not sorted by time | Add `ORDER BY 1` (the time column) |
| 10 | SQL panel is slow | No `$__timeFilter`, so the whole table is scanned | Always filter with `$__timeFilter(col)` on an indexed column |
| 11 | SQL business row empty, but data exists | Dashboard range too short for seed data, and no relative time set | Set panel **Relative time** to `7d` |
| 12 | Bar chart: *Bar charts requires a string or time field* | Query returns only numbers | Return a text column (e.g. `region`) and use Format **Table** |
| 13 | Pie chart shows one slice per point, or odd totals | Range query instead of instant | Query options → Type **Instant**; use `[$__range]` |
| 14 | Heatmap looks like random blocks | Format not set to Heatmap, or `rate` instead of `increase` | Format **Heatmap**; `sum by (le) (increase(…_bucket[$__rate_interval]))` |
| 15 | Loki: `maximum of series (500) reached for a single query` | A metric query grouped by a high-cardinality label (e.g. `orderId`) | Group by low-cardinality labels (`level`, `route`) only |
| 16 | Loki: lines disappear after `\| unwrap` | Lines without that field produce `__error__` | Add `\| __error__=""` after `unwrap`, or pre-filter the lines |
| 17 | Loki: *queries require at least one regexp or equality matcher that does not have an empty-compatible value* | Stream selector like `{job=~".*"}` | Use a specific selector, e.g. `{job="orders-api"}` |
| 18 | Override has no effect | The field name changed (legend changed), or wrong matcher | Check the exact series name in the legend or table view; use **Fields returned by query** |
| 19 | Save fails: *Someone else has updated this dashboard* | Two editors, or two browser tabs, edited the same dashboard | Reload, re-apply your change, save; or **Save as** a copy |
| 20 | Negative-Y series shows negative numbers in the tooltip/legend | Expected behaviour of the transform | Values are mirrored for display only; describe it in the panel description |
| 21 | Logs Drilldown shows no services | Loki `volume_enabled` / pattern ingester not configured | Check `loki-config.yaml` from Day 1 and restart Loki |
| 22 | Host panels show gaps | windows_exporter or Prometheus was stopped, or PC sleep | Check `up{job="windows"}`; restart with `start-lab.ps1` |

---

## Quiz

1. How many columns does the Grafana dashboard grid have?
   - a) 12  b) 16  c) 24  d) 100
2. Which time range expression means "today so far"?
   - a) `now-1d`  b) `now/d` to `now`  c) `today`  d) `now-24h`
3. Why use `$__rate_interval` instead of a fixed `[5m]` in dashboard queries?
4. Which PromQL is correct for requests/sec per route?
   - a) `rate(sum by (route) (http_requests_total)[5m])`
   - b) `sum by (route) (rate(http_requests_total[5m]))`
   - c) `sum(http_requests_total) by route`
   - d) `irate(http_requests_total)`
5. What does `increase(x[1h])` return?
   - a) The per-second rate  b) The total increase over the last hour  c) The maximum value  d) The last value
6. In `histogram_quantile(0.95, sum by (le, route) (rate(..._bucket[5m])))`, what happens if you remove `le` from `by (...)`?
7. An error-ratio query shows *No data* although traffic is flowing. What is the likely cause, and the fix?
8. Which SQL macro limits rows to the dashboard's time range?
   - a) `$__timeGroup`  b) `$__timeFilter`  c) `$__interval`  d) `$__range`
9. What column name does a SQL query need for Grafana's Time series format?
10. Which LogQL part is fastest and should come first after the stream selector?
    - a) `| json`  b) Line filter such as `|= "error"`  c) `line_format`  d) `unwrap`
11. Which visualization best shows a latency **distribution** over time?
    - a) Pie chart  b) Stat  c) Heatmap  d) Table
12. You want a single disk-write series mirrored below the axis. Which feature do you use?
13. A Stat panel shows 0.25 but should show 25%. Which setting is wrong?
14. What happens to the version history when you restore an old dashboard version?
15. Name two things you can do in Logs Drilldown without writing any LogQL.

---

## Summary & Cheat Sheet

### Key points

- Dashboards = panels on a **24-column grid**, grouped in **rows**, sharing a **time range**. Panels can override it with **Relative time** and **Time shift**.
- Pick the visualization by data shape: **time series** for trends, **stat/gauge** for single values, **bar** for categories, **heatmap** for distributions, **state timeline** for states, **logs** for lines.
- **Units, thresholds, value mappings, overrides** turn numbers into meaning.
- Every save is a **version**. Compare and restore are always available. The **JSON model** is the dashboard.
- PromQL: **rate first, then aggregate**. Keep `le` for percentiles. Use `or vector(0)` for empty numerators. Use `$__rate_interval` in dashboards.
- SQL: always `$__timeFilter`. Use `$__timeGroupAlias` for time series. Order by time.
- LogQL: selector → line filter → parser → label filter → format. Metric queries turn logs into numbers.
- Explore for ad-hoc work; **Drilldown** apps for query-less exploration.

### PromQL quick reference

| Goal | Query |
|---|---|
| Request rate | `sum(rate(http_requests_total{job="orders-api"}[$__rate_interval]))` |
| Rate per route | `sum by (route) (rate(http_requests_total{job="orders-api"}[$__rate_interval]))` |
| Error % | `100 * (sum(rate(http_requests_total{status=~"5.."}[$__rate_interval])) or vector(0)) / sum(rate(http_requests_total[$__rate_interval]))` |
| p95 latency | `histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[$__rate_interval])))` |
| Total over range | `sum by (status) (increase(http_requests_total[$__range]))` (Instant) |
| Top N | `topk(3, sum by (route) (rate(http_requests_total[5m])))` |
| Last hour | `… offset 1h` |
| CPU % | `100 - (avg(rate(windows_cpu_time_total{mode="idle"}[$__rate_interval])) * 100)` |
| Memory used % | `100 * (1 - windows_memory_available_bytes / windows_memory_physical_total_bytes)` |
| Disk used % | `100 * (1 - windows_logical_disk_free_bytes / windows_logical_disk_size_bytes)` |
| Uptime | `time() - windows_system_boot_time_timestamp` |

### SQL macro quick reference (PostgreSQL)

```sql
-- Time series (Format: Time series)
SELECT $__timeGroupAlias(created_at, $__interval), <series_col>, <aggregate> AS <name>
FROM <table>
WHERE $__timeFilter(created_at)
GROUP BY 1, 2
ORDER BY 1;

-- Table / bar chart / stat (Format: Table)
SELECT <category>, <aggregate> AS <name>
FROM <table>
WHERE $__timeFilter(created_at)
GROUP BY <category>;
```

### LogQL quick reference

| Goal | Query |
|---|---|
| All lines | `{job="orders-api"}` |
| Contains text | `{job="orders-api"} \|= "order created"` |
| Parse JSON + filter | `{job="orders-api"} \| json \| level="error"` |
| Numeric filter | `{job="orders-api"} \| json \| duration_ms > 1000` |
| Reformat | `… \| line_format "{{.route}} {{.status}}"` |
| Lines per level | `sum by (level) (count_over_time({job="orders-api"} \| json [$__interval]))` |
| p95 from logs | `quantile_over_time(0.95, {job="orders-api"} \| json \| unwrap duration_ms \| __error__="" [5m]) by (route)` |

### Units at a glance

| Data | Unit |
|---|---|
| 0–100 % | Percent (0-100) |
| 0.0–1.0 ratio | Percent (0.0-1.0) |
| Bytes / bytes per second | bytes(IEC) / bytes/sec(IEC) |
| Seconds (latency, uptime) | seconds (s) |
| Requests per second | requests/sec (rps) |
| Counts | short |

---

## Answers

<details>
<summary>Quiz answers</summary>

1. **c**. 24.
2. **b**. `now/d` to `now`.
3. `$__rate_interval` adapts to the time range and the panel width. It is always at least 4× the scrape interval, so `rate()` has enough samples; a fixed window can be too short (gaps) or too long (over-smoothed).
4. **b**. Rate first, then sum.
5. **b**.
6. `histogram_quantile` needs the `le` label to know the bucket boundaries. Without it, there are no buckets to interpolate, and the result is empty or meaningless.
7. The numerator series (e.g. 5xx) doesn't exist yet, so the division returns nothing. Fix: `(numerator or vector(0)) / denominator`.
8. **b**. `$__timeFilter`.
9. `time` (e.g. via `$__timeGroupAlias(...)` or `AS "time"`).
10. **b**. Line filters are the cheapest and reduce the lines that later stages must process.
11. **c**. Heatmap.
12. An override on that series: **Graph styles → Transform → Negative Y**.
13. The unit: it should be **Percent (0.0-1.0)** for a 0–1 ratio, or the query should multiply by 100 and use Percent (0-100).
14. Nothing is lost. The restore creates a **new** version, and all earlier versions, including the "bad" one, remain in the history.
15. Any two of:
    - see services and their log volume;
    - filter by level;
    - view detected fields (e.g. `route`, `status`) and their distribution;
    - view recurring patterns;
    - open individual lines.

</details>

<details>
<summary>Exercise hints</summary>

**Exercise 1:**

```promql
100 * (sum by (route) (rate(http_requests_total{job="orders-api", status=~"5.."}[$__rate_interval])) or sum by (route) (rate(http_requests_total{job="orders-api"}[$__rate_interval])) * 0)
    / sum by (route) (rate(http_requests_total{job="orders-api"}[$__rate_interval]))
```

The `or … * 0` trick gives every route a zero value when it has no 5xx series yet. It works like `or vector(0)`, but keeps the `route` label.

**Exercise 2:**
- (a) Two queries: `sum(rate(http_requests_total[$__rate_interval]))` and `sum(rate(http_requests_total[$__rate_interval] offset 1h))`, both in one panel.
- (b) One query, with **Time shift** `1h` on a second copy of the panel.
- `offset` works per query, so both lines can share one graph. Time shift moves the **whole panel's** time range, so it needs its own panel.

**Exercise 3:**

```sql
SELECT $__timeGroupAlias(created_at, $__interval), product, round(avg(amount), 2) AS avg_order_value
FROM orders
WHERE $__timeFilter(created_at)
GROUP BY 1, 2
ORDER BY 1;
```

**Exercise 4:** Two queries (CPU % and `windows_system_processor_queue_length`). Add an override on the queue series:
- **Axis → Placement: Right**
- **Unit: short**
- **Min: 0**

Keep the default unit Percent (0-100) for CPU.

**Exercise 5:**
- Query: `sum by (class) (label_replace(increase(http_requests_total{job="orders-api"}[$__range]), "class", "${1}xx", "status", "(.).."))` with Type **Instant**.
- Or use three queries (2xx, 4xx, 5xx with `status=~"2.."` etc.), each with a fixed legend.
- Then add colour overrides by name.

**Exercise 6:** Query:

```logql
{job="orders-api"} | json | duration_ms > 1500 | line_format "{{.route}} {{.duration_ms}}"
```

- Use a **Table** visualization with a time range of 15 minutes.
- Loki returns the parsed labels as columns. Use **Transform → Organize fields** (Day 3) to hide the columns you don't need, or simply read the `Line` column.

**Exercise 7:** Typical improvements:
- units on every panel;
- descriptions;
- fewer series per graph;
- consistent colours;
- key stats top-left;
- collapsed detail rows;
- panel titles that say what's shown, e.g. "p95 latency (excl. /api/slow)".

</details>

---

**Previous:** [Day 1 – Observability, Grafana Setup & Data Sources](day01-observability-setup-datasources.md) | **Next:** [Day 3 – Dynamic Dashboards & Alerting](day03-dynamic-dashboards-alerting.md)
