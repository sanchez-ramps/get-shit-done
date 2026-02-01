# Phase 2: Push Sync - Research

**Researched:** 2026-02-01
**Domain:** Jira issue creation/linking via acli, git branch automation, sync state management
**Confidence:** HIGH

## Summary

Phase 2 implements GSD → Jira push sync with automatic creation hooks and branch creation. This research investigated acli commands for creating Issues, Epics, and Initiatives with hierarchy linking, git branch operations for Node.js automation, and patterns for tracking sync state to detect Jira modifications.

The official Atlassian CLI (`acli`) provides full support for issue creation with parent linking via the `--parent` flag. Jira has consolidated `epic-link` and `parent-link` fields into a unified `parent` field, simplifying hierarchy management. The `simple-git` npm package (steveukx/git-js) is the recommended library for programmatic git operations including branch creation, checkout, and status checking.

For sync state tracking, the approach is to store `lastSynced` timestamp per entity in the sync state file, then use acli's JQL search with `updated > "timestamp"` to detect Jira-side modifications before pushing updates.

**Primary recommendation:** Use acli with `--parent` flag for hierarchy linking, simple-git for branch operations, and a sync state YAML file that tracks per-entity timestamps to detect remote modifications.

## Standard Stack

The established libraries/tools for this domain:

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| acli | latest | Atlassian CLI for Jira Cloud | Official Atlassian tool, handles auth, supports all needed operations |
| simple-git | ^3.x | Git operations in Node.js | 311 code snippets in Context7, High reputation, Promise-based API |
| yaml | ^2.x | YAML parsing/serialization | Already established in Phase 1, preserves comments |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| fs/promises | Node.js built-in | Async file operations | Reading/writing sync state files |
| child_process | Node.js built-in | Spawning acli commands | Wrapper implementation (Phase 1 pattern) |
| path | Node.js built-in | Path manipulation | Config and state file paths |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| simple-git | isomorphic-git | isomorphic-git is pure JS (no git binary needed) but heavier; simple-git better for this use case |
| simple-git | child_process spawn | Direct git commands work but simple-git provides better error handling and types |

**Installation:**
```bash
npm install simple-git
# yaml already installed in Phase 1
# acli must be installed separately via Atlassian's installer
```

## Architecture Patterns

### Recommended Project Structure
```
.planning/jira/           # Created by gsd:jira-init
├── config.yaml           # Project configuration (from Phase 1)
├── mapping.yaml          # Entity type mapping (from Phase 1)
└── sync-state.yaml       # NEW: Per-entity sync tracking

commands/gsd/
├── jira-init.md          # (from Phase 1)
├── jira-sync.md          # NEW: Push sync command
└── jira-status.md        # NEW: Show sync status
```

### Pattern 1: Sync State Tracking
**What:** Track per-entity sync status with timestamps and Jira keys
**When to use:** All sync operations to detect changes and prevent conflicts
**Example:**
```yaml
# .planning/jira/sync-state.yaml
# AUTO-GENERATED — Do not edit manually

version: 1
lastFullSync: 2026-02-01T14:30:00Z

entities:
  milestones:
    "01-mvp":
      jiraKey: PROJ-10
      jiraUrl: https://company.atlassian.net/browse/PROJ-10
      lastSynced: 2026-02-01T14:30:00Z
      localHash: abc123   # Hash of local content at sync time
  
  phases:
    "01-foundation":
      jiraKey: PROJ-11
      jiraUrl: https://company.atlassian.net/browse/PROJ-11
      parentKey: PROJ-10  # Links to milestone Initiative
      lastSynced: 2026-02-01T14:30:00Z
      localHash: def456
  
  plans:
    "01-01":
      jiraKey: PROJ-12
      jiraUrl: https://company.atlassian.net/browse/PROJ-12
      parentKey: PROJ-11  # Links to phase Epic
      lastSynced: 2026-02-01T14:30:00Z
      localHash: ghi789

pending:
  - type: phase
    id: "02-push-sync"
    action: create
    reason: "Sync failed: network error"
    timestamp: 2026-02-01T15:00:00Z
```

