# Git-backed memory for Claude Code (and Kiro): setup for a new project

Plain markdown in git, plus five hooks. The same files and hook scripts also work in Kiro (section 8). No vector database, no memory server, no 2,000-line `CLAUDE.md`.

**The idea in one line:** keep *how to work* apart from *what happened*, keep both in git, have Claude read the right slice at the start and write back at the end.

- **How to work** = rules, split into one small skill per job (deploy, writing, UI, data…). Each rule has a weight and a why.
- **What happened** = a short project card (status, where things live, open tasks), a dated log and a decisions file.
- **Hooks** make the notes act like memory. The harness runs them every time, so Claude can't skip them: inject the card at session start, route each prompt to a skill, block risky commands until the right skill is loaded, check written files for known mistakes, and nudge Claude to write the log at the end.

Needs Node 18+ (the hooks are plain `.mjs` with no dependencies). Works on Windows, macOS and Linux.

---

## 1. Folder layout

```
<project>/
├── CLAUDE.md                     # short: commands, code map, invariants, pointer to memory
├── docs/memory/
│   ├── project-card.md           # injected at every session start; keep under ~60 lines
│   ├── log.md                    # dated, append-only; opened on demand
│   └── decisions.md              # dated decisions with the why
└── .claude/
    ├── settings.json             # hook wiring (committed)
    ├── skills/
    │   └── <job>/SKILL.md        # one per job: deploy, writing, ui, …
    └── hooks/
        ├── config.mjs            # the only file you edit per project
        ├── lib.mjs
        ├── session-start.mjs
        ├── route-prompt.mjs
        ├── guard-command.mjs
        ├── check-written.mjs
        └── stop-nudge.mjs
```

Add this to `.gitignore`:

```
.claude/.state/
.claude/settings.local.json
```

**Quick start:** copy `.claude/hooks/` and `.claude/settings.json` from a project that already has them, edit `config.mjs`, then write the three memory files and one skill. Or paste the prompt in section 7 into Claude Code in the new repo.

---

## 2. Memory files (templates)

### `docs/memory/project-card.md`

Only facts Claude can't guess from the code. One fact lives in one place.

```markdown
# Project card: <name>

## Summary
<2-3 sentences: what it does, for whom, main stack.>

## Where things live
- <area>: `<path>`
- Spec / design doc: `<path>`

## Environments
- dev: <url / account / region / how it's deployed>
- prod: <…>. Never touch without asking.
- <Facts like "staging is at X", "never edit folder Y", "deploys run from CI, not locally".>

## Last status (prune by hand every few weeks)
- YYYY-MM-DD: <one line>

## Open tasks
- [ ] <task>
```

### `docs/memory/log.md`

```markdown
# Work log

Append-only, newest at the bottom. Format: `- YYYY-MM-DD: what changed (files/commit)`.

- YYYY-MM-DD: set up git-backed memory (docs/memory, .claude/skills, .claude/hooks).
```

### `docs/memory/decisions.md`

```markdown
# Decisions

Dated, with the why. This makes "why did we do X in September" answerable.

## YYYY-MM-DD: <decision>
**Why:** <reason>
**Instead of:** <the option we didn't take>
```

### `CLAUDE.md`: add this section

```markdown
## Memory (read first, write back at the end)
- What happened: `docs/memory/` (project-card.md is injected at session start; log.md and decisions.md on demand).
- How to work: `.claude/skills/<job>/SKILL.md`. Load the matching skill before the job.
- At the end of a task that changed files: append dated lines to log.md and update Last status / Open tasks in the card.
- "From now on never/always X": add a dated "User decided on <date>" line to the right skill, the same turn.
```

---

## 3. Skills (one per job)

`.claude/skills/<job>/SKILL.md`. The `description` decides when Claude loads it, so say exactly what job it covers.

```markdown
---
name: deploy
description: Rules for deploying, releasing or tearing down <project> environments. Load before running or writing any deploy command.
---

# Deploying <project>

Each rule has a weight (MUST / NEVER / PREFER) and a why. "User decided" lines win.

## User decisions
- User decided on YYYY-MM-DD: <rule>. **Why:** <incident or reason>.

## Rules
- MUST <do X>. **Why:** <reason>.
- NEVER <do Y>. **Why:** <reason>.
- PREFER <Z>. **Why:** <reason>.

## After the job
Append a dated line to `docs/memory/log.md`; update Environments / Last status in the card.
```

