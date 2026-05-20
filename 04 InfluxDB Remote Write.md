---
tags: [hobby-lobby, argocd, monitoring, influxdb, prometheus, grafana, script]
script: 04_influxdb_remote_write.sh
step: "4 of 7"
description: Configure Prometheus remote_write to push ArgoCD metrics into InfluxDB and validate ingestion
depends_on: "[[02 Prometheus Scrape Config]]", InfluxDB 1.x or 2.x
next: "[[05 Icinga Check ArgoCD]]"
---

# 04 — InfluxDB Remote Write

Configures `remote_write` in Prometheus to push ArgoCD metrics into InfluxDB. Supports both InfluxDB 1.x (`/api/v1/prom/write`) and 2.x (`/api/v2/write`). Also generates a Grafana datasource provisioning file for visual validation.

## Architecture

```
Prometheus ──(remote_write)──► InfluxDB ──(datasource)──► Grafana
```

Only `argocd_*` metrics are forwarded (write relabeling drops everything else to save storage).

## Usage

```bash
# InfluxDB 2.x
INFLUX_TOKEN=my-token bash 04_influxdb_remote_write.sh \
  --influx-version 2 \
  --influx-host influxdb.monitoring.svc:8086 \
  --influx-org hobbylobby \
  --influx-bucket argocd_metrics

# InfluxDB 1.x
bash 04_influxdb_remote_write.sh \
  --influx-version 1 \
  --influx-host influxdb:8086 \
  --influx-bucket argocd_metrics \
  --influx-password mypassword
```

## Script

