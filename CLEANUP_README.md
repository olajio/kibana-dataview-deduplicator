# Cleanup Duplicate Data Views — Safe Multi-Cluster Remediation

Safely removes duplicate data views across Kibana deployments by re-pointing references, backing up objects, validating with the user, and deleting only confirmed orphans.

This script is the companion to `find_duplicate_dataviews.py` (scanner) and uses the same configuration, API engine, and batched-request optimizations.

## Safety Features

| Safety Layer | Description |
|-------------|-------------|
| **Dry-run by default** | No changes unless `--execute` is explicitly passed |
| **Full space backup** | NDJSON export of all objects before any modifications |
| **Per-data-view backup** | Individual NDJSON backup before each deletion |
| **Reference re-pointing** | Migrates refs from duplicates → KEEP before deleting |
| **Post-repoint verification** | Confirms 0 references remain before each delete |
| **Default data view protection** | Never deletes the space default (breaks Discover) |
| **Interactive validation** | User approves cleanup plan before execution |
| **Comprehensive audit log** | Every action logged to timestamped file |
| **Item-by-item approval** | Optional granular per-group confirmation |

## Quick Start

```bash
# Dry-run (default) — preview cleanup plan, no changes
python cleanup_duplicate_dataviews.py --config clusters.json

# Dry-run for specific cluster + space
python cleanup_duplicate_dataviews.py --config clusters.json \
    --clusters "FISMA Scorecard" --spaces "FISMA Team"

# Execute with interactive validation
python cleanup_duplicate_dataviews.py --config clusters.json --execute

# Execute with auto-confirm (no prompts — for CI/CD)
python cleanup_duplicate_dataviews.py --config clusters.json --execute --yes

# Custom backup directory and log file
python cleanup_duplicate_dataviews.py --config clusters.json --execute \
    --backup-dir /backups/kibana --log-file cleanup_audit.log
```

## Workflow

```
┌─────────────────────────────────────────────────────────────────────┐
│                 cleanup_duplicate_dataviews.py                       │
└─────────────────────┬───────────────────────────────────────────────┘
                      │
                      ▼
            ┌─────────────────┐
            │  Load Config    │  Same clusters.json as scanner
            │  Resolve $ENV   │
            └────────┬────────┘
                     │
                     ▼
    ┌──────────────────────────────────┐
    │  For each cluster + space:       │
    │                                  │
    │  1. GET all data views           │
    │  2. Find duplicates by title     │
    │  3. GET default data view ID     │
    │  4. GET all saved objects        │
    │     (batched, 1 call per space)  │
    │  5. Count refs per data view     │
    └──────────────┬───────────────────┘
                   │
                   ▼
    ┌──────────────────────────────────┐
    │  Build Cleanup Plan              │
    │                                  │
    │  For each duplicate group:       │
    │                                  │
    │  ┌─ default? ────► KEEP (DEFAULT)│
    │  ├─ highest refs? ► KEEP         │
    │  ├─ refs > 0? ───► REPOINT + DEL │
    │  └─ refs == 0? ──► DELETE        │
    └──────────────┬───────────────────┘
                   │
                   ▼
    ┌──────────────────────────────────┐
    │  Present Plan to User            │
    │                                  │
    │  Shows for each group:           │
    │  - KEEP candidate (ID + refs)    │
    │  - Each duplicate + action       │
    │  - Total repoints + deletions    │
    └──────────────┬───────────────────┘
                   │
          ┌────────┴────────┐
          │   DRY-RUN?      │
          ├── Yes ──► STOP  │  (no changes made)
          └── No ───┬───────┘
                    │
                    ▼
    ┌──────────────────────────────────┐
    │  User Approval                   │
    │                                  │
    │  Options:                        │
    │  - 'y'            → approve all  │
    │  - 'item-by-item' → per-group   │
    │  - 'n' / Enter    → cancel all  │
    │  - --yes flag     → auto-approve │
    └──────────────┬───────────────────┘
                   │
                   ▼
    ┌──────────────────────────────────┐
    │  BACKUP Space                    │
    │  Export ALL objects → NDJSON     │
    └──────────────┬───────────────────┘
                   │
                   ▼
    ┌──────────────────────────────────┐
    │  For each approved duplicate:    │
    │                                  │
    │  1. REPOINT references           │
    │     PUT /api/saved_objects/...   │
    │     (old_id → keep_id)           │
    │                                  │
    │  2. BACKUP data view             │
    │     POST /api/saved_objects/     │
    │       _export → .ndjson          │
    │                                  │
    │  3. VERIFY 0 refs remaining      │
    │     (abort if refs still exist)  │
    │                                  │
    │  4. DELETE data view             │
    │     DELETE /api/data_views/      │
    │       data_view/{id}             │
    └──────────────┬───────────────────┘
                   │
                   ▼
    ┌──────────────────────────────────┐
    │  Summary Report                  │
    │  + Audit log file                │
    └──────────────────────────────────┘
```

