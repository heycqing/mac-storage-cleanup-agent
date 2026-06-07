# npm Installer Package — Design

Date: 2026-06-07
Status: Approved (pending spec review)

## Background

`mac-storage-cleanup-agent` is currently a pure-Markdown skill/prompt playbook
(`CLAUDE.md`, `SKILL.md`, `PROMPT.md`, two READMEs, `LICENSE`) with no
`package.json` or npm infrastructure. The "Use With Codex / Claude Code /
DeepSeek" sections in both READMEs describe **manual** installation:

- Codex: `mkdir -p ~/.codex/skills && cp -R mac-storage-cleanup-agent ~/.codex/skills/`
- Claude Code: copy `CLAUDE.md` into the project directory, or paste its content
- DeepSeek/other: paste the contents of `PROMPT.md`

The user wants these steps replaced by an installable npm package — a real,
publishable package (not just docs describing a hypothetical flow), modeled on
common `npx` CLI installer conventions (e.g. shadcn/`create-*` style
interactive scaffolding tools).

## Goals

- Ship a real npm package, `@heycqing/mac-storage-cleanup-agent`, with a CLI
  binary `mac-storage-cleanup-agent`, runnable via
  `npx @heycqing/mac-storage-cleanup-agent`.
- The package's job is **distribution**, not reimplementation: it copies/writes
  the existing `SKILL.md` / `CLAUDE.md` / `PROMPT.md` content into the right
  places for Codex, Claude Code, and "universal" (DeepSeek/other) usage. The
  project remains an AI-executed playbook; the npm package is just the delivery
  mechanism, replacing manual `mkdir`+`cp`.
- Update both READMEs so the documented installation path matches what the
  package actually does.

## Non-goals

