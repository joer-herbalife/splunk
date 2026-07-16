# Manual setup via Splunk Cloud Web UI (no app packaging)

Build the four objects by hand in Splunk Web instead of installing the packaged app.
Do them in this order — the dashboard depends on the lookup and the data model.

Pick ONE app to hold everything (e.g. **Search & Reporting**) and set the data model
and lookup sharing to **Global** so the dashboard resolves them by name. The
`splunk_app/akamai_health/...` files in this repo are the source of truth for every
value below.

---

## 1. Lookup table file (CSV)

**Settings → Lookups → Lookup table files → New**
- Destination app: **search** (or your chosen app)
- Upload file: `splunk_app/akamai_health/lookups/country_iso_to_name.csv`
- Destination filename: `country_iso_to_name.csv`
- Save → then **Permissions** → Global (All apps), read: Everyone.

## 2. Lookup definition

**Settings → Lookups → Lookup definitions → New**
- Destination app: same app
- Name: `country_iso_to_name`   ← must match exactly (the dashboard calls it by this name)
- Type: **File-based**
- Lookup file: `country_iso_to_name.csv`
- (Advanced) Minimum matches 0; case-sensitive match: **on**
- Save → **Permissions** → Global, read: Everyone.

Quick test in Search:
```
| makeresults | eval country="US" | lookup country_iso_to_name country_code AS country OUTPUT country_name
```
Expect `country_name = United States of America`.

## 3. Data model + acceleration

**Settings → Data models → New Data Model**
- Title: `Akamai Health`
- ID: `akamai_health`   ← must match `datamodel=akamai_health` in the dashboard
- App: same app → Create

**Add Dataset → Root Event**
- Dataset Name: `web_requests`   ← must match `nodename=web_requests` / `web_requests.*` in the SPL
- Constraints: `index=akamai_metrics_prod`
- Save

**Add these 5 fields as `Add Field → Eval Expression`, IN THIS ORDER**
(order matters — `status` uses `statusCode`; `status_class` uses `status`):

| # | Field name | Type | Eval expression |
|---|---|---|---|
| 1 | `reqHost` | String | `coalesce(reqHost, spath(_raw, "reqHost"), "-")` |
| 2 | `country` | String | `coalesce(country, spath(_raw, "country"), "-")` |
| 3 | `statusCode` | String | `coalesce(mvindex(statusCode, 0), spath(_raw, "statusCode"))` |
| 4 | `status` | Number | `tonumber(statusCode)` |
| 5 | `status_class` | String | `case(status=0, "no_response", status>=200 AND status<300, "2xx", status>=300 AND status<400, "3xx", status>=400 AND status<500, "4xx", status>=500 AND status<600, "5xx", true(), "unknown")` |

- Set each field **Flags → Required** is not needed; leave optional.
- Save the data model → **Permissions** → Global, read: Everyone.

**Enable acceleration (REQUIRED for speed):**
- On the data model → **Edit → Edit Acceleration**
  - Accelerate: **on**
  - Summary Range: **7 Days**  (must be ≥ the dashboard's widest time picker)
- Save. Wait for the backfill to finish:
```
| rest /services/data/models/akamai_health | table title acceleration.*
```
Smoke test (should return quickly and cover the full window once built):
```
| tstats count FROM datamodel=akamai_health WHERE nodename=web_requests BY web_requests.status_class
```
If `count` is 0: summary not built yet, or `_raw` isn't JSON with top-level
`statusCode` / `country` / `reqHost` keys.

## 4. Dashboard

**Dashboards → Create New Dashboard → Classic Dashboards** (Simple XML — NOT Dashboard Studio)
- Title: `Herbalife Akamai Website Health (Accelerated)`
- Create → open it → **Edit → Source**
- Delete the placeholder XML and paste the full contents of
  `dashboards/herbalife-akamai-health-dashboard-accelerated.xml`
- Save. Set **Permissions** as desired.

Run the dashboard in the SAME app where you created the data model (or keep the model
Global) so `datamodel=akamai_health` and the `country_iso_to_name` lookup resolve.

---

## Notes
- Until acceleration is enabled and built, the dashboard's `tstats` searches fall back
  to slow raw scans — enable it before judging performance.
- The packaging files (`app.conf`, `app.manifest`, `datamodels.conf`, `transforms.conf`,
  `metadata/default.meta`) are only needed for the packaged-app/ACS route. For manual
  setup they're unused — keep them as the reference spec or ignore them.
