---
name: greptile-worktree
description: Git worktree workflow for Greptile code review on unindexed branches. Use when implementing features that need Greptile review but the target branch isn't indexed. Solves the problem of Greptile only indexing main/master branches by keeping work on main in a worktree, then branching at PR time.
---

# Greptile Worktree Workflow

Greptile only indexes main branches. This workflow lets you run Greptile reviews locally during development by working on main in a worktree, then creating a feature branch when ready to PR.

## Prerequisites

- Repository must be indexed by Greptile
- Greptile API key available at `~/.config/greptile/token`

## Workflow

### 1. Create Worktree

```bash
cd /path/to/repo
git worktree add ../repo-worktree main
cd ../repo-worktree
```

Names the worktree folder `repo-worktree` (sibling to main repo).

### 2. Implement Task

Work normally in the worktree. Commit to main as you go:

```bash
git add .
git commit -m "feat: implement feature"
```

### 3. Run Greptile Review

**Always echo status before running:**

```bash
echo "🔍 Greptile Review - Iteration $ITERATION_COUNT"
```

Then run the review:

```bash
export GREPTILE_TOKEN=$(cat ~/.config/greptile/token)

curl -s -X POST "https://api.greptile.com/v2/query" \
  -H "Authorization: Bearer $GREPTILE_TOKEN" \
  -H "X-Github-Token: $(gh auth token)" \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [{"role": "user", "content": "Review the recent changes for bugs, security issues, and code quality. Be thorough."}],
    "repositories": [{"remote": "github", "repository": "OWNER/REPO", "branch": "main"}],
    "genius": true
  }'
```

Replace `OWNER/REPO` with the actual repository path.

### 4. Address Issues

**Echo status with issue count:**

```bash
echo "🔧 Iteration $ITERATION_COUNT: Addressing $ISSUE_COUNT issues"
```

Fix any problems Greptile identifies. Commit fixes:

```bash
git add .
git commit -m "fix: address review feedback"
```

### 5. Iterate Until 5/5

**Target: 5/5 confidence score.** Keep iterating until you reach it.

Repeat steps 3-4 until Greptile gives confidence score 5/5.

**Echo status after each review:**

```bash
echo "✅ Greptile passed with score $SCORE/5 after $ITERATION_COUNT iterations"
# OR if continuing:
echo "⚠️ Iteration $ITERATION_COUNT: Score $SCORE/5 - continuing..."
```

**Score guidance:**
- **5/5**: Ship it ✓
- **4/5**: Keep iterating — fix the remaining issues
- **<4/5**: Definitely keep iterating

**Only accept 4/5 if:**
1. The flagged issue is a false positive (explain why in PR)
2. The fix would require architectural changes outside scope (document in PR)
3. You've attempted 3+ iterations and the same issue persists despite fixes

If stopping at 4/5, you MUST document the specific reason in the PR description.

### 6. Create Branch and Push

Stay on main. Create a branch pointing to your commits, push it, then reset main:

```bash
# Create feature branch from current HEAD
git branch feat/your-feature-name

# Push the feature branch
git push origin feat/your-feature-name

# Reset main back to remote
git reset --hard origin/main
```

You never checkout the feature branch—main stays clean.

### 7. Create PR

```bash
gh pr create --base main --head feat/your-feature-name \
  --title "feat: your feature" \
  --body "Description of changes"
```

### 8. Cleanup (Optional)

Remove worktree when done iterating:

```bash
cd /path/to/repo
git worktree remove ../repo-worktree
```

**Note:** You can keep the worktree if you expect to iterate further (e.g., PR at 4/5 needs more work, or waiting for human review). See "Continuing After PR" section below.

## Continuing After PR (Iteration or Feedback)

Use this when:
- You submitted a PR at 4/5 and need to reach 5/5
- Human reviewer requested changes
- You discovered issues after PR creation

### Option A: Worktree Flow (Recommended for Greptile)

**If worktree was cleaned up, recreate it:**

```bash
cd /path/to/repo
git fetch origin
git worktree add ../repo-worktree main
cd ../repo-worktree

# Pull in the feature branch changes
git merge origin/feat/your-feature-name
```

**If worktree still exists:**

```bash
cd ../repo-worktree
git fetch origin
git merge origin/feat/your-feature-name
```

**Then iterate:**

1. Make fixes
2. Run Greptile review (Step 3)
3. Repeat until 5/5
4. Force push to feature branch:
   ```bash
   git push origin main:feat/your-feature-name --force
   ```
5. Reset main and cleanup:
   ```bash
   git reset --hard origin/main
   cd /path/to/repo
   git worktree remove ../repo-worktree
   ```

### Option B: Direct Branch Flow (Skip Greptile)

Only for trivial fixes where Greptile review isn't needed:

1. Checkout feature branch in main repo
2. Make fixes directly
3. Push

**Note:** This skips local Greptile review. Only use for obvious fixes.

## Quick Reference

| Step | Command |
|------|---------|
| Create worktree | `git worktree add ../repo-worktree main` |
| Run Greptile | `curl ...` (see step 3) |
| Create branch | `git branch feat/name` |
| Push branch | `git push origin feat/name` |
| Reset main | `git reset --hard origin/main` |
| Remove worktree | `git worktree remove ../repo-worktree` |
