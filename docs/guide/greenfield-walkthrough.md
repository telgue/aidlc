# AI-DLC on a Greenfield Project — Step-by-Step

A detailed walkthrough for running AI-DLC on a **new codebase** — an empty or
near-empty repository where nothing has been built yet.

"Greenfield" is not a mode you switch on. It is a **classification the engine
detects** during Initialization, and it shapes the workflow in four ways:

1. **Stage 2.1 Reverse Engineering is screened out** before scoring — there is
   no code to reverse engineer.
2. **Practices Discovery asks instead of infers.** With no existing code to read
   conventions from, it interviews you across all five practice areas using your
   org defaults as suggestions.
3. **Ideation carries more weight.** Nothing existing constrains the problem, so
   the discovery phase is where ambiguity actually gets resolved.
4. **Brownfield safeguards do not apply** — there is no working behavior to
   regress, which changes how much Construction autonomy is defensible.

> Prerequisite: your project is already installed and configured. If not, work
> through `opencode-copilot-manual.md` first (install → `aidlc config` → provider → trust →
> `aidlc doctor`).

---

## Step 1 — Create and prepare the repository

### 1.1 Initialize

```bash
mkdir my-new-service
cd my-new-service
git init
```

A greenfield project genuinely can be this empty. A `README.md`, `LICENSE`, and
`.gitignore` are fine and do **not** change the classification (verified — see
Appendix A).

### 1.2 Do NOT scaffold first

This is the one mistake worth calling out up front. Running `create-next-app`,
`cargo new`, `spring init`, or similar **before** your first `/aidlc` will make
the engine classify the project as **Brownfield**, because a framework config
and package manifest are brownfield signals.

That is not fatal — but it means stage 2.1 will execute and scan your scaffold,
and design stages will treat generated boilerplate as existing architecture to
reconcile with. If you want AI-DLC to drive the technology choice, start empty.

If you have already scaffolded and want greenfield treatment, see Step 2.4.

### 1.3 Configure and verify

```bash
aidlc config --harness copilot     # or --harness opencode
aidlc doctor
```

`aidlc doctor` must be clean on trust and hook PATH before you start — see
`opencode-copilot-manual.md` Steps 3–4. On greenfield there is no test suite to baseline, so this
health check is the whole of your pre-flight.

---

## Step 2 — Verify the project classifies as greenfield

The classification is made deterministically by the **Workspace Detection**
stage.

### Greenfield requires ALL of these

| Condition | Meaning |
| --- | --- |
| No source files in a recognized language | none of `.js .ts .jsx .tsx .py .java .go .rs .rb .cs .cpp .c .kt .swift .php` |
| No application framework configuration | no `next.config`, `vite.config`, `angular.json`, … |
| No application package manifest | no `package.json` with non-dev deps, `requirements.txt`, `Cargo.toml`, `go.mod`, `pom.xml` |
| No application source directories | no `src/`, `app/`, `lib/`, `pages/`, `components/` |

**Any single** brownfield signal flips the classification. A root `.gitmodules`
with at least one submodule path also counts as brownfield — repo metadata
declares code even before the submodules are initialized.

### What is explicitly harmless

README, `.gitignore`, LICENSE, editor configs, empty directories, CI/CD
boilerplate without application code, the harness directory (`.aidlc/`,
`.github/`, `.opencode/`, …), and the `aidlc/` workspace itself.

### Check it

```bash
aidlc engine workspace detect
```

Expected:

```text
Project type: Greenfield
Languages: Unknown
Frameworks: Unknown
Build system: Unknown
```

After Initialization, confirm it landed in state:

```bash
grep -i "Project Type" aidlc/spaces/*/intents/*/aidlc-state.md
```

You want `**Project Type**: Greenfield`.

### 2.4 If it says Brownfield but you want greenfield

The documented case is a `create-next-app` scaffold you intend to treat as a
fresh start. Two options:

```bash
# 1. Hand-edit the **Project Type** line in <record>/aidlc-state.md, or
# 2. Remove the scaffold, then describe the work again so a fresh intent is
#    minted and detection re-runs:
/aidlc Build <describe your project>
```

Project Type is decided by the `workspace-detection` stage when an intent is
created, and intents are auto-created the first time you describe work. Running
detection again therefore means starting a new intent against the cleaned tree
— there is no separate re-initialize command.

---

## Step 3 — Choose the right workflow profile

