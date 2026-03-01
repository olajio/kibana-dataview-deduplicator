# Kibana Data View Deduplicator

Detect, analyze, and safely remediate duplicate data views across all your Kibana deployments.

## Why This Matters

Duplicate data views are a common problem in Kibana. They appear when users manually create data views that already exist, when Kibana objects are imported multiple times, or when spaces are cloned without cleanup. Left unchecked, duplicates lead to:

- **Performance degradation** — dashboards referencing wildcard patterns like `*:filebeat-*` across duplicate views multiply search load on the cluster
- **User confusion** — team members see multiple identically-named data views and don't know which one to use, leading to inconsistent dashboards
- **Maintenance overhead** — updating or deprecating a data view pattern requires tracking down every copy across every space
- **SLA risk** — in highly regulated environments, unnecessary cluster load from duplicates can push response times past SLA thresholds

This toolkit gives you a two-command way to audit and fix all of this across 100+ deployments.

---

## The Toolkit

| Script | Purpose | Mode |
|--------|---------|------|
| `find_duplicate_dataviews.py` | **Scanner** — detect, label, and report duplicates | Read-only |
| `cleanup_duplicate_dataviews.py` | **Cleanup** — repoint references, backup, and delete | Read-write |

Both scripts share the same `clusters.json` configuration, the same batched API engine, and the same retry/backoff logic. Run the scanner first to understand the landscape, then run cleanup to fix it.

---

## Table of Contents

