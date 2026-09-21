# Walkthrough: Building an Agent Evaluation Platform on GitHub Copilot

A complete, step-by-step run of AI-DLC with **GitHub Copilot** as the harness.

The worked example is deliberately hard: build a unified **agent evaluation
platform** that merges three families of open-source tooling into one harness —

| Family | Libraries |
| --- | --- |
| Agent & RAG evaluation | **Ragas**, **DeepEval**, **MLflow** (evaluation + tracking) |
| Trace observability & trajectory analysis | **Arize Phoenix**, **TruLens**, **NVIDIA NeMo Agent Toolkit** |
| Security & safety red-teaming | **Promptfoo**, **PyRIT** |

This is exactly the kind of work AI-DLC exists for: many overlapping tools, no
single obvious architecture, real compliance exposure, and a strong temptation
to start coding before the problem is framed.

> **Scope of this chapter.** Everything about the *harness* (install, trust,
> hooks, what Copilot does differently) is factual and matches the shipped
> Copilot distribution — see [AI-DLC on GitHub Copilot](harnesses/copilot.md)
> for the reference. The *transcript* is an illustrative composite: your run
> will ask different questions and produce different decisions. The stage
> route, gates, and artifact paths are real.

---

## Part 1 — Set up the Copilot harness

### Step 1. Check the host versions

One AI-DLC install serves **both** Copilot surfaces — the standalone Copilot
CLI and VS Code agent mode. Both read the same `.github/{skills,agents,hooks}`
tree and the root `AGENTS.md`.

```bash
copilot --version   # need >= 1.0.74
code --version      # need >= 1.130 if you want VS Code agent mode
```

Those floors are not cosmetic. They are the verified line for PascalCase hook
registration, the blocking `PreToolUse` deny channel, the blocking `Stop` hook,
and `.github` skills/agents discovery. Below them, AI-DLC's guardrails degrade
silently.

### Step 2. Install AI-DLC

macOS, Linux, or WSL:

```bash
curl -fsSL https://github.com/telgue/aidlc/releases/latest/download/install.sh | sh
```

Windows PowerShell:

```powershell
irm https://github.com/telgue/aidlc/releases/latest/download/install.ps1 | iex
```

The installer places the native `aidlc` command plus every harness runtime. It
needs neither Bun nor Node.js. If a new shell cannot find `aidlc`, apply the
PATH line the installer printed.

### Step 3. Create and configure the project

```bash
mkdir agent-eval-platform && cd agent-eval-platform
git init
aidlc config --harness copilot
aidlc doctor
```

`aidlc config --harness copilot` writes two trees:

- **`.aidlc/`** — the engine (tools, hooks and the Copilot adapter, the 14
  persona agents, knowledge, scopes, sensors). Neither Copilot surface scans
  this directory.
- **`.github/`** — the only natively-consumed surface: `hooks/aidlc.json`,
  `agents/aidlc-*-agent.md`, and `skills/aidlc*/`. Every file is
  `aidlc`-prefixed, so this **merges** into an existing `.github/` without
  touching your workflows or issue templates.

Plus the root `AGENTS.md` (read by both surfaces) and the `aidlc/` workspace
shell — a *sibling* of `.aidlc/`, not a child. That is where your artifacts,
state, and audit log will live.

### Step 4. Trust the folder — the step people skip

Repo hooks run **only** when the project's absolute path appears in
`trustedFolders` in `~/.copilot/config.json`. Start the CLI once interactively
and accept the prompt:

```bash
copilot
```

For headless runs (`copilot -p "..."`) you additionally need:

```bash
export GITHUB_COPILOT_PROMPT_MODE_REPO_HOOKS=1
```

**Untrusted means every hook silently no-ops, with no warning anywhere.** You
would get a chatbot that looks like AI-DLC and enforces nothing — no audit log,
no reviewer scope bound, no state-transition guard. Verify:

```text
/aidlc --doctor
```

The doctor checks both trust conditions explicitly. Do not proceed until it is
clean.

### Step 5. Pick a model provider

