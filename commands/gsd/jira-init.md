---
name: gsd:jira-init
description: Initialize Jira connection for GSD integration
allowed-tools:
  - Read
  - Write
  - Bash
  - AskUserQuestion
---

<objective>

Configure Jira connection for GSD-Jira sync by verifying acli setup, guiding project/board selection, and creating configuration files.

**Creates:**
- `.planning/jira/config.yaml` — Jira site, project, and board configuration
- `.planning/jira/mapping.yaml` — GSD entity to Jira issue type mapping

**After this command:** Run `/gsd:jira-sync` to push GSD state to Jira.

</objective>

<process>

## 1. Check Existing Configuration

Check if Jira configuration already exists:

```bash
ls .planning/jira/config.yaml 2>/dev/null
ls .planning/jira/mapping.yaml 2>/dev/null
```

**Case A: Both config.yaml and mapping.yaml exist (Full config):**

Read and display current configuration:

```bash
cat .planning/jira/config.yaml
```

Display current settings:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► JIRA — EXISTING CONFIGURATION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

| Setting  | Current Value                  |
|----------|--------------------------------|
| Site     | {current site URL}             |
| Project  | {current project key/name}     |
| Board    | {current board name}           |
```

Use AskUserQuestion:
- header: "Re-initialize"
- question: "Jira configuration already exists. What would you like to do?"
- options:
  - "Re-initialize completely" — Start fresh with new project/board selection
  - "Update mapping only" — Keep project/board, reconfigure issue type mapping
  - "View current config" — Show full configuration details
  - "Cancel" — Exit without changes

**If "Re-initialize completely":** Continue to step 2.
**If "Update mapping only":** Read existing config.yaml for project info, skip to step 6.
**If "View current config":** 
  - Display full config.yaml and mapping.yaml contents
  - Re-prompt with same options
**If "Cancel":** Exit command.

**Case B: Only config.yaml exists (Partial config):**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► JIRA — INCOMPLETE CONFIGURATION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Found config.yaml but mapping.yaml is missing.
This may be from an interrupted initialization.

Completing setup with existing project/board...
```

Read existing config.yaml, extract project/board info, skip to step 6.

**Case C: Only mapping.yaml exists (Invalid state):**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► JIRA — INVALID CONFIGURATION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Found mapping.yaml but config.yaml is missing.
Configuration is in an invalid state.

Starting fresh initialization...
```

Delete orphaned mapping.yaml, continue to step 2.

**Case D: Neither file exists (Fresh install):**

Continue to step 2.

## 2. Verify acli Installation

Check that acli CLI is installed:

```bash
acli --version
```

**On ENOENT or command not found:**

Display error and exit:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► JIRA INIT — ERROR
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

acli CLI is not installed.

**Install acli:**

Visit: https://github.com/ankitpokhrel/jira-cli

brew install ankitpokhrel/jira-cli/jira-cli

or

go install github.com/ankitpokhrel/jira-cli/cmd/jira@latest

Run `/gsd:jira-init` again after installation.
```

Exit command.

**On success:** Store version, continue to step 3.

## 3. Verify Authentication

Check that acli is authenticated:

```bash
acli jira auth status
```

**On failure (not authenticated, unauthorized, no credentials):**

Display error and exit:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► JIRA INIT — AUTHENTICATION REQUIRED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

acli is not authenticated to Jira.

**Authenticate:**

Run: acli jira auth login --web

This will open your browser to authenticate with Jira.

Run `/gsd:jira-init` again after authentication.
```

Exit command.

**On success:**

Parse the auth status output to extract:
- Site URL (e.g., `https://your-domain.atlassian.net`)
- Account email

Store site URL for config.yaml.

Continue to step 4.

## 4. Select Project

Fetch available projects:

```bash
acli jira project list --json
```

**Parse JSON response:**

Extract array of projects with:
- `key` — Project key (e.g., "PROJ")
- `name` — Project name (e.g., "My Project")
- `id` — Project ID (e.g., "10001")

