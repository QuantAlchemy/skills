---
name: do-parallel-work
description: Parallel task queue - orchestrate work across multiple sub-agents
argument-hint: run | (task to capture) | verify | cleanup | status
---

# Do-Parallel-Work Skill

A parallel execution task queue. You are the **orchestrator** — you capture tasks, analyze dependencies, and spawn sub-agents to do the actual work. Multiple independent tasks run in parallel.

**Key difference from do-work:** This skill spawns sub-agents for implementation and runs independent tasks concurrently.

**Actions:**
- **do**: Capture new tasks/requests → creates UR folder + REQ files (same as do-work)
- **work**: Process pending requests → analyzes dependencies, spawns parallel sub-agents
- **verify**: Evaluate captured REQs against original input → quality check
- **cleanup**: Consolidate archive → moves loose REQs into UR folders, closes completed URs
- **status**: Show current queue state, in-flight agents, and progress

> **Core concept:** You are the orchestrator. You manage the queue, track dependencies, spawn sub-agents, and monitor progress. Sub-agents do the actual implementation work.

## Architecture

```
Orchestrator (you)
    │
    ├── Scan queue for pending REQs
    │
    ├── Analyze dependencies
    │   ├── REQ-001: no deps → batch 1
    │   ├── REQ-002: no deps → batch 1
    │   └── REQ-003: depends_on: [REQ-001] → batch 2
    │
    ├── Spawn sub-agents for batch 1 (parallel)
    │   ├── sessions_spawn → REQ-001
    │   └── sessions_spawn → REQ-002
    │
    ├── Monitor completions (poll sessions_list)
    │
    ├── As agents complete:
    │   ├── Archive completed REQs
    │   └── Check if blocked REQs are now unblocked
    │
    └── Spawn next batch, repeat until done
```

## Inbox Folder (External Input)

You can drop files into an inbox folder while the orchestrator is working:

```
/path/to/inbox/           ← Drop .md files here anytime
└── new-feature.md        ← Plain text, any format
```

The work action checks the inbox before each batch:
1. Read new files from inbox
2. Process them through the "do" action (create UR + REQs)
3. Move/delete the inbox file
4. Include new REQs in dependency analysis

This allows async task addition without interrupting in-flight work.

## Routing Decision

### Step 1: Parse the Input

Examine what follows "do parallel work":

| Pattern | Example | Route |
|---------|---------|-------|
| Empty or bare invocation | `do parallel work` | → Ask: "Start the work loop?" |
| Action verbs only | `do parallel work run` | → work |
| Verify keywords | `do parallel work verify` | → verify |
| Cleanup keywords | `do parallel work cleanup` | → cleanup |
| Status keywords | `do parallel work status` | → status |
| Descriptive content | `do parallel work add dark mode` | → do |

### Action Verbs (→ Work)
run, go, start, begin, work, process, execute, build, continue, resume

### Verify Verbs (→ Verify)
verify, check, evaluate, review requests, audit

### Cleanup Verbs (→ Cleanup)
cleanup, clean up, tidy, consolidate, organize archive

### Status Verbs (→ Status)
status, progress, show, list, queue

### Content Signals (→ Do)
- Descriptive text beyond a single verb
- Feature requests, bug reports, ideas
- "add", "create", "I need", "we should"

## Request File Schema

Same as do-work, with additional fields for parallel execution:

```yaml
---
id: REQ-001
title: Short descriptive title
status: pending | claimed | in_progress | completed | failed
created_at: 2025-01-26T10:00:00Z
user_request: UR-001

# Parallel execution fields
depends_on: [REQ-002, REQ-003]  # Optional: blocks until these complete
priority: normal | high | low    # Optional: affects batch ordering
assigned_agent: <session-key>    # Set when spawned to sub-agent

# Set by work action
claimed_at: 2025-01-26T10:30:00Z
route: A | B | C
completed_at: 2025-01-26T10:45:00Z
---
```

## Dependency Rules

1. **No `depends_on`** → can run immediately, parallelizable with other independent REQs
2. **`depends_on: [REQ-X]`** → waits until REQ-X status is `completed`
3. **Circular dependencies** → error, report to user
4. **Missing dependency** → error if REQ-X doesn't exist

## Action References

Follow the detailed instructions in:
- [do action](./actions/do.md) - Request capture (same as do-work)
- [work action](./actions/work.md) - Parallel queue processing
- [verify action](./actions/verify.md) - Quality evaluation
- [cleanup action](./actions/cleanup.md) - Archive consolidation
- [status action](./actions/status.md) - Queue and progress display

## Configuration

Set these in your workspace or pass to the work action:

```yaml
# Maximum concurrent sub-agents
max_parallel: 3

# Inbox folder location (for external task drops)
inbox_path: /home/user/gdrive-tasks/inbox

# Plans folder (for master documents)
plans_path: /home/user/gdrive-tasks/plans

# Poll interval for checking sub-agent completion (ms)
poll_interval: 5000
```

## Example Session

```
User: do parallel work run

Orchestrator: Checking queue...
Found 5 pending requests.

Analyzing dependencies:
  REQ-001: no dependencies
  REQ-002: no dependencies  
  REQ-003: depends on REQ-001
  REQ-004: no dependencies
  REQ-005: depends on REQ-003, REQ-004

Batch 1 (parallel): REQ-001, REQ-002, REQ-004
  Spawning sub-agent for REQ-001... ✓
  Spawning sub-agent for REQ-002... ✓
  Spawning sub-agent for REQ-004... ✓

Monitoring... (3 agents in flight)

  ✓ REQ-002 completed (2m 15s)
  ✓ REQ-004 completed (3m 02s)
  ✓ REQ-001 completed (4m 30s)

Batch 2: REQ-003 (was blocked by REQ-001)
  Spawning sub-agent for REQ-003... ✓

Monitoring... (1 agent in flight)

  ✓ REQ-003 completed (2m 45s)

Batch 3: REQ-005 (was blocked by REQ-003, REQ-004)
  Spawning sub-agent for REQ-005... ✓

Monitoring... (1 agent in flight)

  ✓ REQ-005 completed (5m 10s)

All 5 requests completed:
  - REQ-001 (4m 30s)
  - REQ-002 (2m 15s)
  - REQ-003 (2m 45s)
  - REQ-004 (3m 02s)
  - REQ-005 (5m 10s)

Total wall time: 12m 25s (vs ~17m 42s serial)
```
