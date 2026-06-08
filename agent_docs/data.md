# Data Reference

## Directory Layout

```
cfr-data/
├── stations/
│   ├── latest/
│   │   └── stations.csv                         — current authoritative station list
│   └── historical/
│       └── {timestamp}.csv                      — snapshot saved each time the list changes
├── raw/
│   └── {YYYY}/
│       └── {MM}/
│           └── {DD}/
│               └── station/
│                   └── {timestamp}.csv          — arrivals/departures snapshot for that date
└── aggregated/
    └── {YYYY}/
        └── {MM}/
            └── station/
                ├── arrivals/
                │   └── {letter}/
                │       └── {stationName}/
                │           └── delay.csv        — cumulative arrival delay totals
                └── departures/
                    └── {letter}/
                        └── {stationName}/
                            └── delay.csv        — cumulative departure delay totals
```

## Naming Conventions

### Timestamp format

All file timestamps use UTC time with the pattern `YYYY-MM-DDTHH-MM-SSZ` (colons replaced by dashes to be filesystem-safe).

Example: `2026-06-07T21-19-41Z.csv`

### Date path segments

Year, month, and day path components are zero-padded.

Example: `raw/2026/06/07/station/`

### Station name directories (aggregated)

Station names are used as directory names exactly as they appear in the raw CSV. The first-letter bucket (`{letter}`) is the first character of the station name.

## CSV Schemas

### `stations/latest/stations.csv` and `stations/historical/{timestamp}.csv`

Each row represents one railway station.

| Column | Type | Description |
|---|---|---|
| `name` | String | Station name |

### `raw/{YYYY}/{MM}/{DD}/station/{timestamp}.csv`

Each row represents one train's arrival/departure record at one station on the target date.

| Column | Type | Description |
|---|---|---|
| `currentTimestamp` | String | Timestamp when the station was queried (`dd.MM.yyyy HH:mm`) |
| `cfr_date` | String | The date for which data was requested |
| `station` | String | Station name |
| `trainId` | String | Train identifier |
| `trainOperator` | String | Rail operator name |
| `fromStation` | String | Train's origin station |
| `arrival` | String | Scheduled arrival time (`dd.MM.yyyy HH:mm`), null if terminus |
| `arrivalDelayMinutes` | Long | Arrival delay in minutes, null if not available |
| `toStation` | String | Train's destination station |
| `departure` | String | Scheduled departure time (`dd.MM.yyyy HH:mm`), null if terminus |
| `departureDelayMinutes` | Long | Departure delay in minutes, null if not available |
| `platform` | String | Platform number, null if not available |

### `aggregated/{YYYY}/{MM}/station/arrivals/{letter}/{stationName}/delay.csv`

Each row represents one station's cumulative arrival delay total for one aggregation run.

| Column | Type | Description |
|---|---|---|
| `currentTimestamp` | String | Timestamp of the source row for this station |
| `cfr_date` | String | The date for which data was aggregated |
| `station` | String | Station name |
| `totalDelayMinutes` | Long | Sum of arrival delay minutes across all trains at this station |

### `aggregated/{YYYY}/{MM}/station/departures/{letter}/{stationName}/delay.csv`

Same four-column schema as the arrivals delay CSV; `totalDelayMinutes` reflects departure delays.

## Multi-File Behavior

A single calendar date under `raw/` may contain more than one CSV file if the create-raw-station-dataset workflow ran multiple times targeting that date (for example, once on schedule and once via manual dispatch). The aggregate-station-delays workflow selects the newest file by modification time when `raw_file` is set to `latest`; a specific filename can be provided to target an earlier file.

The delay CSV files under `aggregated/` are append-only. Each aggregation run adds one row per station. Multiple rows with the same `cfr_date` are normal and expected — each row reflects a separate workflow run, potentially from a different raw source file.