Nothing in this install pins a model — agents inherit the session model on both
surfaces. Signed-in Copilot works as-is. BYOK works with no GitHub auth at all;
`copilot help providers` documents the environment variables. In VS Code, use
the model picker or a Custom Endpoint provider.

For this workload, prefer a large-context reasoning model. The Inception phase
holds nine libraries' capability surfaces in view simultaneously.

### Step 6. Seed what you already know (optional, high leverage)

AI-DLC reads a layered method stack from the active space's memory. Before
starting, record constraints you already consider settled:

```bash
$EDITOR aidlc/spaces/default/memory/project.md
```

For this example you might record: *Python 3.12; uv for dependency management;
OpenTelemetry as the trace substrate; evaluation results must be reproducible
from a pinned dataset hash; no eval payloads leave the VPC.*

Anything you write here becomes ambient context for every stage, and — more
importantly — the agents will argue with it explicitly rather than quietly
inventing something different.

---

## Part 2 — Choose the workflow profile

The platform is a greenfield product with a genuinely unclear problem boundary.
Three of the nine libraries overlap heavily (Ragas and DeepEval both score RAG
faithfulness; Phoenix and TruLens both capture traces; MLflow does both tracking
*and* evaluation). Deciding what to merge versus what to wrap is the actual work.

That means you want the Ideation phase. Choose **Feature** (33 stages, Standard
depth) or **Enterprise** if red-teaming results feed a regulated assurance
process.

```text
/aidlc feature Build a unified agent evaluation platform that merges Ragas, DeepEval, and MLflow for RAG and agent scoring; Arize Phoenix, TruLens, and NVIDIA NeMo Agent Toolkit for trace and trajectory observability; and Promptfoo plus PyRIT for security and safety red-teaming. One CLI, one result schema, one report.
```

If you prefer AI-DLC to shape the route around this specific task rather than
take a stock profile:

```text
/aidlc compose "Unify nine agent-evaluation libraries behind one CLI and one result schema, including red-teaming"
```

Compose proposes a tailored stage list and stops at an approve/edit/reject gate
before anything is created. It also works mid-workflow to re-shape the stages
you have not reached yet.

---

## Part 3 — The run, stage by stage

### Initialization (0.1–0.3) — auto-proceed

Three stages execute as a single deterministic tool call, well under a second,
with no interaction:

- **0.1 Workspace Scaffold** — creates the intent record at
  `aidlc/spaces/default/intents/<YYMMDD>-agent-eval-platform/`
- **0.2 Workspace Detection** — greenfield, empty repository, Python toolchain
  implied by `project.md`
- **0.3 State Init** — writes `aidlc-state.md` with scope `feature`, depth
  `Standard`, and the 33-stage route

> Progress: 3/33 overall | 3/3 INITIALIZATION complete. Next: Intent Discovery

### Ideation — where the nine libraries get sorted

This phase is the reason you did not pick Express.

**Intent Discovery** asks who the platform serves. The honest answer is usually
three different people with three different needs, and naming that early
prevents a mushy architecture:

> 1. **Application engineers** — "did my change make the agent worse?"
> 2. **ML/eval engineers** — "which retrieval config scores best on my golden set?"
> 3. **Security and safety reviewers** — "can this agent be jailbroken into
>    exfiltrating customer data?"

