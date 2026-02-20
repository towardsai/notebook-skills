# Notebook Updater Skills

Claude Code skills for creating, editing, and testing Jupyter notebooks in isolated environments.

## Skills

| Skill | Command | Description |
|-------|---------|-------------|
| **notebook-create** | `/notebook-create` | Create a new notebook from scratch with validated dependencies |
| **notebook-edit** | `/notebook-edit` | Edit an existing notebook with minimal, focused changes |
| **notebook-test** | `/notebook-test` | Test a notebook's execution without modifying it |

All three skills share a common workflow defined in `.claude/skills/notebook-create/NOTEBOOK_WORKFLOW.md`.

## Installation

These skills are installed at the **project/workspace level** (not user level) using [openskills](https://github.com/numman-ali/openskills).

From the root of your target project:

```bash
npx openskills install towardsai/notebook-skills
```

This installs the skills into `.claude/skills/` in your project directory. They are immediately available in Claude Code — no restart needed.

### Manual installation

If you prefer not to use openskills, clone the repo and copy the skills directory:

```bash
git clone https://github.com/your-org/your-skills.git /tmp/notebook-skills
cp -r /tmp/notebook-skills/.claude/skills/ .claude/skills/
rm -rf /tmp/notebook-skills
```

## Prerequisites

- [uv](https://docs.astral.sh/uv/) -- fast Python package installer
- [Claude Code](https://claude.ai/code) CLI

The skills will set up jupytext and other tools automatically on first use.

## How It Works

1. A temporary directory is created in `/tmp` for isolated execution
2. Notebooks are converted to `.py` scripts using [jupytext](https://github.com/mwouts/jupytext)
3. Dependencies (from `!pip install` cells) are extracted and installed in a temporary venv
4. If a `.env` file exists in the project root, its variables are auto-loaded before execution
5. The script is executed to verify correctness
6. On success, the script is converted back to `.ipynb` (create/edit only — test is read-only)
7. All temporary files are cleaned up

### Environment variables

Place a `.env` file in the project root with your API keys:

```
GOOGLE_API_KEY=your-key-here
OPENAI_API_KEY=your-key-here
```

The skills automatically load it before running notebooks. No manual `export` needed.

### Library management

Notebooks are Colab-ready: they start with `!pip install package==1.2.3` cells with pinned versions. The skills validate that all pinned versions install together without conflicts before writing the notebook.

## Example Prompts

**Create a notebook:**

```
/notebook-create Create a notebook that uses LangChain with Gemini to summarize a webpage
```

**Provide style references:**

```
/notebook-create Create a notebook about RAG with ChromaDB.
Use notebooks/langchain_basics.ipynb as a style reference.
```

**Point to documentation or web resources:**

```
/notebook-create Create a notebook showing LangGraph agents.
Look up the latest LangGraph docs to use the current API — the library has changed recently.
```

```
/notebook-edit Update notebooks/rag_pipeline.ipynb.
Check https://docs.trychroma.com/migration for breaking changes in ChromaDB v0.5 and update the code accordingly.
```

**Test a notebook:**

```
/notebook-test Run notebooks/langchain_basics.ipynb and report any errors
```