### Pattern 2: Issue Creation with Hierarchy
**What:** Create Jira issues with proper parent-child relationships
**When to use:** Creating Initiatives, Epics, Stories with linked hierarchy
**Example:**
```javascript
// Source: https://developer.atlassian.com/cloud/acli/reference/commands/jira-workitem-create/

// Create Initiative (for Milestone)
const initiative = await runAcli([
  'jira', 'workitem', 'create',
  '--project', 'PROJ',
  '--type', 'Initiative',  // Requires Advanced Roadmaps
  '--summary', 'Milestone: MVP Release',
  '--description', 'First milestone for GSD-Jira integration',
  '--json'
], { json: true });
// Returns: { key: 'PROJ-10', self: 'https://...', ... }

// Create Epic (for Phase) with parent Initiative
const epic = await runAcli([
  'jira', 'workitem', 'create',
  '--project', 'PROJ',
  '--type', 'Epic',
  '--summary', 'Phase 1: Foundation',
  '--description', 'CLI scaffold, config schema, acli integration',
  '--parent', initiative.key,  // Links to Initiative
  '--json'
], { json: true });

// Create Story (for Plan) with parent Epic
const story = await runAcli([
  'jira', 'workitem', 'create',
  '--project', 'PROJ',
  '--type', 'Story',
  '--summary', 'Plan 1: gsd:jira-init Command',
  '--parent', epic.key,  // Links to Epic
  '--json'
], { json: true });
```

### Pattern 3: Detecting Remote Modifications
**What:** Check if Jira issues were modified since last sync
**When to use:** Before pushing updates to avoid overwriting remote changes
**Example:**
```javascript
// Source: https://developer.atlassian.com/cloud/acli/reference/commands/jira-workitem-search/

async function getModifiedIssues(projectKey, sinceDate) {
  // Format date for JQL: YYYY-MM-DD HH:mm
  const jqlDate = sinceDate.toISOString().replace('T', ' ').slice(0, 16);
  
  return runAcli([
    'jira', 'workitem', 'search',
    '--jql', `project = ${projectKey} AND updated > "${jqlDate}"`,
    '--fields', 'key,summary,updated',
    '--json'
  ], { json: true });
}

async function checkForConflicts(entities, syncState) {
  const modifiedKeys = [];
  
  for (const [id, entity] of Object.entries(entities)) {
    const jiraKey = syncState.entities[entity.type][id]?.jiraKey;
    if (!jiraKey) continue;
    
    // Check if modified in Jira since our last sync
    const lastSynced = syncState.entities[entity.type][id].lastSynced;
    const issue = await runAcli([
      'jira', 'workitem', 'view', jiraKey,
      '--fields', 'updated',
      '--json'
    ], { json: true });
    
    if (new Date(issue.updated) > new Date(lastSynced)) {
      modifiedKeys.push({
        jiraKey,
        localId: id,
        jiraUpdated: issue.updated,
        lastSynced
      });
    }
  }
  
  return modifiedKeys;
}
```

### Pattern 4: Git Branch Creation
**What:** Create feature branches with Jira issue key following naming convention
**When to use:** On execute-phase to create branch with Jira key
**Example:**
```javascript
// Source: Context7 /steveukx/git-js
import { simpleGit } from 'simple-git';

const git = simpleGit('/path/to/repo');

async function createFeatureBranch(jiraKey, phaseName) {
  // Sanitize phase name for branch naming
  const safeName = phaseName
    .toLowerCase()
    .replace(/[^a-z0-9-]/g, '-')  // Replace invalid chars with hyphen
    .replace(/-+/g, '-')          // Collapse multiple hyphens
    .replace(/^-|-$/g, '');       // Trim leading/trailing hyphens
  
  const branchName = `feature/${jiraKey}-${safeName}`;
  
  // Check if branch already exists
  const branches = await git.branch();
  if (branches.all.includes(branchName)) {
    // Checkout existing branch silently
    await git.checkout(branchName);
    return { action: 'checkout', branch: branchName };
  }
  
  // Create and checkout new branch
  await git.checkoutLocalBranch(branchName);
  return { action: 'create', branch: branchName };
}

// Validate branch name before creation
async function validateBranchName(branchName) {
  try {
    // Use git check-ref-format to validate
    await git.raw(['check-ref-format', '--branch', branchName]);
    return true;
  } catch {
    return false;
  }
}
```

