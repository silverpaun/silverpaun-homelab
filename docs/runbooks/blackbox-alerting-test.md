# Blackbox Alerting Test Runbook

## Purpose

This document describes how the SilverPaun Hetzner Homelab validates end-to-end service-down and service-recovery alerting using:

* Blackbox Exporter
* Prometheus
* Alertmanager
* n8n webhook
* Telegram notification

The goal of this test is to confirm that when a monitored service becomes unavailable, the monitoring stack detects the failure, sends an alert notification, detects recovery, and sends a resolved notification.

---

## Current Alerting Flow

```text
Monitored service goes down
        ↓
Blackbox Exporter probes the HTTP endpoint
        ↓
probe_success becomes 0
        ↓
Prometheus evaluates the ServiceDown rule
        ↓
Alert becomes pending
        ↓
Alert becomes firing after the configured duration
        ↓
Prometheus sends the alert to Alertmanager
        ↓
Alertmanager sends a webhook to n8n
        ↓
n8n sends a Telegram notification
```

Recovery flow:

```text
Monitored service recovers
        ↓
Blackbox Exporter probes the HTTP endpoint
        ↓
probe_success becomes 1
        ↓
Prometheus marks the alert as resolved
        ↓
Alertmanager sends resolved event to n8n
        ↓
n8n sends resolved Telegram notification
```

---

## Tested Service

The tested service was:

```text
BookStack
```

Application container:

```text
bookstack
```

Database container:

```text
bookstack-db
```

During the test, only the BookStack application container was stopped. The database container remained online.

---

## Test Date

```text
2026-06-20
```

---

## Test Procedure

### 1. Stop the BookStack application container

```bash
docker stop bookstack
```

Expected result:

```text
bookstack
```

---

### 2. Confirm that BookStack is stopped

```bash
docker ps -a --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}" | grep -i book
```

Observed result:

```text
bookstack       Exited (0)
bookstack-db    Up
```

Interpretation:

```text
BookStack application is down.
BookStack database is still running.
```

---

### 3. Confirm Blackbox Exporter detects the failure

```bash
curl "http://localhost:9115/probe?target=http://bookstack:80&module=http_2xx" | grep probe_success
```

Observed result:

```text
probe_success 0
```

Interpretation:

```text
Blackbox Exporter successfully detected that the BookStack HTTP endpoint is unavailable.
```

---

### 4. Check Prometheus alert rule state

```bash
curl -s http://localhost:9090/api/v1/rules | grep -i ServiceDown
```

Observed result during the incident:

```text
"state":"pending"
"name":"ServiceDown"
"query":"probe_success == 0"
"duration":60
```

Interpretation:

```text
Prometheus detected the failed probe and started the ServiceDown alert.
The alert was initially pending because the rule requires the failure to last for 1 minute.
```

---

### 5. Check Prometheus active alerts

```bash
curl -s http://localhost:9090/api/v1/alerts | grep -i -A20 ServiceDown
```

Observed result during the incident:

```text
"alertname":"ServiceDown"
"instance":"http://bookstack:80"
"job":"blackbox-http"
"severity":"critical"
"source":"blackbox"
"state":"pending"
"value":"0e+00"
```

Interpretation:

```text
Prometheus correctly created a ServiceDown alert for the BookStack endpoint.
```

---

### 6. Check Alertmanager

```bash
curl -s http://localhost:9093/api/v2/alerts
```

Observed result while the alert was still pending:

```text
[]
```

Interpretation:

```text
Alertmanager had not received the alert yet because Prometheus was still in the pending state.
Alertmanager receives the alert after it becomes firing.
```

---

### 7. Confirm firing Telegram notification

Observed result:

```text
A Telegram notification arrived after approximately 4–5 minutes.
```

Interpretation:

```text
The full firing alert chain worked successfully:
Blackbox Exporter → Prometheus → Alertmanager → n8n → Telegram
```

The delay is expected because of:

```text
Prometheus scrape interval
Prometheus rule evaluation interval
ServiceDown rule duration
Alertmanager grouping/wait timing
n8n and Telegram delivery time
```

---

## Recovery Procedure

### 1. Start BookStack again

```bash
docker start bookstack
```

Expected result:

```text
bookstack
```

---

### 2. Confirm Blackbox Exporter detects recovery

```bash
curl "http://localhost:9115/probe?target=http://bookstack:80&module=http_2xx" | grep probe_success
```

Observed result:

```text
probe_success 1
```

Interpretation:

```text
BookStack is reachable again.
Blackbox Exporter confirms service recovery.
```

---

### 3. Confirm Prometheus rule returns to inactive

```bash
curl -s http://localhost:9090/api/v1/rules | grep -i ServiceDown
```

Observed result after recovery:

```text
"state":"inactive"
"name":"ServiceDown"
"alerts":[]
"health":"ok"
```

Interpretation:

```text
The ServiceDown alert is no longer active.
The monitored service recovered successfully.
```

---

