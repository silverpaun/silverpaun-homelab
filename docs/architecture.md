# Architecture

Internet
→ DNS: Porkbun
→ Nginx Proxy Manager
→ Internal Docker services

## Public Services

- grafana.silverpaun.dev → Grafana
- kuma.silverpaun.dev → Uptime Kuma
- n8n.silverpaun.dev → n8n
- wiki.silverpaun.dev → WikiJS
- docs.silverpaun.dev → BookStack

## Monitoring

- Prometheus collects metrics
- Node Exporter exposes host metrics
- cAdvisor exposes container metrics
- Grafana visualizes metrics
- Uptime Kuma monitors service availability
