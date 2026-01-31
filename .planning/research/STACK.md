# Stack Research: GSD-Jira CLI Integration

**Research Date:** 2026-01-31  
**Project:** Python CLI wrapper for syncing GSD artifacts with Jira Cloud  
**Constraint:** Must use `acli` (Atlassian CLI) — no custom REST API client  
**Target:** Jira Cloud with Advanced Roadmaps (Initiative issue type)

---

## Executive Summary

| Component | Recommendation | Version | Confidence |
|-----------|---------------|---------|------------|
| Python Runtime | Python 3.13 | 3.13.11 | HIGH |
| CLI Framework | Typer | 0.21.1 | HIGH |
| Config Format | TOML | stdlib (tomllib) | HIGH |
| State Management | SQLite via sqlite-utils | 3.39 | HIGH |
| Terminal Output | Rich | 14.3.1 | HIGH |
| External CLI | Atlassian CLI (acli) | Latest | HIGH |

---

## 1. Atlassian CLI (`acli`)

### Current Status
- **Maintained By:** Atlassian (official)
- **Support Policy:** Each version supported for 6 months after release
- **Last Documentation Update:** June 12, 2025
- **Confidence:** HIGH (verified via official Atlassian Developer docs)

### Installation

**macOS (Homebrew):**
```bash
brew tap atlassian/homebrew-acli
brew install acli
```

**macOS (Direct Download):**
```bash
# Intel
curl -L https://acli.atlassian.com/latest/macos/amd64/acli -o /usr/local/bin/acli
chmod +x /usr/local/bin/acli

# Apple Silicon
curl -L https://acli.atlassian.com/latest/macos/arm64/acli -o /usr/local/bin/acli
chmod +x /usr/local/bin/acli
```

**Linux:**
```bash
# Debian/Ubuntu (amd64)
curl -L https://acli.atlassian.com/latest/linux/amd64/acli.deb -o acli.deb
sudo dpkg -i acli.deb

# RHEL/Fedora (amd64)
curl -L https://acli.atlassian.com/latest/linux/amd64/acli.rpm -o acli.rpm
sudo rpm -i acli.rpm

# Tarball (any Linux)
curl -L https://acli.atlassian.com/latest/linux/amd64/acli.tar.gz | tar xz
sudo mv acli /usr/local/bin/
```

**Windows:**
Download from https://acli.atlassian.com/latest/windows/amd64/acli.exe

### Verified Capabilities

**Jira Workitem Commands (core for GSD sync):**
| Command | Purpose | GSD Use Case |
|---------|---------|--------------|
| `jira workitem create` | Create issues | Create Initiative/Epic/Story from phases |
| `jira workitem create-bulk` | Bulk create issues | Batch sync new phases |
| `jira workitem edit` | Update issues | Sync status/field changes |
| `jira workitem view` | Retrieve issue details | Verify sync state |
| `jira workitem search` | JQL search | Find existing issues |
| `jira workitem link` | Link issues | Connect Initiative→Epic→Story |
| `jira workitem transition` | Change status | Sync workflow state |
| `jira workitem assign` | Set assignee | Sync ownership |
| `jira workitem comment-create` | Add comments | Sync notes/updates |
| `jira workitem clone` | Duplicate issues | Template-based creation |
| `jira workitem delete` | Remove issues | Cleanup (use cautiously) |

**Supporting Commands:**
| Command | Purpose |
|---------|---------|
| `jira auth login` | Authenticate (OAuth or API token) |
| `jira auth status` | Verify authentication |
| `jira project list` | List available projects |
| `jira board list` | List boards |
| `jira sprint list` | List sprints |
| `jira field list` | List custom fields |

### Authentication via `acli`

**API Token (Recommended for CLI automation):**
```bash
# Interactive (prompts for token)
acli jira auth login --site "mysite.atlassian.net" --email "user@example.com" --token

# Non-interactive (pipe token)
echo "$JIRA_API_TOKEN" | acli jira auth login --site "$JIRA_SITE" --email "$JIRA_EMAIL" --token

# Verify
acli jira auth status
```