How to grow a skill:
- **Rules come from incidents.** When Claude does something you dislike, add one dated line, not a paragraph. The file turns into a changelog of fixes.
- **Always include the why.** Claude follows a rule it understands and argues with a bare command.
- **Facts beat behaviour rules.** "Staging is at X, never touch folder Y" changes more than "be careful".
- **Keep each skill under ~50 lines.** Split it when it grows.

---

## 4. Hooks

### `.claude/hooks/config.mjs` (edit this per project)

```js
// Per-project settings for the memory hooks. Everything else in .claude/hooks is generic.

export const MEMORY_DIR = "docs/memory"; // card, log, decisions
export const LOG_TAIL = 5;               // log lines shown at session start

// Short rules injected at every session start (and after compaction).
export const CORE_RULES = [
  "Read the project card below before acting. History: docs/memory/log.md; reasons: docs/memory/decisions.md (open on demand).",
  "Load the matching skill before the job (see .claude/skills).",
  "At the end of a task that changed files: append dated lines to docs/memory/log.md and update Last status / Open tasks in the card.",
  '"From now on never/always X" from the user: add a dated "User decided on <date>" line to the right skill, the same turn.',
  "Commit only when the user asks.",
];

// Prompt router: a prompt that matches `re` suggests loading `skill`.
export const ROUTES = [
  { skill: "deploy", re: /\b(deploy|release|rollback|terraform|cdk|helm|kubectl)\b/i },
  // { skill: "writing", re: /\b(blog|docs|copy|email|newsletter)\b/i },
];

// Command guard: a shell command that matches `re` is denied until `skill` is loaded;
// if it also matches `ask`, the human must confirm.
export const GUARDS = [
  {
    skill: "deploy",
    re: /\b(cdk|terraform|pulumi)\s+(deploy|apply|destroy|up)\b|\bnpm\s+run\s+deploy\b/,
    ask: /\bprod(uction)?\b|\bdestroy\b/,
  },
];

// Known-mistake checks run on every written file. `only` limits a check to matching paths.
export const CHECKS = [
  // { re: /console\.log\(/, only: /^src\//, why: "use the logger, not console.log" },
  // { re: /TODO\(me\)/, why: "resolve personal TODOs before finishing" },
];

// Files the checks skip (for example the files that define a banned term).
export const CHECK_EXEMPT = [/^CLAUDE\.md$/, /^\.claude\/hooks\//];

// The stop nudge ignores changes under these prefixes.
export const NUDGE_IGNORE = ["docs/memory/", ".claude/.state/"];
```

### `.claude/hooks/lib.mjs`

