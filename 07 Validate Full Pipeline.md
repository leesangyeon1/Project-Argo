---
tags: [hobby-lobby, argocd, monitoring, validation, ci, script]
script: 07_validate_full_pipeline.sh
step: "7 of 7"
description: End-to-end smoke test validating all 6 layers of the monitoring pipeline
depends_on: All previous scripts (01–06)
next: Done ✓
---

# 07 — Validate Full Pipeline

End-to-end smoke test that checks every layer of the monitoring pipeline in sequence. Each check prints `✓ PASS`, `⚠ WARN`, or `✗ FAIL`. Exit code equals the number of failures — suitable as a CI/CD gate.

## Layers Tested

| Layer | What Is Checked |
|---|---|
| 1 – ArgoCD | Namespace exists, metrics Service present, `/metrics` reachable |
| 2 – Prometheus | Scrape targets registered and UP, `argocd_app_info` populated |
| 3 – Alert Rules | All 5 alert rules + recording rules loaded |
| 4 – InfluxDB | Health endpoint OK, `argocd_app_info` data present in bucket |
| 5 – Icinga | Plugin installed, output is valid Nagios format, exit code 0-3 |
| 6 – Alertmanager | API reachable, ArgoCD/Icinga routing in config |

## Usage

```bash
# Basic run
bash 07_validate_full_pipeline.sh

# Verbose mode (extra debug output per check)
bash 07_validate_full_pipeline.sh --verbose

# CI usage — exit code = number of failures
bash 07_validate_full_pipeline.sh && echo "Pipeline OK" || echo "Pipeline has failures"

# Override endpoints via environment variables
PROM_HOST=prometheus.monitoring.svc:9090 \
INFLUX_HOST=influxdb.monitoring.svc:8086 \
INFLUX_TOKEN=my-token \
ICINGA_HOST=icinga.internal:5665 \
ICINGA_API_PASS=secret \
bash 07_validate_full_pipeline.sh
```

## Script

