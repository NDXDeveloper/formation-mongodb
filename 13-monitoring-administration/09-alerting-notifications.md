🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.9 Alerting et Notifications

## Introduction

L'**alerting** constitue la première ligne de défense dans le maintien de la disponibilité et des performances des systèmes MongoDB en production. Un système d'alerting efficace permet aux SRE de détecter et résoudre les problèmes **avant** qu'ils n'impactent les utilisateurs finaux. Cependant, un alerting mal configuré peut être pire que pas d'alerting du tout : il génère de la fatigue d'alerte, masque les problèmes réels et érode la confiance dans le système de monitoring.

### Philosophie de l'Alerting Moderne

```
┌────────────────────────────────────────────────────────────────┐
│              Principes de l'Alerting Efficace                  │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  1. Alert on Symptoms, Not Causes                              │
│     ❌ "Disk > 80%" → ✓ "Write latency > 100ms"                │
│                                                                │
│  2. Alert on User Impact                                       │
│     ❌ "CPU > 90%" → ✓ "Request error rate > 1%"               │
│                                                                │
│  3. Make Alerts Actionable                                     │
│     ❌ "Something is wrong" → ✓ "Action: Check index X"        │
│                                                                │
│  4. Reduce Alert Fatigue                                       │
│     • Suppress during maintenance                              │
│     • Group related alerts                                     │
│     • Inhibit redundant alerts                                 │
│                                                                │
│  5. Different Severity Levels                                  │
│     • P0: Page immediately (user-facing impact)                │
│     • P1: Investigate within 30 min                            │
│     • P2: During business hours                                │
│     • P3: Weekly review                                        │
│                                                                │
│  6. Alert Hygiene                                              │
│     • Every alert must have a runbook                          │
│     • Review and prune unused alerts quarterly                 │
│     • Track MTTA (Mean Time To Acknowledge)                    │
│     • Track MTTR (Mean Time To Resolution)                     │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Architecture d'Alerting MongoDB

```
┌─────────────────────────────────────────────────────────────────┐
│                   Alert Flow Architecture                       │
└─────────────────────────────────────────────────────────────────┘

┌──────────────────┐
│ MongoDB Cluster  │
│ • Metrics        │
│ • Logs           │
│ • Events         │
└────────┬─────────┘
         │
         ▼
┌────────────────────────────────────────────────────────┐
│              Monitoring Systems                        │
│  ┌──────────────┐   ┌──────────────┐   ┌─────────────┐ │
│  │  Prometheus  │   │   Ops Mgr    │   │   Custom    │ │
│  │  + Exporter  │   │   Alerting   │   │   Scripts   │ │
│  └──────┬───────┘   └──────┬───────┘   └──────┬──────┘ │
└─────────┼──────────────────┼──────────────────┼────────┘
          │                  │                  │
          └──────────────────┼──────────────────┘
                             │
                             ▼
                  ┌─────────────────────┐
                  │  Alert Routing      │
                  │  • Alertmanager     │
                  │  • Grafana          │
                  │  • Custom Router    │
                  └─────────┬───────────┘
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
          ▼                 ▼                 ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  PagerDuty   │  │    Slack     │  │    Email     │
│  (Critical)  │  │  (Warning)   │  │    (Info)    │
└──────────────┘  └──────────────┘  └──────────────┘
          │                 │                 │
          └─────────────────┼─────────────────┘
                            │
                            ▼
                     ┌──────────────┐
                     │   On-Call    │
                     │   Engineer   │
                     └──────────────┘
```

---

## Stratégie d'Alerting par Niveau

### Pyramide des Alertes

```
                    ┌─────────────┐
                    │     P0      │
                    │  CRITICAL   │ <- Page immediately
                    │  ~5 alerts  │    User impact NOW
                    └─────────────┘
                  ┌─────────────────┐
                  │       P1        │
                  │    WARNING      │ <- Investigate 30min
                  │   ~15 alerts    │    Potential impact
                  └─────────────────┘
              ┌───────────────────────┐
              │         P2            │
              │      NOTICE           │ <- Business hours
              │     ~30 alerts        │    Degradation
              └───────────────────────┘
          ┌───────────────────────────────┐
          │            P3                 │
          │        INFORMATION            │ <- Weekly review
          │        ~50 alerts             │    Trends/capacity
          └───────────────────────────────┘
```

### Définition des Niveaux de Sévérité

#### P0 - Critical (Page Immédiat)

**Critères** :
- Impact utilisateur direct et immédiat
- Perte de données en cours ou imminente
- Service indisponible
- SLA breached

**Exemples MongoDB** :
```yaml
Critical Alerts:
  - Primary down (no automatic failover)
  - Replica set majority down
  - Replication stopped (oplog full)
  - Backup failed critical window
  - Data corruption detected
  - Write errors > 5% of operations
  - Query error rate > 1%

Response Time: < 5 minutes
Notification: Page + Phone + Slack
Escalation: If no ack in 5 min → escalate to senior
```

#### P1 - Warning (Investigation Rapide)

**Critères** :
- Impact utilisateur possible à court terme
- Dégradation de performance significative
- Capacité critique atteinte
- Risque de P0 imminent

**Exemples MongoDB** :
```yaml
Warning Alerts:
  - Replication lag > 60 seconds
  - Secondary down (still have quorum)
  - Connection pool > 85% capacity
  - Disk space < 15%
  - Cache dirty > 30%
  - Oplog window < 6 hours
  - Slow query rate spike (> 3σ)

Response Time: < 30 minutes
Notification: Slack + Email
Escalation: If no ack in 30 min → page
```

#### P2 - Notice (Heures Ouvrables)

**Critères** :
- Dégradation mineure
- Trend négatif
- Besoin d'investigation non urgent
- Maintenance préventive requise

**Exemples MongoDB** :
```yaml
Notice Alerts:
  - Replication lag > 30 seconds
  - Connection pool > 70% capacity
  - Disk space < 25%
  - Index suggestion available
  - Memory usage trending up
  - Certificate expiry < 30 days

Response Time: < 4 hours (business hours)
Notification: Email + Ticket
Escalation: None
```

#### P3 - Information (Revue Hebdomadaire)

**Critères** :
- Information contextuelle
- Métriques de tendance
- Planification de capacité
- Maintenance proactive

**Exemples MongoDB** :
```yaml
Info Alerts:
  - Disk growth rate analysis
  - Collection size threshold
  - New version available
  - Unused indexes detected
  - Schema changes detected

Response Time: Next planning session
Notification: Dashboard + Report
Escalation: None
```

---

## Configuration Alertmanager (Prometheus)

### Architecture Alertmanager

```
┌────────────────────────────────────────────────────────────┐
│                    Alertmanager Flow                       │
└────────────────────────────────────────────────────────────┘

