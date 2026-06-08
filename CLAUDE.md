# CFR Data

CFR Data is the persistent data store for Romanian railway on-time performance. It holds station metadata, raw arrival/departure snapshots, and aggregated delay statistics. Data is produced entirely by automated GitHub Actions workflows — the repository itself contains no code or build artifacts.

## Repository Structure

| Path | Contents |
|---|---|
| `stations/latest/` | Current authoritative station list (`stations.csv`) |
| `stations/historical/` | Timestamped snapshots saved whenever the station list changes |
| `raw/YYYY/MM/DD/station/` | Raw arrivals/departures CSVs, one file per collection run |
| `aggregated/YYYY/MM/station/` | Cumulative delay totals per station, organized by direction and first letter |
| `.github/workflows/` | Three GitHub Actions workflows that populate all of the above |

## Key Facts

- Data is produced by two sibling repositories: `cfr-api-adapter` (Spring Boot proxy to CFR's live website) and `cfr-data-aggregator` (CLI tool that fans out API requests and writes CSVs).
- All commits to this repository are automated and carry the message `Automated report`.
- The only manual interaction points are `workflow_dispatch` triggers on the three workflows.
- The default reference timezone used throughout is `Europe/Bucharest`.

## Agent Docs

- [agent_docs/scope.md](agent_docs/scope.md) — purpose, system context, and scope boundaries
- [agent_docs/data.md](agent_docs/data.md) — directory layout, naming conventions, and CSV schemas
- [agent_docs/workflows.md](agent_docs/workflows.md) — workflow schedules, job graphs, inputs, and manual trigger guidance