```js
// Shared helpers for the memory hooks. Works for Claude Code (default) and Kiro (`--kiro` flag).
import { readFileSync, existsSync } from "node:fs";
import path from "node:path";

export const KIRO = process.argv.includes("--kiro");

export async function readInput() {
  let raw = "";
  for await (const chunk of process.stdin) raw += chunk;
  try {
    return JSON.parse(raw || "{}");
  } catch {
    return {};
  }
}

export const projectDir = (input = {}) => process.env.CLAUDE_PROJECT_DIR || input.cwd || process.cwd();
export const readText = (file) => (file && existsSync(file) ? readFileSync(file, "utf8") : "");
export const rel = (root, file) => path.relative(root, file).split(path.sep).join("/");
export const emit = (obj) => process.stdout.write(JSON.stringify(obj));

// Tool input fields differ between tools and versions; try the known names.
export const toolCommand = (input) => {
  const t = input.tool_input ?? {};
  return String(t.command ?? t.cmd ?? JSON.stringify(t));
};
export const toolFile = (input) => {
  const t = input.tool_input ?? {};
  return t.file_path ?? t.path ?? input.file_path ?? input.path;
};
export const promptText = (input) => input.prompt ?? process.env.USER_PROMPT ?? "";

// Add text to the agent's context (prompt hooks).
export function addContext(event, text) {
  if (KIRO) process.stdout.write(text); // Kiro: stdout on exit 0 goes into context
  else emit({ hookSpecificOutput: { hookEventName: event, additionalContext: text } });
}

// Send a problem back to the agent: after a write (fix it) or at stop (keep going).
export function sendBack(event, reason) {
  if (KIRO && event !== "Stop") {
    process.stderr.write(reason); // Kiro: stderr on non-zero exit goes to the agent
    process.exit(2);
  }
  emit({ decision: "block", reason }); // Claude Code, and Kiro V3 Stop hooks
}

// PreToolUse: "deny" blocks the tool; "ask" asks the human (Kiro has no ask, so it blocks).
export function decideTool(decision, reason) {
  if (KIRO) {
    process.stderr.write(decision === "ask" ? `${reason} Ask the human to run it themselves.` : reason);
    process.exit(2);
  }
  emit({ hookSpecificOutput: { hookEventName: "PreToolUse", permissionDecision: decision, permissionDecisionReason: reason } });
}

// Skills used in this session, from the transcript (Skill tool calls and /slash commands).
export function loadedSkills(transcriptPath) {
  const skills = new Set();
  for (const line of readText(transcriptPath).split("\n")) {
    if (!line.includes("Skill") && !line.includes("command-name")) continue;
    try {
      const content = JSON.parse(line)?.message?.content;
      if (Array.isArray(content)) {
        for (const p of content) {
          if (p?.type === "tool_use" && p.name === "Skill" && p.input?.skill) {
            skills.add(String(p.input.skill).split(":").pop());
          }
        }
      }
    } catch {}
    for (const m of line.matchAll(/<command-name>\/?([\w:-]+)<\/command-name>/g)) {
      skills.add(m[1].split(":").pop());
    }
  }
  return skills;
}
```

### `.claude/hooks/session-start.mjs`

Plain stdout from a SessionStart hook is added to Claude's context.

```js
// SessionStart: inject core rules + project card + recent log. Re-fires after compaction.
import path from "node:path";
import { readInput, projectDir, readText } from "./lib.mjs";
import { MEMORY_DIR, LOG_TAIL, CORE_RULES } from "./config.mjs";

const input = await readInput();
const mem = path.join(projectDir(input), MEMORY_DIR);
const card = readText(path.join(mem, "project-card.md")).trim();
const log = readText(path.join(mem, "log.md")).split("\n").filter((l) => l.startsWith("- "));

process.stdout.write(`# Session memory (${input.source ?? "startup"})

## Core rules
${CORE_RULES.map((r) => `- ${r}`).join("\n")}

${card || "(no project card yet: create docs/memory/project-card.md)"}

## Last log entries
${log.slice(-LOG_TAIL).join("\n") || "(empty)"}
`);
```

### `.claude/hooks/route-prompt.mjs`

```js
// UserPromptSubmit: tell Claude which skill this request needs.
import { readInput, promptText, addContext } from "./lib.mjs";
import { ROUTES } from "./config.mjs";

const prompt = promptText(await readInput());
const hits = ROUTES.filter((r) => r.re.test(prompt)).map((r) => `\`${r.skill}\``);
if (hits.length) {
  addContext("UserPromptSubmit", `Router: load the ${hits.join(" and ")} skill before acting, unless already loaded this session.`);
}
```

### `.claude/hooks/guard-command.mjs`

```js
// PreToolUse (shell): deny guarded commands until their skill is loaded; ask for risky ones.
import { mkdirSync, writeFileSync } from "node:fs";
import path from "node:path";
import { KIRO, readInput, projectDir, readText, toolCommand, decideTool, loadedSkills } from "./lib.mjs";
import { GUARDS } from "./config.mjs";

const input = await readInput();
const cmd = toolCommand(input);
const guard = GUARDS.find((g) => g.re.test(cmd));
if (!guard) process.exit(0);

