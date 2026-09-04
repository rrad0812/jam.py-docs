
# Jamdocs

## Changes for compatibility with newer Python versions (3.12+/3.14)

### 1. `distutils` removed from the standard library (Python 3.12+)

- Replaced `distutils.dir_util.copy_tree(...)` with `shutil.copytree(..., dirs_exist_ok=True)` in:

  - jam/admin/import_metadata.py
  - jam/adm_server.py
  - jam/bin/jam-project.py

- Also removed unused `from distutils import dir_utilimport` from item code

- `topics` (task.journals), which was saved in the database admin.sqlite(table SYS_ITEMS, column F_SERVER_MODULE), not in the project files.

### 2. `ast.Str` removed from `ast` module (Python 3.12+)

In built-in (third-party) werkzeug routing code `ast.Str(...)` replaced with
`ast.Constant(...)`, and access the attribute `.s` with `.value`:

- jam/third_party/werkzeug/routing.py

### 3. Missing dependency

Installed package `docutils` (used by item `topics` for `docutils.core.publish_doctree`).

## Remark

It was not necessary to install Python 3.7 — all changes are compatible with Python 3.8+ and work on the current version (Python 3.14).
