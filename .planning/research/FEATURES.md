# Features Research: GSD-Jira Integration

## Executive Summary

This research identifies features typical of Jira integrations, categorized by necessity (table stakes vs differentiating) and risk (anti-features). Each feature includes complexity assessment and dependencies to inform MVP scoping.

**Key Finding:** The Jira integration landscape is dominated by two extremes: simple one-way sync tools and complex enterprise platforms (Exalate, Unito). GSD-Jira can differentiate by providing developer-first bidirectional sync without enterprise complexity.

---

## Hierarchy Mapping

| GSD Concept | Jira Equivalent | Sync Direction | Notes |
|-------------|-----------------|----------------|-------|
| Milestone | Initiative | GSD → Jira | Requires Jira Advanced Roadmaps |
| Phase | Epic | Bidirectional | Primary planning unit |
| Plan | Story | Bidirectional | Implementation details |
| Task | Sub-task | Bidirectional | Atomic work items |

**Jira Cloud Note:** Use unified `Parent` field for all parent-child relationships. `Epic Link` and `Parent Link` are deprecated in Jira Cloud.

---

## Table Stakes Features

*Must-have for viable v1. Without these, the integration provides no value.*

### TS-1: Authentication & Connection
**Complexity:** LOW | **Dependencies:** None

| Feature | Description |
|---------|-------------|
| API token auth | OAuth 2.0 or personal access tokens (not Basic Auth) |
| Connection validation | Test connection before operations |
| Credential storage | Secure local storage, environment variables |
| Multi-instance support | Support different Jira instances per project |

**Industry Standard:** All Jira integrations require auth. Use OAuth 2.0 or API tokens; Basic Auth is deprecated.

---

### TS-2: Unidirectional Sync (GSD → Jira)
**Complexity:** MEDIUM | **Dependencies:** TS-1

| Feature | Description |
|---------|-------------|
| Create issues | Push new GSD artifacts as Jira issues |
| Update issues | Sync title, description, status changes |
| Hierarchy preservation | Maintain Epic→Story→Sub-task relationships |
| ID mapping | Track GSD↔Jira ID relationships in sync state |
| Dry-run mode | Preview changes before applying |

**Why Table Stakes:** The core value proposition. Without push sync, developers must manually update Jira.

---

### TS-3: Status Synchronization
**Complexity:** MEDIUM | **Dependencies:** TS-2

| Feature | Description |
|---------|-------------|
| Status mapping | GSD `[x]`/`[ ]` → Jira workflow transitions |
| Workflow awareness | Query available transitions, apply valid ones |
| Batch updates | Update multiple issues efficiently |
| Error recovery | Handle failed transitions gracefully |

**Status Mapping:**
```
GSD [ ] incomplete  →  Jira "To Do"
GSD [~] in-progress →  Jira "In Progress"  
GSD [x] complete    →  Jira "Done"
```

---

### TS-4: Conflict Detection
**Complexity:** MEDIUM | **Dependencies:** TS-2, TS-3

| Feature | Description |
|---------|-------------|
| Last-modified tracking | Compare timestamps between systems |
| Change detection | Identify modified fields |
| Conflict flagging | Mark items needing resolution |
| Conflict report | Show what diverged and when |

**Conflict Types to Detect:**
- Write-write: Both sides modified same field
- Write-delete: One side deleted, other modified
- Status divergence: Different completion states

---

### TS-5: CLI Interface
**Complexity:** LOW | **Dependencies:** TS-1, TS-2

| Command | Description |
|---------|-------------|
| `jira-init` | Configure connection, initial mapping |
| `jira-sync` | Execute sync operation |
| `jira-status` | Show sync state and pending changes |
| `jira-link <id>` | Get Jira URL for GSD item |

**Why Table Stakes:** GSD is CLI-first. Developers expect command-line tooling.

---

### TS-6: Configuration Management
**Complexity:** LOW | **Dependencies:** None