Prometheus Rules → Alertmanager → Routes → Receivers → Channels
                        ↓
                   [Grouping]
                   [Inhibition]
                   [Silencing]
```

### Configuration Complète

```yaml
# alertmanager.yml
global:
  # Temps avant de considérer une alerte comme résolue
  resolve_timeout: 5m

  # Slack configuration globale
  slack_api_url: 'https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXX'

  # PagerDuty configuration
  pagerduty_url: 'https://events.pagerduty.com/v2/enqueue'

# Templates pour messages personnalisés
templates:
  - '/etc/alertmanager/templates/*.tmpl'

# Routing tree
route:
  # Labels de grouping par défaut
  group_by: ['alertname', 'cluster', 'service']

  # Temps d'attente avant envoi (permet grouping)
  group_wait: 10s

  # Intervalle entre envois pour même groupe
  group_interval: 10s

  # Intervalle de répétition si non résolu
  repeat_interval: 4h

  # Receiver par défaut
  receiver: 'team-mongodb-default'

  # Routes spécifiques
  routes:
    # Critical alerts → PagerDuty
    - match:
        severity: critical
      receiver: 'pagerduty-critical'
      group_wait: 5s
      repeat_interval: 5m
      continue: true  # Continue vers autres routes

    # Warning alerts → Slack
    - match:
        severity: warning
      receiver: 'slack-warnings'
      group_interval: 5m
      repeat_interval: 1h

    # Alerts spécifiques backup
    - match:
        category: backup
      receiver: 'backup-team'
      group_by: ['alertname', 'cluster']

    # Alerts par environnement
    - match:
        environment: production
      receiver: 'prod-oncall'
      routes:
        - match:
            severity: critical
          receiver: 'pagerduty-critical'

    - match:
        environment: staging
      receiver: 'staging-team'

    # Alerts hors heures ouvrables (routing temporel)
    - match:
        severity: warning
      receiver: 'slack-warnings'
      # Active_time_intervals défini ci-dessous
      active_time_intervals:
        - business_hours

# Inhibition rules (supprimer alertes redondantes)
inhibit_rules:
  # Si instance down, ne pas alerter sur ses métriques
  - source_match:
      alertname: 'MongoDBInstanceDown'
    target_match_re:
      alertname: '.*'
    equal: ['instance']

  # Si primary down, ne pas alerter sur replication lag
  - source_match:
      alertname: 'MongoDBPrimaryDown'
    target_match:
      alertname: 'MongoDBReplicationLag'
    equal: ['cluster']

  # Si cluster down, supprimer toutes alertes warning
  - source_match:
      severity: 'critical'
      alertname: 'MongoDBClusterDown'
    target_match:
      severity: 'warning'
    equal: ['cluster']

# Receivers (destinations)
receivers:
  # Default receiver
  - name: 'team-mongodb-default'
    email_configs:
      - to: 'mongodb-team@example.com'
        headers:
          Subject: '[MongoDB Alert] {{ .GroupLabels.alertname }}'

  # PagerDuty pour critical
  - name: 'pagerduty-critical'
    pagerduty_configs:
      - service_key: 'YOUR_PAGERDUTY_SERVICE_KEY'
        description: '{{ .GroupLabels.alertname }}: {{ .Annotations.summary }}'
        severity: '{{ .Labels.severity }}'
        details:
          firing: '{{ template "pagerduty.default.instances" .Alerts.Firing }}'
          resolved: '{{ template "pagerduty.default.instances" .Alerts.Resolved }}'
          num_firing: '{{ .Alerts.Firing | len }}'
          num_resolved: '{{ .Alerts.Resolved | len }}'
        client: 'Prometheus'
        client_url: '{{ template "pagerduty.default.clientURL" . }}'

  # Slack pour warnings
  - name: 'slack-warnings'
    slack_configs:
      - channel: '#mongodb-alerts'
        username: 'Alertmanager'
        color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
        title: '[{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}'
        title_link: '{{ template "slack.default.titlelink" . }}'
        text: |
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Details:* {{ .Annotations.description }}
          *Runbook:* {{ .Annotations.runbook_url }}
          *Cluster:* {{ .Labels.cluster }}
          *Severity:* {{ .Labels.severity }}
          {{ end }}
        actions:
          - type: button
            text: 'View in Grafana'
            url: '{{ (index .Alerts 0).Annotations.grafana_url }}'
          - type: button
            text: 'Silence 1h'
            url: '{{ template "slack.default.silenceURL" . }}'

  # Backup team
  - name: 'backup-team'
    email_configs:
      - to: 'backup-team@example.com'
    slack_configs:
      - channel: '#mongodb-backups'
        text: 'Backup alert: {{ .Annotations.description }}'

  # Production on-call
  - name: 'prod-oncall'
    opsgenie_configs:
      - api_key: 'YOUR_OPSGENIE_KEY'
        message: '{{ .GroupLabels.alertname }}'
        description: '{{ .Annotations.description }}'
        priority: '{{ if eq .Labels.severity "critical" }}P1{{ else }}P2{{ end }}'

# Time intervals pour routing temporel
time_intervals:
  - name: business_hours
    time_intervals:
      - times:
          - start_time: '09:00'
            end_time: '18:00'
        weekdays: ['monday:friday']
```

### Templates de Messages

```go
# /etc/alertmanager/templates/mongodb.tmpl

{{ define "mongodb.title" }}
[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .GroupLabels.alertname }}
{{ end }}

{{ define "mongodb.text" }}
{{ range .Alerts }}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
*Alert:* {{ .Annotations.summary }}
*Severity:* {{ .Labels.severity | toUpper }}
*Cluster:* {{ .Labels.cluster }}
*Environment:* {{ .Labels.environment }}

*Description:*
{{ .Annotations.description }}

*Action Required:*
{{ .Annotations.action }}

*Runbook:* {{ .Annotations.runbook_url }}
*Dashboard:* {{ .Annotations.grafana_url }}

*Started:* {{ .StartsAt.Format "2006-01-02 15:04:05 MST" }}
{{ if .EndsAt }}*Ended:* {{ .EndsAt.Format "2006-01-02 15:04:05 MST" }}{{ end }}
{{ end }}
{{ end }}

