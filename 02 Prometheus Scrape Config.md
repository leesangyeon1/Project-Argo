
tags: [hobby-lobby, argocd, monitoring, prometheus, script]
script: 02_prometheus_scrape_config.sh
step: "2 of 7"
description: Generate and apply Prometheus scrape config targeting all three ArgoCD metrics endpoints
depends_on: "[[01 ArgoCD Metrics Setup]]", Prometheus ≥ 2.x
next: "[[03 Prometheus Alert Rules]]"
---

# 02 — Prometheus Scrape Config

Generates the Prometheus scrape configuration for all three ArgoCD metrics endpoints. Supports two modes: **operator** (ServiceMonitor CRDs for kube-prometheus-stack) and **static** (classic `prometheus.yml` snippet for standalone Prometheus).

## Metrics Collected

| Metric | Description |
|---|---|
| `argocd_app_info` | App metadata — name, project, repo, destination |
| `argocd_app_sync_status` | Current sync state per app |
| `argocd_app_health_status` | Health state (Healthy / Degraded) |
| `argocd_app_reconcile_count` | Number of reconciliation cycles |

## Usage

```bash
# Prometheus Operator (ServiceMonitor CRDs) — default
bash 02_prometheus_scrape_config.sh --mode operator

# Standalone Prometheus (prometheus.yml snippet)
bash 02_prometheus_scrape_config.sh --mode static

# Custom namespaces
bash 02_prometheus_scrape_config.sh --namespace argocd --prom-namespace monitoring
```

## Script

