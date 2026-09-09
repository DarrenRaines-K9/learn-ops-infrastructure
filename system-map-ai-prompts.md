# System Map AI Prompts

## 1. Describe the System

Analyze this codebase/system and generate a system architecture diagram in Mermaid syntax (flowchart or C4-style, whichever renders the relationships more clearly). there is a file for you to place what you find in @system-map-ai.md in the system diagram section-*

Step 1 — Discovery:
- Identify every distinct service, application, or component (e.g. API server, worker, database, queue, cache, external API).
- Identify every connection between them, including inbound calls from external clients/frontends if applicable.

Step 2 — For each service, label its box with:
- Service name
- Primary framework/runtime (e.g. Django, React, Lambda, Node) if identifiable
- Role in one short phrase (e.g. "REST API", "async worker", "message broker")

Step 3 — For each connection, add a directional arrow labeled with:
- Protocol/mechanism (HTTP, gRPC, SQL query, pub/sub event, WebSocket, etc.)
- Port number, if known or configured
- Direction of data/control flow (arrow direction must reflect who initiates the call, not just data movement)

Step 4 — Constraints:
- Only include services and connections that actually exist in the system — no assumed/typical components.
- If a detail (port, protocol) can't be determined from the available information, label it "unknown" rather than guessing or omitting the arrow.
- Do not add legends, titles, styling flourishes, or explanatory text beyond the diagram itself and its labels.
- Output valid Mermaid syntax only, in a single code block.

## 2. Convert to a Mermaid Diagram

Rules:
- Each service is a node with a short label
- Each connection is a directed edge
- Label every edge with the connection type (HTTP, DB, pub/sub, etc.) and port if known
- Do not add anything beyond services and their connections
Output only the Mermaid code block.