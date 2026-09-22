# AI-DLC on a Brownfield Project — Step-by-Step

A detailed walkthrough for running AI-DLC against an **existing codebase**.

"Brownfield" is not a mode you switch on. It is a **classification the engine
detects** during Initialization, and it changes the workflow in four concrete
ways:

1. **Stage 2.1 Reverse Engineering executes** instead of self-skipping. It builds
   a durable, 9-artifact knowledge base of your repo.
2. Downstream design stages **consume that knowledge base** rather than designing
   against nothing.
3. **Brownfield safeguards** activate around any code modification — blast
   radius, test baseline, diff preview, rollback plan.
4. The **composer** (if you use adaptive workflows) scores the whole grid
   differently, because existing structure can justify compressing discovery.

> Prerequisite: your project is already installed and configured. If not, work
> through `opencode-copilot-manual.md` first (install → `aidlc config` → provider → trust →
> `aidlc doctor`).
>
> Starting a brand-new project instead? Use `greenfield-walkthrough.md`.

---

## Step 1 — Prepare the repository

Do these **before** the first `/aidlc`. They materially improve the scan.

### 1.1 Commit or stash outstanding work

Reverse Engineering reads your working tree. A clean tree makes the resulting
knowledge base reproducible, and makes later `git diff` reviews of generated
code readable.

```bash
cd /path/to/your-project
git status
git stash            # or commit
```

### 1.2 Confirm the build and tests pass *now*

This is the most important preparation step. AI-DLC's brownfield safeguard is a
**test baseline**: it records the suite result before changes and compares after.
A suite that is already red makes "did we regress?" unanswerable.

```bash
<your build command>
<your test command>
```

Record the numbers — total / passing / failing / skipped / coverage. If the suite
is already failing, either fix it first or be explicit at the gate that the
baseline is red.

### 1.3 Know your repo layout

The detector excludes `node_modules/`, `.git/`, the harness directory
(`.aidlc/`, `.github/`, `.opencode/`, …) and the `aidlc/` workspace. It scans
everything else. If your repo contains large vendored trees or generated output,
expect them to lengthen the scan.

---

## Step 2 — Verify the project classifies as brownfield

The classification is made by the **Workspace Detection** stage, deterministically.

### What makes a project brownfield

**ANY one** of these is sufficient:

| Signal | Example |
| --- | --- |
| Source files in a recognized language | `.js .ts .jsx .tsx .py .java .go .rs .rb .cs .cpp .c .kt .swift .php` |
| Application framework config | `next.config`, `vite.config`, `angular.json`, … |
| Package manifest with application deps | `package.json` (non-dev deps), `requirements.txt`, `Cargo.toml`, `go.mod`, `pom.xml` |
| Application source directories | `src/`, `app/`, `lib/`, `pages/`, `components/` |
| A parseable root `.gitmodules` with ≥1 submodule path | declares code even before submodules are initialized |

If **none** of these fire, the project is not brownfield and this guide does not
apply — use `greenfield-walkthrough.md` instead.

### What does NOT make it brownfield

README, `.gitignore`, LICENSE, editor configs, empty directories, CI/CD
boilerplate without application code, the harness directory, and the `aidlc/`
workspace.

### Nested projects

If no signal fires at the root, a **nested fallback** walks container
directories up to **three levels** down, re-applying the same signals. This
catches layouts like `services/api/src/main.py`. It does not descend below a hit,
and never runs when the root already has a source signal.

### Checking and overriding

After Initialization, the classification is written to your state file:

```bash
grep -i "Project Type" aidlc/spaces/*/intents/*/aidlc-state.md
```

You want `**Project Type**: Brownfield`.

If it reports Brownfield but you intend to treat the project as a fresh start
(the documented example is a `create-next-app` scaffold), that is a greenfield
workflow — see `greenfield-walkthrough.md` for the override procedure.

If it reports Greenfield but the project genuinely contains code, check that the
code is not inside an excluded directory, then describe the work again so a
fresh intent is minted and the `workspace-detection` stage re-runs against the
corrected tree:

