# Project Scope

## Purpose

CFR Data is the persistent data store for the Romanian railway on-time performance pipeline. It stores three categories of data: station metadata, raw arrival/departure snapshots collected from the live CFR API, and aggregated cumulative delay totals per station. It serves as the append-only historical layer for trend analysis and reporting that is impossible from the live API alone.

## Problem Solved

The cfr-api-adapter service delivers live per-station train data in real time but has no persistence. Any workflow that needs historical data — delay trends, station comparisons, date-range queries — would have no source to query. CFR Data solves this by capturing timestamped snapshots on a recurring schedule and accumulating delay statistics over time, building a queryable record of Romanian railway performance.

## System Context

```
CFR website → cfr-api-adapter (port 8080) → cfr-data-aggregator CLI → cfr-data repo
```

| Component | Role |
|---|---|
| CFR website | Romanian national railway operator; source of live train data |
| `cfr-api-adapter` | Spring Boot service that exposes structured REST endpoints over CFR's live website |
| `cfr-data-aggregator` | CLI tool that fans out requests to cfr-api-adapter and writes the results to CSV files |
| `cfr-data` (this repo) | Receives CSV commits from cfr-data-aggregator; the persistent historical store |

The two sibling repositories are checked out and invoked directly by the GitHub Actions workflows in this repository.

| Repo | Used by workflows |
|---|---|
| `Stavre/cfr-api-adapter` | create-stations-list, create-raw-station-dataset |
| `Stavre/cfr-data-aggregator` | all three workflows |

## What This Repo Contains

- **Station list management** — the current list of known Romanian railway stations and a historical record of every version of that list
- **Raw daily snapshots** — arrivals and departures for all stations on a given date, captured as a single CSV per collection run
- **Aggregated delay totals** — cumulative arrival and departure delay minutes per station per month, appended after each collection run

## Scope Boundaries

**In scope:**
- Storing and versioning the data files described above
- Tracking changes to the station list over time via historical snapshots

**Out of scope:**
- Live data access — use cfr-api-adapter
- Data transformation or export — use cfr-data-aggregator
- Manual data editing of any file other than `stations/latest/stations.csv`
- Authentication, booking, or write operations toward CFR
