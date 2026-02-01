# Pitfalls Research: GSD/Jira Integration

**Date:** 2026-01-31  
**Project:** GSD-Jira Integration (CLI wrapping acli, bidirectional sync)

---

## Critical Pitfalls

### 1. Eventual Consistency - Data Sync Delays
**What happens:** Jira Cloud uses an eventually consistent API model. After a write operation, subsequent search queries may return stale data for seconds to minutes. Operations affecting numerous issues (bulk edits, reordering) take longer. Without `reconcileIssues` parameter, searches may miss recently created/updated items.

**Warning signs:**
- Newly created issues don't appear in searches
- Status changes don't reflect immediately after sync
- Bulk operations show partial results
- Integration tests fail intermittently
- "Issue not found" errors for just-created items

**Prevention:**
- Implement "Search and Reconcile" pattern from Atlassian
- Add configurable delay after writes before reads
- Use issue keys for direct lookups instead of JQL searches
- Cache issue IDs locally after creation for immediate re-access
- Design sync workflow to tolerate temporary inconsistency

**Phase mapping:** Implementation phase - build consistency handling into sync engine core

---

### 2. CLI Wrapper Process Management Issues
**What happens:** Wrapping acli via subprocess introduces reliability issues. Stdout/stderr buffering can cause deadlocks (~100KB buffer). Mixed error messages and data in stdout corrupt parsing. Non-zero exit codes don't always indicate actual failures. Shell escaping issues corrupt data with special characters.

**Warning signs:**
- Process hangs during large operations
- Intermittent parse errors on output
- Commands work manually but fail in wrapper
- Special characters in issue titles cause failures
- Inconsistent error handling across commands

**Prevention:**
- Use `subprocess.communicate()` to prevent deadlock
- Capture stdout and stderr separately using threading
- Use `--json` output format from acli for reliable parsing
- Implement proper shell escaping for all user input
- Parse exit codes AND output content for error detection
- Add timeouts to all subprocess calls
- Test with Unicode, quotes, backticks in issue titles

**Phase mapping:** Implementation phase - establish robust CLI wrapper patterns early

---

### 3. Race Conditions in Multi-Developer Sync
**What happens:** Multiple developers sync simultaneously from different branches. Last write wins, clobbering intermediate updates. Automation rules fail to trigger when multiple systems modify same element. Only one listener captures a single trigger event.

**Warning signs:**
- Updates disappear after merging branches
- Field values revert unexpectedly
- Duplicate issues created for same GSD item
- Sync state file has merge conflicts
- "Issue was modified" warnings ignored

**Prevention:**
- Use Jira's issue ETag/version for optimistic locking
- Implement "check-then-act" with retry on conflict
- Search by GSD identifier before creating issues
- Use append-only sync log instead of mutable state file
- Establish "pull before push" developer workflow
- Add pre-commit hooks to detect stale sync state

**Phase mapping:** Design phase - choose conflict strategy before implementation

---

### 4. Missing Required Fields in API Calls
**What happens:** HTTP 400 errors when creating issues. Fields marked required in Jira project configuration aren't discovered. Custom fields (customfield_xxxxx) have different requirements per project/issue type. Field requirements change when project admins modify configurations.

**Warning signs:**
- 400 errors on issue creation
- Different behavior between projects
- Fields required in UI but not documented
- "Field X is required" errors for unexpected fields
- Issue creation works once then fails later

**Prevention:**
- Query `/rest/api/3/issue/createmeta` dynamically
- Use `expand=projects.issuetypes.fields` for full schema
- Validate field configuration per project before bulk operations
- Cache field requirements with TTL, refresh periodically
- Handle field schema differences across issue types
- Log full error response for debugging

**Phase mapping:** Implementation phase - build robust field discovery system

---

### 5. Webhook Event Ordering Not Guaranteed
**What happens:** Jira 10+ uses asynchronous webhooks by default. Events may arrive out of order. Status transition from "To Do" → "In Progress" → "Done" may arrive as Done before In Progress. Missed webhooks leave sync state inconsistent.

**Warning signs:**
- Status appears to skip transitions
- Sync state shows impossible state sequences
- Webhook payloads reference unknown issues
- Events processed but state is wrong
- Intermittent "issue not found" during sync

**Prevention:**
- Don't rely on event ordering for state machine logic
- Use polling to verify final state after webhook
- Store webhook events with timestamps, reconcile periodically
- Implement idempotent event handlers
- Design for eventual consistency, not immediate consistency
- Use issue `updated` timestamp as canonical ordering

