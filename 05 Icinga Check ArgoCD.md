---
tags: [hobby-lobby, argocd, monitoring, icinga, nagios, script]
script: 05_icinga_check_argocd.sh
step: "5 of 7"
description: Nagios/Icinga-compatible check plugin that queries Prometheus for ArgoCD sync status
depends_on: "[[03 Prometheus Alert Rules]]", python3, curl
next: "[[06 Icinga Config and Notifications]]"
---

# 05 — Icinga Check Plugin

A Nagios-compatible check plugin deployed to `/usr/lib/nagios/plugins/`. It queries Prometheus for ArgoCD sync-status metrics and returns standard exit codes that Icinga uses to determine service state.

## Exit Codes

| Code | State | Condition |
|---|---|---|
| 0 | OK | All apps Synced and Healthy |
| 1 | WARNING | One or more apps OutOfSync |
| 2 | CRITICAL | Sync failures or Degraded health |
| 3 | UNKNOWN | Cannot reach Prometheus |

## Performance Data

The plugin appends `|`-separated performance data used for graphing in Icinga/Grafana:

```
out_of_sync=N;warn;crit  sync_failures=N;;crit  unhealthy=N;;crit
```

## Usage

```bash
# Install
cp 05_icinga_check_argocd.sh /usr/lib/nagios/plugins/check_argocd_sync.sh
chmod +x /usr/lib/nagios/plugins/check_argocd_sync.sh

# Test manually
./check_argocd_sync.sh --prometheus-url http://prometheus:9090

# Custom thresholds
./check_argocd_sync.sh \
  --prometheus-url http://prometheus:9090 \
  --warn-threshold 1 \
  --crit-threshold 1 \
  --verbose
```

## Script