On greenfield, profile choice is mostly a question of **how much discovery you
need** and **whether this reaches production**.

| Your situation | Profile | Stages | Why |
| --- | --- | --- | --- |
| New product, problem still fuzzy | `feature` | 33/33 | Full lifecycle including Ideation — the widest chance to surface unknowns |
| Regulated / audit-sensitive from day one | `enterprise` | 33/33 | Comprehensive depth and full traceability |
| Real first product increment, no prod ops yet | `mvp` | 23/33 | Full Inception + Construction, trims Ideation ceremony, skips Operation |
| "Can this even work?" | `poc` | 8/33 | Minimal path to requirements → code → tests |
| Requirements already clear, want speed | `express` | 10/33 | Shortest supported requirements → code → tests path |
| Established lifecycle, no Ideation needed | `classic` | 26/33 | The implicit engine default |
| Standing up environments / IaC first | `infra` | 13/33 | Requirements, NFRs, IaC, CI/CD, deployment, observability |
| Facilitated training session | `workshop` | 26/33 | Standard depth, minimal tests |

Verified stage counts:

```text
$ aidlc engine graph scope <profile> | wc -l
poc 8 · bugfix 9 · express 10 · refactor 10 · infra 13
mvp 23 · classic 26 · workshop 26 · feature 33 · enterprise 33
```

**Profiles to avoid on greenfield:** `bugfix`, `refactor`, and `security-patch`
all presuppose existing code. They are brownfield-shaped by design.

**The common greenfield choice is `feature` or `mvp`.** Choose `express` or
`poc` only when you genuinely already know what you are building — their speed
comes from removing exactly the decision surfaces a new project usually needs.

Start it:

```text
/aidlc feature Build a REST API for inventory management
/aidlc mvp A task tracker with projects, assignments, and due dates
/aidlc poc Can we serve embeddings fast enough from a single node?
```

Or describe the work and let AI-DLC propose a profile:

```text
/aidlc I want to build an internal tool for tracking equipment loans
```

---

## Step 4 — Initialization (stages 0.x)

The engine scaffolds the workspace, runs Workspace Detection, and writes
`aidlc-state.md`.

On greenfield, State Init:

- sets **Project Type**: Greenfield
- marks `reverse-engineering` as **SKIP**
- sets the first post-initialization stage to **`requirements-analysis`** (2.3)

You are then asked to confirm the route:

```text
▸ Approve scope? [Yes / Change scope / Change depth / Change test strategy]
```

Check that `reverse-engineering` appears under **Stages to Skip**, and that the
stage count matches the profile you chose.

The workspace is created at:

```text
aidlc/spaces/default/intents/<YYMMDD>-<label>/
```

---

## Step 5 — Ideation (stages 1.1 → 1.7)

**This phase matters more on greenfield than anywhere else.** On a brownfield
project, existing code answers many questions implicitly. Here, nothing does.

| Stage | Produces |
| --- | --- |
| 1.1 Intent Capture | The problem statement, target customer, stakeholder map — every claim source-tagged |
| 1.2 Market Research | Competitive and contextual grounding |
| 1.3 Feasibility & Constraints | What is actually achievable, and the hard limits |
| 1.4 Scope Definition | In-scope / out-of-scope boundaries, `scope-document.md`, `intent-backlog.md` |
| 1.5 Team Formation | Who does what |
| 1.6 Rough Mockups | Low-fidelity UI direction (greenfield gets the rough → refined pair) |
| 1.7 Approval & Handoff | The initiative brief; phase-boundary verification of intent-to-scope traceability |

### Source tagging

Intent Capture claims must carry source tags resolving to confirmed sources and
your answers — a sensor checks this. Unsupported content is either asked as a
follow-up or parked under `## Assumptions & Open Questions`. **Retained
assumptions require your explicit acceptance** and stay labeled as assumptions.

Do not let assumptions accumulate silently here. On greenfield they compound:
an unexamined assumption in 1.1 becomes an architectural commitment by 2.6.

### Out-of-scope is the valuable half

Spend real effort on 1.4's **out-of-scope** list. On a new project the natural
failure is unbounded growth, and the out-of-scope list is the only artifact that
pushes back.

> Profiles differ here. `classic` and `express` skip Ideation entirely; `mvp`
> trims selected ceremony. If you chose one of those and the problem is still
> fuzzy, switch to `feature`.

---

## Step 6 — Stage 2.2 Practices Discovery (conventions you choose)

