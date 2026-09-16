# learn-ops-api: AI-Assisted Exploration

## 1. Top-level folders in `learn-ops-api`

| Folder | Why does this folder need to exist? |
|--------|-------------------------------------|
| `.github` | Holds GitHub Actions workflow config, so CI (tests, linting, etc.) runs automatically on pushes/PRs without needing to be triggered by hand. |
| `.vscode` | Shared editor settings/launch configs for VS Code, so everyone on the team gets consistent debugging/run configuration. |
| `config` | Deployment-facing configuration that lives outside the Django app itself — nginx conf files (`nginx.api.conf`, `nginx.client.conf`, an `nginx` folder) and a `learn-ops-api.yaml`. This is infrastructure config, not application code. |
| `LearningAPI` | The actual Django app — models, views, serializers, migrations, tests. This is where the business logic of the LMS lives. |
| `LearningPlatform` | The Django *project* package (as opposed to an *app*) — `settings.py`, root `urls.py`, `wsgi.py`. It's the glue that wires apps together and configures the whole Django process. |
| `logs` | Runtime log output (e.g. `learning_platform.json`) written while the server runs. Exists so log data has a stable place to land, separate from source code. |
| `LogViewer` | A second, small Django app dedicated to viewing logs in the browser (its own `views.py`, `urls.py`, `templates`). Kept separate from `LearningAPI` because it's a distinct concern (observability tooling) rather than core LMS functionality. |
| `static` | Source static assets (currently just `admin`) that Django's static file system collects from. |
| `staticfiles` | The *collected* static output (`admin`, `rest_framework`) produced by `collectstatic` — what actually gets served in production. Separate from `static` because one is source, the other is build output. |
| `templates` | Server-rendered HTML templates (currently admin template overrides) used by Django's templating engine. |

## 2. Folders inside `LearningAPI`

| Folder | What responsibility does it own and why? |
|--------|------------------------------------------|
| `fixtures` | JSON seed data (e.g. `auth_user.json`, `LearningAPI_capstone.json`) loaded via Django's `loaddata` to populate a database for development or tests, so you don't have to manually recreate sample records. |
| `migrations` | Django's auto-generated schema history — one file per model change over time. Owns the responsibility of keeping the database schema in sync with `models/` in a reproducible, version-controlled way. |
| `models` | Defines the data layer — the Django ORM classes that map to database tables, organized into sub-packages (`coursework`, `people`, `skill`) plus standalone files like `tag.py`. Owns what data exists and how it relates. |
| `serializers` | DRF serializers that translate between model instances and JSON. Owns the responsibility of shaping data for the API boundary (validation on the way in, formatting on the way out). |
| `tests` | The automated test suite for the app (e.g. `test_cohort.py`, `test_course.py`). Owns verifying that models/views/serializers behave correctly as the code evolves. |
| `views` | Request-handling logic — one file per resource/concern (e.g. `cohort_view.py`, `capstone_view.py`, `auth.py`), plus `github`/`oauth2` sub-packages for auth flows. Owns interpreting HTTP requests and producing responses, using models and serializers. |

## 3. What is the Pipfile?

`Pipfile` is the dependency manifest for `pipenv`, the Python packaging tool this project uses instead of a bare `requirements.txt`. It declares, by name, every third-party package the project depends on, split into two groups:

- `[packages]` — runtime dependencies (e.g. `django`, `djangorestframework`, `django-allauth`) needed for the app to actually run.
- `[dev-packages]` — dependencies only needed while developing (e.g. `pytest`, `pytest-django`, `pylint`, `debugpy`) that shouldn't ship to production.

It also pins the exact Python interpreter version under `[requires]` (`python_version = "3.11.11"`), and defines a `[scripts]` shortcut (`migrate = "python3 manage.py migrate"`) so common commands can be run as `pipenv run migrate`.

The Pipfile exists so the project's dependencies are declared in one version-controlled place, decoupled from whatever happens to be installed on a given machine. When someone runs `pipenv install`, pipenv resolves these declarations into a `Pipfile.lock` — a fully pinned, hashed snapshot of every package and sub-dependency version — and builds an isolated virtual environment from it. This is the same underlying problem `package.json`/`package-lock.json` solve in Node: reproducible installs across teammates, CI, and deployment, without polluting the system Python.

## 4. Key packages

