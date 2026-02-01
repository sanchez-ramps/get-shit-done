# Phase 1: Foundation - Context

**Gathered:** 2026-02-01
**Status:** Ready for planning

<domain>
## Phase Boundary

Establish configuration schema, CLI scaffold, and acli wrapper patterns for Jira integration. This phase delivers:
1. Configuration files (`.planning/jira/mapping.yaml`, `.planning/jira/config.yaml`)
2. CLI command (`gsd:jira-init`)
3. acli wrapper patterns with authentication verification

Creating Jira projects, syncing data, or git integration are NOT part of this phase.

</domain>

<decisions>
## Implementation Decisions

### Init command flow
- Minimal prompts — only ask for essentials (project, board), infer the rest
- List available projects + accept key directly for speed (user can type key if known)
- If no Jira project exists, guide user to create one in Jira first, offer to retry
- Re-init is additive — only update what's specified, keep existing config
- Rely on existing acli authentication — assume acli is already configured, just verify it works
- For boards: auto-detect default, but if multiple boards exist, list and ask for confirmation

### Configuration structure
- Two files — mapping.yaml separate from config.yaml (cleaner separation)
- Fixed entity mapping — Milestone=Initiative, Phase=Epic, Plan=Story (no user config needed)
- Auto-detect status mapping — read Jira workflow, find matching statuses
- Git-related settings (branch prefix, smart commits) deferred to Phase 2/3

### Error messaging
- Translate acli errors into friendlier messages with fix suggestions
- Validate upfront — verify acli works before asking any questions
- On success: show summary of what was configured + link to Jira project + next steps
- Network failures: explain what failed, offer to retry with guidance

### Default behaviors
- Map alternatives for issue types — if Story doesn't exist, use Task; if Sub-task doesn't exist, use available alternatives
- Use Epic for Milestone if Initiative not available (Advanced Roadmaps not present) — Phase becomes Story
- Always ask for board selection if multiple boards exist
- Auto-push as default sync mode — push to Jira when milestones/phases created

### Claude's Discretion
- Exact acli wrapper implementation patterns
- Config file field ordering and comments
- Specific retry logic for network failures
- Error message wording

</decisions>

<specifics>
## Specific Ideas

- Init should feel quick — minimal prompts, smart detection
- Don't create Jira projects via CLI — too complex, guide user to Jira UI instead
- Additive re-init means users can run init again to update specific settings without losing everything

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 01-foundation*
*Context gathered: 2026-02-01*
