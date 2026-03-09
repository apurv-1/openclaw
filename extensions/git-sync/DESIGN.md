# RFC: `@openclaw/git-sync` — Periodic Git Backup for OpenClaw State

**Status**: Draft / Open for critique  
**Author**: Copilot Agent  
**Date**: 2026-03-09  

---

## 1. Problem Statement

When OpenClaw is installed on a system, all mutable state (config, sessions, credentials, logs, caches) lives in `~/.openclaw/`. There is no built-in mechanism to continuously back this state up to a remote location. Users want:

- **Periodic backup** of the entire `~/.openclaw/` folder to a git repository.
- **Configurable interval** (default: every 10 seconds, but any value `≥ 5s`).
- **An independent package** that can work with OpenClaw (as a plugin) *or* be used standalone with any system that has a local directory to sync.

---

## 2. Proposed Architecture

### 2.1 Dual-mode package

The package ships as **both**:

1. **An OpenClaw extension** (`extensions/git-sync/`) — registered via `api.registerService()`, configured through `openclaw.json`, with CLI commands via `api.registerCli()`.
2. **A standalone Node.js CLI** — can be invoked directly (`npx @openclaw/git-sync --dir ~/.openclaw --remote git@github.com:user/backup.git --interval 10`) without any OpenClaw dependency.

```
extensions/git-sync/
├── package.json          # @openclaw/git-sync, "openclaw.extensions": ["./index.ts"]
├── index.ts              # OpenClaw plugin entry (register service + CLI)
├── DESIGN.md             # This document
├── src/
│   ├── sync-service.ts   # Core sync engine (framework-agnostic)
│   ├── git-ops.ts        # Git operations (init, add, commit, push)
│   ├── config.ts         # Config schema + validation
│   ├── cli.ts            # Standalone CLI entry point
│   ├── debounce.ts       # Change detection / dirty checking
│   ├── gitignore.ts      # Default .gitignore generation for sensitive files
│   └── types.ts          # Shared types
└── src/__tests__/
    ├── sync-service.test.ts
    ├── git-ops.test.ts
    ├── debounce.test.ts
    └── gitignore.test.ts
```

### 2.2 Separation of concerns

```
┌─────────────────────────────────────────────────────────┐
│                    OpenClaw Plugin Layer                 │
│  index.ts: register(api) → registerService + registerCli│
│  Reads config from api.pluginConfig                     │
└────────────────────┬────────────────────────────────────┘
                     │ calls
┌────────────────────▼────────────────────────────────────┐
│               Core Sync Engine (sync-service.ts)        │
│  Framework-agnostic. Accepts { dir, remote, interval }  │
│  Runs setInterval → detectChanges → gitOps.commitAndPush│
└────────────────────┬────────────────────────────────────┘
                     │ calls
┌────────────────────▼────────────────────────────────────┐
│               Git Operations (git-ops.ts)               │
│  init, add, commit, push — via simple-git               │
└─────────────────────────────────────────────────────────┘
```

---

## 3. Tech Choices

### 3.1 Git library: `simple-git`

| Option | Pros | Cons |
|--------|------|------|
| **`simple-git`** ✅ | Mature (14M+ weekly npm downloads), thin wrapper over native git CLI, supports all git operations, good TypeScript types, actively maintained | Requires `git` on PATH |
| `isomorphic-git` | Pure JS (no git binary needed), works in browsers | Incomplete push/auth support, slower for large repos, smaller community for server-side use |
| Raw `child_process` | Zero dependencies, full control | Manual parsing of git output, error-prone, reinventing the wheel |

**Decision**: `simple-git`. It's the Node.js ecosystem standard. The target environment (server/desktop running OpenClaw) will always have `git` installed. The codebase already shells out to `git` for commit-hash resolution, so git-on-PATH is an existing assumption.

### 3.2 Scheduling mechanism: `setInterval` + change detection

| Option | Pros | Cons |
|--------|------|------|
| **`setInterval`** ✅ | Zero dependencies, simple, precise for second-level intervals | Drift over time (negligible for backup use case) |
| `node-cron` | Cron syntax support, battle-tested | Overkill for simple interval; minimum resolution is 1 second; adds a dependency |
| `chokidar` (file watcher) | Event-driven (no polling), reacts to real changes instantly | Does not match user requirement of "every X seconds"; high CPU on large dirs; platform edge cases |
| OpenClaw's built-in `CronService` | Already exists in the codebase | Designed for agent invocations, not raw service loops; tight coupling to gateway internals |