- [Why This Matters](#why-this-matters)
- [The Toolkit](#the-toolkit)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
- [Part 1 — Scanner (`find_duplicate_dataviews.py`)](#part-1--scanner-find_duplicate_dataviewspy)
  - [What It Does](#what-it-does)
  - [How It Works](#how-it-works)
  - [Scanner CLI Reference](#scanner-cli-reference)
  - [Scanner Examples](#scanner-examples)
  - [Scanner Sample Output](#scanner-sample-output)
  - [Interpreting Results](#interpreting-results)
  - [Performance](#performance)
- [Part 2 — Cleanup (`cleanup_duplicate_dataviews.py`)](#part-2--cleanup-cleanup_duplicate_dataviewspy)
  - [Why a Separate Script](#why-a-separate-script)
  - [Safety Features](#safety-features)
  - [How It Works](#how-it-works-1)
  - [Cleanup CLI Reference](#cleanup-cli-reference)
  - [Cleanup Examples](#cleanup-examples)
  - [Cleanup Sample Output](#cleanup-sample-output)
  - [Backup & Restore](#backup--restore)
- [End-to-End Workflow](#end-to-end-workflow)
- [Managing 100+ Deployments](#managing-100-deployments)
- [Troubleshooting](#troubleshooting)
- [Requirements](#requirements)

---

## Quick Start

### 1. Create your config file

Create a `clusters.json` file with your deployment details:

```json
{
  "clusters": {
    "prod": {
      "kibana_url": "https://prod-kibana:5601",
      "api_key": "$PROD_KIBANA_API_KEY",
      "verify_ssl": false
    },
    "qa": {
      "kibana_url": "https://qa-kibana:5601",
      "api_key": "$QA_KIBANA_API_KEY",
      "verify_ssl": false
    }
  }
}
```

### 2. Set your API keys

```bash
export PROD_KIBANA_API_KEY="your-prod-api-key-here"
export QA_KIBANA_API_KEY="your-qa-api-key-here"
```

Or store them in a `.env` file and `source .env` before running.

### 3. Test connectivity

```bash
python find_duplicate_dataviews.py --connectivity-check
```

Expected output:

```
🔌 CONNECTIVITY CHECK
============================================================
  ✅ prod                 — Connected (12 spaces)
  ✅ qa                   — Connected (8 spaces)

  Result: 2/2 clusters reachable
```

### 4. Scan for duplicates

```bash
python find_duplicate_dataviews.py --dry-run-delete --top-offenders
```

### 5. Clean up safely

```bash
# Dry-run first (default — no changes)
python cleanup_duplicate_dataviews.py \
    --clusters "prod" --spaces "Team Alpha"

# Then execute when ready
python cleanup_duplicate_dataviews.py \
    --clusters "prod" --spaces "Team Alpha" --execute
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

Each cluster entry requires:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `kibana_url` | string | Yes | Kibana endpoint URL (trailing slashes stripped automatically) |
| `api_key` | string | Yes | Kibana API key — direct value or `$ENV_VAR` reference |
| `verify_ssl` | boolean | No | Whether to verify SSL certificates (default: `true`) |
| `description` | string | No | Human-readable label for logs |

API keys starting with `$` are resolved from environment variables at runtime:

```bash
# Set API keys as environment variables
export LAYERD_API_KEY="your-base64-encoded-api-key-here"
export FISMA_API_KEY="another-base64-encoded-api-key-here"

# Verify they're set
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

## Part 1 — Scanner (`find_duplicate_dataviews.py`)

### What It Does

`find_duplicate_dataviews.py` scans all clusters and spaces, identifies data views with duplicate titles (same title, different IDs), counts how many dashboards, visualizations, and other saved objects reference each copy, and labels every duplicate with an actionable recommendation.

The reference count is critical — it tells you which duplicate is actively used and which is safe to remove.

| Feature | Description |
|---------|-------------|
| **Multi-cluster scanning** | Scans all deployments and spaces from `clusters.json` automatically |
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

### How It Works

For each configured cluster, the script:

1. Connects to the Kibana API and discovers all spaces automatically
2. Retrieves all data views in each space
3. Groups data views by title and flags duplicates (same title, multiple IDs)
4. Fetches the space's default data view ID
5. Counts how many saved objects (dashboards, visualizations, lenses, etc.) reference each duplicate — using a single batched API call per space
6. Labels each duplicate: **KEEP**, **REVIEW**, or **SAFE TO DELETE**
7. Outputs a consolidated report

```
┌─────────────────────────────────────────────────────────────────────┐
│                    find_duplicate_dataviews.py                       │
└─────────────────────┬───────────────────────────────────────────────┘
                      │
                      ▼
            ┌─────────────────┐
            │  Load Config    │  Read clusters.json (default)
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
| `--clusters NAME...` | Scan only these clusters (default: scan all) |
| `--spaces NAME...` | Scan only these space IDs or names (default: scan all spaces) |
| `--output table\|csv\|json` | Output format (default: table) |
| `--output-file PATH` | Custom export file path (default: auto-timestamped) |
| `--connectivity-check` | Test connectivity to all clusters, then exit |
| `--workers N` | Concurrent cluster scanning (default: 1) |
| `--verbose` | Debug-level logging |
| `--dry-run-delete` | Preview which orphans would be deleted, with exact API URLs |
| `--top-offenders` | Rank spaces by duplicate count |
| `--log-file [PATH]` | Write logs to file (auto-timestamped if no path given) |

### Scanner Examples

```bash
# ── Basic scanning ───────────────────────────────────────────────────
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

# Scan specific spaces within a cluster
python find_duplicate_dataviews.py \
    --clusters "FISMA Scorecard" --spaces "FISMA Team"

# Scan multiple spaces across a cluster
python find_duplicate_dataviews.py \
    --clusters "FISMA Scorecard" --spaces "FISMA Team" "DEFEND A"

# Combine cluster + space filters with analysis
python find_duplicate_dataviews.py \
    --clusters "FISMA Scorecard" --spaces "FISMA Team" \
    --dry-run-delete --top-offenders

# ── Analysis & preview ───────────────────────────────────────────────
# Preview which orphaned data views (0 refs, not default) would be deleted
# Shows the exact DELETE API URL for each — no changes are made
python find_duplicate_dataviews.py --dry-run-delete

# Show top offender spaces ranked by duplicate count
python find_duplicate_dataviews.py --top-offenders

# Combine both for a full analysis report
python find_duplicate_dataviews.py --dry-run-delete --top-offenders

# Full analysis of one cluster with log file
python find_duplicate_dataviews.py \
    --clusters "FISMA Scorecard" \
    --dry-run-delete --top-offenders --log-file

# ── Export results ───────────────────────────────────────────────────
# Export to CSV for spreadsheet review
python find_duplicate_dataviews.py --output csv

# Export to JSON for programmatic use
python find_duplicate_dataviews.py --output json

# Export CSV to a specific file path
python find_duplicate_dataviews.py --output csv \
    --output-file /reports/duplicates_2026-03.csv

# Scan specific clusters and export CSV with log
python find_duplicate_dataviews.py \
    --clusters "FISMA Scorecard" "(LayerD) ES Federal" \
    --output csv --log-file

# ── Performance tuning ───────────────────────────────────────────────
# Scan clusters in parallel (5 concurrent workers)
python find_duplicate_dataviews.py --workers 5

# Fast parallel scan with all analysis features
python find_duplicate_dataviews.py --workers 5 \
    --dry-run-delete --top-offenders --log-file

# ── Connectivity & debugging ─────────────────────────────────────────
# Test connectivity to all clusters without scanning
python find_duplicate_dataviews.py --connectivity-check

# Enable verbose debug logging (shows every API call and response)
python find_duplicate_dataviews.py --verbose

# Verbose scan of one cluster with log file
python find_duplicate_dataviews.py \
    --clusters "(LayerD) ES Federal" \
    --verbose --log-file /var/log/dataview_scan.log

# ── Logging ──────────────────────────────────────────────────────────
# Auto-timestamped log file (e.g. duplicate_dataviews_20260301_143000.log)
python find_duplicate_dataviews.py --log-file

# Log to a specific path
python find_duplicate_dataviews.py --log-file scan_results.log

# ── Cron / automation ────────────────────────────────────────────────
# Weekly scheduled scan: export JSON, write log
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
      ID: 9a12d4dd-4a2e-43b3-86c5-c3766b5b8884      (1 refs)    ← REVIEW (has refs)
      ID: ff6285bb-1017-458e-bf5c-ad66eecb5806      (0 refs)    ← SAFE TO DELETE

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

### Interpreting Results

The **KEEP / REVIEW / SAFE TO DELETE** labels tell you exactly what to do:

| Label | Meaning | Action |
|-------|---------|--------|
| **KEEP** | Highest reference count in the group | This is the "real" data view. Preserve it. |
| **KEEP (DEFAULT)** | The space's default data view | Protected — never delete (would break Discover). |
| **REVIEW** | Has references but isn't the top | Investigate. The cleanup script re-points these refs automatically. |
| **SAFE TO DELETE** | Zero references, not default | Orphaned duplicate. Safe to remove. |

These labels map directly to what the cleanup script does:

| Scanner Label | Cleanup Action |
|---------------|---------------|
| **KEEP** | Preserved as the target — all references re-pointed to this ID |
| **KEEP (DEFAULT)** | Preserved + protected — never touched |
| **REVIEW (has refs)** | References migrated to KEEP candidate, then data view deleted |
| **SAFE TO DELETE** | Deleted directly (with backup) |

### Performance

| Metric | Before optimization | After optimization |
|--------|--------------------|--------------------|
| API calls per space | 34+ (one per type) | 1 (batched) |
| LayerD (20 spaces) | ~16 minutes | ~22 seconds |
| Full 9-cluster scan | >20 minutes (interrupted) | **1 minute 24 seconds** |

The key optimization: instead of making 34 separate API calls per space (one per saved-object type), the script sends all types in a single batched `_find` request using repeated `type` query parameters. This reduces API calls by 97% and makes 100+ deployment scans practical.

---

## Part 2 — Cleanup (`cleanup_duplicate_dataviews.py`)

### Why a Separate Script

The scanner is read-only by design — you can run it freely without risk, share reports with stakeholders, and decide what to clean up. The cleanup script is the one that makes changes, and it's built with multiple safety layers to ensure those changes are correct and reversible.

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
| **Interactive validation** | User approves the full plan — or item-by-item — before execution |
| **Automatic audit log** | Every action logged to a timestamped file by default |

### How It Works

The cleanup script uses the same scan engine as the scanner, then extends it with a 7-step remediation pipeline:

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
| `--clusters NAME...` | Process only these clusters (default: all) |
| `--spaces NAME...` | Process only these space IDs or names (default: all) |
| `--execute` | Actually apply changes (default is dry-run) |
| `--yes` | Auto-confirm all deletions (skip interactive prompts) |
| `--verbose` | Debug-level logging |
| `--log-file [PATH]` | Audit log file (default: auto-timestamped — always created) |
| `--backup-dir DIR` | Backup directory (default: `./backups`) |

### Cleanup Examples

```bash
# ── Dry-run (safe preview — default) ─────────────────────────────────
# Preview cleanup plan for ALL clusters and spaces — no changes made
python cleanup_duplicate_dataviews.py

# Preview for a specific cluster
python cleanup_duplicate_dataviews.py --clusters "FISMA Scorecard"

# Preview for a specific cluster and space
python cleanup_duplicate_dataviews.py \
    --clusters "FISMA Scorecard" --spaces "FISMA Team"

# Preview multiple spaces within a cluster
python cleanup_duplicate_dataviews.py \
    --clusters "FISMA Scorecard" --spaces "FISMA Team" "DEFEND A"

# Dry-run with verbose output to see every API call
python cleanup_duplicate_dataviews.py \
    --clusters "FISMA Scorecard" --spaces "FISMA Team" --verbose

# ── Execute (apply changes) ──────────────────────────────────────────
# Execute with interactive approval (prompts before each action)
python cleanup_duplicate_dataviews.py \
    --clusters "FISMA Scorecard" --spaces "FISMA Team" --execute

# Execute with auto-confirm — skip all interactive prompts
python cleanup_duplicate_dataviews.py \
    --clusters "FISMA Scorecard" --spaces "FISMA Team" --execute --yes

# Execute across ALL clusters and spaces (use with caution)
python cleanup_duplicate_dataviews.py --execute

# ── Backup options ───────────────────────────────────────────────────
# Save NDJSON backups to a custom directory
python cleanup_duplicate_dataviews.py \
    --clusters "FISMA Scorecard" --spaces "FISMA Team" \
    --execute --backup-dir /backups/fisma_20260301

# ── Logging ──────────────────────────────────────────────────────────
# Audit logs are created automatically by default (auto-timestamped)
# You can specify a custom path:
python cleanup_duplicate_dataviews.py \
    --clusters "FISMA Scorecard" --execute \
    --log-file /var/log/cleanup_fisma.log

# ── Full production run ──────────────────────────────────────────────
# Complete cleanup: targeted, verbose, custom backups and log
python cleanup_duplicate_dataviews.py \
    --clusters "FISMA Scorecard" --spaces "FISMA Team" \
    --execute --verbose \
    --log-file cleanup_fisma_team.log \
    --backup-dir /backups/fisma_team_20260301

# ── Using a custom config ────────────────────────────────────────────
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

==========================================================================================
CLEANUP DRY-RUN SUMMARY
==========================================================================================
  References re-pointed : 0
  Data views deleted    : 0
  Skipped (default/safe): 0
  Spaces backed up      : 0
  Errors                : 0
  Total time            : 2.1s
  Audit log             : cleanup_dataviews_20260301_143000.log
==========================================================================================
```

**Execute with `--yes` (auto-confirm):**

```
==========================================================================================
⚠️  EXECUTE MODE — Changes WILL be applied to your Kibana deployments!
==========================================================================================

==========================================================================================
CLEANUP PLAN — [EXECUTE]
==========================================================================================

  📦 FISMA SCORECARD > FISMA Team
    Data View Title: dhs_fisma_score_card_reports
    KEEP:   ft-report-001                                  (50 refs)
    DELETE: ft-report-002                                  (10 refs → repoint to KEEP, then delete)
    DELETE: ft-report-003                                  (0 refs → delete)

  📦 FISMA SCORECARD > FISMA Team
    Data View Title: dhs_fisma_host_defense
    KEEP:   ft-host-001                                    (20 refs)
    DELETE: ft-host-002                                    (0 refs → delete)

──────────────────────────────────────────────────────────────────────────────────────────
  Total reference re-points : 10
  Total data views to delete: 3
==========================================================================================

  Auto-confirm enabled (--yes): all deletions approved.
  ✅ Space backup saved: backups/space_fisma-team_20260228_015000.ndjson (80 objects)

  Re-pointing 10 references: ft-report-002 → ft-report-001
    ✅ Repointed visualization/ft-viz-0: ft-report-002 → ft-report-001
    ✅ Repointed visualization/ft-viz-1: ft-report-002 → ft-report-001
    ✅ Repointed visualization/ft-viz-2: ft-report-002 → ft-report-001
    ...
    Backup: backups/dataview_ft-report-002.ndjson
    ✅ DELETED data view: ft-report-002
    Backup: backups/dataview_ft-report-003.ndjson
    ✅ DELETED data view: ft-report-003
    Backup: backups/dataview_ft-host-002.ndjson
    ✅ DELETED data view: ft-host-002

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

**Default data view protection** (the default is ALWAYS preserved, even if it has fewer refs):

```
==========================================================================================
CLEANUP PLAN — [DRY-RUN]
==========================================================================================

  📦 DHS > Default
    Data View Title: logs-*
    KEEP:   def-logs-002                                   (2 refs) ← DEFAULT
    DELETE: def-logs-001                                   (5 refs → repoint to KEEP, then delete)

──────────────────────────────────────────────────────────────────────────────────────────
  Total reference re-points : 5
  Total data views to delete: 1
==========================================================================================
```

### Backup & Restore

The cleanup script creates two levels of backups automatically:

- **Space-level backup** — full NDJSON export of all objects before any modifications (your rollback point)
- **Per-data-view backup** — individual export before each deletion (surgical restore)

All backups are NDJSON files compatible with the Kibana Import API:

```bash
# List backups
ls -la backups/

# Restore a full space backup (nuclear option — restores everything)
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
  STEP 1                    STEP 2                    STEP 3                    STEP 4
  ┌─────────────┐           ┌─────────────┐           ┌─────────────┐           ┌─────────────┐
  │   SCAN      │           │   CLEANUP   │           │   CLEANUP   │           │   VERIFY    │
  │  (read-only)│    ──►    │  (dry-run)  │    ──►    │  (execute)  │    ──►    │  (re-scan)  │
  └─────────────┘           └─────────────┘           └─────────────┘           └─────────────┘
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
python find_duplicate_dataviews.py \
    --clusters "FISMA Scorecard" --spaces "FISMA Team"
```
Re-run the scanner on the same cluster and space to confirm the duplicates are gone.

---

## Managing 100+ Deployments

For large environments, recommended practices:

- **Use environment variables for API keys** — keep secrets out of the config file so it can be version-controlled safely
- **Use a `.env` file** — store all `export KEY=value` lines in one file and `source .env` before running
- **Use `--workers 10`** (or higher) — concurrent scanning dramatically reduces total runtime across many clusters
- **Use `--clusters`** to target specific deployments when troubleshooting rather than scanning everything
- **Schedule regular scans** — run via cron with `--output json --log-file` to catch new duplicates early
- **Export to CSV** — track duplicate counts over time or share reports with team leads
- **Clean one space at a time** — use `--clusters` and `--spaces` to target the highest-ROI spaces first, verify results, then move to the next

---

## Troubleshooting

**"Environment variable not set. Skipping cluster."**
The API key references an env var (e.g., `$PROD_KIBANA_API_KEY`) that isn't exported in your shell. Run `export PROD_KIBANA_API_KEY=your-key` or check your `.env` file.

**"Failed to retrieve spaces"**
The API key may lack permissions, the Kibana URL may be wrong, or the cluster may be unreachable. Run `--connectivity-check` to diagnose.

**Timeouts on large clusters**
The script uses a 30-second timeout for most API calls. If you have spaces with very large numbers of saved objects, consider running with fewer concurrent workers to reduce load, or use `--verbose` to see which calls are timing out.

**SSL certificate issues**
Set `verify_ssl` to `false` in `clusters.json` for clusters with self-signed certificates:
```json
{
  "clusters": {
    "my-cluster": {
      "kibana_url": "https://...",
      "api_key": "$MY_KEY",
      "verify_ssl": false
    }
  }
}
```

**Cleanup went wrong — how to restore**
```bash
# Check the audit log to see what was changed
cat cleanup_dataviews_20260301_143000.log

# Restore a full space backup
curl -X POST "https://kibana:9243/s/fisma-team/api/saved_objects/_import?overwrite=true" \
  -H "kbn-xsrf: true" -H "Authorization: ApiKey YOUR_KEY" \
  --form file=@backups/space_fisma-team_20260301_143000.ndjson

# Or restore just one data view
curl -X POST "https://kibana:9243/s/fisma-team/api/saved_objects/_import" \
  -H "kbn-xsrf: true" -H "Authorization: ApiKey YOUR_KEY" \
  --form file=@backups/dataview_dv-hwam-002.ndjson
```

---

## Requirements

- Python 3.7+
- `requests` library (`pip install requests`)
- Kibana API key — read-only for scanner, read+write for cleanup
- Environment variables set for any `$VAR`-style API keys in `clusters.json`
