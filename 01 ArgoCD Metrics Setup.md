---
tags: [hobby-lobby, argocd, monitoring, kubernetes, script]
script: 01_argocd_metrics_setup.sh
step: "1 of 7"
description: Verify and enable ArgoCD metrics endpoints for Prometheus scraping
depends_on: kubectl, kubeconfig pointed at target cluster
next: "[[02 Prometheus Scrape Config]]"
---

# 01 — ArgoCD Metrics Setup

Verifies and enables the three ArgoCD metrics Services so Prometheus has stable endpoints to scrape. Creates any missing Services and runs a live port-forward connectivity test against `/metrics`.

## Components

| Service | Port | Metrics |
|---|---|---|
| `argocd-metrics` | 8082 | Sync status, app info, reconcile counts |
| `argocd-server-metrics` | 8083 | gRPC latency, API request counts |
| `argocd-repo-server-metrics` | 8084 | Manifest generation, git fetch duration |

## Usage

```bash
# Default run (namespace: argocd)
bash 01_argocd_metrics_setup.sh

# Custom namespace
bash 01_argocd_metrics_setup.sh --namespace my-argocd

# Preview only — no changes applied
bash 01_argocd_metrics_setup.sh --dry-run
```

## Script

```bash
#!/usr/bin/env bash
# =============================================================================
# Script Name  : 01_argocd_metrics_setup.sh
# Description  : Verifies and enables ArgoCD metrics endpoints so Prometheus
#                can scrape argocd_app_info, argocd_app_sync_status, and
#                related metrics.  Also patches the argocd-metrics Service
#                if it does not yet expose port 8082 (the default metrics port).
# Author       : Sangyeon Lee  –  Hobby Lobby IT Infrastructure
# Environment  : Kubernetes cluster with ArgoCD installed in namespace "argocd"
# Usage        : bash 01_argocd_metrics_setup.sh [--namespace <ns>] [--dry-run]
# Dependencies : kubectl (must be on PATH and kubeconfig must point to target cluster)
# =============================================================================

set -euo pipefail                      # Exit on error, unset var, or pipe failure

# ---------------------------------------------------------------------------
# CONFIGURABLE VARIABLES  –  override with CLI flags or env vars if needed
# ---------------------------------------------------------------------------
ARGOCD_NAMESPACE="${ARGOCD_NAMESPACE:-argocd}"   # Kubernetes namespace where ArgoCD lives
METRICS_PORT=8082                                # ArgoCD application-controller metrics port
REPO_SERVER_METRICS_PORT=8084                    # ArgoCD repo-server metrics port
SERVER_METRICS_PORT=8083                         # ArgoCD server metrics port
DRY_RUN=false                                    # When true, only print what would change

# ---------------------------------------------------------------------------
# PARSE COMMAND-LINE ARGUMENTS
# ---------------------------------------------------------------------------
while [[ $# -gt 0 ]]; do              # Loop over all positional arguments
  case "$1" in
    --namespace|-n)                   # Accept -n or --namespace flag
      ARGOCD_NAMESPACE="$2"           # Set namespace from the next argument
      shift 2                         # Consume both the flag and its value
      ;;
    --dry-run)                        # Accept --dry-run flag for safe previewing
      DRY_RUN=true                    # Enable dry-run mode; no changes applied
      shift                           # Consume the flag token
      ;;
    *)
      echo "Unknown argument: $1"     # Warn about unrecognised flags
      exit 1                          # Exit with a non-zero status
      ;;
  esac
done

# ---------------------------------------------------------------------------
# HELPER FUNCTIONS
# ---------------------------------------------------------------------------

# log_info: Print an informational message with a timestamp prefix
log_info() {
  echo "[INFO]  $(date '+%Y-%m-%d %H:%M:%S')  $*"   # Print timestamp + message
}

# log_warn: Print a warning message
log_warn() {
  echo "[WARN]  $(date '+%Y-%m-%d %H:%M:%S')  $*" >&2   # Redirect warnings to stderr
}

# log_error: Print an error message and exit
log_error() {
  echo "[ERROR] $(date '+%Y-%m-%d %H:%M:%S')  $*" >&2   # Redirect errors to stderr
  exit 1                                                   # Exit with failure
}

# run_or_dry: Execute a command only when DRY_RUN is false
run_or_dry() {
  if [[ "$DRY_RUN" == "true" ]]; then    # If dry-run mode is enabled
    echo "[DRY-RUN] Would run: $*"       # Print what would be executed instead
  else
    "$@"                                  # Actually execute the command
  fi
}

# ---------------------------------------------------------------------------
# PRE-FLIGHT CHECKS
# ---------------------------------------------------------------------------

# Verify kubectl is installed and reachable in PATH
if ! command -v kubectl &>/dev/null; then
  log_error "kubectl not found. Install kubectl and ensure it is in PATH."
fi

# Verify we can communicate with the Kubernetes API server
if ! kubectl cluster-info &>/dev/null; then
  log_error "Cannot reach Kubernetes cluster. Check your kubeconfig."
fi

log_info "Connected to Kubernetes cluster: $(kubectl config current-context)"

# Verify the ArgoCD namespace exists before proceeding
if ! kubectl get namespace "$ARGOCD_NAMESPACE" &>/dev/null; then
  log_error "Namespace '$ARGOCD_NAMESPACE' not found. Is ArgoCD installed?"
fi

log_info "ArgoCD namespace '$ARGOCD_NAMESPACE' confirmed."

# ---------------------------------------------------------------------------
# STEP 1: CHECK ARGOCD APPLICATION CONTROLLER METRICS SERVICE
# ---------------------------------------------------------------------------
log_info "Checking argocd-metrics Service (application-controller metrics)…"

# Fetch the argocd-metrics Service; it exposes port 8082 for Prometheus
if kubectl get svc argocd-metrics -n "$ARGOCD_NAMESPACE" &>/dev/null; then
  log_info "argocd-metrics Service already exists."
else
  # The service is missing – create it so Prometheus has a stable endpoint
  log_warn "argocd-metrics Service not found. Creating it…"
  run_or_dry kubectl apply -n "$ARGOCD_NAMESPACE" -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: argocd-metrics                         # Name Prometheus will target
  namespace: ${ARGOCD_NAMESPACE}               # Must match ArgoCD's namespace
  labels:
    app.kubernetes.io/name: argocd-metrics     # Label used by ServiceMonitor selector
spec:
  ports:
    - name: metrics                            # Port name referenced in Prometheus scrape config
      port: ${METRICS_PORT}                    # External port (8082)
      targetPort: ${METRICS_PORT}              # Container port inside application-controller Pod
      protocol: TCP
  selector:
    app.kubernetes.io/name: argocd-application-controller   # Matches the controller Pod label
EOF
  log_info "argocd-metrics Service created on port ${METRICS_PORT}."
fi

# ---------------------------------------------------------------------------
# STEP 2: CHECK ARGOCD SERVER METRICS SERVICE
# ---------------------------------------------------------------------------
log_info "Checking argocd-server-metrics Service…"

if kubectl get svc argocd-server-metrics -n "$ARGOCD_NAMESPACE" &>/dev/null; then
  log_info "argocd-server-metrics Service already exists."
else
  log_warn "argocd-server-metrics Service not found. Creating it…"
  run_or_dry kubectl apply -n "$ARGOCD_NAMESPACE" -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: argocd-server-metrics               # Separate service for the ArgoCD API server
  namespace: ${ARGOCD_NAMESPACE}
  labels:
    app.kubernetes.io/name: argocd-server-metrics
spec:
  ports:
    - name: metrics
      port: ${SERVER_METRICS_PORT}          # Port 8083 for the API server component
      targetPort: ${SERVER_METRICS_PORT}
      protocol: TCP
  selector:
    app.kubernetes.io/name: argocd-server   # Targets the argocd-server Deployment pods
EOF
  log_info "argocd-server-metrics Service created on port ${SERVER_METRICS_PORT}."
fi

# ---------------------------------------------------------------------------
# STEP 3: CHECK ARGOCD REPO SERVER METRICS SERVICE
# ---------------------------------------------------------------------------
log_info "Checking argocd-repo-server metrics Service…"

if kubectl get svc argocd-repo-server -n "$ARGOCD_NAMESPACE" &>/dev/null; then
  log_info "argocd-repo-server Service already exists."
else
  log_warn "argocd-repo-server Service not found. Creating it…"
  run_or_dry kubectl apply -n "$ARGOCD_NAMESPACE" -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: argocd-repo-server-metrics          # Service for the repo-server component
  namespace: ${ARGOCD_NAMESPACE}
  labels:
    app.kubernetes.io/name: argocd-repo-server-metrics
spec:
  ports:
    - name: metrics
      port: ${REPO_SERVER_METRICS_PORT}     # Port 8084 for the repo-server
      targetPort: ${REPO_SERVER_METRICS_PORT}
      protocol: TCP
  selector:
    app.kubernetes.io/name: argocd-repo-server   # Targets the repo-server Deployment pods
EOF
  log_info "argocd-repo-server-metrics Service created on port ${REPO_SERVER_METRICS_PORT}."
fi

# ---------------------------------------------------------------------------
# STEP 4: VERIFY METRICS ARE BEING EXPOSED (LIVE CONNECTIVITY TEST)
# ---------------------------------------------------------------------------
log_info "Running port-forward test to verify /metrics endpoint…"

# Start a background kubectl port-forward to the application-controller metrics port
kubectl port-forward svc/argocd-metrics "$METRICS_PORT:$METRICS_PORT" \
  -n "$ARGOCD_NAMESPACE" &>/dev/null &    # Run in background, suppress output

PF_PID=$!                                 # Capture the background process PID

sleep 3                                   # Wait for the port-forward to become ready

# Use curl to fetch the /metrics page; grep for a key ArgoCD metric name
if curl -sf "http://localhost:${METRICS_PORT}/metrics" \
    | grep -q "argocd_app_info"; then     # argocd_app_info confirms ArgoCD metrics are live
  log_info "SUCCESS: ArgoCD /metrics endpoint is reachable and returning data."
else
  log_warn "Metrics endpoint responded but argocd_app_info not found. Check ArgoCD version."
fi

kill "$PF_PID" 2>/dev/null || true        # Stop the background port-forward cleanly

# ---------------------------------------------------------------------------
# STEP 5: PRINT SUMMARY
# ---------------------------------------------------------------------------
log_info "===== ARGOCD METRICS SETUP SUMMARY ====="
log_info "Namespace          : $ARGOCD_NAMESPACE"
log_info "App-controller port: $METRICS_PORT   → scrape http://<pod>:${METRICS_PORT}/metrics"
log_info "API-server port    : $SERVER_METRICS_PORT   → scrape http://<pod>:${SERVER_METRICS_PORT}/metrics"
log_info "Repo-server port   : $REPO_SERVER_METRICS_PORT   → scrape http://<pod>:${REPO_SERVER_METRICS_PORT}/metrics"
log_info "Next step: run 02_prometheus_scrape_config.sh to wire Prometheus to these endpoints."
```