```bash
/aidlc <describe the work you want to do>
```

Project Type is recorded at intent creation, and intents are auto-created the
first time you describe work — there is no separate re-initialize command. To
correct an existing workflow in place, edit the **Project Type** line in
`<record>/aidlc-state.md`.

---

## Step 3 — Choose the right workflow profile

Profile choice matters on brownfield because the existing code already answers
some questions.

| Your situation | Profile | Stages | Why |
| --- | --- | --- | --- |
| Add a production feature to an existing system | `feature` | 33/33 | Full lifecycle; widest chance to surface integration unknowns |
| Established lifecycle, no Ideation needed | `classic` | 26/33 | The implicit default |
| Fix a known defect | `bugfix` | 9/33 | Keeps workspace understanding + Code Gen + Build/Test; drops discovery |
| Improve structure, behavior unchanged | `refactor` | 10/33 | Built for exactly this: understand → define change → implement → prove no regression |
| CVE / vulnerability response | `security-patch` | 10/33 | Focused response path |
| Environments, IaC, deployment, cost | `infra` | 13/33 | Drops user-facing product ceremony |
| Regulated / audit-sensitive change | `enterprise` | 33/33 | Comprehensive depth, full traceability |
| Requirements already clear, want speed | `express` | 10/33 | RE stays conditional so you still get brownfield understanding |
| Testing whether an approach works | `poc` | 8/33 | Evidence for a later decision, not production-ready |

Stage counts verified directly against the engine:

```text
$ aidlc engine graph scope <profile> | wc -l
express 10 · poc 8 · bugfix 9 · refactor 10 · feature 33 · enterprise 33
```

**Reverse Engineering is in the static route of 10 of the 11 profiles** —
including `express` and `poc`. It is conditional on *project type*, not on
profile.

> **Exception: `infra` does not include it.** Verified against the stage's
> `scopes:` list. Infrastructure work concentrates on requirements, NFRs, IaC,
> CI/CD, deployment, and observability, and does not build the code knowledge
> base. If you need the codebase understood, choose a different profile.

Start it:

```text
/aidlc refactor Extract the billing calculator into its own module
/aidlc bugfix Orders created near midnight get yesterday's date
/aidlc feature Add webhook delivery with retry to the notifications service
```

Or describe the work and let AI-DLC propose a profile:

```text
/aidlc The payment retry logic is duplicated in three services and drifting
```

---

## Step 4 — Initialization (stages 0.x)

The engine scaffolds the workspace, runs Workspace Detection, and writes
`aidlc-state.md`.

On brownfield, State Init sets:

- **Project Type**: Brownfield
- **First post-initialization stage**: `reverse-engineering` (Inception 2.1)

You will be asked to confirm the scope:

```text
▸ Approve scope? [Yes / Change scope / Change depth / Change test strategy]
```

Check the stage count and that Reverse Engineering is listed under
**Stages to Execute**, not **Stages to Skip**.

---

## Step 5 — Stage 2.1 Reverse Engineering (the brownfield stage)

This is the stage that exists *because* your project is brownfield. Budget real
time for it — it is the most expensive stage in the Inception phase.

### How it runs

`mode: pipeline` — a **two-link chain**, not a mob:

| Link | Agent | Does |
| --- | --- | --- |
| 1 | `aidlc-developer-agent` | Scans the code, returns structured results |
| 2 | `aidlc-architect-agent` | Synthesizes and **writes** the 9 artifacts |

There are no contribution files on a pipeline stage. On resume the conductor
reads `directive.pipeline.completed` and dispatches only the first missing link.

### The rerun guard — answer this carefully

The knowledge base is a **space-level store shared across intents**, so before
scanning, the stage checks freshness:

```text
aidlc engine workspace codekb-scope-diff --repo <repo>
```

| Verdict | Meaning | What you are offered |
| --- | --- | --- |
| `NO_STORE` | First scan for this repo | Proceeds, no question |
| `CURRENT` | Analyzed paths unchanged since built | **Reuse** / Full rescan / Focused scan |
| `STALE` / `UNVERIFIED` / `UNKNOWN_SCOPE` | Out of date, unverifiable, or predates scope tracking | Full rescan / Focused scan (**no reuse option**) |