### Pattern 5: Auto-Hook Integration
**What:** Sync to Jira immediately after GSD commands complete
**When to use:** Any GSD command that modifies artifacts (new-milestone, plan-phase, etc.)
**Example:**
```javascript
// At end of any GSD command that creates/modifies artifacts
async function autoSyncHook(affectedEntities, config) {
  // Check if auto-sync is enabled
  if (!config.autoSync) {
    return { synced: false, reason: 'disabled' };
  }
  
  // Check network connectivity
  try {
    await runAcli(['jira', 'auth', 'status']);
  } catch (error) {
    // Queue for later
    await addToPending(affectedEntities, 'network_error');
    console.log('⚠ Sync queued — will retry on next gsd:jira-sync');
    return { synced: false, reason: 'offline', queued: true };
  }
  
  // Sync affected entities
  const results = [];
  for (const entity of affectedEntities) {
    try {
      const result = await syncEntity(entity);
      results.push(result);
      console.log(`→ Synced to Jira: ${result.jiraKey}`);
    } catch (error) {
      await addToPending([entity], error.message);
      console.log(`⚠ Sync failed for ${entity.id} — queued for retry`);
    }
  }
  
  return { synced: true, results };
}
```

### Anti-Patterns to Avoid
- **Hardcoding issue types:** Always read from mapping.yaml for flexibility
- **Ignoring --json flag:** Always use --json for parseable acli output
- **Sync without conflict check:** Always check `updated` field before pushing changes
- **Creating branches with invalid characters:** Always sanitize phase names before branch creation
- **Blocking on sync failures:** Queue failures for retry, let GSD commands succeed

## Don't Hand-Roll

Problems that look simple but have existing solutions:

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Git branch operations | child_process git calls | simple-git library | Better error handling, Promise API, branch existence checking |
| Branch name validation | Regex pattern matching | git check-ref-format | Git's own rules, edge cases handled |
| Date formatting for JQL | Manual string formatting | Date.toISOString() + slice | Consistent ISO format accepted by Jira |
| JSON response parsing | Manual JSON.parse + field extraction | acli --json flag + destructuring | Consistent structure, type-safe |
| Pending queue persistence | In-memory tracking | sync-state.yaml | Survives restarts, visible to user |

**Key insight:** acli already handles complexity of Jira's API (auth, rate limiting, pagination). simple-git handles git's CLI complexity (error codes, output parsing). Use them rather than reimplementing.

## Common Pitfalls

### Pitfall 1: Initiative Type Not Available
**What goes wrong:** Creating Milestone as Initiative fails with "Issue type not valid"
**Why it happens:** Initiative requires Jira Advanced Roadmaps; standard Jira only has Epic/Story/Task
**How to avoid:** Check mapping.yaml for configured type; use Epic fallback if Initiative not available
**Warning signs:** Error message contains "Issue type 'Initiative' is not valid for project"

```javascript
// Prevention: Check available types and fall back
async function createMilestoneIssue(config, mapping) {
  const issueType = mapping.milestone.jiraType; // Initiative or Epic
  try {
    return await runAcli([...args, '--type', issueType, '--json']);
  } catch (error) {
    if (error.message.includes('Issue type') && issueType === 'Initiative') {
      // Fall back to Epic
      console.log('⚠ Initiative not available, using Epic');
      return await runAcli([...args, '--type', 'Epic', '--json']);
    }
    throw error;
  }
}
```

### Pitfall 2: Parent Not Found for Hierarchy Linking
**What goes wrong:** Creating Epic with --parent fails because parent Issue key doesn't exist
**Why it happens:** Parent wasn't synced first, or sync state is stale
**How to avoid:** Always sync parents before children; verify parent exists before linking
**Warning signs:** Error about parent issue not found or not accessible

```javascript
// Prevention: Sync hierarchy order
async function syncPhase(phase, milestone) {
  // Ensure milestone is synced first
  const milestoneKey = await ensureMilestoneSynced(milestone);
  
  // Now create phase Epic with parent
  return await createEpic({
    summary: `Phase ${phase.number}: ${phase.name}`,
    parent: milestoneKey
  });
}
```

### Pitfall 3: Branch Name Contains Invalid Characters
**What goes wrong:** git branch creation fails with "fatal: not a valid branch name"
**Why it happens:** Phase name contains spaces, special characters, or reserved sequences
**How to avoid:** Sanitize phase name before constructing branch name; validate with git check-ref-format
**Warning signs:** Error from git about invalid ref name

```javascript
// Prevention: Sanitize and validate
function sanitizeBranchName(phaseName) {
  return phaseName
    .toLowerCase()
    .replace(/\s+/g, '-')         // Spaces to hyphens
    .replace(/[^a-z0-9-]/g, '')   // Remove invalid chars
    .replace(/-+/g, '-')          // Collapse consecutive hyphens
    .replace(/^-|-$/g, '');       // Trim edge hyphens
}
```

