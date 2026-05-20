---
tags: [hobby-lobby, argocd, monitoring, icinga, alertmanager, notifications, script]
script: 06_icinga_config_and_notifications.sh
step: "6 of 7"
description: Generate all Icinga 2 objects and Alertmanager config for email/SMS notifications
depends_on: "[[05 Icinga Check ArgoCD]]", icinga2, alertmanager
next: "[[07 Validate Full Pipeline]]"
---

# 06 — Icinga Config and Notifications

Generates and deploys the full Icinga 2 configuration layer and the Alertmanager webhook bridge. Covers everything from the `CheckCommand` definition to email and SMS (Twilio) notification objects.

## Components Generated

| File | Type | Purpose |
|---|---|---|
| `argocd-checkcommand.conf` | Icinga 2 | Maps CLI args to the check plugin |
| `argocd-host.conf` | Icinga 2 | Virtual host for the ArgoCD cluster |
| `argocd-service.conf` | Icinga 2 | The sync-status check on a 1-min schedule |
| `argocd-notifications.conf` | Icinga 2 | User, email + SMS NotificationCommand objects |
| `alertmanager.yml` | Alertmanager | Routes Prometheus alerts → Icinga webhook |

## Notification Flow

```
Prometheus alert fires
  → Alertmanager routes to icinga-webhook receiver
    → Icinga 2 REST API (process-check-result)
      → Icinga sends Email (all Warning/Critical)
      → Icinga sends SMS via Twilio (Critical only)
```

## Usage

```bash
bash 06_icinga_config_and_notifications.sh \
  --notify-email ops-team@hobbylobby.com \
  --notify-sms +15551234567

# Preview only — no files copied, no reload
bash 06_icinga_config_and_notifications.sh --dry-run
```

## Script