The three answers differ in an important way:

- **Reuse** — no scan. Downstream stages read the store as-is. Fastest; correct
  when the store is verified-current *and* its coverage includes your work area.
- **Full rescan** — **REPLACES all 9 artifacts**, covering the whole repo.
- **Focused scan** — **MERGES** into the existing store: scans this intent's
  area, preserves prior prose outside it, and demotes unverifiable deep coverage
  to shallow. This is how knowledge accumulates across intents.

> If `CURRENT` is reported but your intent targets code **outside** the store's
> analyzed paths, the reuse option is withheld — you will only be offered rescan
> vs focused. That is deliberate.

Prefer **Focused scan** for ongoing work on a large repo; prefer **Full rescan**
after a major restructure.

### Multi-repo

The stage runs **per repo** the intent touches, resolved from the intent's
`repos` array in `intents.json`:

- No `repos` recorded → runs once against the workspace root, receipts unqualified.
- One or more registered repos → the guard decision is resolved per repo, each
  repo's sibling directory `<workspace>/<repo>/` is scanned, and each writes to
  its own independent store. Selected scans may run as parallel subagents.

### What it produces

Nine artifacts, written to the **space-level, per-repo** store:

```text
aidlc/spaces/<active-space>/codekb/<repo>/
├── business-overview.md
├── architecture.md
├── code-structure.md
├── api-documentation.md
├── component-inventory.md
├── technology-stack.md
├── dependencies.md
├── code-quality-assessment.md
└── reverse-engineering-timestamp.md
```

Note the location: **`codekb/` is a sibling of `intents/`**, at the space level —
not inside the intent record. That is what lets later intents reuse it.

### Review it properly

This is the highest-leverage review in the whole workflow: every downstream
design stage reads these files. Errors here propagate.

Read at minimum `architecture.md`, `component-inventory.md`, and
`code-quality-assessment.md`. Check that the component boundaries match how your
team actually thinks about the system. Choose **Request Changes** if they do not.

> Not to be confused with **CodeKB the MCP server** — an optional external tool
> serving pre-computed call graphs. The `codekb/` directory above is AI-DLC's own
> local artifact store, unrelated to it.

---

## Step 6 — Stage 2.2 Practices Discovery (conventions from your code)

On brownfield this stage is grounded in the Reverse Engineering evidence: your
conventions are **embodied in the existing code and test trees**, so they are
inferred rather than chosen.

It is a hub-and-spoke: `aidlc-pipeline-deploy-agent` drafts; `aidlc-quality-agent`,
`aidlc-developer-agent`, and `aidlc-devsecops-agent` inspect that draft in
parallel, blind to each other. A human interview resolves evidence gaps and
policy judgments. The lead then integrates into `team-practices.md`,
`discovered-rules.md`, and `evidence.md`.

After **Approve**, `practices-promote` writes:

```text
aidlc/spaces/<active-space>/memory/team.md
aidlc/spaces/<active-space>/memory/project.md
```

…and records a `PRACTICES_AFFIRMED` receipt. **A missing, stale, or failed
promotion leaves the gate open and the stage incomplete** — if you see the gate
reopen, that is why.

Invest in the interview here. These rules govern every later code-generation
stage in the space.

---

## Step 7 — Inception continues (2.3 → 2.9)

From here every stage reads the codekb.

| Stage | Brownfield character |
| --- | --- |
| 2.3 Requirements Analysis | Requirements are framed against existing capability, not a blank page |
| 2.4 User Stories | Mob stage; personas often already implied by existing users |
| 2.5 Refined Mockups | On a brownfield redesign, one design pass grounded in current screens can replace the rough→refined pair |
| 2.6 Domain Design | Components and entity ownership must reconcile with the discovered inventory; ADRs record why you diverge from existing patterns |
| 2.7 Units Generation | Decomposition into Units of work |
| 2.8 Contract Design | Must respect existing API contracts unless you explicitly version them |
| 2.9 Delivery Planning | Bolts; brownfield adds **Transformation Scope** and **Multi-Module Coordination** considerations |

