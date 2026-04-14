---
name: phonebase-skill-creator
description: Write PhoneBase app skills — the scripted automation packages that turn Android apps into pb CLI subcommands. Use this skill whenever the user asks to create a new skill, add commands to an existing skill, write automation scripts for an app, scaffold a skill, or debug/fix a skill script. Trigger phrases include "create a skill for X", "write a pb skill", "add a command to the skill", "scaffold a new skill", "automate X app", "write a script for pb", even if the user doesn't mention "skill" explicitly. This skill covers the full authoring workflow — research, scaffolding, scripting, testing, and iteration.
---

# PhoneBase Skill Creator

Write **app skills** — scripted automation packages that turn an Android app into `pb` CLI subcommands. After installing a skill, the user gets commands like `pb googleplay search "telegram"` or `pb gmail compose`.

## Workflow

Creating a skill is an iterative process:

1. **Research the app** — figure out its package name, deeplink schemes, key screens, and UI patterns
2. **Scaffold** — `pb skills new <name> --package <pkg>` to create the directory and extract the app icon
3. **Write scripts** — start with `open.js`, `close.js`, `state.js`, then add business commands
4. **Test on a real device** — run each command, check the output shape, try edge cases
5. **Iterate** — fix issues, handle dialog interruptions, improve stability

Don't try to get it perfect in one pass. Write the simplest version first, test it, then improve based on what actually happens on the device. Real Android devices are messy — apps update, dialogs pop up, UI elements move.

## Directory Layout

```
~/.phonebase/skills/<name>/
├── SKILL.md                    ← Required: frontmatter + docs
├── resources/
│   └── ic_launcher.webp        ← Required: app icon
└── scripts/                    ← One .js file = one command
    ├── _lib.js                 ← Shared utilities (underscore prefix = not a command)
    ├── open.js
    ├── close.js
    ├── state.js
    └── search.js
```

Rules:
- Files starting with `_` or `.` in `scripts/` are **not** registered as commands
- **One file = one command**. Filename becomes the command name (`search.js` → `pb <skill> search`)
- Skill name: lowercase `[a-z0-9_-]`, 1–64 chars, start with letter or digit

## SKILL.md Frontmatter

```yaml
---
name: instagram
display_name: Instagram
description: Instagram automation — launch, search, close
package: com.instagram.android
category: social
requires:
  - googleservices
---
```

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Must match directory name |
| `display_name` | No | Human-readable app name |
| `description` | No | One-line description |
| `package` | No | Android package name |
| `category` | No | Category slug |
| `requires` | No | Dependencies (auto-installed) |

Below the frontmatter is free-form markdown documentation (not parsed by the scanner).

## JSDoc Command Protocol

The **first `/** ... */` block** in each script declares the command:

```javascript
/**
 * @description 在应用内搜索
 * @description:en Search within the app
 * @arg keyword:string! 搜索关键词
 * @arg:en keyword Search keyword
 * @arg limit:int=20 最多返回几条
 * @arg tags:string* 过滤标签（可多个）
 */
```

### @arg syntax

`@arg <name>:<type>[modifier] [description]`

| Type | Values |
|------|--------|
| Types | `string` `int` `float` `bool` `path` |
| `!` | Required |
| `=<value>` | Optional with default |
| `*` | Multi-value (`--tags a --tags b`) |

### Reading args in script

```javascript
const { parseArgs } = require('node:util');
const { values } = parseArgs({
  options: {
    keyword: { type: 'string' },
    limit: { type: 'string' },
    tags: { type: 'string', multiple: true },
  },
});
```

## SDK API

```javascript
const pb = require('@phonebase-cloud/pb');
```

All methods return `Promise`. The SDK communicates with the pb daemon via Unix socket — `PHONEBASE_DEVICE_ID` is auto-injected.

### Device Control

| Method | Purpose |
|--------|---------|
| `pb.tap(x, y)` | Tap at coordinates |
| `pb.swipe(x1, y1, x2, y2)` | Swipe (Bézier trajectory) |
| `pb.input(text)` | Type into focused field (**appends**, does not replace) |
| `pb.keyevent(key)` | Send key by **string name**: `HOME` `BACK` `ENTER` `DEL` `MENU` etc. |

