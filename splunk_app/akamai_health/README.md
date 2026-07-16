# akamai_health — Splunk Cloud (Victoria) private app

Accelerated data model + `tstats` backing for the health dashboard (Option B).
Replaces raw-event `spath` scans with reads against an accelerated data model,
removing both the per-panel scan cost and the post-process `maxout` truncation
that made the timechart appear to ignore the time range.

Deploys to **Splunk Cloud Victoria Experience** as a vetted **private app via ACS**.
There is no filesystem/CLI access on Splunk Cloud, so the app is packaged, vetted by
AppInspect, and installed with the ACS API/CLI — not copied into `etc/apps`.

## Contents

| File | Purpose |
|---|---|
| `default/app.conf` | App metadata (`[package] id = akamai_health`). |
| `metadata/default.meta` | App-scoped sharing; all users read, admins write. |
| `default/data/models/akamai_health.json` | Data model: root event dataset over `index=akamai_metrics_prod`; `reqHost` / `country` / `statusCode` extracted from the JSON payload, `status` / `status_class` derived — all **accelerated calculated fields** (the `spath` runs at summary-build time, not on load). |
| `default/datamodels.conf` | Acceleration settings (7-day summary, 5-min cadence). |
| `default/transforms.conf` | `country_iso_to_name` lookup definition. |
| `lookups/country_iso_to_name.csv` | ISO alpha-2 → country name (173 rows); replaces the inline `case()`. |
| `default/data/ui/views/akamai_health_accelerated.xml` | The dashboard (canonical shipping copy). |
| `default/data/ui/nav/default.xml` | App nav → opens the dashboard by default. |

> The repo also keeps a mirror at `dashboards/herbalife-akamai-health-dashboard-accelerated.xml`
> for easy diffing. The **app view** above is what ships — keep them in sync.

## Prerequisites

- **`sc_admin`** role on the stack (required to install private apps and accelerate data models).
- **ACS CLI** installed and configured, or use the ACS REST API directly.
  See Splunk docs: "Manage private apps in Splunk Cloud Platform using the ACS API/CLI".
- Confirm available **data-model acceleration (DMA) storage quota** before backfilling `-7d`.

## Package

From the repo root — tar the app directory so the app folder is the top-level entry.
On macOS you MUST disable AppleDouble/xattr sidecars, or `tar` embeds `._*` files that
AppInspect rejects (`check_that_extracted_splunk_app_does_not_contain_prohibited_directories_or_files`):

```bash
cd splunk_app
COPYFILE_DISABLE=1 tar --no-xattrs --exclude='._*' --exclude='.DS_Store' \
  -czf akamai_health.tar.gz akamai_health

# sanity-check the archive is clean before uploading:
tar -tzf akamai_health.tar.gz | grep -E '(^|/)\._|\.DS_Store|__MACOSX' \
  && echo "DIRTY — do not upload" || echo "clean"
```

## Vet with AppInspect (recommended before install)

```bash
# CLI (pip install splunk-appinspect), Cloud tags:
splunk-appinspect inspect akamai_health.tar.gz \
  --mode precert --included-tags cloud --included-tags private_app
```

Or submit through the AppInspect API. ACS also runs vetting at install time; running
it yourself first avoids a failed install round-trip.

## Install via ACS (Victoria)

```bash
# CLI:
acs apps install private --app-package ./akamai_health.tar.gz --acs-legacy-token <token>

# verify:
acs apps list
acs apps describe akamai_health
```

(ACS submits the package to AppInspect, then installs on success.)

## Post-install — enable acceleration (REQUIRED)

Splunk Cloud vetting prohibits shipping an accelerated data model, so the app
installs **un-accelerated**. Until you turn acceleration on, the dashboard's `tstats`
searches fall back to slow raw scans. An `sc_admin` must enable it once:

1. **Settings → Data models → Akamai Health → Edit → Acceleration**
   - Accelerate: **on**
   - Summary Range: **7 Days** (must be ≥ the dashboard's widest time picker)
2. Watch the backfill complete before trusting the dashboard:
   ```
   | rest /services/data/models/akamai_health | table title acceleration.*
   ```
2. **Smoke test** the summary read — should return quickly over the full window:
   ```
   | tstats count FROM datamodel=akamai_health WHERE nodename=web_requests BY web_requests.status_class
   ```
   If `count` is 0, either the summary hasn't built yet or the extractions returned
   null — verify `_raw` is JSON with top-level `statusCode` / `country` / `reqHost` keys.
3. Open the **Akamai Health** app → the accelerated dashboard loads by default.

## Splunk Cloud constraints

- **Time range vs summary window.** `acceleration.earliest_time = -7d`. The dashboard's
  widest picker preset must stay within 7 days or older buckets fall outside the
  summary and panels under-report. Raise `earliest_time` + `backfill_time` together
  (costs more DMA storage).
- **Summary lag.** `max_time = 3600` + 5-min cron means the most recent ~5 min may not
  be summarized yet. Fine for a health view.
- **DMA storage quota** is a finite Cloud resource — monitor after the first build.

## Update / rollback

- **Update:** bump `[launcher] version` in `app.conf`, repackage, and
  `acs apps install private ...` again (ACS handles the upgrade).
- **Rollback:** `acs apps uninstall akamai_health`. Users can fall back to the
  Option A dashboard (`herbalife-akamai-health-dashboard-simplified.xml`), which is
  independent of this app. Uninstalling also stops consuming DMA storage.
