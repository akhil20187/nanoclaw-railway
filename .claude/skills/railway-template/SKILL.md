---
name: railway-template
description: Pull upstream NanoClaw commits into the Railway template repo, replaying each commit individually with Railway adaptation commits where needed.
---

# About

This skill syncs the Railway template repo with the upstream NanoClaw repo (`https://github.com/qwibitai/nanoclaw`). It replays each upstream commit individually, preserving the original commit message, and adds Railway adaptation commits after any commit that touches files with Railway-specific modifications.

Run `/railway-template` in Claude Code.

# Railway-Specific Files

These files have intentional Railway divergences that must be preserved after each upstream commit:

| File | Railway Adaptation |
|------|-------------------|
| `src/config.ts` | `IS_RAILWAY` detection, `RAILWAY_VOLUME` path, `SLACK_MAIN_CHANNEL_ID`, redirect `STORE_DIR`/`GROUPS_DIR`/`DATA_DIR` to Railway volume |
| `src/env.ts` | Skip `.env` on Railway, read all config from `process.env` |
| `src/index.ts` | Conditionally start credential proxy (skip on Railway), add `syncSkillsOnStartup()`/`syncMcpOnStartup()`, auto-register groups, thread context |
| `src/container-runner.ts` | Early-return to `runRailwayAgent()` when `IS_RAILWAY`, `readSecrets()` via stdin, `secrets` field on `ContainerInput` |
| `src/ipc.ts` | Add `install_skills`/`remove_skill`/`list_skills`/`add_mcp_server`/`remove_mcp_server`/`list_mcp_servers` IPC handlers |
| `src/router.ts` | Add `formatThreadWithContext()` with timezone support |
| `src/railway-runner.ts` | Railway-only file: child process spawning instead of Docker |
| `src/mcp-installer.ts` | Railway-only file: persistent MCP server registry |
| `src/skill-installer.ts` | Railway-only file: persistent skill installer |
| `Dockerfile.railway` | Railway-only file: multi-stage Docker build |
| `docker-entrypoint-railway.sh` | Railway-only file: entrypoint script |
| `railway.json` | Railway-only file: platform config |
| `RAILWAY.md` | Railway-only file: deployment docs |
| `container/agent-runner/src/index.ts` | Keeps `createSanitizeBashHook` for stdin-based secret sanitization |

# Goal

Replay every upstream commit from the merge base to `upstream/main` HEAD into this Railway template repo, one at a time. After each commit that modifies a Railway-adapted file, create an additional "railway: adapt <description>" commit to restore Railway-specific behavior.

# Operating Principles

- **Never lose Railway changes.** Every adaptation must be verified.
- **Preserve upstream commit messages exactly.** Use `git cherry-pick` with original messages.
- **Skip already-incorporated commits.** Match by commit message (not hash, since hashes differ).
- **One adaptation commit per upstream commit that needs it.** Don't batch adaptations.
- **Build must pass after every adaptation commit.** Run `npm run build` to verify.
- **Create a backup before starting.** Branch + tag for rollback.

# Step 0: Preflight

1. Verify clean working tree: `git status --porcelain` must be empty
2. Ensure `upstream` remote exists: `git remote add upstream https://github.com/qwibitai/nanoclaw.git`
3. Fetch upstream: `git fetch upstream --prune`
4. Create backup: `git branch backup/pre-railway-sync-$(date +%Y%m%d-%H%M%S)` and `git tag pre-railway-sync-$(date +%Y%m%d-%H%M%S)`

# Step 1: Compute Commit List

1. Find merge base: `BASE=$(git merge-base HEAD upstream/main)`
2. List upstream commits in chronological order (oldest first): `git log --reverse --oneline --no-merges $BASE..upstream/main`
3. Filter out commits already in the local repo (match by commit message using `git log --format="%s" $BASE..HEAD`)
4. Present the remaining commit list to the user for confirmation

# Step 2: Snapshot Railway State

Save current Railway-adapted files as reference:
```bash
mkdir -p /tmp/railway-snapshots
for f in src/config.ts src/env.ts src/index.ts src/container-runner.ts src/ipc.ts src/router.ts; do
  cp "$f" "/tmp/railway-snapshots/$(basename $f)"
done
```

# Step 3: Replay Each Commit

For each upstream commit (oldest to newest):

## 3a: Cherry-pick
```bash
git cherry-pick <hash>
```

## 3b: If conflicts
1. Check `git status` for conflicted files
2. For Railway-adapted files: merge both upstream changes AND Railway adaptations
3. For other files: accept upstream changes (`git checkout --theirs <file>`)
4. `git add` resolved files, `git cherry-pick --continue --no-edit`

## 3c: If Railway files were modified
After the upstream commit is applied, check if Railway adaptations were lost:
- Run `npm run build` — if it fails, fix Railway-specific issues
- Grep for `IS_RAILWAY`, `RAILWAY_VOLUME`, `runRailwayAgent`, `readSecrets` in affected files
- If adaptations are missing, re-apply them and commit:
  ```bash
  git commit -am "railway: adapt <description>"
  ```

# Step 4: Validation

After all commits are replayed:
1. `npm run build` must pass
2. Verify Railway files: `grep -l IS_RAILWAY src/config.ts src/container-runner.ts src/index.ts`
3. Verify Railway-only files exist: `ls src/railway-runner.ts src/mcp-installer.ts src/skill-installer.ts`
4. Show commit log: `git log --oneline <backup-tag>..HEAD`

# Step 5: Summary

Show total commits processed, applied, adapted, skipped. Provide rollback command.

# Rollback
```bash
git reset --hard <backup-tag>
```
