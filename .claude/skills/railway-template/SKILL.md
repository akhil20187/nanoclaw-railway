---
name: railway-template
description: Pull upstream NanoClaw commits into the Railway template repo, replaying each commit individually with Railway adaptation commits where needed.
---

# About

This skill syncs the Railway template repo with the parent NanoClaw repo (`https://github.com/qwibitai/nanoclaw`). It replays each upstream commit individually, preserving the original commit message, and adds Railway adaptation commits after any commit that touches files with Railway-specific modifications.

Run `/railway-template` in Claude Code, or tell Claude: "Sync upstream NanoClaw changes into this Railway repo"

## What This Does

The Railway template is a fork of NanoClaw with Railway-specific adaptations (no Docker, secrets via stdin, persistent volume, etc.). The parent repo evolves independently. This skill pulls all new parent commits into the Railway repo **one at a time**, preserving original commit messages, and adds adaptation commits where Railway-specific code was affected.

---

# Railway-Adapted Files

These files have intentional divergences from upstream. Conflicts in these files need resolution that preserves both upstream changes AND Railway adaptations:

| File | What Railway Changes |
|------|---------------------|
| `src/config.ts` | `IS_RAILWAY` flag, `RAILWAY_VOLUME` path, `SLACK_MAIN_CHANNEL_ID`, redirects `STORE_DIR`/`GROUPS_DIR`/`DATA_DIR` to `/data/...` on Railway |
| `src/env.ts` | Skips `.env` file on Railway — reads everything from `process.env` (Railway service config) |
| `src/index.ts` | Conditionally starts credential proxy (skip on Railway), calls `syncSkillsOnStartup()`/`syncMcpOnStartup()`, auto-registers main group + channel groups, thread-aware context formatting |
| `src/container-runner.ts` | Early-returns to `runRailwayAgent()` on Railway, keeps `readSecrets()` for stdin-based secret passing, `secrets` field on `ContainerInput` interface |
| `src/ipc.ts` | Adds IPC handlers: `install_skills`, `remove_skill`, `list_skills`, `add_mcp_server`, `remove_mcp_server`, `list_mcp_servers` |
| `src/router.ts` | `formatThreadWithContext()` with timezone support |
| `container/agent-runner/src/index.ts` | Keeps `createSanitizeBashHook` for stripping secrets from Bash subprocesses (upstream removed it when adding credential proxy) |

## Railway-Only Files (not in upstream)

| File | Purpose |
|------|---------|
| `src/railway-runner.ts` | Spawns agent as child process instead of Docker container |
| `src/mcp-installer.ts` | Persistent MCP server registry (survives Railway deploys via `/data` volume) |
| `src/skill-installer.ts` | Persistent skill installer (clones skill repos, manages lock file) |
| `Dockerfile.railway` | Multi-stage Docker build for Railway deployment |
| `docker-entrypoint-railway.sh` | Fixes `/data` volume ownership, drops to `node` user |
| `railway.json` | Railway platform config |
| `RAILWAY.md` | Deployment documentation |

---

# Conflict Resolution Cheat Sheet

| Scenario | Resolution |
|----------|-----------|
| Upstream adds new import, Railway has different imports | Keep both — add upstream's new import alongside Railway's |
| Upstream changes function signature Railway also modified | Merge both changes into the signature |
| Upstream adds feature Railway deliberately removed (e.g., credential proxy on Railway) | Keep the feature code but gate it with `if (!IS_RAILWAY)` |
| Upstream removes code Railway still needs (e.g., `readSecrets`) | Keep Railway's version |
| Upstream changes `.env` handling | Keep Railway's `process.env`-first approach |
| CHANGELOG conflict | Accept upstream's additions, remove conflict markers |
| Skill file conflict | Usually accept upstream (`git checkout --theirs`) |

---

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
2. `npm test` should pass (fix test assertions if Railway error messages diverge)
3. Verify Railway files: `grep -l IS_RAILWAY src/config.ts src/container-runner.ts src/index.ts`
4. Verify Railway-only files exist: `ls src/railway-runner.ts src/mcp-installer.ts src/skill-installer.ts`
5. Show commit log: `git log --oneline <backup-tag>..HEAD`

# Step 5: Summary

Show:
- Total upstream commits processed
- Commits applied cleanly (no Railway impact)
- Commits applied with Railway adaptation
- Commits skipped (already in repo or conflicts)
- Features pulled from parent (grouped by: New Skills, Core Improvements, Bug Fixes)
- Backup tag for rollback
- Rollback command: `git reset --hard <backup-tag>`

# Rollback

```bash
git reset --hard <backup-tag>
git push origin main --force
```

---

# Sync History

## 2026-03-15

### Features Pulled from Parent

#### New Skills
- **`/remote-control`** — Host-level Claude Code access from chat. Send `/remote-control` in main group to get a URL for browser-based Claude Code on the host machine.
- **`/add-ollama`** — Local model inference via Ollama MCP server. Cheaper/faster for summarization, translation, general queries.
- **`/compact`** — Manual context compaction. Fixes context rot in long agent sessions by forwarding the SDK's built-in `/compact` command.
- **WhatsApp reactions** — Emoji reaction support: receive, send, store, and search reactions.
- **Image vision** — Processes WhatsApp image attachments and sends them to Claude as multimodal content blocks.
- **PDF reader** — Extracts text from PDFs via `pdftotext` CLI. Handles WhatsApp attachments, URLs, and local files.
- **Local Whisper** — Switch voice transcription from OpenAI API to local `whisper.cpp` on Apple Silicon.

#### Core Improvements
- **Credential proxy** — Enhanced container environment isolation. API calls from containers route through a local proxy instead of receiving secrets directly. On Railway, secrets still go via stdin (proxy is skipped).
- **Timezone-aware context** — Agent prompts now include timezone context and display timestamps in local time instead of UTC.
- **Sender allowlist** — Per-chat access control. Restrict which senders can trigger the agent or have messages stored.
- **`update_task` tool** — Agents can now update scheduled tasks (not just create/delete). Task IDs are returned from `schedule_task`.
- **DB query limits** — Added `LIMIT` to unbounded message history queries to prevent memory issues in active groups.
- **Atomic task claims** — Prevents scheduled tasks from executing twice when container runtime exceeds the poll interval.
- **Skills as branches** — Major architecture change: skills are now git branches, channels are forks. Replaces the old skills-engine directory.
- **Docker Sandboxes** — New deployment option announced in README (alternative to Railway).
- **claude-agent-sdk bumped to ^0.2.76**

#### Bug Fixes
- Close task container promptly when agent uses IPC-only messaging
- WhatsApp: use sender's JID for DM-with-bot registration, skip trigger
- WhatsApp: write pairing code to file for immediate access
- WhatsApp: add error handling to messages.upsert handler
- Voice transcription skill drops WhatsApp registerChannel call
- Correct misleading send_message tool description for scheduled tasks
- Fix broken step references in setup/SKILL.md
- Rename `_chatJid` to `chatJid` in onMessage callback
- Format src/index.ts to pass CI prettier check

#### Railway Adaptations Made
- `railway: adapt credential proxy` — Kept stdin secret passing, conditionally start proxy only when not on Railway
- `railway: fix Slack test assertions` — Updated test to match Railway's error message wording