This is the stage whose **behavior differs most** between greenfield and
brownfield, so it is worth understanding.

On brownfield, conventions are *inferred* — they are embodied in existing code
and test trees. On greenfield there is nothing to read, so the stage **asks**.

### How it runs

A hub-and-spoke ensemble:

1. **Lead draft** — `aidlc-pipeline-deploy-agent` reads the five matching
   sections from `aidlc/spaces/<active-space>/memory/org.md` and treats them as
   **suggested defaults, not established team facts**. It drafts all four
   artifacts.
2. **Blind support review** — `aidlc-quality-agent` (testing posture, coverage,
   CI gates), `aidlc-developer-agent` (naming, layer boundaries, error handling,
   file organization), and `aidlc-devsecops-agent` (lint/format, SAST/DAST,
   secret and dependency scanning, supply chain) each inspect the draft in
   parallel, **mutually blind** — no spoke sees another's contribution.
3. **Interview** — on greenfield you are asked about **all five** areas:

   - Way of Working
   - Walking Skeleton
   - Testing Posture
   - Deployment
   - Code Style

   with the matching `org.md` sections offered as suggested answers.
4. **Integration** — the lead merges all three contributions into
   `team-practices.md`, `discovered-rules.md`, and `evidence.md`.

> The stage is **not skipped** on greenfield. It is skipped only when the active
> scope's compiled plan marks `practices-discovery` as SKIP.

### After Approve

`practices-promote` writes:

```text
aidlc/spaces/<active-space>/memory/team.md
aidlc/spaces/<active-space>/memory/project.md
```

and records a `PRACTICES_AFFIRMED` receipt. **A missing, stale, or failed
promotion leaves the gate open and the stage incomplete** — if the gate reopens,
that is why.

### Why this is your highest-leverage interview

These five answers become the rules every later code-generation stage in the
space obeys — and they persist across every future intent. On greenfield you are
setting them from scratch, with no existing code to correct a bad choice later.

Answer deliberately. "Whatever the default is" is a real decision here.

---

## Step 7 — Inception continues (2.3 → 2.9)

Because 2.1 is skipped, **Requirements Analysis is your first substantive
Inception stage**.

| Stage | Greenfield character |
| --- | --- |
| 2.3 Requirements Analysis | Framed against a blank page — functional and non-functional requirements both start empty |
| 2.4 User Stories | Mob stage: product drafts personas/stories; design, developer, and quality inspect blind; product integrates |
| 2.5 Refined Mockups | Full rough → refined pair (brownfield can compress to one pass grounded in current screens) |
| 2.6 Domain Design | **Unconstrained.** Components, entity ownership, and ADRs record choices rather than justifying divergence from existing patterns |
| 2.7 Units Generation | Decomposition into Units of work |
| 2.8 Contract Design | No legacy API contracts to respect — design them properly the first time |
| 2.9 Delivery Planning | Bolts with a Definition of Done, confidence hypothesis, and ownership |

### Domain Design deserves extra attention

2.6 is where greenfield projects are won or lost. There is no existing structure
pushing back on a bad boundary, and the ADR log in `decisions.md` is the only
record of *why* you chose what you chose.

Read the component catalogue and the entity-ownership table carefully. Ask
whether the boundaries match how your team will actually divide work. Choose
**Request Changes** if they do not — this is far cheaper to fix now than after
Construction.

At every gate you get **Approve** / **Request Changes**, with reviewer findings
listed as resolved, open, or accepted as a risk.

---

## Step 8 — Construction (stages 3.1 → 3.7)

| Stage | Purpose |
| --- | --- |
| 3.1 Functional Design | Behavior specification per Unit |
| 3.2 NFR Requirements | Performance, scale, availability targets |
| 3.3 NFR Design | How those targets are met |
| 3.4 Infrastructure Design | Runtime topology |
| 3.5 Code Generation | Per-Unit implementation (subagent) |
| 3.6 Build and Test | Verification |
| 3.7 CI Pipeline | Automation |

### The walking skeleton gate

The first in-scope Construction stage is **always gated**. On greenfield this is
the first moment real code exists — review it properly. The walking skeleton
establishes the project's actual shape, and everything after inherits it.

### The ladder prompt

Immediately after you approve the walking skeleton, the **ladder prompt** fires
exactly once, asking how much autonomy the rest of Construction gets (for
example, gate every Bolt vs. run through). Your answer is stored in
`aidlc-state.md` as `Construction Autonomy Mode` and is respected across session
resume.

