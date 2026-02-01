# Phase 1: Foundation - Research

**Researched:** 2026-02-01
**Domain:** CLI tooling, YAML configuration, acli wrapper patterns
**Confidence:** HIGH

## Summary

Phase 1 establishes the foundation for GSD-Jira integration: configuration schema, CLI scaffold, and acli wrapper patterns. This research investigated the official Atlassian CLI (`acli`) commands needed for initialization, YAML configuration best practices for Node.js, and patterns for wrapping external CLI tools with robust error handling.

The official Atlassian CLI (`acli`) provides all commands needed for this phase: authentication verification via `acli jira auth status`, project listing via `acli jira project list`, and board discovery via `acli jira board search`. The `yaml` npm package (eemeli/yaml) is the recommended choice for YAML handling as it supports YAML 1.2, preserves comments, and has excellent Node.js integration.

**Primary recommendation:** Use the official Atlassian CLI (not the older Bob Swift ACLI) with JSON output mode for all commands, wrapped in a reusable acli wrapper module that translates exit codes and stderr into friendly error messages.

## Standard Stack

The established libraries/tools for this domain:

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| acli | latest | Atlassian CLI for Jira Cloud | Official Atlassian tool, handles auth, supports all needed operations |
| yaml | ^2.x | YAML parsing/serialization | Supports YAML 1.2, preserves comments, active maintenance, High reputation |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| fs/promises | Node.js built-in | Async file operations | Reading/writing config files |
| child_process | Node.js built-in | Spawning acli commands | Wrapper implementation |
| path | Node.js built-in | Path manipulation | Config file paths |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| yaml | js-yaml | js-yaml is simpler but doesn't preserve comments; yaml better for config files users may edit |
| child_process.spawn | execa | execa adds dependency; built-in is sufficient for this use case |

**Installation:**
```bash
npm install yaml
# acli must be installed separately via Atlassian's installer
```

## Architecture Patterns

### Recommended Project Structure
```
commands/gsd/
├── jira-init.md          # GSD command definition

.planning/jira/           # Created by gsd:jira-init
├── config.yaml           # Project configuration
└── mapping.yaml          # Entity type mapping

lib/ (or inline in command) # Implementation patterns
├── acli.js               # acli wrapper module
├── jira-config.js        # Config file management
└── jira-errors.js        # Error translation
```

### Pattern 1: acli Wrapper Module
**What:** Centralized module for executing acli commands with consistent error handling
**When to use:** Every acli invocation throughout the integration
**Example:**
```javascript
// Source: Node.js child_process docs + Atlassian CLI patterns
const { spawn } = require('child_process');

function runAcli(args, options = {}) {
  return new Promise((resolve, reject) => {
    const child = spawn('acli', args, { ...options });
    let stdout = '';
    let stderr = '';
    
    child.stdout.on('data', (data) => { stdout += data; });
    child.stderr.on('data', (data) => { stderr += data; });
    
    child.on('error', (error) => {
      // Spawn failed (e.g., ENOENT - acli not found)
      reject(new AcliError('acli_not_found', 'acli command not found. Install from https://developer.atlassian.com/cloud/acli/guides/install-acli/'));
    });
    
    child.on('close', (code) => {
      if (code === 0) {
        resolve(options.json ? JSON.parse(stdout) : stdout);
      } else {
        reject(translateAcliError(code, stderr));
      }
    });
  });
}

// Always use --json for parseable output
async function getAuthStatus() {
  return runAcli(['jira', 'auth', 'status'], { json: false });
}

async function listProjects() {
  return runAcli(['jira', 'project', 'list', '--json'], { json: true });
}

async function searchBoards(projectKey) {
  return runAcli(['jira', 'board', 'search', '--project', projectKey, '--json'], { json: true });
}
```

### Pattern 2: YAML Config File Management
**What:** Read/write YAML config files preserving comments and formatting
**When to use:** Managing mapping.yaml and config.yaml
**Example:**
```javascript
// Source: eemeli/yaml documentation (Context7)
const { parse, stringify, parseDocument } = require('yaml');
const fs = require('fs/promises');

// Simple read (loses comments)
async function readConfig(path) {
  const content = await fs.readFile(path, 'utf8');
  return parse(content);
}

// Preserve-comments read/write
async function updateConfig(path, updater) {
  const content = await fs.readFile(path, 'utf8');
  const doc = parseDocument(content);
  updater(doc);
  await fs.writeFile(path, doc.toString());
}

// Write with comments
async function writeConfigWithComments(path, data, headerComment) {
  const doc = new Document(data);
  doc.commentBefore = headerComment;
  await fs.writeFile(path, doc.toString());
}
```

