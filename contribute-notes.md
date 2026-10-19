# Conventions Notes

## JavaScript Linter (learn-ops-client)

### Configuration
Extends: react-app
Plugins active: 0

### Rules I found (at least 5)
Write each rule name, its severity, and what it enforces as a plain sentence.
- a variable was imported into `LearnOps.js` and not used rule no-unused-vars
- 

### First warning fixed
File and line: LearnOps.js line 8
Rule: no-unused-vars
Fix: delete the line 

## Python Linter (learn-ops-api)

### Configuration
Disabled rules and what each would flag: broad-except,arguments-differ,missing-function-docstring
- broad-except: flags except Exception: because catching all exceptions can hide bugs.

- arguments-differ: fires when a subclass overrides a method with a different parameter signature. Django ViewSets override methods this way by design, so the rule is disabled.

- missing-function-docstring: requires a docstring on every function. Disabled here because descriptive test method names already communicate intent.
Good names allowed: i, j, ex, pk
What pylint-django likely adds:

### First warning fixed
File and line: 0011_studentchapter_delete_studentproject.py lines 17-19 line too long
Rule and category: C0301 line too long
Fix: reformat 