# npm Installer Package Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship `@heycqing/mac-storage-cleanup-agent`, a publishable npm package with a `mac-storage-cleanup-agent` CLI that installs the existing `SKILL.md`/`CLAUDE.md`/`PROMPT.md` playbook content into Codex, Claude Code, or any AI assistant — replacing the manual `mkdir`+`cp` instructions in both READMEs with `npx` commands.

**Architecture:** A thin `bin/cli.js` entry point wires Node's `prompts` library and real OS calls into `src/cli.js`'s `main(argv, env)`, which parses flags, decides which of three installer targets (`src/targets/{codex,claude,prompt}.js`) to run, runs them, and prints a summary. Each target module reads the package's own `SKILL.md`/`CLAUDE.md`/`PROMPT.md` (via `src/lib/fs-utils.js`'s `PACKAGE_ROOT`) as its single source of truth, writes to a well-known location, and always resolves to a `{status, path, detail}` result — it never throws — so one target's failure can't crash the run or block the others. Tests inject a fake `env` object (`{homeDir, cwd, log, confirm, prompt}`) so production code never calls `os.homedir()`/`process.cwd()`/`prompts`/`console.log` directly, and every filesystem test runs inside an `fs.mkdtemp()` temp directory — never the developer's real `~/.codex` or this repo's own `CLAUDE.md`.

One small refinement versus the spec's illustrative file tree: rather than a single combined `test/targets.test.js`, tests are split per target — `test/targets/codex.test.js`, `test/targets/claude.test.js`, `test/targets/prompt.test.js` (mirroring `src/targets/*.js`), plus `test/fs-utils.test.js` and `test/cli.test.js`. This keeps each task's test changes self-contained (Task 4 never has to re-open the file Task 3 wrote) and each file focused on one module, per this project's stated file-structure preferences.

**Tech Stack:** Plain JavaScript (Node.js `>=18`), ESM (`"type": "module"`), no build step, one runtime dependency (`prompts@^2.4.2`), `node:test` + `node:assert/strict` for testing.

---

## Final file tree (after Task 9)

```text
mac-storage-cleanup-agent/
├── package.json
├── package-lock.json
├── .gitignore
├── bin/
│   └── cli.js
├── src/
│   ├── cli.js
│   ├── lib/
│   │   └── fs-utils.js
│   └── targets/
│       ├── codex.js
│       ├── claude.js
│       └── prompt.js
├── test/
│   ├── helpers/
│   │   └── fake-env.js
│   ├── fs-utils.test.js
│   ├── cli.test.js
│   └── targets/
│       ├── codex.test.js
│       ├── claude.test.js
│       └── prompt.test.js
├── CLAUDE.md / SKILL.md / PROMPT.md   (unchanged — read at runtime as the
│                                        single source of truth for distributed content)
├── README.md / README.zh-CN.md        (updated in Task 8)
└── LICENSE
```

---

### Task 1: Scaffold the package manifest and install dependencies

**Files:**
- Create: `package.json`
- Create: `.gitignore`

- [ ] **Step 1: Create `package.json`**

```json
{
  "name": "@heycqing/mac-storage-cleanup-agent",
  "version": "0.1.0",
  "description": "Install the Mac Storage Cleanup Agent skill into Codex, Claude Code, or any AI assistant — a cautious, audit-first workflow for freeing macOS disk space.",
  "type": "module",
  "bin": {
    "mac-storage-cleanup-agent": "./bin/cli.js"
  },
  "files": [
    "bin",
    "src",
    "CLAUDE.md",
    "SKILL.md",
    "PROMPT.md",
    "README.md",
    "README.zh-CN.md"
  ],
  "engines": {
    "node": ">=18"
  },
  "scripts": {
    "test": "node --test"
  },
  "keywords": [
    "ai-agent",
    "claude-code",
    "codex",
    "deepseek",
    "macos",
    "storage-cleanup",
    "cli",
    "installer",
    "developer-tools",
    "prompt-engineering"
  ],
  "author": "heycqing",
  "license": "MIT",
  "repository": {
    "type": "git",
    "url": "git+https://github.com/heycqing/mac-storage-cleanup-agent.git"
  },
  "bugs": {
    "url": "https://github.com/heycqing/mac-storage-cleanup-agent/issues"
  },
  "homepage": "https://github.com/heycqing/mac-storage-cleanup-agent#readme",
  "dependencies": {
    "prompts": "^2.4.2"
  }
}
```

- [ ] **Step 2: Create `.gitignore`**

```text
node_modules/
```

- [ ] **Step 3: Install dependencies**

Run: `npm install`

Expected: npm creates `node_modules/` and `package-lock.json`, and the final line of output is something like `added N packages in Xs` with no `npm error` lines.

- [ ] **Step 4: Commit**

```bash
git add package.json package-lock.json .gitignore
git commit -m "Add package manifest and dependencies"
```

---

### Task 2: Shared filesystem helpers

**Files:**
- Create: `src/lib/fs-utils.js`
- Test: `test/fs-utils.test.js`

- [ ] **Step 1: Write the failing test**

Create `test/fs-utils.test.js`:

```js
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { mkdtemp, readFile, writeFile } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import {
  PACKAGE_ROOT,
  pathExists,
  readPackageFile,
  writeFileEnsuringDir,
} from '../src/lib/fs-utils.js';

async function tempDir() {
  return mkdtemp(join(tmpdir(), 'mac-cleanup-fsutils-'));
}

test('PACKAGE_ROOT points at the package root containing package.json', async () => {
  assert.equal(await pathExists(join(PACKAGE_ROOT, 'package.json')), true);
});

test('pathExists returns true for an existing file', async () => {
  const dir = await tempDir();
  const file = join(dir, 'present.txt');
  await writeFile(file, 'hello', 'utf8');

  assert.equal(await pathExists(file), true);
});

test('pathExists returns false for a missing path', async () => {
  const dir = await tempDir();

  assert.equal(await pathExists(join(dir, 'missing.txt')), false);
});

test('readPackageFile reads a file relative to the package root', async () => {
  const content = await readPackageFile('package.json');

  assert.match(content, /"name": "@heycqing\/mac-storage-cleanup-agent"/);
});

test('writeFileEnsuringDir creates missing parent directories', async () => {
  const dir = await tempDir();
  const target = join(dir, 'nested', 'deeper', 'file.txt');

  await writeFileEnsuringDir(target, 'content');

  assert.equal(await readFile(target, 'utf8'), 'content');
});

test('writeFileEnsuringDir overwrites an existing file', async () => {
  const dir = await tempDir();
  const target = join(dir, 'file.txt');
  await writeFile(target, 'old', 'utf8');

  await writeFileEnsuringDir(target, 'new');

  assert.equal(await readFile(target, 'utf8'), 'new');
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `node --test test/fs-utils.test.js`
Expected: FAIL — Node reports the test file errored because `../src/lib/fs-utils.js` cannot be found (`Cannot find module` / `ERR_MODULE_NOT_FOUND`).

- [ ] **Step 3: Write the implementation**

Create `src/lib/fs-utils.js`:

```js
import { access, mkdir, readFile, writeFile } from 'node:fs/promises';
import { dirname, join } from 'node:path';
import { fileURLToPath } from 'node:url';