| Package | What functionality does it provide and why? |
|---------|---------------------------------------------|
| django | The web framework the whole backend is built on — ORM (models → database tables), the admin site, URL routing, middleware, migrations, auth primitives (`django.contrib.auth`), and the request/response cycle. It's the foundation everything else in the Pipfile plugs into; `LearningPlatform/settings.py` and `urls.py` are pure Django project config, and `LearningAPI/models/` is the Django ORM layer. |
| djangorestframework | Layered on top of Django to turn it into a JSON API rather than a server-rendered HTML site. Provides serializers (`LearningAPI/serializers/`) for converting model instances to/from JSON, generic views/viewsets for CRUD endpoints, browsable API tooling, pagination, and pluggable authentication/permission classes. This project's `settings.py` sets DRF's `TokenAuthentication` as the default authenticator and `IsAuthenticated` as the default permission — i.e. DRF is what makes `LearningAPI` an API rather than just a Django app. |
| django-allauth | Handles authentication flows beyond Django's built-in username/password login — account registration/management (`allauth.account`) and, critically for this project, social/OAuth login (`allauth.socialaccount`). `settings.py` enables `allauth.socialaccount.providers.github` specifically, which matches the `oauth2`/`github` sub-packages seen under `LearningAPI/views/` in section 2 — this project lets users authenticate via GitHub OAuth instead of (or alongside) a local password, which fits an LMS aimed at developers. |

## 5. What does `decorators.py` do?

A decorator is a function that wraps another function to add behavior around it, without changing the wrapped function's own code. In Python, `@some_decorator` above a function definition is shorthand for `func = some_decorator(func)` — the decorator returns a new function that typically does something before and/or after calling the original.

`LearningAPI/decorators.py` defines two permission-check decorators, `is_instructor()` and `is_staff()`. Both follow the same pattern: they take no arguments, return an inner `decorator(func)`, which returns a `__wrapper(request, *args, **kwargs)` that:

1. Checks `request.user.groups.filter(name='Instructors').exists()` (or `'Staff'` for `is_staff`).
2. If true, calls and returns the original view method (`func(request, *args, **kwargs)`).
3. If false, short-circuits and returns a `401 UNAUTHORIZED` DRF `Response` instead — the wrapped view logic never runs.

This is used to gate individual view methods behind a group-membership check without repeating an `if`/`else` permission check inside every view. In this codebase it's applied via `@method_decorator(is_instructor())` on class-based view methods in `LearningAPI/views/course_view.py`, `student_view.py`, and `student_assessment.py` — e.g. only instructors can create/update courses or edit student assessments. It's a cross-cutting concern (authorization) factored out of the business logic so each view method stays focused on what it does, not who's allowed to do it.

## 6. What is a serializer, and how does it fit the request/response cycle?

Note: `LearningAPI/serializers/` is a folder (one file per resource, e.g. `cohort_serializer.py`), not a single `serializers.py`; some views also define extra serializers inline (e.g. `CohortSerializer`/`MiniCohortSerializer` in `views/cohort_view.py`). Content below is based on those files.

A serializer converts between Django model instances (Python objects backed by the database) and JSON (what an HTTP client sends and receives). It's the translation layer at the API boundary, and it works in both directions:

- **Outbound (model → JSON):** given a model instance (or queryset), `.data` produces a JSON-serializable dict/list — what actually gets written into the HTTP response body.
- **Inbound (JSON → model):** given raw request data, the serializer validates it (`.is_valid()`) against field types/constraints and, once valid, can create or update a model instance (`.save()`) from `.validated_data`.

The simplest form, `LearningAPI/serializers/cohort_serializer.py`, is a `ModelSerializer` that just points at a model and lets DRF infer the fields:

```python
class CohortSerializer(serializers.ModelSerializer):
    class Meta:
        model = Cohort
        fields = '__all__'
```

But serializers aren't just a 1:1 mirror of the model — they also shape the API response. The `CohortSerializer` defined in `views/cohort_view.py` (a separate, more detailed serializer than the one above) adds computed/nested fields that don't exist directly as model columns:

```python
class CohortSerializer(serializers.ModelSerializer):
    courses = CohortCourseSerializer(many=True)
    attendance_sheet_url = serializers.SerializerMethodField()
    ...
    def get_attendance_sheet_url(self, obj):
        try:
            return obj.info.attendance_sheet_url
        except Exception:
            return ""
```

`courses` nests a related model's data inline; `SerializerMethodField()` computes a value at serialization time by calling a `get_<field>()` method. This is the kind of shaping a raw model-to-dict conversion can't do — a Django REST API needs serializers because the shape of data useful over HTTP (nested, computed, filtered down, validated) is rarely identical to the shape of a database row, and because incoming request data has to be validated and converted to Python types *before* it's trusted enough to save to the database.

**Where it sits in the request/response cycle** (seen concretely in `views/cohort_view.py`):

1. A request hits a DRF view (`CohortViewSet`).
2. On write (`create`): the view reads `request.data` (parsed JSON), builds/updates a `Cohort` model instance, and saves it — validation here is done manually in this view rather than via the serializer's `.is_valid()`, though that's the more idiomatic DRF pattern.
3. On read (`retrieve`/`list`): the view fetches `Cohort` instance(s) from the ORM, then does `serializer = CohortSerializer(cohort, context={'request': request})`.
4. `serializer.data` converts the model instance(s) into a JSON-serializable structure.
5. The view wraps that in `Response(serializer.data)`, and DRF's renderer turns it into the actual JSON HTTP response body sent back to the client.