**If no projects returned:**

Display error:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► JIRA INIT — NO PROJECTS FOUND
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

No Jira projects accessible with your account.

**Possible causes:**
- You don't have access to any projects
- Your Jira instance has no projects
- API permissions are restricted

**Next steps:**
1. Check your Jira permissions
2. Create a project in Jira if none exist
3. Run `/gsd:jira-init` again
```

Exit command.

**If projects found:**

Use AskUserQuestion:
- header: "Project"
- question: "Select Jira project for GSD sync:"
- options:
  - For each project: "[KEY] — [name]"
  - "Enter key directly" — I know my project key

**If "Enter key directly":**

Ask inline: "Enter project key:"

Validate entered key:

```bash
acli jira project view --key {ENTERED_KEY} --json
```

**If key not found:**

```
Project key "{ENTERED_KEY}" not found or not accessible.

Available projects:
- [KEY1]: [name1]
- [KEY2]: [name2]
...

Try again or select from list.
```

Re-prompt for project.

**If key found:** Use the returned project data.

**Store selected project:**
- `project.key`
- `project.name`
- `project.id`

Continue to step 5.

## 5. Detect and Select Board

Search for Scrum boards in the selected project:

```bash
acli jira board search --project {PROJECT_KEY} --type scrum --json
```

**Parse JSON response:**

Extract array of boards with:
- `id` — Board ID
- `name` — Board name
- `type` — Board type (scrum/kanban)

**If exactly 1 Scrum board:**

Display auto-selection with confirmation:

```
Found Scrum board: [{id}] {name}

Using this board for sprint management.
```

Store board info and continue.

**If multiple Scrum boards:**

Use AskUserQuestion:
- header: "Board"
- question: "Multiple Scrum boards found. Select board for sprint sync:"
- options:
  - For each board: "[{id}] {name}"

Store selected board info.

**If no Scrum boards:**

Check for Kanban boards:

```bash
acli jira board search --project {PROJECT_KEY} --type kanban --json
```

**If Kanban boards found:**

Use AskUserQuestion:
- header: "Board"
- question: "No Scrum boards found. Select Kanban board (sprint features will be limited):"
- options:
  - For each board: "[{id}] {name} (kanban)"
  - "Create Scrum board first" — Exit and create board in Jira

**If "Create Scrum board first":**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► CREATE SCRUM BOARD
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

To create a Scrum board:

1. Go to: {SITE_URL}/jira/software/projects/{PROJECT_KEY}/boards
2. Click "Create board"
3. Select "Scrum"
4. Configure board settings

Run `/gsd:jira-init` again after creating the board.
```

Exit command.

**If no boards at all:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► JIRA INIT — NO BOARDS FOUND
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

No boards found for project {PROJECT_KEY}.

**Create a board:**

1. Go to: {SITE_URL}/jira/software/projects/{PROJECT_KEY}/boards
2. Click "Create board"
3. Select "Scrum" (recommended) or "Kanban"

Run `/gsd:jira-init` again after creating a board.
```

Exit command.

**Store selected board:**
- `board.id`
- `board.name`
- `board.type` (scrum/kanban)

Continue to step 6.

## 6. Detect Issue Types

Fetch project details with issue types:

```bash
acli jira project view --key {PROJECT_KEY} --json
```

**Handle API errors:**

**If network error or timeout:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► JIRA INIT — API ERROR
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Failed to fetch project details from Jira.

**Error:** {error message}

**Possible causes:**
- Network connectivity issues
- Jira API is temporarily unavailable
- Rate limiting

**Try again:**
Run `/gsd:jira-init` to retry.
```

Exit command.

**If JSON parse error:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► JIRA INIT — PARSE ERROR
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Failed to parse Jira API response.

This may indicate an incompatible acli version or API changes.