## CLI Reference

| Flag | Description |
|------|-------------|
| `--config FILE` | **(required)** Path to clusters.json |
| `--clusters NAME...` | Process only these clusters |
| `--spaces NAME...` | Process only these space IDs or names |
| `--execute` | Actually apply changes (default is dry-run) |
| `--yes` | Auto-confirm all deletions (skip prompts) |
| `--verbose` | Debug-level logging |
| `--log-file [PATH]` | Audit log file (default: auto-timestamped) |
| `--backup-dir DIR` | Backup directory (default: ./backups) |

## Sample Dry-Run Output

```
==========================================================================================
🔒 DRY-RUN MODE — No changes will be made. Use --execute to apply changes.
==========================================================================================

==========================================================================================
CLEANUP PLAN — [DRY-RUN]
==========================================================================================

  📦 FISMA SCORECARD > DEFEND A
    Data View Title: cdm_hwam_current
    KEEP:   dv-hwam-001                                    (47 refs) ← DEFAULT
    DELETE: dv-hwam-002                                    (3 refs → repoint to KEEP, then delete)
    DELETE: dv-hwam-003                                    (0 refs → delete)

  📦 FISMA SCORECARD > DEFEND A
    Data View Title: cdm_vuln_current
    KEEP:   dv-vuln-001                                    (15 refs)
    DELETE: dv-vuln-002                                    (2 refs → repoint to KEEP, then delete)

──────────────────────────────────────────────────────────────────────────────────────────
  Total reference re-points : 5
  Total data views to delete: 3
==========================================================================================
```

## Sample Execute Output

```
==========================================================================================
⚠️  EXECUTE MODE — Changes WILL be applied to your Kibana deployments!
==========================================================================================

  ✅ Space backup saved: backups/space_fisma-team_20260228_015000.ndjson (80 objects)

  Re-pointing 10 references: ft-report-002 → ft-report-001
    ✅ Repointed visualization/ft-viz-0: ft-report-002 → ft-report-001
    ✅ Repointed visualization/ft-viz-1: ft-report-002 → ft-report-001
    ...
    Backup: backups/dataview_ft-report-002.ndjson
    ✅ DELETED data view: ft-report-002
    Backup: backups/dataview_ft-report-003.ndjson
    ✅ DELETED data view: ft-report-003

==========================================================================================
CLEANUP SUMMARY
==========================================================================================
  References re-pointed : 10
  Data views deleted    : 3
  Skipped (default/safe): 0
  Spaces backed up      : 1
  Errors                : 0
  Total time            : 12.3s
  Audit log             : cleanup_dataviews_20260228_015000.log
==========================================================================================
```

## Backup & Restore

All backups are NDJSON files compatible with the Kibana Import API:

```bash
# List backups
ls -la backups/

# Restore a space backup
curl -X POST "https://kibana:9243/s/fisma-team/api/saved_objects/_import?overwrite=true" \
  -H "kbn-xsrf: true" -H "Authorization: ApiKey YOUR_KEY" \
  --form file=@backups/space_fisma-team_20260228_015000.ndjson

# Restore a single data view
curl -X POST "https://kibana:9243/s/fisma-team/api/saved_objects/_import" \
  -H "kbn-xsrf: true" -H "Authorization: ApiKey YOUR_KEY" \
  --form file=@backups/dataview_ft-report-002.ndjson
```

## Requirements

- Python 3.7+
- `requests` library (`pip install requests`)
- Kibana API key with **read + write** access to spaces (data views + saved objects)
- Same `clusters.json` config used by `find_duplicate_dataviews.py`