**Phase mapping:** Design phase - plan event handling architecture

---

### 6. Sprint State Transition Errors
**What happens:** Jira sprint states (future → active → closed) have specific rules. Can't move issues to closed sprints. Can't have multiple active sprints without specific board configuration. Moving issues between sprints fails silently when conditions aren't met.

**Warning signs:**
- "Sprint not found" for valid sprint IDs
- Issues don't move to expected sprint
- Sprint completion fails with no clear error
- Sprint operations work in UI but fail via API
- Board configuration affects API behavior

**Prevention:**
- Query sprint state before operations
- Check board configuration for multi-sprint support
- Use correct Jira Agile endpoints (`/rest/agile/1.0/`)
- Handle "sprint is closed" errors gracefully
- Implement sprint state validation in sync workflow
- Test sprint operations against actual board configuration

**Phase mapping:** Implementation phase - build sprint state machine awareness

---

### 7. Rate Limiting Under Points-Based Model (Effective March 2026)
**What happens:** Starting March 2, 2026, Jira Cloud enforces points-based rate limiting. Read operations cost more points than writes. Large JQL queries consume many points. Bulk operations can exhaust quota quickly. HTTP 429 responses require exponential backoff.

**Warning signs:**
- 429 responses during normal operations
- Sync slows down dramatically mid-operation
- Different performance between test and production
- Large syncs fail partway through
- Read-heavy operations fail more than writes

**Prevention:**
- Implement robust 429 handling with exponential backoff
- Batch operations to reduce API calls
- Use pagination with reasonable page sizes (50-100)
- Monitor point usage via response headers
- Prefer writes (1 point) over reads (variable points)
- Request quota increase for sustained high usage
- Test with production-like data volumes

**Phase mapping:** Implementation phase - build rate limiting from day one

---

## Moderate Pitfalls

### 8. Bidirectional Field Sync Conflicts
**What happens:** Bidirectional sync on same field is not supported by most integrations. Many-to-one field mappings fail silently. If one mapped field is misconfigured, all mapped fields fail in the same API call.

**Warning signs:**
- Fields sync one direction only
- Multiple field updates fail together
- Sync stops without clear error
- Field values don't match between systems

**Prevention:**
- Avoid bidirectional sync on same field
- Use clear "source of truth" per field
- Separate field mappings to prevent cascading failures
- Test each field mapping independently
- Document sync direction explicitly per field

**Phase mapping:** Design phase - define field ownership strategy

---

### 9. Permission and Configuration Silent Failures
**What happens:** Over 33% of automation failures trace to permission issues. Automation actor lacks access to required fields or issue types. Rules fail silently - no errors logged. Different behavior between projects with different permission schemes.

**Warning signs:**
- Rules work in one project, fail in another
- No error logs but operations don't execute
- Status transitions have no effect
- Field updates don't persist

**Prevention:**
- Verify automation account permissions before deployment
- Use dedicated service account with documented permissions
- Test with production-like permission configuration
- Check project-level permissions, not just global
- Enable audit logging to detect silent failures

**Phase mapping:** Testing phase - permission verification checklist

---

### 10. Workflow Action Invalid Errors
**What happens:** Moving issues on sprint board triggers "Workflow Action Invalid" even though the issue sometimes transitions successfully. Corrupted transitions, missing transitions, or failed validators cause these errors. Status maps to board column but workflow doesn't support the transition path.

**Warning signs:**
- Error dialogs when transitioning issues
- Status changes but column doesn't update
- Some issues transition, others fail
- Errors mention unknown transition IDs

**Prevention:**
- Query workflow for available transitions
- Don't assume transition paths exist
- Validate workflow maps to board columns
- Handle "no valid transition" gracefully
- Use transition IDs, not transition names

**Phase mapping:** Implementation phase - build workflow-aware transition logic

---

### 11. acli Output Parsing Fragility
**What happens:** acli output format may change between versions. Table-formatted output is hard to parse reliably. Error messages mixed with data output. Different output formats between commands.

**Warning signs:**
- Parse errors after acli update
- Different results between developers
- Regex patterns fail on edge cases
- Output truncated or wrapped unexpectedly

**Prevention:**
- Always use `--json` output format
- Pin acli version in project requirements
- Validate JSON output schema
- Handle empty results explicitly
- Test with various data shapes (empty, one, many)

**Phase mapping:** Implementation phase - standardize output parsing

---

### 12. Max Result Window Pagination Limit
**What happens:** JQL searches hit 10,000 result limit. Attempting to paginate beyond this returns 500 errors. Increasing the limit causes memory issues and potential node instability.