**OAuth (Browser-based):**
```bash
acli jira auth login --web
```

### Output Formats
- Default: Human-readable tables
- JSON: Use `--output json` for machine parsing
- **Critical for Python wrapper:** Always use `--output json` and parse with `json.loads()`

### Why `acli` Over Direct API
1. **Official support:** Atlassian maintains it; matches their API changes
2. **Auth handling:** Manages token refresh, OAuth flows internally
3. **Error messages:** Consistent, user-friendly error handling
4. **No REST boilerplate:** Avoids building/maintaining HTTP client code
5. **Constraint compliance:** Project requirement specifies `acli` usage

### Limitations
- No direct Python bindings (must shell out via `subprocess`)
- Output parsing required for automation
- Limited Advanced Roadmaps-specific commands (uses standard issue APIs)

---

## 2. Python Runtime

### Recommendation: Python 3.13
- **Version:** 3.13.11 (released December 5, 2025)
- **Status:** Bugfix maintenance phase
- **Support End:** October 2029
- **Confidence:** HIGH

### Why 3.13 Over Alternatives

| Version | Status | Recommendation |
|---------|--------|----------------|
| 3.14 | Current feature release | Too new; ecosystem catching up |
| **3.13** | **Bugfix phase** | **Best balance of features + stability** |
| 3.12 | Security-only | Good fallback if 3.13 issues |
| 3.11 | Security-only | Acceptable minimum |
| 3.10 | EOL October 2026 | Avoid for new projects |

**Key 3.13 Features Used:**
- `tomllib` in stdlib (TOML parsing, no external dep)
- Improved error messages for debugging
- Better type hint support for Typer

### Installation
```bash
# Ubuntu/Debian
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt update
sudo apt install python3.13 python3.13-venv

# macOS (Homebrew)
brew install python@3.13

# pyenv (any platform)
pyenv install 3.13.11
pyenv local 3.13.11
```

---

## 3. CLI Framework

### Recommendation: Typer
- **Version:** 0.21.1 (released January 6, 2025)
- **Confidence:** HIGH

### Why Typer Over Alternatives

| Framework | External Dep | Type Hints | Learning Curve | Shell Completion |
|-----------|--------------|------------|----------------|------------------|
| argparse | No | Manual | Medium | No |
| Click | Yes | Manual | Medium | Yes |
| **Typer** | **Yes** | **Automatic** | **Low** | **Yes** |

**Typer Advantages:**
1. **Type hints = CLI args:** Define `def sync(project: str, dry_run: bool = False)` and get automatic arg parsing
2. **Rich integration:** Built-in support for Rich terminal formatting
3. **Shell completion:** Auto-generates bash/zsh/fish/PowerShell completions
4. **Subcommand trees:** Natural support for `gsd-jira sync`, `gsd-jira status`, etc.
5. **Testing utilities:** `CliRunner` for testing CLI commands
6. **FastAPI lineage:** Same design philosophy as FastAPI (explicit, type-safe)

### Installation
```bash
pip install typer
# or
uv add typer
```

**Note:** `typer` package includes `rich` and `shellingham` as standard dependencies.

### Example Structure for GSD-Jira CLI
```python
import typer
from typing import Optional

app = typer.Typer(help="Sync GSD planning artifacts with Jira Cloud")

@app.command()
def sync(
    project: str = typer.Argument(..., help="GSD project directory"),
    dry_run: bool = typer.Option(False, "--dry-run", "-n", help="Preview changes"),
    initiative: Optional[str] = typer.Option(None, help="Jira Initiative key"),
):
    """Sync GSD phases to Jira issues."""
    ...

@app.command()
def status(
    project: str = typer.Argument(...),
    verbose: bool = typer.Option(False, "-v", "--verbose"),
):
    """Show sync status between GSD and Jira."""
    ...

@app.command()
def init(
    project: str = typer.Argument(...),
    jira_project: str = typer.Option(..., prompt=True),
):
    """Initialize Jira sync for a GSD project."""
    ...

if __name__ == "__main__":
    app()
```