if (guard.ask?.test(cmd)) {
  decideTool("ask", "Risky command (prod or destroy): the human must confirm.");
} else if (!KIRO && !loadedSkills(input.transcript_path).has(guard.skill)) {
  decideTool("deny", `Load the \`${guard.skill}\` skill and follow it before running this command.`);
} else if (KIRO) {
  // Kiro gives hooks no transcript, so we can't see which skills are loaded. Instead, block the
  // first attempt per session and hand the agent the skill text; the retry is allowed.
  const root = projectDir(input);
  const marker = path.join(root, ".claude", ".state", `guard-${guard.skill}-${input.session_id ?? new Date().toISOString().slice(0, 10)}`);
  if (!readText(marker)) {
    mkdirSync(path.dirname(marker), { recursive: true });
    writeFileSync(marker, "shown");
    const skill = readText(path.join(root, ".kiro", "skills", guard.skill, "SKILL.md")) ||
      readText(path.join(root, ".claude", "skills", guard.skill, "SKILL.md"));
    decideTool("deny", `Before running this, follow the \`${guard.skill}\` rules below, then retry.\n\n${skill}`);
  }
}
```

### `.claude/hooks/check-written.mjs`

```js
// PostToolUse (Write|Edit|MultiEdit): flag known mistakes in the written file so Claude fixes them.
import path from "node:path";
import { readInput, projectDir, readText, rel, toolFile, sendBack } from "./lib.mjs";
import { CHECKS, CHECK_EXEMPT } from "./config.mjs";

const input = await readInput();
const root = projectDir(input);
const file = toolFile(input);
if (!file || !CHECKS.length) process.exit(0);

const r = rel(root, path.resolve(root, file));
// Skip files outside the project (on Windows another drive gives an absolute path) and exempt files.
if (r.startsWith("..") || path.isAbsolute(r) || CHECK_EXEMPT.some((re) => re.test(r))) process.exit(0);

const lines = readText(path.resolve(root, file)).split("\n");
const problems = [];
for (const c of CHECKS) {
  if (c.only && !c.only.test(r)) continue;
  lines.forEach((line, i) => {
    const m = line.match(c.re);
    if (m) problems.push(`${r}:${i + 1}: "${m[0]}": ${c.why}`);
  });
}
if (problems.length) {
  sendBack("PostToolUse", `Known-mistake check failed. Fix these now:\n${problems.join("\n")}`);
}
```

### `.claude/hooks/stop-nudge.mjs`

```js
// Stop: if project files changed but the log didn't, nudge once per session per set of changes.
import { execSync } from "node:child_process";
import { mkdirSync, writeFileSync } from "node:fs";
import path from "node:path";
import { createHash } from "node:crypto";
import { readInput, projectDir, readText, sendBack } from "./lib.mjs";
import { MEMORY_DIR, NUDGE_IGNORE } from "./config.mjs";

const input = await readInput();
if (input.stop_hook_active) process.exit(0); // already nudged in this stop cycle

const root = projectDir(input);
let status;
try {
  // -uall lists untracked files one by one, so new folders don't hide what's inside.
  status = execSync("git status --porcelain -uall", { cwd: root, encoding: "utf8" });
} catch {
  process.exit(0); // not a git repo
}

const changed = status.split("\n").map((l) => l.slice(3).trim().replace(/^"|"$/g, "")).filter(Boolean);
if (changed.includes(`${MEMORY_DIR}/log.md`)) process.exit(0);
const work = changed.filter((f) => !NUDGE_IGNORE.some((p) => f.startsWith(p)));
if (!work.length) process.exit(0);

const stateDir = path.join(root, ".claude", ".state");
const key = createHash("sha1").update(work.sort().join("\n")).digest("hex").slice(0, 12);
const marker = path.join(stateDir, `nudged-${input.session_id ?? "x"}`);
if (readText(marker).trim() === key) process.exit(0);
mkdirSync(stateDir, { recursive: true });
writeFileSync(marker, key);