**Warning signs:**
- 500 errors on large pagination offsets
- "Search limit exceeded" messages
- Inconsistent results across pages
- Performance degradation on large queries

**Prevention:**
- Add date/criteria filters to reduce result sets
- Split queries by project if needed
- Paginate using issue ID for single-project queries
- Implement incremental sync (last N days, not all time)
- Monitor result counts before full iteration

**Phase mapping:** Implementation phase - design scalable query patterns

---

### 13. SSO Authentication Compatibility
**What happens:** CLI tools fail with 401 Unauthorized in SSO environments. Basic email/password auth doesn't work with SSO. Many CLI tools don't handle OAuth/SSO flows properly.

**Warning signs:**
- 401 errors with correct credentials
- Authentication works in browser but not CLI
- Token refresh fails
- Intermittent auth failures

**Prevention:**
- Use API tokens exclusively (Bearer auth)
- Generate tokens from id.atlassian.com/manage-profile/security/api-tokens
- Store tokens in environment variables, never config files
- Implement token validation on startup
- Document token generation steps clearly

**Phase mapping:** Setup phase - establish auth patterns first

---

### 14. Field Configuration New Mapping Not Triggering Sync
**What happens:** Adding new field mapping doesn't automatically sync existing issues. This is by design - only future changes trigger sync. Users expect retroactive sync that doesn't happen.

**Warning signs:**
- New field mapping shows no effect
- Existing issues unchanged after config update
- Only new issues have correct field values
- Confusion about when fields sync

**Prevention:**
- Document that field mapping is future-only
- Provide explicit "resync" command for existing issues
- Warn users during field mapping configuration
- Implement bulk update utility for retroactive sync

**Phase mapping:** Design phase - plan resync workflow

---

## Minor Pitfalls

### 15. Overcomplicated Automation Rules
**What happens:** Overlapping triggers cause cascading failures (68% of automation failures). Rules with multiple objectives have 42% higher error rates. Unused automations accumulate and cause system lag.

**Warning signs:**
- Multiple rules on same event
- Rules with many actions
- Audit logs show repeated failures
- System lag when processing issues

**Prevention:**
- Single-purpose rules only
- Quarterly rule audits
- Disable unused automations
- Test one change at a time

**Phase mapping:** Design phase - establish rule architecture

---

### 16. Default Status Assumptions
**What happens:** Code assumes statuses like "In Progress" and "Resolved" exist. Projects with custom workflows fail when statuses don't match. Silent failures when transitioning to non-existent status.

**Warning signs:**
- Transitions fail on some projects
- Status names don't match across projects
- Hardcoded status values in code
- Tests pass with default workflow only

**Prevention:**
- Query workflow for available statuses
- Map GSD statuses to project-specific statuses
- Handle missing statuses gracefully
- Use status IDs, not names when possible

**Phase mapping:** Implementation phase - dynamic status discovery

---

### 17. Time Zone Handling in Sprint Dates
**What happens:** Sprint start/end dates interpreted in wrong time zone. Off-by-one day errors on sprint boundaries. Issues appear in wrong sprint based on timezone.

**Warning signs:**
- Sprint dates off by a day
- Issues assigned to wrong sprint
- Date calculations inconsistent
- Different behavior for different users

**Prevention:**
- Use ISO 8601 format with explicit timezone
- Store all dates in UTC internally
- Convert to local time only for display
- Test with non-UTC timezone users

**Phase mapping:** Implementation phase - date handling standards

---

### 18. Cluster Lock Timeouts in Large Syncs
**What happens:** Scheduled sync jobs take too long and block subsequent syncs. Lock health check fails on `syncRepositoryList`. Jobs pile up waiting for locks.

**Warning signs:**
- Sync jobs queue up
- Health check warnings
- Sync takes longer over time
- Jobs timeout without completing

**Prevention:**
- Break large syncs into smaller batches
- Increase sync interval for long-running jobs
- Implement progress checkpointing
- Monitor job duration trends

**Phase mapping:** Operations phase - performance monitoring

---

### 19. Incorrect API Base URL Formatting
**What happens:** Double-appending paths causes 405 Method Not Allowed. Base URL includes `/rest/api/2` when it shouldn't. Different URL formats between Cloud and Server.

**Warning signs:**
- 405 errors on operations
- URLs have duplicate segments
- Different behavior per environment
- Auth works but operations fail

**Prevention:**
- Validate URL configuration format
- Document URL format clearly
- Test URL construction in setup wizard
- Provide URL format examples