**Higher autonomy is more defensible on greenfield than on brownfield.** There is
no working behavior to break, and no regression risk to existing users — the
worst case is code you discard. That said, gate at least the first Bolt or two
until you have seen that the generated code matches the practices you affirmed
in Step 6.

Stage 3.5 runs as a subagent per Unit. The per-Unit gate is suppressed — a
single stage-level gate replaces it after the last Unit settles.

### What does *not* apply

The brownfield safeguards — blast radius analysis, test baseline, diff preview
against existing behavior, rollback plan — are **not applicable** here. There is
no prior behavior to compare against and no existing dependents to analyze.

Your substitute is the test suite the workflow is building. Watch that it is
actually growing alongside the code, not deferred.

---

## Step 9 — Operation (stages 4.1 → 4.7, profile dependent)

| Stage | Purpose |
| --- | --- |
| 4.1 Environment Provisioning | Environments stood up |
| 4.2 Deployment Pipeline | Path to production |
| 4.3 Deployment Execution | The deploy itself |
| 4.4 Observability Setup | Logs, metrics, traces |
| 4.5 Performance Validation | Meeting the NFR targets from 3.2 |
| 4.6 Incident Response | Runbooks and escalation |
| 4.7 Feedback & Optimization | Closing the loop |

`mvp` skips this phase entirely. `feature` and `enterprise` include it. If you
chose `mvp` and the product now needs production operations, move to `feature`
or `enterprise` for the next intent.

---

## Step 10 — Commit the workspace records

AI-DLC records are **shared work** and travel by git:

```bash
git add aidlc/
git commit -m "AI-DLC: <intent> — records and artifacts"
git push
```

Committed: `memory/**` (including the `team.md` and `project.md` you just
affirmed), `codekb/**`, `intents/intents.json`, `aidlc-state.md`, `audit/*.md`
(per-clone shards), and stage artifacts.

Ignored: per-user cursors and machine-local runtime.

On greenfield the most valuable thing you are committing is `memory/team.md` —
the practices you chose in Step 6, now available to every future intent and
every teammate.

---

## Step 11 — Your project is now brownfield

This is the most important thing to understand about greenfield: **it is a
one-time state.**

After your first Construction phase the repository contains source files, a
package manifest, and source directories. The next intent will classify as
**Brownfield**, and:

- Stage 2.1 Reverse Engineering will **execute**, building the 9-artifact
  `codekb/` store
- Practices Discovery will **infer** from your new code instead of asking all
  five areas
- Brownfield safeguards will activate around code modification

That is correct and intended. From your second intent onward, follow
`brownfield-walkthrough.md`.

To make that transition smooth, before you finish this workflow:

- Make sure the test suite is **green** — it becomes the baseline for every
  future regression check.
- Commit `aidlc/` so `memory/team.md` carries forward.

---

## Greenfield checklist

```text
[ ] Repository initialized; NOT scaffolded with a framework generator
[ ] aidlc config done; aidlc doctor clean (trust + hook PATH)
[ ] aidlc engine workspace detect reports Greenfield
[ ] Project Type: Greenfield confirmed in aidlc-state.md
[ ] reverse-engineering listed under Stages to Skip
[ ] Profile chosen deliberately (feature / mvp / poc / express)
[ ] Ideation: out-of-scope list taken seriously; assumptions explicitly accepted
[ ] Practices Discovery: all five areas answered deliberately
[ ] memory/team.md and project.md written (PRACTICES_AFFIRMED recorded)
[ ] Domain Design component boundaries reviewed, not rubber-stamped
[ ] Walking skeleton reviewed before granting Construction autonomy
[ ] Test suite green at the end — it is the baseline for next time
[ ] aidlc/ committed and pushed
```

---

## Common greenfield pitfalls