### App Management

| Method | Purpose |
|--------|---------|
| `pb.launch(pkg)` | Launch app's default Activity |
| `pb.startActivity(opts)` | Send custom Intent (`action`, `data`, `package_name`, `component`, `extras`) |
| `pb.forceStop(pkg)` | Kill app |
| `pb.topActivity()` | Get foreground Activity → `{package_name, class_name}` |
| `pb.packagesList()` | List installed packages |
| `pb.installPackage(opts)` | Install APK: `{uri: "https://..." }` or `{uri: "/sdcard/..."}` |
| `pb.uninstallPackage(pkg)` | Uninstall app |

### UI Observation

| Method | Purpose |
|--------|---------|
| `pb.dumpc()` | Compact UI tree (text, bounds, clickable) — **preferred** |
| `pb.dump()` | Full XML hierarchy (when you need parent-child) |
| `pb.screencap(opts?)` | Screenshot image |
| `pb.findTextInDump(dumpStr, text)` | Find node → `{bounds, center, line}` or `null` |
| `pb.waitText(text, opts?)` | Poll dumpc until text appears (default 10s, 0.5s interval) |
| `pb.tapText(text)` | Find + tap in one call |

### Other

| Method | Purpose |
|--------|---------|
| `pb.browse(url, pkg?)` | Open URL in browser |
| `pb.clipboardGet()` / `pb.clipboardSet(text)` | Clipboard |
| `pb.displayInfo()` | Screen resolution, density, rotation |
| `pb.shell(cmd)` | Raw shell → `{stdout, stderr, code}` |
| `pb.pushFile(local, remote?)` / `pb.pullFile(path)` | File transfer |
| `pb.listFiles(path)` | List directory |
| `pb.run(path, args?)` | Generic capability passthrough |

## Output Format

Your script's stdout is **only the business data JSON**. pb automatically wraps it:

```
Script stdout:          pb returns to user:
{"x": 1, "y": 2}  →   {"code": 200, "data": {"x": 1, "y": 2}, "msg": "OK"}
```

Three rules:

1. **Never add `status` field** — metadata belongs in pb's `code` field
2. **Use boolean / objects** — not string status codes (`{logged_in: false}` not `{status: "needs_login"}`)
3. **Failure = `process.exit(1)` + stderr** — pb converts non-zero exit to error response

Helper functions (put in `_lib.js`):

```javascript
function finish(data) {
  console.log(JSON.stringify(data));
  process.exit(0);
}

function fail(err, context) {
  console.error(`${context}: ${err.message}`);
  process.exit(1);
}
```

## Standard Commands

Every app skill should provide at least `open` and `close`. `state` is strongly recommended.

| Command | Responsibility | Returns | Does NOT return |
|---------|---------------|---------|-----------------|
| `open` | Launch only, return fast | `top_activity`, `foreground` | dump data, `logged_in` |
| `state` | Full introspection | `top_activity`, `foreground`, `logged_in`, `account`, `visible_texts` | — |
| `close` | Force-stop only | `top_activity` | extra data |

The reason for this separation: callers choose what they need. `open` is fast because it skips the dump. `state` is thorough because that's its job. Mixing them makes both slower and harder to reason about.

### open.js

```javascript
const pb = require('@phonebase-cloud/pb');
const { PACKAGE, sleep, isForeground, finish, fail } = require('./_lib.js');

async function main() {
  await pb.launch(PACKAGE);
  await sleep(2500);
  const top = await pb.topActivity();
  finish({ top_activity: top, foreground: isForeground(top) });
}

main().catch(err => fail(err, 'open'));
```

### state.js

```javascript
async function main() {
  const top = await pb.topActivity();
  const dumpStr = await pb.dumpc();
  const nodes = parseVisibleNodes(dumpStr);
  const login = detectLoginStatus(nodes);
  finish({
    top_activity: top,
    foreground: isForeground(top),
    logged_in: login.logged_in,
    account: login.account,
    visible_texts: nodes.map(n => n.text || n.content_desc).filter(Boolean).slice(0, 30),
  });
}
```

## _lib.js Essentials

Every skill needs dump parsing utilities:

```javascript
const PACKAGE = 'com.example.app';

function isForeground(top) {
  return top && top.package_name === PACKAGE;
}

function parseVisibleNodes(dumpStr) {
  if (typeof dumpStr !== 'string') return [];
  const nodes = [];
  for (const line of dumpStr.split('\n')) {
    const textMatch = line.match(/\btext="([^"]*)"/);
    const descMatch = line.match(/content-desc="([^"]*)"/);
    const text = textMatch ? textMatch[1] : '';
    const desc = descMatch ? descMatch[1] : '';
    if (!text && !desc) continue;
    const boundsMatch = line.match(/bounds=\[(\d+),(\d+)\]\[(\d+),(\d+)\]/);
    if (!boundsMatch) continue;
    const [x1, y1, x2, y2] = boundsMatch.slice(1, 5).map(v => parseInt(v, 10));
    nodes.push({
      text, content_desc: desc,
      bounds: [x1, y1, x2, y2],
      center: [Math.floor((x1 + x2) / 2), Math.floor((y1 + y2) / 2)],
      width: x2 - x1, height: y2 - y1,
      clickable: /\bclickable=true\b/.test(line),
    });
  }
  return nodes;
}

function parseScreenSize(dumpStr) {
  if (typeof dumpStr !== 'string') return null;
  const m = dumpStr.match(/Screen (\d+)x(\d+)/);
  return m ? { width: parseInt(m[1], 10), height: parseInt(m[2], 10) } : null;
}

function findClickableByText(nodes, candidates) {
  const normalized = candidates.map(c => c.toLowerCase());
  const clickable = nodes.filter(n => n.clickable);
  // exact → startsWith → contains (three-pass, most stable)
  for (const pass of ['exact', 'prefix', 'contains']) {
    for (const n of clickable) {
      const t = (n.text || '').toLowerCase();
      const d = (n.content_desc || '').toLowerCase();
      const match = pass === 'exact'
        ? normalized.includes(t) || normalized.includes(d)
        : pass === 'prefix'
        ? normalized.some(c => t.startsWith(c) || d.startsWith(c))
        : normalized.some(c => t.includes(c) || d.includes(c));
      if (match) return n;
    }
  }
  return null;
}

function sleep(ms) { return new Promise(r => setTimeout(r, ms)); }
function finish(data) { console.log(JSON.stringify(data)); process.exit(0); }
function fail(err, ctx) { console.error(`${ctx}: ${err.message}`); process.exit(1); }

module.exports = {
  PACKAGE, isForeground, parseVisibleNodes, parseScreenSize,
  findClickableByText, sleep, finish, fail,
};
```

## Stability Patterns

### 1. Deeplinks over UI taps

Deeplinks are faster, more stable, and survive app version updates. Always check if the app supports them before writing tap sequences:

```javascript
await pb.startActivity({
  action: 'android.intent.action.VIEW',
  data: 'market://search?q=WhatsApp&c=apps',
  package_name: 'com.android.vending',
});
```

Why this matters: a deeplink is one API call. The UI tap equivalent is "tap search icon → wait for keyboard → type text → tap search button" — four steps, each of which can fail if the UI layout changes.

### 2. Clear input before typing

`pb.input()` **appends** text. If the user runs the same search command twice, the second keyword gets appended to the first:

```javascript
async function clearFocusedField(maxChars = 200) {
  await pb.shell('input keyevent KEYCODE_MOVE_END');
  await sleep(150);
  await pb.shell(`for i in $(seq 1 ${maxChars}); do input keyevent KEYCODE_DEL; done`);
  await sleep(300);
}
```

### 3. Handle dialog interruptions

Apps often interrupt with Google Sign-In, permission dialogs, or update prompts. Your script needs to handle these or it will get stuck:

```javascript
async function dismissDialogs(targetPkg, maxAttempts = 3) {
  for (let i = 0; i < maxAttempts; i++) {
    const top = await pb.topActivity();
    if (top.package_name === targetPkg) return top;
    if (top.package_name === 'com.google.android.gms') {
      await pb.keyevent('BACK');
      await sleep(800);
      continue;
    }
    const nodes = parseVisibleNodes(await pb.dumpc());
    const dismiss = findClickableByText(nodes, [
      'Accept', 'Agree', 'Got it', 'OK', 'Continue',
      'Not now', 'Skip', 'No thanks', 'Maybe later',
    ]);
    if (dismiss) {
      await pb.tap(dismiss.center[0], dismiss.center[1]);
      await sleep(1000);
    } else {
      break;
    }
  }
  return pb.topActivity();
}
```

