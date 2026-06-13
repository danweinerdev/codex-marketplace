---
name: "sdd-planner:code-review"
description: "Review implementation code against the plan for drift, gaps, and blind spots. Trigger with /sdd-planner:code-review, or use natural language such as review code, check implementation, compare to plan, or code vs plan."
---

# /sdd-planner:code-review — Implementation Code Review

## ⚠️ CONTRACT — READ FIRST

This skill's entire value comes from dispatching four intent-isolated Codex lane agents in parallel and synthesizing their independent reports. You are the orchestrator.

**You MUST:**
1. Dispatch all four built-in lane agents using Codex sub-agents. Attach or quote the matching lane instruction file for each spawned agent: `agents/drift-detector.md`, `agents/quality-scanner.md`, `agents/spec-compliance.md`, and `agents/blind-spot-finder.md`.
2. Dispatch them **in parallel** — spawn all four built-in lanes before waiting for results, plus one additional spawned agent for each matching project review lane (see Step 2e and Step 3).
3. Give each lane agent only the inputs for its lane. See Step 3 below for the exact input map. Passing extra context destroys the intent isolation that makes the review worthwhile.
4. Wait for all dispatched lanes (the four built-ins plus any project lanes) to return, then synthesize.

**You MUST NOT:**
1. Read the full diff, the phase doc contents, spec contents, or design contents in the primary context and write findings yourself. That is a single-pass review cosplaying as a four-lane review. It is the bug this contract exists to prevent.
2. Skip lane dispatch because "you already know the answer" after loading context. The answer you'd produce is exactly the single-pass review this skill exists to replace.
3. Fall back to self-synthesis if a built-in lane cannot be spawned or cannot run. If a built-in lane fails, **STOP** and return a loud error to the user describing which lane failed and why. Do not silently continue.
4. Let a lane read outside its input bundle. Built-in lanes are internal agent instruction files, not user-triggerable commands; each spawned sub-agent is just a Codex agent running the lane instructions you provide.

