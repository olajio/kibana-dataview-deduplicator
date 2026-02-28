# Find Duplicate Data Views — Multi-Cluster Scanner

Scan 100+ Elasticsearch/Kibana deployments for duplicate data views in under 2 minutes. Identifies duplicates, counts saved-object references, labels KEEP/DELETE recommendations, detects space defaults, and exports actionable reports.

## Features

| Feature | Description |
|---------|-------------|
| **Multi-cluster scanning** | Reads `clusters.json`, scans all deployments and spaces automatically |
| **Batched API calls** | 34 object types in 1 request per space (was 34 separate calls) |
| **KEEP / SAFE TO DELETE labels** | Highest-ref-count ID tagged KEEP; zero-ref orphans tagged SAFE TO DELETE |
| **Default data view detection** | Calls `/api/data_views/default` per space — never recommends deleting the default |
| **Dry-run delete preview** | `--dry-run-delete` shows exact DELETE API URLs without making changes |
| **Top offenders ranking** | `--top-offenders` ranks spaces by duplicate count for prioritization |
| **Progress bar** | Real-time terminal progress with ETA, no external dependencies |
| **Retry with backoff** | Automatic retries on timeouts/5xx with exponential backoff |
| **Keyboard interrupt** | Ctrl+C prints partial results instead of a raw traceback |
| **Log file support** | `--log-file` writes to both stdout and file for cron/audit use |
| **Concurrent workers** | `--workers N` for parallel cluster scanning |
| **CSV/JSON export** | `--output csv` or `--output json` with all fields including labels |

## Quick Start

```bash
# Basic scan — all clusters
python find_duplicate_dataviews.py --config clusters.json

# Scan + preview what would be deleted + show worst spaces
python find_duplicate_dataviews.py --config clusters.json --dry-run-delete --top-offenders

# Scan specific clusters, export CSV, write log
python find_duplicate_dataviews.py --config clusters.json \
    --clusters "FISMA Scorecard" "(LayerD) ES Federal" \
    --output csv --log-file

# Test connectivity only
python find_duplicate_dataviews.py --config clusters.json --connectivity-check
```

## Configuration

Create a `clusters.json` file:

```json
{
  "clusters": {
    "(LayerD) ES Federal": {
      "kibana_url": "https://es6fedlayerd.kb.ps.cdm-db.com:9243",
      "api_key": "$LAYERD_API_KEY",
      "verify_ssl": true,
      "description": "LayerD Federal cluster"
    },
    "FISMA Scorecard": {
      "kibana_url": "https://fisma.kb.ps.cdm-db.com:9243",
      "api_key": "$FISMA_API_KEY",
      "verify_ssl": true
    }
  }
}
```

API keys starting with `$` are resolved from environment variables.

## Workflow

