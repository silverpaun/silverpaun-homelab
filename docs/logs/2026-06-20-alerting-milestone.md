# 2026-06-20 — Alerting Milestone

## Summary

Today we confirmed end-to-end alerting in the SilverPaun Hetzner Homelab.

## What was tested

BookStack was manually stopped to simulate a service outage.

## Result

The monitoring stack successfully detected the outage and sent a Telegram notification.

After BookStack was started again, a resolved Telegram notification was also received.

## Confirmed flow

```text
BookStack down
↓
Blackbox Exporter probe_success = 0
↓
Prometheus ServiceDown alert
↓
Alertmanager
↓
n8n webhook
↓
Telegram firing notification

BookStack up
↓
Blackbox Exporter probe_success = 1
↓
Prometheus alert resolved
↓
Alertmanager
↓
n8n webhook
↓
Telegram resolved notification

docker stop bookstack

docker ps -a --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}" | grep -i book

curl "http://localhost:9115/probe?target=http://bookstack:80&module=http_2xx" | grep probe_success

curl -s http://localhost:9090/api/v1/rules | grep -i ServiceDown

curl -s http://localhost:9090/api/v1/alerts | grep -i -A20 ServiceDown

curl -s http://localhost:9093/api/v2/alerts

docker start bookstack

curl "http://localhost:9115/probe?target=http://bookstack:80&module=http_2xx" | grep probe_success

Status: PASSED
Firing notification: received
Resolved notification: received
Recovery: confirmed

Next time
Improve Telegram alert formatting.
Add automatic BookStack incident page creation.
Add more monitored services.
Document Alertmanager and n8n workflow.