```bash
#!/usr/bin/env bash
# =============================================================================
# Script Name  : 04_influxdb_remote_write.sh
# Description  : Configures Prometheus remote_write to push ArgoCD metrics into
#                InfluxDB, then validates ingestion by running a Flux query.
#
#                Architecture (from canvas):
#                  Prometheus ──(remote_write)──► InfluxDB
#                  InfluxDB   ──(data source)───► Grafana (visual validation)
#
#                Supports InfluxDB 1.x (/api/v1/prom/write) and
#                         InfluxDB 2.x (/api/v2/write with token auth)
#
# Author       : Sangyeon Lee  –  Hobby Lobby IT Infrastructure
# Usage        : bash 04_influxdb_remote_write.sh \
#                     --influx-version <1|2> --influx-host <host:port> \
#                     [--influx-org <org>] [--influx-bucket <bucket>] \
#                     [--influx-token <token>]
# Dependencies : kubectl, curl, influx CLI (optional for query validation)
# =============================================================================

set -euo pipefail

# ---------------------------------------------------------------------------
# DEFAULTS
# ---------------------------------------------------------------------------
INFLUX_VERSION="${INFLUX_VERSION:-2}"                   # InfluxDB major version (1 or 2)
INFLUX_HOST="${INFLUX_HOST:-localhost:8086}"             # InfluxDB host and port
INFLUX_ORG="${INFLUX_ORG:-hobbylobby}"                  # InfluxDB 2.x organisation name
INFLUX_BUCKET="${INFLUX_BUCKET:-argocd_metrics}"        # InfluxDB bucket (2.x) or DB (1.x)
INFLUX_TOKEN="${INFLUX_TOKEN:-}"                        # InfluxDB 2.x API token (REQUIRED for 2.x)
INFLUX_USERNAME="${INFLUX_USERNAME:-prometheus}"        # InfluxDB 1.x username
INFLUX_PASSWORD="${INFLUX_PASSWORD:-}"                  # InfluxDB 1.x password
PROM_NAMESPACE="${PROM_NAMESPACE:-monitoring}"
OUTPUT_DIR="./prometheus"                               # Where to write generated config
RETENTION_POLICY="${RETENTION_POLICY:-autogen}"         # InfluxDB 1.x retention policy

while [[ $# -gt 0 ]]; do
  case "$1" in
    --influx-version)  INFLUX_VERSION="$2";   shift 2 ;;
    --influx-host)     INFLUX_HOST="$2";      shift 2 ;;
    --influx-org)      INFLUX_ORG="$2";       shift 2 ;;
    --influx-bucket)   INFLUX_BUCKET="$2";    shift 2 ;;
    --influx-token)    INFLUX_TOKEN="$2";     shift 2 ;;
    --influx-username) INFLUX_USERNAME="$2";  shift 2 ;;
    --influx-password) INFLUX_PASSWORD="$2";  shift 2 ;;
    *) echo "Unknown flag: $1"; exit 1 ;;
  esac
done

log()  { echo "[INFO]  $(date '+%H:%M:%S')  $*"; }
warn() { echo "[WARN]  $(date '+%H:%M:%S')  $*" >&2; }
err()  { echo "[ERR]   $(date '+%H:%M:%S')  $*" >&2; exit 1; }

mkdir -p "$OUTPUT_DIR"

# Enforce required credentials based on InfluxDB version
if [[ "$INFLUX_VERSION" == "2" && -z "$INFLUX_TOKEN" ]]; then
  err "InfluxDB 2.x requires --influx-token."
fi

# ---------------------------------------------------------------------------
# STEP 1: GENERATE REMOTE_WRITE CONFIG SNIPPET
# ---------------------------------------------------------------------------
log "Generating Prometheus remote_write configuration for InfluxDB ${INFLUX_VERSION}.x…"

if [[ "$INFLUX_VERSION" == "2" ]]; then
  cat > "$OUTPUT_DIR/remote_write_influxdb2.yaml" <<EOF
# Merge this block into your prometheus.yml under remote_write:
remote_write:
  - url: "http://${INFLUX_HOST}/api/v2/write?org=${INFLUX_ORG}&bucket=${INFLUX_BUCKET}"
    # ^ InfluxDB 2.x endpoint. 'org' and 'bucket' are query params.
    headers:
      Authorization: "Token ${INFLUX_TOKEN}"   # Bearer token for InfluxDB 2.x auth
    remote_timeout: 30s
    queue_config:
      capacity: 10000          # Samples buffered in memory before blocking
      max_shards: 10           # Max parallel write goroutines to InfluxDB
      min_shards: 1            # Start with 1 shard, scale up as needed
      max_samples_per_send: 5000   # Batch size per HTTP POST
      batch_send_deadline: 5s      # Max wait to fill a batch before flushing
      min_backoff: 30ms        # Initial retry delay on error
      max_backoff: 5s          # Maximum retry delay (exponential back-off ceiling)
    write_relabel_configs:
      # Only forward ArgoCD metrics to InfluxDB — drop everything else
      - source_labels: [__name__]
        regex: "argocd_.*"                           # Match only argocd_* metrics
        action: keep                                 # Keep matching; drop everything else
      # Drop high-cardinality labels that bloat InfluxDB storage
      - regex: "^(pod_template_hash|controller_revision_hash)$"
        action: labeldrop
EOF
  log "InfluxDB 2.x config → $OUTPUT_DIR/remote_write_influxdb2.yaml"

else
  cat > "$OUTPUT_DIR/remote_write_influxdb1.yaml" <<EOF
remote_write:
  - url: "http://${INFLUX_HOST}/api/v1/prom/write?db=${INFLUX_BUCKET}&rp=${RETENTION_POLICY}"
    # ^ 'db' = InfluxDB 1.x database; 'rp' = retention policy
    basic_auth:                          # HTTP Basic Auth for InfluxDB 1.x
      username: "${INFLUX_USERNAME}"
      password: "${INFLUX_PASSWORD}"
    remote_timeout: 30s
    write_relabel_configs:
      - source_labels: [__name__]        # Only send ArgoCD metrics
        regex: "argocd_.*"
        action: keep
EOF
  log "InfluxDB 1.x config → $OUTPUT_DIR/remote_write_influxdb1.yaml"
fi

# ---------------------------------------------------------------------------
# STEP 2: TEST CONNECTIVITY TO INFLUXDB
# ---------------------------------------------------------------------------
log "Testing HTTP connectivity to InfluxDB at http://${INFLUX_HOST}…"

if [[ "$INFLUX_VERSION" == "2" ]]; then
  HTTP_STATUS=$(curl -so /dev/null -w "%{http_code}" "http://${INFLUX_HOST}/health")
  [[ "$HTTP_STATUS" == "200" ]] && log "InfluxDB 2.x healthy (HTTP 200)." \
    || err "InfluxDB health check returned HTTP ${HTTP_STATUS}."
else
  HTTP_STATUS=$(curl -so /dev/null -w "%{http_code}" "http://${INFLUX_HOST}/ping")
  [[ "$HTTP_STATUS" == "204" ]] && log "InfluxDB 1.x reachable (HTTP 204)." \
    || err "InfluxDB ping returned HTTP ${HTTP_STATUS}."
fi

# ---------------------------------------------------------------------------
# STEP 3: VALIDATE DATA INGESTION
# ---------------------------------------------------------------------------
log "Checking if ArgoCD metrics have been ingested into InfluxDB…"

if [[ "$INFLUX_VERSION" == "2" ]]; then
  FLUX_QUERY='from(bucket: "'"${INFLUX_BUCKET}"'")
  |> range(start: -5m)
  |> filter(fn: (r) => r._measurement =~ /argocd_.*/)
  |> count()
  |> yield(name: "count")'

  RESPONSE=$(curl -sf \
    -H "Authorization: Token ${INFLUX_TOKEN}" \
    -H "Content-Type: application/vnd.flux" \
    --data "$FLUX_QUERY" \
    "http://${INFLUX_HOST}/api/v2/query?org=${INFLUX_ORG}" 2>/dev/null || echo "")

  echo "$RESPONSE" | grep -q "_value" \
    && log "SUCCESS: ArgoCD metrics found in InfluxDB bucket '${INFLUX_BUCKET}'." \
    || warn "No ArgoCD metrics yet. Allow one scrape cycle (30s) then re-run."
else
  RESPONSE=$(curl -sf \
    -u "${INFLUX_USERNAME}:${INFLUX_PASSWORD}" \
    "http://${INFLUX_HOST}/query" \
    --data-urlencode "db=${INFLUX_BUCKET}" \
    --data-urlencode "q=SELECT count(*) FROM argocd_app_info WHERE time > now() - 5m" \
    2>/dev/null || echo "")

  echo "$RESPONSE" | grep -q '"results"' \
    && log "SUCCESS: ArgoCD metrics ingested in InfluxDB 1.x." \
    || warn "No results returned. Check remote_write is configured and Prometheus is running."
fi

# ---------------------------------------------------------------------------
# STEP 4: WRITE GRAFANA DATA-SOURCE CONFIG
# ---------------------------------------------------------------------------
log "Generating Grafana datasource provisioning file…"

cat > "$OUTPUT_DIR/grafana-datasource-influxdb.yaml" <<EOF
# Place in /etc/grafana/provisioning/datasources/ or mount as ConfigMap volume
apiVersion: 1
datasources:
  - name: InfluxDB-ArgoCD
    type: influxdb                               # Grafana plugin type for InfluxDB
    access: proxy                                # Grafana queries InfluxDB on behalf of browser
    url: http://${INFLUX_HOST}
    jsonData:
      version: Flux                              # Use Flux query language (InfluxDB 2.x)
      organization: ${INFLUX_ORG}
      defaultBucket: ${INFLUX_BUCKET}
    secureJsonData:
      token: ${INFLUX_TOKEN}                    # Stored in Grafana secure storage
    isDefault: false
    editable: true
EOF

log "Grafana datasource config → $OUTPUT_DIR/grafana-datasource-influxdb.yaml"
log "ACTION: Merge remote_write block into prometheus.yml, then reload Prometheus."
log "Next step: run 05_icinga_check_argocd.sh to set up the Icinga check plugin."
```
