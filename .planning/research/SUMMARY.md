# Research Summary: GSD-Jira Integration

**Domain:** CLI tooling for bidirectional sync between GSD workflow artifacts and Jira Cloud  
**Researched:** 2026-01-31  
**Overall Confidence:** HIGH

## Executive Summary

The GSD-Jira Integration project sits in a market gap between simple one-way sync tools and complex enterprise platforms (Exalate, Unito). By providing developer-first bidirectional sync without enterprise complexity, this integration can offer significant value to teams using GSD for planning while needing Jira visibility.

The architecture centers on wrapping `acli` (Atlassian CLI) via subprocess rather than building a custom Jira REST client. This constraint simplifies auth handling and reduces maintenance burden, though it introduces CLI wrapper challenges (process management, output parsing, error handling).

Critical technical challenges include Jira's eventual consistency model (writes may not be immediately visible in searches), race conditions in multi-developer sync scenarios, and the complexity of mapping GSD's markdown-based artifacts to Jira's structured issue hierarchy (Initiative → Epic → Story → Sub-task).

## Key Findings

**Stack:** Python 3.13 + Typer CLI framework + SQLite (via sqlite-utils) for sync state + Rich for terminal output. Wrap `acli` via subprocess with JSON output parsing.

**Table Stakes:** Authentication, unidirectional push sync (GSD → Jira), status synchronization, conflict detection, basic CLI commands (`sync`, `status`, `init`).

**Differentiators:** Full bidirectional sync, auto-import of Jira-created issues, automatic sprint lifecycle management, conflict resolution flow.

**Critical Pitfall:** Jira's eventual consistency means searches may return stale data after writes. Build retry logic and "search and reconcile" patterns into sync engine from day one.

## Implications for Roadmap

Based on research, suggested phase structure:

1. **Foundation & acli Integration** — CLI scaffold, acli wrapper with robust error handling, config management
   - Addresses: Auth, connection validation, acli subprocess patterns
   - Avoids: CLI wrapper process deadlocks (Critical Pitfall #2)

2. **GSD Parser & Mapping** — Parse GSD artifacts, define entity mapping to Jira hierarchy
   - Addresses: Milestone→Initiative, Phase→Epic, Plan→Story, Task→Sub-task mapping
   - Avoids: Field mapping inconsistencies (Moderate Pitfall)

3. **Push Sync (GSD → Jira)** — Create/update Jira issues from GSD artifacts
   - Addresses: Table stakes TS-2, TS-3
   - Avoids: Missing required fields (Critical Pitfall #4)

4. **Pull Sync & Bidirectional** — Jira → GSD sync, bidirectional conflict detection
   - Addresses: Differentiator D-2, D-3
   - Avoids: Race conditions (Critical Pitfall #3)

5. **Conflict Resolution** — Manual conflict resolution flow
   - Addresses: TS-4 (Conflict Detection)
   - Avoids: Data loss from automated resolution

6. **Sprint Management** — Auto sprint lifecycle, completed item assignment
   - Addresses: Differentiator D-5
   - Avoids: Sprint state transitions (Critical Pitfall #6)

7. **GSD Command Integration** — Auto-sync hooks in existing GSD commands
   - Addresses: Seamless developer workflow
   - Avoids: Manual sync burden

**Phase ordering rationale:**
- Foundation must come first (auth, CLI wrapper patterns)
- Push sync before pull (simpler, validates acli integration)
- Conflict detection required before bidirectional sync
- Sprint management depends on working sync engine

**Research flags for phases:**
- Phase 1 (Foundation): Needs acli documentation verification
- Phase 4 (Bidirectional): Complex, may need phase-specific research on conflict strategies
- Phase 6 (Sprint): Jira sprint API has edge cases, verify with acli capabilities

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | Python/Typer well-documented, acli verified via official docs |
| Features | HIGH | Clear table stakes vs differentiators based on market analysis |
| Architecture | MEDIUM | CLI wrapper patterns need validation during implementation |
| Pitfalls | HIGH | Well-documented Jira gotchas, actionable prevention strategies |

## Gaps to Address

- **acli sprint commands:** Need to verify exact command syntax for sprint management
- **Initiative issue type:** Requires Jira Advanced Roadmaps — need fallback for projects without it
- **Markdown parsing:** GSD artifact format varies — need to standardize parser expectations
- **Multi-project sync:** Out of scope for v1, but architecture should not preclude it

## Files Created

| File | Lines | Purpose |
|------|-------|---------|
| STACK.md | 629 | Technology recommendations with acli details |
| FEATURES.md | 471 | Feature landscape with prioritization |
| ARCHITECTURE.md | 823 | System design, components, data flow |
| PITFALLS.md | 532 | Domain-specific pitfalls with prevention |

---
*Research complete: 2026-01-31*