**Troubleshooting:**
1. Update acli: `brew upgrade jira-cli` or `go install github.com/ankitpokhrel/jira-cli/cmd/jira@latest`
2. Verify raw output: `acli jira project view --key {PROJECT_KEY} --json`
3. Report issue if persists

Run `/gsd:jira-init` to retry after updating.
```

Exit command.

**Parse JSON response:**

Extract `issueTypes` array with:
- `id` — Issue type ID
- `name` — Issue type name (Epic, Story, Task, Sub-task, Initiative, Bug, etc.)
- `subtask` — Boolean indicating if it's a subtask type

**If issueTypes array is empty or missing:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► JIRA INIT — NO ISSUE TYPES
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

No issue types found for project {PROJECT_KEY}.

This is unusual — most Jira projects have default issue types.

**Check:**
1. Project settings in Jira: {SITE_URL}/jira/software/projects/{PROJECT_KEY}/settings/issuetypes
2. Your account permissions for this project

Run `/gsd:jira-init` to retry after checking.
```

Exit command.

**Build available types list:**

Check for presence of GSD-relevant types:
- `Initiative` — For milestones (requires Advanced Roadmaps)
- `Epic` — For phases (fallback for milestones)
- `Story` — For plans
- `Task` — For tasks (fallback)
- `Sub-task` — For tasks (preferred)

**Resolve active mapping:**

Apply mapping rules with fallbacks:

| GSD Entity | Primary Type | Fallback Type |
|------------|--------------|---------------|
| Milestone  | Initiative   | Epic          |
| Phase      | Epic         | Story         |
| Plan       | Story        | Task          |
| Task       | Sub-task     | Task          |

For each GSD entity:
1. Check if primary type exists in available types
2. If not, use fallback type
3. If fallback also missing, set to `null` (will error on sync)

**Display resolved mapping:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► ISSUE TYPE MAPPING
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Available issue types in {PROJECT_KEY}:
{list of available types}

| GSD Entity | Jira Type | Source   |
|------------|-----------|----------|
| Milestone  | {type}    | primary/fallback |
| Phase      | {type}    | primary/fallback |
| Plan       | {type}    | primary/fallback |
| Task       | {type}    | primary/fallback |
```

**If any entity resolved to `null` (critical types missing):**

```
⚠ WARNING: Some required issue types are not available:

| GSD Entity | Needed         | Status  |
|------------|----------------|---------|
| [entity]   | [primary type] | MISSING |
| ...        | ...            | ...     |

**Impact:**
- Sync for missing types will be skipped
- Related GSD entities won't appear in Jira

**Options:**
1. Add missing issue types in Jira project settings
2. Continue with available types (some sync features will be limited)
```

Use AskUserQuestion:
- header: "Continue?"
- question: "Continue with available issue types?"
- options:
  - "Continue" — Use available types, some features limited
  - "Exit" — Fix issue types in Jira first

**If "Exit":**

```
**Add issue types in Jira:**

1. Go to: {SITE_URL}/jira/software/projects/{PROJECT_KEY}/settings/issuetypes
2. Click "Add issue type"
3. Add: Story, Epic, Sub-task (minimum required)

For Initiative (Advanced Roadmaps):
- Requires Jira Premium or higher
- Or use Epic as fallback for Milestones

Run `/gsd:jira-init` after adding issue types.
```

Exit command.

**Store resolved mapping:**
- `available_types[]` — All issue types from project
- `active.milestone` — Resolved type for milestones (or null)
- `active.phase` — Resolved type for phases (or null)
- `active.plan` — Resolved type for plans (or null)
- `active.task` — Resolved type for tasks (or null)

Continue to step 7.

## 7. Create Configuration Files

Create the `.planning/jira/` directory:

```bash
mkdir -p .planning/jira
```

**Write config.yaml:**

```yaml
# GSD-Jira Configuration
# Created by gsd:jira-init on {DATE}
# 
# This file stores your Jira connection settings.
# Edit manually only if you know what you're doing.

