# Tech Stack AI Prompts

## 1. Run Questions

### 1a. Config Files
before anything locate the @learn-ops-infrastructure/tech-stack-ai.md. also recognize that there are multiple different repos opened in this IDE. 

i need you to find all config and environment files in the repo and fill in a markdown table in @learn-ops-infrastructure/tech-stack-ai.md  with columns Config File, Location, Config Value, What it's for, How it's used. Include at least 3 values per file.

### 1b. How to Start It
ok so were moving on to the next question in the @learn-ops-infrastructure/tech-stack-ai.md  1b.

i need you to look at the Makefile at the root of the learn-ops-infrastructure repo and document how to start the system. A Makefile is a list of named shortcuts (called "targets") that automate common tasks - things like make start or make dev. There are several targets in this one

### 1c. Where to Access It
ok now lets take a look somewhere else 

i need you to find each service's port and URL and put them in a markdown table with columns Service, Port, URL located at @learn-ops-infrastructure/tech-stack-ai.md section 1c

### 1d. Service Dependencies
now i need you to map the service dependencies in a markdown table with columns Service, Depends On, Why? located @learn-ops-infrastructure/tech-stack-ai.md 1d 

focus on the why i need a clear concise answer on the why x depends on y

### 1e. Main Entry Points
For each service in this codebase/repo, locate:
1. The startup/entry file (e.g., manage.py, app.py, main.py, server.js, wsgi.py, asgi.py — whichever applies to that service's stack)
2. The routes / URL configuration file (e.g., urls.py, routes.js, endpoints file — wherever URL patterns or route registrations are defined)

Output a single markdown table with these exact columns:

| Service | Startup File | Routes / URL Config File |
|---|---|---|

Requirements:
- One row per service.
- Give the file path relative to the repo root (not just the filename).
- If a service has multiple candidate files for either column (e.g., separate dev/prod entry points, or split URL configs), list all of them in that cell, separated by commas.
- If a file can't be found for a service, write "Not found" in that cell rather than leaving it blank or guessing.
- Do not include unrelated config files (e.g., Docker, CI/CD, env files) unless they are literally the startup entry point.


## 2. Services
I need you to fill out the table in Section 2 of @learn-ops-infrastructure/tech-stack-ai.md.

For each service in this repo, describe:
1. Service — the service name
2. Tech Stack — languages, frameworks, and key libraries used, each with its version number
3. Purpose — a brief (1–2 sentence) description of what the service does

Requirements:
- One row per service.
- Pull version numbers from the actual dependency files (e.g., requirements.txt, package.json, pyproject.toml, Pipfile.lock) rather than guessing — if a version can't be confirmed, write "unspecified" rather than inventing one.
- Match whatever table format/columns already exist in Section 2 of the file — don't invent new columns unless the section is currently empty.
- If a service's tech stack spans multiple layers (e.g., backend + frontend), list them clearly within the same cell (e.g., separated by line breaks or semicolons).
- Keep purpose descriptions factual and grounded in what the code actually does, not assumed intent.

## 3. System Overview
Add a "System Overview" section to `tech-stack-ai.md`, consisting of exactly 3 paragraphs:

1. **What it is** — What kind of application this is (architecture style, domain) and what problem it solves.
2. **Main features** — The application's main features, described from a user's perspective (what users can do, not implementation detail).
3. **Who uses it** — The user base and roles. If different roles interact with the system differently (e.g., admin vs. end user, different permission levels), describe how their experience or access differs.

Requirements:
- Ground every claim in the actual codebase — features, roles, and permissions should reflect what's implemented, not assumed.
- Write in plain prose (no bullet points, headers, or tables within the section) — 3 paragraphs only.
- Place the section logically within the existing document structure (e.g., before or after Section 2, wherever it best sets context).
- Keep each paragraph focused — don't let feature descriptions bleed into the "who uses it" paragraph or vice versa.