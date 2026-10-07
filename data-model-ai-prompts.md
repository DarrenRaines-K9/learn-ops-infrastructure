# Data Model AI Prompts

## 1. Find the Database Connection Details

I need to connect to this project's PostgreSQL database from pgAdmin. This is a read-only task: do not modify any files, do not start containers, and do not connect to the database.

SOURCES
Check all of these and report which ones exist:
- docker-compose.yml / compose.yaml (and any override files, e.g. docker-compose.override.yml)
- .env, .env.local, .env.example, and any file referenced by `env_file:` in the compose file
- Django settings (settings.py, DATABASES), which may read from environment variables
- Dockerfile, CI config, or infrastructure files, only if the values aren't found above

WHAT TO FIND
1. Host
2. Port
3. Database name
4. Username
5. Password

HOW TO RESOLVE EACH VALUE
- Follow variable substitution. If the compose file says ${POSTGRES_PORT:-5432}, report the variable name, the default, and the value from the .env file if one is set.
- Distinguish the two sides of a compose `ports:` mapping (HOST:CONTAINER). pgAdmin running on my machine connects to the HOST port on localhost, not the container port, and not the compose service name (e.g. "db"). The service name only works from inside the Docker network.
- If pgAdmin itself runs as a container in the same compose file, say so and give both connection options: the service name with the container port (inside the network) and localhost with the published port (outside).
- If the database is not Postgres (e.g. SQLite), or the compose file doesn't expose the port to the host, say so and stop. Don't invent a workaround.

OUTPUT
Return a table:

| Setting | Value | Source file : line | Variable name | Notes |

Then a second table titled "pgAdmin form values" with exactly what to type in each field:
General > Name (suggest one), Connection > Host name/address, Port, Maintenance database, Username, Password.

SECRETS
- Local development credentials (e.g. postgres/postgres, values from .env or .env.example for a local container): show the value, since I need it to connect.
- Anything that looks real (AWS RDS endpoints, production hostnames, non-default strong passwords, anything under a prod/staging name): give the file path and variable name, but replace the value with <redacted>. Tell me where to read it myself.
- Do not copy secrets into any file or command.

RULES
- Cite file path and line number for every value.
- If a value is not found, write "not found" and list where you looked. Do not fall back to Postgres defaults (5432, postgres) unless a file actually states them or the compose image's documented default applies. If you use an image default, label it "image default, not declared in project."
- If .env and .env.example disagree, report both and say which one the running containers would use.
- If anything is ambiguous (multiple compose files, multiple databases or environments), ask me before choosing.

## 2. Identify the Database Type

Determine which database engine this application uses, and which version of it.

1. Search the project files (not just one config file) for where the database is declared or configured. Check places such as:
   - Framework settings (e.g., Django settings.py DATABASES, ORM config)
   - docker-compose.yml / Dockerfile (image tags like postgres:15)
   - Infrastructure-as-code (Terraform, CloudFormation, ECS task definitions)
   - Dependency files (requirements.txt, package.json) for driver packages
   - CI config and .env.example files

2. Report:
   - Engine: the database name (e.g., PostgreSQL)
   - Version: the exact version string as written, or "not pinned" if none is specified
   - Source: file path and line number for each place it is declared

3. If the engine or version differs between environments (e.g., SQLite locally, PostgreSQL in production), list each environment separately.

4. If you cannot find a declaration, say so and list what you searched. Do not guess or infer a version from general knowledge.

## 3. Map the ORM to the Database

Answer in three parts. This is a read-only task: do not modify any files.

PART 1: Database communication
1. Identify the ORM the application uses and the DRF/Django layer that calls it.
2. Find where the database connection is configured (likely settings.py, DATABASES).
   Report the file path and line number.
3. Quote the ENGINE value exactly as written. Also report NAME, HOST, and PORT,
   but redact any password or secret values.
4. If the settings read from environment variables, say which variable names are
   used and where they are defined. Do not guess their values.

PART 2: Model fields vs. SQL columns
1. List the files in LearningAPI/models/ and pick one model that has at least
   one ForeignKey. Tell me which one you picked and why.
2. Output a table with these columns:
   | Python field | Field class | SQL column | SQL data type | Constraints |
3. Derive SQL column names and types from the actual model code plus the
   migrations in LearningAPI/migrations/. Account for db_column, the implicit
   id primary key, and the _id suffix on ForeignKey columns.
4. State the table name (db_table if set, otherwise the Django default).
5. Name the database engine your SQL types are for, since types differ between
   engines.

