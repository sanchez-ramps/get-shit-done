# Architecture Research: GSD-Jira CLI Sync Tool

**Research Date:** 2026-01-31  
**Status:** Architecture Design Complete  
**Goal:** CLI wrapper architecture that bridges GSD planning artifacts with Jira Cloud via `acli`

---

## Executive Summary

This document defines the architecture for a CLI sync tool that synchronizes GSD (Get Shit Done) workflow artifacts with Jira Cloud Scrum boards. The solution wraps the Atlassian CLI (`acli`) rather than using the Jira API directly, providing a subprocess-based integration layer.

**Key Architectural Decisions:**
- **CLI Wrapper:** Subprocess wrapper around `acli` (not direct REST API)
- **State Management:** File-based sync state in `.planning/jira/sync-state.json`
- **Conflict Strategy:** Version-based detection with timestamp comparison
- **Sync Model:** Goal-oriented synchronization (drive toward desired state)
- **Technology:** Node.js (aligns with existing GSD tooling)

---

## System Overview Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           GSD-JIRA SYNC ARCHITECTURE                        │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│   GSD COMMANDS   │     │   SYNC ENGINE    │     │   JIRA CLOUD     │
│                  │     │                  │     │                  │
│  gsd sync        │────▶│  Orchestrator    │────▶│  Scrum Board     │
│  gsd push        │     │       │          │     │                  │
│  gsd pull        │     │       ▼          │     │  Epic            │
│  gsd status      │     │  ┌─────────┐     │     │   └─ Story       │
│                  │     │  │ State   │     │     │      └─ Task     │
└──────────────────┘     │  │ Manager │     │     └──────────────────┘
                         │  └─────────┘     │              ▲
                         │       │          │              │
┌──────────────────┐     │       ▼          │     ┌───────┴──────────┐
│   GSD ARTIFACTS  │     │  ┌─────────┐     │     │   ACLI WRAPPER   │
│                  │     │  │Conflict │     │     │                  │
│  .planning/      │────▶│  │Detector │     │────▶│  subprocess      │
│   ├─ PROJECT.md  │     │  └─────────┘     │     │  spawn/exec      │
│   ├─ ROADMAP.md  │     │       │          │     │                  │
│   ├─ phases/     │     │       ▼          │     │  ┌────────────┐  │
│   └─ jira/       │     │  ┌─────────┐     │     │  │   acli     │  │
│      ├─config    │     │  │ Mapper  │     │     │  │   jira     │  │
│      └─sync-state│     │  └─────────┘     │     │  │   board    │  │
└──────────────────┘     └──────────────────┘     │  │   sprint   │  │
                                                  │  │   workitem │  │
┌──────────────────┐     ┌──────────────────┐     │  └────────────┘  │
│   GSD PARSER     │     │   TRANSFORMER    │     └──────────────────┘
│                  │     │                  │
│  Markdown AST    │────▶│  GSD ←→ Jira    │
│  YAML frontmatter│     │  Entity mapping  │
│  Phase extraction│     │  Field transform │
└──────────────────┘     └──────────────────┘
```

---

## Core Components

### 1. CLI Command Layer

**Responsibility:** User-facing commands that integrate with existing GSD workflow.

**Boundaries:**
- INPUT: User commands, CLI flags, environment variables
- OUTPUT: Status messages, sync reports, error handling
- DOES NOT: Parse GSD files, call acli directly, manage state

**Commands:**
```
gsd jira sync          # Bidirectional sync (smart)
gsd jira push          # GSD → Jira only
gsd jira pull          # Jira → GSD only  
gsd jira status        # Show sync state and conflicts
gsd jira resolve       # Interactive conflict resolution
gsd jira init          # Initialize jira config
```

**Component Structure:**
```
commands/gsd/
├── jira-sync.md       # Command definition
├── jira-push.md
├── jira-pull.md
├── jira-status.md
├── jira-resolve.md
└── jira-init.md
```

---

### 2. Sync Engine (Orchestrator)

**Responsibility:** Coordinate sync operations, manage workflow state machine.

**Boundaries:**
- INPUT: Parsed GSD entities, Jira state, sync configuration
- OUTPUT: Sync operations (push/pull/conflict), state updates
- DOES NOT: Parse markdown, execute acli commands, transform data

**State Machine:**
```
┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
│  IDLE   │────▶│  SCAN   │────▶│ COMPARE │────▶│ EXECUTE │
└─────────┘     └─────────┘     └─────────┘     └─────────┘
     ▲               │               │               │
     │               ▼               ▼               ▼
     │          ┌─────────┐     ┌─────────┐     ┌─────────┐
     └──────────│  ERROR  │     │CONFLICT │     │ COMMIT  │
                └─────────┘     └─────────┘     └─────────┘