| Feature | Description |
|---------|-------------|
| Project config | `.planning/jira/config.json` |
| Sync state | `.planning/jira/sync-state.json` |
| Field mapping | Configure which fields sync |
| Ignore patterns | Exclude items from sync |

---

### TS-7: Error Handling
**Complexity:** MEDIUM | **Dependencies:** TS-1, TS-2

| Feature | Description |
|---------|-------------|
| Rate limit handling | Detect 429, implement backoff |
| Retry logic | Transient failure recovery |
| Partial failure handling | Continue sync after individual item fails |
| Clear error messages | Actionable guidance for users |

**Jira Rate Limits:** Points-based system (not request count). Write operations: 1 point base. Read operations: 1 + points per object returned. New limits enforced March 2026.

---

## Differentiating Features

*Competitive advantages. Build after table stakes are solid.*

### D-1: Bidirectional Sync (GSD ↔ Jira)
**Complexity:** HIGH | **Dependencies:** TS-2, TS-3, TS-4

| Feature | Description |
|---------|-------------|
| Pull status changes | Jira "Done" → GSD `[x]` |
| Pull comments | Import PM comments as GSD notes |
| Pull new issues | Jira-created issues appear in GSD |
| Conflict resolution | Interactive or policy-based |

**Why Differentiating:** Most CLI tools are push-only. Bidirectional enables true collaboration where PMs can work in Jira.

**Conflict Resolution Strategies:**
| Strategy | Use Case |
|----------|----------|
| `force-local` | GSD is source of truth |
| `force-remote` | Jira is source of truth |
| `interactive` | Prompt user to choose |
| `timestamp-wins` | Most recent change wins |
| `skip` | Ignore conflict, log warning |

---

### D-2: Sprint Management
**Complexity:** HIGH | **Dependencies:** TS-2, TS-3

| Feature | Description |
|---------|-------------|
| Sprint creation | Create sprints from GSD phases |
| Auto-assignment | Assign Stories to sprint based on Phase |
| Sprint lifecycle | Auto-close expired, create next |
| Incomplete handling | Move unfinished items to next sprint |

**Sprint API Capabilities:**
- CRUD operations on sprints
- Move issues to/from sprints
- Set sprint properties (goal, dates)
- Swap sprint order

---

### D-3: Incremental Sync
**Complexity:** MEDIUM | **Dependencies:** TS-2, TS-4

| Feature | Description |
|---------|-------------|
| Change detection | Only sync modified items |
| Delta tracking | Track what changed since last sync |
| Efficient API usage | Minimize API calls |
| Resume capability | Continue interrupted sync |

**Why Differentiating:** Full sync is expensive. Incremental sync respects rate limits and reduces latency.

---

### D-4: Git Integration
**Complexity:** MEDIUM | **Dependencies:** TS-2

| Feature | Description |
|---------|-------------|
| Hook triggers | Sync on commit/push |
| Branch awareness | Track which branch items exist on |
| PR linking | Connect PRs to Jira issues |
| Merge handling | Resolve GSD file merges |

---

### D-5: Reporting & Visibility
**Complexity:** LOW | **Dependencies:** TS-2, TS-5

| Feature | Description |
|---------|-------------|
| Sync report | What's synced, pending, conflicted |
| Progress dashboard | Milestone/phase completion |
| Audit trail | History of sync operations |
| Health indicators | Sync lag, error rate |

---

### D-6: Bulk Operations
**Complexity:** MEDIUM | **Dependencies:** TS-2

| Feature | Description |
|---------|-------------|
| Batch create | Create multiple issues in one operation |
| Batch update | Update multiple issues efficiently |
| Batch assign | Assign items to sprint/user in bulk |
| Dry-run support | Preview bulk changes |

**Why Differentiating:** Initial sync of large projects requires efficient bulk operations.

---

### D-7: Offline-First Support
**Complexity:** HIGH | **Dependencies:** TS-2, TS-4, D-1

| Feature | Description |
|---------|-------------|
| Local queue | Queue changes when offline |
| Eventual consistency | Sync when connection restored |
| Conflict merging | Handle offline period conflicts |