At each gate you get **Approve** / **Request Changes**. Reviewer findings are
listed with whether each was resolved, remains open, or was accepted as a risk.

---

## Step 8 — Construction with brownfield safeguards

This is where existing code gets modified, and where the safeguards matter.

### The safeguard matrix

| Safeguard | What it does | When |
| --- | --- | --- |
| **Blast Radius Analysis** | Identifies affected files/components and their downstream dependents | Before code generation (3.5) |
| **Test Baseline** | Runs existing tests **before** changes to establish a baseline | Before code generation (3.5) |
| **Diff Preview** | Shows exact proposed changes before applying | Before any file modification |
| **Impact Analysis** | Documents affected APIs, components, dependencies | During RE (2.1) and Code Gen (3.5) |
| **Test Validation** | Re-runs tests **after** changes to confirm nothing broke | After code generation (3.6 Build and Test) |
| **Rollback Plan** | Documents how to undo the change | Before deployment (4.3) |

### Blast radius — what to expect

Before modifying existing code the agent should:

1. List every file that will change.
2. For each, identify imports/consumers, test files, configuration references.
3. Classify impact: **low** (isolated), **medium** (2–3 dependents), **high**
   (cross-cutting).
4. Present the impact summary to you before proceeding.

**If you are shown a `high` classification, slow down.** That is the signal to
consider splitting the Unit.

### Test baseline protocol

1. Full suite runs before **any** code change.
2. Records total / passing / failing / skipped / coverage.
3. Suite re-runs after code generation.
4. **New failures = regressions introduced by the change.**
5. Regressions are fixed before proceeding.

This is why Step 1.2 mattered. A red baseline makes step 4 meaningless.

### The walking skeleton gate and the ladder prompt

The first in-scope Construction stage is **always gated**. Immediately after you
approve it, the **ladder prompt** fires exactly once, asking how much autonomy
the rest of Construction gets (e.g. gate every Bolt vs. run through).

Your answer is stored in `aidlc-state.md` as `Construction Autonomy Mode` and is
respected across session resume.

**Prefer the more conservative setting on your first workflow.** You are
modifying code that already works, so the value of per-Bolt inspection is high.

Stage 3.5 runs as a subagent per Unit. The per-Unit gate is suppressed — a single
stage-level gate replaces it after the last Unit settles.

---

## Step 9 — Verify the change was safe

Beyond AI-DLC's own gates, do your own check before deployment stages:

```bash
git diff --stat                 # scope of change
git diff                        # read it
<your test command>             # compare against the Step 1.2 numbers
```

Cross-check against the recorded baseline. If coverage dropped or new tests were
skipped, raise it at the gate rather than approving.

---

## Step 10 — Commit the workspace records

The AI-DLC records are **shared work** and travel by git:

```bash
git add aidlc/
git commit -m "AI-DLC: <intent> — records and artifacts"
git push
```

What is committed: `memory/**`, `codekb/**`, `intents/intents.json`,
`aidlc-state.md`, `audit/*.md` (per-clone shards), and stage artifacts.
What stays ignored: per-user cursors and machine-local runtime.

Committing `codekb/**` is what lets your teammates — and your next intent —
reuse the knowledge base instead of rescanning.

---

## Step 11 — Subsequent intents on the same repo

The second workflow is much cheaper than the first, because the knowledge base
already exists.

```text
/aidlc bugfix <the next piece of work>
```

At stage 2.1 the guard will report `CURRENT` (if nothing changed) and offer
**Reuse**. Use it.

Keep the store healthy over time:

- After a **major restructure** → Full rescan.
- After **normal feature work in a new area** → Focused scan (merges, preserves
  prior coverage).
- **Verified unchanged, coverage fits** → Reuse.

---

## Brownfield checklist

