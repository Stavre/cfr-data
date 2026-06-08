# Workflows

## Overview

| Workflow file | Schedule | Purpose |
|---|---|---|
| `create-stations-list-workflow.yml` | Daily 02:00 UTC | Detect station list changes and reconcile |
| `create-raw-station-dataset-workflow.yml` | Every 3 hours | Collect arrivals/departures snapshot for a date |
| `aggregate-station-delays-workflow.yml` | Every 4 hours | Compute cumulative delay totals from a raw snapshot |

## Pipeline Relationship

```
[create-stations-list]         ──► stations/latest/stations.csv
                                   stations/historical/{ts}.csv

[create-raw-station-dataset]   ──► raw/{Y}/{M}/{D}/station/{ts}.csv
                                              │
                                              ▼
[aggregate-station-delays]     ──► aggregated/{Y}/{M}/station/{arrivals|departures}/{L}/{name}/delay.csv
```

The stations workflow is independent of the other two. The aggregate workflow reads output produced by the raw-dataset workflow.

## Shared Concepts

### Date resolution

Both `create-raw-station-dataset` and `aggregate-station-delays` accept a `cfr_date` input with three forms:

| Value | Resolves to |
|---|---|
| `yesterday` (default) | Yesterday's date in the specified timezone |
| `today` | Today's date in the specified timezone |
| `DD.MM.YYYY` | The exact date provided (e.g. `09.05.2020`) |

The `timezone` input (default `Europe/Bucharest`) is used only when resolving `today` or `yesterday`.

### Commit step

All three workflows commit back to cfr-data using the same pattern:

```bash
git config --global user.name '{actor}'
git add --all
git commit -am "Automated report"
git pull --rebase origin {branch}
git push
```

### Gradle invocation pattern

CLI commands are invoked via Gradle, not a standalone binary:

```bash
./gradlew bootRun --args="<command> <options>"
```

This starts the Spring Boot application as a one-shot CLI process. It does not start a persistent server.

---

## Workflow 1: create-stations-list-workflow.yml

### Schedule

Daily at 02:00 UTC. Also triggerable via `workflow_dispatch` (no inputs).

### Purpose

Fetches the current list of stations from cfr-api-adapter, compares it to `stations/latest/stations.csv`, and if changes are detected saves a timestamped historical snapshot and reconciles the latest file.

### Permissions

```yaml
permissions:
  contents: write
  actions: write   # needed to delete the stations-list artifact when unchanged
```

### Job Graph

```
Create-stations-list → Check-stations-list → Reconcile-stations-list (conditional)
```

### Job: Create-stations-list

1. Checks out `cfr-api-adapter` and `cfr-data-aggregator`
2. Starts cfr-api-adapter: `./gradlew bootRun &` on port 8080; polls health endpoint for up to 120 seconds
3. Runs: `export-stations --output='../stations.csv'`
4. Uploads `stations.csv` as a GitHub Actions artifact named `stations-list`

### Job: Check-stations-list

1. Checks out `cfr-data`
2. Downloads the `stations-list` artifact
3. Diffs the new file against `stations/latest/stations.csv`:
   - Latest file does not exist → `changes_detected=true`
   - Files are identical → `changes_detected=false`; deletes the artifact via the GitHub API
   - Files differ → `changes_detected=true`
4. Outputs: `changes_detected`

### Job: Reconcile-stations-list (runs only if `changes_detected == 'true'`)

1. Generates a UTC timestamp in `YYYY-MM-DDTHH-MM-SSZ` format
2. Checks out `cfr-data` and `cfr-data-aggregator`; downloads the `stations-list` artifact
3. Copies the new CSV to `stations/historical/{timestamp}.csv`
4. Runs: `reconcile-stations -b '../cfr-data/stations/latest/stations.csv' -n '../cfr-data/stations/historical/{timestamp}.csv'`
   — cfr-api-adapter is **not** required for this step
5. Commits to cfr-data

### CLI Commands Used

| Command | Requires cfr-api-adapter | Purpose |
|---|---|---|
| `export-stations --output` | yes | Fetch current station list |
| `reconcile-stations -b <base> -n <new>` | no | Merge new stations into latest file |

### Manual Trigger Notes

No inputs are available on `workflow_dispatch`. The workflow always runs the full `Create-stations-list` job and then conditionally reconciles based on whether changes are detected.

---

## Workflow 2: create-raw-station-dataset-workflow.yml

### Schedule

Every 3 hours: 00:00, 03:00, 06:00, 09:00, 12:00, 15:00, 18:00, 21:00 UTC. Also triggerable via `workflow_dispatch`.

### Purpose

Collects a full arrivals/departures snapshot for the target date from cfr-api-adapter and saves it to `raw/`. Skips the collection if a CSV already exists for that date.

### Permissions

```yaml
permissions:
  contents: write
```

### Inputs (workflow_dispatch)

