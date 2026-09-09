# System Map (AI)

## 1. System Diagram

```mermaid
flowchart TD
    Browser["Browser<br/>external client<br/>end user"]
    Debugger["IDE Debugger<br/>debugpy client<br/>remote debugging tool"]
    client["client<br/>React (react-scripts)<br/>SPA frontend"]
    api["api<br/>Django + DRF<br/>REST API"]
    database["database<br/>PostgreSQL 16<br/>relational database"]
    valkey["valkey<br/>Valkey<br/>cache / pub-sub broker"]
    valkeyMonitor["valkey-monitor<br/>valkey-cli<br/>debug monitor sidecar"]
    monarch["monarch<br/>Python service<br/>async pub/sub worker (ticket migration)"]
    prometheus["prometheus<br/>Prometheus<br/>metrics scraper"]
    grafana["grafana<br/>Grafana<br/>metrics dashboard"]
    postgresExporter["postgres_exporter<br/>postgres-exporter<br/>Postgres metrics exporter"]
    github["GitHub API<br/>external service<br/>REST API (issues, OAuth)"]
    slack["Slack API<br/>external service<br/>chat.postMessage webhook"]

    Browser -->|"HTTP, port 3000, browser-initiated"| client
    Browser -->|"HTTP, port 3001, browser-initiated"| grafana
    Browser -->|"HTTP GET /health /logs, port 8081, browser-initiated"| monarch
    Browser -->|"HTTP GET /metrics, port 8080, browser-initiated"| monarch
    Debugger -->|"debugpy protocol, port 5678, debugger-initiated"| api
    client -->|"HTTP REST (axios), port 8000, client-initiated"| api
    api -->|"SQL query (psycopg2), port 5432, api-initiated"| database
    postgresExporter -->|"SQL query (stats collection), port 5432, exporter-initiated"| database
    api -->|"GET/SET cache commands, port 6379, api-initiated"| valkey
    api -->|"PUBLISH channel_migrate_issue_tickets, port 6379, api-initiated"| valkey
    valkey -->|"pub/sub message delivery (SUBSCRIBE), port 6379, pushed to subscriber"| monarch
    valkeyMonitor -->|"MONITOR command (TCP), port 6379, monitor-initiated"| valkey
    api -->|"HTTPS REST (OAuth token exchange, profile fetch), port 443, api-initiated"| github
    monarch -->|"HTTPS REST (GET/POST issues), port 443, monarch-initiated"| github
    api -->|"HTTPS POST chat.postMessage, port 443, api-initiated"| slack
    monarch -->|"HTTPS POST (migration status), port 443, monarch-initiated"| slack
    prometheus -->|"HTTP GET /metrics/metrics (scrape), port 8000, prometheus-initiated"| api
    prometheus -->|"HTTP GET /metrics (scrape), port 9187, prometheus-initiated"| postgresExporter
    grafana -->|"HTTP query API (datasource), port 9090, grafana-initiated"| prometheus
```