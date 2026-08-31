# What config files exist in the system? 
The config files are already filled out for you - your job is to understand them, not edit them. Find the config files, then pick 3 values from each and explain what each one is for and how it is used by the system.

| Config File | Location | Config Value | What its for | How its used |
|----------|----------|----------|----------|----------|
| Makefile Targets    | SETUP_README.md         | multi         | to spin up project         | make '_____'        |
| Dependencies    | docker-compose.yml         | multi         | spin up dev infrastructure         | make up/down         |
| environment variables    | .env         | muilti        | secrets for env variables         | multi         |

# How do I start it? 
Find the `Makefile` at the root of the repo and look through its targets. Which ones are relevant for getting the system running? List the commands and detail how they differ.

- make up `spins up entire infra`
- make up-api `spins just the api up`
- make up-client-api `spins up the api and client`

# Where do i access it? 
Once the system is running, find the URL and port for each service and add them to your template:

- `localhost:3000` client
- `localhost:8000` api
- `5432` database
- `9090` prometheus
- `3001` grafana
- `9187` pg export

# What are the service dependencies?
For each service, does it need another service to be running before it can work? A dependency means one service relies on another to do its job - for example, an API that can't respond to requests unless a database is already running, or a frontend that is useless without a backend to talk to. List each service and anything it depends on.

- client - api, database
- api - database
- database - none 
- prometheus - api
- grafana - prometheus
- postgres_export - database

# What are the main entry points?
For each service, find the file where the process starts (e.g. where the server boots up) and where incoming requests are handled (e.g. routes or URL config). These are two different things - try to find both.
- api - entrypoint.sh

# Document the services
For each service in the system, add a row to the services table in your template with its name, tech stack (including versions), and purpose.