export const PACKAGE_ROOT = join(dirname(fileURLToPath(import.meta.url)), '..', '..');

export async function pathExists(targetPath) {
  try {
    await access(targetPath);
    return true;
  } catch {
    return false;
  }
}

export async function readPackageFile(filename) {
  return readFile(join(PACKAGE_ROOT, filename), 'utf8');
}

export async function writeFileEnsuringDir(targetPath, content) {
  await mkdir(dirname(targetPath), { recursive: true });
  await writeFile(targetPath, content, 'utf8');
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `node --test test/fs-utils.test.js`
Expected: PASS — output ends with `# pass 6` and `# fail 0`.

- [ ] **Step 5: Commit**

```bash
git add src/lib/fs-utils.js test/fs-utils.test.js
git commit -m "Add shared filesystem helpers"
```

---

### Task 3: Codex install target

**Files:**
- Create: `test/helpers/fake-env.js`
- Create: `src/targets/codex.js`
- Test: `test/targets/codex.test.js`

- [ ] **Step 1: Create the shared fake-environment test helper**

Every target test needs a fake `env` that records `confirm` prompts and `log` output instead of touching the real terminal. Create `test/helpers/fake-env.js`:

```js
export function fakeEnv({ homeDir, cwd, confirmAnswers = [] } = {}) {
  const logs = [];
  const confirmPrompts = [];
  let confirmIndex = 0;

  return {
    homeDir,
    cwd,
    log: (message) => logs.push(message),
    confirm: async (message, defaultValue) => {
      confirmPrompts.push(message);
      if (confirmIndex < confirmAnswers.length) {
        return confirmAnswers[confirmIndex++];
      }
      return defaultValue;
    },
    logs,
    confirmPrompts,
  };
}
```

`confirmAnswers` is a queue of canned answers; once it's exhausted, `confirm` returns whatever `defaultValue` the production code passed in — which is exactly how you verify "what does this target do under `-y`'s default" without supplying an answer at all.

- [ ] **Step 2: Write the failing test**

Create `test/targets/codex.test.js`:

```js
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { mkdtemp, readFile, writeFile, mkdir } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import { install } from '../../src/targets/codex.js';
import { fakeEnv } from '../helpers/fake-env.js';

async function tempDir() {
  return mkdtemp(join(tmpdir(), 'mac-cleanup-codex-'));
}

test('codex: installs SKILL.md when absent', async () => {
  const home = await tempDir();
  const env = fakeEnv({ homeDir: home });

  const result = await install(env, {});

  const targetPath = join(home, '.codex', 'skills', 'mac-storage-cleanup-agent', 'SKILL.md');
  assert.equal(result.status, 'installed');
  assert.equal(result.path, targetPath);
  assert.match(await readFile(targetPath, 'utf8'), /^---\nname: mac-storage-cleanup-agent/);
  assert.match(result.nextStep, /Tell Codex/);
});

test('codex: asks before overwriting an existing SKILL.md and respects "no"', async () => {
  const home = await tempDir();
  const targetDir = join(home, '.codex', 'skills', 'mac-storage-cleanup-agent');
  await mkdir(targetDir, { recursive: true });
  const targetPath = join(targetDir, 'SKILL.md');
  await writeFile(targetPath, 'old content', 'utf8');

  const env = fakeEnv({ homeDir: home, confirmAnswers: [false] });
  const result = await install(env, {});

  assert.equal(result.status, 'skipped');
  assert.equal(env.confirmPrompts.length, 1);
  assert.match(env.confirmPrompts[0], /overwrite/i);
  assert.equal(await readFile(targetPath, 'utf8'), 'old content');
});

test('codex: overwrites when the user confirms', async () => {
  const home = await tempDir();
  const targetDir = join(home, '.codex', 'skills', 'mac-storage-cleanup-agent');
  await mkdir(targetDir, { recursive: true });
  const targetPath = join(targetDir, 'SKILL.md');
  await writeFile(targetPath, 'old content', 'utf8');

  const env = fakeEnv({ homeDir: home, confirmAnswers: [true] });
  const result = await install(env, {});

  assert.equal(result.status, 'installed');
  assert.equal(result.detail, 'reinstalled (overwritten)');
  assert.match(await readFile(targetPath, 'utf8'), /^---\nname: mac-storage-cleanup-agent/);
});

test('codex: --yes overwrites without asking', async () => {
  const home = await tempDir();
  const targetDir = join(home, '.codex', 'skills', 'mac-storage-cleanup-agent');
  await mkdir(targetDir, { recursive: true });
  const targetPath = join(targetDir, 'SKILL.md');
  await writeFile(targetPath, 'old content', 'utf8');

  const env = fakeEnv({ homeDir: home });
  const result = await install(env, { yes: true });

  assert.equal(result.status, 'installed');
  assert.equal(env.confirmPrompts.length, 0);
  assert.match(await readFile(targetPath, 'utf8'), /^---\nname: mac-storage-cleanup-agent/);
});

test('codex: reports failure without crashing when the target directory cannot be created', async () => {
  const home = await tempDir();
  const skillsDir = join(home, '.codex', 'skills');
  await mkdir(skillsDir, { recursive: true });
  await writeFile(join(skillsDir, 'mac-storage-cleanup-agent'), 'a file blocking the directory', 'utf8');

  const env = fakeEnv({ homeDir: home });
  const result = await install(env, {});

  assert.equal(result.status, 'failed');
  assert.equal(result.path, join(skillsDir, 'mac-storage-cleanup-agent', 'SKILL.md'));
  assert.equal(typeof result.detail, 'string');
  assert.ok(result.detail.length > 0);
});
```

The last test simulates a filesystem error realistically: it creates a plain *file* at the path where the target *directory* needs to go, so `fs.mkdir(..., { recursive: true })` fails with `EEXIST` — caught by `install`'s try/catch, never thrown to the caller.

- [ ] **Step 3: Run the test to verify it fails**

Run: `node --test test/targets/codex.test.js`
Expected: FAIL — `Cannot find module '../../src/targets/codex.js'`.

- [ ] **Step 4: Write the implementation**

Create `src/targets/codex.js`:

```js
import { join } from 'node:path';
import { pathExists, readPackageFile, writeFileEnsuringDir } from '../lib/fs-utils.js';

const NEXT_STEP = 'Tell Codex: "Use the mac-storage-cleanup-agent skill to help me free '
  + 'disk space on my Mac. Only audit first, don\'t delete anything until I confirm."';

export async function install(env, options = {}) {
  const { homeDir, confirm } = env;
  const { yes = false } = options;
  const targetPath = join(homeDir, '.codex', 'skills', 'mac-storage-cleanup-agent', 'SKILL.md');

  try {
    const exists = await pathExists(targetPath);
    if (exists) {
      const overwrite = yes
        || await confirm(`Codex skill already installed at ${targetPath}. Overwrite?`, true);
      if (!overwrite) {
        return {
          status: 'skipped',
          path: targetPath,
          detail: 'already installed, left untouched (run with --yes to overwrite automatically)',
        };
      }
    }

    const content = await readPackageFile('SKILL.md');
    await writeFileEnsuringDir(targetPath, content);

    return {
      status: 'installed',
      path: targetPath,
      detail: exists ? 'reinstalled (overwritten)' : 'installed',
      nextStep: NEXT_STEP,
    };
  } catch (error) {
    return { status: 'failed', path: targetPath, detail: error.message };
  }
}
```

- [ ] **Step 5: Run the test to verify it passes**

Run: `node --test test/targets/codex.test.js`
Expected: PASS — output ends with `# pass 5` and `# fail 0`.

- [ ] **Step 6: Commit**

```bash
git add test/helpers/fake-env.js src/targets/codex.js test/targets/codex.test.js
git commit -m "Add Codex install target"
```

---

### Task 4: Claude Code install target

**Files:**
- Create: `src/targets/claude.js`
- Test: `test/targets/claude.test.js`

- [ ] **Step 1: Write the failing test**

Create `test/targets/claude.test.js`:

```js
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { mkdtemp, readFile, writeFile, mkdir } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import { install, MARKER } from '../../src/targets/claude.js';
import { fakeEnv } from '../helpers/fake-env.js';

async function tempDir() {
  return mkdtemp(join(tmpdir(), 'mac-cleanup-claude-'));
}

test('claude: writes a new CLAUDE.md with the marker when absent', async () => {
  const cwd = await tempDir();
  const env = fakeEnv({ cwd });

  const result = await install(env, {});

  const targetPath = join(cwd, 'CLAUDE.md');
  assert.equal(result.status, 'installed');
  const written = await readFile(targetPath, 'utf8');
  assert.ok(written.startsWith(`${MARKER}\n`));
  assert.match(written, /# Mac Storage Cleanup Agent Instructions/);
});

test('claude: skips when the marker is already present', async () => {
  const cwd = await tempDir();
  const targetPath = join(cwd, 'CLAUDE.md');
  const original = `${MARKER}\nAlready installed before.\n`;
  await writeFile(targetPath, original, 'utf8');

  const env = fakeEnv({ cwd });
  const result = await install(env, {});

  assert.equal(result.status, 'skipped');
  assert.equal(env.confirmPrompts.length, 0);
  assert.equal(await readFile(targetPath, 'utf8'), original);
});

test('claude: asks before appending to an unmarked CLAUDE.md and appends on "yes"', async () => {
  const cwd = await tempDir();
  const targetPath = join(cwd, 'CLAUDE.md');
  await writeFile(targetPath, '# My Project Instructions\n\nBe nice.\n', 'utf8');

  const env = fakeEnv({ cwd, confirmAnswers: [true] });
  const result = await install(env, {});

  assert.equal(result.status, 'installed');
  assert.equal(result.detail, 'appended to existing CLAUDE.md');
  assert.equal(env.confirmPrompts.length, 1);
  assert.match(env.confirmPrompts[0], /append/i);

  const written = await readFile(targetPath, 'utf8');
  assert.ok(written.startsWith('# My Project Instructions\n\nBe nice.\n\n<!-- mac-storage-cleanup-agent -->\n'));
  assert.match(written, /# Mac Storage Cleanup Agent Instructions/);
});

test('claude: leaves an unmarked CLAUDE.md untouched when the user declines to append', async () => {
  const cwd = await tempDir();
  const targetPath = join(cwd, 'CLAUDE.md');
  const original = '# My Project Instructions\n\nBe nice.\n';
  await writeFile(targetPath, original, 'utf8');

  const env = fakeEnv({ cwd, confirmAnswers: [false] });
  const result = await install(env, {});

  assert.equal(result.status, 'skipped');
  assert.equal(await readFile(targetPath, 'utf8'), original);
});

test('claude: --yes appends without asking', async () => {
  const cwd = await tempDir();
  const targetPath = join(cwd, 'CLAUDE.md');
  await writeFile(targetPath, '# My Project Instructions\n', 'utf8');

  const env = fakeEnv({ cwd });
  const result = await install(env, { yes: true });

  assert.equal(result.status, 'installed');
  assert.equal(env.confirmPrompts.length, 0);
  assert.match(await readFile(targetPath, 'utf8'), /<!-- mac-storage-cleanup-agent -->/);
});

test('claude: reports failure without crashing when CLAUDE.md path is a directory', async () => {
  const cwd = await tempDir();
  await mkdir(join(cwd, 'CLAUDE.md'), { recursive: true });

  const env = fakeEnv({ cwd });
  const result = await install(env, {});

  assert.equal(result.status, 'failed');
  assert.equal(typeof result.detail, 'string');
  assert.ok(result.detail.length > 0);
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `node --test test/targets/claude.test.js`
Expected: FAIL — `Cannot find module '../../src/targets/claude.js'`.

- [ ] **Step 3: Write the implementation**

Create `src/targets/claude.js`:

```js
import { join } from 'node:path';
import { readFile } from 'node:fs/promises';
import { pathExists, readPackageFile, writeFileEnsuringDir } from '../lib/fs-utils.js';

export const MARKER = '<!-- mac-storage-cleanup-agent -->';

export async function install(env, options = {}) {
  const { cwd, confirm } = env;
  const { yes = false } = options;
  const targetPath = join(cwd, 'CLAUDE.md');

  try {
    const content = await readPackageFile('CLAUDE.md');
    const distributed = `${MARKER}\n${content}`;
    const exists = await pathExists(targetPath);

    if (!exists) {
      await writeFileEnsuringDir(targetPath, distributed);
      return { status: 'installed', path: targetPath, detail: 'created CLAUDE.md' };
    }

    const existing = await readFile(targetPath, 'utf8');
    if (existing.includes(MARKER)) {
      return {
        status: 'skipped',
        path: targetPath,
        detail: 'CLAUDE.md already includes this skill, left untouched',
      };
    }

    const append = yes
      || await confirm(`${targetPath} already exists. Append the Mac Storage Cleanup Agent instructions to the end?`, true);
    if (!append) {
      return {
        status: 'skipped',
        path: targetPath,
        detail: 'left existing CLAUDE.md untouched (run with --yes to append automatically)',
      };
    }

    const merged = `${existing.trimEnd()}\n\n${distributed}\n`;
    await writeFileEnsuringDir(targetPath, merged);
    return { status: 'installed', path: targetPath, detail: 'appended to existing CLAUDE.md' };
  } catch (error) {
    return { status: 'failed', path: targetPath, detail: error.message };
  }
}
```

The marker `<!-- mac-storage-cleanup-agent -->` is an HTML comment placed on its own line directly above the distributed `CLAUDE.md` content — invisible when the instructions are rendered, but `.includes(MARKER)` makes "is this skill already here?" a one-line check on plain text.

- [ ] **Step 4: Run the test to verify it passes**

Run: `node --test test/targets/claude.test.js`
Expected: PASS — output ends with `# pass 6` and `# fail 0`.

- [ ] **Step 5: Commit**

```bash
git add src/targets/claude.js test/targets/claude.test.js
git commit -m "Add Claude Code install target"
```

---

### Task 5: Universal prompt target

**Files:**
- Create: `src/targets/prompt.js`
- Test: `test/targets/prompt.test.js`

- [ ] **Step 1: Write the failing test**

Create `test/targets/prompt.test.js`:

```js
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { mkdtemp, readFile, writeFile, mkdir } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import { install } from '../../src/targets/prompt.js';
import { fakeEnv } from '../helpers/fake-env.js';

async function tempDir() {
  return mkdtemp(join(tmpdir(), 'mac-cleanup-prompt-'));
}

test('prompt: prints the prompt and writes a local copy when absent', async () => {
  const cwd = await tempDir();
  const env = fakeEnv({ cwd });

  const result = await install(env, {});

  const targetPath = join(cwd, 'mac-cleanup-prompt.md');
  assert.equal(result.status, 'installed');
  assert.equal(result.path, targetPath);

  assert.match(await readFile(targetPath, 'utf8'), /macOS storage cleanup assistant/);

  const printed = env.logs.join('\n');
  assert.match(printed, /macOS storage cleanup assistant/);
  assert.ok(env.logs.length >= 3, 'expected at least: delimiter, content, delimiter');
  assert.ok(env.logs[0].includes('-----'));
});

test('prompt: asks before overwriting an existing local copy and respects "no"', async () => {
  const cwd = await tempDir();
  const targetPath = join(cwd, 'mac-cleanup-prompt.md');
  await writeFile(targetPath, 'old prompt copy', 'utf8');

  const env = fakeEnv({ cwd, confirmAnswers: [false] });
  const result = await install(env, {});

  assert.equal(result.status, 'skipped');
  assert.equal(env.confirmPrompts.length, 1);
  assert.match(env.confirmPrompts[0], /overwrite/i);
  assert.equal(await readFile(targetPath, 'utf8'), 'old prompt copy');
});

test('prompt: --yes overwrites the local copy without asking', async () => {
  const cwd = await tempDir();
  const targetPath = join(cwd, 'mac-cleanup-prompt.md');
  await writeFile(targetPath, 'old prompt copy', 'utf8');

  const env = fakeEnv({ cwd });
  const result = await install(env, { yes: true });

  assert.equal(result.status, 'installed');
  assert.equal(env.confirmPrompts.length, 0);
  assert.match(await readFile(targetPath, 'utf8'), /macOS storage cleanup assistant/);
});

test('prompt: reports failure without crashing when the target path is a directory', async () => {
  const cwd = await tempDir();
  await mkdir(join(cwd, 'mac-cleanup-prompt.md'), { recursive: true });

  const env = fakeEnv({ cwd });
  const result = await install(env, {});

  assert.equal(result.status, 'failed');
  assert.equal(typeof result.detail, 'string');
  assert.ok(result.detail.length > 0);
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `node --test test/targets/prompt.test.js`
Expected: FAIL — `Cannot find module '../../src/targets/prompt.js'`.

- [ ] **Step 3: Write the implementation**

Create `src/targets/prompt.js`:

```js
import { join } from 'node:path';
import { pathExists, readPackageFile, writeFileEnsuringDir } from '../lib/fs-utils.js';

const DELIMITER = '----- mac-storage-cleanup-agent prompt -----';

export async function install(env, options = {}) {
  const { cwd, confirm, log } = env;
  const { yes = false } = options;
  const targetPath = join(cwd, 'mac-cleanup-prompt.md');

  try {
    const content = await readPackageFile('PROMPT.md');

    log(DELIMITER);
    log(content.trimEnd());
    log(DELIMITER);

    const exists = await pathExists(targetPath);
    if (exists) {
      const overwrite = yes
        || await confirm(`${targetPath} already exists. Overwrite it with the latest prompt?`, true);
      if (!overwrite) {
        return {
          status: 'skipped',
          path: targetPath,
          detail: 'printed prompt; left existing mac-cleanup-prompt.md untouched '
            + '(run with --yes to overwrite automatically)',
        };
      }
    }

    await writeFileEnsuringDir(targetPath, content);
    return {
      status: 'installed',
      path: targetPath,
      detail: exists
        ? 'printed prompt and refreshed mac-cleanup-prompt.md'
        : 'printed prompt and wrote mac-cleanup-prompt.md',
    };
  } catch (error) {
    return { status: 'failed', path: targetPath, detail: error.message };
  }
}
```

Printing happens before the existence check so the user always sees the prompt — even in the failure case where writing the local copy doesn't work, they still got the copy-pasteable text on screen.

- [ ] **Step 4: Run the test to verify it passes**

Run: `node --test test/targets/prompt.test.js`
Expected: PASS — output ends with `# pass 4` and `# fail 0`.

- [ ] **Step 5: Commit**

```bash
git add src/targets/prompt.js test/targets/prompt.test.js
git commit -m "Add universal prompt install target"
```

---

### Task 6: CLI orchestration

**Files:**
- Create: `src/cli.js`
- Test: `test/cli.test.js`

- [ ] **Step 1: Write the failing test**

Create `test/cli.test.js`:

```js
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { mkdtemp } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import {
  TARGETS,
  HELP_TEXT,
  parseArgs,
  explicitTargetIds,
  selectTargetIds,
  formatSummary,
  main,
} from '../src/cli.js';
import { fakeEnv } from './helpers/fake-env.js';

test('parseArgs recognizes every documented flag', () => {
  const flags = parseArgs(['--codex', '--claude', '--prompt', '--all', '-y', '-h']);

  assert.deepEqual(flags, {
    codex: true,
    claude: true,
    prompt: true,
    all: true,
    yes: true,
    help: true,
  });
});

test('parseArgs accepts the long forms of -y and -h', () => {
  const flags = parseArgs(['--yes', '--help']);

  assert.equal(flags.yes, true);
  assert.equal(flags.help, true);
});

test('parseArgs rejects unknown options', () => {
  assert.throws(() => parseArgs(['--bogus']), /Unknown option: --bogus/);
});

test('explicitTargetIds returns an empty list when no target flags are set', () => {
  assert.deepEqual(explicitTargetIds(parseArgs([])), []);
});

test('explicitTargetIds returns only the requested targets, in TARGETS order', () => {
  assert.deepEqual(explicitTargetIds(parseArgs(['--prompt', '--codex'])), ['codex', 'prompt']);
});

test('explicitTargetIds returns every target for --all', () => {
  assert.deepEqual(explicitTargetIds(parseArgs(['--all'])), TARGETS.map((target) => target.id));
});

test('selectTargetIds skips the interactive prompt when flags are explicit', async () => {
  let promptCalled = false;
  const ids = await selectTargetIds(parseArgs(['--claude']), async () => {
    promptCalled = true;
    return { targets: [] };
  });

  assert.deepEqual(ids, ['claude']);
  assert.equal(promptCalled, false);
});

test('selectTargetIds falls back to an interactive multiselect when no flags are set', async () => {
  const ids = await selectTargetIds(parseArgs([]), async (question) => {
    assert.equal(question.type, 'multiselect');
    assert.equal(question.choices.length, TARGETS.length);
    assert.ok(question.choices.every((choice) => choice.selected === true));
    return { targets: ['codex', 'prompt'] };
  });

  assert.deepEqual(ids, ['codex', 'prompt']);
});

test('formatSummary groups results by status and lists next steps once', () => {
  const summary = formatSummary([
    {
      status: 'installed',
      path: '/home/.codex/skills/mac-storage-cleanup-agent/SKILL.md',
      label: 'Codex skill',
      detail: 'installed',
      nextStep: 'Tell Codex: "..."',
    },
    {
      status: 'skipped',
      path: '/project/CLAUDE.md',
      label: 'Claude Code',
      detail: 'CLAUDE.md already includes this skill, left untouched',
    },
    {
      status: 'failed',
      path: '/project/mac-cleanup-prompt.md',
      label: 'Universal prompt',
      detail: 'EACCES: permission denied',
    },
  ]);

  assert.match(summary, /^Installed:\n {2}✔ Codex skill → \/home\/\.codex\/skills\/mac-storage-cleanup-agent\/SKILL\.md/);
  assert.match(summary, /Skipped:\n {2}- Claude Code: CLAUDE\.md already includes this skill, left untouched/);
  assert.match(summary, /Failed:\n {2}✗ Universal prompt: EACCES: permission denied/);
  assert.match(summary, /Next steps:\n {2}Tell Codex: "\.\.\."/);
});

test('formatSummary lists each distinct next step only once', () => {
  const summary = formatSummary([
    { status: 'installed', path: '/a', label: 'A', detail: 'installed', nextStep: 'Do X' },
    { status: 'installed', path: '/b', label: 'B', detail: 'installed', nextStep: 'Do X' },
  ]);

  assert.equal(summary.match(/Do X/g).length, 1);
});

test('formatSummary omits sections that have no results', () => {
  const summary = formatSummary([
    { status: 'installed', path: '/a', label: 'A', detail: 'installed' },
  ]);

  assert.doesNotMatch(summary, /Skipped:/);
  assert.doesNotMatch(summary, /Failed:/);
  assert.doesNotMatch(summary, /Next steps:/);
});

test('main prints an unknown-option error and help text, and sets a failing exit code', async () => {
  const env = fakeEnv({ homeDir: '/home/fake', cwd: '/project/fake' });
  const originalExitCode = process.exitCode;

  try {
    await main(['--bogus'], { ...env, prompt: async () => { throw new Error('prompt should not be called'); } });
    assert.equal(process.exitCode, 1);
  } finally {
    process.exitCode = originalExitCode;
  }

  assert.ok(env.logs.some((line) => line.includes('Unknown option: --bogus')));
  assert.ok(env.logs.some((line) => line.includes('Usage: mac-storage-cleanup-agent')));
});

test('main prints help text and performs no installs when --help is passed', async () => {
  const env = fakeEnv({ homeDir: '/home/fake', cwd: '/project/fake' });

  await main(['--help'], { ...env, prompt: async () => { throw new Error('prompt should not be called'); } });

  assert.ok(env.logs.some((line) => line.includes('Usage: mac-storage-cleanup-agent')));
});

test('main reports "nothing to do" when the user selects no targets interactively', async () => {
  const env = fakeEnv({ homeDir: '/home/fake', cwd: '/project/fake' });

  await main([], { ...env, prompt: async () => ({ targets: [] }) });

  assert.ok(env.logs.some((line) => line.includes('No targets selected')));
});

test('main runs only the explicitly flagged target end-to-end', async () => {
  const dir = await mkdtemp(join(tmpdir(), 'mac-cleanup-cli-'));
  const env = fakeEnv({ homeDir: dir, cwd: dir });

  await main(
    ['--codex', '--yes'],
    { ...env, prompt: async () => { throw new Error('prompt should not be called when flags are explicit'); } },
  );

  const summary = env.logs.join('\n');
  assert.match(summary, /Installed:/);
  assert.match(summary, /Codex skill/);
  assert.doesNotMatch(summary, /Claude Code/);
  assert.doesNotMatch(summary, /Universal prompt/);
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `node --test test/cli.test.js`
Expected: FAIL — `Cannot find module '../src/cli.js'`.

- [ ] **Step 3: Write the implementation**

Create `src/cli.js`:

```js
import { install as installCodex } from './targets/codex.js';
import { install as installClaude } from './targets/claude.js';
import { install as installPrompt } from './targets/prompt.js';

export const TARGETS = [
  {
    id: 'codex',
    label: 'Codex skill',
    choiceTitle: 'Codex (~/.codex/skills)',
    install: installCodex,
  },
  {
    id: 'claude',
    label: 'Claude Code',
    choiceTitle: 'Claude Code (CLAUDE.md in this directory)',
    install: installClaude,
  },
  {
    id: 'prompt',
    label: 'Universal prompt',
    choiceTitle: 'Universal prompt (print + write a local copy)',
    install: installPrompt,
  },
];

export const HELP_TEXT = `Usage: mac-storage-cleanup-agent [options]

Install the Mac Storage Cleanup Agent skill into Codex, Claude Code, or any
AI assistant that accepts a pasted prompt.

Options:
  --codex          Install only the Codex skill
  --claude         Only write/merge CLAUDE.md in the current directory
  --prompt         Only emit the universal prompt output
  --all            Run all targets
  -y, --yes        Skip confirmation prompts and use safe defaults
  -h, --help       Show this help text

Examples:
  npx @heycqing/mac-storage-cleanup-agent
  npx @heycqing/mac-storage-cleanup-agent --codex
  npx @heycqing/mac-storage-cleanup-agent --all --yes`;

export function parseArgs(argv) {
  const flags = { codex: false, claude: false, prompt: false, all: false, yes: false, help: false };

  for (const arg of argv) {
    switch (arg) {
      case '--codex':
        flags.codex = true;
        break;
      case '--claude':
        flags.claude = true;
        break;
      case '--prompt':
        flags.prompt = true;
        break;
      case '--all':
        flags.all = true;
        break;
      case '-y':
      case '--yes':
        flags.yes = true;
        break;
      case '-h':
      case '--help':
        flags.help = true;
        break;
      default:
        throw new Error(`Unknown option: ${arg}`);
    }
  }

  return flags;
}

export function explicitTargetIds(flags) {
  if (flags.all) {
    return TARGETS.map((target) => target.id);
  }

  return TARGETS.filter((target) => flags[target.id]).map((target) => target.id);
}

export async function selectTargetIds(flags, promptFn) {
  const explicit = explicitTargetIds(flags);
  if (explicit.length > 0) {
    return explicit;
  }

  const response = await promptFn({
    type: 'multiselect',
    name: 'targets',
    message: 'Where do you want to install this skill?',
    choices: TARGETS.map((target) => ({ title: target.choiceTitle, value: target.id, selected: true })),
    min: 1,
  });

  return response.targets ?? [];
}

export function formatSummary(results) {
  const byStatus = { installed: [], skipped: [], failed: [] };
  for (const result of results) {
    byStatus[result.status].push(result);
  }

  const lines = [];

  if (byStatus.installed.length > 0) {
    lines.push('Installed:');
    for (const result of byStatus.installed) {
      lines.push(`  ✔ ${result.label} → ${result.path}`);
    }
  }

  if (byStatus.skipped.length > 0) {
    lines.push('Skipped:');
    for (const result of byStatus.skipped) {
      lines.push(`  - ${result.label}: ${result.detail}`);
    }
  }

  if (byStatus.failed.length > 0) {
    lines.push('Failed:');
    for (const result of byStatus.failed) {
      lines.push(`  ✗ ${result.label}: ${result.detail}`);
    }
  }

  const nextSteps = results
    .map((result) => result.nextStep)
    .filter((step, index, all) => Boolean(step) && all.indexOf(step) === index);

  if (nextSteps.length > 0) {
    lines.push('Next steps:');
    for (const step of nextSteps) {
      lines.push(`  ${step}`);
    }
  }

  return lines.join('\n');
}

export async function main(argv, env) {
  const { log, prompt } = env;

  let flags;
  try {
    flags = parseArgs(argv);
  } catch (error) {
    log(error.message);
    log('');
    log(HELP_TEXT);
    process.exitCode = 1;
    return;
  }

  if (flags.help) {
    log(HELP_TEXT);
    return;
  }

  const targetIds = await selectTargetIds(flags, prompt);
  if (targetIds.length === 0) {
    log('No targets selected. Nothing to do.');
    return;
  }

  const results = [];
  for (const id of targetIds) {
    const target = TARGETS.find((candidate) => candidate.id === id);
    const result = await target.install(env, { yes: flags.yes });
    results.push({ ...result, label: target.label });
  }

  log(formatSummary(results));
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `node --test test/cli.test.js`
Expected: PASS — output ends with `# pass 15` and `# fail 0`.

- [ ] **Step 5: Commit**

```bash
git add src/cli.js test/cli.test.js
git commit -m "Add CLI argument parsing and orchestration"
```

---

### Task 7: CLI entry point and smoke test

**Files:**
- Create: `bin/cli.js`

- [ ] **Step 1: Write the entry point**

Create `bin/cli.js`:

```js
#!/usr/bin/env node
import { homedir } from 'node:os';
import prompts from 'prompts';
import { main } from '../src/cli.js';

const env = {
  homeDir: homedir(),
  cwd: process.cwd(),
  log: (message) => console.log(message),
  confirm: async (message, initial) => {
    const response = await prompts({ type: 'confirm', name: 'value', message, initial });
    return response.value ?? false;
  },
  prompt: (question) => prompts(question),
};

main(process.argv.slice(2), env).catch((error) => {
  console.error(error.message);
  process.exitCode = 1;
});
```

`confirm` and `prompt` both fall back to a safe value (`false` / no targets) if the user cancels the prompt (e.g. Ctrl+C) — `prompts` resolves to `{}` on cancel, so `response.value ?? false` and `response.targets ?? []` keep the run from crashing or silently doing something the user didn't ask for.

- [ ] **Step 2: Mark the entry point executable in git**

Windows can't set the Unix executable bit through the filesystem, but git still records file mode in its index, and npm/Unix shells rely on that bit being set once the package is installed on macOS or Linux. Set it directly in git's index:

Run: `git update-index --chmod=+x bin/cli.js`
Expected: no output (the command is silent on success). Verify with `git diff --cached --summary`, which should show `mode change 100644 => 100755 bin/cli.js`.

- [ ] **Step 3: Smoke-test the CLI without touching real directories**

This project's own `CLAUDE.md` says to "resolve paths first, show them to the user, then execute only after confirmation." Apply the same caution to testing the installer itself: run it against a throwaway home directory and working directory, never your real `~/.codex` or this repo's own `CLAUDE.md`. `--all --yes` makes the whole run non-interactive (every `confirm` short-circuits to its safe default and the target list is explicit, so `prompts` is never invoked) — perfect for an automated check.

Run in PowerShell:

```powershell
$tmpHome = Join-Path $env:TEMP ("mac-cleanup-smoke-" + [guid]::NewGuid())
New-Item -ItemType Directory -Path $tmpHome | Out-Null
$originalUserProfile = $env:USERPROFILE
try {
  $env:USERPROFILE = $tmpHome
  Push-Location $tmpHome
  node "D:\mine-prop\mac-storage-cleanup-agent\bin\cli.js" --all --yes
} finally {
  Pop-Location
  $env:USERPROFILE = $originalUserProfile
}
Write-Host "--- files written under the fake home ---"
Get-ChildItem -Recurse $tmpHome | Select-Object FullName
Remove-Item -Recurse -Force $tmpHome
```

Expected:
- The command prints the universal prompt between two `----- mac-storage-cleanup-agent prompt -----` delimiter lines, followed by a summary that looks like:
  ```text
  Installed:
    ✔ Codex skill → <tmpHome>\.codex\skills\mac-storage-cleanup-agent\SKILL.md
    ✔ Claude Code → <tmpHome>\CLAUDE.md
    ✔ Universal prompt → <tmpHome>\mac-cleanup-prompt.md
  Next steps:
    Tell Codex: "Use the mac-storage-cleanup-agent skill to help me free disk space on my Mac. Only audit first, don't delete anything until I confirm."
  ```
- The `Get-ChildItem` listing shows exactly those three files under `$tmpHome` (plus the `.codex\skills\mac-storage-cleanup-agent` directories).
- `git status` in the repo shows **no** changes to `CLAUDE.md` (confirms the run never touched the repo's real working directory), and your real `~\.codex\skills\mac-storage-cleanup-agent` is unaffected (confirms it never touched your real home directory). `$env:USERPROFILE` is restored to its original value by the `finally` block even if the command errors.

- [ ] **Step 4: Commit**

```bash
git add bin/cli.js
git commit -m "Add CLI entry point"
```

---

### Task 8: Document the npx installer in both READMEs

**Files:**
- Modify: `README.zh-CN.md`
- Modify: `README.md`

Each step below replaces one section's *entire* contents (from its `##` heading line up to, but not including, the next `##` heading) or inserts a new section between two existing ones. The anchor headings are unique within each file, so you can locate each block with a simple search before replacing it.

- [ ] **Step 1: Insert "快速开始" into `README.zh-CN.md`**

Insert the following directly after the workflow code block (`先只读盘点 -> ... -> 最后复查空间` / closing ` ``` `) and before the `## 它能做什么` heading, so the section order becomes intro → workflow diagram → 快速开始 → 它能做什么 → …:

````text
## 快速开始

在你的项目目录下运行（不需要提前安装，`npx` 会自动下载并运行最新版本）：

```bash
npx @heycqing/mac-storage-cleanup-agent
```

会出现一个交互式列表，让你勾选要安装到 Codex、Claude Code，还是输出通用提示语（支持多选，默认全部勾选）。如果已经明确目标，也可以直接加参数跳过交互，例如 `npx @heycqing/mac-storage-cleanup-agent --codex`（完整参数说明见下面对应章节）。
````

- [ ] **Step 2: Replace "在 Codex 中使用" in `README.zh-CN.md`**

Replace the entire `## 在 Codex 中使用` section (it currently runs from that heading through the line just before `## 在 Claude Code 中使用`) with:

````text
## 在 Codex 中使用

最快的方式是运行：

```bash
npx @heycqing/mac-storage-cleanup-agent --codex
```

它会把 `SKILL.md` 写入：

```text
~/.codex/skills/mac-storage-cleanup-agent/SKILL.md
```

完成后重启 Codex，或新开一个 Codex 线程。

你可以这样说：

```text
使用 mac-storage-cleanup-agent skill 帮我清理 Mac 空间。
先只读盘点，不要删除任何文件，等我确认后再执行。
```

### 或者手动安装

把这个目录复制到 Codex skills 目录：

```bash
mkdir -p ~/.codex/skills
cp -R mac-storage-cleanup-agent ~/.codex/skills/
```

最终路径应该和上面一样，是 `~/.codex/skills/mac-storage-cleanup-agent/SKILL.md`。

````

- [ ] **Step 3: Replace "在 Claude Code 中使用" in `README.zh-CN.md`**

Replace the entire `## 在 Claude Code 中使用` section (from that heading through the line just before `## 在 DeepSeek 或其他 AI 中使用`) with:

````text
## 在 Claude Code 中使用

在你的项目目录下运行：

```bash
npx @heycqing/mac-storage-cleanup-agent --claude
```

它会在当前目录创建或更新 `CLAUDE.md`：目录里还没有 `CLAUDE.md` 时会直接写入；已经存在但还没有这份技能时会询问是否追加到末尾；已经包含这份技能时会跳过，不会重复写入。

然后对 Claude Code 说：

```text
按照 CLAUDE.md 里的 Mac Storage Cleanup Agent 流程帮我清理 macOS 存储。
先只读盘点，不要删除任何内容，等我确认清理计划后再执行。
```

### 或者手动安装

把 `CLAUDE.md` 放到项目或工作目录里，或者直接把其中内容复制给 Claude Code。

````

- [ ] **Step 4: Replace "在 DeepSeek 或其他 AI 中使用" in `README.zh-CN.md`**

Replace the entire `## 在 DeepSeek 或其他 AI 中使用` section (from that heading through the line just before `## 安全模型`) with:

````text
## 在 DeepSeek 或其他 AI 中使用

运行：

```bash
npx @heycqing/mac-storage-cleanup-agent --prompt
```

它会把 `PROMPT.md` 的内容打印到终端（方便直接复制），并在当前目录写一份 `mac-cleanup-prompt.md`，方便随时粘贴给 DeepSeek 或其他 AI。

也可以这样开始：

```text
请按照下面的 macOS 存储清理流程工作：先只读盘点，列出候选清理项并分类风险；不要删除任何文件；等我确认后再执行低风险清理；最后复查释放空间。
```

### 或者手动安装

直接复制 `PROMPT.md` 的内容给 AI。

````

- [ ] **Step 5: Insert "开发与发布" into `README.zh-CN.md`**

Insert the following directly after the "发布到 GitHub" topics code block (the one ending in `prompt-engineering` then ` ``` `) and before the `## 许可证` heading:

````text
## 开发与发布

如果你要修改这个 npm 安装包本身（而不是 playbook 内容）：

- 本地试运行：`node bin/cli.js`，或者先执行一次 `npm link`，之后就能直接用 `mac-storage-cleanup-agent` 命令调用。
- 运行测试：`npm test`。
- 发布到 npm（`npm publish`）是一个手动、需要谨慎确认的步骤，不会被自动触发。
````

- [ ] **Step 6: Insert "Quick Start" into `README.md`**

Insert the following directly after the intro paragraph (`...or any terminal-capable coding agent.`) and before the `## What It Does` heading:

````text
## Quick Start

Run this in your project directory (no install needed — `npx` downloads and runs the latest version):

```bash
npx @heycqing/mac-storage-cleanup-agent
```

You'll see an interactive checklist for installing into Codex, Claude Code, and/or printing the universal prompt (multiple selections allowed, all pre-checked). If you already know your target, skip the prompt with a flag, e.g. `npx @heycqing/mac-storage-cleanup-agent --codex` (full flag reference is in each section below).
````

- [ ] **Step 7: Replace "Use With Codex" in `README.md`**

Replace the entire `## Use With Codex` section (from that heading through the line just before `## Use With Claude Code`) with:

````text
## Use With Codex

The fastest way:

```bash
npx @heycqing/mac-storage-cleanup-agent --codex
```

This writes `SKILL.md` to:

```text
~/.codex/skills/mac-storage-cleanup-agent/SKILL.md
```

Restart Codex or open a new Codex thread so the skill list refreshes.

Ask Codex something like:

```text
Use the mac-storage-cleanup-agent skill to help me free disk space on my Mac.
Only audit first. Do not delete anything until I confirm.
```

### Or Install Manually

Copy this folder into your Codex skills directory:

```bash
mkdir -p ~/.codex/skills
cp -R mac-storage-cleanup-agent ~/.codex/skills/
```

The final path should look the same as above: `~/.codex/skills/mac-storage-cleanup-agent/SKILL.md`.

````

- [ ] **Step 8: Replace "Use With Claude Code" in `README.md`**

Replace the entire `## Use With Claude Code` section (from that heading through the line just before `## Use With DeepSeek Or Other Assistants`) with:

````text
## Use With Claude Code

Run this in the project or workspace directory where Claude Code should follow this workflow:

```bash
npx @heycqing/mac-storage-cleanup-agent --claude
```

It creates or updates `CLAUDE.md` in the current directory: writes it directly if one is missing, asks before appending if one exists without this skill, and skips it untouched if this skill is already present.

Then tell Claude Code:

```text
Use the Mac Storage Cleanup Agent workflow from CLAUDE.md. Start with read-only audit only. Do not delete anything until I explicitly confirm the cleanup plan.
```

### Or Install Manually

Copy `CLAUDE.md` into the project or workspace where Claude Code should follow this workflow, or paste its content into your Claude Code session.

````

- [ ] **Step 9: Replace "Use With DeepSeek Or Other Assistants" in `README.md`**

Replace the entire `## Use With DeepSeek Or Other Assistants` section (from that heading through the line just before `## Safety Model`) with:

````text
## Use With DeepSeek Or Other Assistants

Run:

```bash
npx @heycqing/mac-storage-cleanup-agent --prompt
```

This prints the contents of `PROMPT.md` to your terminal (ready to copy) and writes a local `mac-cleanup-prompt.md` in the current directory, so you can paste it into DeepSeek or any other assistant whenever you need it.

You can also start with:

```text
Follow this macOS storage cleanup workflow. First audit only, classify risk, propose cleanup commands, wait for confirmation, then verify recovered space.
```

### Or Install Manually

Paste the contents of `PROMPT.md` into the assistant directly.

````

- [ ] **Step 10: Insert "Development And Publishing" into `README.md`**

Insert the following directly after the "Publishing As Open Source" topics code block (the one ending in `prompt-engineering` then ` ``` `) and before the `## License` heading:

````text
## Development And Publishing

If you're changing this npm installer package itself (not the playbook content):

- Run it locally with `node bin/cli.js`, or run `npm link` once and then call it as `mac-storage-cleanup-agent`.
- Run the test suite with `npm test`.
- Publishing to npm (`npm publish`) is a manual, deliberate step and isn't automated by anything in this repo.
````

- [ ] **Step 11: Commit**

```bash
git add README.md README.zh-CN.md
git commit -m "Document npx installer usage in both READMEs"
```

---

### Task 9: Final verification

**Files:** none — this task only verifies the work from Tasks 1–8.

- [ ] **Step 1: Run the full automated test suite**

Run: `npm test`
Expected: every test file runs and the run ends with `# fail 0`. The `# pass` count should equal the total number of `test(...)` blocks across `test/fs-utils.test.js`, `test/targets/codex.test.js`, `test/targets/claude.test.js`, `test/targets/prompt.test.js`, and `test/cli.test.js` (6 + 5 + 6 + 4 + 15 = 36, if every step above was followed as written).

- [ ] **Step 2: Verify the published package contents**

Run: `npm pack --dry-run`
Expected: the "Tarball Contents" listing includes exactly: `package.json`, `LICENSE`, `README.md`, `README.zh-CN.md`, `CLAUDE.md`, `SKILL.md`, `PROMPT.md`, `bin/cli.js`, `src/cli.js`, `src/lib/fs-utils.js`, `src/targets/codex.js`, `src/targets/claude.js`, and `src/targets/prompt.js`. `LICENSE` is included automatically by npm even though it isn't in the `files` array. The listing must **not** include `node_modules/`, `test/`, `.gitignore`, `package-lock.json`, or `docs/`.

- [ ] **Step 3: Manually exercise the interactive prompts (recommended before publishing)**

Task 7's smoke test only covers the non-interactive `--all --yes` path — `prompts`' on-screen multiselect and confirm dialogs need a human at the keyboard at least once. Using the same temporary-`$env:USERPROFILE`-and-`cwd` technique as Task 7 (a fresh temp directory, restored in a `finally` block), run:

```powershell
node "D:\mine-prop\mac-storage-cleanup-agent\bin\cli.js"
```

with no flags. Use the arrow keys and spacebar to toggle the Codex / Claude Code / Universal-prompt checkboxes, press Enter to confirm the selection, and check that the on-screen wording reads naturally. Run it a second time in the same temp directory (so `CLAUDE.md` and the Codex `SKILL.md` already exist) and answer one of the "overwrite/append?" confirmations with "no" and the other with "yes" — confirm the declined one is left untouched and the accepted one updates as described in the spec. Delete the temp directory afterward.

- [ ] **Step 4: Confirm the working tree is clean**

Run: `git status`
Expected: `nothing to commit, working tree clean` — everything from Tasks 1–8 is already committed, so this just confirms no stray edits remain before you (the maintainer) decide when to run `npm publish` yourself.
