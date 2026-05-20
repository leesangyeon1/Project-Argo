---
tags: [hobby-lobby, argocd, monitoring, prometheus, alerting, script]
script: 03_prometheus_alert_rules.sh
step: "3 of 7"
description: Generate and apply Prometheus alerting rules for ArgoCD sync anomalies
depends_on: "[[02 Prometheus Scrape Config]]"
next: "[[04 InfluxDB Remote Write]]"
---

# 03 — Prometheus Alert Rules

Generates and applies five alerting rules plus two recording rules for ArgoCD sync-status monitoring. Rules are deployed as a `PrometheusRule` CRD (Operator mode) or a `ConfigMap` (vanilla Prometheus).

## Alert Rules

| Alert | Severity | Fires When |
|---|---|---|
| `ArgoCDAppOutOfSync` | warning | App not Synced for > 5 min |
| `ArgoCDAppSyncFailed` | critical | Sync operation returned Failed/Error |
| `ArgoCDAppHealthDegraded` | critical | Health is Degraded or Missing for > 5 min |
| `ArgoCDAppUnknownStatus` | warning | Sync status is Unknown for > 5 min |
| `ArgoCDHighReconciliationErrorRate` | warning | K8s API error rate > 10% |

## Recording Rules

| Rule | Description |
|---|---|
| `argocd:apps:out_of_sync_count` | Pre-computed count of non-Synced apps |
| `argocd:apps:unhealthy_count` | Pre-computed count of Degraded/Missing apps |

## Usage

```bash
# Apply as PrometheusRule CRD (Prometheus Operator)
bash 03_prometheus_alert_rules.sh --mode operator

# Apply as ConfigMap (vanilla Prometheus)
bash 03_prometheus_alert_rules.sh --mode configmap

# Custom Prometheus namespace
bash 03_prometheus_alert_rules.sh --namespace monitoring
```

## Script