**Project review lanes (best-effort, additive).** Beyond the four, a project can plug in its own specialized lanes via the socket defined in `shared/review-lanes.md` (for example a read-only `.codex/sdd-review-lanes/sql-reviewer.md` carrying `reviewLane: true`). These are **additive, read-only, and never abort the four**:
- The four built-ins are the floor and always run. A project lane only ever *adds* findings — never removes, replaces, or weakens a built-in lane's inputs or dispatch.
- A project lane that fails to dispatch, errors, or is malformed is **recorded and dropped** — never aborts the four. The strict "STOP on dispatch failure" rule in this contract applies to the **four built-ins only**. Synthesis proceeds over the four plus whatever project lanes returned. (Lane *responsiveness* is the project's responsibility — the socket dispatches and waits; it imposes no timeout, so a lane that hangs will hold up the review.)
- Failure is **not silent**: a declared lane that didn't run **degrades the verdict headline**, and an opt-in `required: true` lane that didn't run **forces the verdict to BLOCKED** (it gates the verdict, not the floor). The verdict is computed from the four plus any *successful* project lanes.
- Lanes execute repo-supplied instructions with session tool access, so `/sdd-planner:code-review` **confirms discovered lanes** with the user before dispatch when the target repo isn't the session's own project.

If you find yourself reading a spec file or running `git diff` against the full patch in the primary context, stop. That work belongs in the sub-agents, not here. (Listing changed file *paths* with `--name-only` for `appliesTo` matching is allowed — that's paths, not content.)

## Path Resolution
**Artifacts** (Plans/, Research/, Specs/, etc.) are read from and written to the **planning root**.
Read `planning-config.json` (at repo root) to find the planning root:
- `planningRoot` of `"."` or absent → artifacts at repository root
- `planningRoot` of `"<dir>"` → artifacts under `<dir>/` from repo root
- `planningRoot` of `"/absolute/path"` → artifacts in an external directory

**Shared specs, templates, and built-in lane agents** (`shared/` and `agents/`) are read from the **plugin directory**, not from the planning root. The plugin directory contains `.codex-plugin/`, `skills/`, `agents/`, and `shared/` as siblings. In a local checkout, use this repository root. In an installed plugin, find the plugin directory by locating `skills/code-review/SKILL.md` under the Codex plugin cache or the active skill path, then strip `skills/code-review/SKILL.md` from the matched path.

## When to Use
During or after implementation of a plan phase, when you want to verify that the actual code matches what was planned. This skill dispatches four intent-isolated sub-reviewers against the diff and synthesizes their reports into a single review.

Use this:
- Mid-phase: to course-correct before implementation drifts further
- End-of-phase: before running `/debrief`, to have a clear picture of what happened vs. what was planned
- After resuming work: to re-orient on where things stand
- Before merging: to ensure the branch delivers what the plan promised

## Why four sub-agents

A single reviewer juggling plan, specs, designs, code, and adversarial perspective inevitably forgives code for one reason while missing flaws visible from another angle. Four intent-restricted reviewers preserve the perspective each lane needs:

| Agent | Sees | Doesn't see | Role |
|---|---|---|---|
| `drift-detector` (`agents/drift-detector.md`) | Diff + plan + phase doc + prior debriefs | Specs, designs, code-quality heuristics | Missing work, scope creep, approach drift |
| `quality-scanner` (`agents/quality-scanner.md`) | Diff + code | Plan, specs, designs | Correctness, safety, maintainability (including comment quality: WHAT-restating, PR-time-context, and tombstone comments are flagged as noise), testing, over-engineering, intent-blind |
| `spec-compliance` (`agents/spec-compliance.md`) | Diff + specs + designs | Plan, phase doc | Requirements coverage, contract violations |
| `blind-spot-finder` (`agents/blind-spot-finder.md`) | Diff only | Everything else | Adversarial fresh eyes, scenarios the author did not consider |

Each runs in its own fresh context. The whole point is that one reviewer's framing cannot contaminate another's.

Projects can add their own specialized lanes on top of these four via the socket in `shared/review-lanes.md` (drop a `.codex/sdd-review-lanes/*-reviewer.md` with `reviewLane: true`). Project lanes are **additive**: they never replace or weaken the built-in four, and their failure never fails the review.

**`blind-spot-finder`'s diff-only guarantee is the sharpest and most fragile of these.** If you, as the primary orchestrator, read the plan before you dispatch it, your synthesis of its output will inevitably carry plan-aware framing back into the "blind-spot" conclusions. Keep your own reads shallow — file paths only, not contents — until after the sub-agents return.

## Process

### 1. Identify the Review Target

Your goal in this step is to produce four strings: plan path, phase doc path, target repo path, and diff scope. Do NOT read the contents of the plan or phase doc — just their paths.

- **Plan and phase**: ask the user, or infer from context. If inferring, use `Glob` on `Plans/*/README.md` (filenames only — do **not** `Read` them) and filter to those whose frontmatter `status` is `active` (read only the frontmatter, not the body). If there is a single active plan, pick it; if there are multiple, ask. Capture:
  - Plan path (e.g., `Plans/<PlanName>/README.md`)
  - Phase doc path (e.g., `Plans/<PlanName>/<NN>-<Phase-Name>.md`)
- **Target repo path**: read `planning-config.json` for `planMapping` to find the repo key for this plan, then read `planning-config.local.json` for `repositories.<key>.path` to get the absolute local path. If that doesn't resolve, ask the user where the code lives.
- **Diff scope**: if the user specified a branch, commit range, or "working + staged", record that verbatim. Otherwise, record "determine from phase created date" — the sub-agents can resolve it.

### 2. Gather Only What Dispatch Needs

To dispatch the sub-agents with the right inputs, you need a little bit of information from the planning artifacts and the target repo. Keep this step as narrow as possible — read frontmatter, not bodies.

**a. Plan README frontmatter only.** Use `Read` on the plan README, but stop after the frontmatter. Extract the `related` field to get the list of spec paths (under `Specs/`) and design paths (under `Designs/`). **Do not** read the body of the README — the sub-agents will do that.

**b. Prior debrief paths.** Use `Glob` on `Plans/<PlanName>/notes/*.md` to get the list of prior debrief paths. Do not read their contents — `drift-detector` will do that.

**c. Resolve the diff scope.** First, detect the target repo's VCS using `shared/vcs-detection.md`. Then orient using the VCS-appropriate "working-tree status" and "recent history" commands from that file (e.g., `git status` + `git log --oneline -20` for git, `p4 opened` + `p4 changes -m 20` for perforce). If the user gave an explicit range, use it. If the user said "determine from phase created date", read only the frontmatter of the phase doc to get the `created` date, then find the earliest commit/changelist on or after that date. If you still can't resolve (or the VCS is `none`, in which case there is no history), ask the user for a base. Capture the scope as a concrete diff command (`git diff <base>..<head>` for git, `p4 diff2 -dw //path/...@<base> //path/...@<head>` for perforce) plus any unsubmitted/working coverage the user requested. **Do not** run the diff and read the patch content — the sub-agents will. Pass the detected VCS and the resolved diff command to each sub-agent so they don't have to re-detect.

**d. Language-verification note (optional).** If `shared/language-verification.md` exists and the project language warrants structural checks, pass a one-line note to `drift-detector` and `quality-scanner` so they can flag missing sanitizers/static-analysis/type-checking work. You do not need to read the full language-verification doc — just detect the project language from a quick file-extension glance at the target repo and include it as context.

**e. Discover project review lanes (optional socket).** Read `shared/review-lanes.md` first; it is the full convention. Glob, **non-recursively**, the **target repo's** top-level `.codex/sdd-review-lanes/*-reviewer.md` and the user's `~/.codex/sdd-review-lanes/*-reviewer.md`, de-duplicating by frontmatter `name`. From each match read **only the socket fields**: `name`, `reviewLane`, `lane`, `appliesTo`, and `required`. Do **not** read the `description` or body until after the lane passes validation, matching, and trust gating. Keep those whose `reviewLane` is boolean `true`; the filename is only a discovery hint. For each kept reviewer:
   - **Validate** — drop as **Malformed** (report later, do not dispatch): `reviewLane` present but not boolean `true`; `lane` of an invalid type; a bare `name` equal to a built-in (`drift-detector`/`quality-scanner`/`spec-compliance`/`blind-spot-finder`); or two discovered files resolving to the same bare `name`.
   - **Dispatch gate.** If it declares `appliesTo` (a list of minimatch/globstar globs, repo-root-relative, case-sensitive), get the changed file *paths* via the VCS-appropriate name-only listing run with **cwd = target repo** (`git -C <target> diff --name-only <base>..<head>`, or `p4` + `p4 where` for depth→relative paths — see `shared/review-lanes.md`). Paths only, never hunk content. Keep the lane only if some glob matches; otherwise mark it **Skipped** and remember the tested paths. `appliesTo: []` always Skips; absent `appliesTo` always dispatches and self-scopes.
   - **Input bundle.** Match `lane` exact-lowercase. Recognized values (`code`/`spec`/`plan`/`diff-only`) get the **same** curated bundle as the matching built-in. An absent or unrecognized `lane` gets base inputs only (target repo path, VCS label, diff command) and self-gathers the rest. Group reviewers sharing the same unrecognized `lane`. Record any `required: true`.
   - **Trust gate.** If the target repo is **not** the session's own project, these are externally-supplied instructions executing with session tool access: **list the discovered lanes and ask the user to confirm** before dispatching. Even when it is the session project, name the discovered lanes to the user. After the trust gate passes, read the full lane file and use it as the spawned agent's instruction bundle.

   If there are no project reviewers, this step is a no-op and the review runs the four built-ins exactly as before.

At the end of Step 2, you should have:
- Plan path, phase doc path
- Spec paths (list), design paths (list)
- Prior debrief paths (list)
- Target repo path
- Resolved diff scope as a concrete git command/range
- Optional language-verification note
- Project review lanes to dispatch (list) — each with its resolved input bundle, lane grouping, and `required` flag
- Project lanes Skipped or Malformed at discovery (with reason + the paths a Skip was tested against), to report in Step 5
- Changed file paths (name-only listing, target-repo-relative) — only if some project lane declared `appliesTo`

You should NOT have read any spec contents, design contents, diff hunks, the body of the phase doc, or any project reviewer's `description`/body. (Changed file *paths* from a `--name-only` listing are fine — they are not hunk content.)

### 3. Dispatch All Lanes in Parallel

**This is the step the contract at the top of this file is about.** Spawn all four built-in Codex lane agents in parallel, plus one spawned agent per matching project lane from Step 2e. Each spawned agent must receive the lane instruction file plus only the inputs for its lane. Do not wait for one built-in lane before spawning the next.

**Spawned lane 1 - `drift-detector` (`agents/drift-detector.md`)**
- Plan path, phase doc path
- Prior debrief paths
- Target repo path
- Detected VCS label (`git`, `git-worktree`, `perforce`, `none`)
- Resolved diff command for that VCS
- Language-verification note (if applicable)
- ❌ Do NOT pass spec paths, design paths, or any diff content.

**Spawned lane 2 - `quality-scanner` (`agents/quality-scanner.md`)**
- Target repo path
- Detected VCS label
- Resolved diff command
- `mode: review`
- Language-verification note (if applicable)
- ❌ Do NOT pass plan path, phase path, spec paths, or design paths. Intent-blindness is the point.

**Spawned lane 3 - `spec-compliance` (`agents/spec-compliance.md`)**
- Spec paths, design paths
- Target repo path
- Detected VCS label
- Resolved diff command
- ❌ Do NOT pass plan path or phase path.

**Spawned lane 4 - `blind-spot-finder` (`agents/blind-spot-finder.md`)**
- Target repo path
- Detected VCS label
- Resolved diff command
- ❌ Do NOT pass anything else. Not the plan, not the specs, not the designs, not the phase doc, not even the language-verification note. The diff-only guarantee is this reviewer's entire contribution.

**Spawned lanes 5..N - project review lanes (if any, from Step 2e).** Spawn one additional Codex agent per matching project lane. For each:
- The spawned agent's instruction bundle is the full project lane file body after validation and trust gating.
- Pass the input bundle its `lane` resolved to: a recognized `lane` (`code`/`spec`/`plan`/`diff-only`) gets the **same** inputs as the matching built-in above; an absent or unrecognized `lane` gets only the base inputs (target repo path, detected VCS label, resolved diff command) and is left to gather anything else itself.
- ❌ Do NOT hand a standalone or unrecognized lane the plan, specs, or designs — it gets base inputs only; if it wants intent, it reads it itself. ✅ Do hand a recognized-lane reviewer exactly the bundle that lane names, and nothing more.
- Hand every lane the **frozen** diff reference (fixed base/head — a commit SHA where the VCS allows) so a lane can't shift what the built-ins review. Project lanes are **read-only**; they must not write to the repo.

Wait for all dispatched lanes to return before continuing. The socket imposes no timeout on project lanes — keeping them fast is the project's responsibility (a hung lane will hold up the review).

**If a built-in lane spawn or result (one of the four above) fails**, stop immediately and return a loud error to the user:

```
ERROR: /sdd-planner:code-review could not run built-in lane `<name>`.
Reason: <the exact spawn or lane error>
The four-lane review cannot proceed. Fix the dispatch issue and re-run.
```

**Do NOT** fall back to self-synthesis. A single-pass review pretending to be a four-lane review is worse than no review at all — it gives the user false confidence in an un-triangulated report.

**Project-lane failures are different: never abort the four.** If a project lane fails, do **not** stop. Record its outcome and continue synthesizing the rest. Classify each per `shared/review-lanes.md`: **Skipped** (no `appliesTo` match; show the tested paths), **Failed to dispatch**, **Errored**, **Oversized** (truncate with a note), or **Malformed**. The four built-ins remain the floor; a broken project lane degrades coverage, it does not fail the four, but the degradation must surface in the verdict (Step 4h/Step 5), never only in a sub-section.

### 4. Synthesize the Reports

Once all dispatched lanes have returned, produce a single unified review over **all** their reports — the four built-ins plus any project lanes that returned. Synthesis is the whole value-add of this step — it is not concatenation.

**a. Build a findings table.** Enumerate every finding from all returned reports (built-in and project lanes). For each, record: source agent, severity, location, one-line summary.

**b. Detect agreements.** Findings that multiple reviewers hit independently are high-confidence. Flag them as **Confirmed by N reviewers**. When `drift-detector` says a task is missing and `spec-compliance` says the requirement is uncovered, that's the same hole seen from two angles — strong signal.

**c. Detect disagreements.** When reviewers contradict each other, the disagreement is itself a finding:
- `drift-detector` says the task was completed; `spec-compliance` says the requirement is uncovered. → Plan and spec are out of sync. Surface this.
- `quality-scanner` says the code is fine; `blind-spot-finder` flags a concurrency scenario. → The code is locally correct but globally fragile. Surface both perspectives.
- `drift-detector` says the code is unplanned scope creep; `spec-compliance` says it satisfies a spec requirement. → The plan missed a requirement. Surface as a planning gap.

Disagreements get their own section in the output. Never quietly reconcile them by picking a winner.

**d. Highlight unique blind-spot findings.** Any `blind-spot-finder` finding that **no other reviewer caught** gets promoted into a dedicated "Blind Spots Only `blind-spot-finder` Caught" section. This is explicit recognition of the value the adversarial reviewer adds. If `blind-spot-finder` had zero unique findings, say so — it's a signal the other reviewers were thorough, not a reason to omit the section.

**e. Cross-validate questionable findings.** If a reviewer flagged something as a **Question** (unverified suspicion), check whether any other reviewer's findings corroborate or rule it out. Promote corroborated questions to findings; drop the ones that other reviewers ruled out.

**f. Deduplicate.** When multiple reviewers report the same issue from the same angle, collapse them into one entry and list the sources. Don't double-count in severity tallies.

**g. Fold in project lanes.** Treat each project lane's findings like any built-in lane's for agreement, disagreement, and dedup, with three rules:
- **Within-lane grouping.** Reviewers sharing the same unrecognized `lane` are peers in one lane — present them under one heading, and for the "Confirmed by N reviewers" tally **count the lane group as a single source** (two `security` peers agreeing is one corroborated Security finding, not "confirmed by 2").
- **Identity by dispatch, not by string.** Key the "Blind Spots Only `blind-spot-finder` Caught" section and the per-lane sections on the **dispatched identity** (the namespaced built-in, or the project lane's resolved name), never a bare matched string. A project lane cannot be mistaken for a built-in — a name collision was already rejected as Malformed in Step 2e.
- **Recognized lanes stay in their bundle.** If a recognized-lane reviewer (`code`/`spec`/`plan`/`diff-only`) asserts a finding about an artifact it was not granted (e.g. a `spec` lane claiming "the plan requires X"), demote that claim to a **Question** — it spoke outside its inputs.

**h. Compute the verdict, then degrade it visibly.** The Overall Verdict and counts come from the four built-ins plus any *successfully returned* project lanes — a lane that failed to run is a coverage gap, not a finding, and never drags the verdict down on its own. But the gap must not be silent:
- If any **declared** project lane did not run (Failed to dispatch / Errored / Oversized-to-unusable / Malformed — **not** Skipped), append `(DEGRADED — N declared lane(s) did not run: <names>)` to the verdict line.
- If a **`required`** lane did not run, force the verdict to `BLOCKED — required lane <name> did not run`, overriding an otherwise-passing headline. The four still ran and their findings still stand; BLOCKED reflects that a gate the project declared mandatory was not satisfied.
- A purely **Skipped** lane (no matching changed paths) degrades nothing — it had nothing to review.

**Do NOT introduce new findings of your own during synthesis.** Your job is to synthesize what the four sub-agents returned, not to add findings based on your own reading. If you notice something none of the four agents caught, it means one of the agents needs to be improved — note it in an "Orchestrator Observations" addendum rather than silently inserting it as a finding.

### 5. Present the Unified Review to the User

Render the synthesis using `shared/templates/code-review.md`. Include the raw sub-reports verbatim in `<details>` blocks so the user can drill in. Do not re-summarize the sub-reports: the synthesis is the summary, the raw reports are the evidence.

### 6. Write the Review Artifact

Every code review creates a Markdown artifact. This is mandatory, even if there are no findings.

**Location:** write the review under the target repo root:

```text
reviews/NN-<slug>-review.md
```

Where:
- `reviews/` is relative to the target repo path unless the user explicitly asks for a different output root.
- `NN` is the next two-digit counter. Scan existing files matching `reviews/[0-9][0-9]-*-review.md`, take the highest prefix, and add 1. If none exist, start at `01`.
- `<slug>` is a short kebab-case label derived from the plan, phase, branch, or review subject. Keep it stable and descriptive, for example `01-auth-refresh-review.md` or `02-auth-refresh-fix-review.md`.
- Never overwrite an existing review file. If the chosen path exists, increment the counter or refine the slug.

The artifact must contain the full unified review output, including raw lane reports in `<details>` blocks. After writing it, report the path to the user and give only a concise summary in chat.

Use `shared/templates/code-review.md` as the source of truth for section order and placeholder semantics. Omit optional sections only where the template says they may be omitted. Fill empty required sections with `None.` rather than deleting them.

### 7. Resolution Tracking for Fix Iterations

When the user asks to fix findings from a review, or a later review is clearly checking whether earlier findings were fixed, update the relevant earlier review file(s) with a resolution section at the bottom. Do not rewrite or delete the original findings.

Append this section to each reviewed file that is being resolved:

```markdown
---

## Resolution
**Status:** Pending | Partially fixed | Fixed | Superseded | Won't fix
**Resolved in:** [commit, branch, working tree, or new review artifact path]
**Verified by:** [commands run and/or review artifact path]

### Finding Resolutions
- **[original finding heading or stable summary]** — Fixed | Partial | Still open | Superseded | Won't fix
  - Evidence: [file paths, tests, commits, or follow-up review section]
  - Notes: [short explanation]
```

Rules:
- If a review file already has a `## Resolution` section, append a new dated `### Resolution Update - YYYY-MM-DD` subsection instead of replacing prior resolution history.
- Run the four lane review first, then compare the new lane reports against prior review findings. Do not pass prior review files into `blind-spot-finder`; they are post-dispatch verification context only.
- The new review artifact should include a `### Prior Review Verification` section when it is checking fixes, listing which earlier review files were examined and which findings are Fixed, Partial, Still open, Superseded, or Won't fix.
- If the agent fixes code before re-reviewing, it must update the prior review file's resolution section after verification, not before.

## Review Artifact Template

Use `shared/templates/code-review.md` for the review findings document. The generated file must be self-contained: metadata, verdict, synthesized findings, prior-review verification when applicable, raw lane reports, and a bottom `## Resolution` section. Do not invent a parallel output format in this skill.

### 8. Offer Next Steps
Based on findings, suggest appropriate actions:

**If alignment is strong:**
- Proceed to `/debrief` to capture the phase outcome
- Note any minor items for the debrief

**If drift is detected:**
- Fix the code to match the plan, OR
- Update the plan to reflect intentional changes (scope was wrong)
- Document the deviation rationale

**If planning blind spots are found:**
- Update specs/designs to account for discovered complexity
- Add tasks to the current or a future phase
- Create a research document for unknowns that need investigation

**If assumptions need checking:**
- Present the assumption checklist for the user to verify
- Flag any that could cause issues if wrong

Never use "pre-existing" to justify deferring or hiding a finding. "Pre-existing" describes origin, not impact. Present findings by what they do to the user, not when they were introduced. The user decides what is worth fixing.

Never downscope a finding, recommendation, or fix by estimating how long it would take a human. Agents are not constrained by human development timelines. The right fix is right; surface it. Prefer a smaller change only when it is genuinely better on its own merits — clearer, lower risk, smaller surface area — never because a larger one would "take too long." The user decides what is worth fixing; don't pre-decide for them on time grounds.

## Output
This skill always writes a review artifact under `reviews/NN-<slug>-review.md` in the target repo. Chat output should be concise: path written, verdict, and top findings. If the review verifies fixes from earlier review files, append or update `## Resolution` sections at the bottom of those earlier files after verification.

## What This Is NOT
- Not a general code review (style, formatting, best practices) — focuses specifically on plan alignment
- Not `/poke-holes` (which analyzes planning artifacts, not code)
- Not `/simplify` (which improves code clarity, not plan compliance)
- Not a substitute for tests — assumes the test suite validates correctness independently

## Context
- Orchestration: `shared/orchestration.md`
- Project review-lane socket: `shared/review-lanes.md` (template: `shared/templates/custom-reviewer.md`)
- Schema: `shared/frontmatter-schema.md`
- Target plan: `Plans/<PlanName>/` (status: `active`)
- Related specs: `Specs/`
- Related designs: `Designs/`
- Prior debriefs: `Plans/<PlanName>/notes/`
- Local repo paths: `planning-config.local.json`
- Lane agents spawned from the primary context: `agents/drift-detector.md`, `agents/quality-scanner.md`, `agents/spec-compliance.md`, `agents/blind-spot-finder.md`
