---
name: notebook-edit
description: Edit an existing Jupyter notebook -- make changes, test execution, and convert back.
---

# notebook-edit

**Description:** Edit an existing Jupyter notebook -- make changes, test execution, and convert back.

---

## Shared Workflow

**Read [../notebook-create/NOTEBOOK_WORKFLOW.md](../notebook-create/NOTEBOOK_WORKFLOW.md) first.** It covers the two-environment architecture, jupytext setup, dependency validation, API key management, LLM defaults, error handling, and cleanup.

---

## Key Principle

**Make minimal, focused changes** unless the user explicitly requests broader modifications. Change only what is needed to fulfill the user's request. Don't refactor surrounding code, add features, or restructure the notebook beyond what was asked for.

---

## Workflow

### 1. Setup

```bash
PROJECT_DIR="$(pwd)"
TEMP_DIR="/tmp/notebook_test_$(date +%s)"
mkdir -p "$TEMP_DIR"
cd "$TEMP_DIR"
```

Ensure `.venv_tools` exists (see shared workflow for jupytext setup).

### 2. Convert Notebook to Python

```bash
$PROJECT_DIR/.venv_tools/bin/jupytext --to py $PROJECT_DIR/notebooks/<notebook_name>.ipynb --output <notebook_name>.py
```

### 3. Install Existing Dependencies

Extract and install the dependencies from the notebook:

```bash
grep -E "^# !pip install" <notebook_name>.py | sed 's/^# !pip install //' > requirements.txt
uv venv venv_notebook
uv pip install --python venv_notebook $(cat requirements.txt) 2>&1 | tee install_log.txt
```

If installation fails due to dependency conflicts, follow the dependency validation workflow in the shared workflow to resolve them.

### 4. Make Edits

Edit the `.py` file to implement the user's requested changes. Keep changes minimal and focused.

If you add or change dependencies:
- Re-validate using the full dependency validation workflow from the shared workflow
- Update the `!pip install` line in the `.py` file with the verified versions

### 5. Test Execution

```bash
# Load .env if it exists
if [ -f "$PROJECT_DIR/.env" ]; then
  export $(grep -v '^#' "$PROJECT_DIR/.env" | xargs)
fi
uv run --python venv_notebook/bin/python <notebook_name>.py
```

If errors occur, fix them in the `.py` file and re-test. See the shared workflow for error handling guidance.

### 6. Convert Back to Notebook

Once the code runs successfully:

```bash
$PROJECT_DIR/.venv_tools/bin/jupytext --to notebook <notebook_name>.py --output $PROJECT_DIR/notebooks/<notebook_name>.ipynb
```

### 7. Cleanup

```bash
cd "$PROJECT_DIR"
rm -rf "$TEMP_DIR"
```

---

## Quick Reference

```bash
# 1. Setup
PROJECT_DIR="$(pwd)"
TEMP_DIR="/tmp/notebook_test_$(date +%s)"
mkdir -p "$TEMP_DIR" && cd "$TEMP_DIR"

# 2. Convert to Python
$PROJECT_DIR/.venv_tools/bin/jupytext --to py $PROJECT_DIR/notebooks/<notebook_name>.ipynb --output <notebook_name>.py

# 3. Install dependencies
grep -E "^# !pip install" <notebook_name>.py | sed 's/^# !pip install //' > requirements.txt
uv venv venv_notebook
uv pip install --python venv_notebook $(cat requirements.txt)

# 4. Make edits to <notebook_name>.py (minimal changes)

# 5. If deps changed: re-validate (see shared workflow)

# 6. Test
if [ -f "$PROJECT_DIR/.env" ]; then export $(grep -v '^#' "$PROJECT_DIR/.env" | xargs); fi
uv run --python venv_notebook/bin/python <notebook_name>.py

# 7. Fix errors and re-test until passing

# 8. Convert back to notebook
$PROJECT_DIR/.venv_tools/bin/jupytext --to notebook <notebook_name>.py --output $PROJECT_DIR/notebooks/<notebook_name>.ipynb

# 9. Cleanup
cd "$PROJECT_DIR" && rm -rf "$TEMP_DIR"
```
