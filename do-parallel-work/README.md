# Do-Parallel-Work Skill

A parallel task queue for AI agents. Fork of [do-work](https://github.com/bladnman/do-work) with added support for:

- **Parallel execution** — spawn multiple sub-agents for independent tasks
- **Dependency tracking** — `depends_on` field blocks tasks until prerequisites complete
- **Inbox folder** — drop files asynchronously while work is in progress
- **Source document linking** — trace REQs back to master plan documents
- **Priority levels** — high/normal/low for batch ordering

## Installation

Copy the `do-parallel-work` folder to your skills directory.

## Usage

### Capture tasks
```
do parallel work add user authentication
do parallel work fix the search bug and also add dark mode
```

### Process queue (parallel)
```
do parallel work run
```

### Check status
```
do parallel work status
```

### Verify captures
```
do parallel work verify
```

## Architecture

```
You (Orchestrator)
    │
    ├── Check inbox for new items
    ├── Analyze dependencies
    ├── Spawn sub-agents for independent tasks (parallel)
    ├── Monitor completions
    ├── Archive and unblock dependents
    └── Repeat until done
```

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `max_parallel` | 3 | Maximum concurrent sub-agents |
| `inbox_path` | null | Folder to watch for new task files |
| `plans_path` | null | Folder for master plan documents |
| `poll_interval` | 5000 | Ms between status checks |

## Key Differences from do-work

| Feature | do-work | do-parallel-work |
|---------|---------|------------------|
| Execution | Serial | Parallel |
| Sub-agents | Optional | Always (you orchestrate) |
| Dependencies | Not tracked | `depends_on` field |
| Inbox | Not supported | Async file drops |
| Source docs | Not tracked | `source_doc` field |

## Credits

Based on [do-work](https://github.com/bladnman/do-work) by bladnman.
