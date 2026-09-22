# AI-DLC Manual — opencode and GitHub Copilot

A step-by-step walkthrough for running AI-DLC on the **opencode** and **GitHub
Copilot** harnesses.

Both harnesses share the same engine. The engine, state machine, audit log,
graph, swarm referee, and learnings gate are byte-identical across every
distribution — only the shell differs. The differences below are about *where
files land*, *how you trust/authorize*, and *how hooks fire*.

---

## Step 0 — Prerequisites

| | opencode | GitHub Copilot |
| --- | --- | --- |
| Host version | `opencode >= 1.17` (`opencode --version`) | Copilot CLI `>= 1.0.74` and/or VS Code `>= 1.130` |
| Auth / provider | Your **global** opencode config supplies the model. The project pins none. | GitHub sign-in **or** BYOK env vars (`COPILOT_PROVIDER_BASE_URL`, `COPILOT_PROVIDER_TYPE`, `COPILOT_MODEL`, …). In VS Code, use the model picker or a Custom Endpoint provider. |
| Extra required step | none | **Folder trust** — the project's absolute path must be in `trustedFolders` in `~/.copilot/config.json` |
| Bun / Node.js | Not required for native installs | Not required for native installs |

Recommended model: a strong reasoning model (Claude Opus 4.8 class). Weaker
models tend to stall at approval gates.

---

## Step 1 — Install the `aidlc` command (once per machine)

macOS / Linux / WSL:

```bash
tmp="$(mktemp -d)"
curl -fsSL https://github.com/telgue/aidlc/releases/latest/download/install.sh -o "$tmp/install.sh"
sh "$tmp/install.sh"
rm -rf "$tmp"
```

Windows PowerShell:

```powershell
irm https://github.com/telgue/aidlc/releases/latest/download/install.ps1 | iex
```

This installs the native `aidlc` binary **plus every harness runtime**. If a new
shell cannot find `aidlc`, apply the PATH instruction the installer printed.

Verify:

```bash
aidlc --version
# aidlc 1.0.0 (runtime 1.0.0)
```

A successful run prints:

```text
Downloaded version.json
Downloaded checksums.txt
Downloaded aidlc-release.intoto.jsonl
Downloaded aidlc-<os>-<arch>
Downloaded aidlc-runtime-<X.Y.Z>.tar.gz
PASS installed AI-DLC <X.Y.Z> with all harness runtimes
Next: aidlc config
```

The binary lands in `~/.local/bin/aidlc`. If a new shell cannot find it, apply
the PATH line the installer printed (commonly `. "$HOME/.local/bin/env"` in
`~/.zshrc`). Confirm with:

```bash
zsh -lic 'command -v aidlc'
```

> `WARN GitHub CLI attestation verification is unavailable; continuing with
> SHA-256 release checksums.` is benign. The installer verifies every asset
> against the published checksums regardless; installing the GitHub CLI (`gh`)
> additionally enables provenance attestation verification.

### If the install fails with `ERROR download failed` (exit code 3)