```bash
#!/usr/bin/env bash
# =============================================================================
# Script Name  : 06_icinga_config_and_notifications.sh
# Description  : Generates and installs all Icinga 2 configuration objects and
#                the Alertmanager webhook bridge for ArgoCD alerts via email/SMS.
#
#                Components:
#                  A) Icinga 2 CheckCommand  – maps CLI flags to the check plugin
#                  B) Icinga 2 Host          – virtual host for ArgoCD cluster
#                  C) Icinga 2 Service       – the sync-status check
#                  D) Icinga 2 Notification  – email + SMS (Twilio)
#                  E) Alertmanager config    – routes Prometheus alerts to Icinga
#
#                Flow: Prometheus → Alertmanager → Icinga webhook → Email/SMS
#
# Author       : Sangyeon Lee  –  Hobby Lobby IT Infrastructure
# Usage        : bash 06_icinga_config_and_notifications.sh \
#                     [--notify-email <email>] [--notify-sms <+E164>] [--dry-run]
# Dependencies : icinga2, alertmanager, postfix or sendmail
# =============================================================================

set -euo pipefail

ICINGA_CONF_DIR="${ICINGA_CONF_DIR:-/etc/icinga2/conf.d}"
NOTIFY_EMAIL="${NOTIFY_EMAIL:-ops-team@hobbylobby.com}"
NOTIFY_SMS="${NOTIFY_SMS:-}"                             # E.164 format e.g. +15551234567
PROM_URL="${PROM_URL:-http://prometheus.monitoring.svc:9090}"
ICINGA_API_HOST="${ICINGA_API_HOST:-localhost}"
ICINGA_API_PORT="${ICINGA_API_PORT:-5665}"
ICINGA_API_USER="${ICINGA_API_USER:-root}"
ICINGA_API_PASS="${ICINGA_API_PASS:-}"
PLUGIN_DIR="${PLUGIN_DIR:-/usr/lib/nagios/plugins}"
OUTPUT_DIR="./icinga"
DRY_RUN="${DRY_RUN:-false}"

while [[ $# -gt 0 ]]; do
  case "$1" in
    --icinga-dir)    ICINGA_CONF_DIR="$2"; shift 2 ;;
    --notify-email)  NOTIFY_EMAIL="$2";    shift 2 ;;
    --notify-sms)    NOTIFY_SMS="$2";      shift 2 ;;
    --dry-run)       DRY_RUN=true;         shift   ;;
    *) echo "Unknown flag: $1"; exit 1 ;;
  esac
done

log()  { echo "[INFO]  $(date '+%H:%M:%S')  $*"; }
warn() { echo "[WARN]  $(date '+%H:%M:%S')  $*" >&2; }
run_or_dry() { [[ "$DRY_RUN" == "true" ]] && echo "[DRY-RUN] $*" || "$@"; }

mkdir -p "$OUTPUT_DIR"

# ---------------------------------------------------------------------------
# PART A: CHECKCOMMAND — maps Icinga macro vars to plugin CLI flags
# ---------------------------------------------------------------------------
log "Generating Icinga 2 CheckCommand object…"

cat > "$OUTPUT_DIR/argocd-checkcommand.conf" <<'EOF'
object CheckCommand "check_argocd_sync" {
  command = [ PluginDir + "/check_argocd_sync.sh" ]   // Path to script 05

  arguments = {
    "--prometheus-url" = {
      value       = "$argocd_prometheus_url$"          // Mapped from service custom var
      required    = true
    }
    "--warn-threshold" = { value = "$argocd_warn_threshold$" }
    "--crit-threshold" = { value = "$argocd_crit_threshold$" }
    "--app-filter"     = { value = "$argocd_app_filter$" }
    "--timeout"        = { value = "$argocd_timeout$" }
  }

  // Default values used if the service does not override them
  vars.argocd_prometheus_url  = "http://prometheus.monitoring.svc:9090"
  vars.argocd_warn_threshold  = 1      // Warn if ≥1 apps are OutOfSync
  vars.argocd_crit_threshold  = 1      // Crit if ≥1 apps have sync failures
  vars.argocd_app_filter      = ".*"   // Monitor all apps by default
  vars.argocd_timeout         = 10     // 10-second HTTP timeout
}
EOF

# ---------------------------------------------------------------------------
# PART B: HOST OBJECT — virtual host for the ArgoCD Kubernetes workload
# ---------------------------------------------------------------------------
log "Generating Icinga 2 Host object…"

cat > "$OUTPUT_DIR/argocd-host.conf" <<EOF
object Host "argocd-cluster" {
  display_name  = "ArgoCD GitOps Controller"  // Shown in Icinga web UI

  // Use dummy check so host is always UP — real monitoring is at Service level
  check_command = "dummy"
  vars.dummy_state = 0                        // Return OK so host stays green
  vars.dummy_text  = "Synthetic host for ArgoCD monitoring"

  address = "${PROM_URL}"                     // Informational only; not pinged

  vars.environment = "production"
  vars.team        = "infrastructure"
  vars.cluster     = "hobbylobby-prod"
}
EOF

# ---------------------------------------------------------------------------
# PART C: SERVICE OBJECT — runs the check plugin every 60 seconds
# ---------------------------------------------------------------------------
log "Generating Icinga 2 Service object…"

cat > "$OUTPUT_DIR/argocd-service.conf" <<EOF
object Service "argocd-sync-status" {
  host_name    = "argocd-cluster"              // Attached to the virtual host above
  display_name = "ArgoCD Application Sync Status"

  check_command = "check_argocd_sync"          // Uses CheckCommand from Part A

  vars.argocd_prometheus_url = "${PROM_URL}"
  vars.argocd_warn_threshold = 1               // 1 OutOfSync app → WARNING
  vars.argocd_crit_threshold = 1               // 1 sync failure  → CRITICAL

  check_interval     = 1m     // Active check every 60 seconds
  retry_interval     = 30s    // Re-check every 30s in soft state before alerting
  max_check_attempts = 3      // 3 consecutive failures → HARD state + notification

  vars.notification_email = "${NOTIFY_EMAIL}"
}
EOF

# ---------------------------------------------------------------------------
# PART D: NOTIFICATION OBJECTS — User, email command, SMS command
# ---------------------------------------------------------------------------
log "Generating Icinga 2 Notification objects…"

cat > "$OUTPUT_DIR/argocd-notifications.conf" <<EOF
// ── On-call user ─────────────────────────────────────────────────────────
object User "argocd-oncall" {
  display_name = "ArgoCD On-Call Engineer"
  email        = "${NOTIFY_EMAIL}"    // Alert recipient email
  pager        = "${NOTIFY_SMS}"      // SMS number (E.164 format)

  states  = [ OK, Warning, Critical, Unknown ]
  types   = [ Problem, Recovery, Acknowledgement ]
}

// ── Email NotificationCommand ─────────────────────────────────────────────
object NotificationCommand "argocd-mail-notification" {
  command = [ SysconfDir + "/icinga2/scripts/mail-service-notification.sh" ]

  env = {
    NOTIFICATIONTYPE       = "\$notification.type\$"     // PROBLEM or RECOVERY
    SERVICEDESC            = "\$service.name\$"
    HOSTALIAS              = "\$host.display_name\$"
    SERVICESTATE           = "\$service.state\$"         // OK/WARNING/CRITICAL/UNKNOWN
    LONGDATETIME           = "\$icinga.long_date_time\$"
    SERVICEOUTPUT          = "\$service.output\$"
    NOTIFICATIONAUTHORNAME = "\$notification.author\$"
    NOTIFICATIONCOMMENT    = "\$notification.comment\$"
    USEREMAIL              = "\$user.email\$"
  }
}

// ── SMS NotificationCommand via Twilio HTTP API ───────────────────────────
object NotificationCommand "argocd-sms-notification" {
  command = [ "/bin/bash", "-c" ]

  arguments = {
    "-c" = {
      value = {{{
        curl -sf -X POST \
          https://api.twilio.com/2010-04-01/Accounts/\$TWILIO_ACCOUNT_SID/Messages.json \
          --data-urlencode "To=\$PAGER" \
          --data-urlencode "From=\$TWILIO_FROM" \
          --data-urlencode "Body=[\$SERVICESTATE] \$HOSTALIAS/\$SERVICEDESC: \$SERVICEOUTPUT" \
          -u "\$TWILIO_ACCOUNT_SID:\$TWILIO_AUTH_TOKEN"
      }}}
      skip_key = true
    }
  }

  env = {
    SERVICESTATE  = "\$service.state\$"
    HOSTALIAS     = "\$host.display_name\$"
    SERVICEDESC   = "\$service.name\$"
    SERVICEOUTPUT = "\$service.output\$"
    PAGER         = "\$user.pager\$"    // SMS number from User object
  }
}

// ── Email notification (Warning + Critical) ───────────────────────────────
object Notification "argocd-sync-email" {
  service_name = "argocd-sync-status"
  host_name    = "argocd-cluster"
  command      = "argocd-mail-notification"
  users        = [ "argocd-oncall" ]
  types        = [ Problem, Recovery ]
  states       = [ Warning, Critical ]
  interval     = 30m    // Re-notify every 30min if unacknowledged
}

// ── SMS notification (Critical only) ─────────────────────────────────────
object Notification "argocd-sync-sms" {
  service_name = "argocd-sync-status"
  host_name    = "argocd-cluster"
  command      = "argocd-sms-notification"
  users        = [ "argocd-oncall" ]
  types        = [ Problem, Recovery ]
  states       = [ Critical ]           // SMS only on Critical — avoids SMS spam for warnings
  interval     = 1h                     // Re-send every 1h if unacknowledged
}
EOF

# ---------------------------------------------------------------------------
# PART E: ALERTMANAGER CONFIG — routes Prometheus → Icinga webhook
# ---------------------------------------------------------------------------
log "Generating Alertmanager configuration…"

cat > "$OUTPUT_DIR/alertmanager.yml" <<EOF
global:
  smtp_smarthost: "localhost:587"
  smtp_from: "alertmanager@hobbylobby.com"
  smtp_require_tls: false               # Set true in production

route:
  receiver: "icinga-webhook"            # Default receiver for all unmatched alerts
  group_by: ["alertname", "cluster"]    # Group to reduce noise
  group_wait:      30s                  # Collect alerts 30s before first notification
  group_interval:  5m                   # Delay before re-sending an already-sent group
  repeat_interval: 4h                   # Re-alert every 4h if unresolved

  routes:
    - match:
        alertname: "ArgoCDAppSyncFailed"    # Critical: fire immediately, no delay
      receiver: "icinga-critical"
      group_wait:     0s
      repeat_interval: 30m

    - match_re:
        alertname: "ArgoCDApp.*"            # All other ArgoCD alerts
      receiver: "icinga-webhook"
      group_wait: 1m                        # 1-min buffer to group flapping apps

receivers:
  - name: "icinga-webhook"
    webhook_configs:
      - url: "http://${ICINGA_API_HOST}:${ICINGA_API_PORT}/v1/actions/process-check-result"
        http_config:
          basic_auth:
            username: "${ICINGA_API_USER}"
            password: "${ICINGA_API_PASS}"
        send_resolved: true               # Notify Icinga when the alert clears
        max_alerts: 20

  - name: "icinga-critical"
    webhook_configs:
      - url: "http://${ICINGA_API_HOST}:${ICINGA_API_PORT}/v1/actions/process-check-result"
        http_config:
          basic_auth:
            username: "${ICINGA_API_USER}"
            password: "${ICINGA_API_PASS}"
        send_resolved: true
    email_configs:
      - to: "${NOTIFY_EMAIL}"             # Backup email if Icinga is unreachable
        subject: "[CRITICAL] ArgoCD Sync Failure: {{ range .Alerts }}{{ .Labels.name }}{{ end }}"
        body: |
          {{ range .Alerts }}
          Alert: {{ .Labels.alertname }}
          App:   {{ .Labels.name }}
          Desc:  {{ .Annotations.description }}
          {{ end }}

inhibit_rules:
  # Suppress Warning when Critical already fires for the same app
  - source_match:
      severity: "critical"
    target_match:
      severity: "warning"
    equal: ["alertname", "name"]          # Match on both alert name and app name
EOF

# ---------------------------------------------------------------------------
# DEPLOY
# ---------------------------------------------------------------------------
log "Deploying Icinga 2 configuration files…"

# Install check plugin
run_or_dry cp "./05_icinga_check_argocd.sh" "${PLUGIN_DIR}/check_argocd_sync.sh"
run_or_dry chmod +x "${PLUGIN_DIR}/check_argocd_sync.sh"

# Copy config files to conf.d
for conf_file in \
    "$OUTPUT_DIR/argocd-checkcommand.conf" \
    "$OUTPUT_DIR/argocd-host.conf" \
    "$OUTPUT_DIR/argocd-service.conf" \
    "$OUTPUT_DIR/argocd-notifications.conf"; do
  run_or_dry cp "$conf_file" "${ICINGA_CONF_DIR}/$(basename "$conf_file")"
  log "Copied $(basename "$conf_file") → ${ICINGA_CONF_DIR}/"
done

# Validate config syntax before reloading
log "Validating Icinga 2 configuration syntax…"
run_or_dry icinga2 daemon --validate -c /etc/icinga2/icinga2.conf \
  && log "Icinga 2 config is valid." \
  || { warn "Validation FAILED. Fix errors before reloading."; exit 1; }

# Reload Icinga (SIGHUP — no service interruption)
log "Reloading Icinga 2…"
run_or_dry systemctl reload icinga2

log "Plugin  : ${PLUGIN_DIR}/check_argocd_sync.sh"
log "Configs : ${ICINGA_CONF_DIR}/argocd-*.conf"
log "Email   : ${NOTIFY_EMAIL}"
[[ -n "$NOTIFY_SMS" ]] && log "SMS     : ${NOTIFY_SMS}"
log "Next step: run 07_validate_full_pipeline.sh for end-to-end smoke test."
```