```

**Sync Flow:**
1. **SCAN:** Read GSD files + query Jira via acli
2. **COMPARE:** Diff local vs remote state using timestamps/versions
3. **EXECUTE:** Apply changes (push/pull) via acli wrapper
4. **COMMIT:** Update sync-state.json with new checksums

---

### 3. ACLI Wrapper

**Responsibility:** Execute Atlassian CLI commands via subprocess.

**Boundaries:**
- INPUT: Structured command objects (action, entity, fields)
- OUTPUT: Parsed JSON responses, error objects
- DOES NOT: Understand GSD entities, manage state, handle conflicts

**Subprocess Pattern (Node.js):**
```javascript
// RECOMMENDED: Use execa for subprocess execution
// - No shell injection risk (no shell by default)
// - Promise-based API
// - Better error handling
// - Cross-platform compatibility

const { execa } = require('execa');

class AcliWrapper {
  constructor(config) {
    this.profile = config.profile;  // acli profile name
    this.timeout = config.timeout || 30000;
  }

  async execute(command, args = []) {
    const fullArgs = [
      command,
      '--profile', this.profile,
      '--output', 'json',
      ...args
    ];
    
    try {
      const { stdout, stderr } = await execa('acli', fullArgs, {
        timeout: this.timeout,
        reject: true
      });
      return JSON.parse(stdout);
    } catch (error) {
      throw new AcliError(error.message, error.exitCode);
    }
  }
  
  // Jira-specific commands
  async createIssue(projectKey, issueType, fields) {
    return this.execute('jira', [
      'workitem', 'create',
      '--project', projectKey,
      '--type', issueType,
      '--json', JSON.stringify(fields)
    ]);
  }
  
  async updateIssue(issueKey, fields) {
    return this.execute('jira', [
      'workitem', 'update',
      '--key', issueKey,
      '--json', JSON.stringify(fields)
    ]);
  }
  
  async getIssue(issueKey) {
    return this.execute('jira', [
      'workitem', 'view',
      '--key', issueKey
    ]);
  }
  
  async listBoardIssues(boardId) {
    return this.execute('jira', [
      'board', 'list',
      '--board', boardId
    ]);
  }
}
```

**Command Construction:**
```
acli jira workitem create \
  --profile gsd-sync \
  --project PROJ \
  --type Story \
  --summary "Phase 8.5: API Implementation" \
  --description "..." \
  --output json
```

---

### 4. State Manager

**Responsibility:** Track sync state, entity mappings, timestamps.

**Boundaries:**
- INPUT: Entity changes, sync results
- OUTPUT: Sync state queries, entity lookups
- DOES NOT: Execute sync, parse files, call external systems

**State File Structure:**
```json
{
  "version": "1.0",
  "lastSync": "2026-01-31T10:30:00Z",
  "profile": "gsd-sync",
  "projectKey": "PROJ",
  "entities": {
    "milestone:v1.0": {
      "jiraKey": "PROJ-100",
      "jiraType": "Epic",
      "localHash": "sha256:abc123...",
      "remoteUpdated": "2026-01-30T15:20:00Z",
      "localPath": "milestones/v1.0-AUDIT.md",
      "status": "synced"
    },
    "phase:8.5": {
      "jiraKey": "PROJ-105", 
      "jiraType": "Story",
      "parentKey": "PROJ-100",
      "localHash": "sha256:def456...",
      "remoteUpdated": "2026-01-29T10:00:00Z",
      "localPath": "phases/08-api/08-05-PLAN.md",
      "status": "synced"
    }
  },
  "conflicts": []
}
```

**Entity ID Convention:**
```
{entityType}:{identifier}

