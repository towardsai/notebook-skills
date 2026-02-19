---
name: notebook-test
description: Test an existing Jupyter notebook by executing it in an isolated environment and reporting results. Read-only -- does not modify the notebook.
---

# notebook-test

**Description:** Test an existing Jupyter notebook by executing it in an isolated environment and reporting results. Read-only -- does not modify the notebook.

---

## Shared Workflow

**Read [../shared/NOTEBOOK_WORKFLOW.md](../shared/NOTEBOOK_WORKFLOW.md) first.** It covers the two-environment architecture, jupytext setup, dependency validation, API key management, LLM defaults, error handling, and cleanup.

---

## Key Principle

**This skill is read-only.** It tests whether a notebook executes without errors, but does not modify the original `.ipynb` file. There is no conversion back to `.ipynb`.

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

### 3. Install Dependencies

Extract and install the dependencies from the notebook:

```bash
grep -E "^# !pip install" <notebook_name>.py | sed 's/^# !pip install //' > requirements.txt
uv venv venv_notebook
uv pip install --python venv_notebook $(cat requirements.txt) 2>&1 | tee install_log.txt
```

If installation fails, note the dependency conflicts for the error report.

### 4. Execute the Script

```bash
export GOOGLE_API_KEY="your-key-here"  # if needed
uv run --python venv_notebook/bin/python <notebook_name>.py 2>&1 | tee execution_log.txt
```

### 5. Report Results

#### If execution succeeds:

Report that the notebook executed successfully. Include any relevant output summaries.

#### If execution fails:

Provide a structured error report with:

1. **Error message**: The exact error and traceback from the execution log
2. **Location**: Which cell/section of the notebook failed (identify the relevant code)
3. **Possible causes**: Explain what likely went wrong, e.g.:
   - Missing or incompatible dependency versions
   - API key not set or expired
   - Network-dependent code failing offline
   - Hardcoded paths or environment-specific assumptions
   - Breaking changes in upstream libraries
   - Data files or resources not available
4. **Suggested fixes**: Concrete steps to resolve each possible cause, e.g.:
   - "Add `package==X.Y.Z` to the `!pip install` cell"
   - "Set the `GOOGLE_API_KEY` environment variable before running"
   - "Update the import from `old_module` to `new_module` (changed in v2.0)"
   - "Replace the hardcoded path with a relative path or URL"

### 6. Cleanup

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
uv pip install --python venv_notebook $(cat requirements.txt) 2>&1 | tee install_log.txt

# 4. Execute
export GOOGLE_API_KEY="your-key-here"  # if needed
uv run --python venv_notebook/bin/python <notebook_name>.py 2>&1 | tee execution_log.txt

# 5. Report results (success or structured error report)

# 6. Cleanup
cd "$PROJECT_DIR" && rm -rf "$TEMP_DIR"
```