{{ define "mongodb.pagerduty.description" }}
{{ .Annotations.summary }}
Cluster: {{ .Labels.cluster }} | Env: {{ .Labels.environment }}
{{ end }}
```

---

## Règles d'Alerte Prometheus

### Alerts MongoDB Critiques

```yaml
# mongodb_alerts_critical.yml
groups:
  - name: mongodb_critical
    interval: 30s
    rules:
      # Instance complètement down
      - alert: MongoDBInstanceDown
        expr: up{job=~"mongodb.*"} == 0
        for: 2m
        labels:
          severity: critical
          category: availability
        annotations:
          summary: "MongoDB instance {{ $labels.instance }} is down"
          description: |
            MongoDB instance {{ $labels.instance }} in cluster {{ $labels.cluster }}
            has been down for more than 2 minutes.

            Current status: DOWN
            Last seen: {{ $value | humanizeDuration }} ago
          action: |
            1. Check if host is reachable
            2. Verify mongod process: systemctl status mongod
            3. Check logs: tail -100 /var/log/mongodb/mongod.log
            4. If primary, verify automatic failover occurred
          runbook_url: "https://runbooks.example.com/mongodb/instance-down"
          grafana_url: "https://grafana.example.com/d/mongodb/cluster={{ $labels.cluster }}"

      # Primary down sans automatic failover
      - alert: MongoDBPrimaryDown
        expr: |
          (
            mongodb_rs_members_state{state="PRIMARY"} == 0
            and
            absent(mongodb_rs_members_state{state="PRIMARY"} == 1)
          )
        for: 1m
        labels:
          severity: critical
          category: availability
        annotations:
          summary: "MongoDB primary down in {{ $labels.cluster }}"
          description: |
            No primary available in replica set {{ $labels.cluster }}.
            Automatic failover may have failed.
            Write operations are blocked.
          action: |
            IMMEDIATE ACTION REQUIRED:
            1. Check replica set status: rs.status()
            2. Verify network connectivity between members
            3. Check if quorum is available
            4. Manual intervention may be needed to trigger election
          runbook_url: "https://runbooks.example.com/mongodb/primary-down"

      # Replication complètement arrêtée
      - alert: MongoDBReplicationStopped
        expr: |
          (
            mongodb_rs_members_optimeDate{state="SECONDARY"}
            - on(cluster) group_left
            mongodb_rs_members_optimeDate{state="PRIMARY"}
          ) > 3600
        for: 5m
        labels:
          severity: critical
          category: replication
        annotations:
          summary: "Replication stopped on {{ $labels.member_name }}"
          description: |
            Secondary {{ $labels.member_name }} has not replicated for {{ $value }} seconds.
            Replication lag exceeds 1 hour, indicating replication has likely stopped.
          action: |
            1. Check secondary logs for errors
            2. Verify oplog size: db.getReplicationInfo()
            3. Check if secondary needs initial sync
            4. Monitor network between primary and secondary

      # Write errors élevés
      - alert: MongoDBHighWriteErrors
        expr: |
          (
            rate(mongodb_ss_opcounters{type="insert"}[5m]) > 0
            and
            (
              rate(mongodb_ss_opcounters_failed{type="insert"}[5m])
              / rate(mongodb_ss_opcounters{type="insert"}[5m])
            ) > 0.05
          )
        for: 2m
        labels:
          severity: critical
          category: operations
        annotations:
          summary: "High write error rate on {{ $labels.instance }}"
          description: |
            Write error rate is {{ $value | humanizePercentage }} (threshold: 5%).
            This indicates serious issues with write operations.
          action: |
            1. Check for disk space issues
            2. Verify authentication/authorization
            3. Check for schema validation failures
            4. Review application logs for error details

      # Oplog plein (risque de resync)
      - alert: MongoDBOplogFull
        expr: |
          (
            mongodb_rs_oplog_usedSizeMB
            / mongodb_rs_oplog_logSizeMB
          ) > 0.95
        labels:
          severity: critical
          category: replication
        annotations:
          summary: "Oplog nearly full on {{ $labels.instance }}"
          description: |
            Oplog usage is {{ $value | humanizePercentage }} (threshold: 95%).
            Risk of secondary requiring full resync.
          action: |
            URGENT: Increase oplog size immediately
            1. Stop writes if possible
            2. Increase oplog: replSetResizeOplog command
            3. Monitor replication lag closely

      # Backup critique raté
      - alert: MongoDBBackupFailedCritical
        expr: |
          (
            time() - mongodb_backup_last_success_timestamp
          ) > 86400
        labels:
          severity: critical
          category: backup
        annotations:
          summary: "No successful backup in 24h for {{ $labels.cluster }}"
          description: |
            Last successful backup was {{ $value | humanizeDuration }} ago.
            Exceeds 24h threshold for critical backups.
          action: |
            1. Check backup daemon status
            2. Verify storage space
            3. Review backup logs
            4. Escalate to backup team immediately
```

### Alerts MongoDB Warning

```yaml
# mongodb_alerts_warning.yml
groups:
  - name: mongodb_warning
    interval: 1m
    rules:
      # Replication lag élevé
      - alert: MongoDBReplicationLagHigh
        expr: |
          (
            mongodb_rs_members_optimeDate{state="PRIMARY"}
            - on(cluster) group_right
            mongodb_rs_members_optimeDate{state="SECONDARY"}
          ) > 60
        for: 5m
        labels:
          severity: warning
          category: replication
        annotations:
          summary: "High replication lag on {{ $labels.member_name }}"
          description: |
            Replication lag is {{ $value }} seconds (threshold: 60s).
            Secondary is falling behind primary.
          action: |
            1. Check for slow queries on secondary
            2. Verify network latency
            3. Check I/O performance on secondary
            4. Review read preference settings

      # Cache dirty élevé
      - alert: MongoDBCacheDirtyHigh
        expr: |
          (
            mongodb_ss_wt_cache_tracked_dirty_bytes_in_the_cache
            / mongodb_ss_wt_cache_maximum_bytes_configured
          ) * 100 > 30
        for: 10m
        labels:
          severity: warning
          category: performance
        annotations:
          summary: "High dirty cache on {{ $labels.instance }}"
          description: |
            Dirty cache is {{ $value | printf "%.2f" }}% (threshold: 30%).
            May indicate I/O bottleneck or excessive writes.
          action: |
            1. Check I/O wait: iostat -x 1 10
            2. Review write operations rate
            3. Check checkpoint duration
            4. Consider faster storage

      # Connexions proche limite
      - alert: MongoDBConnectionsNearLimit
        expr: |
          (
            mongodb_ss_connections{conn_type="current"}
            / (
              mongodb_ss_connections{conn_type="current"}
              + mongodb_ss_connections{conn_type="available"}
            )
          ) * 100 > 80
        for: 5m
        labels:
          severity: warning
          category: capacity
        annotations:
          summary: "Connections near limit on {{ $labels.instance }}"
          description: |
            Using {{ $value | printf "%.2f" }}% of available connections (threshold: 80%).
            Risk of connection exhaustion.
          action: |
            1. Check for connection leaks in application
            2. Review connection pooling settings
            3. Identify clients with most connections
            4. Consider increasing maxIncomingConnections

      # Queue depth élevée
      - alert: MongoDBHighQueueDepth
        expr: |
          (
            mongodb_ss_globalLock_currentQueue_readers
            + mongodb_ss_globalLock_currentQueue_writers
          ) > 20
        for: 5m
        labels:
          severity: warning
          category: performance
        annotations:
          summary: "High queue depth on {{ $labels.instance }}"
          description: |
            Queue depth is {{ $value }} (threshold: 20).
            Operations waiting for execution.
          action: |
            1. Identify slow queries: db.currentOp()
            2. Check for lock contention
            3. Review index usage
            4. Consider query optimization

      # Espace disque faible
      - alert: MongoDBLowDiskSpace
        expr: |
          (
            node_filesystem_avail_bytes{mountpoint="/data"}
            / node_filesystem_size_bytes{mountpoint="/data"}
          ) * 100 < 15
        for: 5m
        labels:
          severity: warning
          category: capacity
        annotations:
          summary: "Low disk space on {{ $labels.instance }}"
          description: |
            Only {{ $value | printf "%.2f" }}% disk space remaining (threshold: 15%).
          action: |
            1. Identify large collections: db.stats()
            2. Review and clean old data
            3. Check for unexpected data growth
            4. Plan disk expansion

      # Certificat proche expiration
      - alert: MongoDBCertificateExpiringSoon
        expr: |
          (
            mongodb_ssl_certificate_expiry_seconds
            / 86400
          ) < 30
        labels:
          severity: warning
          category: security
        annotations:
          summary: "TLS certificate expiring soon on {{ $labels.instance }}"
          description: |
            Certificate expires in {{ $value | printf "%.0f" }} days (threshold: 30 days).
          action: |
            1. Prepare new certificate
            2. Plan rolling restart for certificate rotation
            3. Update monitoring after rotation
