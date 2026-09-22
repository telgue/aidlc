# AI-DLC — Roadmap

Status as of 2026-09-22.

- The current version is **1.0.0**, published from this repository.
- Version numbering restarts at 1.0.0 for this distribution. Releases here are
  independent, and `aidlc update` resolves against this repository only.
- The methodology, stages, agents, sensors, and harness surfaces are carried
  forward intact. This distribution's own work is provider coverage,
  distribution, and documentation.

Items below describe what ships today and where effort is directed next.
Directional themes are intent, not committed release promises.

## What this distribution changes

Three lines of work distinguish this distribution:

- **Multi-cloud model providers.** Amazon Bedrock, Azure AI Foundry, and Google
  Vertex AI are all first-class in `aidlc config providers` and in the first-run
  setup wizard.
- **Independent distribution.** Release discovery, installers, update checks,
  and documentation all resolve against this repository.
- **Executed documentation.** Guides are verified by running them, not by
  reasoning about them. Steps that fail in practice are corrected in the guide.

## North star reference

The seven functional goals the framework is built against:

1. **Mimic what we practice in the real world** — a stage executed by a
   configurable ensemble (Owner, Collaborator, Verifier) with consistent
   semantics across harnesses.
2. **Customization of behaviour** — encode new behaviours, policies, or
   constraints in no more than two targeted changes, reusable across harnesses
   without tool-specific rewrites.
3. **Adaptiveness of workflows** — scale in (report triage to compact Fix, Test,
   PR) and scale out (decide next stages at boundaries); composition is not
   hard-wired.
4. **Verifier as a true adversary** — an adversarial quality gate that may use a
   different model than the producer, validates against machine-checkable
   evidence, and runs a budgeted self-heal loop escalating to a human.
5. **Support for cyclic, directional flows** — forward progression plus
   governed, directional feedback loops.
6. **Preserve artefact traceability** — downstream stages enrich upstream
   artefacts rather than spawning disconnected ones.
7. **Organizational, not project-local, artefact repository** — a shared
   knowledge layer across projects, intents, and repos.

## Goal status

<!-- markdownlint-disable MD013 -->

| # | Goal | Status | Shipping today | Remaining work |
| --- | --- | --- | --- | --- |
| 1 | Real-world ensemble | Shipped | Independent collaborators, selectable topologies, enforced reviewer receipts, batch-parallel per-unit waves, team-owned parallel Units | Harness-native live-team transports remain an enhancement |
| 2 | Customization | Shipped, with follow-ups | Plugin seam, content projection and selection, deterministic rule delivery, plugin scopes, reusable plugin test kit, plugin doctor extensions, standalone authoring toolchain | Stage-specific rules, `when:` evaluation, remote discovery and a marketplace |
| 3 | Adaptiveness | Shipped | Composer, entropy-scored composition, deterministic adaptive routing, unit-major Code Generation, Classic/Express scopes, per-session workflow bindings | Boundary changes remain human-approved by design |
| 4 | Verifier as adversary | Shipped | Adversarial evidence contract, gate-and-completion enforcement, reviewer-class cost dial, turn and recovery backstops, gate-bound blocking sensors | Pull-request-level adversarial review is not implemented |
| 5 | Cyclic flows | Partial | Within-stage review/revision loops, bounded recovery, human-authorized forward/backward/redo stage jumps, bounded Build & Test to Code Generation loop-back | General governed cross-stage feedback loops remain unbuilt |
| 6 | Traceability | Partial | Artefact graph, upstream coverage, per-stage enforcement, claim provenance, shared CodeKB safeguards, domain/contract boundaries, stale-result propagation, per-Unit review receipts, commit provenance via `aidlc attest resolve` | Progressive in-place enrichment and cross-unit discovery propagation |
| 7 | Org repository | Shipped | Spaces, intents, org knowledge base, declared multi-repo manifest and sync, DocumentKB indexing, citations, summaries and tags | Auditable supplemental-knowledge selection |

<!-- markdownlint-enable MD013 -->

## Delivered in 1.0.0

- **Amazon Bedrock, Azure AI Foundry, and Google Vertex AI** as selectable model
  providers. `aidlc config providers --provider` accepts `amazon-bedrock`,
  `azure-ai-foundry`, `gcp-vertex-ai`, and `other`. Azure requires
  `--resource <name>`; Vertex requires `--project <id>` and
  `--location <region>`; Bedrock continues to use `--region` and `--profile`.
- **Full engine rewrites on provider switch**, so a previously selected engine
  cannot persist through a stale `CLAUDE_CODE_USE_*` variable. Model aliases are
  repinned to the selected engine's identifiers.
- **Codex Azure OpenAI provider block** generated and selectable.
- **Independent release channel**: installers, binaries, release discovery, and
  update checks all resolve against this repository.
