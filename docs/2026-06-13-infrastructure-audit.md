# Infrastructure Audit - 2026-06-13

## Server

- Hostname: deb-infra01
- OS: Debian
- Uptime during audit: 28 days
- RAM: 125 GiB
- Disk: 1.8 TB

## Running Services

- Portainer
- n8n
- WikiJS
- BookStack
- Uptime Kuma
- Prometheus
- Grafana
- Node Exporter
- cAdvisor
- Nginx Proxy Manager

## Completed

- DNS configured for silverpaun.dev
- Reverse proxy configured
- SSL enabled
- Grafana connected to Prometheus
- Node Exporter dashboard working
- cAdvisor dashboard working

## Pending

- Uptime Kuma to n8n production webhook
- Telegram alerts
- Email alerts
- UFW hardening
- Ollama/OpenWebUI