```

### Recording Rules pour Alerting

```yaml
# mongodb_recording_rules.yml
groups:
  - name: mongodb_recording
    interval: 30s
    rules:
      # Replication lag calculé
      - record: mongodb:replication:lag_seconds
        expr: |
          mongodb_rs_members_optimeDate{state="PRIMARY"}
          - on(cluster) group_right
          mongodb_rs_members_optimeDate{state="SECONDARY"}

      # Utilisation connexions pourcentage
      - record: mongodb:connections:usage_percent
        expr: |
          (
            mongodb_ss_connections{conn_type="current"}
            / (
              mongodb_ss_connections{conn_type="current"}
              + mongodb_ss_connections{conn_type="available"}
            )
          ) * 100

      # Cache dirty pourcentage
      - record: mongodb:cache:dirty_percent
        expr: |
          (
            mongodb_ss_wt_cache_tracked_dirty_bytes_in_the_cache
            / mongodb_ss_wt_cache_maximum_bytes_configured
          ) * 100

      # Queue depth total
      - record: mongodb:queue:total
        expr: |
          mongodb_ss_globalLock_currentQueue_readers
          + mongodb_ss_globalLock_currentQueue_writers

      # Taux d'erreur write
      - record: mongodb:operations:write_error_rate
        expr: |
          rate(mongodb_ss_opcounters_failed{type="insert"}[5m])
          / rate(mongodb_ss_opcounters{type="insert"}[5m])
```

---

## Configuration Grafana Alerting

### Alert Grafana sur Dashboard

```json
{
  "alert": {
    "name": "MongoDB Replication Lag Warning",
    "message": "Replication lag exceeds threshold on {{ $labels.member_name }}",
    "executionErrorState": "alerting",
    "for": "5m",
    "frequency": "1m",
    "handler": 1,
    "noDataState": "no_data",
    "conditions": [
      {
        "evaluator": {
          "params": [60],
          "type": "gt"
        },
        "operator": {
          "type": "and"
        },
        "query": {
          "params": ["A", "5m", "now"]
        },
        "reducer": {
          "params": [],
          "type": "avg"
        },
        "type": "query"
      }
    ],
    "notifications": [
      {
        "uid": "slack-notifications"
      },
      {
        "uid": "email-team"
      }
    ]
  },
  "targets": [
    {
      "expr": "mongodb:replication:lag_seconds{cluster=\"production\"}",
      "refId": "A"
    }
  ]
}
```

### Notification Channels

```json
{
  "name": "Slack MongoDB Alerts",
  "type": "slack",
  "isDefault": false,
  "sendReminder": true,
  "disableResolveMessage": false,
  "frequency": "5m",
  "settings": {
    "url": "https://hooks.slack.com/services/XXX",
    "channel": "#mongodb-alerts",
    "username": "Grafana",
    "icon_emoji": ":warning:",
    "mentionChannel": "here",
    "uploadImage": true
  }
}
```

---

## Intégration PagerDuty

### Configuration Avancée

```python
#!/usr/bin/env python3
# pagerduty_integration.py

import requests
import json
from datetime import datetime

class PagerDutyIntegration:
    def __init__(self, integration_key):
        self.integration_key = integration_key
        self.api_url = "https://events.pagerduty.com/v2/enqueue"

    def trigger_alert(self, summary, severity, source, component, details):
        """Trigger a PagerDuty alert"""

        event = {
            "routing_key": self.integration_key,
            "event_action": "trigger",
            "dedup_key": f"{source}_{component}_{datetime.utcnow().strftime('%Y%m%d')}",
            "payload": {
                "summary": summary,
                "severity": severity,
                "source": source,
                "component": component,
                "timestamp": datetime.utcnow().isoformat() + "Z",
                "custom_details": details
            },
            "links": [
                {
                    "href": details.get("dashboard_url"),
                    "text": "View Dashboard"
                },
                {
                    "href": details.get("runbook_url"),
                    "text": "Runbook"
                }
            ]
        }

        response = requests.post(
            self.api_url,
            headers={"Content-Type": "application/json"},
            data=json.dumps(event)
        )

        return response.json()

    def resolve_alert(self, dedup_key):
        """Resolve a PagerDuty alert"""

        event = {
            "routing_key": self.integration_key,
            "event_action": "resolve",
            "dedup_key": dedup_key
        }

        response = requests.post(
            self.api_url,
            headers={"Content-Type": "application/json"},
            data=json.dumps(event)
        )

        return response.json()

# Usage example
pd = PagerDutyIntegration("YOUR_INTEGRATION_KEY")

# Trigger critical alert
pd.trigger_alert(
    summary="MongoDB Primary Down - production-rs0",
    severity="critical",
    source="prometheus-alertmanager",
    component="mongodb-production-rs0",
    details={
        "cluster": "production-rs0",
        "environment": "production",
        "instance": "mongo-prod-01:27017",
        "alert_duration": "2m",
        "dashboard_url": "https://grafana.example.com/d/mongodb",
        "runbook_url": "https://runbooks.example.com/mongodb/primary-down"
    }
)
```

---

## Notification Channels

### 1. Slack Integration Avancée

```python
#!/usr/bin/env python3
# slack_notifier.py