```
┌─────────────────────────────────────────────────────────────────────┐
│                    find_duplicate_dataviews.py                       │
└─────────────────────┬───────────────────────────────────────────────┘
                      │
                      ▼
            ┌─────────────────┐
            │  Load Config    │  Read clusters.json
            │  Resolve $ENV   │  Resolve API key env vars
            └────────┬────────┘
                     │
         ┌───────────┼───────────┐
         ▼           ▼           ▼
    ┌─────────┐ ┌─────────┐ ┌─────────┐
    │Cluster 1│ │Cluster 2│ │Cluster N│  (sequential or --workers N)
    └────┬────┘ └────┬────┘ └────┬────┘
         │           │           │
         ▼           ▼           ▼
    ┌──────────────────────────────────┐
    │  For each cluster:               │
    │  GET /api/spaces/space           │
    │  → Discover all Kibana spaces    │
    └──────────────┬───────────────────┘
                   │
         ┌─────────┼─────────┐
         ▼         ▼         ▼
    ┌─────────┐ ┌─────┐ ┌─────────┐
    │ Space A │ │  B  │ │ Space N │
    └────┬────┘ └──┬──┘ └────┬────┘
         │         │         │
         ▼         ▼         ▼
    ┌──────────────────────────────────┐
    │  For each space:                 │
    │  1. GET /s/{id}/api/data_views   │
    │     → Get all data views         │
    │                                  │
    │  2. Group by title               │
    │     → Find titles with 2+ IDs   │
    │                                  │
    │  3. GET /api/data_views/default  │
    │     → Detect default data view   │
    │                                  │
    │  4. GET /api/saved_objects/_find │
    │     → Batched: ALL 34 types in   │
    │       ONE request (pagination)   │
    │     → Count refs per data view   │
    └──────────────┬───────────────────┘
                   │
                   ▼
    ┌──────────────────────────────────┐
    │  Label each duplicate:           │
    │                                  │
    │  ├─ is_default? ──► KEEP (DEFAULT)│
    │  ├─ highest refs? ─► KEEP        │
    │  ├─ refs > 0? ─────► REVIEW      │
    │  └─ refs == 0? ────► SAFE TO DEL │
    └──────────────┬───────────────────┘
                   │
                   ▼
    ┌──────────────────────────────────┐
    │  Output Results                  │
    │                                  │
    │  ├── Table report (default)      │
    │  ├── --output csv / json         │
    │  ├── --dry-run-delete preview    │
    │  ├── --top-offenders ranking     │
    │  └── --log-file audit trail      │
    └──────────────────────────────────┘
```

## CLI Reference

| Flag | Description |
|------|-------------|
| `--config FILE` | **(required)** Path to clusters.json |
| `--clusters NAME...` | Scan only these clusters |
| `--output table\|csv\|json` | Output format (default: table) |
| `--output-file PATH` | Custom output file path |
| `--connectivity-check` | Test connectivity only, then exit |
| `--workers N` | Concurrent cluster scanning (default: 1) |
| `--verbose` | Debug-level logging |
| `--dry-run-delete` | Preview which orphans would be deleted with API URLs |
| `--top-offenders` | Rank spaces by duplicate count |
| `--log-file [PATH]` | Write logs to file (auto-timestamped if no path given) |

## Sample Output

```
==========================================================================================
DUPLICATE DATA VIEWS REPORT
==========================================================================================

──────────────────────────────────────────────────────────────────────────────────────────
📦 DEPLOYMENT: FISMA SCORECARD
──────────────────────────────────────────────────────────────────────────────────────────

  🔹 Space: FISMA Team

    Data View Title: dhs_fisma_score_card_reports
    Copies: 27
      ID: 2d18abea-6bf0-4f24-b864-ad57c68c0d39      (2995 refs)  ← KEEP
      ID: aeb90b26-35c3-4ab7-9ab6-bac047be1a8b      (1718 refs)  ← REVIEW (has refs)
      ID: f0ff444b-5dbd-42e7-8805-ff71d466e783      (513 refs)   ← REVIEW (has refs)
      ID: 9a12d4dd-4a2e-43b3-86c5-c3766b5b8884      (1 refs)    ← REVIEW (has refs)
      ID: ff6285bb-1017-458e-bf5c-ad66eecb5806      (1 refs)    ← REVIEW (has refs)

==========================================================================================
SUMMARY
==========================================================================================
  Deployments with duplicates : 7
  Duplicate title groups       : 119
  Total duplicate data view IDs: 335
  Safe to delete (0 refs)      : 142
  Needs review (has refs)      : 74
──────────────────────────────────────────────────────────────────────────────────────────
  Clusters scanned: 9  |  Clean: 2  |  With duplicates: 7  |  Failed: 0
  Total scan time: 1m 24s
==========================================================================================
```

## Performance

| Metric | Before optimization | After optimization |
|--------|--------------------|--------------------|
| API calls per space | 34+ (one per type) | 1 (batched) |
| LayerD (20 spaces) | ~16 minutes | ~22 seconds |
| Full 9-cluster scan | >20 minutes (interrupted) | **1 minute 24 seconds** |

## Requirements

- Python 3.7+
- `requests` library (`pip install requests`)
- Kibana API key with read access to all spaces
