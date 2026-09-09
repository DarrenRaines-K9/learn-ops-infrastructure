# System Map (AI)

## 1. System Diagram

```mermaid
flowchart TD
    Browser["Browser"]
    DevTooling["Dev Tooling"]
    client["client"]
    api["api"]
    database["database"]
    valkey["valkey"]
    monarch["monarch"]
    Observability["Observability"]
    ExternalAPIs["External APIs"]

    Browser -->|"HTTP, 3000"| client
    Browser -->|"HTTP, 3001"| Observability
    Browser -->|"HTTP, 8080/8081"| monarch
    client -->|"HTTP, 8000"| api
    api -->|"DB, 5432"| database
    api -->|"pub/sub, 6379"| valkey
    valkey -->|"pub/sub, 6379"| monarch
    api -->|"HTTPS, 443"| ExternalAPIs
    monarch -->|"HTTPS, 443"| ExternalAPIs
    Observability -->|"HTTP, 8000"| api
    Observability -->|"DB, 5432"| database
    DevTooling -->|"TCP, 5678"| api
    DevTooling -->|"TCP, 6379"| valkey
```
