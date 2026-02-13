# Changelog

All notable changes to this skill will be documented in this file.

## [1.0.0] - 2026-02-04

### Added
- Forked from do-work skill
- **Parallel execution** — work action spawns multiple sub-agents for independent tasks
- **Dependency tracking** — `depends_on` field in REQ frontmatter blocks execution until prerequisites complete
- **Inbox folder** — async task input via file drops while orchestrator is working
- **Source document linking** — `source_doc` field traces REQs back to master plan documents
- **Priority levels** — `priority` field (high/normal/low) affects batch ordering
- **Status action** — new action to show queue state, in-flight agents, and progress
- **Assigned agent tracking** — `assigned_agent` field tracks which sub-agent owns a REQ

### Changed
- Work action is now purely an orchestrator — never does implementation itself
- Always spawns sub-agents via `sessions_spawn`
- Monitors multiple in-flight agents simultaneously
- Re-checks inbox during monitoring loop

### Based On
- do-work v0.8.0 by bladnman
- https://github.com/bladnman/do-work