import requests
import json

class SlackNotifier:
    def __init__(self, webhook_url):
        self.webhook_url = webhook_url

    def send_alert(self, alert_data):
        """Send formatted alert to Slack"""

        severity = alert_data.get('severity', 'unknown')
        color = {
            'critical': 'danger',
            'warning': 'warning',
            'info': 'good'
        }.get(severity, '#808080')

        message = {
            "attachments": [
                {
                    "color": color,
                    "title": f"[{severity.upper()}] {alert_data['alertname']}",
                    "title_link": alert_data.get('dashboard_url'),
                    "text": alert_data['description'],
                    "fields": [
                        {
                            "title": "Cluster",
                            "value": alert_data.get('cluster'),
                            "short": True
                        },
                        {
                            "title": "Instance",
                            "value": alert_data.get('instance'),
                            "short": True
                        },
                        {
                            "title": "Duration",
                            "value": alert_data.get('duration'),
                            "short": True
                        },
                        {
                            "title": "Environment",
                            "value": alert_data.get('environment'),
                            "short": True
                        }
                    ],
                    "actions": [
                        {
                            "type": "button",
                            "text": "View Dashboard",
                            "url": alert_data.get('dashboard_url')
                        },
                        {
                            "type": "button",
                            "text": "Runbook",
                            "url": alert_data.get('runbook_url'),
                            "style": "primary"
                        },
                        {
                            "type": "button",
                            "text": "Acknowledge",
                            "url": alert_data.get('ack_url'),
                            "style": "danger"
                        }
                    ],
                    "footer": "MongoDB Alerting",
                    "footer_icon": "https://www.mongodb.com/favicon.ico",
                    "ts": alert_data.get('timestamp')
                }
            ]
        }

        response = requests.post(
            self.webhook_url,
            headers={"Content-Type": "application/json"},
            data=json.dumps(message)
        )

        return response.status_code == 200

# Usage
notifier = SlackNotifier("https://hooks.slack.com/services/XXX")
notifier.send_alert({
    'alertname': 'MongoDBReplicationLag',
    'severity': 'warning',
    'description': 'Replication lag is 75 seconds on production-rs0-sec1',
    'cluster': 'production-rs0',
    'instance': 'mongo-prod-02:27017',
    'duration': '5m',
    'environment': 'production',
    'dashboard_url': 'https://grafana.example.com/d/mongodb',
    'runbook_url': 'https://runbooks.example.com/mongodb/replication-lag',
    'ack_url': 'https://alertmanager.example.com/acknowledge',
    'timestamp': 1702220445
})
```

### 2. Email avec Template HTML

```python
#!/usr/bin/env python3
# email_notifier.py

import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from jinja2 import Template

class EmailNotifier:
    def __init__(self, smtp_server, smtp_port, username, password):
        self.smtp_server = smtp_server
        self.smtp_port = smtp_port
        self.username = username
        self.password = password

    def send_alert(self, to_addresses, alert_data):
        """Send formatted alert email"""

        # HTML template
        html_template = """
        <!DOCTYPE html>
        <html>
        <head>
            <style>
                body { font-family: Arial, sans-serif; }
                .header { background-color: {{ color }}; color: white; padding: 20px; }
                .content { padding: 20px; }
                .details { background-color: #f5f5f5; padding: 15px; margin: 10px 0; }
                .button {
                    background-color: #007bff;
                    color: white;
                    padding: 10px 20px;
                    text-decoration: none;
                    border-radius: 5px;
                    display: inline-block;
                    margin: 5px;
                }
                .metric { margin: 10px 0; }
                .label { font-weight: bold; }
            </style>
        </head>
        <body>
            <div class="header">
                <h2>[{{ severity | upper }}] {{ alertname }}</h2>
                <p>{{ timestamp }}</p>
            </div>
            <div class="content">
                <p>{{ description }}</p>

                <div class="details">
                    <h3>Alert Details</h3>
                    <div class="metric">
                        <span class="label">Cluster:</span> {{ cluster }}
                    </div>
                    <div class="metric">
                        <span class="label">Instance:</span> {{ instance }}
                    </div>
                    <div class="metric">
                        <span class="label">Environment:</span> {{ environment }}
                    </div>
                    <div class="metric">
                        <span class="label">Duration:</span> {{ duration }}
                    </div>
                </div>

                <h3>Required Action</h3>
                <p>{{ action }}</p>

                <div style="margin-top: 20px;">
                    <a href="{{ dashboard_url }}" class="button">View Dashboard</a>
                    <a href="{{ runbook_url }}" class="button">View Runbook</a>
                </div>
            </div>
        </body>
        </html>
        """

        color_map = {
            'critical': '#d9534f',
            'warning': '#f0ad4e',
            'info': '#5bc0de'
        }

        template = Template(html_template)
        html_content = template.render(
            color=color_map.get(alert_data.get('severity'), '#808080'),
            **alert_data
        )

        # Create message
        msg = MIMEMultipart('alternative')
        msg['Subject'] = f"[{alert_data['severity'].upper()}] {alert_data['alertname']}"
        msg['From'] = self.username
        msg['To'] = ', '.join(to_addresses)

        html_part = MIMEText(html_content, 'html')
        msg.attach(html_part)

        # Send email
        with smtplib.SMTP(self.smtp_server, self.smtp_port) as server:
            server.starttls()
            server.login(self.username, self.password)
            server.send_message(msg)
```

### 3. Webhook Générique

```python
#!/usr/bin/env python3
# webhook_handler.py

from flask import Flask, request, jsonify
import requests
import logging

app = Flask(__name__)
logging.basicConfig(level=logging.INFO)

@app.route('/webhook/alert', methods=['POST'])
def handle_alert():
    """Generic webhook handler for alerts"""

    data = request.json
    logging.info(f"Received alert: {data.get('alertname')}")

    # Extract alert information
    alert_name = data.get('alertname')
    severity = data.get('severity')
    status = data.get('status')

    # Route based on severity
    if severity == 'critical':
        handle_critical_alert(data)
    elif severity == 'warning':
        handle_warning_alert(data)
    else:
        handle_info_alert(data)

    return jsonify({'status': 'processed'}), 200

def handle_critical_alert(data):
    """Handle critical alerts"""
    # Send to PagerDuty
    requests.post('https://events.pagerduty.com/v2/enqueue', json={
        'routing_key': 'CRITICAL_KEY',
        'event_action': 'trigger',
        'payload': {
            'summary': data.get('summary'),
            'severity': 'critical',
            'source': data.get('instance')
        }
    })

    # Send to Slack
    requests.post('https://hooks.slack.com/services/XXX', json={
        'text': f":rotating_light: CRITICAL: {data.get('summary')}"
    })

    # Create JIRA ticket
    requests.post('https://jira.example.com/rest/api/2/issue', auth=('user', 'pass'), json={
        'fields': {
            'project': {'key': 'OPS'},
            'summary': f"MongoDB Critical: {data.get('alertname')}",
            'description': data.get('description'),
            'issuetype': {'name': 'Incident'},
            'priority': {'name': 'Critical'}
        }
    })