### Pitfall 4: Conflict Detection Fails Silently
**What goes wrong:** Push overwrites changes made in Jira because conflict wasn't detected
**Why it happens:** Timestamp comparison uses wrong timezone, or lastSynced not updated
**How to avoid:** Store timestamps in ISO format (UTC); always update lastSynced after successful sync
**Warning signs:** Changes made in Jira UI disappear after gsd:jira-sync

```javascript
// Prevention: Proper timestamp handling
function storeTimestamp() {
  return new Date().toISOString(); // Always UTC
}

function parseJiraTimestamp(jiraDate) {
  return new Date(jiraDate); // Jira returns ISO format
}

async function hasConflict(entity, syncState) {
  const issue = await getJiraIssue(entity.jiraKey);
  const jiraUpdated = parseJiraTimestamp(issue.updated);
  const lastSynced = new Date(syncState.lastSynced);
  return jiraUpdated > lastSynced;
}
```

### Pitfall 5: Pending Queue Grows Unbounded
**What goes wrong:** Pending items accumulate, user unaware of sync failures
**Why it happens:** Auto-sync fails silently, no prompt to retry
**How to avoid:** Show pending count in gsd:jira-status; auto-retry on next gsd:jira-sync; cap queue size
**Warning signs:** pending array in sync-state.yaml grows large

## Code Examples

Verified patterns from official sources:

### acli Issue Creation with Parent
```bash
# Source: https://developer.atlassian.com/cloud/acli/reference/commands/jira-workitem-create/

# Create Epic with parent Initiative
$ acli jira workitem create \
  --project "PROJ" \
  --type "Epic" \
  --summary "Phase 2: Push Sync" \
  --parent "PROJ-10" \
  --json

# Output (JSON):
# {
#   "id": "10456",
#   "key": "PROJ-11",
#   "self": "https://company.atlassian.net/rest/api/3/issue/10456"
# }
```

### acli Issue Update
```bash
# Source: https://developer.atlassian.com/cloud/acli/reference/commands/jira-workitem-edit/

# Update issue summary and description
$ acli jira workitem edit \
  --key "PROJ-11" \
  --summary "Phase 2: Push Sync (Updated)" \
  --description "Enable GSD to Jira push sync with hooks" \
  --json
```

### acli Search for Modified Issues
```bash
# Source: https://developer.atlassian.com/cloud/acli/reference/commands/jira-workitem-search/

# Find issues updated after a specific datetime
$ acli jira workitem search \
  --jql "project = PROJ AND updated > '2026-02-01 14:30'" \
  --fields "key,summary,updated" \
  --json
```

### simple-git Branch Operations
```javascript
// Source: Context7 /steveukx/git-js
import { simpleGit, BranchSummary } from 'simple-git';

const git = simpleGit('/path/to/repo');

// Get current branch status
const status = await git.status();
console.log('Current branch:', status.current);
console.log('Is clean:', status.isClean());

// List all branches
const branches: BranchSummary = await git.branch();
console.log('All branches:', branches.all);
console.log('Current:', branches.current);

// Create and checkout new branch
await git.checkoutLocalBranch('feature/PROJ-123-new-feature');

// Checkout existing branch
await git.checkout('feature/PROJ-123-new-feature');
```