milestone:v1.0          # Milestone with version
phase:8.5               # Phase number
plan:8.5.1              # Plan within phase
requirement:8.5.1.REQ01 # Requirement within plan
```

---

### 5. Conflict Detector

**Responsibility:** Identify sync conflicts using version comparison.

**Boundaries:**
- INPUT: Local state, remote state, sync history
- OUTPUT: Conflict list with resolution options
- DOES NOT: Resolve conflicts, modify files, call external systems

**Detection Strategy (Version-Based):**
```
┌────────────────────────────────────────────────────────────────────┐
│                    CONFLICT DETECTION MATRIX                       │
├────────────────┬───────────────┬───────────────┬──────────────────┤
│ Local Changed  │ Remote Changed│ Action        │ Conflict?        │
├────────────────┼───────────────┼───────────────┼──────────────────┤
│ NO             │ NO            │ SKIP          │ NO               │
│ YES            │ NO            │ PUSH          │ NO               │
│ NO             │ YES           │ PULL          │ NO               │
│ YES            │ YES           │ CONFLICT      │ YES              │
├────────────────┼───────────────┼───────────────┼──────────────────┤
│ DELETED        │ NO            │ DELETE_REMOTE │ NO               │
│ NO             │ DELETED       │ DELETE_LOCAL  │ CONFIRM          │
│ DELETED        │ YES           │ CONFLICT      │ YES              │
│ YES            │ DELETED       │ CONFLICT      │ YES              │
└────────────────┴───────────────┴───────────────┴──────────────────┘
```

**Conflict Types:**
1. **BOTH_MODIFIED:** GSD and Jira both changed since last sync
2. **DELETE_CONFLICT:** One side deleted, other modified
3. **MAPPING_MISMATCH:** Entity exists but IDs don't match
4. **PARENT_MISSING:** Child entity without synced parent

---

### 6. GSD Parser

**Responsibility:** Extract structured data from GSD markdown files.

**Boundaries:**
- INPUT: Markdown file paths, file content
- OUTPUT: Parsed entity objects (milestones, phases, plans)
- DOES NOT: Transform to Jira format, manage state, handle sync

**Parser Structure:**
```javascript
class GsdParser {
  parseProject(projectPath) {
    // Returns: { name, description, milestones[] }
  }
  
  parseRoadmap(roadmapPath) {
    // Returns: { phases[], dependencies }
  }
  
  parseMilestone(milestonePath) {
    // Returns: { id, name, description, phases[], status }
  }
  
  parsePhase(phasePath) {
    // Returns: { id, name, goal, plans[], status }
  }
  