```bash
#!/usr/bin/env bash
# =============================================================================
# Script Name  : 02_prometheus_scrape_config.sh
# Description  : Generates and applies the Prometheus scrape configuration
#                (and optional Prometheus Operator ServiceMonitor) that targets
#                all three ArgoCD metrics endpoints:
#                  • argocd-metrics          (application-controller) port 8082
#                  • argocd-server-metrics   (API server)             port 8083
#                  • argocd-repo-server-metrics (repo server)         port 8084
#
#                Key metrics collected:
#                  argocd_app_info              – app metadata (name, project, repo)
#                  argocd_app_sync_status       – current sync state per app
#                  argocd_app_health_status     – health state (Healthy / Degraded)
#                  argocd_app_reconcile_count   – number of reconciliation cycles
#
# Author       : Sangyeon Lee  –  Hobby Lobby IT Infrastructure
# Usage        : bash 02_prometheus_scrape_config.sh [--mode operator|static] \
#                     [--namespace <argocd-ns>] [--prom-namespace <prom-ns>]
# Dependencies : kubectl, yq (optional for YAML merge), Prometheus ≥ 2.x
# =============================================================================

set -euo pipefail                        # Exit on any error or unset variable

# ---------------------------------------------------------------------------
# DEFAULTS
# ---------------------------------------------------------------------------
ARGOCD_NAMESPACE="${ARGOCD_NAMESPACE:-argocd}"          # Namespace where ArgoCD runs
PROM_NAMESPACE="${PROM_NAMESPACE:-monitoring}"          # Namespace where Prometheus runs
SCRAPE_INTERVAL="30s"                                   # How often Prometheus scrapes ArgoCD
SCRAPE_TIMEOUT="10s"                                    # Max time to wait per scrape
MODE="operator"                                         # "operator" = ServiceMonitor CRD
                                                        # "static"   = prometheus.yml snippet
OUTPUT_DIR="./prometheus"                               # Directory to write output files to

# ---------------------------------------------------------------------------
# PARSE CLI FLAGS
# ---------------------------------------------------------------------------
while [[ $# -gt 0 ]]; do
  case "$1" in
    --mode)
      MODE="$2"                        # Set scrape mode (operator or static)
      shift 2
      ;;
    --namespace|-n)
      ARGOCD_NAMESPACE="$2"            # Override the ArgoCD namespace
      shift 2
      ;;
    --prom-namespace)
      PROM_NAMESPACE="$2"              # Override the Prometheus namespace
      shift 2
      ;;
    *)
      echo "Unknown flag: $1"; exit 1  # Fail fast on typos
      ;;
  esac
done

# ---------------------------------------------------------------------------
# HELPER: Coloured log output
# ---------------------------------------------------------------------------
log()  { echo -e "\033[0;32m[INFO]\033[0m  $*"; }     # Green INFO
warn() { echo -e "\033[0;33m[WARN]\033[0m  $*" >&2; } # Yellow WARN to stderr
err()  { echo -e "\033[0;31m[ERR]\033[0m   $*" >&2; exit 1; }  # Red ERR, exits

mkdir -p "$OUTPUT_DIR"     # Ensure the output directory exists before writing files

# ---------------------------------------------------------------------------
# MODE: OPERATOR  (Prometheus Operator ServiceMonitor CRD)
# Preferred for clusters using kube-prometheus-stack or the Prometheus Operator.
# A ServiceMonitor is a custom resource that tells the Operator which Services
# to scrape so you do NOT need to edit prometheus.yml directly.
# ---------------------------------------------------------------------------
if [[ "$MODE" == "operator" ]]; then
  log "Generating Prometheus Operator ServiceMonitor manifests…"

  # ---- ServiceMonitor for the Application Controller (primary sync metrics) ----
  cat > "$OUTPUT_DIR/servicemonitor-argocd-metrics.yaml" <<EOF
apiVersion: monitoring.coreos.com/v1          # Prometheus Operator CRD API version
kind: ServiceMonitor                          # Custom resource type (not built-in K8s)
metadata:
  name: argocd-metrics                        # Unique name for this ServiceMonitor
  namespace: ${PROM_NAMESPACE}                # Must be in Prometheus' namespace
  labels:
    release: prometheus                       # Label Prometheus Operator uses to discover this SM
spec:
  namespaceSelector:
    matchNames:
      - ${ARGOCD_NAMESPACE}                   # Only look for Services inside the ArgoCD namespace
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-metrics  # Matches the Service created in script 01
  endpoints:
    - port: metrics                           # Port NAME in the Service spec (port 8082)
      interval: ${SCRAPE_INTERVAL}            # How frequently Prometheus hits this endpoint
      scrapeTimeout: ${SCRAPE_TIMEOUT}        # Give up after this long waiting for a response
      path: /metrics                          # HTTP path returning Prometheus text format
      relabelings:
        - sourceLabels: [__meta_kubernetes_pod_name]   # Source: pod name from discovery
          targetLabel: pod                             # Store under the 'pod' label in TSDB
        - sourceLabels: [__meta_kubernetes_namespace]  # Source: namespace from discovery
          targetLabel: namespace                       # Store under 'namespace'
EOF

  # ---- ServiceMonitor for the ArgoCD API Server ----
  cat > "$OUTPUT_DIR/servicemonitor-argocd-server.yaml" <<EOF
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-server-metrics
  namespace: ${PROM_NAMESPACE}
  labels:
    release: prometheus
spec:
  namespaceSelector:
    matchNames:
      - ${ARGOCD_NAMESPACE}
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-server-metrics   # Matches service from script 01
  endpoints:
    - port: metrics                       # Port 8083 (argocd-server metrics)
      interval: ${SCRAPE_INTERVAL}
      scrapeTimeout: ${SCRAPE_TIMEOUT}
      path: /metrics
EOF

  # ---- ServiceMonitor for the Repo Server ----
  cat > "$OUTPUT_DIR/servicemonitor-argocd-repo-server.yaml" <<EOF
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-repo-server-metrics
  namespace: ${PROM_NAMESPACE}
  labels:
    release: prometheus
spec:
  namespaceSelector:
    matchNames:
      - ${ARGOCD_NAMESPACE}
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-repo-server-metrics   # Matches service from script 01
  endpoints:
    - port: metrics                       # Port 8084 (repo-server metrics)
      interval: ${SCRAPE_INTERVAL}
      scrapeTimeout: ${SCRAPE_TIMEOUT}
      path: /metrics
EOF

  # Apply all three ServiceMonitor manifests to the cluster
  log "Applying ServiceMonitor manifests to namespace '${PROM_NAMESPACE}'…"
  kubectl apply -f "$OUTPUT_DIR/servicemonitor-argocd-metrics.yaml"       # App-controller SM
  kubectl apply -f "$OUTPUT_DIR/servicemonitor-argocd-server.yaml"        # API-server SM
  kubectl apply -f "$OUTPUT_DIR/servicemonitor-argocd-repo-server.yaml"   # Repo-server SM
  log "ServiceMonitors applied. Prometheus will begin scraping within one scrape cycle."

# ---------------------------------------------------------------------------
# MODE: STATIC  (Classic prometheus.yml scrape_configs)
# ---------------------------------------------------------------------------
elif [[ "$MODE" == "static" ]]; then
  log "Generating static prometheus.yml scrape_configs snippet…"

  cat > "$OUTPUT_DIR/prometheus_argocd_scrape.yaml" <<'YAMLEOF'
scrape_configs:

  # ── 1. ArgoCD Application Controller ──────────────────────────────────────
  - job_name: "argocd-application-controller"   # Unique job identifier in Prometheus UI
    metrics_path: "/metrics"                    # Standard Prometheus text exposition path
    scrape_interval: 30s                        # Poll every 30 seconds
    scrape_timeout: 10s                         # Abandon scrape if no response after 10s
    static_configs:
      - targets:
          - "argocd-metrics.argocd.svc.cluster.local:8082"   # <svc>.<ns>.svc.cluster.local:<port>
        labels:
          component: "application-controller"   # Extra label added to every scraped metric
          cluster: "hobbylobby-prod"            # Identify which cluster this data comes from

  # ── 2. ArgoCD API Server ──────────────────────────────────────────────────
  - job_name: "argocd-server"
    metrics_path: "/metrics"
    scrape_interval: 30s
    scrape_timeout: 10s
    static_configs:
      - targets:
          - "argocd-server-metrics.argocd.svc.cluster.local:8083"
        labels:
          component: "api-server"
          cluster: "hobbylobby-prod"

  # ── 3. ArgoCD Repo Server ─────────────────────────────────────────────────
  - job_name: "argocd-repo-server"
    metrics_path: "/metrics"
    scrape_interval: 30s
    scrape_timeout: 10s
    static_configs:
      - targets:
          - "argocd-repo-server-metrics.argocd.svc.cluster.local:8084"
        labels:
          component: "repo-server"
          cluster: "hobbylobby-prod"
YAMLEOF

  log "Static scrape config written to: $OUTPUT_DIR/prometheus_argocd_scrape.yaml"
  warn "ACTION REQUIRED: Manually merge the scrape_configs block into your prometheus.yml"
  warn "Then reload Prometheus: curl -X POST http://<prometheus-host>:9090/-/reload"

else
  err "Unknown --mode '$MODE'. Use 'operator' or 'static'."
fi

# ---------------------------------------------------------------------------
# VERIFICATION: Confirm Prometheus can see the new targets
# ---------------------------------------------------------------------------
log "Waiting 15 seconds then checking Prometheus targets API…"
sleep 15   # Give Prometheus time to reload and attempt the first scrape

PROM_HOST="${PROM_HOST:-localhost:9090}"   # Allow override via env variable

if curl -sf "http://${PROM_HOST}/api/v1/targets" \
    | grep -q "argocd"; then              # Look for any target labelled with "argocd"
  log "SUCCESS: Prometheus is aware of ArgoCD scrape targets."
else
  warn "ArgoCD targets not yet visible in Prometheus. This may be normal on first run."
  warn "Check: http://${PROM_HOST}/targets  to see target state."
fi

log "Output files written to: $OUTPUT_DIR"
log "Next step: run 03_prometheus_alert_rules.sh to add alerting rules."
```