### Sync State File Management
```javascript
// Using yaml package from Phase 1
const YAML = require('yaml');
const fs = require('fs/promises');

const SYNC_STATE_PATH = '.planning/jira/sync-state.yaml';

async function loadSyncState() {
  try {
    const content = await fs.readFile(SYNC_STATE_PATH, 'utf8');
    return YAML.parse(content);
  } catch (error) {
    if (error.code === 'ENOENT') {
      return createEmptySyncState();
    }
    throw error;
  }
}

async function saveSyncState(state) {
  const doc = new YAML.Document(state);
  doc.commentBefore = 'AUTO-GENERATED — Do not edit manually';
  await fs.writeFile(SYNC_STATE_PATH, doc.toString());
}

function createEmptySyncState() {
  return {
    version: 1,
    lastFullSync: null,
    entities: {
      milestones: {},
      phases: {},
      plans: {}
    },
    pending: []
  };
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| epic-link and parent-link fields | Unified parent field | 2022-2023 | Single field for all hierarchy levels |
| Manual git commands | simple-git library | 2020+ | Promise-based, better error handling |
| Storing sync state in memory | Persistent YAML file | Best practice | Survives restarts, visible to user |
| Sync all or nothing | Per-entity sync tracking | Best practice | Granular conflict detection |

**Deprecated/outdated:**
- **epic-link field:** Use parent field instead; epic-link is being phased out
- **parent-link field:** Merged into parent field
- **Manual date formatting for JQL:** Use ISO format consistently

## Open Questions

Things that couldn't be fully resolved:

1. **acli JSON Output Structure**
   - What we know: acli supports --json flag; returns issue key and self URL
   - What's unclear: Full JSON structure including all fields (updated, parent, etc.)
   - Recommendation: Test locally with `acli jira workitem create --generate-json` to see expected format; parse key and self fields which are confirmed

2. **Rate Limiting Behavior**
   - What we know: Jira has API rate limits
   - What's unclear: How acli handles rate limits; whether it retries automatically
   - Recommendation: Add exponential backoff in sync wrapper; watch for 429 errors

3. **Bulk Create Optimization**
   - What we know: acli has `workitem create-bulk` command
   - What's unclear: Whether bulk create supports parent linking
   - Recommendation: Use sequential creates for Phase 2; investigate bulk for Phase 5 optimization

4. **Branch Creation When Detached HEAD**
   - What we know: simple-git creates branches from current HEAD
   - What's unclear: Behavior when repo is in detached HEAD state
   - Recommendation: Check git status before branch creation; warn user if detached

## Sources

### Primary (HIGH confidence)
- https://developer.atlassian.com/cloud/acli/reference/commands/jira-workitem-create/ - Issue creation with --parent flag
- https://developer.atlassian.com/cloud/acli/reference/commands/jira-workitem-edit/ - Issue update command
- https://developer.atlassian.com/cloud/acli/reference/commands/jira-workitem-search/ - JQL search command
- https://developer.atlassian.com/cloud/acli/reference/commands/jira-workitem-view/ - Issue view with --fields
- https://developer.atlassian.com/cloud/acli/reference/commands/jira-workitem-link-create/ - Issue linking
- https://support.atlassian.com/jira-software-cloud/docs/upcoming-changes-epic-link-replaced-with-parent/ - Parent field migration
- Context7 /steveukx/git-js - simple-git library documentation

### Secondary (MEDIUM confidence)
- https://git-scm.com/docs/git-check-ref-format.html - Branch naming rules
- https://support.atlassian.com/jira-software-cloud/docs/use-advanced-search-with-jira-query-language-jql - JQL syntax

### Tertiary (LOW confidence)
- WebSearch for sync state patterns - needs validation against actual implementation

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - Verified with official Atlassian docs and Context7
- Architecture: HIGH - Based on Phase 1 patterns and official acli structure
- Pitfalls: MEDIUM - Based on documented errors and Phase 1 concerns
- Sync patterns: MEDIUM - Based on standard practices, needs validation

**Research date:** 2026-02-01
**Valid until:** 2026-03-01 (30 days - acli is stable tooling)

---

## acli Command Reference for Phase 2

Quick reference of verified acli commands needed:

| Operation | Command | Key Flags |
|-----------|---------|-----------|
| Create issue | `acli jira workitem create` | `--project`, `--type`, `--summary`, `--parent`, `--json` |
| Update issue | `acli jira workitem edit` | `--key`, `--summary`, `--description`, `--json` |
| View issue | `acli jira workitem view KEY` | `--fields`, `--json` |
| Search issues | `acli jira workitem search` | `--jql`, `--fields`, `--json` |
| Link issues | `acli jira workitem link create` | `--out`, `--in`, `--type` |
| List link types | `acli jira workitem link type` | (none) |

All commands support `--help` for detailed options.

---

## Config Schema Extension

Based on decisions in CONTEXT.md, extend `.planning/jira/config.yaml`:

```yaml
# .planning/jira/config.yaml (extended for Phase 2)
site: https://company.atlassian.net
project:
  key: PROJ
  name: My Project
board:
  id: 1
  name: PROJ board

# NEW: Auto-sync configuration
sync:
  enabled: true           # Toggle auto-sync hooks
  verbosity: progress     # quiet | progress | verbose
  onConflict: skip        # skip | prompt | overwrite
  branchOnExecute: true   # Create feature branch on execute-phase

# NEW: Last sync info (read-only, updated by sync)
lastSync: 2026-02-01T14:30:00Z
```