```text
[ ] Working tree clean (committed or stashed)
[ ] Build passes
[ ] Test suite passes — baseline numbers recorded
[ ] aidlc doctor clean (esp. trust + hook PATH)
[ ] Project Type: Brownfield confirmed in aidlc-state.md
[ ] Reverse Engineering listed under Stages to Execute
[ ] Profile chosen deliberately (refactor / bugfix / feature / …)
[ ] codekb rerun-guard answer chosen deliberately (reuse / full / focused)
[ ] 9 codekb artifacts reviewed — architecture + component-inventory at minimum
[ ] Practices Discovery interview answered properly; team.md/project.md written
[ ] Blast radius reviewed before code generation
[ ] Post-change test run compared against the baseline
[ ] aidlc/ committed and pushed
```

---

## Common brownfield pitfalls

| Pitfall | Consequence | Avoid by |
| --- | --- | --- |
| Starting with a red test suite | The regression check cannot distinguish new breakage from old | Fix or explicitly declare the baseline (Step 1.2) |
| Rubber-stamping the 2.1 gate | Every downstream design stage inherits wrong structure | Read `architecture.md` + `component-inventory.md` |
| Choosing **Full rescan** every time | Discards accumulated cross-intent knowledge | Use **Focused scan** for ordinary work |
| Not committing `aidlc/` | Teammates rescan from scratch; no shared audit trail | Step 10 |
| Picking `feature` for a pure restructure | Excess ceremony for behavior-preserving work | Use `refactor` |
| Picking `refactor` when behavior changes | Under-specified user-visible change | Use `feature` |
| Approving a `high` blast radius without splitting | Wide, hard-to-review change | Split the Unit |
| Maximum Construction autonomy on run #1 | Unreviewed edits to working code | Gate every Bolt the first time |

---

## Appendix A — Verifying the classification yourself

Everything in this guide about detection and routing is **deterministic** — you
can confirm it without spending a model token.

### Detection

```bash
cd /path/to/your-project
aidlc engine workspace detect
```

Brownfield output:

```text
Project type: Brownfield
Languages: JavaScript
Frameworks: Unknown
Build system: npm (package.json)
Valid scopes: bugfix, classic, enterprise, express, feature, infra, mvp, poc,
              refactor, security-patch, workshop
```

### Confirming stage 2.1 will execute

`aidlc engine graph scope <profile>` returns the **static route** only — the
conditional filter is applied by the screen, not the scope listing. Check the
screen instead:

```bash
aidlc engine graph ars \
  --iae 0.5 --csu 0.5 --ve 0.5 --r 0.5 --ua 0.5 \
  --scope feature --project-type brownfield
```

The `reverse-engineering` entry on a brownfield project:

```json
{
  "stage": "reverse-engineering",
  "number": "2.1",
  "decision": "EXECUTE",
  "targets": ["csu"],
  "cost": 4,
  "maxTargetScore": 0.5,
  "threshold": 0.4,
  "screen": "component",
  "reason": "reduces CSU=0.50 > threshold 0.4 (cost 4)"
}
```

`decision: EXECUTE` with `screen: component` means the stage was scored normally
and earned its place. If you instead see `screen: "project-type"`, the engine has
classified the project as not-brownfield — revisit Step 2.

### The codekb rerun guard

```bash
aidlc engine workspace codekb-scope-diff
```

On a repo that has never been scanned:

```text
NO_STORE: no reverse-engineering-timestamp.md at
aidlc/spaces/default/codekb/<repo>/ - first scan, nothing to compare.
```

---

## Reference

- [opencode and GitHub Copilot Manual](opencode-copilot-manual.md) — install, config, provider, trust, doctor
- [Greenfield Walkthrough](greenfield-walkthrough.md) — the companion guide, for a brand-new project
- [Workflow Profiles](workflow-profiles.md)
- [Scopes, Depth, and Test Strategy](05-scopes-and-depth.md)
- [Spaces and Intents](03-spaces-and-intents.md)
- [Worked Examples](16-worked-examples.md)
- [Your First Workflow](02-your-first-workflow.md)
- [Artifacts Reference](14-artifacts-reference.md)