sendBack(
  "Stop",
  `Files changed (${work.slice(0, 5).join(", ")}${work.length > 5 ? ", …" : ""}) but ${MEMORY_DIR}/log.md wasn't updated. ` +
    "Append dated log lines and update Last status / Open tasks in the project card. " +
    "If this was exploratory or the changes aren't yours, say so in one line and stop.",
);
```

### `.claude/settings.json`

```json
{
  "hooks": {
    "SessionStart": [
      { "matcher": "startup|resume|clear|compact",
        "hooks": [{ "type": "command", "command": "node \"$CLAUDE_PROJECT_DIR/.claude/hooks/session-start.mjs\"" }] }
    ],
    "UserPromptSubmit": [
      { "hooks": [{ "type": "command", "command": "node \"$CLAUDE_PROJECT_DIR/.claude/hooks/route-prompt.mjs\"" }] }
    ],
    "PreToolUse": [
      { "matcher": "Bash|PowerShell",
        "hooks": [{ "type": "command", "command": "node \"$CLAUDE_PROJECT_DIR/.claude/hooks/guard-command.mjs\"" }] }
    ],
    "PostToolUse": [
      { "matcher": "Write|Edit|MultiEdit",
        "hooks": [{ "type": "command", "command": "node \"$CLAUDE_PROJECT_DIR/.claude/hooks/check-written.mjs\"" }] }
    ],
    "Stop": [
      { "hooks": [{ "type": "command", "command": "node \"$CLAUDE_PROJECT_DIR/.claude/hooks/stop-nudge.mjs\"" }] }
    ]
  }
}
```

If the project already has a `settings.json`, merge the `hooks` key in rather than replacing the file.

---

## 5. Check that it works

Run these from the project root, in Git Bash on Windows or any shell on macOS/Linux:

```bash
echo '{"source":"startup"}' | node .claude/hooks/session-start.mjs | head -20
```

```bash
echo '{"prompt":"please deploy to staging"}' | node .claude/hooks/route-prompt.mjs
```

```bash
echo '{"tool_input":{"command":"terraform apply"}}' | node .claude/hooks/guard-command.mjs
```

Expected results: the card prints; the router suggests `deploy`; the guard returns `"permissionDecision":"deny"`.

Then start a new Claude Code session in the repo and ask "what's the status of this project?". It should answer from the card without opening any files. `/hooks` in a terminal session lists the wired hooks.

Notes:
- On Windows, Claude Code runs hook commands through Git Bash, so `$CLAUDE_PROJECT_DIR` works.
- A hook that changes mid-session applies to the next tool call. Session-start and new skills take effect in the next session.
- A guard matches the command text, so it also fires on an `echo` that only mentions the command. That's on purpose.

---

## 6. Daily use

- **Write back at the end of every task.** Dated log lines, updated Last status and Open tasks, then commit with the code. The stop hook reminds Claude if it forgets.
- **"From now on never do X"** becomes a dated "User decided" line in the right skill, the same minute.
- **Prune by hand:** only the "Last status" block in the card, a couple of minutes every few weeks. The log can grow forever because it's opened only on demand.
- **Add a check** to `CHECKS` the second time Claude makes the same mistake.
- **Add a guard** for any command that should never run without its checklist: deploys, migrations, data deletes, publishing.

### Going further: the three-repo version

When you have many projects, move the shared parts out of each repo:

1. **How I work:** a separate git repo packaged as a Claude Code plugin, with one skill per job across all projects. Install it in each project.
2. **What happened:** a repo with one folder per project (card, log, decisions). The session-start hook pulls it with `git pull` and injects the folder that matches the current project.
3. **Sources:** raw notes from books, talks and newsletters. Never give these to Claude directly; distil the useful parts into rules in repo 1 and keep the raw text as a receipt.

Start with the single-repo version above. That's about 90% of the benefit.

---

## 7. One-shot setup prompt

Paste this into Claude Code in a new repo:

```text
Set up git-backed memory for this repo following docs/CLAUDE-MEMORY-SETUP.md from my call_coach project
(I'll paste it if you can't read it). Specifically:
1. Read the codebase and write docs/memory/project-card.md (only facts you can't guess from code — ask me
   for environments, URLs and "never touch" rules), log.md (backfill from git log), decisions.md.