  parsePlan(planPath) {
    // Returns: { id, name, requirements[], status }
  }
}
```

**Content Hashing:**
```javascript
// Hash content for change detection (not timestamps)
function hashEntity(entity) {
  const normalized = JSON.stringify({
    name: entity.name,
    description: entity.description,
    status: entity.status
    // Exclude metadata, timestamps, ids
  });
  return crypto.createHash('sha256').update(normalized).digest('hex');
}
```

---

### 7. Transformer (Mapper)

**Responsibility:** Convert between GSD entities and Jira issue formats.

**Boundaries:**
- INPUT: GSD entities, Jira issues, mapping config
- OUTPUT: Transformed entities in target format
- DOES NOT: Parse files, execute commands, manage state

**Mapping Configuration:**
```json
{
  "mappings": {
    "milestone": {
      "jiraType": "Epic",
      "fields": {
        "name": "summary",
        "description": "description",
        "status": { "field": "status", "transform": "statusMap" }
      }
    },
    "phase": {
      "jiraType": "Story",
      "parent": "milestone",
      "fields": {
        "name": "summary",
        "goal": "description",
        "status": { "field": "status", "transform": "statusMap" }
      }
    },
    "plan": {
      "jiraType": "Task",
      "parent": "phase",
      "fields": {
        "name": "summary",
        "requirements": { "field": "description", "transform": "listToMarkdown" }
      }
    }
  },
  "transforms": {
    "statusMap": {
      "pending": "To Do",
      "in_progress": "In Progress", 
      "complete": "Done",
      "blocked": "Blocked"
    }
  }
}
```

---

## Data Flow Patterns

### Push Flow (GSD → Jira)

```
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│  GSD    │    │ Parser  │    │ State   │    │Conflict │    │  ACLI   │
│  Files  │    │         │    │ Manager │    │Detector │    │ Wrapper │
└────┬────┘    └────┬────┘    └────┬────┘    └────┬────┘    └────┬────┘
     │              │              │              │              │
     │   read       │              │              │              │
     ├─────────────▶│              │              │              │
     │              │              │              │              │
     │              │  entities    │              │              │
     │              ├─────────────▶│              │              │
     │              │              │              │              │
     │              │              │  get state   │              │
     │              │              ├─────────────▶│              │
     │              │              │              │              │
     │              │              │  conflicts?  │              │
     │              │              │◀─────────────┤              │
     │              │              │              │              │
     │              │              │              │   push       │
     │              │              │              ├─────────────▶│
     │              │              │              │              │
     │              │              │  update      │              │
     │              │              │◀─────────────┼──────────────┤
     │              │              │              │              │
     ▼              ▼              ▼              ▼              ▼
```

### Pull Flow (Jira → GSD)

```
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│  ACLI   │    │Transform│    │ State   │    │Conflict │    │  GSD    │
│ Wrapper │    │         │    │ Manager │    │Detector │    │  Files  │
└────┬────┘    └────┬────┘    └────┬────┘    └────┬────┘    └────┬────┘
     │              │              │              │              │
     │  list issues │              │              │              │
     ├─────────────▶│              │              │              │
     │              │              │              │              │
     │              │  transform   │              │              │
     │              ├─────────────▶│              │              │
     │              │              │              │              │
     │              │              │  check local │              │
     │              │              ├─────────────▶│              │
     │              │              │              │              │
     │              │              │  conflicts?  │              │
     │              │              │◀─────────────┤              │
     │              │              │              │              │
     │              │              │              │   write      │
     │              │              │              ├─────────────▶│
     │              │              │              │              │
     │              │              │  update      │              │
     │              │              │◀─────────────┼──────────────┤
     │              │              │              │              │
     ▼              ▼              ▼              ▼              ▼
```

---

## Recommended Project Structure

```
get-shit-done/
├── commands/gsd/
│   ├── jira-sync.md           # Sync command definition
│   ├── jira-push.md           # Push command definition
│   ├── jira-pull.md           # Pull command definition
│   ├── jira-status.md         # Status command definition
│   ├── jira-resolve.md        # Resolve command definition
│   └── jira-init.md           # Init command definition
│
├── lib/
│   └── jira/                  # Jira integration module
│       ├── index.js           # Module entry point
│       │
│       ├── acli/              # ACLI wrapper layer
│       │   ├── wrapper.js     # Subprocess execution
│       │   ├── commands.js    # Command builders
│       │   └── parser.js      # Response parsing
│       │
│       ├── sync/              # Sync engine layer
│       │   ├── engine.js      # Orchestrator
│       │   ├── state.js       # State manager
│       │   ├── conflict.js    # Conflict detector
│       │   └── resolver.js    # Conflict resolution
│       │
│       ├── parser/            # GSD parsing layer
│       │   ├── gsd.js         # GSD file parser
│       │   ├── markdown.js    # Markdown utilities
│       │   └── hash.js        # Content hashing
│       │
│       └── transform/         # Transformation layer
│           ├── mapper.js      # Entity mapper
│           ├── gsd-to-jira.js # GSD → Jira transform
│           └── jira-to-gsd.js # Jira → GSD transform
│
└── templates/
    └── jira/                  # Jira config templates
        ├── config.json        # Default config structure
        └── mapping.json       # Default field mappings