**Industry Note:** Not standard in Jira integrations. Jira itself has no native offline mode. Would require CRDT-like conflict resolution.

---

## Anti-Features

*Commonly requested but problematic. Explicitly avoid.*

### AF-1: Real-Time Sync
**Complexity:** VERY HIGH | **Risk:** HIGH

| Why Requested | Why Problematic |
|---------------|-----------------|
| "Instant updates" | Webhook performance issues at scale |
| "Always current" | Creates sync loops, race conditions |
| "No manual sync" | Difficult to debug, noisy |

**Evidence:** Jira webhooks execute JQL synchronously on every event. Multiple webhooks create sequential execution that blocks user requests. Atlassian recommends minimizing webhook count.

**Alternative:** Explicit sync commands with optional cron/scheduled sync.

---

### AF-2: Full Jira UI Replication
**Complexity:** VERY HIGH | **Risk:** MEDIUM

| Why Requested | Why Problematic |
|---------------|-----------------|
| "Don't want to use Jira" | Violates Atlassian guidelines |
| "Custom boards" | Customer configs won't render |
| "Better UX" | Maintenance nightmare |

**Atlassian Guideline:** "Don't recreate Jira boards, backlogs, or workflows—link directly to genuine Jira features."

**Alternative:** Link to Jira for complex views, keep CLI focused on sync operations.

---

### AF-3: Comment/Attachment Sync
**Complexity:** HIGH | **Risk:** MEDIUM

| Why Requested | Why Problematic |
|---------------|-----------------|
| "Full conversation history" | Large data volume |
| "Don't miss PM feedback" | Storage in markdown is awkward |
| "Complete sync" | Attachments in git = bloat |

**Alternative:** Sync comment notifications as GSD alerts, link to Jira for full thread.

---

### AF-4: Multi-Project Sync
**Complexity:** VERY HIGH | **Risk:** HIGH

| Why Requested | Why Problematic |
|---------------|-----------------|
| "Monorepo with multiple Jira projects" | Complex mapping, cross-project links |
| "Team works across projects" | Permission complexity |

**Alternative:** One GSD project = one Jira project. Support multiple configs for different repos.

---

### AF-5: Historical Retroactive Sync
**Complexity:** HIGH | **Risk:** MEDIUM

| Why Requested | Why Problematic |
|---------------|-----------------|
| "Sync past work" | Creates massive duplicate issues |
| "Backfill Jira" | Timestamps and history are wrong |
| "Complete record" | Confuses Jira reporting |

**Alternative:** Initial sync captures current state. History lives in git.

---

### AF-6: Custom Workflow Engine
**Complexity:** VERY HIGH | **Risk:** HIGH

| Why Requested | Why Problematic |
|---------------|-----------------|
| "Auto-transition on PR merge" | Jira Automation exists |
| "Complex state machines" | Maintenance, testing burden |
| "Business logic in sync" | Scope creep |

**Alternative:** Use Jira Automation for workflow rules. GSD syncs status, Jira handles transitions.

---

## Feature Dependencies

```
TS-1: Auth ─────────────────────────────────────────────┐
                                                        │
TS-6: Config ───────────────────────────────────────────┤
                                                        ▼
TS-2: Push Sync ◄──────────────────────────────────── START
     │
     ├──► TS-3: Status Sync
     │         │
     │         └──► TS-4: Conflict Detection
     │                   │
     │                   └──► D-1: Bidirectional Sync
     │                             │
     │                             └──► D-7: Offline-First
     │
     ├──► TS-5: CLI Interface
     │         │
     │         └──► D-5: Reporting
     │
     ├──► TS-7: Error Handling
     │
     ├──► D-2: Sprint Management
     │
     ├──► D-3: Incremental Sync
     │
     ├──► D-4: Git Integration
     │
     └──► D-6: Bulk Operations
```

---

## MVP Recommendation

### MVP Scope (v1.0)

**Goal:** Developers can push GSD artifacts to Jira without touching Jira UI.

