# Shared Notebook Workflow

This document contains the common workflow steps shared by the `notebook-create`, `notebook-edit`, and `notebook-test` skills.

---

## Two-Environment Architecture

This workflow uses **two separate virtual environments** for clean dependency isolation:

1. **Project Tools Environment** (`.venv_tools`)
   - Location: In the project directory
   - Purpose: Houses conversion tools only (jupytext)
   - Persistent: Created once, reused across workflows

2. **Notebook Runtime Environment** (`venv_notebook`)
   - Location: In the temporary directory (`/tmp/...`)
   - Purpose: Runs the notebook's code with its specific dependencies
   - Temporary: Created for each workflow run, deleted at the end

This separation prevents conflicts, keeps the project clean, and ensures reproducible testing.

---

## Temp Directory Setup

Create a temporary directory in `/tmp` for testing the notebook in isolation:

```bash
# Save original project path BEFORE changing directories
PROJECT_DIR="$(pwd)"
TEMP_DIR="/tmp/notebook_test_$(date +%s)"
mkdir -p "$TEMP_DIR"
cd "$TEMP_DIR"
```

**Why save PROJECT_DIR?** This prevents shell errors when cleaning up the temp directory later. If you delete the directory you're currently in, the shell encounters "cannot access parent directories" errors.

---

## jupytext Setup

Install jupytext in a lightweight project virtual environment (one-time setup):

```bash
# In the project directory (not temp directory)
uv venv --clear .venv_tools
uv pip install --python .venv_tools jupytext
```

### Converting .ipynb to .py

```bash
$PROJECT_DIR/.venv_tools/bin/jupytext --to py $PROJECT_DIR/notebooks/<notebook_name>.ipynb --output <notebook_name>.py
```

### Converting .py to .ipynb

```bash
$PROJECT_DIR/.venv_tools/bin/jupytext --to notebook <notebook_name>.py --output $PROJECT_DIR/notebooks/<notebook_name>.ipynb
```

**Why jupytext?** It handles bidirectional conversion (notebook <-> Python) consistently and reliably, unlike `nbconvert` which only works well in one direction. It preserves markdown cells formatted as `# %% [markdown]` comments.

### Cell Format in .py Files

- Code cells: separated by `# %%`
- Markdown cells: `# %% [markdown]` followed by `#`-prefixed lines

---

## Colab-Ready Notebooks

All notebooks must start with dependency installation cells for Google Colab compatibility:

```python
# Cell 1: Install dependencies with specific versions
!pip install package1==1.2.3 package2==4.5.6

# Import statements follow in the next cell
import package1
```

**Version management:**
- Always specify exact versions (e.g., `package==1.2.3`)
- Use the latest stable versions available on PyPI

**How jupytext handles `!pip install`:**
- jupytext converts `!pip install` cells to `# !pip install` comments in `.py` files
- These don't execute when running the Python script locally (which is correct)
- We extract these to install in the virtual environment instead

---

## Colab-Specific Code

Some notebooks contain code that only works in Google Colab and will fail or behave unexpectedly when run locally. This code must be temporarily removed before local execution.

### Patterns to Identify and Remove

| Pattern | Why it fails locally |
|---|---|
| `IPython.Application.instance().kernel.do_shutdown(True)` | Kills the local Python process immediately |
| `from google.colab import drive` / `drive.mount(...)` | Google Drive not available locally |
| `from google.colab import files` + `files.upload()` / `files.download()` | Interactive file dialogs not available locally |
| `from google.colab import output` (non-fallback usage) | Colab-only display API |
| `display(...)` calls where `display` is not explicitly imported | Jupyter kernels auto-inject `display` into the global namespace; plain Python scripts do not — raises `NameError` |
| Helper functions whose sole purpose is to call `display()` (e.g. `display_image()`), and their call sites | No visual rendering target locally; calling them raises `NameError` on `display` |

> **Note:** `from google.colab import userdata` is **not** in this list — it's already handled by the existing try/except fallback pattern and works correctly locally.
>
> **Note on `display`:** If a notebook imports `display` explicitly (`from IPython.display import display`) and uses it to render results that are meaningful to the script's logic, that is fine. Only remove `display` calls that serve purely visual/interactive purposes and would be no-ops or errors in a script context.