So the cycle is: HTTP request → view → ORM (model) ⇄ serializer ⇄ JSON → HTTP response.

## 7. One model and what it represents

A Django model is a Python class that maps to a database table — each class attribute defined as a `models.Field` becomes a column, each instance becomes a row, and Django's ORM generates the SQL (and, via `migrations/`, the schema changes) behind it. Models are where the shape and behavior of the application's data lives, separate from how that data gets exposed (serializers) or manipulated over HTTP (views).

**Model: `Cohort`** (`LearningAPI/models/people/cohort.py`)

```python
class Cohort(models.Model):
    name = models.CharField(max_length=55, unique=True)
    slack_channel = models.CharField(max_length=55, unique=False)
    start_date = models.DateField(...)
    end_date = models.DateField(...)
    break_start_date = models.DateField(...)
    break_end_date = models.DateField(...)
    active = models.BooleanField(default=False)
```

A `Cohort` represents a single class/group of students going through the program together — e.g. "the JavaScript cohort that started in January." It tracks:

- `name` — the cohort's identifying label (unique, so two cohorts can't collide).
- `slack_channel` — the Slack channel used to communicate with that specific group, tying the LMS into the tools the school actually uses.
- `start_date`/`end_date` — the cohort's active enrollment window.
- `break_start_date`/`break_end_date` — a scheduled break within the program.
- `active` — whether the cohort is currently running.

The API needs to track this because a cohort is the organizing unit almost everything else in the LMS hangs off of: `NssUserCohort` links students/staff to a specific cohort, `CohortCourse` links a cohort to the courses it's taking, and the model's own `coaches` property queries `NssUserCohort` to find staff assigned to it. Without a `Cohort` record, there'd be no way to group students, scope assignments/assessments to "this class right now," or answer basic scheduling questions like "is this cohort currently in session" — which is exactly what its `is_active_on_date()` method answers by checking a given date against `start_date`/`end_date`.

## 8. Views vs. viewsets

| Type | Example class | When to use it |
|------|--------------|----------------|
| View | `GithubLogin` — `LearningAPI/views/github_login.py:12` (subclasses `dj_rest_auth`'s `SocialLoginView`, itself a DRF `CreateAPIView`/`APIView`) | Use a plain view when an endpoint does one specific thing that isn't CRUD on a model — here, "exchange a GitHub OAuth code for a session." It's wired to a single explicit route in `LearningPlatform/urls.py:67`: `path('auth/github', views.GithubLogin.as_view(), name='github_login')`. One class, one URL, one job. |
| ViewSet | `CohortViewSet` — `LearningAPI/views/cohort_view.py:26` (subclasses DRF's `ViewSet`) | Use a viewset when a resource needs the standard set of operations — list, retrieve, create, update, delete, plus a few custom `@action`s (`assign`, `migrate`, `active` are checked in `CohortPermission.has_permission`). Instead of writing/wiring one URL per operation, it's registered once with a router: `router.register(r'cohorts', views.CohortViewSet, 'cohort')` in `urls.py:33`, and the router auto-generates the full set of RESTful URLs (`GET /cohorts/`, `POST /cohorts/`, `GET /cohorts/{pk}/`, etc.) from the methods the class defines (`list`, `retrieve`, `create`, ...). |

**The difference in short:** a `View`/`APIView` maps to one URL and you define exactly what happens for whatever HTTP methods you implement — full manual control, no assumptions about REST conventions. A `ViewSet` groups the whole family of operations for one resource into a single class and relies on a router to derive the URLs, which removes boilerplate but only pays off when the resource actually follows that CRUD-ish shape. This project's `urls.py` mixes both: most models get a `ViewSet` registered with the router (`cohorts`, `students`, `courses`, ...), while one-off, non-CRUD concerns like GitHub OAuth login get a single explicit `path()` to a plain view.

## 9. What replaces templates and why?

There are no HTML templates driving the API's actual responses (the `templates/` folder noted in section 1 only overrides Django *admin* templates — unrelated to `LearningAPI`'s endpoints). In the Model-Template-View pattern, the template's job is to take data and render it into a response format for the client. In this project, that role is filled by **DRF serializers** (`LearningAPI/serializers/`, plus the inline ones in `views/*.py`) combined with DRF's JSON renderer: the serializer decides what the output looks like (which fields, nested/computed values — see section 6), and DRF's renderer turns `serializer.data` into the actual JSON response body, the same way a template engine turns context data into an HTML response body.

This makes sense for a REST API because the client isn't a browser rendering HTML for a human — it's `learn-ops-client` (a separate frontend app) or some other program consuming JSON and rendering its own UI. Templates exist to produce *presentation* (HTML/CSS meant for a browser to display); an API's job stops at producing *data* in a predictable, machine-readable structure and letting the client decide how to present it. Serializers are the right tool for that because they express data shape and validation rules, not markup — which is exactly the boundary a REST API needs to draw between backend and frontend.