jira:
  # Your Atlassian site URL
  site: "{SITE_URL}"
  
  # Selected project for issue sync
  project:
    key: "{PROJECT_KEY}"
    name: "{PROJECT_NAME}"
    id: "{PROJECT_ID}"
  
  # Board for sprint management
  board:
    id: "{BOARD_ID}"
    name: "{BOARD_NAME}"
    type: "{BOARD_TYPE}"  # scrum or kanban

sync:
  # Sync direction: push = GSD → Jira (Phase 1-2)
  # Future: pull, bidirectional
  mode: "push"
  
  # Automatically sync on phase completion
  auto_sync: false
```

**Write mapping.yaml:**

```yaml
# GSD-Jira Entity Mapping
# Created by gsd:jira-init on {DATE}
#
# This defines how GSD entities map to Jira issue types.
# The mapping is auto-detected from your project's available types.
#
# IMPORTANT: Do not edit unless you understand the implications.
# Changing mappings after initial sync may cause orphaned issues.

# Mapping hierarchy (fixed, based on GSD structure):
#   Milestone → Initiative (or Epic fallback)
#   Phase     → Epic (or Story fallback)
#   Plan      → Story (or Task fallback)  
#   Task      → Sub-task (or Task fallback)

entities:
  milestone:
    primary: "Initiative"
    fallback: "Epic"
    description: "GSD Milestone = Jira Initiative/Epic"
  phase:
    primary: "Epic"
    fallback: "Story"
    description: "GSD Phase = Jira Epic/Story"
  plan:
    primary: "Story"
    fallback: "Task"
    description: "GSD Plan = Jira Story/Task"
  task:
    primary: "Sub-task"
    fallback: "Task"
    description: "GSD Task = Jira Sub-task/Task"

# Issue types available in project {PROJECT_KEY}
available_types:
{AVAILABLE_TYPES_LIST}

# Resolved active mapping based on available types
# These are the actual issue types that will be used for sync
active:
  milestone: {ACTIVE_MILESTONE}  # {MILESTONE_NOTE}
  phase: {ACTIVE_PHASE}          # {PHASE_NOTE}
  plan: {ACTIVE_PLAN}            # {PLAN_NOTE}
  task: {ACTIVE_TASK}            # {TASK_NOTE}
```

## 8. Show Summary

Display success message with configuration summary:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► JIRA INITIALIZED ✓
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Configuration saved to `.planning/jira/`**

| Setting        | Value                           |
|----------------|---------------------------------|
| Site           | {SITE_URL}                      |
| Project        | {PROJECT_KEY} — {PROJECT_NAME}  |
| Board          | {BOARD_NAME} ({BOARD_TYPE})     |

**Issue Type Mapping:**

| GSD Entity | Jira Type       | Status |
|------------|-----------------|--------|
| Milestone  | {ACTIVE_MILESTONE} | ✓   |
| Phase      | {ACTIVE_PHASE}     | ✓   |
| Plan       | {ACTIVE_PLAN}      | ✓   |
| Task       | {ACTIVE_TASK}      | ✓   |

**Files created:**
- `.planning/jira/config.yaml`
- `.planning/jira/mapping.yaml`

───────────────────────────────────────────────────────────────

## ▶ Next Steps

**Push current GSD state to Jira:**

/gsd:jira-sync

<sub>This will create issues in Jira for your phases and plans.</sub>

───────────────────────────────────────────────────────────────

**Jira Project:** {SITE_URL}/jira/software/projects/{PROJECT_KEY}/boards/{BOARD_ID}
```

</process>

<success_criteria>
- [ ] acli installation verified
- [ ] acli authentication verified
- [ ] Jira project selected (from list or by key)
- [ ] Board selected or auto-detected
- [ ] Issue types detected and mapping resolved
- [ ] `.planning/jira/config.yaml` created with site, project, board
- [ ] `.planning/jira/mapping.yaml` created with available types and active mapping
- [ ] Summary displayed with configuration and next steps
</success_criteria>