### Alternatives Considered

**Click (v8.1.x):**
- Mature, battle-tested
- Typer is built on Click anyway
- Requires more boilerplate for type conversion
- Choose if team already knows Click

**argparse:**
- No external dependencies
- More verbose, manual type handling
- Choose only if zero-dependency is critical

---

## 4. Configuration Format

### Recommendation: TOML
- **Parser:** `tomllib` (Python 3.11+ stdlib)
- **Writer:** `tomli-w` (for writing TOML files)
- **Confidence:** HIGH

### Why TOML Over Alternatives

| Format | Human-Editable | Comments | Python Stdlib | Best For |
|--------|----------------|----------|---------------|----------|
| **TOML** | **Excellent** | **Yes** | **Yes (3.11+)** | **Config files** |
| YAML | Good | Yes | No | DevOps/K8s |
| JSON | Poor | No | Yes | APIs/data exchange |

**TOML Advantages for CLI Config:**
1. **Python standard:** `pyproject.toml` is Python's config standard
2. **Clean syntax:** No quotes for simple strings, inline tables
3. **Comments:** Users can document their config
4. **Error messages:** Better than JSON parse errors
5. **Type clarity:** Explicit `"string"` vs `123` vs `true`

### Config File Example: `.gsd-jira.toml`
```toml
# GSD-Jira sync configuration

[jira]
site = "mycompany.atlassian.net"
project_key = "GSD"
default_issue_type = "Story"

[jira.hierarchy]
# Map GSD concepts to Jira issue types
milestone = "Initiative"  # Requires Advanced Roadmaps
phase = "Epic"
task = "Story"

[sync]
# What to sync
include_completed = false
sync_comments = true
bidirectional = false  # GSD → Jira only for now

[auth]
# Auth method: "env" (use JIRA_* env vars) or "keyring"
method = "env"
```

### Reading Config
```python
import tomllib
from pathlib import Path

def load_config(project_path: Path) -> dict:
    config_path = project_path / ".gsd-jira.toml"
    if not config_path.exists():
        return {}  # Use defaults
    
    with open(config_path, "rb") as f:
        return tomllib.load(f)
```

### Writing Config (for `init` command)
```bash
pip install tomli-w
# or
uv add tomli-w
```

```python
import tomli_w

def write_config(project_path: Path, config: dict):
    config_path = project_path / ".gsd-jira.toml"
    with open(config_path, "wb") as f:
        tomli_w.dump(config, f)
```

---

## 5. Sync State Management

### Recommendation: SQLite via sqlite-utils
- **Version:** sqlite-utils 3.39 (released November 24, 2025)
- **Confidence:** HIGH

### Why SQLite Over Alternatives

| Approach | Queryable | Concurrent Safe | Portable | Complexity |
|----------|-----------|-----------------|----------|------------|
| JSON file | No | No | Yes | Low |
| **SQLite** | **Yes** | **Yes** | **Yes** | **Low-Medium** |
| PostgreSQL | Yes | Yes | No | High |

**SQLite Advantages for Sync State:**
1. **Single file:** `.gsd-jira.db` lives in project directory
2. **Atomic writes:** No partial state on crash
3. **Queryable:** "What phases haven't synced?" = SQL query
4. **sqlite-utils:** CLI + Python library in one package
5. **No server:** Zero infrastructure

