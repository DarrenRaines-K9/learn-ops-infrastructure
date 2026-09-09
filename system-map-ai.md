# System Map (AI)

## 1. System Diagram

```mermaid
flowchart TD
    Browser["Browser<br/>end user"]
    DevTooling["Dev Tooling<br/>IDE Debugger + valkey-monitor<br/>debug/inspection only"]
    client["client<br/>React SPA"]
    api["api<br/>Django + DRF"]
    database["database<br/>PostgreSQL 16"]
    valkey["valkey<br/>cache / pub-sub broker"]
    monarch["monarch<br/>async worker (ticket migration)"]
    Observability["Observability Stack<br/>Prometheus + Grafana + postgres_exporter"]
    ExternalAPIs["External APIs<br/>GitHub + Slack"]

    Browser -->|"HTTP, port 3000"| client
    Browser -->|"HTTP, port 3001 (Grafana UI)"| Observability
    Browser -->|"HTTP GET /health, /metrics, ports 8080-8081"| monarch
    client -->|"HTTP REST (axios), port 8000"| api
    api -->|"SQL (psycopg2), port 5432"| database
    api -->|"cache GET/SET + PUBLISH, port 6379"| valkey
    valkey -->|"pub/sub delivery, port 6379"| monarch
    api -->|"HTTPS REST (OAuth, chat.postMessage), port 443"| ExternalAPIs
    monarch -->|"HTTPS REST (issues, status updates), port 443"| ExternalAPIs
    Observability -->|"scrape /metrics, port 8000"| api
    Observability -->|"SQL stats query, port 5432"| database
    DevTooling -->|"debugpy protocol, port 5678"| api
    DevTooling -->|"MONITOR command, port 6379"| valkey
```

Grouped for readability: **Dev Tooling** = IDE Debugger + valkey-monitor sidecar (debug-only, not core request flow). **Observability Stack** = Prometheus + Grafana + postgres_exporter (Grafana's Prometheus datasource query and Prometheus's scrape of postgres_exporter are now internal to this box). **External APIs** = GitHub + Slack.