**Decision**: `setInterval` in the core engine. The user explicitly wants time-based intervals ("every X seconds"), not event-driven. We add a lightweight change-detection step (compare file hashes or `git status`) to avoid empty commits.

### 3.3 Change detection: `git status --porcelain`

Before each commit cycle, run `git status --porcelain` on the synced directory. If output is empty, skip the commit+push cycle entirely. This is:
- Cheaper than computing file hashes manually.
- Already provided by `simple-git` via `git.status()`.
- Handles adds, modifications, deletions, and renames.

### 3.4 Authentication for git push

| Method | How |
|--------|-----|
| **SSH key** (default) | User configures `remote` as `git@github.com:user/repo.git`; relies on `~/.ssh/` keys or SSH agent |
| **HTTPS + credential helper** | User configures `remote` as `https://...`; relies on system git credential helper or `GIT_ASKPASS` |
| **HTTPS + token in URL** | `https://<token>@github.com/user/repo.git` — convenient but token visible in config |
| **GitHub CLI (`gh`)** | `gh auth` provides credentials — works if `gh` is installed |

**Decision**: Delegate authentication entirely to the system's git configuration. The extension does not store or manage git credentials. The user sets up SSH keys or a credential helper as they normally would. The `remote` config field is a plain git URL. This keeps the extension simple and avoids storing secrets.

### 3.5 Config validation: Manual `safeParse` (matching existing patterns)

The codebase uses a custom `OpenClawPluginConfigSchema` interface (not raw Zod). Extensions like `diffs` and `memory-lancedb` implement `safeParse()` manually with JSON Schema for documentation. We follow this pattern.

---

## 4. Configuration Schema

### 4.1 Plugin config (in `~/.openclaw/openclaw.json`)

```json5
{
  "plugins": {
    "entries": {
      "git-sync": {
        "config": {
          // Required: git remote URL
          "remote": "git@github.com:user/openclaw-backup.git",

          // Optional: directory to sync (defaults to ~/.openclaw)
          "dir": "~/.openclaw",

          // Optional: sync interval in seconds (default: 30, min: 5)
          "intervalSeconds": 30,

          // Optional: git branch name (default: "main")
          "branch": "main",

          // Optional: commit message template (default: "openclaw auto-sync {timestamp}")
          // Supports {timestamp}, {hostname} placeholders
          "commitMessage": "openclaw auto-sync {timestamp}",

          // Optional: paths/globs to exclude (merged with default sensitive exclusions)
          "exclude": [],

          // Optional: enable auto-generating .gitignore for sensitive files (default: true)
          "autoGitignore": true,

          // Optional: enable pushing to remote (default: true)
          // When false, only local commits are made (useful for local git history)
          "push": true,

          // Optional: author name/email for git commits
          "authorName": "openclaw-git-sync",
          "authorEmail": "git-sync@openclaw.local"
        }
      }
    }
  }
}
```

### 4.2 Default .gitignore for sensitive files

When `autoGitignore: true` (default), the extension writes/maintains a `.gitignore` in the synced directory root to exclude sensitive data:

```gitignore
# Auto-generated by @openclaw/git-sync
# These patterns protect sensitive data from being pushed to git.
# Edit with care. Removing entries may expose credentials.

# OAuth tokens and credentials
credentials/

# Cached data (large, regenerable)
cache/

# Temporary files
*.tmp
*.swp

# Logs (optional — user can remove this to sync logs)
logs/
```

**Open question for critique**: Should `credentials/` be *always* excluded (hardcoded, non-overridable) or should we trust the user's `.gitignore` choices? The risk is that a user removes the exclusion and accidentally pushes OAuth tokens to a public repo.

### 4.3 Standalone CLI config

When used standalone (without OpenClaw), config is passed via CLI flags:

```bash
npx @openclaw/git-sync \
  --dir ~/.openclaw \
  --remote git@github.com:user/backup.git \
  --interval 30 \
  --branch main \
  --commit-message "auto-sync {timestamp}"
```

Or via a config file:

```bash
npx @openclaw/git-sync --config ./git-sync.json
```

