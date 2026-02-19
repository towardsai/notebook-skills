# Notebook Updater Skills

Claude Code skills for creating, editing, and testing Jupyter notebooks in isolated environments.

## Skills

| Skill | Command | Description |
|-------|---------|-------------|
| **notebook-create** | `/notebook-create` | Create a new notebook from scratch with validated dependencies |
| **notebook-edit** | `/notebook-edit` | Edit an existing notebook with minimal, focused changes |
| **notebook-test** | `/notebook-test` | Test a notebook's execution without modifying it |

All three skills share a common workflow (dependency validation, jupytext conversion, API key management, etc.) defined in `.claude/skills/shared/NOTEBOOK_WORKFLOW.md`.

## Installation

These skills are installed at the **project/workspace level** (not user level) using [openskills](https://github.com/numman-ali/openskills).

From the root of your target project:

```bash
npx openskills install your-org/your-skills
```

> Replace `your-org/your-skills` with the actual GitHub org/repo name where this project is published.

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

1. Notebooks are converted to `.py` scripts using jupytext
2. Dependencies are validated in an isolated virtual environment (in `/tmp`)
3. The script is executed to verify correctness
4. On success, the script is converted back to `.ipynb` (create/edit only)
5. All temporary files are cleaned up

See `.claude/skills/shared/NOTEBOOK_WORKFLOW.md` for the full shared workflow details.
