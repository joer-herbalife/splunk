# Splunk Dashboards
A collection of Splunk Dashboards created for Splunk Cloud Observability

## Herbalife Akamai Website Health

Endpoint health, latency, regional impact, and cache/origin symptoms derived from Akamai CDN metrics.

- **Splunk:** https://hrbl.splunkcloud.com/en-US/app/search/herbalife_health_akamai
- **Source:** [`dashboards/herbalife-akamai-health-dashboard.xml`](dashboards/herbalife-akamai-health-dashboard.xml)
- **Index:** `akamai_metrics_prod`

### How it works

A single base search parses each Akamai log event and normalizes the fields the panels rely on
(`reqHost`, `reqPath`, `country`, `statusCode`, `timeToFirstByte`, `downloadTime`, `turnAroundTimeMSec`,
`transferTimeMSec`, `tlsOverheadTimeMSec`, `edgeAttempts`, `throughput`). Fields are extracted with `spath`
and fall back to regex extraction, since some events deliver values as strings or multivalue fields.

From those it derives the classifications every panel shares:

- `status_class` — `2xx`, `3xx`, `4xx`, `5xx`, `no_response` (status 0), or `unknown`
- `is_error` — a 5xx **or** no response; this is what drives availability and error rate
- `is_slow_ttfb` / `is_slow_download` — time to first byte or download time above the thresholds you set

All panels run off this base search, so the inputs filter the whole dashboard consistently.

### Inputs

| Input | Default | Notes |
|---|---|---|
| Time Range | Last 10 hours | |
| Health Filter | All | All, Error, Slow TimeToFirstByte, Slow Download, or Errors and Slow |
| Status | All | Restrict to a status class (`2xx`–`5xx`, No Response, Unknown) |
| HTTP Method | `GET` | Defaults to `GET`, not All — widen it when investigating APIs or form posts |
| Country | All Countries | Populated from the data in range, labeled with request counts |
| Path Filter | `*` | Wildcards allowed, e.g. `/*en-us/*` |
| Host Filter | `*` | Wildcards allowed, e.g. `vn.herbalife.com` |
| TimeToFirstByte Threshold | 3000 ms | Defines a "slow TTFB" request |
| Download Threshold | 5000 ms | Defines a "slow download" request |

Note that the Health Filter narrows the events the panels aggregate over. With it set to anything but
"All", rates such as availability and error rate are computed within that subset, so read them as a
breakdown of the filtered traffic rather than as site-wide numbers.

### Panels

**Summary tiles** — Availability, Error Rate, P95 TimeToFirstByte, and total Requests. Availability and
error rate are color-coded (availability green at ≥99.9%, red below 99%; error rate green under 0.1%, red
above 1%).

**Availability, Errors, and Latency Over Time** — 5-minute timechart of requests, errors, error and
availability percentages, and P95/P99 TTFB. Use it to place an incident in time before drilling down.

**Top Hosts by Volume** — Busiest hosts with request count, errors, and P95 TTFB.

**Top Hosts by Error Rate** — Hosts ranked by combined error rate, splitting server errors and no-response
(`is_error`) from client `4xx` errors so you can tell an origin problem from a bad-request pattern. Hosts
with fewer than 20 requests are excluded.

**Top Hosts by P95 (TimeToFirstByte)** — Hosts ranked by P95 TTFB, with P50/P95/P99 and P95 download time
to show the shape of the latency distribution rather than just its tail.

**Slowest Pages by P95 (TimeToFirstByte)** — Same latency breakdown at the host + path level, including how
many of the page's requests crossed the slow-TTFB threshold and which status codes it returned.

**Top Unhealthy Pages** — Pages ranked by the share of requests that were bad in any way: server error,
client error, slow TTFB, or slow download. Includes up to five referers per page to help identify who is
driving the traffic.

**Status Mix by Host** — Status class distribution per host, in both counts and percentages, sorted by 5xx
then 4xx share.

**Status Code Detail** — Bar chart of the top 25 individual status codes, alongside a per-host table of
each status code as a share of that host's traffic.

**Regional Health** — Requests, errors, availability, and latency broken out by country, state, serving
country, billing region, and host — for when a problem is geographic rather than global.

**Cache and Origin Symptoms** — Per host and cache status: error rate, P95 TTFB, P95 turnaround (origin
response time), P95 download, average edge attempts, and average throughput. A rising turnaround time or
edge attempt count on cache misses points at the origin rather than the CDN.

**Pages With Regional Skew** — Pages whose error rate or P95 latency in one country deviates sharply from
that page's own average (more than 2 percentage points of error, or more than 1000 ms of latency),
surfacing regional problems that a global average would hide.

Most tables require at least 20 requests per row (10 for regional skew) so that low-volume noise doesn't
dominate the rankings, and are capped at the top 50 rows.

## Herbalife Akamai Website Health (Accelerated)

