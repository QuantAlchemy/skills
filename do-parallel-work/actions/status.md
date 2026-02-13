# Status Action

> **Part of the do-parallel-work skill.** Shows current queue state, in-flight agents, and progress.

## Usage

```
do parallel work status
```

## Output

Display a summary of the current state:

```
╔══════════════════════════════════════════════════════════════╗
║                    DO-PARALLEL-WORK STATUS                   ║
╠══════════════════════════════════════════════════════════════╣
║ Queue:     5 pending                                         ║
║ In-flight: 2 agents                                          ║
║ Completed: 12 (today)                                        ║
║ Failed:    1 (today)                                         ║
╚══════════════════════════════════════════════════════════════╝

PENDING (5)
───────────────────────────────────────────────────────────────
  REQ-018  Add user avatars           ready
  REQ-019  Fix search performance     ready
  REQ-020  Implement dark mode        blocked by REQ-018
  REQ-021  Update API endpoints       ready
  REQ-022  Refactor auth system       blocked by REQ-019, REQ-021

IN-FLIGHT (2)
───────────────────────────────────────────────────────────────
  REQ-016  OAuth integration    Route C   3m 42s   agent:...:abc
  REQ-017  Add export button    Route A   1m 15s   agent:...:def

RECENTLY COMPLETED (last 5)
───────────────────────────────────────────────────────────────
  REQ-015  Fix typo             Route A   0m 32s   ✓
  REQ-014  Update styles        Route B   2m 10s   ✓
  REQ-013  Add validation       Route B   4m 22s   ✓
  REQ-012  New component        Route C   8m 15s   ✓
  REQ-011  Config change        Route A   0m 18s   ✓

INBOX
───────────────────────────────────────────────────────────────
  Path: /home/user/gdrive-tasks/inbox
  Items: 2 files waiting
    - new-feature-idea.md
    - bug-report.md
```

## Implementation

### Step 1: Gather Queue Data

```bash
# Pending
ls do-work/REQ-*.md 2>/dev/null | wc -l

# In-flight  
ls do-work/working/REQ-*.md 2>/dev/null

# Completed today
ls do-work/archive/REQ-*.md 2>/dev/null  # Check dates
```

### Step 2: Check In-Flight Agents

For each REQ in `working/`:
1. Read `assigned_agent` from frontmatter
2. Use `sessions_list` to get agent status
3. Calculate elapsed time from `claimed_at`

### Step 3: Analyze Dependencies

For each pending REQ:
1. Read `depends_on` field
2. Check if dependencies are completed
3. Mark as "ready" or "blocked by X, Y"

### Step 4: Check Inbox

If inbox_path configured:
1. List files
2. Show count and filenames

### Step 5: Show Recent Completions

Read from archive, sort by `completed_at`, show last 5.

## Compact Mode

For quick checks:

```
do parallel work status --compact
```

Output:
```
Queue: 5 pending (3 ready) | In-flight: 2 | Done today: 12 | Inbox: 2
```