### 4. Confirm resolved Telegram notification

Observed result:

```text
A resolved Telegram notification arrived after BookStack was started again.
```

Interpretation:

```text
The alerting stack successfully detected both the outage and the recovery event.
The notification path works for both firing and resolved alert states.
```

Confirmed notification lifecycle:

```text
Service down → Telegram firing alert
Service restored → Telegram resolved alert
```

---

## Configuration File Locations

### Git repository

```text
/home/unpas/git/silverpaun-homelab/stacks/monitoring/
```

Relevant files:

```text
stacks/monitoring/prometheus/prometheus.yml
stacks/monitoring/prometheus/rules/blackbox-alerts.yml
stacks/monitoring/blackbox/blackbox.yml
stacks/monitoring/alertmanager/alertmanager.yml
```

---

## Runtime Mounts

### Prometheus

Prometheus container reads configuration from:

```text
/opt/stacks/monitoring/prometheus/prometheus.yml
/opt/stacks/monitoring/prometheus/rules
```

Inside the container:

```text
/etc/prometheus/prometheus.yml
/etc/prometheus/rules
```

Important note:

```text
Prometheus configuration is maintained in the Git repository, but the running container currently uses the /opt path.
Changes to Prometheus config should be made in Git first, then copied to /opt.
```

Sync command:

```bash
sudo cp ~/git/silverpaun-homelab/stacks/monitoring/prometheus/prometheus.yml \
/opt/stacks/monitoring/prometheus/prometheus.yml

sudo cp -r ~/git/silverpaun-homelab/stacks/monitoring/prometheus/rules/* \
/opt/stacks/monitoring/prometheus/rules/

docker restart prometheus
```

---

### Blackbox Exporter

Blackbox Exporter reads configuration directly from the Git repository:

```text
/home/unpas/git/silverpaun-homelab/stacks/monitoring/blackbox/blackbox.yml
```

Inside the container:

```text
/etc/blackbox_exporter/config.yml
```

Restart command:

```bash
docker restart blackbox-exporter
```

---

### Alertmanager

Alertmanager reads configuration directly from the Git repository:

```text
/home/unpas/git/silverpaun-homelab/stacks/monitoring/alertmanager/alertmanager.yml
```

Inside the container:

```text
/etc/alertmanager/alertmanager.yml
```

Restart command:

```bash
docker restart alertmanager
```

---

## Health Checks After Restart

### Prometheus logs

```bash
docker logs --tail=50 prometheus
```

Healthy indicators:

```text
Completed loading of configuration file
Server is ready to receive web requests.
Starting rule manager
```

---

### Prometheus readiness

```bash
curl -s http://localhost:9090/-/ready
```

Expected result:

```text
Prometheus Server is Ready.
```

---

### Prometheus rule check

```bash
curl -s http://localhost:9090/api/v1/rules | grep -i ServiceDown
```

Expected healthy state when all services are online:

```text
"state":"inactive"
"health":"ok"
```

---

## Confirmed Result

The end-to-end alerting test was successful.

Confirmed:

```text
BookStack was manually stopped.
Blackbox Exporter detected probe_success 0.
Prometheus created the ServiceDown alert.
The firing alert notification reached Telegram after several minutes.
BookStack was restarted.
Blackbox Exporter confirmed recovery with probe_success 1.
Prometheus returned ServiceDown to inactive state.
A resolved notification also reached Telegram after recovery.
```

---

## Current Status

```text
Status: PASSED
Severity tested: critical
Service tested: BookStack
Alert rule tested: ServiceDown
Notification path tested: Alertmanager → n8n → Telegram
Firing notification tested: yes
Resolved notification tested: yes
Recovery tested: yes
```

---

## Operational Notes

### Why the firing alert was delayed

The Telegram alert did not arrive immediately. It arrived after approximately 4–5 minutes.

This is expected because several timing layers exist:

```text
Blackbox scrape interval
Prometheus rule evaluation interval
ServiceDown rule duration
Alertmanager group wait / grouping behavior
n8n processing
Telegram delivery
```

The Prometheus rule was observed in this state during the incident:

```text
state: pending
duration: 60
```

This means Prometheus detected the problem, but waited for the configured duration before marking the alert as firing.

---

### Difference between pending, firing, and resolved

```text
pending:
Prometheus sees the problem, but the alert condition has not lasted long enough yet.

firing:
The alert condition lasted long enough and Prometheus sent the alert to Alertmanager.

resolved:
The monitored service recovered and Prometheus/Alertmanager sent a recovery notification.
```

---

## Next Improvements

Recommended next steps:

```text
1. Reduce alert delay if needed.
2. Improve Telegram alert message formatting.
3. Add service name, environment, severity, and timestamp to Telegram messages.
4. Document Alertmanager receiver configuration.
5. Add more monitored HTTP targets.
6. Add a weekly alerting test procedure.
7. Standardize all config mounts so containers read from one source of truth.
8. Add this runbook to BookStack.
```