```

---

## Build Order & Dependencies

**Layer Dependency Graph:**
```
┌─────────────────────────────────────────────────────────────────────┐
│                         BUILD ORDER                                  │
└─────────────────────────────────────────────────────────────────────┘

Layer 1 (No Dependencies):
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ acli/       │  │ parser/     │  │ transform/  │
│ wrapper.js  │  │ markdown.js │  │ mapper.js   │
└─────────────┘  └─────────────┘  └─────────────┘
      │                │                │
      ▼                ▼                ▼
Layer 2 (Depends on Layer 1):
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ acli/       │  │ parser/     │  │ transform/  │
│ commands.js │  │ gsd.js      │  │ gsd-to-jira │
└─────────────┘  └─────────────┘  └─────────────┘
      │                │                │
      ▼                ▼                ▼
Layer 3 (Depends on Layer 2):
┌─────────────────────────────────────────────────┐
│              sync/state.js                       │
│              sync/conflict.js                    │
└─────────────────────────────────────────────────┘
                      │
                      ▼
Layer 4 (Depends on Layer 3):
┌─────────────────────────────────────────────────┐
│              sync/engine.js                      │
│              sync/resolver.js                    │
└─────────────────────────────────────────────────┘
                      │
                      ▼
Layer 5 (Commands - Depends on All):
┌─────────────────────────────────────────────────┐
│              commands/gsd/jira-*.md              │
└─────────────────────────────────────────────────┘
```

**Implementation Order:**
1. `acli/wrapper.js` - Subprocess execution (testable in isolation)
2. `parser/markdown.js` - Markdown utilities (pure functions)
3. `transform/mapper.js` - Mapping logic (pure functions)
4. `acli/commands.js` - Jira command builders
5. `parser/gsd.js` - GSD file parser
6. `transform/gsd-to-jira.js` - Entity transformation
7. `sync/state.js` - State management
8. `sync/conflict.js` - Conflict detection
9. `sync/engine.js` - Orchestration
10. Commands integration

---

## Subprocess Patterns for ACLI

### Safe Execution Pattern

```javascript
const { execa } = require('execa');

// GOOD: Use execa with explicit args array (no shell injection)
async function safeExecute(args) {
  return execa('acli', args, {
    timeout: 30000,
    reject: false  // Don't throw, check exitCode
  });
}

// BAD: Never use shell string interpolation
async function unsafeExecute(userInput) {
  // DANGEROUS - shell injection vulnerability
  return exec(`acli jira workitem view --key ${userInput}`);
}
```

### Error Handling Pattern

```javascript
class AcliError extends Error {
  constructor(message, exitCode, stderr) {
    super(message);
    this.name = 'AcliError';
    this.exitCode = exitCode;
    this.stderr = stderr;
  }
}

async function executeWithRetry(args, maxRetries = 3) {
  let lastError;
  
  for (let i = 0; i < maxRetries; i++) {
    const result = await execa('acli', args, { reject: false });
    
    if (result.exitCode === 0) {
      return JSON.parse(result.stdout);
    }
    
    // Retry on transient errors (network, rate limit)
    if (result.exitCode === 1 && result.stderr.includes('rate limit')) {
      await sleep(1000 * Math.pow(2, i));  // Exponential backoff
      lastError = new AcliError(result.stderr, result.exitCode, result.stderr);
      continue;
    }
    
    // Don't retry on auth/permission errors
    throw new AcliError(result.stderr, result.exitCode, result.stderr);
  }
  
  throw lastError;
}
```

### Output Parsing Pattern

```javascript
// ACLI supports --output json for structured output
async function getIssue(key) {
  const result = await execa('acli', [
    'jira', 'workitem', 'view',
    '--key', key,
    '--output', 'json'
  ]);
  
  // Parse JSON output
  const issue = JSON.parse(result.stdout);
  
  // Normalize to internal format
  return {
    key: issue.key,
    summary: issue.fields.summary,
    description: issue.fields.description,
    status: issue.fields.status.name,
    updated: new Date(issue.fields.updated)
  };
}
```

---

## Anti-Patterns to Avoid

### 1. Direct Shell String Construction
```javascript
// BAD: Shell injection vulnerability
const cmd = `acli jira workitem view --key ${userInput}`;
exec(cmd);

