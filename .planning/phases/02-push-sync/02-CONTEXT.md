# Phase 2 Context: Push Sync

## Phase Overview

**Goal:** Enable GSD → Jira push sync with automatic creation hooks and branch creation

**Requirements:** CLI-02, CLI-03, AUTO-01, AUTO-02, SYNC-01, GIT-01

**Success Criteria:**
1. `gsd:jira-sync` pushes GSD artifacts to Jira
2. `gsd:jira-status` shows pending changes and last sync time
3. New milestones auto-create Jira Initiatives
4. New phases auto-create Jira Epics with linked Stories
5. Hierarchy preserved (Initiative → Epic → Story)
6. Feature branches auto-created with Jira issue key

---

## Decisions

### Sync Behavior & Output

| Decision | Choice |
|----------|--------|
| Default verbosity | **Progress** — show each item as it syncs (e.g., "Creating Epic: Phase 2... ✓") |
| Dry-run mode | **Available** (`--dry-run`) but not prompted |
| `gsd:jira-status` display | **Pending only** — list of items not yet synced |
| Completion summary | **Table format** — Item \| Action \| Jira Key \| Link |

### Conflict Prevention at Push

| Decision | Choice |
|----------|--------|
| Existing Jira issue matches GSD item | **Auto-link** — silently link and continue |
| Partial failure during sync | **Continue** — sync what you can, report failures at end |
| No local changes since last sync | **Skip silently** — "Nothing to sync" |
| Jira issue modified since last sync | **Skip by default**, `--overwrite` flag to force |

### Auto-Hook Triggers

| Decision | Choice |
|----------|--------|
| When hooks fire | **Immediately** after GSD command completes |
| Success feedback | **Inline** — "→ Synced to Jira: PROJ-45" at end of command output |
| Sync failure handling | **Warn and continue** — GSD command succeeds, sync marked pending |
| Toggle mechanism | **Config setting** in `.planning/jira/config.yaml` |
| Pending sync tracking | **STATE.md** — listed under Pending Todos or Blockers |
| Retry pending syncs | **Automatic** — next `gsd:jira-sync` picks up pending items |
| Commands that trigger hooks | **Any command** that modifies GSD artifacts |
| First sync after init | **Prompt** — "Found 12 unsynced items. Sync all now?" |
| Update sync (edits to existing items) | **Always** — any edit triggers update to Jira |
| Multi-item commands | **Sync all affected** — create Epic and update parent as needed |
| Offline behavior | **Queue silently** — add to pending, no prompt |

### Branch Naming & Workflow

| Decision | Choice |
|----------|--------|
| Branch naming convention | `feature/PROJ-123-phase-name` (e.g., `feature/PROJ-45-push-sync`) |
| Branch creation timing | **On execute-phase** — branch created when execution starts |
| Existing branch handling | **Checkout silently** — switch to existing branch |
| Auto-checkout after creation | **Always** — create and switch to branch |

---

## Deferred Ideas

*None captured during this discussion.*

---

## Research Guidance

Based on these decisions, research should focus on:

1. **acli commands** for creating Issues, Epics, Initiatives with proper hierarchy linking
2. **acli response parsing** to extract Jira keys and URLs for table output
3. **Git branch operations** — create, checkout, naming validation
4. **Sync state tracking** — how to detect "modified in Jira" without full bidirectional sync
5. **Config schema extension** for auto-sync toggle

---

*Context captured: 2026-02-01*
