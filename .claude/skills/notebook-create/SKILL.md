---
name: notebook-create
description: Create a new Jupyter notebook from scratch, with tested dependencies and verified execution.
---

# notebook-create

**Description:** Create a new Jupyter notebook from scratch, with tested dependencies and verified execution.

---

## Shared Workflow

**Read [./NOTEBOOK_WORKFLOW.md](./NOTEBOOK_WORKFLOW.md) first.** It covers the two-environment architecture, jupytext setup, dependency validation, API key management, LLM defaults, error handling, and cleanup.

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

### 2. Define Required Packages

Create a `packages.txt` file with the packages your notebook will need (no versions yet):

```bash
cat > packages.txt << 'EOF'
langchain
langchain-google-genai
pydantic
EOF
```

Choose packages based on the notebook's topic. Do NOT pin versions yet.

### 3. Validate Dependencies

Follow the full dependency validation workflow from the shared workflow:
1. Install without pins to find compatible versions
2. Record the resolved versions
3. Validate the pinned versions in a fresh venv

### 4. Write the Notebook .py File

Now that you have verified pinned versions, write the full `<notebook_name>.py` file in the temp directory:

1. Use the Write tool to create the file
2. Use the exact versions from `verified_versions.txt` in the `!pip install` line
3. Structure with proper cell markers:

```python
# %% [markdown]
# # Notebook Title
# Description of what this notebook does

# %%
# !pip install langchain==1.2.9 langchain-google-genai==2.0.10 pydantic==2.12.5

# %%
import langchain
import sys

print(f"Python version: {sys.version}")
print(f"langchain version: {langchain.__version__}")

# Add your code here...
```

4. Include markdown documentation cells (`# %% [markdown]`) explaining the code
5. If the user has provided style references or templates, follow them

### 5. Test Execution

Before running, read the `.py` file and use the Edit tool to remove any Colab-specific lines (see the shared workflow's "Colab-Specific Code" section for patterns). Remember what you removed and where — you will need to restore them in step 6.

```bash
# Load .env if it exists
if [ -f "$PROJECT_DIR/.env" ]; then
  export $(grep -v '^#' "$PROJECT_DIR/.env" | xargs)
fi
uv run --python venv_notebook/bin/python <notebook_name>.py
```

If errors occur, fix them in the `.py` file and re-test. See the shared workflow for error handling guidance.

### 6. Convert to Notebook

Before converting, restore any Colab-specific lines that were removed in step 5. Use the Edit tool to add them back in their original positions in the `.py` file.

Once restored, determine the output path (project root by default, auto-incrementing if a file already exists) and convert:

```bash
# Save at project root; auto-increment version suffix if file already exists
BASE="$PROJECT_DIR/<notebook_name>"
OUTPUT="${BASE}.ipynb"
VERSION=2
while [ -f "$OUTPUT" ]; do
  OUTPUT="${BASE}_v${VERSION}.ipynb"
  VERSION=$((VERSION + 1))
done
$PROJECT_DIR/.venv_tools/bin/jupytext --to notebook <notebook_name>.py --output "$OUTPUT"
echo "Saved as: $OUTPUT"
```

If the user specifies a different output path or filename, use that instead.

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

# 2. Define packages (no versions)
cat > packages.txt << 'EOF'
package1
package2
EOF

# 3. Find compatible versions
uv venv venv_notebook
uv pip install --python venv_notebook $(cat packages.txt)

# 4. Record resolved versions
venv_notebook/bin/python -c "
import importlib.metadata
packages = open('packages.txt').read().split()
for pkg in packages:
    try: print(f'{pkg}=={importlib.metadata.version(pkg)}')
    except: pass
" > verified_versions.txt
cat verified_versions.txt

# 5. Validate pinned versions
rm -rf venv_notebook && uv venv venv_notebook
uv pip install --python venv_notebook $(cat verified_versions.txt)

# 6. Write <notebook_name>.py (use Write tool, with verified versions)

# 7. Remove Colab-specific lines from <notebook_name>.py (Edit tool)
#    (IPython kernel shutdown, drive.mount, files.upload, etc. — see shared workflow)
#    Remember removed lines and their positions for restoration in step 9.

# 8. Test
if [ -f "$PROJECT_DIR/.env" ]; then export $(grep -v '^#' "$PROJECT_DIR/.env" | xargs); fi
uv run --python venv_notebook/bin/python <notebook_name>.py

# 9. Fix errors and re-test until passing

# 10. Restore removed Colab-specific lines in <notebook_name>.py (Edit tool)

# 11. Determine output path (project root, auto-increment if exists) and convert
BASE="$PROJECT_DIR/<notebook_name>"; OUTPUT="${BASE}.ipynb"; VERSION=2
while [ -f "$OUTPUT" ]; do OUTPUT="${BASE}_v${VERSION}.ipynb"; VERSION=$((VERSION+1)); done
$PROJECT_DIR/.venv_tools/bin/jupytext --to notebook <notebook_name>.py --output "$OUTPUT"
echo "Saved as: $OUTPUT"

# 12. Cleanup
cd "$PROJECT_DIR" && rm -rf "$TEMP_DIR"
```