```bash
#!/usr/bin/env bash
# =============================================================================
# Script Name  : 05_icinga_check_argocd.sh
# Description  : Icinga 2 / Nagios-compatible check plugin that queries
#                Prometheus for ArgoCD sync-status metrics and returns
#                standard Nagios exit codes:
#                  0  OK       – all applications are Synced and Healthy
#                  1  WARNING  – one or more apps are OutOfSync
#                  2  CRITICAL – one or more apps have Sync Failures or Degraded health
#                  3  UNKNOWN  – cannot reach Prometheus or query returned no data
#
# Author       : Sangyeon Lee  –  Hobby Lobby IT Infrastructure
# Usage        : ./05_icinga_check_argocd.sh \
#                     --prometheus-url http://prometheus:9090 \
#                     [--warn-threshold <int>] [--crit-threshold <int>]
# Exit codes   : 0=OK  1=WARNING  2=CRITICAL  3=UNKNOWN  (Nagios standard)
# =============================================================================

# Do NOT use set -e; Icinga expects non-zero exits to signal alert states
set -uo pipefail

# ---------------------------------------------------------------------------
# NAGIOS EXIT CODE CONSTANTS
# ---------------------------------------------------------------------------
EXIT_OK=0           # Service is healthy
EXIT_WARNING=1      # Minor issue (OutOfSync apps)
EXIT_CRITICAL=2     # Serious issue (sync failures / degraded health)
EXIT_UNKNOWN=3      # Plugin cannot determine status

# ---------------------------------------------------------------------------
# DEFAULTS
# ---------------------------------------------------------------------------
PROM_URL="${PROM_URL:-http://localhost:9090}"    # Prometheus base URL
WARN_THRESHOLD="${WARN_THRESHOLD:-1}"            # Warn when N or more apps OutOfSync
CRIT_THRESHOLD="${CRIT_THRESHOLD:-1}"            # Crit when N or more apps have failures
APP_FILTER="${APP_FILTER:-.*}"                   # Regex to filter app names (default: all)
TIMEOUT=10                                       # HTTP request timeout in seconds
VERBOSE=false                                    # Set true for extra debug output

while [[ $# -gt 0 ]]; do
  case "$1" in
    --prometheus-url|-u)  PROM_URL="$2";         shift 2 ;;
    --warn-threshold|-w)  WARN_THRESHOLD="$2";   shift 2 ;;
    --crit-threshold|-c)  CRIT_THRESHOLD="$2";   shift 2 ;;
    --app-filter|-f)      APP_FILTER="$2";        shift 2 ;;
    --timeout|-t)         TIMEOUT="$2";           shift 2 ;;
    --verbose|-v)         VERBOSE=true;           shift   ;;
    --help|-h)
      echo "Usage: $0 [--prometheus-url URL] [--warn-threshold N] [--crit-threshold N]"
      exit $EXIT_OK ;;
    *)
      echo "UNKNOWN: Unrecognised argument '$1'"
      exit $EXIT_UNKNOWN ;;
  esac
done

# ---------------------------------------------------------------------------
# HELPERS
# ---------------------------------------------------------------------------

# debug: Only print if --verbose flag was passed
debug() {
  [[ "$VERBOSE" == "true" ]] && echo "[DEBUG] $*" >&2 || true
}

# prom_query: Execute an instant Prometheus query, return the scalar value
# Returns "ERROR" on network failure or parse error
prom_query() {
  local expr="$1"

  # URL-encode the PromQL expression for use as a query parameter
  local encoded
  encoded=$(python3 -c \
    "import urllib.parse, sys; print(urllib.parse.quote(sys.argv[1]))" \
    "$expr" 2>/dev/null) || { echo "ERROR"; return; }

  debug "Querying: ${PROM_URL}/api/v1/query?query=${encoded}"

  # Send the HTTP GET; -sf = silent + fail on HTTP error
  local response
  response=$(curl -sf --max-time "$TIMEOUT" \
    "${PROM_URL}/api/v1/query?query=${encoded}" 2>/dev/null) || { echo "ERROR"; return; }

  # Extract numeric value from Prometheus JSON response
  # Format: {"data":{"result":[{"value":[timestamp,"VALUE"]}]}}
  python3 -c "
import json, sys
data = json.load(sys.stdin)
results = data.get('data', {}).get('result', [])
if not results:
    print('0')      # No results = metric not present = treat as 0
    sys.exit(0)
print(results[0]['value'][1])    # Return the value from the first result row
" <<< "$response" 2>/dev/null || { echo "ERROR"; return; }
}

# ---------------------------------------------------------------------------
# STEP 1: CHECK PROMETHEUS CONNECTIVITY
# ---------------------------------------------------------------------------
if ! curl -sf --max-time "$TIMEOUT" \
    "${PROM_URL}/api/v1/query?query=1" &>/dev/null; then
  echo "UNKNOWN: Cannot reach Prometheus at ${PROM_URL} (timeout: ${TIMEOUT}s)"
  exit $EXIT_UNKNOWN
fi

# ---------------------------------------------------------------------------
# STEP 2: QUERY OUT-OF-SYNC COUNT
# Uses recording rule if available, falls back to full PromQL expression
# ---------------------------------------------------------------------------
OUT_OF_SYNC_COUNT=$(prom_query \
  "argocd:apps:out_of_sync_count or count(argocd_app_info{sync_status!=\"Synced\"}) or vector(0)")

debug "Out-of-sync count: $OUT_OF_SYNC_COUNT"

if [[ "$OUT_OF_SYNC_COUNT" == "ERROR" ]]; then
  echo "UNKNOWN: Prometheus query for sync status failed"
  exit $EXIT_UNKNOWN
fi

OUT_OF_SYNC_INT=$(printf "%.0f" "$OUT_OF_SYNC_COUNT" 2>/dev/null || echo "0")

# ---------------------------------------------------------------------------
# STEP 3: QUERY SYNC-FAILURE COUNT
# Counts apps with at least one Failed/Error sync in the last 10 minutes
# ---------------------------------------------------------------------------
SYNC_FAIL_COUNT=$(prom_query \
  "count(increase(argocd_app_sync_total{phase=~\"Failed|Error\"}[10m]) > 0) or vector(0)")

debug "Sync-failure count: $SYNC_FAIL_COUNT"
[[ "$SYNC_FAIL_COUNT" == "ERROR" ]] && SYNC_FAIL_COUNT=0   # Treat error as 0
SYNC_FAIL_INT=$(printf "%.0f" "$SYNC_FAIL_COUNT" 2>/dev/null || echo "0")

# ---------------------------------------------------------------------------
# STEP 4: QUERY DEGRADED / UNHEALTHY COUNT
# ---------------------------------------------------------------------------
UNHEALTHY_COUNT=$(prom_query \
  "argocd:apps:unhealthy_count or count(argocd_app_info{health_status=~\"Degraded|Missing\"}) or vector(0)")

debug "Unhealthy count: $UNHEALTHY_COUNT"
UNHEALTHY_INT=$(printf "%.0f" "$UNHEALTHY_COUNT" 2>/dev/null || echo "0")

# ---------------------------------------------------------------------------
# STEP 5: GET NAMES OF OUT-OF-SYNC APPS FOR HUMAN-READABLE OUTPUT
# ---------------------------------------------------------------------------
OOS_APP_NAMES=$(python3 -c "
import json, sys
raw = sys.argv[1]
try:
    data = json.loads(raw)
    results = data.get('data', {}).get('result', [])
    names = [r['metric'].get('name', '?') for r in results]
    print(', '.join(names) if names else 'none')
except Exception:
    print('unknown')
" "$(curl -sf --max-time "$TIMEOUT" \
  "${PROM_URL}/api/v1/query?query=$(python3 -c \
  'import urllib.parse; print(urllib.parse.quote("argocd_app_info{sync_status!=\"Synced\"}"))')" \
  2>/dev/null || echo '{}')" 2>/dev/null || echo "unknown")

# ---------------------------------------------------------------------------
# STEP 6: DETERMINE EXIT STATE
# Performance data format: metric=value;warn;crit;min
# ---------------------------------------------------------------------------
PERF_DATA="out_of_sync=${OUT_OF_SYNC_INT};${WARN_THRESHOLD};${CRIT_THRESHOLD};0 \
sync_failures=${SYNC_FAIL_INT};;${CRIT_THRESHOLD};0 \
unhealthy=${UNHEALTHY_INT};;${CRIT_THRESHOLD};0"

# CRITICAL: sync failures or degraded apps require immediate action
if (( SYNC_FAIL_INT >= CRIT_THRESHOLD )) || (( UNHEALTHY_INT >= CRIT_THRESHOLD )); then
  echo "CRITICAL: ${SYNC_FAIL_INT} sync failure(s), ${UNHEALTHY_INT} degraded, \
${OUT_OF_SYNC_INT} out-of-sync | ${PERF_DATA}"
  exit $EXIT_CRITICAL
fi

# WARNING: apps out of sync exceed threshold
if (( OUT_OF_SYNC_INT >= WARN_THRESHOLD )); then
  echo "WARNING: ${OUT_OF_SYNC_INT} app(s) out of sync (${OOS_APP_NAMES}) | ${PERF_DATA}"
  exit $EXIT_WARNING
fi

# ALL CLEAR
echo "OK: All ArgoCD applications are Synced and Healthy | ${PERF_DATA}"
exit $EXIT_OK
```