def handle_warning_alert(data):
    """Handle warning alerts"""
    requests.post('https://hooks.slack.com/services/XXX', json={
        'text': f":warning: Warning: {data.get('summary')}"
    })

def handle_info_alert(data):
    """Handle info alerts"""
    logging.info(f"Info alert: {data.get('summary')}")

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

---

## Runbooks et Documentation

### Template de Runbook

```markdown
# Runbook: MongoDB Replication Lag High

## Alert Details

**Alert Name:** MongoDBReplicationLagHigh
**Severity:** Warning
**Category:** Replication
**Threshold:** Lag > 60 seconds for 5 minutes

## Description

This alert fires when a secondary member of a MongoDB replica set falls
behind the primary by more than 60 seconds for at least 5 minutes. This
indicates the secondary is not keeping up with the replication stream.

## Impact

- **Read Concern:** Stale reads if querying the lagging secondary
- **Failover Risk:** If primary fails, lagging secondary may not be
  eligible for election
- **Backup Issues:** Backups from lagging secondary may be inconsistent
- **Oplog Risk:** Extended lag may cause oplog window exhaustion

## Diagnosis

### 1. Verify the Alert

```bash
# Check current replication status
mongo --eval "rs.status()" | grep -A 5 "optimeDate"

# Check lag specifically
mongo --eval "rs.printSlaveReplicationInfo()"
```

Expected output showing lag:
```
source: mongo-prod-02:27017
    syncedTo: Mon Dec 08 2024 15:30:00 GMT+0000 (UTC)
    75 secs (0.02 hrs) behind the primary
```

### 2. Identify Root Cause

#### A. Check for Slow Queries on Secondary

```javascript
// Connect to the lagging secondary
use admin
db.currentOp({
  "active": true,
  "secs_running": { "$gt": 5 }
})
```

#### B. Check Network Latency

```bash
# From secondary to primary
ping -c 10 mongo-primary-host

# Expected: < 10ms
# If > 50ms: Network issue
```

#### C. Check I/O Performance

```bash
# On the secondary server
iostat -x 1 10

# Look for:
# - High %util (> 80%): Disk saturation
# - High await (> 50ms): Slow disk
```

#### D. Check System Resources

```bash
# CPU usage
top -bn1 | grep mongod

# Memory
free -h

# Disk space
df -h /data
```

### 3. Check Historical Metrics

- **Grafana Dashboard:** https://grafana.example.com/d/mongodb-replication
- **Metrics to check:**
  - Replication lag trend (last 24h)
  - Operations rate on secondary
  - Network traffic
  - Disk I/O

## Resolution Steps

### Step 1: Quick Wins

```bash
# 1. Check if secondary is hidden (should not serve reads if lagging)
mongo --eval "rs.config()" | grep hidden

# 2. Temporarily increase write concern timeout (if safe)
# This gives secondary more time to catch up
```

### Step 2: Address Slow Queries

```javascript
// If slow queries found in currentOp
// Kill long-running queries (after verification)
db.killOp(<opid>)

// Or if safe, restart the secondary
// It will catch up on restart
```

### Step 3: Network Issues

```bash
# If network latency high:
# 1. Check for network congestion
# 2. Verify firewall rules
# 3. Consider network path optimization
```

### Step 4: I/O Performance

```bash
# If disk I/O is the bottleneck:
# 1. Identify large queries via profiler
# 2. Check for missing indexes
# 3. Consider hardware upgrade
```

### Step 5: Increase Oplog Size (if approaching full)

```javascript
// Check oplog status
db.getReplicationInfo()

// If oplog window < 24h, increase size
db.adminCommand({
  replSetResizeOplog: 1,
  size: 20480  // 20 GB
})
```

## Verification

After implementing fixes, verify:

```bash
# 1. Check lag is decreasing
mongo --eval "rs.printSlaveReplicationInfo()"

# 2. Monitor for 15 minutes
watch -n 60 'mongo --quiet --eval "rs.printSlaveReplicationInfo()"'

# 3. Verify metrics in Grafana
```

## Escalation

**Escalate if:**
- Lag continues to increase after 30 minutes
- Lag exceeds 300 seconds (5 minutes)
- Multiple secondaries are lagging
- Primary performance is degraded

**Escalation Path:**
1. Senior DBA (30 min)
2. MongoDB Architect (1 hour)
3. MongoDB Support (2 hours for Enterprise customers)

## Prevention

1. **Monitoring:**
   - Alert at 30s lag (info)
   - Alert at 60s lag (warning)
   - Alert at 300s lag (critical)

2. **Capacity Planning:**
   - Review oplog size monthly
   - Monitor disk I/O trends
   - Ensure secondaries have equal resources to primary

3. **Best Practices:**
   - Use appropriate read preference
   - Implement proper indexing
   - Regular performance reviews

## Related Alerts

- MongoDBOplogWindowShort
- MongoDBSlowQueriesHigh
- MongoDBHighDiskIO

## References

- [MongoDB Replication Docs](https://docs.mongodb.com/manual/replication/)
- [Troubleshooting Replication Lag](https://docs.mongodb.com/manual/tutorial/troubleshoot-replica-sets/)
- Internal Wiki: https://wiki.example.com/mongodb/replication-lag

## Change Log

| Date | Author | Change |
|------|--------|--------|
| 2024-12-01 | John Doe | Initial version |
| 2024-12-08 | Jane Smith | Added escalation paths |
```

---

## Alert Hygiene et Maintenance

### 1. Review Trimestriel des Alertes

