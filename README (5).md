# Duplicate Data Views Toolkit

Detect, analyze, and safely remediate duplicate data views across all your Kibana deployments — in two steps.

| Script | Role | Mode |
|--------|------|------|
| `find_duplicate_dataviews.py` | **Scanner** — detect and label duplicates | Read-only |
| `cleanup_duplicate_dataviews.py` | **Cleanup** — repoint refs, backup, delete | Read-write |

Both scripts share the same `clusters.json` configuration, the same batched API engine, and the same retry/backoff logic. Run the scanner first to understand the landscape, then run cleanup to fix it.

---

## Table of Contents

- [Quick Start](#quick-start)
- [Configuration](#configuration)
  - [Repository Structure](#repository-structure)
- [Part 1 — Scanner](#part-1--scanner)
  - [What It Does](#what-it-does)
  - [Scanner Workflow](#scanner-workflow)
  - [Scanner CLI Reference](#scanner-cli-reference)
  - [Scanner Examples](#scanner-examples)
  - [Scanner Sample Output](#scanner-sample-output)
  - [How Scanner Labels Map to Cleanup Actions](#how-scanner-labels-map-to-cleanup-actions)
  - [Performance](#performance)
- [Part 2 — Cleanup](#part-2--cleanup)
  - [Safety Features](#safety-features)
  - [Cleanup Workflow](#cleanup-workflow)
  - [Cleanup CLI Reference](#cleanup-cli-reference)
  - [Cleanup Examples](#cleanup-examples)
  - [Cleanup Sample Output](#cleanup-sample-output)
  - [Backup & Restore](#backup--restore)
- [End-to-End Workflow](#end-to-end-workflow)
- [Troubleshooting](#troubleshooting)
- [Requirements](#requirements)

---

## Quick Start

```bash
# ── STEP 1: Scan — understand what you're dealing with ──────────────
# (uses clusters.json in current directory by default)
python find_duplicate_dataviews.py --dry-run-delete --top-offenders

# ── STEP 2: Cleanup — fix it safely ─────────────────────────────────
# Dry-run first (default — no changes)
python cleanup_duplicate_dataviews.py \
    --clusters "FISMA Scorecard" --spaces "FISMA Team"

# Then execute when ready
python cleanup_duplicate_dataviews.py \
    --clusters "FISMA Scorecard" --spaces "FISMA Team" --execute
```

---

## Configuration

Both scripts look for `clusters.json` in the current directory by default. Use `--config /path/to/file.json` to specify a different location.

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

API keys starting with `$` are resolved from environment variables at runtime. Trailing slashes in URLs are stripped automatically.

```bash
# Set API keys as environment variables before running
export LAYERD_API_KEY="your-base64-encoded-api-key-here"
export FISMA_API_KEY="another-base64-encoded-api-key-here"

# Verify keys are set
echo $LAYERD_API_KEY
```

The scanner needs **read-only** API access. The cleanup script needs **read + write** access (data views + saved objects).

### Repository Structure

```
├── find_duplicate_dataviews.py    # Scanner (read-only)
├── cleanup_duplicate_dataviews.py # Cleanup (read-write)
├── clusters.json                  # Cluster configuration
├── README.md                      # This file
└── backups/                       # Created automatically by cleanup script
    ├── space_fisma-team_20260301_143000.ndjson
    ├── dataview_dv-hwam-002.ndjson
    └── ...
```

---

## Part 1 — Scanner

### What It Does

`find_duplicate_dataviews.py` scans all clusters and spaces, identifies data views with duplicate titles, counts how many dashboards/visualizations reference each copy, and labels every duplicate with an actionable recommendation.

| Feature | Description |
|---------|-------------|
| **Multi-cluster scanning** | Scans all deployments and spaces from config automatically |
| **Batched API calls** | 34 object types in 1 request per space (was 34 separate calls) |
| **KEEP / SAFE TO DELETE labels** | Highest-ref-count ID → KEEP; zero-ref orphans → SAFE TO DELETE |
| **Default data view detection** | Calls `/api/data_views/default` — never recommends deleting the default |
| **Dry-run delete preview** | `--dry-run-delete` shows exact DELETE API URLs without making changes |
| **Top offenders ranking** | `--top-offenders` ranks spaces by duplicate count for prioritization |
| **Progress bar** | Real-time terminal progress with ETA, no external dependencies |
| **Retry with backoff** | Automatic retries on timeouts/5xx with exponential backoff |
| **Keyboard interrupt** | Ctrl+C prints partial results instead of a raw traceback |
| **Log file support** | `--log-file` writes to both stdout and file for cron/audit use |
| **Concurrent workers** | `--workers N` for parallel cluster scanning |
| **CSV/JSON export** | `--output csv` or `--output json` with all fields including labels |

### Scanner Workflow

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

### Scanner CLI Reference

| Flag | Description |
|------|-------------|
| `--config FILE` | Path to clusters.json (default: `clusters.json` in current dir) |
| `--clusters NAME...` | Scan only these clusters |
| `--output table\|csv\|json` | Output format (default: table) |
| `--output-file PATH` | Custom export file path |
| `--connectivity-check` | Test connectivity only, then exit |
| `--workers N` | Concurrent cluster scanning (default: 1) |
| `--verbose` | Debug-level logging |
| `--dry-run-delete` | Preview which orphans would be deleted with API URLs |
| `--top-offenders` | Rank spaces by duplicate count |
| `--log-file [PATH]` | Write logs to file (auto-timestamped if no path given) |

### Scanner Examples

```bash
# ── Basic scan ───────────────────────────────────────────────────────
# Scan ALL clusters and spaces (uses clusters.json in current dir)
python find_duplicate_dataviews.py

# Use a custom config file
python find_duplicate_dataviews.py --config /opt/configs/prod_clusters.json

# ── Targeted scanning ────────────────────────────────────────────────
# Scan only specific clusters by name
python find_duplicate_dataviews.py --clusters "FISMA Scorecard"

# Scan multiple specific clusters
python find_duplicate_dataviews.py \
    --clusters "FISMA Scorecard" "(LayerD) ES Federal"

# ── Export results ───────────────────────────────────────────────────
# Export to CSV for spreadsheet review
python find_duplicate_dataviews.py --output csv

# Export to JSON for programmatic use
python find_duplicate_dataviews.py --output json

# Export CSV to a specific file path
python find_duplicate_dataviews.py --output csv \
    --output-file /reports/duplicates_2026-03.csv

# Scan specific clusters and export CSV
python find_duplicate_dataviews.py \
    --clusters "FISMA Scorecard" "(LayerD) ES Federal" \
    --output csv --log-file

# ── Analysis & preview ───────────────────────────────────────────────
# Preview which orphaned data views (0 refs, not default) would be deleted
# Shows the exact DELETE API URL for each — no changes are made
python find_duplicate_dataviews.py --dry-run-delete

# Show top offender spaces ranked by duplicate count — helps prioritize cleanup
python find_duplicate_dataviews.py --top-offenders

# Combine both for a full analysis report
python find_duplicate_dataviews.py --dry-run-delete --top-offenders

# Full analysis of one cluster with log file
python find_duplicate_dataviews.py \
    --clusters "FISMA Scorecard" \
    --dry-run-delete --top-offenders --log-file

# ── Connectivity & debugging ─────────────────────────────────────────
# Test connectivity to all clusters without scanning
python find_duplicate_dataviews.py --connectivity-check

# Enable verbose debug logging (shows every API call and response)
python find_duplicate_dataviews.py --verbose

# Verbose scan of one cluster with log file saved to specific path
python find_duplicate_dataviews.py \
    --clusters "(LayerD) ES Federal" \
    --verbose --log-file /var/log/dataview_scan.log

# ── Performance tuning ───────────────────────────────────────────────
# Scan clusters in parallel (5 concurrent workers)
python find_duplicate_dataviews.py --workers 5

# Fast parallel scan with all analysis features enabled
python find_duplicate_dataviews.py --workers 5 \
    --dry-run-delete --top-offenders --log-file

# ── Logging ──────────────────────────────────────────────────────────
# Auto-timestamped log file (e.g. scan_dataviews_20260301_143000.log)
python find_duplicate_dataviews.py --log-file

# Log to a specific path
python find_duplicate_dataviews.py --log-file scan_results.log

# ── Cron / automation ────────────────────────────────────────────────
# Weekly scheduled scan: export JSON, write log, no terminal interaction
python find_duplicate_dataviews.py \
    --output json --output-file /reports/weekly_scan.json \
    --log-file /var/log/weekly_dataview_scan.log
```

### Scanner Sample Output

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
      ID: ff6285bb-1017-458e-bf5c-ad66eecb5806      (1 refs)    ← REVIEW (has refs)
      ID: 9a12d4dd-4a2e-43b3-86c5-c3766b5b8884      (1 refs)    ← REVIEW (has refs)

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

The **KEEP / REVIEW / SAFE TO DELETE** labels from the scanner map directly to the cleanup script's actions — what the scanner labels, the cleanup script acts on.

### How Scanner Labels Map to Cleanup Actions

| Scanner Label | Cleanup Action | Description |
|---------------|---------------|-------------|
| **KEEP** | Preserved as target | All references are re-pointed to this ID |
| **KEEP (DEFAULT)** | Preserved + protected | Space default — never touched, never deleted |
| **REVIEW (has refs)** | **REPOINT → DELETE** | References migrated to KEEP, then data view deleted |
| **SAFE TO DELETE** | **DELETE** | Zero references — deleted directly (with backup) |

### Performance

| Metric | Before optimization | After optimization |
|--------|--------------------|--------------------|
| API calls per space | 34+ (one per type) | 1 (batched) |
| LayerD (20 spaces) | ~16 minutes | ~22 seconds |
| Full 9-cluster scan | >20 minutes (interrupted) | **1 minute 24 seconds** |

---

## Part 2 — Cleanup

`cleanup_duplicate_dataviews.py` takes what the scanner identifies and safely remediates it: re-pointing references, backing up everything, and deleting only confirmed orphans — with user approval at every step.

### Safety Features

| Safety Layer | Description |
|-------------|-------------|
| **Dry-run by default** | No changes unless `--execute` is explicitly passed |
| **Full space backup** | NDJSON export of all objects before any modification |
| **Per-data-view backup** | Individual NDJSON backup before each deletion |
| **Reference re-pointing** | Migrates dashboard/viz refs from duplicate → KEEP before deleting |
| **Post-repoint verification** | Confirms 0 references remain before issuing DELETE; aborts if not |
| **Default data view protection** | Space default is never deleted (would break Discover) |
| **Interactive validation** | User approves the full plan, or item-by-item, before execution |
| **Comprehensive audit log** | Every action logged to timestamped file automatically |

### Cleanup Workflow

The cleanup script extends the scanner's detection with a 7-step remediation pipeline:

```
┌─────────────────────────────────────────────────────────────────────┐
│              cleanup_duplicate_dataviews.py                          │
└─────────────────────┬───────────────────────────────────────────────┘
                      │
                      ▼
    ┌──────────────────────────────────┐
    │  STEP 1 — Scan                   │
    │  Same engine as the scanner:     │
    │  find duplicates, count refs,    │
    │  detect default data views       │
    └──────────────┬───────────────────┘
                   │
                   ▼
    ┌──────────────────────────────────┐
    │  STEP 2 — Build Cleanup Plan     │
    │                                  │
    │  For each duplicate group:       │
    │  ┌─ default? ────► KEEP (DEFAULT)│
    │  ├─ highest refs? ► KEEP         │
    │  ├─ refs > 0? ───► REPOINT + DEL │
    │  └─ refs == 0? ──► DELETE        │
    └──────────────┬───────────────────┘
                   │
                   ▼
    ┌──────────────────────────────────┐
    │  STEP 3 — Present & Approve      │
    │                                  │
    │  Show full plan to user:         │
    │  - Which IDs to KEEP             │
    │  - Which IDs to REPOINT + DELETE │
    │  - Which IDs to DELETE (0 refs)  │
    │  - Total repoints & deletions    │
    │                                  │
    │  Approval options:               │
    │  'y'            → approve all    │
    │  'item-by-item' → per-group      │
    │  'n' / Enter    → cancel         │
    │  --yes flag     → auto-approve   │
    └──────────┬────┬──────────────────┘
               │    │
      ┌────────┘    └── DRY-RUN? → STOP
      │                 (no changes)
      ▼
    ┌──────────────────────────────────┐
    │  STEP 4 — Backup Space           │
    │  Export ALL objects to NDJSON     │
    │  (full restore point)            │
    └──────────────┬───────────────────┘
                   │
                   ▼
    ┌──────────────────────────────────┐
    │  STEP 5 — Repoint References     │
    │                                  │
    │  For each duplicate with refs:   │
    │  PUT /api/saved_objects/{type}/  │
    │    {id} → update references[]    │
    │    old_data_view → keep_data_view│
    └──────────────┬───────────────────┘
                   │
                   ▼
    ┌──────────────────────────────────┐
    │  STEP 6 — Verify                 │
    │                                  │
    │  Re-check reference count = 0    │
    │  If refs remain → ABORT delete   │
    │  (logs error, skips to next)     │
    └──────────────┬───────────────────┘
                   │
                   ▼
    ┌──────────────────────────────────┐
    │  STEP 7 — Delete                 │
    │                                  │
    │  Backup individual data view     │
    │  DELETE /api/data_views/         │
    │    data_view/{id}                │
    └──────────────┬───────────────────┘
                   │
                   ▼
    ┌──────────────────────────────────┐
    │  Summary Report + Audit Log      │
    └──────────────────────────────────┘
```

### Cleanup CLI Reference

| Flag | Description |
|------|-------------|
| `--config FILE` | Path to clusters.json (default: `clusters.json` in current dir) |
| `--clusters NAME...` | Process only these clusters |
| `--spaces NAME...` | Process only these space IDs or names |
| `--execute` | Actually apply changes (default is dry-run) |
| `--yes` | Auto-confirm all deletions (skip prompts) |
| `--verbose` | Debug-level logging |
| `--log-file [PATH]` | Audit log file (default: auto-timestamped) |
| `--backup-dir DIR` | Backup directory (default: ./backups) |

### Cleanup Examples

```bash
# ── Dry-run (safe preview) ───────────────────────────────────────────
# Preview cleanup plan for ALL clusters and spaces — no changes made
python cleanup_duplicate_dataviews.py

# Preview cleanup plan for a specific cluster
python cleanup_duplicate_dataviews.py --clusters "FISMA Scorecard"

# Preview cleanup plan for a specific cluster and space
python cleanup_duplicate_dataviews.py \
    --clusters "FISMA Scorecard" --spaces "FISMA Team"

# Preview multiple spaces within a cluster
python cleanup_duplicate_dataviews.py \
    --clusters "FISMA Scorecard" --spaces "FISMA Team" "DEFEND A"

# Dry-run with verbose output to see every API call
python cleanup_duplicate_dataviews.py \
    --clusters "FISMA Scorecard" --spaces "FISMA Team" --verbose

# ── Execute (apply changes) ──────────────────────────────────────────
# Execute cleanup with interactive approval (prompts before each action)
python cleanup_duplicate_dataviews.py \
    --clusters "FISMA Scorecard" --spaces "FISMA Team" --execute

# Execute with auto-confirm — skip all interactive prompts
python cleanup_duplicate_dataviews.py \
    --clusters "FISMA Scorecard" --spaces "FISMA Team" --execute --yes

# Execute cleanup across ALL clusters and spaces (use with caution)
python cleanup_duplicate_dataviews.py --execute

# ── Backup options ───────────────────────────────────────────────────
# Save NDJSON backups to a custom directory
python cleanup_duplicate_dataviews.py \
    --clusters "FISMA Scorecard" --spaces "FISMA Team" \
    --execute --backup-dir /backups/fisma_20260301

# ── Logging ──────────────────────────────────────────────────────────
# Auto-timestamped audit log (e.g. cleanup_dataviews_20260301_143000.log)
python cleanup_duplicate_dataviews.py \
    --clusters "FISMA Scorecard" --execute --log-file

# Log to a specific file path
python cleanup_duplicate_dataviews.py \
    --clusters "FISMA Scorecard" --execute \
    --log-file /var/log/cleanup_fisma.log

# ── Full production run ──────────────────────────────────────────────
# Complete cleanup with all safety options: targeted, logged, custom backups
python cleanup_duplicate_dataviews.py \
    --clusters "FISMA Scorecard" --spaces "FISMA Team" \
    --execute --verbose \
    --log-file cleanup_fisma_team.log \
    --backup-dir /backups/fisma_team_20260301

# ── Using a custom config ────────────────────────────────────────────
# Point to a different config file
python cleanup_duplicate_dataviews.py \
    --config /opt/configs/staging_clusters.json --execute
```

### Cleanup Sample Output

**Dry-run (default):**
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

**Execute with --yes:**
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

### Backup & Restore

All backups are NDJSON files compatible with the Kibana Import API. The cleanup script creates two levels of backups:

- **Space-level backup** — full export of all objects before any modifications (your rollback point)
- **Per-data-view backup** — individual export before each deletion (surgical restore)

```bash
# List backups
ls -la backups/

# Restore a full space backup (nuclear option)
curl -X POST "https://kibana:9243/s/fisma-team/api/saved_objects/_import?overwrite=true" \
  -H "kbn-xsrf: true" -H "Authorization: ApiKey YOUR_KEY" \
  --form file=@backups/space_fisma-team_20260228_015000.ndjson

# Restore a single data view
curl -X POST "https://kibana:9243/s/fisma-team/api/saved_objects/_import" \
  -H "kbn-xsrf: true" -H "Authorization: ApiKey YOUR_KEY" \
  --form file=@backups/dataview_ft-report-002.ndjson
```

---

## End-to-End Workflow

Here is the recommended workflow for a production cleanup:

```
  STEP 1                    STEP 2                    STEP 3
  ┌─────────────┐           ┌─────────────┐           ┌─────────────┐
  │   SCAN      │           │   CLEANUP   │           │   CLEANUP   │
  │  (read-only)│    ──►    │  (dry-run)  │    ──►    │  (execute)  │
  └─────────────┘           └─────────────┘           └─────────────┘

  find_duplicate_           cleanup_duplicate_        cleanup_duplicate_
  dataviews.py              dataviews.py              dataviews.py
  --dry-run-delete          (default = dry-run)       --execute
  --top-offenders
```

**Step 1 — Scan and prioritize**
```bash
python find_duplicate_dataviews.py --dry-run-delete --top-offenders --log-file
```
Review the top offenders ranking to decide which spaces to clean first. Export to CSV for team review if needed (`--output csv`).

**Step 2 — Dry-run cleanup on priority space**
```bash
python cleanup_duplicate_dataviews.py \
    --clusters "FISMA Scorecard" --spaces "FISMA Team"
```
Review the cleanup plan. Verify that the KEEP candidates are correct. Share the plan with stakeholders if needed.

**Step 3 — Execute cleanup**
```bash
python cleanup_duplicate_dataviews.py \
    --clusters "FISMA Scorecard" --spaces "FISMA Team" --execute
```
Approve at the interactive prompt (or use `--yes` for CI/CD). Backups are created automatically. Audit log is written for compliance.

**Step 4 — Verify**
```bash
python find_duplicate_dataviews.py --clusters "FISMA Scorecard"
```
Re-run the scanner to confirm the duplicates are gone.

---

## Troubleshooting

```bash
# ── Connection issues ────────────────────────────────────────────────
# Test connectivity to all clusters
python find_duplicate_dataviews.py --connectivity-check

# Test with verbose to see full request/response details
python find_duplicate_dataviews.py --connectivity-check --verbose

# ── API key issues ───────────────────────────────────────────────────
# Verify your environment variable is set
echo $LAYERD_API_KEY

# Test a single cluster to isolate the issue
python find_duplicate_dataviews.py --clusters "(LayerD) ES Federal" --verbose

# ── SSL certificate issues ───────────────────────────────────────────
# Set verify_ssl to false in clusters.json for clusters with self-signed certs
# {
#   "clusters": {
#     "my-cluster": {
#       "kibana_url": "https://...",
#       "api_key": "$MY_KEY",
#       "verify_ssl": false
#     }
#   }
# }

# ── Cleanup went wrong ──────────────────────────────────────────────
# Restore a full space backup
curl -X POST "https://kibana:9243/s/fisma-team/api/saved_objects/_import?overwrite=true" \
  -H "kbn-xsrf: true" -H "Authorization: ApiKey YOUR_KEY" \
  --form file=@backups/space_fisma-team_20260301_143000.ndjson

# Restore a single data view
curl -X POST "https://kibana:9243/s/fisma-team/api/saved_objects/_import" \
  -H "kbn-xsrf: true" -H "Authorization: ApiKey YOUR_KEY" \
  --form file=@backups/dataview_dv-hwam-002.ndjson

# Check the audit log for what was changed
cat cleanup_dataviews_20260301_143000.log
```

---

## Requirements

- Python 3.7+
- `requests` library (`pip install requests`)
- Kibana API key — read-only for scanner, read+write for cleanup
- Environment variables set for any `$VAR`-style API keys in clusters.json
