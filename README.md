# Mac Storage Cleanup Agent Playbook

A cautious, tool-agnostic AI agent playbook for cleaning macOS storage. It works with Codex, Claude Code, DeepSeek, or any terminal-capable coding agent.

## What It Does

- Audits macOS disk usage with read-only commands first.
- Finds large directories, caches, build outputs, logs, package-manager caches, and app leftovers.
- Classifies candidates as low, medium, high risk, or do-not-delete.
- Requires confirmation before deletion.
- Produces a cleanup report after execution.

## Repository Layout

Recommended open-source layout:

```text
mac-storage-cleanup-agent/
├── README.md          # English usage and safety model
├── README.zh-CN.md    # Chinese usage and safety model
├── PROMPT.md          # Universal prompt for any AI assistant
├── SKILL.md           # Codex skill adapter
├── CLAUDE.md          # Claude Code adapter
└── LICENSE
```

## Use With Codex

Copy this folder into your Codex skills directory:

```bash
mkdir -p ~/.codex/skills
cp -R mac-storage-cleanup-agent ~/.codex/skills/
```

The final path should look like this:

```text
~/.codex/skills/mac-storage-cleanup-agent/SKILL.md
```

Restart Codex or open a new Codex thread so the skill list refreshes.

Ask Codex something like:

```text
Use the mac-storage-cleanup-agent skill to help me free disk space on my Mac.
Only audit first. Do not delete anything until I confirm.
```

## Use With Claude Code

Copy `CLAUDE.md` into the project or workspace where Claude Code should follow this workflow, or paste the instruction into your Claude Code session:

```text
Use the Mac Storage Cleanup Agent workflow from CLAUDE.md. Start with read-only audit only. Do not delete anything until I explicitly confirm the cleanup plan.
```

## Use With DeepSeek Or Other Assistants

Paste the contents of `PROMPT.md` into the assistant:

```text
Follow this macOS storage cleanup workflow. First audit only, classify risk, propose cleanup commands, wait for confirmation, then verify recovered space.
```

## Safety Model

This skill is intentionally conservative:

- No destructive commands before read-only audit.
- No deletion before explicit confirmation.
- User documents, photos, videos, source code, databases, credentials, and config files are skipped by default.
- Unknown paths are treated as high risk.
- Prefer tool-native cleanup commands over raw recursive deletion.
- Cleanup must end with a space-recovered report.

## Risk Classification

### Low Risk

Usually safe to clean, but still requires user confirmation:

- System or application caches
- Temporary logs
- Package-manager caches
- Regenerable build outputs
- Xcode DerivedData
- Old temporary files

### Medium Risk

Suggest only, do not delete by default:

- Installers in the Downloads folder
- Archives
- Duplicate files
- Build directories of old projects
- Unused application installers

### High Risk

Skipped by default:

- Source code
- Documents
- Photos and videos
- Databases
- Application data directories
- Large files of unclear purpose

### Do Not Delete Automatically

Never touch these unless the user explicitly names them:

- Keys and certificates
- Account data
- Configuration files
- SSH/GPG-related files
- User-created content
- Data of unconfirmed ownership

## Universal Prompt

```text
You are my macOS storage cleanup assistant. Start with read-only audit only. Do not delete anything. Measure disk usage and large directories, list cleanup candidates, and classify them as low risk, medium risk, high risk, or do-not-delete. Provide path, size, type, risk, suggested action, and expected recovery. Only after I explicitly confirm may you generate and execute low-risk cleanup commands. After cleanup, verify recovered space and produce a cleanup report.
```

## Publishing As Open Source

Recommended repository layout:

```text
mac-storage-cleanup-agent/
├── README.md
├── README.zh-CN.md
├── PROMPT.md
├── SKILL.md
├── CLAUDE.md
└── LICENSE
```

Suggested repo name:

```text
mac-storage-cleanup-agent
```

Suggested description:

```text
A cautious AI agent workflow for macOS storage cleanup: audit first, classify risk, confirm deletion, verify recovered space.
```

Suggested topics:

```text
ai-agent
claude-code
codex
deepseek
macos
storage-cleanup
developer-tools
prompt-engineering
```

## License

Released under the [MIT License](LICENSE), so others can freely use, modify, and share this skill.