Answer in a chat message. On Copilot, questions render as **numbered prose
options** rather than a native picker — see [Part 5](#part-5--what-copilot-does-differently)
for why. Your typed reply is what advances the workflow.

**Market and Capability Research** (`aidlc-product-agent`) is where you should
push hardest. Ask it to produce an explicit overlap matrix rather than a list of
nine summaries:

```text
Before we go further: build a capability matrix over Ragas, DeepEval, MLflow,
Phoenix, TruLens, NeMo Agent Toolkit, Promptfoo, and PyRIT. Columns: offline
metric scoring, LLM-as-judge, dataset/golden-set management, span-level trace
capture, multi-step trajectory analysis, experiment tracking, adversarial prompt
generation, CI gating. Mark each cell primary / partial / none, and call out
every place two tools are genuinely redundant versus merely similar.
```

A typical outcome — and the one that shapes everything downstream — is that the
nine tools collapse into **three distinct roles**, not nine integrations:

| Role | Primary | Secondary / adapter | Redundant with primary |
| --- | --- | --- | --- |
| Metric scoring | Ragas (RAG metrics) | DeepEval (assertion-style unit tests) | — |
| Trace capture & trajectory | Phoenix (OTel-native spans) | NeMo Agent Toolkit (agent-workflow profiling) | TruLens feedback functions overlap Ragas metrics |
| Experiment tracking | MLflow | — | Phoenix experiment UI overlaps MLflow runs |
| Red-teaming | PyRIT (attack orchestration) | Promptfoo (declarative CI red-team suites) | — |

The `[ASSUMPTION]` markers matter here. The agent does not have your traffic
patterns; when it asserts "Phoenix is sufficient for span capture at your
volume," that will be tagged, and you should either confirm it or push back.

**Feasibility and Scope Definition** is where you cut. The sober scope is:
normalize everything into one result schema, own the CLI and the schema, and
treat all nine libraries as *plugins behind an adapter interface* — not as
framework dependencies you inherit.

**Gate.** Approve, or request changes.

```
Ideation complete. How would you like to proceed?
- Approve         -> Continue to Inception
- Request Changes -> Provide revision feedback
```

### Inception — requirements, architecture, units

**Requirements Analysis** (`aidlc-product-agent`) produces numbered, testable
requirements. For this platform the load-bearing ones are usually about the
*seams*, not the features:

- A single `EvalResult` schema every adapter emits, with a stable
  `(run_id, case_id, metric, score, span_ref)` shape
- Every score traceable to the trace span that produced it
- Adapters are optional at install time — no hard dependency on all nine
- Red-team findings enter the same schema, carrying a severity dimension
- Deterministic replay from a pinned dataset hash

**Architecture** (`aidlc-architect-agent`) is reviewed by the read-only
`aidlc-architecture-reviewer-agent`. The recurring design tension it should
surface: Phoenix, TruLens, and MLflow each want to *own* the trace and
experiment store. Resolving that — typically by making OpenTelemetry the
substrate, Phoenix a consumer rather than the system of record, and MLflow the
run ledger — is the highest-value decision of the whole workflow, and it lands
in the architecture artifact with its rationale intact.

**Unit Decomposition** splits the build into independently buildable units. A
natural cut:

| Unit | Contents |
| --- | --- |
| `core-schema` | `EvalResult`, run manifest, dataset hashing, adapter protocol |
| `adapters-scoring` | Ragas, DeepEval, MLflow-evaluate |
| `adapters-observability` | Phoenix, TruLens, NeMo Agent Toolkit |
| `adapters-redteam` | Promptfoo, PyRIT |
| `cli-and-report` | `aeval run`, `aeval redteam`, `aeval report`, CI exit codes |

**Compliance** (`aidlc-compliance-agent`) earns its place here rather than
being ceremony: PyRIT and Promptfoo generate adversarial prompts and store
model responses that may contain harmful content. Retention, access control,
and who is permitted to run a red-team suite are real requirements, and this is
where they get written down.

**Gate.** Approve, or request changes.

### Construction — build, per unit

**Code Generation** (`aidlc-developer-agent`, dispatched as a subagent) runs
per unit. Build `core-schema` first: every other unit depends on the adapter
protocol, and getting it wrong is expensive.

Copilot enforces a **plan-approval gate** before code generation. You will see
the plan; approve it before any file is written. This is a blocking hook, not a
convention — the tool call is denied if the gate has not been satisfied.

A realistic adapter seam looks like this, and is worth insisting on because it
is what keeps nine libraries from leaking into your core:

```python
class EvalAdapter(Protocol):
    name: str
    def available(self) -> bool: ...
    def run(self, cases: Sequence[EvalCase], ctx: RunContext) -> Iterable[EvalResult]: ...
```

`available()` is what makes the optional-dependency requirement real: an
adapter whose library is not installed reports unavailable and is skipped, and
the run still succeeds.

**Build and Test** (`aidlc-quality-agent`) runs the suite. Insist that adapter
tests use recorded fixtures rather than live model calls — otherwise your CI
cost scales with your metric count and your tests are flaky by construction.

**Gate** per unit.

### Operation

**Deployment Pipeline** (`aidlc-pipeline-deploy-agent`) wires the CI gate. For
an evaluation platform the pipeline *is* a product surface: the exit-code
contract for `aeval run --gate` is what other teams integrate against.

**Deployment Execution** ships it.

---

## Part 4 — What you end up with

```
aidlc/spaces/default/
  intents/260921-agent-eval-platform/
    ideation/       market-research.md, feasibility.md, scope-definition.md
    inception/      requirements.md, architecture.md, units.md, compliance.md
    construction/   per-unit plans, test strategy, build reports
    operation/      pipeline.md, deployment.md
    aidlc-state.md
    questions/      every question asked, with your [Answer]: tags
  memory/           org.md, team.md, project.md, phases/*.md
  audit/            append-only decision log
```

The audit log is the part that pays off months later. When someone asks *"why
does this use Phoenix for spans but MLflow for runs?"*, the answer is recorded
with the alternatives that were rejected and the assumptions that were flagged
at the time.

Useful commands during and after the run:

```text
/aidlc --status          # where am I
/aidlc --stage 3.5       # jump to a stage
/aidlc --depth minimal   # override depth
/aidlc --doctor          # re-validate the install
```

---

## Part 5 — What Copilot does differently

These are harness facts, not style choices:

- **One install, two surfaces.** The CLI and VS Code agent mode read the same
  tree and behave identically except where noted.
- **Questions are numbered prose, not pickers.** Picker answers return as tool
  results and do not fire the `UserPromptSubmit` event the human-presence guard
  requires. While a workflow has `Status: Running`, a matcher-free `PreToolUse`
  guard denies picker calls and tells the model to render numbered prose and end
  the turn. Your next chat message is what satisfies the guard. The questions
  file with `[Answer]:` tags stays the source of truth.
- **Hooks block natively.** The adapter (`.aidlc/hooks/aidlc-copilot-adapter.ts`,
  wired by `.github/hooks/aidlc.json`) converts a core-guard block into
  Copilot's `permissionDecision: deny`. Reviewer read-scope, the
  state-transition guard, and the plan-approval gate actually refuse tool calls.
  Live-verified on the CLI; on VS Code the same channels are documented but
  treat IDE enforcement as best-effort until verified.
- **No statusline.** Use `/aidlc --status` and the progress lines at gates.
- **No MCP servers ship.** Add your own via `copilot mcp add` or
  `.vscode/mcp.json` — note the two surfaces use different MCP config files.
  For this project you might add a Phoenix or MLflow MCP server, but nothing in
  AI-DLC requires it.
- **Construction swarm is subagent fan-out only.** `AIDLC_USE_SWARM=1` is a
  loud no-op.
- **No `SessionEnd` event on VS Code.** The adapter reconciles the prior session
  at the next `SessionStart`.

---

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| Nothing is written to `aidlc/`, no gates appear | Folder not trusted — hooks are no-opping | Run `copilot` interactively and accept the prompt; re-run `/aidlc --doctor` |
| Hooks work interactively but not in `copilot -p` | Headless mode needs the opt-in | `export GITHUB_COPILOT_PROMPT_MODE_REPO_HOOKS=1` |
| Copilot shows a picker instead of numbered options | Host below the version floor, or no running workflow | Upgrade to CLI >= 1.0.74 / VS Code >= 1.130 |
| `/aidlc` not recognized | Skills not discovered | Confirm `.github/skills/aidlc/` exists and the project root is the workspace root |
| Agent invents libraries or versions | Missing grounding | Record the pinned set in `aidlc/spaces/default/memory/project.md` and re-run the stage |

Full reference: [AI-DLC on GitHub Copilot](harnesses/copilot.md). For the
general flow on any harness, see [Your First Workflow](02-your-first-workflow.md)
and [Worked Examples](16-worked-examples.md).
