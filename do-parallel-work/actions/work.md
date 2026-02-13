# Work Action (Parallel)

> **Part of the do-parallel-work skill.** Processes requests from the queue using parallel sub-agent execution.

You are the **orchestrator**. You do NOT implement tasks yourself. You:
1. Analyze the queue and dependencies
2. Spawn sub-agents for independent tasks (in parallel)
3. Monitor their progress
4. Archive completed work
5. Unblock dependent tasks
6. Repeat until queue is empty

## Architecture Overview

```
You (Orchestrator)
    │
    ├─── [1] Check inbox for new items
    │
    ├─── [2] Scan queue, build dependency graph
    │
    ├─── [3] Identify parallelizable batch
    │         (REQs with no unmet dependencies)
    │
    ├─── [4] For each REQ in batch:
    │         └── sessions_spawn → sub-agent
    │
    ├─── [5] Monitor loop:
    │         ├── sessions_list to check status
    │         ├── On completion: archive, update deps
    │         └── Check inbox for new items
    │
    └─── [6] When batch done, go to step 2
```

## Workflow

### Step 0: Configuration

Check for configuration. Default values:

```
max_parallel: 3          # Max concurrent sub-agents
inbox_path: null         # If set, check for new items here
poll_interval: 5000      # Ms between status checks
```

Configuration can come from:
- Workspace config file
- Environment variables
- Passed as arguments to work action

### Step 1: Check Inbox

If `inbox_path` is configured:

1. List `.md` files in inbox folder
2. For each file:
   a. Read content
   b. Run the **do action** to create UR + REQs
   c. Delete (or move) the inbox file
3. New REQs are now in the queue

```python
# Pseudocode
for file in inbox_path/*.md:
    content = read(file)
    do_action(content)  # Creates UR + REQs
    delete(file)
```

### Step 2: Scan Queue and Build Dependency Graph

1. List all `REQ-*.md` files in `do-work/` (pending)
2. List all `REQ-*.md` files in `do-work/working/` (in-flight)
3. For each pending REQ, read frontmatter to get:
   - `id`
   - `depends_on` (array of REQ IDs, may be empty/absent)
   - `priority` (optional)

Build a graph:
```
{
  "REQ-001": { depends_on: [], status: "pending" },
  "REQ-002": { depends_on: [], status: "pending" },
  "REQ-003": { depends_on: ["REQ-001"], status: "pending" },
  "REQ-004": { depends_on: [], status: "in_progress", agent: "session-key" },
  "REQ-005": { depends_on: ["REQ-003", "REQ-004"], status: "pending" }
}
```

### Step 3: Identify Parallelizable Batch

A REQ is **ready** if:
- Status is `pending`
- All items in `depends_on` have status `completed`
- (Or `depends_on` is empty/absent)

```python
# Pseudocode
ready = []
for req in pending_reqs:
    deps = req.depends_on or []
    if all(dep.status == "completed" for dep in deps):
        ready.append(req)
```

Sort by priority (high → normal → low), then by ID.

Limit to `max_parallel - currently_in_flight` items.

### Step 4: Spawn Sub-Agents

For each ready REQ:

1. **Claim it** (move to working/, update frontmatter)
2. **Triage** (determine Route A/B/C)
3. **Spawn sub-agent** using `sessions_spawn`

#### Claiming

```bash
mkdir -p do-work/working
mv do-work/REQ-XXX.md do-work/working/
```

Update frontmatter:
```yaml
status: in_progress
claimed_at: 2025-01-26T10:30:00Z
route: B
assigned_agent: <will be set after spawn>
```

#### Triage (Same as do-work)

- **Route A (Simple):** Bug fix, config change, single file edit
- **Route B (Medium):** Clear outcome, needs exploration
- **Route C (Complex):** Multi-system, architectural, ambiguous

#### Spawning

Use `sessions_spawn` to create a background sub-agent:

```
sessions_spawn(
  task: <implementation prompt based on route>,
  label: "REQ-001-dark-mode",
  agentId: <optional, use default>,
  runTimeoutSeconds: 1800  # 30 min max
)
```

**Route A prompt:**
```
You are implementing this request:

## Request
[Full content of request file]

## Instructions
Implement this change. This was triaged as simple (Route A).
When complete, provide a summary of what was changed.
```

**Route B prompt:**
```
You are implementing this request:

## Request
[Full content of request file]

## Instructions
First explore the codebase to find relevant patterns and files.
Then implement the change following existing conventions.
When complete, provide a summary of what was changed.
```

**Route C prompt:**
```
You are implementing this request:

## Request
[Full content of request file]

## Instructions
1. First, create an implementation plan
2. Then explore the codebase for relevant patterns
3. Implement according to your plan
4. Run tests if available

When complete, provide a summary of:
- Your plan
- What was changed
- Any follow-up items
```

After spawn, update frontmatter with the session key:
```yaml
assigned_agent: agent:main:subagent:abc123
```

### Step 5: Monitor Loop

Poll for completion using `sessions_list`:

```python
while in_flight_count > 0:
    sleep(poll_interval)
    
    # Check inbox for new items
    check_inbox()
    
    # Check sub-agent status
    sessions = sessions_list(messageLimit=1)
    
    for session in sessions:
        if session.key in assigned_agents:
            if session_is_complete(session):
                handle_completion(session)
            elif session_has_error(session):
                handle_failure(session)
```

#### Detecting Completion

A sub-agent is complete when:
- Its last message indicates completion (contains summary)
- Or it has been idle for a threshold period
- Or the session reports completion status

Check the last message content for completion signals:
- "Implementation complete"
- "Summary:"
- "Completed:"
- "Done."

#### Handle Completion

1. Read the sub-agent's output (implementation summary)
2. Update the REQ file:
   ```yaml
   status: completed
   completed_at: 2025-01-26T10:45:00Z
   ```
3. Append implementation summary to REQ file
4. Archive the REQ (move to archive/, handle UR if all done)
5. Git commit if applicable
6. Remove from in-flight tracking
7. Report to user

#### Handle Failure

1. Read error from sub-agent output
2. Update REQ file:
   ```yaml
   status: failed
   error: "Description of failure"
   ```
3. Archive the failed REQ
4. Report to user
5. Check if any REQs depended on this one → they remain blocked

### Step 6: Continue Until Done

After each completion:
1. Re-scan queue (Step 2) — picks up new items and updated deps
2. Identify next batch (Step 3)
3. If batch is non-empty and under max_parallel, spawn more (Step 4)
4. Continue monitoring (Step 5)

Exit when:
- No pending REQs
- No in-flight agents
- All work complete

### Step 7: Final Cleanup

Run the cleanup action to:
- Consolidate archive
- Close completed UR folders
- Report final summary

## Agent Count Reporting

**REQUIRED:** When starting work, always report the current agent load:

```
[do-parallel-work] Starting...

Current agents: 2 in-flight (max: 3)
Available slots: 1
```

This helps the user understand system capacity. Report again whenever the count changes significantly:
- When spawning new agents
- When agents complete
- When hitting max_parallel limit

Example:
```
Spawning 2 agents... (will be 4 in-flight, at max capacity)
  ⚡ REQ-005 → agent:...:abc
  ⚡ REQ-006 → agent:...:def

[At max capacity - waiting for completions before spawning more]
```

## Progress Reporting

Keep the user informed:

```
[do-parallel-work] Starting...

Inbox: 0 new items
Queue: 5 pending, 0 in-flight

Dependency analysis:
  REQ-001: ready
  REQ-002: ready
  REQ-003: blocked by REQ-001
  REQ-004: ready
  REQ-005: blocked by REQ-003, REQ-004

Spawning batch 1 (3 agents):
  ⚡ REQ-001 (Route B) → agent:main:subagent:abc
  ⚡ REQ-002 (Route A) → agent:main:subagent:def
  ⚡ REQ-004 (Route C) → agent:main:subagent:ghi

Monitoring... [3 in-flight]

  ✓ REQ-002 completed (2m 15s) → archived
  
Monitoring... [2 in-flight, 1 now unblocked]

  ✓ REQ-004 completed (3m 02s) → archived
  
Spawning: REQ-005 now ready (REQ-004 done, still waiting on REQ-003)
  — REQ-005 still blocked by REQ-003

Monitoring... [1 in-flight]

  ✓ REQ-001 completed (4m 30s) → archived

REQ-003 now unblocked. Spawning...
  ⚡ REQ-003 (Route B) → agent:main:subagent:jkl

... [continues until done]
```

## Error Handling

### Sub-agent timeout
- Mark REQ as failed
- Archive with error
- Continue with other work

### Sub-agent crash
- Detect via session status
- Mark REQ as failed
- Report to user

### Circular dependency
- Detect during graph build
- Report error with cycle details
- Skip affected REQs

### Inbox file error
- Report which file failed
- Continue with other files
- Leave failed file in inbox

## Orchestrator Checklist

```
□ Check inbox for new items (Step 1)
□ Scan queue, build dependency graph (Step 2)
□ Identify ready REQs (Step 3)
□ For each ready REQ:
    □ Move to working/
    □ Update frontmatter (status, claimed_at)
    □ Triage (determine route)
    □ Write triage to REQ file
    □ Spawn sub-agent
    □ Update frontmatter (assigned_agent)
□ Monitor loop (Step 5):
    □ Poll sessions_list
    □ Check inbox periodically
    □ On completion: archive, update deps
    □ On failure: archive with error
□ Continue until queue empty (Step 6)
□ Run cleanup action (Step 7)
```

## What You Do vs What Sub-Agents Do

### You (Orchestrator) Do:
- File management (move, archive)
- Frontmatter updates
- Dependency analysis
- Spawning sub-agents
- Monitoring progress
- Git commits
- Reporting to user

### Sub-Agents Do:
- Planning (Route C)
- Exploration (Routes B, C)
- Implementation (all routes)
- Running tests
- Providing completion summary

**Sub-agents NEVER touch:**
- Request files (moving, status updates)
- Archive folder
- Other REQs
- Git operations