See [Troubleshooting the installer](#troubleshooting-the-installer) below — on
macOS this is almost always a shadowed `curl` with a stale CA store, not a
network or release problem.

---

## Step 2 — Configure your project

> **Do not run `aidlc config` inside the `aidlc-workflows` repository itself.**
> That repo is the hand-authored *source*, not a consumer. Configuring it would
> write `.aidlc/`, the harness shell, and an `aidlc/` workspace into the
> framework tree. Target a real project directory instead.

```bash
cd /path/to/your-project

# preview first (always safe)
aidlc config --dry-run

# opencode
aidlc config --harness opencode
# …or GitHub Copilot
aidlc config --harness copilot

aidlc doctor
```

A bare `aidlc config` starts interactive setup when a terminal is available. It
detects installed harnesses, provider state, runtime needs, and trust actions
before writing anything.

The dry run reports a plan; a clean first install shows only creates and no
conflicts:

```text
config plan for /path/to/your-project: create=314 update=0 merge=0 preserve=0 remove=0 conflict=0
```

The apply prints the next action, then a seven-section setup check:

```text
configured /path/to/your-project for opencode 1.0.0; next: run `opencode`, then `/aidlc --doctor`

  Setup check - 2 of 7 sections need you.
    [ok]     Harnesses   opencode recorded
    [ok]     Models      shipped defaults
    [needs]  Runtime     ...
    [ok]     Flags       defaults
    [ok]     Project     plugins: all installed, MCP: none, completions: none
    [needs]  Providers   no recorded answers; provider access unverified
    [ok]     Trust       no unmet host trust

  Fix the 2 sections that need you now? [Y/n]:
```

> **`--yes` does not suppress this prompt.** It applies to the config
> transaction, not to the post-config setup check. In scripts and CI, answer it
> on stdin (`printf 'n\n' | aidlc config --harness opencode --yes`) or run the
> `aidlc config <section>` subcommands directly. The project is already
> configured at this point — declining the prompt does not roll anything back.

### What gets written — opencode

Two dot-directories, on purpose:

| Path | Contents |
| --- | --- |
| `.aidlc/` | The engine tree: tools, hooks, skills, agents, knowledge, scopes, sensors, `aidlc-common`. opencode never scans it. |
| `.opencode/` | Only natively-consumed surfaces: `agents/*.md` (14 personas, `mode: subagent`), `command/aidlc.md` (the `/aidlc` command), `plugin/aidlc-opencode-adapter.ts` (hook adapter, auto-discovered). |
| `opencode.json` | Three load-bearing blocks: `skills.paths: [".aidlc/skills"]`, `instructions` (the method-tree include), and permission rules for AIDLC bash entrypoints. |
| `aidlc/` | The workspace (spaces / intents / artifacts) — a **sibling** of `.aidlc/`, not inside it. |
| `AGENTS.md` | Onboarding plus the `.gitignore` block you must apply. |

> The split exists because opencode auto-imports every `*.ts` under
> `.opencode/tools/` and `.opencode/tool/` as a custom tool definition, and
> importing a CLI-style engine script (top-level dispatch, `process.exit`)
> crashes the session.

### What gets written — GitHub Copilot

Engine directory plus a **merged** `.github` shell:

| Path | Contents |
| --- | --- |
| `.aidlc/` | Engine tree, including `hooks/aidlc-copilot-adapter.ts`. |
| `.github/hooks/aidlc.json` | Hook wiring (matcher-free by design — VS Code parses but ignores matchers). |
| `.github/agents/aidlc-*-agent.md` | The 14 personas as Copilot custom agents. |
| `.github/skills/aidlc*/` | Orchestrator, per-stage runners, scope runners, session skills. |
| `AGENTS.md` | Onboarding. Keep the `@`-import block — that is the method include. |
| `aidlc/` | The workspace. |

> Everything under `.github/` is `aidlc`-prefixed, so it **merges** with your
> existing workflows and templates — nothing of yours is overwritten. **One
> install serves both Copilot CLI and VS Code agent mode.**

---

## Step 2b — "Manual provider setup" prompt

During `aidlc config` you will be asked about a model provider. **This is by
design, not an error.** `aidlc config` has no network access and brokers no
credentials — it *records* your decision; you do the actual setup in the
harness. For Copilot and Cursor, provider setup is **instruct-only** by type.

You will hit it in one of two shapes.

### Shape A — the interactive wizard

```text
  Step 2 of 6 - Model provider
  No AWS credentials were detected.
    1. amazon-bedrock
    2. other            record your own provider setup
  Provider:
```

**Choose `2. other`** if you are using GitHub sign-in (the normal Copilot case),
Azure, Google Cloud, or any non-AWS provider. Choosing it prints:

```text
  Using other provider setup.
```

Choose `1. amazon-bedrock` **only** for Copilot BYOK pointed at a Bedrock
endpoint — it then also asks for region, profile, and (on opencode) whether to
make Bedrock the default.

> The wizard's default is derived from **offline AWS-credential detection**. If
> stray AWS credentials exist on the machine it pre-selects `1`, so explicitly
> type `2` rather than pressing Enter.

### Shape B — the post-apply outstanding action

If you picked `other`, config finishes and then lists:

```text
  One thing needs you - it can't be done automatically:

  providers / provider-manual-setup:
    Choose and configure a model provider, then acknowledge the completed setup.
```

That is an instruction to come back **after** configuring the provider in the
harness.

### Clearing it

1. **Configure the provider in the harness first.**

   | Harness | What "configure the provider" means |
   | --- | --- |
   | Copilot CLI | Sign in to GitHub, **or** set BYOK env vars: `COPILOT_PROVIDER_BASE_URL`, `COPILOT_PROVIDER_TYPE`, `COPILOT_MODEL`, `COPILOT_PROVIDER_WIRE_MODEL` + bearer token. Run `copilot help providers` for the current set |
   | Copilot in VS Code | Use the model picker or a Custom Endpoint provider — **not** env vars |
   | opencode | Set the provider in your **global** opencode config; confirm with `opencode auth list` |

2. **Then acknowledge it:**

   ```bash
   cd /path/to/your-project
   aidlc config providers --provider other --acknowledge --yes
   aidlc config providers --check     # exits 0 when clean
   ```

`--acknowledge` is mandatory here; see
[Recording the choice](#recording-the-choice---provider-other) for the full
mechanics, the pending-action table, and where the answer is stored.

### Two Copilot-specific gotchas

- **Picking `amazon-bedrock` on Copilot adds a second pending action**,
  `copilot-byok-configuration`, which also requires `--acknowledge`. Unless you
  genuinely need Bedrock BYOK, `other` is the simpler path.
- **Folder trust is a separate, mandatory step.** Provider setup does not help
  if hooks are dead. See Step 3 — untrusted means every hook silently no-ops.

### Skipping the prompts entirely

Non-interactive, for scripts and CI:

```bash
printf 'n\n' | aidlc config --harness copilot --yes
aidlc config providers --provider other --acknowledge --yes
```

---

## Step 3 — Establish host trust

**Do this before the first workflow.** Trust is what lets hooks fire. It is the
single most consequential step in this manual, because the failure mode is
silent.

### opencode

Just start `opencode` in the project directory:

```bash
cd /path/to/your-project
opencode
```

opencode has **no folder-trust model**. Its equivalent is the `permission` block
already written into `opencode.json` by `aidlc config`:

```json
"permission": {
  "edit": { "*": "allow", ".aidlc/tools/**": "ask", ".aidlc/hooks/**": "ask" },
  "bash": { "*": "ask", "aidlc engine *": "allow" }
}
```

`aidlc engine *` is pre-allowed so the engine runs without prompting; everything
else still asks. Edits to engine code under `.aidlc/tools/` and `.aidlc/hooks/`
deliberately prompt. If you merged this into your own config, keep the block —
without it every engine call turns into an approval prompt.

### GitHub Copilot — folder trust (mandatory for the CLI)

Repo hooks run **only** when the project's absolute path is listed in
`trustedFolders` in `~/.copilot/config.json`. Follow these sub-steps.

#### 3.1 — Open the config file

```bash
open -e ~/.copilot/config.json      # macOS; or use any editor
code ~/.copilot/config.json         # VS Code
```

If the file does not exist yet, create it with just the `trustedFolders` key
from 3.2.

#### 3.2 — Add your project's absolute path

Add a `trustedFolders` array, or append to it if one already exists. **Keep
every other key in the file intact:**

```jsonc
// User settings belong in settings.json.
// This file is managed automatically.
{
  "firstLaunchAt": "2026-09-15T13:15:49.990Z",
  "loggedInUsers": [{ "host": "https://github.com", "login": "your-login" }],

  "trustedFolders": [
    "/Users/you/code/your-project"
  ]
}
```

Rules for the path:

- **Absolute only** — `~` is **not** expanded.
- Trailing slashes are trimmed, and both sides are `realpath`-resolved, so a
  symlinked project matches either way.
- Add one entry per project; the array holds many.
- `COPILOT_HOME` overrides the `~/.copilot` location if you have set it.

Print the exact path to paste:

```bash
cd /path/to/your-project && pwd
```

#### 3.3 — Verify

```bash
cd /path/to/your-project
aidlc doctor
```

The check must flip from `fail` to `ok`. Repeat 3.2 if it has not.

#### Alternative: let the CLI record it

If you use the Copilot CLI, one interactive run writes the entry for you:

```bash
cd /path/to/your-project
copilot          # accept the interactive trust prompt on first run
```

Headless `copilot -p` runs additionally need:

```bash
export GITHUB_COPILOT_PROMPT_MODE_REPO_HOOKS=1
```

### Verifying trust

`aidlc doctor` is the **only** surface that reports this — Copilot itself says
nothing when a folder is untrusted.

```bash
cd /path/to/your-project
aidlc doctor
```

An untrusted project fails, by design:

```text
Project (.aidlc, GitHub Copilot)
  fail  project folder in ~/.copilot/config.json trustedFolders (CLI hooks silently no-op without it)
        fix: add "/Users/you/code/your-project" to trustedFolders in ~/.copilot/config.json
             (or accept the CLI's interactive trust prompt)
```

Fix it with either route above, then re-run `aidlc doctor` until the check
passes.

The three outcomes this check can produce:

| `~/.copilot/config.json` | Result | Meaning |
| --- | --- | --- |
| Absent | **pass** (advisory) | No CLI config exists yet |
| Present, project listed | **pass** | CLI hooks will fire |
| Present, project **not** listed | **fail** | See the VS Code note below |
| Present but unreadable/malformed | **fail** | Trust cannot be verified. Repair it as valid JSONC, then re-run doctor |

> The config is parsed as **JSONC**, so comments and trailing commas are
> tolerated — the CLI itself writes that dialect.

#### If you use VS Code agent mode only

**VS Code also creates `~/.copilot/config.json`.** A file written by the IDE
looks like this — note it has no `trustedFolders` key at all:

```jsonc
// User settings belong in settings.json.
// This file is managed automatically.
{
  "firstLaunchAt": "...",
  "askedSetupTerminals": ["vscode"],
  "loggedInUsers": [{ "host": "https://github.com", "login": "..." }]
}
```

Because the file **exists** but does not list your project, the doctor check
reports `fail` even though you never installed the CLI. For agent mode this is
a **false alarm**: `trustedFolders` gates *CLI* repo hooks, while VS Code uses
the IDE's own workspace-trust model.

To clear the check, add the key by hand and keep the rest of the file intact:

```jsonc
"trustedFolders": [
  "/Users/you/code/your-project"
]
```

> The file's own header says it is "managed automatically", so the IDE may
> rewrite it. Re-add the entry if the check regresses.

### Why this matters so much

**Untrusted = every hook silently no-ops, with no warning anywhere in Copilot.**
The workflow will *appear* to run. What you actually lose is the entire
enforcement layer:

- audit-log emission
- sensor dispatch
- reviewer read-scope bounds
- the state-transition and plan-approval guards
- review-receipt write-freeze

You get AI-DLC's prose without its guarantees. Treat a failing trust check as a
blocker, not a warning.

**Git integration.** On the **native channel** `aidlc config` writes the managed
`.gitignore` block for you, delimited by `# BEGIN AI-DLC:gitignore` /
`# END AI-DLC:gitignore` — no manual step is needed. Only the **manual-copy
channel** requires applying the entries from the shipped `AGENTS.md` § "Git
Integration" yourself.

The split it encodes: per-clone audit shards, state, method, and artifacts are
**committed** (they are the shared work); per-user session cursors and
machine-local runtime are **ignored**. Do not add ignore rules for
`aidlc/spaces/*/intents/*/audit/*.md`.

---

## Step 4 — Verify inside the harness

```text
/aidlc --doctor
```

Or from a shell:

```bash
# opencode
opencode run --command aidlc -- "--status"

# Copilot CLI
copilot -p "/aidlc --doctor" -s --allow-all-tools
```

Harness-specific doctor checks:

- **opencode:** adapter plugin present at `.opencode/plugin/`, a project-root
  `opencode.json` or `opencode.jsonc` present, `.opencode/command/aidlc.md`
  present.
- **Copilot:** host version floors, folder trust, hook registration.

A healthy first run ends with `0 problems` — warnings are advisory:

```text
0 problems, 4 warnings.
Warnings are advisory - if everything works, ignore them.
```

### The one warning worth acting on: hook PATH

```text
warn  Runtime hook PATH: aidlc is interactive-only at ~/.local/bin/aidlc
      fix: Add ~/.local/bin to the login-independent environment used by the
           harness, not only an interactive shell rc file.
```

Inspect it in detail with:

```bash
aidlc config runtime --show
aidlc config runtime --show --json    # lists every file carrying hook commands
```

Typical output on macOS:

```text
Runtime configuration for copilot
  Hook baseline PATH: /usr/bin:/bin:/usr/sbin:/sbin:/usr/local/bin:...
  bun: not-required
  aidlc: interactive-only -> interactive only at ~/.local/bin/aidlc
  Harness CLI: missing
  Files carrying hook commands:
    ... 52 agent/hook files
```

Read it field by field.

#### `aidlc: interactive-only` — the one that matters

Hooks are spawned by the harness in a **non-interactive** environment. A `PATH`
export that lives in `~/.zshrc` is invisible to them, so hooks silently fail to
find `aidlc`.

**The probe does not read your shell rc files at all.** On macOS it builds the
baseline from exactly three sources:

| Source | Example |
| --- | --- |
| `getconf PATH` | `/usr/bin:/bin:/usr/sbin:/sbin` |
| `/etc/paths` | `/usr/local/bin`, `/System/Cryptexes/App/usr/bin`, … |
| `/etc/paths.d/*` (each file, sorted) | `/usr/local/munki`, `/pkg/env/global/bin`, … |

On Windows it reads the **Machine** and **User** `Path` environment variables
via PowerShell — again, not a shell profile.

So `~/.zshenv` **does not fix this**. Use one of these instead:

```bash
# Option A (recommended) — symlink into a directory already on the baseline
sudo ln -sf "$HOME/.local/bin/aidlc" /usr/local/bin/aidlc

# Option B — add your bin directory to the system path list.
#   Must be an ABSOLUTE path; `~` is not expanded in these files.
echo "$HOME/.local/bin" | sudo tee /etc/paths.d/50-aidlc
```

```powershell
# Windows — set the User Path, not a profile script
[Environment]::SetEnvironmentVariable(
  'Path',
  [Environment]::GetEnvironmentVariable('Path','User') + ';' + "$HOME\.local\bin",
  'User')
```

Both macOS options need `sudo` because `/usr/local/bin` and `/etc/paths.d` are
root-owned. Option B writes a user-specific absolute path into a system-wide
file, which is fine on a single-user machine but not on a shared one — prefer
Option A there. Restart the harness afterwards so it picks up the new
environment.

Then record it:

```bash
aidlc config runtime --check          # non-writing probe
aidlc config runtime --record-paths --yes
```

Note the ordering constraint: `--record-paths` **refuses while the probe is
dirty** —

```text
error: runtime paths cannot be recorded until the non-interactive probe is clean
```

so fix the environment first, then record.

#### `bun: not-required` — ignore

Correct and expected for a native install. Bun is only needed for the
source/development `dist/` projection.

#### `Harness CLI: missing` — install the harness

The harness binary itself was not found. `aidlc config` configures a project for
a harness whether or not that harness is installed, so this is normal if you
have not installed it yet:

```bash
command -v copilot   # or: command -v opencode
```

For Copilot this is only a problem if you intend to use the **CLI**. If you work
exclusively in **VS Code agent mode**, there is no `copilot` binary to find and
this stays `missing` — harmless. Install the CLI if you want both surfaces.

#### `Files carrying hook commands` — informational

The list of agent and hook files that embed an `aidlc engine …` command. It is
shown so you can see what the PATH problem would affect; there is nothing to do.
`--show --json` prints the full list instead of truncating.

### Other warnings you can usually ignore

| Warning | Meaning |
| --- | --- |
| `Workspace records: N uncommitted change(s) under aidlc/` | Advisory — the shared records travel by git. `git add aidlc/ && git commit` |
| `Plugins: 1 need attention - host:inventory-unavailable` | The host inventory is only readable through the SessionStart adapter; it resolves once you start the harness |
| `Update: ... transport failure` | The update check could not reach the release asset host. `aidlc` uses its own embedded HTTP client here, **not** `curl`, so this is distinct from the installer's CA problem. Exit code is still 0 |

---

## Step 5 — Start a workflow

Both harnesses invoke with `/aidlc`:

```text
/aidlc Build a REST API for inventory management
```

Pick a profile explicitly if you want:

```text
/aidlc express
/aidlc feature Add customer notifications
/aidlc bugfix Fix the login timeout
```

A bare `/aidlc` on an existing workflow gives the **Resume / Redo / Jump / Start
Fresh** menu.

- On **opencode** there is no channel for the session-start hook's injected
  context, so a bare `/aidlc` performs one read-only status probe first.
  `/aidlc --resume` skips both the probe and the menu and continues directly.

---

## Step 6 — Work the loop

AI-DLC drives phases → stages, adopting or delegating to personas, and **stops at
human approval gates**.

**Answering questions.** On both harnesses, questions render as **numbered prose
options**, not native pickers.

- *opencode:* no structured-question widget exists.
- *Copilot:* native pickers exist, but their answers return as tool results and do
  not fire the trusted `UserPromptSubmit` event required by the human-presence
  guard. While the session-selected workflow has valid `Status: Running` state,
  the matcher-free PreToolUse guard denies those picker calls and directs the
  model to render numbered prose. Without a running workflow, native pickers are
  left untouched.

In both cases the **questions FILE with `[Answer]:` tags is the source of truth** —
you can edit it directly.

Useful in-session commands:

```text
/aidlc --status      # where am I
/aidlc --resume      # continue directly
/aidlc --doctor      # health check
/aidlc space <name>  # switch method/space (re-points the include in place)
```

Artifacts accumulate under:

```text
aidlc/spaces/<space>/intents/<YYMMDD>-<label>/
```

That record holds workflow state, audit shards, questions, decisions, and stage
artifacts. Team knowledge and learned rules live at the **space** level so later
intents can reuse them.

---

## Step 7 — Know your harness's enforcement strength

| Behavior | opencode | GitHub Copilot |
| --- | --- | --- |
| Hook mechanism | `.opencode/plugin/aidlc-opencode-adapter.ts` maps opencode plugin moments onto the core hook bodies in `.aidlc/hooks/` | `.github/hooks/aidlc.json` → `.aidlc/hooks/aidlc-copilot-adapter.ts` |
| Guard enforcement | Reviewer read-scope and the AIDLC bash boundary enforced before tool execution | Core block converts to `permissionDecision: deny` — guards actually **refuse** the tool call |
| Forwarding loop | **Advisory** — the Stop seam is `session.idle` (reactive). A `block` re-engages the loop by injecting a sentinel-marked nudge prompt | **Blocking** Stop hook |
| Session end | No session-end moment; `SESSION_ENDED` events are not emitted. Pre-compaction validation **does** fire (`experimental.session.compacting`) | No SessionEnd documented; the prior session is reconciled at the next SessionStart with inferred provenance |
| Statusline | None — use `/aidlc --status` and the progress lines at gates | None — same |
| Delegation | Personas are native subagents; their permission map denies `task`, so no nested delegation | Explicit built-in `tools:` allowlist that omits the `agent` tool, so no nested delegation |
| Construction swarm | `task`-tool fan-out only; `AIDLC_USE_SWARM=1` is a loud no-op | Subagent fan-out only; `AIDLC_USE_SWARM=1` is a loud no-op |
| Model pinning | Tiered personas pin `amazon-bedrock/global.anthropic.claude-sonnet-4-6`; override per agent in `opencode.json` | **No `model:` pin** — agents inherit the session model (the CLI and IDE disagree on model-value syntax) |
| MCP | None ships; configure your own under `mcp:` in `opencode.json` | None ships. CLI reads `~/.copilot/mcp-config.json`, VS Code reads `.vscode/mcp.json`. The conductor can use them; delegated worker personas cannot |
| Space switches | `/aidlc space <name>` updates the method glob in `opencode.json` or `opencode.jsonc` without stripping comments or trailing commas | The method include rides `AGENTS.md` `@`-imports; `/aidlc space <name>` re-points the block in place, including the `.github/agents/` persona twins |

> **Copilot VS Code caveat:** the deny/block channels are documented and the
> adapter normalizes names such as `runTerminalCommand`, `createFile`,
> `editFiles`, and `readFile` — but the IDE side has not been verified live.
> Treat IDE enforcement as best-effort; the CLI path is live-verified.

---

## Step 8 — Update and refresh

```bash
aidlc update                 # machine runtime only; configured projects untouched
cd /path/to/your-project
aidlc doctor                 # reports project-vs-engine version skew
aidlc config --dry-run       # preview the refresh
aidlc config                 # apply
```

Rules:

- Config **refuses to refresh while a workflow is active** — complete or park the
  workflow first.
- Project-owned content and managed root blocks are preserved; local edits to
  framework files are reported as **conflicts**, not overwritten.
- **opencode:** `opencode.json` is a whole-file integration, so a local edit is
  preserved as a conflict. If you merged it into your own config, keep all three
  blocks (`skills.paths`, `instructions`, permissions).
- Using plugins? Run `/aidlc plugin sync` after an engine refresh.
- Machine-runtime upgrade and rollback are safe mid-workflow because they do not
  touch the project.

---

## Troubleshooting the installer

### `ERROR download failed` / `Run: check the release URL and proxy` (exit 3)

The installer's `download()` helper surfaces any `curl`/`wget` non-zero exit as
`fail 3 unavailable "download failed"`. Exit code 3 therefore means *the
transfer failed*, not that the release is missing. Reproduce the underlying
error directly:

```bash
curl -sSL -o /dev/null -w '%{http_code}\n' \
  https://github.com/telgue/aidlc/releases/latest/download/version.json
```

#### Cause 1 — a shadowed `curl` with a stale CA store (most common on macOS)

Anaconda, Homebrew, and some corporate images ship their own `curl` linked
against OpenSSL with a bundled CA store. GitHub 302-redirects release assets to
`release-assets.githubusercontent.com`, and those stores frequently cannot
verify that chain:

```text
curl: (60) SSL certificate problem: unable to get local issuer certificate
```

Diagnose by listing every `curl` on `PATH` and testing the system one:

```bash
which -a curl
curl --version | head -1
/usr/bin/curl -sSL -o /dev/null -w '%{http_code}\n' \
  https://github.com/telgue/aidlc/releases/latest/download/version.json
```

A worked example:

| curl | Result |
| --- | --- |
| `/opt/anaconda3/bin/curl` (OpenSSL 3.0.13) — first on `PATH` | `curl: (60) SSL certificate problem` |
| `/usr/bin/curl` (macOS system) | `200` |

**Immediate fix** — put the system `curl` first for the install only:

```bash
PATH="/usr/bin:$PATH" sh ./install.sh --yes
```

**Permanent fixes** (pick one; otherwise `aidlc update` and other HTTPS tooling
will hit the same wall):

```bash
# A. Point the shadowing curl at a real CA bundle — add to ~/.zshrc
export CURL_CA_BUNDLE="$(python -c 'import certifi; print(certifi.where())')"

# B. Prefer the system curl by putting /usr/bin ahead of the other toolchain
#    in your ~/.zshrc PATH assignment.
```

**Per-invocation alternative** — the installer accepts an explicit CA bundle:

```bash
sh ./install.sh --ca-bundle /path/to/ca-bundle.pem
```

#### Cause 2 — corporate proxy or TLS interception

Set `HTTPS_PROXY` / `HTTP_PROXY` as your network requires, and pass your
organization's root certificate with `--ca-bundle <file>`.

#### Cause 3 — no network at all

Use the offline path. Download the release directory on a connected machine,
then:

```bash
sh ./install.sh --from /path/to/release-directory --offline
```

`--offline` requires `--from`; the source directory must contain `version.json`,
`checksums.txt`, `aidlc-release.intoto.jsonl`, `install.sh`, the platform
binary, and `aidlc-runtime-X.Y.Z.tar.gz`.

### Other installer exit codes

| Exit | Meaning |
| --- | --- |
| `1` | A hard prerequisite is missing (`curl`/`wget`, or `sha256sum`/`shasum`) |
| `3` | Download unavailable — see above; also raised for `--offline` without `--from` |
| `4` | Verification failure: checksum mismatch, bad/oversized metadata, invalid source ref or digest, failed provenance, or a missing offline asset |

### Useful installer flags

```text
install.sh [--version <x.y.z|x.y.z-preview.YYYYMMDD.N>] [--from <dir>] [--offline]
           [--profile <startup-file>] [--json|--quiet] [--no-color] [--yes]
           [--ca-bundle <file>] [--release-base-url <url>]
```

On Windows the PowerShell installer mirrors these as `-From`, `-Offline`,
`-Yes`, `-Json`, `-Quiet`. Note that redirected input, `pwsh -NonInteractive`,
`--yes`, `--json`, and `--quiet` require the explicit flag; a fully interactive
run may omit it.

### `aidlc: command not found` after a successful install

The binary is at `~/.local/bin/aidlc`. Either start a new shell, or ensure your
startup file sources the installer's env shim:

```bash
grep -n 'local/bin' ~/.zshrc ~/.zprofile
zsh -lic 'command -v aidlc || echo "NOT ON PATH"'
```

---

## Appendix — Using a non-AWS provider (Azure, Google Cloud, others)

**Short answer: yes.** AI-DLC never calls a model API itself. It is
provider-independent — the *harness* owns provider selection, authentication,
and model routing. Amazon Bedrock is a *distribution default* on some harnesses
(Claude Code, Codex), not a methodology requirement, and **neither the opencode
nor the Copilot install pins a session model.**

So "can I run AI-DLC on Azure OpenAI or Google Vertex AI?" reduces to "does my
harness support that provider?" Configure it the harness's normal way; AI-DLC
inherits it.

### opencode

opencode has a first-class multi-provider `provider` block (it routes through
Models.dev / AI SDK provider packages). The shipped project `opencode.json`
deliberately contains **no** `provider` block and **no** session model — your
**global** opencode config supplies both. Point it at Azure or Vertex there and
AI-DLC just runs.

Two things to know:

1. **Two personas carry an explicit Bedrock pin** — verified on 1.0.0. Of the 14
   generated `.opencode/agents/aidlc-*-agent.md` files, **12 omit `model:`** and
   inherit the session model; only the two review-only agents pin it:

   ```text
   .opencode/agents/aidlc-architecture-reviewer-agent.md
   .opencode/agents/aidlc-product-lead-agent.md
   # model: amazon-bedrock/global.anthropic.claude-sonnet-4-6
   ```

   On a non-Bedrock provider **override just those two**, or review stages will
   try to resolve a Bedrock model id. Set `agent.<name>.model` in the project
   `opencode.json` using your provider's `provider/model` form, e.g.
   `azure/gpt-5.6-terra` or `google-vertex/gemini-3-pro`. Verify the current set
   yourself with:

   ```bash
   grep -l '^model:' .opencode/agents/*.md
   ```

2. **`/aidlc space <name>` preserves JSONC**, so you can keep comments in
   `opencode.json`/`opencode.jsonc` around your provider block.

> `aidlc config providers` offers to write
> `provider.amazon-bedrock.options.region/profile` into `opencode.json`. Decline
> it (or use `--opencode-default no`) on Azure/GCP — see *Recording the choice*
> below.

### GitHub Copilot

Copilot supports **BYOK (bring your own key)** and it works with *no GitHub
auth at all*. The relevant environment variables are:

```bash
export COPILOT_PROVIDER_BASE_URL="<your endpoint>"
export COPILOT_PROVIDER_TYPE="<wire protocol: anthropic | openai | ...>"
export COPILOT_MODEL="<catalog name>"
export COPILOT_PROVIDER_WIRE_MODEL="<the provider's own model id>"
# plus the provider's bearer token
```

Run `copilot help providers` for the authoritative, version-current set — that
command is the source of truth, not this manual.

Rough shape per cloud:

| Cloud | `COPILOT_PROVIDER_BASE_URL` | `COPILOT_PROVIDER_TYPE` |
| --- | --- | --- |
| Amazon Bedrock (documented example) | `https://bedrock-runtime.<region>.amazonaws.com/anthropic` | `anthropic` |
| Azure OpenAI | your Azure OpenAI resource endpoint / deployment path | `openai` |
| Google Vertex AI | the Vertex endpoint for the model family | match the family's wire protocol |

In **VS Code**, do not use env vars — use the model picker or add a **Custom
Endpoint** provider in settings.

Two Copilot-specific consequences worth planning around:

- **Personas carry no `model:` pin by design.** The CLI forwards a frontmatter
  model string verbatim to the BYOK provider, while an IDE display name would
  `400` there — so AI-DLC omits the field and every agent **inherits the session
  model**. That is exactly what makes Copilot the most provider-portable
  harness: there is nothing Bedrock-shaped to override.
- Auth is the provider's bearer token; GitHub sign-in is not required.

### Other harnesses, briefly

| Harness | Provider story |
| --- | --- |
| Claude Code | Ships Bedrock-default. To move off it, remove/replace the Bedrock env mappings in `.claude/settings.json` (and any `.claude/settings.local.json`), then complete that provider's Claude Code auth flow. Google Vertex is a supported Claude Code backend; Azure OpenAI is not, because the harness speaks the Anthropic API. |
| Codex CLI | Ships a Bedrock provider default under `[model_providers.amazon-bedrock]`. Codex supports additional `model_providers` entries, so an OpenAI-compatible Azure endpoint is configurable there. |
| Cursor / Kiro | Use their own provider configuration and sign-in; set it in the product, not in AI-DLC. |

### The hard constraint: model capability, not vendor

AI-DLC's gates, sensors, and multi-agent delegation need a **strong reasoning
model**. Vendor is irrelevant; capability is not. A weak or small model will
stall at approval gates, fail sensor checks, or loop. The current recommendation
is a Claude Opus 4.8-class model or equivalent.

### Recording the choice: `--provider other`

`aidlc config providers` models exactly **two** provider values —
`amazon-bedrock` and `other`. There is no `azure` or `gcp` value, and that is
deliberate: AI-DLC does not broker credentials for any cloud. `other` is the
supported, first-class escape hatch that records "this project is intentionally
not on Bedrock" **without writing provider bytes**.

Full flag surface:

```text
aidlc config providers [--show [--json]|--check|--reset]
                       [--provider <amazon-bedrock|other>]
                       [--region <region>] [--profile <profile>]
                       [--opencode-default <yes|no>]
                       [--acknowledge] [--mark-done <id>]
                       [--dry-run] [--yes]
```

#### The Azure / GCP command

Run this from the configured project root:

```bash
aidlc config providers --provider other --acknowledge --yes
```

`--acknowledge` is **mandatory** with `--provider other`. Without it the command
refuses (verified, exit code **2**):

```text
error: opencode provider setup is instruct-only; pass --acknowledge after completing the manual provider step
usage: aidlc config providers --help
```

That wording is the contract: *you* configure Azure or Vertex in the harness,
then acknowledge that you did. Preview first if you like — `--dry-run` is
supported.

Note what is **not** required on this path:

| Flag | Required for `other`? |
| --- | --- |
| `--region` | No — Bedrock-only (`--provider amazon-bedrock` fails without it) |
| `--profile` | No — Bedrock-only |
| `--opencode-default` | No — only valid on opencode, and only meaningful for Bedrock |

#### What it records

Choosing `other` replaces the Bedrock action set with a single pending action,
`non-bedrock-provider-configuration`. That action is **acknowledge-gated**, so
passing `--acknowledge` marks it `done` in the same command and `--check` goes
clean immediately.

For contrast, the action sets are:

| `--provider` | Harness | Pending actions |
| --- | --- | --- |
| `other` | any | `non-bedrock-provider-configuration` (auto-`done` with `--acknowledge`) |
| `amazon-bedrock` | any | `bedrock-model-access` |
| `amazon-bedrock` | + Kiro IDE | also `kiro-ide-chat-model` |
| `amazon-bedrock` | + Copilot | also `copilot-byok-configuration` |
| `amazon-bedrock` | + Cursor | also `cursor-provider-configuration` |

Inspect and verify:

```bash
aidlc config providers --show           # human-readable, lists pending actions
aidlc config providers --show --json    # machine-readable, for CI
aidlc config providers --check          # exit non-zero while any action is pending
```

Verified output after the opt-out:

```text
Providers configuration for opencode
  Provider: other
  Region: shipped fallback
  Profile: default credential chain
  Offline credentials: not found
  Files carrying provider settings:
    provider answers and pending actions: .aidlc/tools/data/harness.json
```

The `--json` record shows the action already settled:

```json
{ "provider": "other", "acknowledged": true,
  "pendingActions": [{ "id": "non-bedrock-provider-configuration", "status": "done" }] }
```

and `--check` exits **0** with `providers configuration is clean for opencode`.

The decision is stored in **`.aidlc/tools/data/harness.json`** — a committed
project file, so the whole team inherits the choice.

`--check` is the CI-friendly gate. Start over with:

```bash
aidlc config providers --reset --yes
```

#### Important limitation

The record stores `other` as an **opaque value** — it does not distinguish Azure
from Vertex from a self-hosted endpoint, and it stores no endpoint, key, or
model name. It is a decision record, not configuration. The real Azure/GCP
settings live in the harness:

| Harness | Where the Azure/GCP settings actually go |
| --- | --- |
| opencode | Global opencode config `provider` block, **plus** per-agent `model:` overrides in the project `opencode.json` (see the opencode note above) |
| Copilot CLI | `COPILOT_PROVIDER_BASE_URL` / `COPILOT_PROVIDER_TYPE` / `COPILOT_MODEL` / `COPILOT_PROVIDER_WIRE_MODEL` + bearer token |
| Copilot in VS Code | Model picker or a Custom Endpoint provider |

#### Recommended order

```bash
# 1. Configure the provider in the harness itself (Azure/Vertex endpoint + auth)
# 2. Verify a plain chat turn works in the harness
# 3. Then record the decision in AI-DLC:
cd /path/to/your-project
aidlc config providers --provider other --acknowledge --yes
aidlc config providers --check
aidlc doctor
```

Doing step 3 first is harmless but meaningless — nothing validates the endpoint.

#### Interactive mode

A bare `aidlc config providers` (with a TTY) prompts:

```text
    1. amazon-bedrock   ...
    2. other
```

Choosing **2** takes the same `other` path and asks for the acknowledgement.
Note the interactive default is derived from **offline AWS credential
detection** — if AWS credentials happen to exist on the machine it pre-selects
`amazon-bedrock`, so on an Azure/GCP box explicitly pick option 2 rather than
accepting the default.

> Scope note: AI-DLC does not validate your provider credentials or endpoints.
> If the harness can talk to Azure or Vertex, AI-DLC works; if it cannot, that
> is a harness/provider issue, not an AI-DLC one. The Bedrock examples above are
> the ones documented and exercised in this repository — Azure and Vertex shapes
> come from the harnesses' own provider documentation, so confirm them against
> `copilot help providers` / the opencode provider docs for your version.

---

## Quick reference card

```bash
# machine
aidlc --version
aidlc update
aidlc doctor

# project
aidlc config --harness opencode      # or --harness copilot
aidlc config --dry-run
```

```text
# in-session
/aidlc --doctor
/aidlc <what you want to build>
/aidlc express | feature <x> | bugfix <x>
/aidlc --status
/aidlc --resume
/aidlc space <name>
```

---

## Further reading

- [`docs/guide/harnesses/opencode.md`](harnesses/opencode.md)
- [`docs/guide/harnesses/copilot.md`](harnesses/copilot.md)
- [`docs/guide/01-getting-started.md`](01-getting-started.md)
- [`docs/guide/02-your-first-workflow.md`](02-your-first-workflow.md)
- [`docs/guide/workflow-profiles.md`](workflow-profiles.md)
- [`docs/guide/18-install-and-lifecycle.md`](18-install-and-lifecycle.md)