**Phase mapping:** Setup phase - configuration validation

---

### 20. Git Merge Conflicts in Mapping Files
**What happens:** Multiple branches modify sync state file. Merge conflicts corrupt mapping state. Lost issue references after merge resolution.

**Warning signs:**
- Git merge conflicts in .planning/jira/
- Lost issue mappings
- Duplicate entries after merge
- Orphaned Jira references

**Prevention:**
- Use append-only format (one entry per line)
- Implement merge-friendly JSON structure
- Add Git hooks for mapping validation
- Provide mapping repair utility

**Phase mapping:** Design phase - choose merge-friendly formats

---

## "Looks Done But Isn't" Checklist

Use this checklist before declaring any integration phase complete:

### Setup Phase
- [ ] API tokens work from developer machines AND CI/CD
- [ ] Authentication tested with SSO-enabled Jira instance
- [ ] Service account has permissions on ALL target projects
- [ ] Token storage follows security best practices (env vars, not files)
- [ ] acli version pinned and documented

### Design Phase
- [ ] Field mapping documented with explicit sync direction per field
- [ ] Conflict resolution strategy chosen AND documented
- [ ] Sprint state transitions mapped to GSD workflow
- [ ] Rate limit handling designed (not just "we'll add backoff")
- [ ] Multi-developer workflow documented with examples

### Implementation Phase
- [ ] All subprocess calls have timeouts
- [ ] 429 handling tested with actual rate limit conditions
- [ ] Pagination tested beyond single page (>50 results)
- [ ] Unicode/special characters tested in all text fields
- [ ] Empty result handling tested (0 issues, 0 sprints)
- [ ] Error messages include actionable guidance

### Testing Phase
- [ ] Tested with production-like permissions (not admin account)
- [ ] Tested with custom workflows (not default project template)
- [ ] Tested with required custom fields configured
- [ ] Concurrent sync from multiple developers tested
- [ ] Sprint state transitions tested (future→active→closed)
- [ ] Large sync tested (100+ issues)

### Integration Phase
- [ ] Webhook delivery verified (not just configured)
- [ ] Eventual consistency delays tested (read-after-write)
- [ ] Rate limits tested under production-like load
- [ ] Error logging captures enough detail to diagnose issues
- [ ] Recovery from partial sync failure tested

### Operations Phase
- [ ] Monitoring/alerting configured for sync failures
- [ ] Runbook exists for common failure scenarios
- [ ] Token rotation procedure documented and tested
- [ ] Cleanup procedure for orphaned issues documented
- [ ] Performance baseline established

---

## API Deprecation Timeline

### Already Deprecated (Migration Required)
- **Username/userKey parameters** → Use `accountId` instead
- **Non-paginated endpoints** → Use `/project/search` and `/filter/search`
- **Epic Link field in changelogs** → Use `IssueParentAssociation`

### Upcoming Changes
- **March 2026:** Points-based rate limiting enforcement begins
- **July 2026:** Field configuration schemes APIs removed (migrate to "field schemes")

### acli-Specific Considerations
- Monitor Atlassian CLI changelog for breaking changes
- Pin to specific version in project
- Test after each acli update before production deployment

---

## Recovery Strategies

### Sync State Corruption
1. Export current Jira state via API
2. Compare with local GSD state
3. Regenerate sync-state.json from comparison
4. Validate regenerated state before resuming sync

### Duplicate Issue Cleanup
1. Query Jira for issues with GSD identifier label
2. Identify duplicates by GSD ID
3. Choose canonical issue (earliest created)
4. Move work/comments from duplicates
5. Close duplicates with "Duplicate" resolution

### Rate Limit Exhaustion
1. Stop all sync operations immediately
2. Wait for rate limit window to reset (check response headers)
3. Resume with reduced batch size
4. Consider requesting quota increase

### Authentication Failure Recovery
1. Verify token hasn't expired
2. Generate new API token if needed
3. Update token in environment variables
4. Verify permissions haven't changed
5. Test with single API call before full sync

---

## Research Sources

- Atlassian Developer Documentation (rate limiting, webhooks)
- Atlassian Community (automation pitfalls, workflow issues)
- Atlassian Support KBs (webhook troubleshooting, sync issues)
- SpecSync Documentation (conflict resolution patterns)
- Exalate Documentation (bidirectional sync challenges)
- Stack Overflow (subprocess patterns, CLI wrappers)

---

*Research completed: 2026-01-31*
*Quality gates: All checked*