### 4. NAF (Not Accessibility Friendly) buttons

Icon-only buttons without text or content-desc can't be found by text. Use geometry instead — "the clickable element in the top-right corner":

```javascript
function findTopRightIcon(clickableNodes, screenSize) {
  const candidates = clickableNodes.filter(n => {
    const [cx, cy] = n.center;
    return cy < screenSize.height * 0.15
        && cx > screenSize.width * 0.8
        && n.width < screenSize.width * 0.9;
  });
  candidates.sort((a, b) => b.center[0] - a.center[0]);
  return candidates[0] || null;
}
```

### 5. Polling with timeout

For long-running operations like app installation:

```javascript
async function pollUntil(checkFn, { timeout = 30, interval = 2 } = {}) {
  const deadline = Date.now() + timeout * 1000;
  while (Date.now() < deadline) {
    const result = await checkFn();
    if (result) return result;
    await sleep(interval * 1000);
  }
  return null;
}
```

### 6. Login detection

Detect login pages via Activity class name + visible text:

```javascript
const LOGIN_HINTS = ['SignUp', 'Login', 'Welcome', 'Auth', 'Onboard'];
const LOGIN_TEXTS = ['Log in', 'Sign up', 'Sign in', 'Create account',
  'Continue with Google', 'Continue with Facebook'];

function isLoginPage(topActivity, nodes) {
  const cls = (topActivity.class_name || '').toLowerCase();
  if (LOGIN_HINTS.some(h => cls.includes(h.toLowerCase()))) return true;
  return nodes.some(n => LOGIN_TEXTS.some(h => (n.text || '').startsWith(h)));
}
```

## Writing Principles

- **Explain the why.** Instead of "NEVER use resource-id", write "Don't rely on resource-id — apps like TikTok and Douyin obfuscate and change IDs every release. Use `text`, `content-desc`, or geometry instead." The model is smart; when it understands the reason, it makes better judgment calls in edge cases.

- **Generalize from examples.** You're testing on one device with one app version, but the skill will run across many devices and versions. Don't overfit to a specific UI layout. Prefer text matching over coordinate matching, deeplinks over tap sequences.

- **Keep it lean.** If a pattern isn't pulling its weight, remove it. Read the test run transcripts — if the script wastes time on something unproductive, cut that part.

## Anti-Patterns

| Don't | Why |
|-------|-----|
| Add `status` field to output | pb's `code` field handles transport metadata |
| Put introspection in `open` | `open` = launch only, `state` = inspect |
| Rely on `resource-id` | Many apps obfuscate and rotate IDs each release |
| Hardcode coordinates | Priority: `pb.tapText()` → geometry → hardcoded (last resort, with comment) |
| `console.log()` debug info | stdout is JSON only. Debug goes to `console.error()` (stderr) |
| Use numeric key codes | `pb.keyevent(4)` fails. Use string names: `pb.keyevent('BACK')` |
| Skip clearing before input | `pb.input()` appends. Always clear in repeat scenarios |
| Multiple commands per file | One file = one command |

## Testing

```bash
# Scaffold
pb skills new <name> --package <pkg>

# Validate structure + JSDoc
pb skills validate <name>

# Run on device
pb -s <device_id> <skill> <command> --keyword "test"

# Verify output shape
pb -s <device_id> <skill> state | python3 -c "
import sys, json
o = json.load(sys.stdin)
assert o['code'] == 200
assert 'status' not in o['data']
print('OK')
"
```

Always test on a real device. No mocks.

## Reference Skills

When writing a new skill, study existing implementations in [phonebase-skill-hub](https://github.com/phonebase-cloud/phonebase-skill-hub):

- **googleplay** — deeplink-driven, login detection via email regex, install polling with timeout
- **gmail** — complex dump parsing, mailto deeplink, compose page validation
- **tiktok** — NAF button detection, login page double-check, Google Sign-In dismissal
- **googleservices** — account login orchestration shared across Google apps