```bash
#!/usr/bin/env bash
# =============================================================================
# Script Name  : 03_prometheus_alert_rules.sh
# Description  : Generates and applies Prometheus alerting rules that fire
#                when ArgoCD applications are in an unhealthy sync state.
#
#                Rules created:
#                  1. ArgoCDAppOutOfSync         – fires when sync_status != Synced
#                  2. ArgoCDAppSyncFailed        – fires after repeated sync failures
#                  3. ArgoCDAppHealthDegraded    – fires when health is Degraded/Missing
#                  4. ArgoCDAppUnknownStatus     – fires when status is Unknown
#                  5. ArgoCDHighSyncErrorRate    – fires when reconcile errors spike
#
# Author       : Sangyeon Lee  –  Hobby Lobby IT Infrastructure
# Usage        : bash 03_prometheus_alert_rules.sh [--mode operator|configmap] \
#                     [--namespace <ns>]
# Dependencies : kubectl, Prometheus ≥ 2.x with --web.enable-lifecycle
# =============================================================================

set -euo pipefail

PROM_NAMESPACE="${PROM_NAMESPACE:-monitoring}"         # Namespace where Prometheus runs
MODE="${MODE:-operator}"                               # "operator" = PrometheusRule CRD
OUTPUT_DIR="./prometheus"                              # Directory for generated YAML files

while [[ $# -gt 0 ]]; do
  case "$1" in
    --mode)         MODE="$2";            shift 2 ;;   # Override mode
    --namespace|-n) PROM_NAMESPACE="$2";  shift 2 ;;   # Override namespace
    *)              echo "Unknown flag: $1"; exit 1 ;;
  esac
done

log()  { echo "[INFO]  $(date '+%H:%M:%S')  $*"; }
warn() { echo "[WARN]  $(date '+%H:%M:%S')  $*" >&2; }

mkdir -p "$OUTPUT_DIR"    # Ensure destination directory exists

log "Writing Prometheus alerting rules to $OUTPUT_DIR/argocd-alert-rules.yaml…"

cat > "$OUTPUT_DIR/argocd-alert-rules.yaml" <<'YAML'
apiVersion: monitoring.coreos.com/v1       # PrometheusRule is a Prometheus Operator CRD
kind: PrometheusRule                       # Custom resource defining alert/record rules
metadata:
  name: argocd-sync-alerts
  namespace: monitoring
  labels:
    release: prometheus                    # Operator uses this label to auto-load rules
    app: argocd-monitoring
spec:
  groups:

    # ── Group 1: Sync Status Rules ─────────────────────────────────────────
    - name: argocd.sync_status
      interval: 60s                        # How often Prometheus evaluates these expressions
      rules:

        # Rule 1: Application Out of Sync (PRIMARY alert for this epic)
        - alert: ArgoCDAppOutOfSync
          expr: |
            argocd_app_info{sync_status!="Synced"}
          for: 5m                          # Must stay out-of-sync for 5 consecutive minutes
          labels:
            severity: warning
            team: infrastructure
          annotations:
            summary: "ArgoCD app {{ $labels.name }} is {{ $labels.sync_status }}"
            description: |
              Application '{{ $labels.name }}' in project '{{ $labels.project }}'
              has been '{{ $labels.sync_status }}' for more than 5 minutes.
            runbook_url: "https://wiki.hobbylobby.internal/runbooks/argocd-out-of-sync"

        # Rule 2: Sync Operation Failed
        - alert: ArgoCDAppSyncFailed
          expr: |
            increase(argocd_app_sync_total{phase=~"Failed|Error"}[10m]) > 0
          for: 1m                          # Alert after failure persists for 1 minute
          labels:
            severity: critical
            team: infrastructure
          annotations:
            summary: "ArgoCD sync FAILED for app {{ $labels.name }}"
            description: |
              Application '{{ $labels.name }}' experienced {{ $value }} sync failure(s)
              in the last 10 minutes.
            runbook_url: "https://wiki.hobbylobby.internal/runbooks/argocd-sync-failed"

        # Rule 3: Application Health Degraded
        - alert: ArgoCDAppHealthDegraded
          expr: |
            argocd_app_info{health_status=~"Degraded|Missing"}
          for: 5m
          labels:
            severity: critical
            team: infrastructure
          annotations:
            summary: "ArgoCD app {{ $labels.name }} health is {{ $labels.health_status }}"
            description: |
              Application '{{ $labels.name }}' health has been '{{ $labels.health_status }}'
              for over 5 minutes. Investigate resources in '{{ $labels.dest_namespace }}'.

        # Rule 4: Unknown Sync Status
        - alert: ArgoCDAppUnknownStatus
          expr: |
            argocd_app_info{sync_status="Unknown"}   # Filter for Unknown sync state only
          for: 5m
          labels:
            severity: warning
            team: infrastructure
          annotations:
            summary: "ArgoCD app {{ $labels.name }} sync status is Unknown"
            description: |
              Cannot determine sync state for '{{ $labels.name }}'.
              Possible causes: cluster unreachable, credentials expired, or app just created.

    # ── Group 2: Reconciliation Error Rate ────────────────────────────────
    - name: argocd.reconciliation
      interval: 60s
      rules:

        # Rule 5: High K8s API Error Rate
        - alert: ArgoCDHighReconciliationErrorRate
          expr: |
            rate(argocd_app_k8s_request_total{response_code=~"5.."}[5m])
              /
            rate(argocd_app_k8s_request_total[5m])
              > 0.10                       # Alert when > 10% of K8s API calls fail
          for: 5m
          labels:
            severity: warning
            team: infrastructure
          annotations:
            summary: "ArgoCD K8s API error rate above 10%"
            description: |
              ArgoCD K8s API error rate is {{ $value | humanizePercentage }}.
              Check API server health and ArgoCD RBAC/credentials.

    # ── Group 3: Recording Rules (pre-compute expensive queries) ──────────
    - name: argocd.recording_rules
      interval: 60s
      rules:

        # Counts total apps currently NOT synced
        - record: argocd:apps:out_of_sync_count
          expr: |
            count(argocd_app_info{sync_status!="Synced"}) or vector(0)
            # `or vector(0)` returns 0 (not "no data") when all apps are Synced

        # Counts total apps with Degraded or Missing health
        - record: argocd:apps:unhealthy_count
          expr: |
            count(argocd_app_info{health_status=~"Degraded|Missing"}) or vector(0)
YAML

log "Alert rules YAML written."

# ---------------------------------------------------------------------------
# APPLY TO CLUSTER
# ---------------------------------------------------------------------------
if [[ "$MODE" == "operator" ]]; then
  log "Applying PrometheusRule to namespace '$PROM_NAMESPACE'…"
  kubectl apply -f "$OUTPUT_DIR/argocd-alert-rules.yaml" -n "$PROM_NAMESPACE"
  log "PrometheusRule applied. Prometheus Operator will load rules within ~30 seconds."

elif [[ "$MODE" == "configmap" ]]; then
  log "Wrapping rules in ConfigMap for vanilla Prometheus…"
  kubectl create configmap argocd-alert-rules \
    --from-file=argocd-alert-rules.yaml="$OUTPUT_DIR/argocd-alert-rules.yaml" \
    -n "$PROM_NAMESPACE" \
    --dry-run=client -o yaml | kubectl apply -f -    # Idempotent apply via dry-run pipe
  warn "Reload Prometheus: curl -X POST http://<prometheus>:9090/-/reload"
else
  echo "Unknown mode: $MODE (use 'operator' or 'configmap')"; exit 1
fi

# ---------------------------------------------------------------------------
# VERIFY RULES LOADED
# ---------------------------------------------------------------------------
log "Checking Prometheus rules API for new alert rules…"
sleep 10   # Short pause to allow the rule to be ingested

PROM_HOST="${PROM_HOST:-localhost:9090}"

if curl -sf "http://${PROM_HOST}/api/v1/rules" \
    | grep -q "ArgoCDAppOutOfSync"; then   # Presence confirms rules are loaded
  log "SUCCESS: Alert rules are loaded in Prometheus."
else
  warn "Rules may not be loaded yet. Check: http://${PROM_HOST}/rules"
fi

log "Next step: run 04_influxdb_remote_write.sh to persist metrics to InfluxDB."
```