PART 3: What book.save() does
1. Open LearningAPI/views/book_view.py and find the create method. Show the
   method's code with line numbers.
2. State exactly what is called: book.save(), serializer.save(), or
   Book.objects.create(). Note it if they differ from what I wrote.
3. Explain what save() does when the instance has no primary key versus when it
   has one (INSERT vs UPDATE).
4. Show the SQL statement the ORM generates for this create call, with the real
   table and column names from Part 2. Label it one of:
   - Verified: captured by running it (e.g., CaptureQueriesContext or
     connection.queries in the Django shell)
   - Derived: reasoned from the code, not executed
   Do not present a derived statement as verified.

Rules for all parts:
- Cite file path and line number for every claim.
- If something cannot be found, say so and list where you looked. Do not fill
  gaps from general knowledge.
- If anything is ambiguous (multiple settings files, multiple

## 4. Generate a Database Diagram

Generate a Mermaid erDiagram of the database schema for this application. This is a read-only task: do not modify any files.

SOURCES
1. Read every model file in LearningAPI/models/ (including __init__.py, to catch models defined or re-exported there).
2. Cross-check against LearningAPI/migrations/ for fields or tables the model files don't show.
3. Include models that come from Django itself (e.g., auth User) if any model has a ForeignKey or OneToOne to them. Mark those entities with a comment.

ENTITIES
- One entity per table. Use the actual table name (db_table if set, otherwise the Django default: <app_label>_<modelname_lowercase>), not the Python class name.
- List every field as: <type> <column_name> [PK|FK|UK]
- Include the implicit `id` primary key on any model that doesn't declare its own.
- Use the real column name: apply db_column if set, and the _id suffix on ForeignKey and OneToOne columns.
- Map Django field classes to simple types (CharField -> string, IntegerField -> int, DateTimeField -> datetime, BooleanField -> boolean, TextField -> text, etc.). Do not invent types.

RELATIONSHIPS
- ForeignKey: one-to-many. Put the "one" side on the left, e.g. Author ||--o{ Book : "writes".
- OneToOneField: ||--||, or ||--o| if the field is nullable.
- ManyToManyField: show the auto-generated join table as its own entity (<table>_<field>) with two FKs, or the explicit `through` model if one is declared. Do not draw a direct many-to-many line.
- Use optional (o|) on the "one" side when the FK has null=True.
- Label every relationship with the field name that creates it.

OUTPUT
- Output only one ```mermaid code block. No explanation before or after.
- The block must be valid Mermaid erDiagram syntax. Entity names contain no spaces or hyphens, and attribute names contain no spaces.
- Order entities alphabetically so the diff is stable if I regenerate it.

RULES
- If a field or relationship can't be determined from the code, leave it out and put a `%% UNRESOLVED: <what and where>` comment inside the block instead of guessing.
- If anything is ambiguous in a way that changes the diagram (multiple apps with models, unclear join tables), ask me before generating.

## 5. Find Relationship Examples

Find one example each of a one-to-one, one-to-many, and many-to-many relationship in the Django models in LearningAPI/models/. This is a read-only task: do not modify any files.

HOW TO IDENTIFY EACH TYPE
- One-to-one: a OneToOneField on one model.
- One-to-many: a ForeignKey on the "many" side, pointing at the "one" side.
- Many-to-many: a ManyToManyField, OR an explicit join model (a model with two ForeignKeys, usually referenced through `through=`).

SEARCH
1. Read every model file in LearningAPI/models/, including __init__.py.
2. Search for OneToOneField, ForeignKey, and ManyToManyField (grep is fine, but read the surrounding code to confirm each match).
3. Count a relationship only if it is in a model defined by this project. Skip relationships that appear only in comments, migrations, or Django's built-in models.
4. Note that Django's built-in User may be the target of a relationship. That's fine, but the field must still live in a project model.

OUTPUT
Return a table with one row per relationship type:

| Type | Model (class) | File path : line | Field name | Points to | Notes |

- "Notes": include related_name, on_delete, and whether it is nullable, if set.
- For many-to-many via an explicit join model, give the join model's file and BOTH of its ForeignKey field names.

RULES
- If a type does not exist in the project, say so explicitly and list the files you searched. Do not invent an example or substitute a different relationship type.
- If several examples of a type exist, pick the clearest one and list the other candidates in one line each (model.field only).
- Cite file path and line number for every claim.
- If a judgment call changes the answer (for example, a many-to-many implemented as a join model rather than a ManyToManyField), tell me which you chose and why.