2. Create .claude/hooks/* and .claude/settings.json from the guide; fill config.mjs ROUTES/GUARDS/CHECKS
   for this project's real deploy commands and known pitfalls.
3. Create one skill per job I actually do here (ask me which), each rule with MUST/NEVER/PREFER and a why.
4. Add the Memory section to CLAUDE.md and the .gitignore lines.
5. Test each hook with sample stdin and show me the output. Don't commit until I say so.
```

---

## 8. Using it with Kiro

Kiro (IDE 1.0+, CLI V3+) has the same building blocks, so one repo can serve both tools. The memory files are shared, the skills are copied, and the same hook scripts run with a `--kiro` flag.

| Piece | Claude Code | Kiro |
|---|---|---|
| Memory files | `docs/memory/` | Same files |
| Always-on instructions | `CLAUDE.md` | `AGENTS.md` at the repo root (always included), or `.kiro/steering/*.md` with `inclusion: always` |
| Job skills | `.claude/skills/<job>/SKILL.md` | `.kiro/skills/<job>/SKILL.md`, same Agent Skills format (`name` must match the folder) |
| Hook wiring | `.claude/settings.json` | `.kiro/hooks/memory.json` |
| Hook scripts | `.claude/hooks/*.mjs` | The same scripts, run with `--kiro` |

### Shared instructions

Put the Memory section (section 2) and anything else both tools need in `AGENTS.md`. Then make `CLAUDE.md` import it, so there's one source:

```markdown
@AGENTS.md

<Claude-only notes, if any>
```

### Skills

Copy each skill folder: `.claude/skills/deploy/` → `.kiro/skills/deploy/`. Kiro loads a skill when the request matches its `description`, or when you type `/deploy`. Keep the two copies in step, or keep one folder and have a small script copy it.

### `.kiro/hooks/memory.json`

Kiro runs hook commands from the project root.

```json
{
  "version": "v1",
  "hooks": [
    { "name": "memory-card", "trigger": "SessionStart",
      "action": { "type": "command", "command": "node .claude/hooks/session-start.mjs --kiro" } },
    { "name": "memory-card-after-compaction", "trigger": "PostCompact",
      "action": { "type": "command", "command": "node .claude/hooks/session-start.mjs --kiro" } },
    { "name": "skill-router", "trigger": "UserPromptSubmit",
      "action": { "type": "command", "command": "node .claude/hooks/route-prompt.mjs --kiro" } },
    { "name": "command-guard", "trigger": "PreToolUse", "matcher": "shell",
      "action": { "type": "command", "command": "node .claude/hooks/guard-command.mjs --kiro" } },
    { "name": "known-mistakes", "trigger": "PostToolUse", "matcher": "write",
      "action": { "type": "command", "command": "node .claude/hooks/check-written.mjs --kiro" } },
    { "name": "log-nudge", "trigger": "Stop",
      "action": { "type": "command", "command": "node .claude/hooks/stop-nudge.mjs --kiro" } }
  ]
}
```

### What `--kiro` changes

| Hook | Claude Code | Kiro |
|---|---|---|
| session-start | stdout goes into context | Same |
| route-prompt | Reads `prompt` from stdin; returns `additionalContext` JSON | Reads the `USER_PROMPT` env var; prints plain text (stdout on exit 0 goes into context). Kiro also matches skills by description, so this hook is optional there. |
| guard-command | Denies until the skill appears in the transcript; asks the human for prod/destroy | Kiro gives hooks no transcript, so the **first** guarded command in a session is blocked and the skill text is sent to the agent; the retry is allowed. Prod/destroy is always blocked (Kiro has no "ask"), so you run those yourself. |
| check-written | Returns `{"decision":"block"}` | Exits 2; stderr goes to the agent |
| stop-nudge | Returns `{"decision":"block"}` | Same JSON (Kiro V3 Stop hooks continue the session on it) |

The scripts try the known names for tool inputs (`command`/`cmd`, `file_path`/`path`), because Kiro's built-in tool fields aren't fully documented.

### First run in Kiro: check it

1. Start a session and ask "what's the status of this project?". It should answer from the card.
2. Ask it to run a guarded command (for example a dev deploy). The first attempt should be blocked with the skill text.
3. Have it write a file that breaks one of your `CHECKS`. It should fix it.
4. If a hook does nothing, check what Kiro sends: temporarily point that hook's command at `node -e "process.stdin.pipe(process.stdout)"` (or write stdin to a file), then add any missing field name to `toolCommand` / `toolFile` in `lib.mjs`.

Kiro's docs: [hooks](https://kiro.dev/docs/hooks/), [triggers](https://kiro.dev/docs/hooks/types/), [actions](https://kiro.dev/docs/hooks/actions/), [steering](https://kiro.dev/docs/steering/), [skills](https://kiro.dev/docs/skills/).
