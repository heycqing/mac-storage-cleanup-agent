---
name: mac-storage-cleanup-agent
description: Guide an AI coding agent through a cautious Mac storage cleanup workflow with read-only audit, risk classification, confirmed cleanup, and post-cleanup verification. Use when the user wants to free disk space on macOS, clean a Mac mini/MacBook, inspect large files, remove caches, or turn storage cleanup into a repeatable agent workflow.
---

# Mac Storage Cleanup Agent

## Quick Start

Use this skill when helping a user recover storage on macOS without damaging personal files or development projects.

Default posture:

1. Audit first.
2. Classify risk.
3. Ask for confirmation before deletion.
4. Clean only confirmed low-risk items.
5. Verify space recovered.

Never start by deleting files.

## Workflow

### 1. Read-Only Audit

Run only safe inspection commands first:

- Check total disk usage.
- Find large top-level directories.
- Inspect common cache and build-output locations.
- List large files without deleting them.

Useful targets:

```text
~/Downloads
~/Desktop
~/Documents
~/Library/Caches
~/Library/Logs
~/Library/Developer
~/.npm
~/.pnpm-store
~/.cache
~/.cargo
~/.rustup
Docker data, if Docker is installed
Project node_modules and build outputs
```

Output a table with:

```text
Path | Size | Type | Risk | Suggested action | Needs confirmation
```

### 2. Risk Classification

Classify every candidate before cleanup:

- Low risk: caches, temporary logs, package-manager caches, build artifacts that can be regenerated.
- Medium risk: downloads, old installers, archives, duplicate project outputs.
- High risk: source code, documents, photos, videos, databases, application support data.
- Do not delete automatically: keys, certificates, credentials, account data, unknown user data, configuration files.

If the purpose of a path is unclear, mark it as high risk and skip it.

### 3. Cleanup Plan

Before deleting anything, present:

- Exact absolute path.
- Command to run.
- Why the item is safe or risky.
- Expected recovery.
- How to verify after cleanup.

Require explicit user confirmation for deletion or destructive cleanup.

### 4. Execution Rules

When the user confirms:

- Re-check that each resolved absolute path is inside the intended directory.
- Prefer tool-specific cleanup commands over raw deletion when available.
- Avoid broad recursive deletion unless the path is precise and confirmed.
- Do not combine path discovery and deletion in one fragile command.

Examples of safer cleanup categories:

```text
npm/pnpm/yarn cache cleanup
Xcode derived data cleanup
old build outputs
application caches
temporary logs
```

### 5. Verification

After cleanup:

- Re-check free disk space.
- Summarize actual or estimated recovered space.
- List what was cleaned.
- List skipped items and why.
- Suggest manual follow-up for medium/high-risk items.

Use this final report shape:

```text
Recovered:
Cleaned:
Skipped:
Risks avoided:
Next checks:
```

## Prompt Template

```text
You are my macOS storage cleanup assistant. Start with read-only audit only. Do not delete anything. Measure disk usage and large directories, list cleanup candidates, classify them as low risk, medium risk, high risk, or do-not-delete, and provide path, size, type, risk, suggested action, and expected recovery. Only after I explicitly confirm may you generate and execute low-risk cleanup commands. After cleanup, verify recovered space and produce a cleanup report.
```

## Safety Checklist

- [ ] No destructive command before audit.
- [ ] No deletion before explicit confirmation.
- [ ] Absolute paths verified.
- [ ] User data skipped unless explicitly selected.
- [ ] Secrets and credentials never deleted automatically.
- [ ] Post-cleanup disk usage verified.