| Input | Default | Description |
|---|---|---|
| `stations` | `all` | Station filter: `all` or comma-delimited list of station names |
| `cfr_date` | `yesterday` | Date to collect: `today`, `yesterday`, or `DD.MM.YYYY` |
| `timezone` | `Europe/Bucharest` | Timezone for resolving `today`/`yesterday` |

On a scheduled run all inputs use their defaults.

### Job Graph

```
Set-workflow-variables → Check-raw-file-exists → Create-raw-station-dataset (conditional)
```

### Job: Set-workflow-variables

Parses `cfr_date` using the date resolution logic described in Shared Concepts.

Outputs: `year` (YYYY), `month` (MM), `day` (DD), `workflow_run_timestamp` (UTC, `YYYY-MM-DDTHH-MM-SSZ`)

### Job: Check-raw-file-exists

Checks whether `raw/{year}/{month}/{day}/station/*.csv` already contains any file.

Outputs: `file_exists` (true/false)

### Job: Create-raw-station-dataset (runs only if `file_exists != 'true'`)

1. Checks out `cfr-data`, `cfr-api-adapter`, and `cfr-data-aggregator`
2. Starts cfr-api-adapter on port 8080; waits up to 120 seconds for health
3. Runs:
   ```
   export-arrivals-departures
     --date='{DD}.{MM}.{YYYY}'
     --stations='{stations}'
     --output='../cfr-data/raw/{YYYY}/{MM}/{DD}/station/{workflow_run_timestamp}.csv'
   ```
4. Commits to cfr-data

### CLI Commands Used

| Command | Requires cfr-api-adapter | Purpose |
|---|---|---|
| `export-arrivals-departures --date --stations --output` | yes | Collect arrivals/departures snapshot |

### Manual Trigger Notes

To backfill a specific past date, set `cfr_date` to `DD.MM.YYYY`. The `Check-raw-file-exists` job will skip collection if any CSV already exists for that date — there is no force-override input. To collect a second snapshot for a date that already has a file, the only option is to delete the existing file from the repository first.

---

## Workflow 3: aggregate-station-delays-workflow.yml

### Schedule

Every 4 hours: 00:00, 04:00, 08:00, 12:00, 16:00, 20:00 UTC. Also triggerable via `workflow_dispatch`.

### Purpose

Reads a raw arrivals/departures CSV and computes cumulative arrival and departure delay totals per station, appending one row per station to the appropriate `delay.csv` file under `aggregated/`.

### Permissions

```yaml
permissions:
  contents: write
```

### Inputs (workflow_dispatch)

| Input | Default | Description |
|---|---|---|
| `stations` | `all` | Station filter: `all` or comma-delimited list of station names |
| `cfr_date` | `yesterday` | Date to aggregate: `today`, `yesterday`, or `DD.MM.YYYY` |
| `timezone` | `Europe/Bucharest` | Timezone for resolving `today`/`yesterday` |
| `raw_file` | `latest` | `latest` to auto-pick the newest file for the resolved date, or a specific filename (e.g. `2026-06-07T04-00-30Z.csv`) |

On a scheduled run all inputs use their defaults.

### Job Graph

```
Set-workflow-variables → Aggregate-station-delays
```

There is no skip check — this workflow always runs the aggregation step regardless of whether it has run before for the same date.

### Job: Set-workflow-variables

Parses `cfr_date` using the date resolution logic described in Shared Concepts.

Outputs: `year` (YYYY), `month` (MM), `day` (DD)

### Job: Aggregate-station-delays

1. Checks out `cfr-data` and `cfr-data-aggregator`
2. Resolves the raw input file:
   - If `raw_file == 'latest'`: selects the newest file in `raw/{Y}/{M}/{D}/station/` by modification time
   - Otherwise: uses `raw/{Y}/{M}/{D}/station/{raw_file}` directly
3. Runs:
   ```
   aggregate-delays
     --stations='{stations}'
     --input-file='{resolved raw file}'
     -p '../cfr-data/aggregated/{YYYY}/{MM}/station'
   ```
   — cfr-api-adapter is **not** required for this step
4. Commits to cfr-data

### CLI Commands Used

| Command | Requires cfr-api-adapter | Purpose |
|---|---|---|
| `aggregate-delays --stations --input-file -p` | no | Compute and append delay totals |

### Output Written

For each station processed:

```
aggregated/{YYYY}/{MM}/station/arrivals/{firstLetter}/{stationName}/delay.csv
aggregated/{YYYY}/{MM}/station/departures/{firstLetter}/{stationName}/delay.csv
```

One row is appended per run. Running the workflow multiple times against the same raw file produces identical duplicate rows — this is expected behavior since the files are append-only.

### Manual Trigger Notes

To aggregate a specific raw file, set `cfr_date` to the matching date and `raw_file` to the exact filename (e.g. `2026-06-07T04-00-30Z.csv`). To use the newest available file for a past date, set `cfr_date` and leave `raw_file` as `latest`.

---

## Secrets

| Secret | Source | Used by |
|---|---|---|
| `GITHUB_TOKEN` | Automatically provided by GitHub Actions | All three workflows (checkout with write access, git push, artifact API) |

No additional secrets need to be configured manually.