### Handling Strategy by Skill

**`notebook-test`**: After converting `.ipynb` → `.py`, scan the file for Colab-specific lines and use the Edit tool to remove them from the `.py` before executing. No restoration is needed (notebook-test never converts back to `.ipynb`). Note the removed lines in the final report.

**`notebook-edit` and `notebook-create`**: Before testing the `.py` script, scan for and remove Colab-specific lines using the Edit tool. **Remember which lines were removed and their exact position.** After the script runs successfully, restore those lines in the `.py` file before running `jupytext --to notebook` to convert back to `.ipynb`.

---

## Dependency Validation Workflow

Notebooks specify exact package versions in `!pip install` cells. These versions MUST be installable together without conflicts.

### Step 1: Define Packages

For **new notebooks**, create a `packages.txt` file with package names (no versions):

```bash
cat > packages.txt << 'EOF'
langchain
langchain-google-genai
pydantic
EOF
```

For **existing notebooks**, extract from the converted `.py` file:

```bash
grep -E "^# !pip install" <notebook_name>.py | sed 's/^# !pip install //' | sed 's/==\S*//g' | tr ' ' '\n' > packages.txt
```

### Step 2: Install Without Version Pins

```bash
uv venv venv_notebook
uv pip install --python venv_notebook $(cat packages.txt) 2>&1 | tee install_log.txt

if [ $? -eq 0 ]; then
    echo "Installation successful"
else
    echo "Installation failed - check install_log.txt for errors"
fi
```

If installation fails, there's a fundamental incompatibility. Adjust package choices.

### Step 3: Record Installed Versions

```bash
venv_notebook/bin/python -c "
import importlib.metadata

packages = open('packages.txt').read().split()
for pkg in packages:
    try:
        version = importlib.metadata.version(pkg)
        print(f'{pkg}=={version}')
    except importlib.metadata.PackageNotFoundError:
        pass
" | tee verified_versions.txt
```

### Step 4: Validate Pinned Versions

Test if the exact pinned versions can be installed together in a fresh environment:

```bash
rm -rf venv_notebook
uv venv venv_notebook
uv pip install --python venv_notebook $(cat verified_versions.txt) 2>&1 | tee install_validation.txt

if grep -qi "error\|conflict\|cannot install" install_validation.txt; then
    echo "ERROR: Pinned versions have conflicts!"
    exit 1
else
    echo "Pinned versions validated successfully"
fi
```

**Key principles:**
- **NEVER skip dependency validation** -- conflicts will break the notebook for users
- Install without pins first, then validate the resolved versions work when pinned
- Keep conversion tools (jupytext) separate from notebook runtime dependencies
- If you find conflicts, try different package combinations before giving up

---

## API Key Management

Notebooks use **Google Colab Secrets** (`google.colab.userdata`) to load API keys. This is the only pattern used in notebook code.

### Colab Secrets Pattern

```python
# %% [markdown]
# ## Setting Up API Access
#
# We'll configure access to the Gemini API. In Google Colab, your API key is loaded
# from Colab Secrets (accessible via the key icon in the sidebar). Add your
# `GOOGLE_API_KEY` there before running this notebook.

# %%
from google.colab import userdata

GOOGLE_API_KEY = userdata.get('GOOGLE_API_KEY')
print(f"API key loaded ({len(GOOGLE_API_KEY)} characters)")
```

### Local Testing Fallback

When running locally, `google.colab` is not available. The code should fall back to environment variables:

```python
import os

try:
    from google.colab import userdata
    GOOGLE_API_KEY = userdata.get('GOOGLE_API_KEY')
except ImportError:
    GOOGLE_API_KEY = os.environ['GOOGLE_API_KEY']
```

Set the API key before running (see `.env` auto-loading below), or manually:

```bash
export GOOGLE_API_KEY="your-key-here"
uv run --python venv_notebook/bin/python <notebook_name>.py
```

---

## .env Auto-Loading

If the project has a `.env` file in its root directory, **always load it before running any Python script**. This replaces manual `export` commands:

```bash
# Load environment variables from .env if it exists
if [ -f "$PROJECT_DIR/.env" ]; then
  export $(grep -v '^#' "$PROJECT_DIR/.env" | xargs)
fi
```