- **GitHub Copilot agent-evaluation walkthrough**
  (`guide/copilot-agent-evaluation-walkthrough.md`), an end-to-end worked
  example carried through all five phases and corrected against executed runs.

## In flight

- **Debranding and distribution hygiene.** Removing residual upstream pointers
  from documentation, fixtures, and scan baselines.
- **Copilot harness verification.** Continuing to execute the documented flows
  on both Copilot CLI and VS Code agent mode and correcting what diverges.

## Directional themes

These are supported by observed gaps in this tree. None has a committed release
version.

### Harness parity for providers

- Google Vertex AI is not wired for Codex, because Codex exposes no OpenAI
  Responses endpoint for it. The provider walk reports this as an outstanding
  action rather than silently writing a broken configuration. Closing it needs
  either an adapter or a suitable endpoint.
- Kiro writes no provider file for Azure or Vertex: its MCP registry holds AWS
  tooling servers, and its chat model is selected in the IDE model picker.
  Bringing Kiro into the same provider contract is unresolved.

### Copilot harness gaps

- **`rebuild-stage-graph` does not fire on this harness.** `aidlc doctor`
  reports no compiled runtime graph and names the hook. Other hooks
  (`session-start`, `continue-workflow`, `review-freeze`, `reviewer-scope`,
  `plan-approval-guard`) do fire and are observable in doctor output. The
  workflow still proceeds, but the hook path needs a fix.
- **Headless runs bypass approval gates.** `copilot -p --allow-all-tools` will
  carry an intent from Inception into Construction and write source files with
  no human confirmation at any gate. The gates are real in interactive mode; the
  headless flag defeats them. This needs either a refusal or a documented,
  deliberate opt-in.

### Governed feedback loops

- One bounded Build & Test to Code Generation return path ships. It is an
  incremental loop, not a general cyclic graph engine.
- General cross-stage backward edges still need engine-level governance, stale
  artefact handling, and explicit human authorization.

### Traceability and progressive enrichment

- Commit-level provenance ships as content-derived attribution: reviewed-source
  evidence is committed into the intent record, and `aidlc attest resolve` maps
  any commit or diff range back to its owning units, intents, and drift status.
  No hooks, trailers, or session state are required. See
  [Commit Provenance](reference/20-commit-provenance.md).
- **One byte-form gap stays open**, reported in `resolve`'s `warnings[]`: review
  evidence hashes working-tree bytes while commit listings read repository
  blobs, so Git LFS, `core.autocrlf`, working-tree encodings, and submodule
  gitlinks can report unchanged content as `drifted`. Reconciling them changes
  what the `Unit Source Fingerprint` is computed over, so it needs its own
  change with a migration story for existing receipts.
- Richer trust roots remain future work: per-approval signatures and an identity
  policy for who may approve. Today's `signed` level checks git's commit-level
  `%G?` on whoever last wrote each authority-bearing file, not a reviewer
  identity.
- Progressive in-place enrichment — downstream stages enriching upstream
  artefacts rather than spawning new ones — remains the North Star destination.
- Cross-unit discovery propagation remains unbuilt.

### Plugins

- The plugin mechanism, content projection, selection, plugin-contributed
  scopes, the plugin test kit, and plugin-extensible doctor checks all ship.
- The offline CREATE, VALIDATE, BUILD, and TEST authoring tiers ship as
  `aidlc-plugin-create.ts`, `aidlc-plugin-validate.ts`, `aidlc-plugin-build.ts`,
  and `aidlc-plugin-test.ts`. Top-level `plugin validate` and `plugin build`
  routes ship; top-level `plugin create` and `plugin test` routes do not.
- Remote discovery, plugin trust, and a marketplace are unbuilt.

### Knowledge and documents

- DocumentKB indexing, citation delivery, summaries, and tags ship.
- Intent-aware discovery and auditable supplemental-knowledge delivery across
  stage topologies remain open.

### Evaluation and operations

- Repeatable benchmarks for measuring AI-DLC outcomes are not implemented.
- Operations-phase steering remains a direction rather than an active
  implementation stream.

## Known gaps

- Stage-specific rules (`aidlc-stage-<slug>.md`) are reserved but unbuilt.
- Plugin `when:` evaluation, remote discovery, and marketplace trust are open.
- Write-fired sensors remain advisory; gate-bound sensors support blocking
  severity and human-backed override.
- General cross-stage cycles and progressive in-place artefact enrichment remain
  North Star gaps.
- `rebuild-stage-graph` does not fire on the Copilot harness.
- Headless Copilot runs with `--allow-all-tools` bypass every approval gate.
- Google Vertex AI is unsupported on Codex; Azure and Vertex write no provider
  file on Kiro.
- `t298` (knowledge transaction) has a hardlink time-of-check/time-of-use race
  that can fail under high test concurrency. It passes on re-run and in
  isolation, which makes it a CI flake rather than a defect in shipped
  behaviour.
