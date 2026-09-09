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
    Browser -->|"HTTP GET /metrics (8080), /health (8081)"| monarch
    client -->|"HTTP REST (fetch), port 8000"| api
    api -->|"SQL (psycopg2), port 5432"| database
    api -->|"cache GET + PUBLISH, port 6379"| valkey
    valkey -->|"pub/sub delivery, port 6379"| monarch
    api -->|"HTTPS REST (OAuth, chat.postMessage), port 443"| ExternalAPIs
    monarch -->|"HTTPS REST (issues, status updates), port 443"| ExternalAPIs
    Observability -->|"scrape /metrics, port 8000"| api
    Observability -->|"SQL stats query, port 5432"| database
    DevTooling -->|"debugpy protocol, port 5678"| api
    DevTooling -->|"MONITOR command, port 6379"| valkey
```

Grouped for readability: **Dev Tooling** = IDE Debugger + valkey-monitor sidecar (debug-only, not core request flow). **Observability Stack** = Prometheus + Grafana + postgres_exporter (Grafana's Prometheus datasource query and Prometheus's scrape of postgres_exporter are now internal to this box). **External APIs** = GitHub + Slack.

## 2. Known Gaps / Caveats

- **api → valkey is a one-way cache read.** `popular_query.py` calls `valkey_client.get('search_results')`, but no code path in the api was found that ever `SET`s that key — the write side is either missing, dead, or lives outside this repo. Treat the "GET" edge as confirmed and the cache's population as an open question, not as a working read/write cache.
- **monarch's Prometheus metrics are not scraped.** `prometheus.yml` only defines `django` (`api:8000`) and `postgresql` (`postgres_exporter:9187`) jobs — there is no `monarch:8080` target. monarch exposes `/metrics` (port 8080, via `prometheus_client.start_http_server`) and `/health` (port 8081, via the Flask log web interface), but the only consumer of either today is a direct Browser/tooling request, not the Observability Stack.
- **nginx config exists but is unused here.** `learn-ops-api/config/nginx/` is not wired into any `docker-compose.yml` service in this environment, so it's correctly omitted from the diagram rather than a missing edge.