The current error-focused view: availability, error rate, regional impact, and a 4xx/5xx breakdown for
hosts and regions — backed by an accelerated data model so it renders fast over the full selected time
range. **This supersedes the deprecated Top 20 dashboard below.**

- **Splunk:** https://hrbl.splunkcloud.com/en-US/app/search/herbalife_akamai_website_health_simplified
- **Source:** [`dashboards/herbalife-akamai-health-dashboard-accelerated.xml`](dashboards/herbalife-akamai-health-dashboard-accelerated.xml)
- **Data model:** `akamai_health` (must be accelerated — setup in [`splunk_app/akamai_health/MANUAL_SETUP.md`](splunk_app/akamai_health/MANUAL_SETUP.md))
- **Index:** `akamai_metrics_prod`

### How it works

Instead of parsing raw JSON on every load, the panels read a pre-built summary via `tstats` against the
`akamai_health` data model. The model extracts `reqHost`, `country`, and `statusCode` from each event once
(at summary-build time) and derives `status_class` (`2xx`/`3xx`/`4xx`/`5xx`/`no_response`/`unknown`). This
removes the per-panel raw-event scan and the post-process result cap, so every panel aggregates the entire
selected range quickly.

Because it reads an accelerated summary:

- **It is not real-time.** Use relative/preset time ranges — a Real-time selection errors, since `tstats`
  is unsupported in real-time search. Data is typically current to within a minute or two (5-minute
  summary rebuild, plus a raw scan of the not-yet-summarized tail).
- **Acceleration must be enabled** on the data model (7-day summary range by default). Until the summary is
  built, `tstats` falls back to slow raw scans. The map uses the `country_iso_to_name` lookup to turn ISO
  codes into names for the `geom` overlay.

### Inputs

| Input | Default | Notes |
|---|---|---|
| Time Range | Last 10 hours | The only input. Do **not** pick a Real-time range — `tstats` does not support it |

### Panels

The same error-focused panels as the deprecated Top 20 dashboard — Availability / Server Error Rate (5xx) /
Client Error Rate (4xx) tiles, Total Requests by 4xx & 5xx over time, Error Rate Heatmap by Region, and
Errors by Region / by Host tables with per-status-code drilldowns — but every panel is driven by the
accelerated `akamai_health` data model rather than its own raw search, and each honors the selected time
range.

## Herbalife Akamai Website Health (Top 20) — DEPRECATED

> **Deprecated.** Superseded by the [Accelerated dashboard](#herbalife-akamai-website-health-accelerated)
> above, which has the same panels but reads an accelerated data model (faster, full-range accurate). Kept
> here for reference only; not recommended for new use.

An error-focused, stripped-down companion to the full dashboard: availability, error rate, regional
impact, and a 4xx/5xx breakdown for the top 20 hosts and regions. Use it for a fast read on "is anything
broken and where," then jump to the full dashboard (or the built-in drilldowns) to investigate.

- **Source:** [`dashboards/herbalife-akamai-health-dashboard-simplified.xml`](dashboards/herbalife-akamai-health-dashboard-simplified.xml)
- **Index:** `akamai_metrics_prod`

### How it works

Like the full dashboard, each panel derives the status class from `statusCode`, extracted with `spath` and
a regex fallback (some events deliver the code as a string or multivalue field). Requests are then bucketed
into `2xx`/`3xx`/`4xx`/`5xx`/no-response and counted. This view carries only the error and regional signals,
so the panels run their own focused searches rather than sharing filters through a base search.

### Inputs

| Input | Default | Notes |
|---|---|---|
| Time Range | Last 10 hours | The only input — there are no health, status, method, country, path, or host filters here |

### Panels

**Summary tiles** — Availability, Server Error Rate (5xx / no-response), and Client Error Rate (4xx), all
color-coded. Availability is green at ≥99.9% and red below 99%; server error rate is green under 0.1% and
red above 1%; client error rate is green under 1% and red above 5%. Splitting server from client errors
keeps a wave of 404s from masking (or being mistaken for) an origin outage.

**Total Requests by 4xx & 5xx Over Time** — 5-minute line chart of 4xx and 5xx counts, to place a spike in
time.

**Error Rate Heatmap by Region** — Choropleth world map colored by each country's error rate
(`(4xx + 5xx + no_response) / requests`). Countries need at least 100 requests and one error to appear, so
low-volume regions don't light up the map. ISO country codes are mapped to names for the `geom` overlay.

**Errors [4xx, 5xx] by Region** — Top 20 countries by request volume that saw at least one error, with
request count, overall error %, and a column per individual status code. Clicking a status-code cell opens
a raw search filtered to that country and status code (in a new tab); clicking elsewhere on the row opens
the country's traffic unfiltered.

**Errors [4xx, 5xx] by Host** — The same breakdown and drilldowns at the host (`reqHost`) level, for the
top 20 hosts by volume with errors.