| Pitfall | Consequence | Avoid by |
| --- | --- | --- |
| Scaffolding before the first `/aidlc` | Project classifies Brownfield; boilerplate treated as existing architecture | Start empty (Step 1.2) |
| Choosing `express` or `poc` for a real product | The removed decision surfaces are exactly the ones a new project needs | Use `feature` or `mvp` |
| Choosing `bugfix` / `refactor` / `security-patch` | These presuppose existing code | Not applicable to greenfield |
| Skipping Ideation on a fuzzy problem | Ambiguity surfaces later as rework in Domain Design | Use `feature`; do not pick `classic`/`express` |
| Accepting assumptions silently in 1.1 | They compound into architectural commitments by 2.6 | Explicitly accept or resolve each one |
| Answering Practices Discovery with "defaults" | Rules governing all future code generation set by accident | Treat it as the highest-leverage interview |
| Empty out-of-scope list | Unbounded growth | Spend real effort on 1.4 |
| Rubber-stamping Domain Design | No existing structure pushes back on a bad boundary | Read `components.md` and `decisions.md` |
| Maximum autonomy from the first Bolt | Code diverges from affirmed practices before you notice | Gate the first Bolt or two |
| Leaving the suite red at the end | The next (brownfield) intent has no usable regression baseline | Finish green |
| Not committing `aidlc/` | `memory/team.md` lost; teammates and future intents start over | Step 10 |

---

## Appendix A — Verifying the classification yourself

Detection and routing are **deterministic** — confirm them without spending a
model token.

### Detection

A repo holding only `README.md`, `LICENSE`, and `.gitignore`:

```bash
$ aidlc engine workspace detect
Project type: Greenfield
Languages: Unknown
Frameworks: Unknown
Build system: Unknown
Valid scopes: bugfix, classic, enterprise, express, feature, infra, mvp, poc,
              refactor, security-patch, workshop
```

Adding `package.json` (with a real dependency) and `src/index.js` flips it:

```text
Project type: Brownfield
Languages: JavaScript
Build system: npm (package.json)
```

### Proving that 2.1 is screened out

`aidlc engine graph scope <profile>` returns the **static route**, which is
identical for both project types — the conditional filter is applied by the
screen, not the scope listing. Run the screen with identical entropy scores and
vary only the project type:

```bash
aidlc engine graph ars \
  --iae 0.5 --csu 0.5 --ve 0.5 --r 0.5 --ua 0.5 \
  --scope feature --project-type greenfield
```

The `reverse-engineering` entry:

| Project type | `decision` | `screen` | `reason` |
| --- | --- | --- | --- |
| greenfield | `SKIP` | `project-type` | "project is greenfield - the stage's compiled condition restricts it to brownfield projects" |
| brownfield | `EXECUTE` | `component` | "reduces CSU=0.50 > threshold 0.4 (cost 4)" |

On greenfield the decision is made by the **`project-type` screen before any
component arithmetic** — the stage is never scored (`maxTargetScore: null`), so
the screen can never contradict the stage's own compiled condition.

---

## Appendix B — Greenfield vs brownfield at a glance

| | Greenfield | Brownfield |
| --- | --- | --- |
| Stage 2.1 Reverse Engineering | **SKIP** (project-type screen) | **EXECUTE** (9 artifacts per repo) |
| First post-initialization stage | `requirements-analysis` (2.3) | `reverse-engineering` (2.1) |
| `codekb/` store | Never created | `aidlc/spaces/<space>/codekb/<repo>/` |
| Practices Discovery source | `memory/org.md` defaults + interview on **all five** areas | Existing code, git history, CI config, RE artifacts; asks only what evidence cannot establish |
| Ideation weight | High — nothing else resolves ambiguity | Lower — existing structure answers implicitly |
| Mockups | Full rough → refined pair | One pass grounded in current screens can suffice |
| Domain Design | Unconstrained; ADRs record choices | Must reconcile with discovered inventory; ADRs justify divergence |
| Contract Design | Design from scratch | Respect existing contracts unless explicitly versioned |
| Safeguards | Not applicable | Blast radius, test baseline, diff preview, rollback |
| Sensible autonomy | Higher — no working behavior to break | Lower — gate Bolts on the first run |
| Typical profiles | `feature`, `mvp`, `poc` | `feature`, `refactor`, `bugfix`, `security-patch` |
| Durability | One-time state | Every subsequent intent |

---

## Reference

- [opencode and GitHub Copilot Manual](opencode-copilot-manual.md) — install, config, provider, trust, doctor
- [Brownfield Walkthrough](brownfield-walkthrough.md) — the guide for every intent after your first
- [Workflow Profiles](workflow-profiles.md)
- [Scopes, Depth, and Test Strategy](05-scopes-and-depth.md)
- [Spaces and Intents](03-spaces-and-intents.md)
- [Your First Workflow](02-your-first-workflow.md)
- [Worked Examples](16-worked-examples.md)
- [Artifacts Reference](14-artifacts-reference.md)