```python
#!/usr/bin/env python3
# alert_hygiene.py

import requests
from collections import defaultdict
from datetime import datetime, timedelta

class AlertHygiene:
    def __init__(self, prometheus_url):
        self.prometheus_url = prometheus_url

    def get_alert_statistics(self, days=90):
        """Analyze alert firing frequency"""

        # Query for alert history
        end = datetime.now()
        start = end - timedelta(days=days)

        query = 'ALERTS{alertstate="firing"}'
        response = requests.get(
            f"{self.prometheus_url}/api/v1/query_range",
            params={
                'query': query,
                'start': start.timestamp(),
                'end': end.timestamp(),
                'step': '1h'
            }
        )

        data = response.json()['data']['result']

        # Analyze by alert name
        alert_stats = defaultdict(lambda: {
            'firing_count': 0,
            'total_duration_hours': 0,
            'last_fired': None
        })

        for series in data:
            alert_name = series['metric']['alertname']
            for timestamp, value in series['values']:
                if float(value) > 0:
                    alert_stats[alert_name]['firing_count'] += 1

        return alert_stats

    def generate_report(self):
        """Generate alert hygiene report"""

        stats = self.get_alert_statistics()

        report = []
        report.append("# Alert Hygiene Report\n")
        report.append(f"Period: Last 90 days\n")
        report.append(f"Generated: {datetime.now().isoformat()}\n\n")

        # Alerts that never fired
        report.append("## Alerts Never Fired\n")
        never_fired = [name for name, data in stats.items()
                      if data['firing_count'] == 0]
        for alert in never_fired:
            report.append(f"- {alert} - **Consider removing**\n")

        # Alerts firing too frequently
        report.append("\n## Alerts Firing Too Frequently\n")
        frequent = [(name, data) for name, data in stats.items()
                   if data['firing_count'] > 100]
        frequent.sort(key=lambda x: x[1]['firing_count'], reverse=True)

        for name, data in frequent:
            report.append(f"- {name}: {data['firing_count']} times\n")
            report.append(f"  **Action:** Review threshold or add inhibition\n")

        # Recommendations
        report.append("\n## Recommendations\n")
        report.append("1. Review alerts that never fired\n")
        report.append("2. Adjust thresholds for frequently firing alerts\n")
        report.append("3. Ensure all alerts have runbooks\n")
        report.append("4. Test alert routing monthly\n")

        return ''.join(report)

# Usage
hygiene = AlertHygiene("http://prometheus:9090")
print(hygiene.generate_report())
```

### 2. Alert Testing Framework

```python
#!/usr/bin/env python3
# test_alerts.py

import requests
import time

class AlertTester:
    def __init__(self, prometheus_url, alertmanager_url):
        self.prometheus_url = prometheus_url
        self.alertmanager_url = alertmanager_url

    def inject_test_metric(self, metric_name, value, labels):
        """Inject a test metric to trigger alert"""

        # Push to Pushgateway
        pushgateway_url = "http://pushgateway:9091/metrics/job/test"

        metric_line = f'{metric_name}{{{",".join([f"{k}=\"{v}\"" for k, v in labels.items()])}}} {value}'

        response = requests.post(
            pushgateway_url,
            data=metric_line,
            headers={'Content-Type': 'text/plain'}
        )

        return response.status_code == 200

    def wait_for_alert(self, alert_name, timeout=300):
        """Wait for alert to fire"""

        start_time = time.time()

        while time.time() - start_time < timeout:
            response = requests.get(f"{self.alertmanager_url}/api/v1/alerts")
            alerts = response.json()['data']

            for alert in alerts:
                if alert['labels']['alertname'] == alert_name:
                    return True

            time.sleep(10)

        return False

    def test_alert_chain(self, test_config):
        """Test complete alert chain"""

        print(f"Testing alert: {test_config['alert_name']}")

        # Step 1: Inject metric
        print("1. Injecting test metric...")
        self.inject_test_metric(
            test_config['metric'],
            test_config['value'],
            test_config['labels']
        )

        # Step 2: Wait for alert
        print("2. Waiting for alert to fire...")
        if self.wait_for_alert(test_config['alert_name']):
            print("✓ Alert fired successfully")
        else:
            print("✗ Alert did not fire (timeout)")
            return False

        # Step 3: Check notifications (manual verification)
        print("3. Check notification channels:")
        print("   - Slack: Check #mongodb-alerts")
        print("   - Email: Check inbox")
        print("   - PagerDuty: Check incidents")

        # Step 4: Cleanup
        print("4. Cleaning up...")
        self.cleanup_test_metric(test_config)

        return True

    def cleanup_test_metric(self, test_config):
        """Remove test metric"""
        requests.delete(f"http://pushgateway:9091/metrics/job/test")

# Test configuration
test_config = {
    'alert_name': 'MongoDBReplicationLagHigh',
    'metric': 'mongodb_rs_members_optimeDate',
    'value': 1000,  # Simulate high lag
    'labels': {
        'cluster': 'test',
        'state': 'SECONDARY',
        'member_name': 'test-secondary'
    }
}

tester = AlertTester("http://prometheus:9090", "http://alertmanager:9093")
tester.test_alert_chain(test_config)
```

---

## Troubleshooting Alerting

### Problème 1 : Alertes Non Reçues

**Diagnostic** :

```bash
# 1. Vérifier que l'alerte fire dans Prometheus
curl http://prometheus:9090/api/v1/alerts | jq '.data.alerts[] | select(.labels.alertname=="MongoDBReplicationLagHigh")'

# 2. Vérifier Alertmanager a reçu l'alerte
curl http://alertmanager:9093/api/v1/alerts | jq '.data[] | select(.labels.alertname=="MongoDBReplicationLagHigh")'

# 3. Vérifier les silences
curl http://alertmanager:9093/api/v1/silences | jq .

# 4. Vérifier les routes
curl http://alertmanager:9093/api/v1/status | jq .config.route

# 5. Tester le receiver manuellement
curl -XPOST http://alertmanager:9093/api/v1/alerts -d '[
  {
    "labels": {
      "alertname": "TestAlert",
      "severity": "warning"
    },
    "annotations": {
      "summary": "Test alert"
    }
  }
]'
```

### Problème 2 : Alert Storm (Trop d'Alertes)

**Solutions** :

```yaml
# 1. Grouping agressif
route:
  group_by: ['alertname', 'cluster']
  group_wait: 30s
  group_interval: 5m

# 2. Inhibition rules
inhibit_rules:
  # Si cluster down, supprimer tout
  - source_match:
      alertname: 'ClusterDown'
    target_match_re:
      alertname: '.*'
    equal: ['cluster']

# 3. Rate limiting (Alertmanager ne supporte pas nativement)
# Utiliser un proxy ou script
```

### Problème 3 : Faux Positifs

**Analyse** :

```python
#!/usr/bin/env python3
# false_positive_analyzer.py

import requests
from collections import Counter

def analyze_false_positives(prometheus_url, days=30):
    """Analyze alerts that resolve quickly (likely false positives)"""

    query = '''
        ALERTS{alertstate="firing"}
        unless
        ALERTS{alertstate="firing"} offset 5m
    '''

    response = requests.get(f"{prometheus_url}/api/v1/query", params={'query': query})
    alerts = response.json()['data']['result']

    # Count by alert name
    alert_counts = Counter(alert['metric']['alertname'] for alert in alerts)

    print("Potential False Positives (firing < 5 minutes):")
    for alert, count in alert_counts.most_common(10):
        print(f"  {alert}: {count} occurrences")
        print(f"    → Consider increasing 'for' duration")

analyze_false_positives("http://prometheus:9090")
```