Run this **before** every `uv run` invocation. This ensures API keys and other secrets defined in `.env` are available to the notebook runtime without hardcoding them in commands.

**Notes:**
- `uv run` does not automatically forward all environment variables or load `.env` files — this step is required.
- The `.env` file is git-ignored and should never be committed.

### Framework-Specific Environment Variables

Some frameworks require additional environment variables for non-interactive execution:

```python
import os
# Disable CrewAI's interactive tracing prompt during testing
os.environ["CREWAI_TRACING_ENABLED"] = "false"
```

### Important Notes

- **Never hardcode API keys** in notebooks
- The only API key used in notebooks is `GOOGLE_API_KEY` (loaded via `google.colab.userdata`)
- For local testing, set env vars in the shell: `export GOOGLE_API_KEY="..."`

---

## Running Python Scripts

Execute the `.py` script using the notebook runtime environment:

```bash
uv run --python venv_notebook/bin/python <notebook_name>.py
```

For notebooks with API calls, set the required environment variables first:

```bash
# Load .env if it exists
if [ -f "$PROJECT_DIR/.env" ]; then
  export $(grep -v '^#' "$PROJECT_DIR/.env" | xargs)
fi
uv run --python venv_notebook/bin/python <notebook_name>.py
```

---

## LLM Defaults

- **Use the Gemini models already present in the notebook.** Do not change them unless the user asks.
- For **new notebooks**, use `gemini-2.5-flash` by default unless the user specifies otherwise.
- **Notebooks should be cheap to execute**: limit the number of API calls (e.g., 3-5 examples, not 50+), use reasonable token limits (500-1000 tokens is usually sufficient).
- Don't include commented-out expensive model configurations. Users who want to upgrade can change the model name themselves.

---

## Print Statements for Verification

During development, add labeled `print()` calls to verify key variables and intermediate results:

```python
# Good: labeled prints that show what's happening
print(f"Loaded {len(df)} rows from dataset")
print(f"Model response: {response.content[:100]}")
print(f"Filtered results: {len(results)} items matched")
```

You can add extra debugging prints freely in the `.py` script during development. Before converting back to `.ipynb`, remove any temporary debugging prints and keep only the prints that help the reader understand the notebook's output.

---

## Error Handling

### Module Not Found Errors

- Check that all packages from `!pip install` are installed in `venv_notebook`
- Verify versions match the notebook's requirements
- Re-run: `uv pip install --python venv_notebook <missing_package>==<version>`

### Shell "Cannot Access Parent Directories" Errors

- This happens if you're still in a deleted directory
- Always use the `$PROJECT_DIR` variable saved at the start
- Use absolute paths instead of relative paths
- If stuck, open a fresh shell session

### Import Errors or Test Failures (notebook-create and notebook-edit only)

**This section applies to `notebook-create` and `notebook-edit` only. The `notebook-test` skill is read-only and must NOT modify the `.py` script — it should report errors instead.**

- Identify the problematic code in the `.py` file
- Fix the errors directly in the `.py` file
- Re-run until the script executes successfully
- Do NOT convert back to notebook until all errors are resolved

### Stale `.venv_tools` ("Bad Interpreter" Errors)

- The `.venv_tools` shebang may point to a Python path that no longer exists
- Symptoms: `bad interpreter: No such file or directory` when running `.venv_tools/bin/jupytext`
- `uv venv .venv_tools` without `--clear` just warns and keeps the stale venv
- Fix: `uv venv --clear .venv_tools && uv pip install --python .venv_tools jupytext`

### General Debugging

- Read the full error message carefully
- Check that the virtual environment is activated/used correctly
- Ensure all paths use `$PROJECT_DIR` for reliability
- Test individual cells by copying them to a separate test script

---

## Cleanup

Remove the temporary directory when done:

```bash
# IMPORTANT: Navigate away from temp directory BEFORE deleting it
cd "$PROJECT_DIR"
rm -rf "$TEMP_DIR"
```

**Critical:** Always `cd` to a safe location before removing the temp directory. If you delete the directory you're currently in, the shell will encounter "cannot access parent directories" errors on subsequent commands.
