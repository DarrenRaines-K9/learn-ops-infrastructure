# lms-client

# Linting conventions for `learn-ops-client`

The project has **one** linting tool configured: ESLint. There is no Python linting (no `.pylintrc`, `setup.cfg`, or `pyproject.toml`), no Prettier config, and no standalone `.eslintrc*` file.

## ESLint

| | |
|---|---|
| **Version** | ESLint 8.41.0, using the `eslint-config-react-app` 7.0.1 preset (bundled with `react-scripts` 5.0.1) |
| **Config location** | [package.json:39-41](package.json#L39-L41): `"eslintConfig": { "extends": "react-app" }` |
| **Command** | `npx eslint . --ext .js,.jsx` |

The project has no `lint` script in `package.json`. Day to day, ESLint runs inside the Create React App dev server (`npm start`) and during `npm run build`, and its warnings show up in the terminal and the browser overlay. The `--ext` flag matters because ESLint 8 only checks `.js` files when you give it a directory.

**What this setup means in practice:** the project adds no rules of its own. It takes everything from the CRA preset, which is built to catch real bugs rather than enforce style. That's why nothing complains about semicolons, quote style or indentation. Two other settings make it lenient:
- The build script is `CI=false && react-scripts build` ([package.json:33](package.json#L33)). CRA normally fails a CI build when there are lint warnings; `CI=false` turns that off, so warnings never block a build.
- Almost every rule in the preset is set to `warn`. Only a few are `error`.

### Rule categories, with examples

**1. Possible errors and bad patterns (core ESLint)**
- Errors: `no-undef` (using an undeclared variable), `no-unused-expressions`, `no-restricted-globals` (blocks bare `event`, `name`, `location`, and similar browser globals), `no-restricted-properties`
- Warnings: `no-unused-vars`, `no-use-before-define`, `no-dupe-keys`, `no-duplicate-case`, `no-fallthrough`, `no-cond-assign` (except inside extra parentheses), `no-unreachable`, `array-callback-return`, `default-case` (unless there's a `// no default` comment), `eqeqeq` (smart mode), `no-eval`, `no-implied-eval`, `no-useless-escape`, `no-mixed-operators`, `no-template-curly-in-string`, `use-isnan`, `valid-typeof`

**2. React Hooks** (`eslint-plugin-react-hooks`), the most relevant category while you're learning hooks
- `react-hooks/rules-of-hooks`: **error**. Hooks can only be called at the top level of a component or custom hook, never inside conditions or loops.
- `react-hooks/exhaustive-deps`: warn. A `useEffect`, `useCallback` or `useMemo` dependency array must list every value the hook uses. Missing dependencies cause stale-closure bugs.

**3. React / JSX** (`eslint-plugin-react`)
- Errors: `react/jsx-no-undef` (using a component that hasn't been imported)
- Warnings: `react/jsx-no-duplicate-props`, `react/jsx-no-target-blank` (a `target="_blank"` link needs `rel="noreferrer"`), `react/jsx-no-comment-textnodes`, `react/no-direct-mutation-state`, `react/forbid-foreign-prop-types`

**4. Imports** (`eslint-plugin-import`)
- Errors: `import/first` (imports must come before other code), `import/no-amd`, `import/no-webpack-loader-syntax`
- Warnings: `import/no-anonymous-default-export` (use `export default function MyThing()` rather than `export default () => …`)

**5. Accessibility** (`eslint-plugin-jsx-a11y`), all warnings
- `jsx-a11y/alt-text` (an `<img>` needs `alt`), `jsx-a11y/anchor-is-valid` (no `href="#"` links used as buttons), `jsx-a11y/anchor-has-content`, `jsx-a11y/heading-has-content`, `jsx-a11y/iframe-has-title`, `jsx-a11y/aria-props`, `jsx-a11y/aria-role`, `jsx-a11y/role-has-required-aria-props`, `jsx-a11y/no-redundant-roles`, `jsx-a11y/scope`

**6. Flow** (`eslint-plugin-flowtype`): a few type-annotation sanity checks. They do nothing here because the project doesn't use Flow.

### Explicitly disabled rules

**In the preset:** `eslint-config-react-app` turns off `no-undef`, `no-unused-vars`, `no-redeclare`, `no-use-before-define`, `no-dupe-class-members`, `no-useless-constructor`, `no-array-constructor`, `no-unused-expressions` and `default-case`, but only for `.ts`/`.tsx` files, where TypeScript-aware versions take over. The repo has no TypeScript, so none of this affects you.

**In the project's own code:** there are four inline suppressions.

| Location | Directive | What it hides |
|---|---|---|
| [HumanDate.js:21](src/components/utils/HumanDate.js#L21) | `//eslint-disable-next-line` (no rule named) | `no-useless-escape` on `'\/'` in the regex replacement. `'/'` works the same without the backslash. Because no rule is named, the comment silences **every** rule on that line. |
| [StudentDashboard.js:20](src/components/dashboard/student/StudentDashboard.js#L20) | `// eslint-disable-next-line` (no rule named) | `no-unused-vars` on `firstName`, which is declared but never read. This also silences every rule on that line. |
| [StudentCapstoneList.js:55](src/components/cohorts/StudentCapstoneList.js#L55) | `/* eslint-disable no-undef */` | `no-undef` on `structuredClone`. ESLint 8's browser globals list predates that API, so it reports a false positive. |
| [StudentCardList.js:152](src/components/cohorts/StudentCardList.js#L152) | `/* eslint-disable no-undef */` | The same `structuredClone` false positive. |

**Something to watch in the last two rows:** a `/* eslint-disable */` block comment with no matching `/* eslint-enable */` turns the rule off **from that line to the end of the file**. In both files, `no-undef` is off for everything below the comment, so a real typo there (for example, a misspelled variable name) would not be reported. Two narrower fixes:
- Put `// eslint-disable-next-line no-undef` directly above the `structuredClone` line.
- Declare the global once in the config: `"eslintConfig": { "extends": "react-app", "globals": { "structuredClone": "readonly" } }`. This removes both suppressions.

### Related settings outside ESLint
- [.vscode/settings.json](.vscode/settings.json) sets `"javascript.validate.enable": false`. This turns off VS Code's built-in TypeScript-based JS checking so it doesn't overlap with ESLint. It doesn't turn ESLint off. Inline lint feedback in the editor needs the ESLint extension.
- The CI workflow in [.github/workflows/main.yml](.github/workflows/main.yml) has no lint step. Together with `CI=false`, that means **nothing automated enforces lint warnings**. They show up only if you read the dev-server or build output, or run the `npx eslint` command above.


# lms-api

# Linting Conventions: `learn-ops-api`

This file lists the automated linting tools this repo uses, how to run them, and what they enforce.

## Tools found

| Checked for | Result |
|---|---|
| `package.json` with an `eslintConfig` key | Not present (there's no `package.json` at all) |
| `.eslintrc`, `.eslintrc.js`, `.eslintrc.json` | Not present |
| **`.pylintrc`** | ✅ **Found** at [.pylintrc](.pylintrc) |
| `setup.cfg` with a `[pylint]` section | Not present |
| `pyproject.toml` with a `[tool.pylint]` section | Not present |

The project uses **one linter: Pylint.**

---

## Pylint

**Configuration file:** [.pylintrc](.pylintrc). It's 5 lines and changes only what's listed below. Everything else uses Pylint's defaults.

**Installed via:** [Pipfile](Pipfile). `pylint` is listed in both `[packages]` and `[dev-packages]`, and `pylint-django` is in `[packages]`.

### Command to run it

```bash
pipenv run pylint --load-plugins=pylint_django --django-settings-module=LearningPlatform.settings LearningAPI LearningPlatform
```

Or the minimal form, which uses only what the rc file configures:

```bash
pipenv run pylint LearningAPI LearningPlatform
```

> ⚠️ **Gap worth knowing:** `pylint-django` is installed, but `.pylintrc` has no `load-plugins=pylint_django` line, so the plugin only runs if you pass the flag yourself. Without it, Pylint doesn't understand Django magic. You'll get false positives like `no-member` on `Model.objects` and on reverse relations such as `student.notes`. The settings module `LearningPlatform.settings` is assumed from the directory layout; check it before relying on it.

### Rule categories enforced (Pylint defaults)

Pylint sorts its messages into five categories, and all of them are on except the three rules disabled below:

| Category | Code prefix | What it catches | Example rules |
|---|---|---|---|
| **Convention** | `C` | Style and naming (PEP 8) | `invalid-name` (C0103), `missing-module-docstring` (C0114), `missing-class-docstring` (C0115), `line-too-long` (C0301, default 100 chars), `wrong-import-order` (C0411) |
| **Refactor** | `R` | Code smells and complexity | `too-many-arguments` (R0913), `too-many-branches` (R0912), `no-else-return` (R1705), `too-few-public-methods` (R0903) |
| **Warning** | `W` | Likely bugs | `unused-import` (W0611), `unused-variable` (W0612), `dangerous-default-value` (W0102), `logging-fstring-interpolation` (W1203) |
| **Error** | `E` | Almost certainly bugs | `no-member` (E1101), `undefined-variable` (E0602), `import-error` (E0401) |
| **Fatal** | `F` | Pylint couldn't process the file | `astroid-error` (F0002) |

### Customized rules

**`[FORMAT] good-names=i,j,ex,pk`**

This is an allowlist for `invalid-name` (C0103). By default Pylint flags variable names shorter than 3 characters. These four are allowed:

- `i`, `j` for loop counters
- `ex` for the exception in `except Exception as ex:`
- `pk` for Django/DRF primary keys, as in `def retrieve(self, request, pk=None):`. Without this entry, every DRF ViewSet method would raise a warning.

### Explicitly disabled rules

From `[MESSAGES CONTROL] disable=...`:

| Rule | Code | What it would have flagged | Likely reason it's off |
|---|---|---|---|
| `broad-except` | W0718 (formerly W0703) | `except Exception:` blocks, i.e. catching every exception instead of specific ones | API views often wrap logic in a catch-all and return a `400`/`500` `Response`, so this pattern is deliberate here |
| `arguments-differ` | W0221 | An overriding method whose signature differs from the parent's | DRF ViewSet overrides (`list`, `retrieve`, `update`, `get_queryset`) often add or drop parameters such as `pk=None` compared with the base class |
| `missing-function-docstring` | C0116 | Any function or method without a docstring | Cuts noise in view, serializer and test files. Note that `missing-module-docstring` and `missing-class-docstring` are **still on** |

---

## Other observations

- **`autopep8`** is also in `[packages]`. It's a formatter, not a linter: it rewrites code to fix PEP 8 style instead of reporting problems. It has no config file, so it runs with defaults (`pipenv run autopep8 --in-place --recursive LearningAPI`).
- **Nothing runs automatically.** The GitHub workflows (`main.yml`, `collectstatic.yml`, `seed.yml`) and `.vscode/` contain no lint step, and there's no pre-commit config. Pylint only runs when someone runs it by hand.

**Possible follow-up:** add `load-plugins=pylint_django` and `django-settings-module=LearningPlatform.settings` under a `[MAIN]` section in `.pylintrc`, so the short `pylint LearningAPI` command understands Django without extra flags.