---

## Bonnes Pratiques

### 1. Checklist Configuration Alerting

```yaml
Setup Initial:
  ✓ Prometheus rules configurées
  ✓ Alertmanager déployé et configuré
  ✓ Channels de notification testés (Slack, PagerDuty, Email)
  ✓ Templates de messages personnalisés
  ✓ Inhibition rules définies
  ✓ Silences pour maintenance configurables

Par Alert:
  ✓ Sévérité appropriée (P0-P3)
  ✓ Threshold basé sur baseline + analyse
  ✓ 'for' duration pour éviter faux positifs
  ✓ Annotations complètes (summary, description, action)
  ✓ Runbook créé et lié
  ✓ Dashboard Grafana lié
  ✓ Testé en environnement non-prod

Documentation:
  ✓ Runbooks à jour pour chaque alerte P0/P1
  ✓ Escalation paths documentés
  ✓ On-call rotation définie
  ✓ Postmortems archivés

Maintenance:
  ✓ Review trimestriel des alertes
  ✓ Suppression des alertes jamais fired
  ✓ Ajustement des thresholds selon tendances
  ✓ Test mensuel du système d'alerting
```

### 2. Métriques de Qualité de l'Alerting

```python
#!/usr/bin/env python3
# alerting_quality_metrics.py

class AlertingQualityMetrics:
    def __init__(self, incident_data):
        self.incidents = incident_data

    def calculate_metrics(self):
        """Calculate alerting quality metrics"""

        metrics = {}

        # MTTA - Mean Time To Acknowledge
        ack_times = [i['ack_time'] - i['alert_time']
                    for i in self.incidents if i['ack_time']]
        metrics['mtta_minutes'] = sum(ack_times) / len(ack_times) / 60

        # MTTR - Mean Time To Resolution
        resolution_times = [i['resolved_time'] - i['alert_time']
                           for i in self.incidents if i['resolved_time']]
        metrics['mttr_minutes'] = sum(resolution_times) / len(resolution_times) / 60

        # False Positive Rate
        false_positives = [i for i in self.incidents if i['false_positive']]
        metrics['false_positive_rate'] = len(false_positives) / len(self.incidents) * 100

        # Alert Actionability
        actionable = [i for i in self.incidents if i['action_taken']]
        metrics['actionability_rate'] = len(actionable) / len(self.incidents) * 100

        return metrics

# Target metrics
target_metrics = {
    'mtta_minutes': 5,      # < 5 minutes pour P0
    'mttr_minutes': 30,     # < 30 minutes pour P0
    'false_positive_rate': 5,  # < 5%
    'actionability_rate': 95   # > 95%
}
```

### 3. Alert Fatigue Prevention

```yaml
Stratégies:
  1. Grouping Intelligent:
     - Group by cluster + alertname
     - Wait period: 10-30s
     - Interval: 5-10 minutes

  2. Progressive Severity:
     - Info → Notice → Warning → Critical
     - Ne page que sur Critical
     - Temps d'escalade appropriés

  3. Suppression Contextuelle:
     - Maintenance windows
     - Inhibition rules
     - Automatic silence during deploys

  4. Quality Over Quantity:
     - Privilégier precision sur recall
     - Mieux manquer une alerte non-critique
       que spammer avec des faux positifs

  5. Self-Service Silencing:
     - Permettre aux équipes de créer silences
     - Avec approbation pour alertes P0
     - Audit trail complet
```

---

## Résumé pour SRE

### Alertes MongoDB Critiques (Must Have)

```yaml
Top 10 Critical Alerts:
  1. MongoDBInstanceDown
     - Threshold: down for 2 min
     - Action: Page immediately

  2. MongoDBPrimaryDown
     - Threshold: no primary for 1 min
     - Action: Page + escalate

  3. MongoDBReplicationStopped
     - Threshold: lag > 1 hour
     - Action: Page

  4. MongoDBWriteErrors
     - Threshold: > 5% error rate
     - Action: Page

  5. MongoDBBackupFailed
     - Threshold: no backup in 24h
     - Action: Escalate to backup team

  6. MongoDBDiskSpaceCritical
     - Threshold: < 10% free
     - Action: Immediate expansion

  7. MongoDBOplogFull
     - Threshold: > 95% used
     - Action: Increase size immediately

  8. MongoDBConnectionsExhausted
     - Threshold: > 95% used
     - Action: Investigate + increase limit

  9. MongoDBCertificateExpired
     - Threshold: expired or < 7 days
     - Action: Rotate immediately

  10. MongoDBReplicaSetMajorityDown
      - Threshold: lost quorum
      - Action: Page + manual intervention
```

### Quick Commands

```bash
# Vérifier alertes actives
curl -s http://prometheus:9090/api/v1/alerts | jq '.data.alerts[] | select(.state=="firing") | .labels.alertname' | sort -u

# Créer un silence (1 heure)
curl -XPOST http://alertmanager:9093/api/v1/silences -d '{
  "matchers": [{"name": "alertname", "value": "MongoDBReplicationLag", "isRegex": false}],
  "startsAt": "2024-12-08T15:00:00Z",
  "endsAt": "2024-12-08T16:00:00Z",
  "createdBy": "oncall@example.com",
  "comment": "Planned maintenance"
}'

# Lister silences actifs
curl -s http://alertmanager:9093/api/v1/silences | jq '.data[] | select(.status.state=="active")'

# Tester une alerte
amtool alert add alertname=TestAlert severity=warning cluster=test
```

---

## Conclusion

Un système d'**alerting efficace** est crucial pour maintenir la disponibilité et les performances de MongoDB en production. Les points clés à retenir :

1. **Alerter sur les symptômes**, pas les causes
2. **Éviter l'alert fatigue** via grouping et inhibition
3. **Chaque alerte doit être actionable** avec runbook
4. **Tester régulièrement** le système d'alerting
5. **Maintenir et ajuster** basé sur les incidents réels

**Prochaines étapes** :
- Implémenter les alertes critiques (Top 10)
- Créer les runbooks pour alertes P0/P1
- Configurer les channels de notification
- Tester le flow complet end-to-end
- Former l'équipe on-call aux procédures
- Établir la rotation et l'escalation

---

**Références** :
- [Prometheus Alerting](https://prometheus.io/docs/alerting/latest/)
- [Alertmanager Configuration](https://prometheus.io/docs/alerting/latest/configuration/)
- [SRE Book - Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [My Philosophy on Alerting](https://docs.google.com/document/d/199PqyG3UsyXlwieHaqbGiWVa8eMWi8zzAn0YfcApr8Q/edit)

⏭️ [Diagnostics avec FTDC](/13-monitoring-administration/10-diagnostics-ftdc.md)