### State Schema
```sql
-- Track GSD → Jira mappings
CREATE TABLE sync_mapping (
    id INTEGER PRIMARY KEY,
    gsd_type TEXT NOT NULL,        -- 'milestone', 'phase', 'task'
    gsd_id TEXT NOT NULL,          -- GSD identifier (e.g., phase filename)
    jira_key TEXT,                 -- Jira issue key (e.g., GSD-123)
    jira_id TEXT,                  -- Jira issue ID
    last_sync_at TEXT,             -- ISO timestamp
    last_sync_hash TEXT,           -- Hash of GSD content at sync time
    UNIQUE(gsd_type, gsd_id)
);

-- Track sync operations
CREATE TABLE sync_log (
    id INTEGER PRIMARY KEY,
    timestamp TEXT NOT NULL,
    operation TEXT NOT NULL,       -- 'create', 'update', 'skip'
    gsd_type TEXT NOT NULL,
    gsd_id TEXT NOT NULL,
    jira_key TEXT,
    details TEXT                   -- JSON blob for debugging
);
```

### Usage with sqlite-utils
```python
import sqlite_utils
from pathlib import Path

def get_db(project_path: Path) -> sqlite_utils.Database:
    db_path = project_path / ".gsd-jira.db"
    db = sqlite_utils.Database(db_path)
    
    # Ensure tables exist
    if "sync_mapping" not in db.table_names():
        db["sync_mapping"].create({
            "id": int,
            "gsd_type": str,
            "gsd_id": str,
            "jira_key": str,
            "jira_id": str,
            "last_sync_at": str,
            "last_sync_hash": str,
        }, pk="id")
        db["sync_mapping"].create_index(["gsd_type", "gsd_id"], unique=True)
    
    return db

def get_jira_key(db: sqlite_utils.Database, gsd_type: str, gsd_id: str) -> str | None:
    """Look up existing Jira key for a GSD artifact."""
    rows = list(db["sync_mapping"].rows_where(
        "gsd_type = ? AND gsd_id = ?", 
        [gsd_type, gsd_id]
    ))
    return rows[0]["jira_key"] if rows else None
```

### Installation
```bash
pip install sqlite-utils
# or
uv add sqlite-utils
```

---

## 6. Terminal Output

### Recommendation: Rich
- **Version:** 14.3.1 (released January 24, 2026)
- **Confidence:** HIGH
- **Note:** Included with Typer by default

### Why Rich
1. **Typer integration:** Already a Typer dependency
2. **Progress bars:** Essential for bulk sync operations
3. **Tables:** Display sync status clearly
4. **Syntax highlighting:** Show Jira API responses
5. **Panels/Trees:** Visualize GSD→Jira hierarchy

### Example Usage
```python
from rich.console import Console
from rich.table import Table
from rich.progress import Progress

console = Console()

def show_sync_status(mappings: list[dict]):
    table = Table(title="GSD-Jira Sync Status")
    table.add_column("GSD Type", style="cyan")
    table.add_column("GSD ID", style="green")
    table.add_column("Jira Key", style="yellow")
    table.add_column("Status", style="magenta")
    
    for m in mappings:
        table.add_row(
            m["gsd_type"],
            m["gsd_id"],
            m["jira_key"] or "-",
            "✓ Synced" if m["jira_key"] else "⊘ Pending"
        )
    
    console.print(table)

def sync_with_progress(items: list):
    with Progress() as progress:
        task = progress.add_task("Syncing to Jira...", total=len(items))
        for item in items:
            # ... sync logic ...
            progress.advance(task)
```

---

## 7. Complete Dependencies

### `pyproject.toml`
```toml
[project]
name = "gsd-jira"
version = "0.1.0"
description = "Sync GSD planning artifacts with Jira Cloud"
requires-python = ">=3.13"
dependencies = [
    "typer>=0.21.1",          # CLI framework (includes rich, shellingham)
    "sqlite-utils>=3.39",     # State management
    "tomli-w>=1.0.0",         # TOML writing (reading via stdlib)
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-cov>=4.0",
    "ruff>=0.4",
    "mypy>=1.10",
]

[project.scripts]
gsd-jira = "gsd_jira.cli:app"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.ruff]
target-version = "py313"
line-length = 100

[tool.mypy]
python_version = "3.13"
strict = true
```