- Not building an actual disk-cleanup CLI (no scanning/classification/deletion
  logic in code — that remains the AI agent's job per the existing prompts).
- Not publishing to the npm registry as part of this work — the user will run
  `npm publish` themselves once the package is ready.
- No Windows-specific variant of the cleanup playbook (explicitly out of scope
  per user request — separate future effort).
- No clipboard integration (rejected in favor of zero extra dependencies).

## Package Identity

- npm package name: `@heycqing/mac-storage-cleanup-agent` (scoped to match the
  `LICENSE` copyright holder and avoid name collisions on the registry)
- CLI bin name: `mac-storage-cleanup-agent` (matches the package's unscoped
  name and the GitHub repo `heycqing/mac-storage-cleanup-agent`, so it's
  recognizable across npx examples, README text, and the repo)
- License: MIT (matches existing `LICENSE`)
- Node engine requirement: `>=18` (repo's Node is v20.20.2; `fs.cp`,
  `node:test`, and stable ESM are all available)

## Architecture

Modular structure — one small module per distribution target — chosen over a
single monolithic CLI script (harder to test/extend in isolation) and over a
scaffolding-framework approach like `degit`/Yeoman (designed for templating new
projects from templates, not for installing fixed files into known tool
directories; would add dependency weight against the "light dependencies"
goal).

```text
mac-storage-cleanup-agent/
├── package.json              # name=@heycqing/mac-storage-cleanup-agent, bin=mac-storage-cleanup-agent
├── bin/
│   └── cli.js                # entry: parse argv → interactive multi-select if no target flags → dispatch
├── src/
│   ├── targets/
│   │   ├── codex.js          # copy SKILL.md → ~/.codex/skills/mac-storage-cleanup-agent/
│   │   ├── claude.js         # write/merge CLAUDE.md into cwd
│   │   └── prompt.js         # print PROMPT.md + write a local copy in cwd
│   └── lib/
│       └── fs-utils.js       # shared helpers: home-dir resolution, copy, "exists → ask" flow
├── test/
│   └── targets.test.js       # node:test, runs against fs.mkdtemp() temp dirs only
├── CLAUDE.md / SKILL.md / PROMPT.md   # unchanged — the CLI reads these as the
│                                        single source of truth for distributed content
├── README.md / README.zh-CN.md        # updated "how to use" sections
└── LICENSE
```

**Tech stack:** plain JavaScript, ESM, no build step. One runtime dependency:
`prompts` (small, widely used for exactly this kind of interactive multi-select
— e.g. shadcn's CLI). Argument parsing is hand-rolled (only ~5 boolean flags;
not worth a dependency for that).

The CLI reads `CLAUDE.md`, `SKILL.md`, and `PROMPT.md` directly from the
package's installed location (resolved relative to `import.meta.url`, included
via `package.json`'s `files` field) — there is no second copy of this content
to keep in sync.

## CLI Behavior

**Invocation:** `npx @heycqing/mac-storage-cleanup-agent` (or, once installed
globally, `mac-storage-cleanup-agent`).

**Flags:**

| Flag | Effect |
|---|---|
| `--codex` | install only the Codex target |
| `--claude` | only write/merge `CLAUDE.md` into the current directory |
| `--prompt` | only emit the universal PROMPT output |
| `--all` | run all three targets |
| `-y`, `--yes` | skip "file exists, overwrite/merge?" prompts; pick the safe default per target (see below) |
| `-h`, `--help` | print usage |

Flags are combinable (e.g. `--codex --prompt`).

**Flow:**

1. Parse `argv`. If any of `--codex`/`--claude`/`--prompt`/`--all` is present,
   skip the interactive prompt and run exactly the requested target(s).
2. Otherwise, show a `prompts` multi-select: "Where do you want to install
   this skill?" with Codex / Claude Code (current project) / Universal Prompt
   output, all pre-selected.
3. Run each selected target's installer module. Each returns a result —
   `{ status: "installed" | "skipped" | "failed", path, detail }`.
4. Print a summary report, echoing the report style already established in
   this project's `CLAUDE.md` ("Recovered / Cleaned / Skipped / ..."). **All
   CLI output — prompts, this summary, `--help` text — is in English**,
   matching npm/CLI convention and the package's primary (English) identity;
   this is a separate decision from the project's bilingual *playbook* docs
   (`README.md` / `README.zh-CN.md`), which stay as they are:

   ```text
   Installed:
     ✔ Codex skill → ~/.codex/skills/mac-storage-cleanup-agent/SKILL.md
   Skipped:
     - Claude Code: CLAUDE.md already exists in this directory, left untouched
       (run with --yes to merge automatically)
   Next steps:
     Tell Codex: "Use the mac-storage-cleanup-agent skill to help me free disk
     space on my Mac. Only audit first, don't delete until I confirm."
   ```

   The "Next steps" section reuses the existing "you can say this to the AI"
   snippets from the README (translated to English for CLI output), so the
   user doesn't need to flip back to the docs after installing.

## Per-Target Behavior

### Codex (`src/targets/codex.js`)

- Target path: `path.join(os.homedir(), '.codex', 'skills', 'mac-storage-cleanup-agent', 'SKILL.md')`
- Existence check is on the **`SKILL.md` file itself**, not just the parent
  directory (the directory may exist with unrelated contents).
- File not present (directory may or may not exist) → `fs.mkdir(dir, { recursive: true })`,
  then copy the package's `SKILL.md` into it. Only `SKILL.md` is copied — not
  the whole repo — because the file is fully self-contained (its frontmatter
  `name`/`description` is the entire manifest Codex needs, and it doesn't
  reference any sibling files or resources).
- File already present → ask "already installed, overwrite?" (default **yes**
  under `-y`, since reinstall/upgrade is the expected reason to run this again
  and overwriting a single self-contained skill file is lossless).

### Claude Code (`src/targets/claude.js`)

- Target path: `path.join(process.cwd(), 'CLAUDE.md')`
- Not present → write the package's `CLAUDE.md` content directly.
- Present **and** already contains this skill's marker (a one-line HTML
  comment, e.g. `<!-- mac-storage-cleanup-agent -->`, embedded at the top of
  the distributed content) → report "already installed, skipped".