### Pattern 3: GSD Command Structure
**What:** Follow existing GSD command patterns from commands/gsd/*.md
**When to use:** Creating gsd:jira-init command
**Example:**
```markdown
---
name: gsd:jira-init
description: Configure Jira connection for GSD integration
allowed-tools:
  - Read
  - Write
  - Bash
  - AskUserQuestion
---

<objective>
Configure Jira connection by verifying acli authentication and creating configuration files.
</objective>

<process>
## 1. Verify acli Installation
## 2. Verify Authentication
## 3. Select Project
## 4. Select Board
## 5. Create Configuration Files
## 6. Show Summary
</process>

<success_criteria>
- [ ] acli authenticated
- [ ] .planning/jira/config.yaml created
- [ ] .planning/jira/mapping.yaml created
</success_criteria>
```

### Anti-Patterns to Avoid
- **Direct REST API calls:** Use acli only, not axios/fetch to Jira REST API
- **Synchronous file operations:** Use fs/promises, not fs.readFileSync
- **Ignoring acli exit codes:** Always check exit code and stderr
- **Hardcoding Jira URLs:** Extract from acli auth status or user input
- **Storing credentials in config files:** Rely on acli's secure token storage

## Don't Hand-Roll

Problems that look simple but have existing solutions:

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Jira authentication | Custom OAuth/token management | acli auth | acli handles token refresh, secure storage, multiple accounts |
| YAML with comments | String templates | yaml package | Edge cases with multiline, escaping, indentation |
| Config file watching | fs.watch + debounce | N/A for Phase 1 | Not needed; config read at command start |
| Project/board discovery | Custom Jira REST calls | acli project list, board search | API pagination, auth headers handled |

**Key insight:** acli exists specifically to handle the complexity of Jira Cloud authentication and API interaction. Rolling custom solutions would require maintaining OAuth flows, token refresh, rate limiting, and API versioning.

## Common Pitfalls

### Pitfall 1: acli Not Installed or Not in PATH
**What goes wrong:** Command fails with ENOENT or "command not found"
**Why it happens:** acli requires separate installation, not an npm package
**How to avoid:** Check for acli existence before any operations; provide clear installation instructions
**Warning signs:** child.on('error') fires with ENOENT

```javascript
// Prevention: Check upfront
async function verifyAcliInstalled() {
  try {
    await runAcli(['--version']);
    return true;
  } catch (error) {
    if (error.code === 'acli_not_found') {
      console.error(`
acli is not installed or not in PATH.

Install from: https://developer.atlassian.com/cloud/acli/guides/install-acli/
      `);
      return false;
    }
    throw error;
  }
}
```

### Pitfall 2: acli Not Authenticated
**What goes wrong:** Commands fail with authentication errors
**Why it happens:** User hasn't run `acli jira auth login` yet
**How to avoid:** Check auth status first; guide user to authenticate if needed
**Warning signs:** Exit code non-zero with "authentication" in stderr

```javascript
async function verifyAuthenticated() {
  try {
    const result = await runAcli(['jira', 'auth', 'status']);
    // Parse result to check if authenticated
    return result.includes('Logged in');
  } catch (error) {
    return false;
  }
}
```

### Pitfall 3: Issue Type Not Available
**What goes wrong:** Initiative type doesn't exist (requires Advanced Roadmaps)
**Why it happens:** Standard Jira doesn't have Initiative; only Epic, Story, Task, Sub-task
**How to avoid:** Detect available issue types via project configuration; fall back to Epic for Milestone
**Warning signs:** "Issue type 'Initiative' is not valid for project" error

```javascript
// Decision from CONTEXT.md: Use Epic for Milestone if Initiative not available
const FALLBACK_MAPPING = {
  'Milestone': ['Initiative', 'Epic'],  // Try Initiative, fall back to Epic
  'Phase': ['Epic', 'Story'],           // Try Epic, fall back to Story
  'Plan': ['Story', 'Task'],            // Try Story, fall back to Task
  'Task': ['Sub-task', 'Task']          // Try Sub-task, fall back to Task
};
```

### Pitfall 4: Multiple Boards for Project
**What goes wrong:** User confused which board to use
**Why it happens:** Jira projects can have multiple Scrum/Kanban boards
**How to avoid:** List all boards and ask user to confirm/select
**Warning signs:** acli board search returns multiple results

### Pitfall 5: Network/Timeout Errors
**What goes wrong:** Commands hang or fail with timeout
**Why it happens:** Network issues, Jira Cloud downtime, rate limiting
**How to avoid:** Set reasonable timeouts; provide retry guidance
**Warning signs:** Command takes >30 seconds; connection refused errors

## Code Examples

Verified patterns from official sources:

### acli Authentication Verification
```bash
# Source: https://developer.atlassian.com/cloud/acli/reference/commands/jira-auth-status/
$ acli jira auth status
# Returns account info if authenticated, error if not
```

### acli Project Listing
```bash
# Source: https://developer.atlassian.com/cloud/acli/reference/commands/jira-project-list/
# List all projects in JSON format
$ acli jira project list --json

# List with pagination for large instances
$ acli jira project list --paginate --json

# List recently viewed projects
$ acli jira project list --recent --json
```

### acli Board Search
```bash
# Source: https://developer.atlassian.com/cloud/acli/reference/commands/jira-board-search/
# Search boards for a specific project
$ acli jira board search --project "PROJ" --json

# Filter by board type (scrum only for this integration)
$ acli jira board search --project "PROJ" --type scrum --json
```

### acli Project View
```bash
# Source: https://developer.atlassian.com/cloud/acli/reference/commands/jira-project-view/
# Get project details including issue types
$ acli jira project view --key "PROJ" --json
```

### YAML Parsing with Comments
```javascript
// Source: eemeli/yaml documentation (Context7 /eemeli/yaml)
const YAML = require('yaml');

// Parse YAML preserving document structure
const doc = YAML.parseDocument(`
# GSD-Jira Mapping Configuration
# Do not edit manually unless you know what you're doing

mapping:
  milestone: Initiative
  phase: Epic
  plan: Story
  task: Sub-task
`);

// Access values
console.log(doc.get('mapping')); // { milestone: 'Initiative', ... }

// Modify preserving comments
doc.set('mapping', { ...doc.get('mapping'), plan: 'Task' });

// Stringify preserving comments
console.log(doc.toString());
// Output includes original comments
```

### Node.js Child Process with Error Handling
```javascript
// Source: Node.js v25.x child_process documentation
const { spawn } = require('child_process');

function execCommand(command, args) {
  return new Promise((resolve, reject) => {
    const child = spawn(command, args);
    let stdout = '';
    let stderr = '';

    child.stdout.on('data', (data) => { stdout += data.toString(); });
    child.stderr.on('data', (data) => { stderr += data.toString(); });

    child.on('error', (error) => {
      // ENOENT = command not found
      reject({ type: 'spawn_error', error, message: `Failed to spawn ${command}` });
    });

    child.on('close', (code) => {
      if (code === 0) {
        resolve(stdout);
      } else {
        reject({ type: 'exit_error', code, stderr });
      }
    });
  });
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Bob Swift ACLI (third-party) | Official Atlassian CLI | 2024+ | Official tool now available; use it instead |
| js-yaml | yaml (eemeli/yaml) | 2020+ | yaml supports YAML 1.2 fully, preserves comments |
| REST API with oauth | acli with built-in auth | 2024+ | No need to manage tokens manually |

**Deprecated/outdated:**
- **Bob Swift CLI (bobswift.atlassian.net):** Third-party tool; use official Atlassian CLI instead
- **js-yaml for config files:** Doesn't preserve comments; use yaml package for user-editable configs

## Open Questions

Things that couldn't be fully resolved:

1. **Issue Type Detection**
   - What we know: Projects have different issue type schemes; not all have Initiative
   - What's unclear: Exact acli command to list available issue types for a project
   - Recommendation: Parse `acli jira project view --key PROJ --json` output for issueTypes field; if not available, attempt creation and handle errors

2. **Workflow Status Mapping**
   - What we know: Decision to auto-detect status mapping from Jira workflow
   - What's unclear: acli doesn't have a direct "list statuses" command
   - Recommendation: Defer to Phase 2 (push sync) when status mapping is actually needed; use JQL search to discover statuses if required

3. **acli Exit Codes**
   - What we know: acli uses HTTP status codes for errors
   - What's unclear: Complete mapping of exit codes to error types
   - Recommendation: Check stderr content for error categorization; exit code 0 = success, non-zero = check stderr message

## Sources

### Primary (HIGH confidence)
- https://developer.atlassian.com/cloud/acli/reference/commands/jira-auth-status/ - Auth verification command
- https://developer.atlassian.com/cloud/acli/reference/commands/jira-auth-login/ - Auth login command
- https://developer.atlassian.com/cloud/acli/reference/commands/jira-project-list/ - Project listing
- https://developer.atlassian.com/cloud/acli/reference/commands/jira-project-view/ - Project details
- https://developer.atlassian.com/cloud/acli/reference/commands/jira-board-search/ - Board discovery
- https://developer.atlassian.com/cloud/acli/reference/commands/jira-workitem-create/ - Issue creation (for Phase 2)
- Context7 /eemeli/yaml - YAML library documentation
- Node.js v25.x child_process documentation

### Secondary (MEDIUM confidence)
- https://developer.atlassian.com/cloud/acli/guides/troubleshooting-guide/ - Error handling patterns
- https://developer.atlassian.com/cloud/acli/guides/install-acli/ - Installation requirements

### Tertiary (LOW confidence)
- General Node.js CLI wrapper patterns from WebSearch - needs validation against actual acli behavior

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - Verified with official Atlassian docs and Context7
- Architecture: HIGH - Based on existing GSD patterns and official acli structure
- Pitfalls: MEDIUM - Based on documented errors; edge cases may exist

**Research date:** 2026-02-01
**Valid until:** 2026-03-01 (30 days - acli is stable tooling)

---

## acli Command Reference for Phase 1

Quick reference of verified acli commands needed:

| Operation | Command | Output |
|-----------|---------|--------|
| Check auth | `acli jira auth status` | Text with account info |
| Login (if needed) | `acli jira auth login --web` | Opens browser for OAuth |
| List projects | `acli jira project list --json` | JSON array of projects |
| View project | `acli jira project view --key PROJ --json` | Project details with issue types |
| Search boards | `acli jira board search --project PROJ --type scrum --json` | JSON array of boards |

All commands support `--help` for detailed options.