// GOOD: Use args array
execa('acli', ['jira', 'workitem', 'view', '--key', userInput]);
```

### 2. Polling Without Backoff
```javascript
// BAD: Hammers Jira API
while (!done) {
  await checkStatus();
}

// GOOD: Exponential backoff
let delay = 1000;
while (!done) {
  await checkStatus();
  await sleep(delay);
  delay = Math.min(delay * 2, 30000);
}
```

### 3. State File Race Conditions
```javascript
// BAD: Read-modify-write without lock
const state = await readState();
state.entities[id] = newEntity;
await writeState(state);

// GOOD: Atomic update with file lock
await withLock('sync-state.json', async () => {
  const state = await readState();
  state.entities[id] = newEntity;
  await writeState(state);
});
```

### 4. Ignoring Parent Dependencies
```javascript
// BAD: Create child before parent
await createTask(plan);      // Task
await createStory(phase);    // Story (parent)
await createEpic(milestone); // Epic (grandparent)

// GOOD: Create parents first (topological order)
const epicKey = await createEpic(milestone);
const storyKey = await createStory(phase, { parent: epicKey });
await createTask(plan, { parent: storyKey });
```

### 5. Timestamp-Only Change Detection
```javascript
// BAD: Timestamps can be unreliable
if (localMtime > remoteMtime) {
  push();
}

// GOOD: Use content hash + timestamps
if (localHash !== lastSyncHash || localMtime > lastSyncTime) {
  // Content changed, needs sync
}
```

### 6. Monolithic Sync Operations
```javascript
// BAD: All-or-nothing sync
async function sync() {
  const entities = await getAllEntities();
  await pushAllToJira(entities);  // Fails completely if one fails
}

// GOOD: Individual entity operations with rollback
async function sync() {
  const entities = await getAllEntities();
  const results = [];
  
  for (const entity of entities) {
    try {
      results.push(await syncEntity(entity));
    } catch (error) {
      results.push({ entity, error, status: 'failed' });
    }
  }
  
  return results;  // Partial success supported
}
```

### 7. Blocking on Large Operations
```javascript
// BAD: Blocks entire sync for one slow operation
await syncAllEntities();

// GOOD: Show progress, allow cancellation
async function syncWithProgress(entities, onProgress) {
  for (let i = 0; i < entities.length; i++) {
    await syncEntity(entities[i]);
    onProgress({ current: i + 1, total: entities.length });
  }
}
```

---

## Configuration Structure

### `.planning/jira/config.json`
```json
{
  "version": "1.0",
  "profile": "gsd-sync",
  "project": {
    "key": "PROJ",
    "boardId": 123
  },
  "mapping": {
    "milestone": "Epic",
    "phase": "Story", 
    "plan": "Task"
  },
  "sync": {
    "direction": "bidirectional",
    "conflictResolution": "prompt",
    "dryRun": false
  }
}
```

### `.planning/jira/sync-state.json`
```json
{
  "version": "1.0",
  "lastSync": "2026-01-31T10:30:00Z",
  "entities": {},
  "conflicts": []
}
```

---

## Quality Gates Checklist

- [x] Components clearly defined with boundaries
- [x] Data flow direction explicit (Push/Pull diagrams)
- [x] Build order implications noted (Layer dependency graph)
- [x] Subprocess/CLI wrapping patterns documented (execa, error handling)
- [x] Anti-patterns documented (7 patterns to avoid)
- [x] State management strategy defined
- [x] Conflict detection strategy defined

---

## References

- **Atlassian CLI Documentation:** https://developer.atlassian.com/cloud/acli/
- **execa (Node.js subprocess):** https://github.com/sindresorhus/execa
- **rclone bisync (bidirectional sync patterns):** https://rclone.org/bisync
- **XOS Synchronizer Architecture:** https://guide.xosproject.org/dev/sync_arch.html
- **AWS AppSync Conflict Resolution:** https://docs.aws.amazon.com/appsync/latest/devguide/conflict-detection-and-resolution.html

---

*Last updated: 2026-01-31*