- Present **without** the marker → ask "append this skill's instructions to
  the end of the existing CLAUDE.md?" (default **yes** under `-y`, because
  `CLAUDE.md` is designed to hold multiple stacked instruction blocks and
  appending is non-destructive — unlike overwriting, which could discard the
  user's existing project instructions).

### Universal / Prompt (`src/targets/prompt.js`)

- Print the package's `PROMPT.md` content to stdout, wrapped in clear
  delimiter lines so the user can select-and-copy the whole block easily.
- Also write a copy to `path.join(process.cwd(), 'mac-cleanup-prompt.md')`.
- Existing file at that path → ask "overwrite?" (default **yes** under `-y`;
  this is a plain output file, overwriting is lossless).

## Error Handling

- Resolve paths via `os.homedir()` / `process.cwd()` (cross-platform-correct;
  matches Node's recommended approach).
- Always `fs.mkdir(dir, { recursive: true })` before writing.
- Filesystem errors (e.g. permission denied) are caught per-target, recorded
  as `{ status: "failed", path, detail: <error message> }`, and surfaced in the
  summary's failed/skipped section — one target's failure does not stop the
  others from running or crash the process.
- Guiding principle across all targets: **never overwrite without the user's
  consent** — either interactively (a prompt, answered per run) or upfront
  (`-y`/`--yes`, the user's blanket consent to "use the safe defaults and don't
  ask me each time"). `-y` is not a way around asking; it *is* the answer,
  given once for the whole run. Within that, the default each target picks
  under `-y` differs by whether the action is lossless (Codex skill reinstall,
  prompt output file → default yes) or could discard user content (CLAUDE.md
  full overwrite — which is why that case is designed as "append" rather than
  "overwrite" in the first place, making default-yes safe there too).

## Testing

- Use Node's built-in `node:test` + `node:assert` (no extra test-framework
  dependency, consistent with the "light dependencies" goal).
- Each test runs inside a directory created by `fs.mkdtemp(path.join(os.tmpdir(), ...))`.
  Target modules accept the home directory and working directory as parameters
  (or read them through a small overridable resolver in `fs-utils.js`) so tests
  can redirect both — **the test suite must never write to the developer's real
  `~/.codex` or the repo's own working directory**.
- Scenarios covered per target: destination absent (creates correctly),
  present-and-marked (skips), present-and-unmarked (asks/appends per the
  matrix above), and a simulated filesystem error (caught, reported, doesn't
  crash the run).

## Documentation Updates

Both `README.md` and `README.zh-CN.md`:

- Add a top-level "Quick Start" / "快速开始" section near the top showing the
  single interactive command: `npx @heycqing/mac-storage-cleanup-agent`.
- In each of the existing "Use With Codex / Claude Code / DeepSeek" sections,
  lead with the equivalent targeted npx command (e.g.
  `npx @heycqing/mac-storage-cleanup-agent --codex`), and keep the existing
  manual `mkdir`+`cp` / copy-paste steps below it labeled as "or install
  manually" (for users who can't or prefer not to run `npx`, e.g. restricted
  network environments).
- Add a short "Development / Publishing" note aimed at the maintainer: how to
  test locally (`npm link` or `node bin/cli.js`), and that publishing
  (`npm publish`) is a manual, deliberate step — not automated by this change.

## Open Items / Things the implementation plan should pin down

- Exact wording/format of the marker comment embedded in the distributed
  `CLAUDE.md` content (must not visually clutter the instructions when a user
  reads them).
- Exact `package.json` metadata fields (`description`, `keywords`, `repository`,
  `bugs`, `homepage` — `repository` should point at
  `git+https://github.com/heycqing/mac-storage-cleanup-agent.git`).
- Initial version number (suggest `0.1.0` for a first publishable release).