---

## 5. Core Sync Engine

### 5.1 Lifecycle

```
start(config)
  │
  ├── Validate config
  ├── Ensure dir exists
  ├── git init (if not already a git repo)
  ├── git remote add/set-url origin <remote>
  ├── Write .gitignore (if autoGitignore)
  ├── Initial sync (commit + push current state)
  │
  └── setInterval(syncCycle, intervalSeconds * 1000)
        │
        ├── git status --porcelain
        ├── if clean → skip
        ├── if dirty:
        │     ├── git add -A
        │     ├── git commit -m <message>
        │     └── git push origin <branch> (if push: true)
        └── log result (success/skip/error)

stop()
  │
  ├── clearInterval
  ├── Final sync (commit + push any remaining changes)
  └── Cleanup
```

### 5.2 Error handling

| Error | Handling |
|-------|----------|
| `git` not on PATH | Fail at `start()` with clear error message |
| Network failure (push fails) | Log warning, retry on next cycle. Do NOT retry immediately (could spam). Commit is preserved locally. |
| Merge conflict (remote has diverged) | Log error, skip push. On next cycle attempt `git pull --rebase` then push. If rebase fails, log error and pause sync (require manual resolution). |
| Permission denied (dir) | Fail at `start()` with error |
| Lock contention (`index.lock`) | Wait briefly, retry once. If still locked, skip cycle. |
| Rate limiting (GitHub API) | `simple-git` surfaces the error; back off exponentially up to 5 minutes |

### 5.3 Concurrency safety

- Only one sync cycle runs at a time (mutex via a boolean flag or `p-limit(1)`).
- If a cycle is still running when the next interval fires, the next cycle is skipped (not queued).
- `git.index.lock` contention is handled by detecting and waiting.

### 5.4 Commit message templating

```typescript
function formatCommitMessage(template: string): string {
  return template
    .replace("{timestamp}", new Date().toISOString())
    .replace("{hostname}", os.hostname());
}
```

---

## 6. OpenClaw Plugin Integration

### 6.1 Service registration

```typescript
// index.ts
import type { OpenClawPluginApi } from "openclaw/plugin-sdk/git-sync";
import { gitSyncConfigSchema } from "./src/config.js";
import { createGitSyncService } from "./src/sync-service.js";

const plugin = {
  id: "git-sync",
  name: "Git Sync",
  description: "Periodically sync OpenClaw state directory to a git repository",
  configSchema: gitSyncConfigSchema,
  register(api: OpenClawPluginApi) {
    const config = api.pluginConfig as GitSyncConfig | undefined;
    if (!config?.remote) {
      api.logger.info("git-sync: no remote configured, skipping");
      return;
    }

    api.registerService(createGitSyncService({
      dir: config.dir ?? api.resolvePath("~/.openclaw"),
      remote: config.remote,
      intervalSeconds: config.intervalSeconds ?? 30,
      branch: config.branch ?? "main",
      commitMessage: config.commitMessage ?? "openclaw auto-sync {timestamp}",
      exclude: config.exclude ?? [],
      autoGitignore: config.autoGitignore ?? true,
      push: config.push ?? true,
      authorName: config.authorName ?? "openclaw-git-sync",
      authorEmail: config.authorEmail ?? "git-sync@openclaw.local",
      logger: api.logger,
    }));

    api.registerCli(
      ({ program }) => registerGitSyncCli(program, api),
      { commands: ["git-sync"] },
    );
  },
};

export default plugin;
```

### 6.2 CLI commands (via registerCli)

```
openclaw git-sync status        # Show sync status (last sync time, pending changes, remote)
openclaw git-sync trigger       # Force an immediate sync cycle
openclaw git-sync pause         # Pause periodic sync
openclaw git-sync resume        # Resume periodic sync
openclaw git-sync history       # Show recent sync history (last N commits)
```

### 6.3 Service lifecycle mapping

| OpenClaw Event | Git Sync Action |
|----------------|-----------------|
| `service.start(ctx)` | Initialize git repo, start interval |
| `service.stop(ctx)` | Final sync, clear interval |
| Gateway restart | Service restarts automatically |
| Config change | Requires gateway restart (standard for plugin config) |

---

## 7. Security Considerations

### 7.1 Sensitive data protection

The `~/.openclaw/` directory contains sensitive data:

| Path | Sensitivity | Default handling |
|------|-------------|-----------------|
| `credentials/` | 🔴 HIGH — OAuth tokens, API keys | **Always excluded** (hardcoded, non-overridable) |
| `openclaw.json` | 🟡 MEDIUM — may contain API keys in provider config | Included (user should use env var references `${API_KEY}`) |
| `sessions/` | 🟡 MEDIUM — conversation history | Included by default |
| `cache/` | 🟢 LOW — regenerable | Excluded by default |
| `logs/` | 🟢 LOW — operational logs | Excluded by default |

**Hardcoded exclusions** (cannot be overridden):
- `credentials/` — always added to `.gitignore`
- `*.key`, `*.pem`, `*.p12` — private key patterns

### 7.2 Remote repository security

- The extension **never** stores git credentials. Authentication is delegated to the user's git config (SSH keys, credential helpers).
- The `remote` URL is stored in `openclaw.json` in plaintext. If this contains a token (HTTPS token-in-URL), it is the user's responsibility to secure their config file.
- **Recommendation in docs**: Use SSH remotes or credential helpers; avoid token-in-URL.

### 7.3 Audit trail

- Every sync cycle logs: timestamp, number of files changed, commit SHA, push result.
- Logs go to the OpenClaw plugin logger (when running as extension) or stdout (when standalone).

---

## 8. Performance Considerations

### 8.1 Typical `.openclaw/` size

- Fresh install: ~50 KB
- After moderate use (weeks): 5–50 MB (mostly sessions and memory DB)
- Heavy use with workspaces: 100 MB+

### 8.2 Git performance at scale

- `git status` on a 50 MB directory: < 100ms (fast)
- `git add -A` + `git commit`: < 500ms for typical changes
- `git push`: network-bound (1–10 seconds depending on change size and connection)

### 8.3 Interval tuning

- Default 30 seconds is conservative. Even at 10 seconds:
  - With change detection, most cycles are no-ops (`git status` only)
  - Push only happens when there are actual changes
  - The `git push` is the expensive operation; commit is fast

### 8.4 Binary files

SQLite databases (memory, sessions) and binary caches will be in the repo. Git handles these poorly for diffs but fine for storage. Over time, the `.git` directory in the synced folder will grow. **Mitigation**: document that users should periodically run `git gc` or consider `git-lfs` for large binary files.

---

## 9. Alternatives Considered (and rejected)

### 9.1 rsync instead of git

- **Pro**: Faster for pure file sync, handles binary files better.
- **Con**: No history, no conflict detection, no rollback, not cross-platform (Windows needs WSL or cygwin). Git gives us versioned history for free.

### 9.2 Cloud storage SDK (S3, GCS, Azure Blob)

- **Pro**: Scalable, no git overhead.
- **Con**: Requires cloud credentials, vendor lock-in, no diff/history, adds heavy SDK dependencies. Git remotes can point to any git host (GitHub, GitLab, self-hosted, local bare repo).

### 9.3 Full-directory tar + upload

- **Pro**: Simple, atomic snapshots.
- **Con**: OpenClaw already has `backup create` for this. The user specifically wants git-based sync with history.

### 9.4 Using OpenClaw's built-in CronService

- **Pro**: Reuses existing infrastructure.
- **Con**: CronService is designed for agent invocations (spawns isolated agent processes). Using it for a simple file sync would be over-engineered. A `setInterval` in a plugin service is simpler and more appropriate.

---

## 10. Open Questions for Review

1. **Hardcoded credential exclusion**: Should `credentials/` be *always* excluded (hardcoded) or should we allow users to override this? Risk: accidental credential exposure.

2. **Conflict resolution strategy**: When the remote has diverged (e.g., user syncs from two machines), should we:
   - (a) Force-push (lose remote changes) — simple but dangerous
   - (b) Rebase then push (preserve both) — complex, may fail on binary files
   - (c) Create a new branch per machine — safe but clutters branches
   - (d) Fail and alert the user — safest

3. **Default interval**: 10 seconds (as mentioned in problem statement) vs 30 seconds (less aggressive). For most users, state changes happen infrequently, so 30s seems reasonable. But the user specifically mentioned 10s.