```bash
#!/usr/bin/env bash
# =============================================================================
# Script Name  : 07_validate_full_pipeline.sh
# Description  : End-to-end smoke test validating every layer of the ArgoCD
#                monitoring pipeline described in the epic:
#
#                  Layer 1 – ArgoCD     : metrics endpoint reachable
#                  Layer 2 – Prometheus : ArgoCD scrape targets UP
#                  Layer 3 – Prometheus : Alert rules loaded
#                  Layer 4 – InfluxDB   : ArgoCD metrics present in bucket
#                  Layer 5 – Icinga     : Check plugin returns valid exit code
#                  Layer 6 – Alertmanager: Alert routing configured
#
#                Exit code = number of failures (CI/CD gate compatible).
#
# Author       : Sangyeon Lee  –  Hobby Lobby IT Infrastructure
# Usage        : bash 07_validate_full_pipeline.sh [--verbose]
# =============================================================================

set -uo pipefail

# ---------------------------------------------------------------------------
# CONFIGURATION  –  override via env vars
# ---------------------------------------------------------------------------
ARGOCD_NAMESPACE="${ARGOCD_NAMESPACE:-argocd}"
PROM_HOST="${PROM_HOST:-localhost:9090}"
INFLUX_HOST="${INFLUX_HOST:-localhost:8086}"
INFLUX_VERSION="${INFLUX_VERSION:-2}"
INFLUX_BUCKET="${INFLUX_BUCKET:-argocd_metrics}"
INFLUX_TOKEN="${INFLUX_TOKEN:-}"
INFLUX_ORG="${INFLUX_ORG:-hobbylobby}"
ICINGA_HOST="${ICINGA_HOST:-localhost:5665}"
ICINGA_API_USER="${ICINGA_API_USER:-root}"
ICINGA_API_PASS="${ICINGA_API_PASS:-}"
ALERTMANAGER_HOST="${ALERTMANAGER_HOST:-localhost:9093}"
PLUGIN_PATH="/usr/lib/nagios/plugins/check_argocd_sync.sh"
VERBOSE="${VERBOSE:-false}"
HTTP_TIMEOUT=10

# ---------------------------------------------------------------------------
# COUNTERS
# ---------------------------------------------------------------------------
PASS_COUNT=0
FAIL_COUNT=0
WARN_COUNT=0

while [[ $# -gt 0 ]]; do
  case "$1" in
    --verbose|-v) VERBOSE=true; shift ;;
    *) echo "Unknown flag: $1"; exit 1 ;;
  esac
done

# ---------------------------------------------------------------------------
# OUTPUT HELPERS
# ---------------------------------------------------------------------------
GREEN="\033[0;32m"; RED="\033[0;31m"; YELLOW="\033[0;33m"; RESET="\033[0m"

pass()       { PASS_COUNT=$((PASS_COUNT + 1)); echo -e "  ${GREEN}✓ PASS${RESET}  $*"; }
fail()       { FAIL_COUNT=$((FAIL_COUNT + 1)); echo -e "  ${RED}✗ FAIL${RESET}  $*"; }
warn_check() { WARN_COUNT=$((WARN_COUNT + 1)); echo -e "  ${YELLOW}⚠ WARN${RESET}  $*"; }
section()    { echo ""; echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"; echo "  $*"; echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"; }
debug()      { [[ "$VERBOSE" == "true" ]] && echo "    [debug] $*" || true; }

echo "╔══════════════════════════════════════════════════════╗"
echo "║  ArgoCD Monitoring Pipeline – Validation Smoke Test ║"
echo "║  $(date '+%Y-%m-%d %H:%M:%S %Z')                    ║"
echo "╚══════════════════════════════════════════════════════╝"

# ===========================================================================
# LAYER 1: ARGOCD METRICS ENDPOINT
# ===========================================================================
section "Layer 1: ArgoCD Metrics Endpoint"

kubectl get namespace "$ARGOCD_NAMESPACE" &>/dev/null \
  && pass "ArgoCD namespace '$ARGOCD_NAMESPACE' exists." \
  || fail "ArgoCD namespace '$ARGOCD_NAMESPACE' NOT found. Is ArgoCD installed?"

if kubectl get svc argocd-metrics -n "$ARGOCD_NAMESPACE" &>/dev/null; then
  METRICS_PORT=$(kubectl get svc argocd-metrics -n "$ARGOCD_NAMESPACE" \
    -o jsonpath='{.spec.ports[?(@.name=="metrics")].port}' 2>/dev/null)
  pass "argocd-metrics Service found. Port: ${METRICS_PORT}"
else
  fail "argocd-metrics Service missing. Run 01_argocd_metrics_setup.sh"
fi

# Port-forward test against /metrics
kubectl port-forward svc/argocd-metrics 18082:8082 -n "$ARGOCD_NAMESPACE" &>/dev/null &
PF_PID=$!; sleep 3

HTTP_RESP=$(curl -sf --max-time "$HTTP_TIMEOUT" \
  "http://localhost:18082/metrics" 2>/dev/null | head -5 || echo "")
kill "$PF_PID" 2>/dev/null || true

if echo "$HTTP_RESP" | grep -q "argocd_app_info"; then
  pass "ArgoCD /metrics returns argocd_app_info."
elif [[ -n "$HTTP_RESP" ]]; then
  warn_check "/metrics reachable but argocd_app_info not found. Check ArgoCD version."
else
  fail "Cannot reach ArgoCD /metrics via port-forward."
fi

# ===========================================================================
# LAYER 2: PROMETHEUS SCRAPE TARGETS
# ===========================================================================
section "Layer 2: Prometheus Scrape Configuration"

if curl -sf --max-time "$HTTP_TIMEOUT" \
    "http://${PROM_HOST}/api/v1/query?query=1" &>/dev/null; then
  pass "Prometheus API reachable at http://${PROM_HOST}"
else
  fail "Prometheus unreachable. Skipping remaining Prometheus tests."
  PROM_HOST=""
fi

if [[ -n "$PROM_HOST" ]]; then
  TARGETS_JSON=$(curl -sf --max-time "$HTTP_TIMEOUT" \
    "http://${PROM_HOST}/api/v1/targets" 2>/dev/null || echo "{}")

  ARGOCD_TARGET_COUNT=$(echo "$TARGETS_JSON" | python3 -c "
import json, sys
data = json.load(sys.stdin)
active = data.get('data', {}).get('activeTargets', [])
argocd = [t for t in active if 'argocd' in t.get('labels', {}).get('job', '')]
print(len(argocd))
" 2>/dev/null || echo "0")

  (( ARGOCD_TARGET_COUNT >= 1 )) \
    && pass "${ARGOCD_TARGET_COUNT} ArgoCD scrape target(s) registered in Prometheus." \
    || fail "No ArgoCD scrape targets. Run 02_prometheus_scrape_config.sh"

  METRIC_CHECK=$(curl -sf --max-time "$HTTP_TIMEOUT" \
    "http://${PROM_HOST}/api/v1/query?query=count(argocd_app_info)" 2>/dev/null \
    | python3 -c "
import json,sys
d=json.load(sys.stdin); r=d.get('data',{}).get('result',[])
print(r[0]['value'][1] if r else '0')
" 2>/dev/null || echo "0")

  (( $(printf "%.0f" "$METRIC_CHECK") > 0 )) \
    && pass "argocd_app_info has ${METRIC_CHECK} app(s) in Prometheus." \
    || warn_check "argocd_app_info returned 0 apps. Check ArgoCD applications exist."
fi

# ===========================================================================
# LAYER 3: PROMETHEUS ALERT RULES
# ===========================================================================
section "Layer 3: Prometheus Alert Rules"

if [[ -n "$PROM_HOST" ]]; then
  RULES_JSON=$(curl -sf --max-time "$HTTP_TIMEOUT" \
    "http://${PROM_HOST}/api/v1/rules" 2>/dev/null || echo "{}")

  for RULE_NAME in \
      "ArgoCDAppOutOfSync" "ArgoCDAppSyncFailed" \
      "ArgoCDAppHealthDegraded" "ArgoCDAppUnknownStatus" \
      "ArgoCDHighReconciliationErrorRate"; do
    echo "$RULES_JSON" | grep -q "\"$RULE_NAME\"" \
      && pass "Alert rule '$RULE_NAME' loaded." \
      || fail "Alert rule '$RULE_NAME' missing. Run 03_prometheus_alert_rules.sh"
  done

  echo "$RULES_JSON" | grep -q "argocd:apps:out_of_sync_count" \
    && pass "Recording rule 'argocd:apps:out_of_sync_count' active." \
    || warn_check "Recording rule not found (optional but recommended)."
else
  warn_check "Skipping alert rule checks — Prometheus unreachable."
fi

# ===========================================================================
# LAYER 4: INFLUXDB DATA INGESTION
# ===========================================================================
section "Layer 4: InfluxDB Data Ingestion"

if [[ "$INFLUX_VERSION" == "2" ]]; then
  INFLUX_HEALTH=$(curl -sf --max-time "$HTTP_TIMEOUT" \
    "http://${INFLUX_HOST}/health" 2>/dev/null \
    | python3 -c "import json,sys; print(json.load(sys.stdin).get('status','unknown'))" \
    2>/dev/null || echo "unreachable")
  [[ "$INFLUX_HEALTH" == "pass" ]] \
    && pass "InfluxDB 2.x healthy." \
    || fail "InfluxDB 2.x health check failed (status: ${INFLUX_HEALTH})."
else
  PING_CODE=$(curl -so /dev/null -w "%{http_code}" --max-time "$HTTP_TIMEOUT" \
    "http://${INFLUX_HOST}/ping" 2>/dev/null || echo "0")
  [[ "$PING_CODE" == "204" ]] \
    && pass "InfluxDB 1.x ping OK (HTTP 204)." \
    || fail "InfluxDB 1.x ping returned HTTP ${PING_CODE}."
fi

if [[ "$INFLUX_VERSION" == "2" && -n "$INFLUX_TOKEN" ]]; then
  INFLUX_RESULT=$(curl -sf --max-time "$HTTP_TIMEOUT" \
    -H "Authorization: Token ${INFLUX_TOKEN}" \
    -H "Content-Type: application/vnd.flux" \
    --data 'from(bucket: "'"$INFLUX_BUCKET"'") |> range(start: -1h) |> filter(fn: (r) => r._measurement == "argocd_app_info") |> count() |> yield()' \
    "http://${INFLUX_HOST}/api/v2/query?org=${INFLUX_ORG}" 2>/dev/null \
    | grep "_value" | head -1 || echo "")
  [[ -n "$INFLUX_RESULT" ]] \
    && pass "argocd_app_info found in InfluxDB bucket '${INFLUX_BUCKET}'." \
    || warn_check "No argocd_app_info in InfluxDB yet. Allow one scrape cycle."
else
  warn_check "Skipping InfluxDB data check (token not set or 1.x mode)."
fi

# ===========================================================================
# LAYER 5: ICINGA CHECK PLUGIN
# ===========================================================================
section "Layer 5: Icinga Check Plugin"

[[ -x "$PLUGIN_PATH" ]] \
  && pass "Plugin present and executable: $PLUGIN_PATH" \
  || fail "Plugin not found at $PLUGIN_PATH. Run 06_icinga_config_and_notifications.sh"

if [[ -x "$PLUGIN_PATH" ]]; then
  PLUGIN_OUTPUT=$("$PLUGIN_PATH" --prometheus-url "http://${PROM_HOST}" 2>/dev/null || true)
  PLUGIN_EXIT=$?
  debug "Plugin output: $PLUGIN_OUTPUT | Exit: $PLUGIN_EXIT"

  echo "$PLUGIN_OUTPUT" | grep -qE "^(OK|WARNING|CRITICAL|UNKNOWN):" \
    && pass "Plugin output is valid Nagios format: $(echo "$PLUGIN_OUTPUT" | cut -c1-70)" \
    || fail "Plugin output does not match Nagios format."

  (( PLUGIN_EXIT <= 3 )) \
    && pass "Plugin exit code ${PLUGIN_EXIT} is valid (0-3)." \
    || fail "Plugin returned invalid exit code ${PLUGIN_EXIT}."
fi

if [[ -n "$ICINGA_API_PASS" ]]; then
  ICINGA_STATUS=$(curl -so /dev/null -w "%{http_code}" --max-time "$HTTP_TIMEOUT" \
    -k -u "${ICINGA_API_USER}:${ICINGA_API_PASS}" \
    "https://${ICINGA_HOST}/v1/status" 2>/dev/null || echo "0")
  [[ "$ICINGA_STATUS" == "200" ]] \
    && pass "Icinga 2 REST API reachable (HTTP 200)." \
    || fail "Icinga 2 API returned HTTP ${ICINGA_STATUS}."
else
  warn_check "ICINGA_API_PASS not set. Skipping Icinga API test."
fi

# ===========================================================================
# LAYER 6: ALERTMANAGER NOTIFICATION ROUTING
# ===========================================================================
section "Layer 6: Alertmanager Notification Routing"

AMGR_STATUS=$(curl -so /dev/null -w "%{http_code}" --max-time "$HTTP_TIMEOUT" \
  "http://${ALERTMANAGER_HOST}/api/v2/status" 2>/dev/null || echo "0")

[[ "$AMGR_STATUS" == "200" ]] \
  && pass "Alertmanager reachable (HTTP 200)." \
  || fail "Alertmanager unreachable (HTTP ${AMGR_STATUS})."

if [[ "$AMGR_STATUS" == "200" ]]; then
  CONFIG_JSON=$(curl -sf --max-time "$HTTP_TIMEOUT" \
    "http://${ALERTMANAGER_HOST}/api/v2/status" 2>/dev/null || echo "{}")
  echo "$CONFIG_JSON" | grep -qi "icinga\|argocd" \
    && pass "Alertmanager config references ArgoCD/Icinga routing." \
    || warn_check "ArgoCD routing may be missing. Run 06_icinga_config_and_notifications.sh"
fi

# ===========================================================================
# FINAL SUMMARY
# ===========================================================================
echo ""
echo "╔══════════════════════════════════════════════════════╗"
echo "║               VALIDATION SUMMARY                    ║"
printf "║  Completed: %-39s ║\n" "$(date '+%Y-%m-%d %H:%M:%S')"
printf "║  ${GREEN}PASS: %-3s${RESET}   ${YELLOW}WARN: %-3s${RESET}   ${RED}FAIL: %-3s${RESET}                   ║\n" \
  "$PASS_COUNT" "$WARN_COUNT" "$FAIL_COUNT"
echo "╚══════════════════════════════════════════════════════╝"
echo ""

if (( FAIL_COUNT > 0 )); then
  echo "  Pipeline has ${FAIL_COUNT} failing check(s). Review FAIL items above."
  exit "$FAIL_COUNT"    # Exit code = failure count for CI gating
elif (( WARN_COUNT > 0 )); then
  echo "  Pipeline operational with ${WARN_COUNT} warning(s)."
  exit 0
else
  echo "  All checks passed. ArgoCD monitoring pipeline is fully operational."
  exit 0
fi
```