### Installation Commands
```bash
# Create project
mkdir gsd-jira && cd gsd-jira
uv init

# Add dependencies
uv add typer sqlite-utils tomli-w

# Add dev dependencies
uv add --dev pytest pytest-cov ruff mypy

# Verify acli is installed
acli --version
```

---

## 8. Environment Variables

```bash
# Required for acli authentication
export JIRA_SITE="mycompany.atlassian.net"
export JIRA_EMAIL="user@example.com"
export JIRA_API_TOKEN="your-api-token-here"

# Optional: default project
export JIRA_PROJECT_KEY="GSD"
```

---

## 9. Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     gsd-jira CLI                            │
│  (Typer + Rich)                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ Config       │  │ State        │  │ GSD Parser       │  │
│  │ (TOML)       │  │ (SQLite)     │  │ (Markdown)       │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
│         │                 │                   │             │
│         └─────────────────┴───────────────────┘             │
│                           │                                 │
│                    ┌──────▼──────┐                          │
│                    │ Sync Engine │                          │
│                    └──────┬──────┘                          │
│                           │                                 │
│                    ┌──────▼──────┐                          │
│                    │ acli        │                          │
│                    │ subprocess  │                          │
│                    └──────┬──────┘                          │
│                           │                                 │
└───────────────────────────┼─────────────────────────────────┘
                            │
                    ┌───────▼───────┐
                    │  Jira Cloud   │
                    │  (via acli)   │
                    └───────────────┘
```

---

## 10. Confidence Summary

| Decision | Confidence | Rationale |
|----------|------------|-----------|
| acli for Jira ops | HIGH | Official Atlassian tool; project constraint |
| Python 3.13 | HIGH | Current stable; stdlib TOML; long support |
| Typer for CLI | HIGH | Type hints; Rich included; active development |
| TOML for config | HIGH | Python standard; human-editable; stdlib parser |
| SQLite for state | HIGH | Single-file; queryable; sqlite-utils tooling |
| Rich for output | HIGH | Typer dependency; progress bars essential |

---

## 11. Alternatives Rejected

### Direct Jira REST API (via requests/httpx)
- **Rejected because:** Project constraint requires `acli`
- **Would choose if:** Need fine-grained control, async operations, or custom auth

### atlassian-python-api Library
- **Rejected because:** Project constraint requires `acli`
- **Would choose if:** Building server-side integration (FastAPI backend)

### JSON State File
- **Rejected because:** Not queryable; concurrent write issues
- **Would choose if:** Zero external dependencies required

### Click over Typer
- **Rejected because:** More boilerplate for same functionality
- **Would choose if:** Team has existing Click expertise

### YAML for Config
- **Rejected because:** Not Python stdlib; indentation-sensitive
- **Would choose if:** DevOps/infrastructure context

---

## 12. Research Sources

1. Atlassian CLI Installation - https://developer.atlassian.com/cloud/acli/guides/install-acli/ (verified Jan 2026)
2. Atlassian CLI Commands - https://developer.atlassian.com/cloud/acli/reference/commands/ (verified Jan 2026)
3. Typer Documentation - https://typer.tiangolo.com/ (verified Jan 2026)
4. Python Release Schedule - https://www.python.org/downloads/ (verified Jan 2026)
5. sqlite-utils Documentation - https://sqlite-utils.datasette.io/ (verified Jan 2026)
6. Rich Documentation - https://rich.readthedocs.io/ (verified Jan 2026)
7. TOML Specification - https://toml.io/en/ (verified Jan 2026)

---

*Research completed: 2026-01-31*  
*Researcher: GSD Project Researcher (Stack Dimension)*  
*Next: Review with team, then proceed to ARCHITECTURE.md*