4. **Standalone CLI packaging**: Should the standalone CLI be a separate npm package (`@openclaw/git-sync-cli`) or the same package with a `bin` entry? Same package is simpler but pulls in OpenClaw plugin SDK types (even if unused at runtime).

5. **Initial clone**: If the remote repo already has content (e.g., synced from another machine), should `start()` clone/pull first before starting the sync loop? This enables restore-from-backup as a natural side effect.

6. **Large file handling**: Should we recommend/integrate `git-lfs` for SQLite databases and binary caches, or keep it simple with vanilla git?

7. **Plugin SDK export**: The plugin uses `openclaw/plugin-sdk/git-sync`. This requires adding a new export entry in the root `package.json` and a corresponding plugin SDK file. Should we do this, or use the generic `openclaw/plugin-sdk` import?

---

## 11. Implementation Plan

### Phase 1: Core engine + extension (MVP)
- [ ] `src/git-ops.ts` — git init, add, commit, push via `simple-git`
- [ ] `src/sync-service.ts` — interval loop with change detection
- [ ] `src/config.ts` — config schema with validation
- [ ] `src/gitignore.ts` — default .gitignore generation
- [ ] `index.ts` — OpenClaw plugin registration
- [ ] `package.json` — extension package with `simple-git` dependency
- [ ] Tests for git-ops, sync-service, config validation

### Phase 2: CLI + polish
- [ ] CLI commands (`status`, `trigger`, `pause`, `resume`, `history`)
- [ ] Standalone CLI entry point
- [ ] Conflict detection and recovery
- [ ] Documentation

### Phase 3: Future enhancements
- [ ] Encryption at rest (encrypt before commit)
- [ ] Selective sync (only specific subdirs)
- [ ] Webhook/notification on sync failure
- [ ] Multi-remote support (push to multiple git remotes)
- [ ] git-lfs integration for large binary files

---

## 12. Dependencies

| Package | Version | Purpose | Weekly Downloads |
|---------|---------|---------|-----------------|
| `simple-git` | `^3.27.0` | Git operations | ~14M |

**That's it.** One runtime dependency. The extension relies on `git` being available on the system PATH (same assumption the rest of OpenClaw makes).

Dev dependencies inherit from the OpenClaw monorepo (TypeScript, Vitest, etc.).

---

## 13. Example: Full Sync Cycle (Happy Path)

```
[00:00] Service starts
[00:00] git init ~/.openclaw (already initialized, no-op)
[00:00] git remote set-url origin git@github.com:user/backup.git
[00:00] Write .gitignore (credentials/, cache/, logs/)
[00:00] git add -A && git commit -m "openclaw auto-sync 2026-03-09T02:00:00Z"
[00:00] git push origin main ✓

[00:30] Interval fires
[00:30] git status --porcelain → "" (no changes)
[00:30] Skip cycle ✓

[01:00] Interval fires
[01:00] git status --porcelain → "M openclaw.json\nM sessions/abc.json"
[01:00] git add -A
[01:00] git commit -m "openclaw auto-sync 2026-03-09T02:01:00Z"
[01:00] git push origin main ✓

[01:30] Interval fires
[01:30] git status → no changes → skip ✓

... (continues until stop())

[STOP] Gateway shutting down
[STOP] Final sync: git add -A && commit && push
[STOP] clearInterval
[STOP] Done ✓
```

---

## 14. Example: Error Recovery

```
[05:00] Interval fires
[05:00] git status → dirty
[05:00] git add -A && git commit ✓
[05:00] git push → FAILED (network timeout)
[05:00] Log warning: "git-sync: push failed, will retry next cycle"
[05:00] Commit is preserved locally ✓

[05:30] Interval fires
[05:30] git status → clean (already committed)
[05:30] git push origin main ✓ (pushes the commit from 05:00)

---

[10:00] Interval fires
[10:00] git push → FAILED (remote has diverged — another machine pushed)
[10:00] Attempt: git pull --rebase origin main
[10:00] Rebase succeeded ✓
[10:00] git push origin main ✓

---

[15:00] Interval fires
[15:00] git push → FAILED (remote diverged)
[15:00] git pull --rebase → CONFLICT on sessions/xyz.json
[15:00] git rebase --abort
[15:00] Log error: "git-sync: merge conflict detected, sync paused. Run 'openclaw git-sync resolve' to fix."
[15:00] Pause sync loop
```