| Feature | Rationale |
|---------|-----------|
| TS-1: Auth | Required for any Jira interaction |
| TS-2: Push Sync | Core value proposition |
| TS-3: Status Sync | Developers mark work done in GSD |
| TS-5: CLI Interface | `jira-init`, `jira-sync`, `jira-status` |
| TS-6: Config | Store connection and mapping |
| TS-7: Error Handling | Rate limits, basic retry |

**Excluded from MVP:**
- Bidirectional sync (D-1) — adds significant complexity
- Sprint management (D-2) — manual sprint assignment acceptable
- Conflict resolution — MVP: GSD always wins, warn on conflicts

**MVP Commands:**
```bash
gsd:jira-init          # Configure connection
gsd:jira-sync          # Push all changes to Jira
gsd:jira-sync --dry-run # Preview changes
gsd:jira-status        # Show sync state
gsd:jira-link <id>     # Get Jira URL
```

### v1.1 Additions

| Feature | Rationale |
|---------|-----------|
| TS-4: Conflict Detection | Required before bidirectional |
| D-1: Bidirectional Sync | PM collaboration in Jira |
| D-3: Incremental Sync | Performance for large projects |

### v1.2 Additions

| Feature | Rationale |
|---------|-----------|
| D-2: Sprint Management | Full Scrum workflow support |
| D-5: Reporting | Visibility for stakeholders |
| D-6: Bulk Operations | Large project efficiency |

---

## Competitive Landscape

| Tool | Approach | Strengths | Weaknesses |
|------|----------|-----------|------------|
| **Exalate** | Scripted sync (Groovy) | Highly customizable | Requires developers, expensive |
| **Unito** | No-code visual flows | Easy setup, 54+ connectors | Less customizable |
| **jira-cli** | CLI for Jira operations | Developer-friendly | No sync, just Jira client |
| **Atlassian CLI** | Official CLI suite | Full API coverage | No planning tool integration |
| **GSD-Jira** | Developer-first sync | GSD-native, bidirectional | New, limited connectors |

**GSD-Jira Differentiation:**
1. **Developer-first**: CLI-native, works with existing GSD workflow
2. **Structured sync**: Understands GSD hierarchy, not generic field mapping
3. **No enterprise overhead**: No consultants, no Groovy scripting
4. **Bidirectional without complexity**: Pull PM changes without full Unito-style platform

---

## Technical Constraints

### Jira API Rate Limits (Effective March 2026)
- Points-based system, not request count
- Write operations: 1 point base cost
- Read operations: 1 point + points per object
- 429 response when exceeded → implement exponential backoff

### Jira Cloud vs Server/Data Center
- **Cloud:** Unified `Parent` field for hierarchy
- **Server/DC:** Separate `Epic Link`, `Parent Link`, `Parent ID` fields
- **Recommendation:** Cloud-only for MVP, reduces complexity

### Webhook Limitations
- JQL filters execute synchronously on events
- Multiple webhooks create sequential blocking
- High webhook count degrades Jira performance
- **Recommendation:** Avoid webhook-based real-time sync

---

## Open Questions

1. **Conflict strategy default:** GSD-wins vs timestamp-wins vs interactive?
2. **Deletion handling:** Archive Jira issues or soft-delete?
3. **Initiative support:** Require Advanced Roadmaps or fallback to Epic?
4. **Sprint auto-mode:** How to handle incomplete items at sprint end?
5. **Comment sync scope:** Just notifications or full threads?

---

## Research Sources

- Atlassian Jira Cloud REST API documentation
- Atlassian Developer Guidelines for Jira Integration
- Exalate vs Unito comparison analysis
- Jira webhook performance documentation
- jira-cli (ankitpokhrel) and Atlassian CLI documentation
- Rate limiting announcement (March 2026 changes)
- Local-first software architecture patterns

---

*Research completed: 2026-01-31*
*Quality gates: Categories ✓ | Complexity ✓ | Dependencies ✓ | MVP ✓*
