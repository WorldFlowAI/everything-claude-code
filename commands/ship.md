---
description: End-to-end feature delivery. Spec → explore → plan (user approves) → implement test-first → parallel code + security review → fix loop → E2E → verify → docs + CI/infra → commit/PR (user approves). Use for any feature or bug fix you want taken from idea to a reviewable PR.
argument-hint: "<feature or bug description> [--no-e2e] [--no-pr] [--resume <slug>]"
---

# Ship Command

Takes a feature from a one-line idea to a verified, reviewed, committed change, using the toolkit's agents in sequence. You (the main session) are the **orchestrator**: you run the stages, pass handoffs between agents, enforce the gates, and keep the user informed. You do not write production code yourself; the `implementer` does.

## Arguments

`$ARGUMENTS`:
- `<description>`: what to build or fix (required unless `--resume`)
- `--no-e2e`: skip Stage 7 even if UI flows changed
- `--no-pr`: stop after the local commit (no push, no PR)
- `--resume <slug>`: continue a previous run from its state file

## State File

Keep run state in `.claude/ship/<slug>/state.json` (slug = short kebab-case from the description) so a run survives compaction or a new session:

```json
{
  "slug": "add-rate-limiting",
  "task": "Add per-user rate limiting to the public API",
  "branch": "feat/add-rate-limiting",
  "stage": "implement",
  "stack": ["rust"],
  "spec": "...", "explorerHandoff": "...", "plan": "...",
  "implementerHandoff": "...", "reviewFindings": [], "fixLoops": 0,
  "verification": null
}
```

Update it at the end of every stage. Make sure `.claude/ship/` is in `.gitignore` (add it if missing). Handoffs live in this JSON file, not separate markdown files, because the plugin's hooks block stray `.md` files.

## Pipeline

### Stage 0: Preflight
- `git status`: if the tree is dirty with unrelated changes, stop and ask the user to commit or stash.
- Create a branch from the current default branch: `feat/<slug>` or `fix/<slug>`. Never work on `main`/`master`.
- Detect the stack (`Cargo.toml`, `package.json`, `pyproject.toml`, ...) and record it.

### Stage 1: Spec (you, inline)
Turn the description into a short spec:
- **Goal**: one sentence
- **User stories / behaviors**: what changes for whom
- **Acceptance criteria**: concrete, testable statements (these become tests)
- **Out of scope**
- **Open questions**

If an open question changes what gets built (e.g. "per-user or per-IP?"), ask the user now with concrete options. Don't guess on product decisions. Implementation details are not questions; the planner decides those.

### Stage 2: Explore → `explorer` agent
Send the spec. Receive `HANDOFF: explorer -> planner` (relevant files, conventions, current behavior, root cause for bugs). For tiny changes to a single, well-understood file, you may skip this stage and say so.

### Stage 3: Design + Plan → specialists, then `planner`

**3a. Design specialists (conditional, in parallel).** Using the spec and explorer handoff, launch every specialist that applies **in a single message**:

| Specialist | Invoke when the change... |
|------------|---------------------------|
| `architect` | adds a new service, datastore, crate/package, or cross-cutting pattern |
| `database-architect` | adds or alters tables, columns, indexes, constraints, relationships, or data access patterns |
| `api-designer` | adds or changes an endpoint, RPC, GraphQL field, webhook/event payload, or public library interface |

Skip this sub-stage if none apply. If `architect` changes the overall design in a way that affects the others' work, re-run the affected specialist with the architect's handoff.

**3b. Plan → `planner`.** Send the spec, the explorer handoff, and all specialist handoffs. The plan must:
- map every acceptance criterion to at least one step and one test
- follow the `database-architect` migration plan, in its release order, if present
- turn the `api-designer` "Contract Tests for the Implementer" into test steps, if present
- carry forward any breaking changes, flagged for `release-manager`

**GATE 1: user approval.** Show the spec summary, the key design decisions (schema changes, API contract, breaking changes), and the plan. Wait for the user to approve or change it. Do not proceed without approval. If the user requests changes, re-run the planner with their feedback.

### Stage 4: Implement → `implementer` agent
Send the approved plan, spec, explorer handoff, and specialist handoffs (schema/migration plan, API contract). Receive `HANDOFF: implementer -> code-reviewer, security-reviewer`.

If the implementer reports **BLOCKED** on a design issue, return to Stage 3 with the blocker (re-plan, re-approve). Don't push the implementer to work around it.

If the build or tests are broken and the implementer can't fix them, use `rust-build-resolver` (Rust) or `build-error-resolver` (TS/JS), then resume.

### Stage 5: Review → `code-reviewer` + `security-reviewer` (in parallel)
Launch both in a **single message** so they run concurrently. Give each the diff scope (`git diff <default-branch>...HEAD`), the spec, and the implementer handoff (especially "Areas Needing Careful Review"). For Rust, tell them to apply the `rust-patterns` review checklist.

Merge the findings into one list, de-duplicated and ranked CRITICAL > HIGH > MEDIUM > LOW.

### Stage 6: Fix Loop → `implementer` (fix-loop mode)
- If there are any CRITICAL or HIGH findings, send them to the implementer, then re-run Stage 5 **only for the reviewer(s) that raised them**.
- **At most 2 fix loops.** If CRITICAL/HIGH findings remain after that, stop and present them to the user with the options.
- MEDIUM/LOW findings the implementer deferred go into the final report.

### Stage 7: E2E → `e2e-runner` (conditional)
Run only if user-facing flows changed (pages, components, routes, CLI UX) and `--no-e2e` wasn't given. Cover the acceptance criteria that describe user-visible behavior. Failures go back through Stage 6 (they count toward the same 2-loop limit).

### Stage 8: Verify (you, via the `verification-loop` skill)
Run the full verification with the project's CI flags: build, types, lint, all tests, security scan, debug-statement audit. It uses cargo commands for Rust projects. Every acceptance criterion from Stage 1 must have a passing test; list the mapping.

If anything fails, go back to Stage 4/6. Don't report success on a red build.

### Stage 9: Docs + Infra → `doc-updater` and `devops` (conditional, in parallel)
- **`doc-updater`**: only if a public API, CLI, config option, environment variable, or setup step changed. Update the README, docs, and codemaps that are affected. Skip internal-only changes.
- **`devops`**: only if the change adds or alters environment variables, services, ports, system dependencies, build steps, Dockerfiles, CI workflows, or deployment config. It updates `.env.example`, the Dockerfile, CI, and deploy config, and returns the list of secrets the user must set. `devops` never deploys or applies infrastructure inside `/ship`; any such action goes into the final report as a follow-up for the user.

If both apply, launch them in a single message. If `devops` changed CI or build files, re-run the affected parts of Stage 8 before delivering.

### Stage 10: Deliver
1. Stage only the files belonging to this change and write a conventional commit message (`feat:` / `fix:` ...) whose body summarizes the change and references the acceptance criteria. Follow the repository's commit conventions and any attribution rules.
2. Show the final report (below).
3. **GATE 2: user approval to publish.** Unless `--no-pr`, ask before pushing and opening a PR. On approval: push the branch and open the PR with `gh pr create`, using the report as the body (spec, plan summary, test evidence, review outcome, deferred items).
4. Delete `.claude/ship/<slug>/` after a successful PR, or leave it if the run stopped early.

## Final Report

```
SHIP REPORT: <slug>
====================
Task:    <one line>
Branch:  feat/<slug>     Commit: <sha>

Acceptance criteria
  [x] <criterion>  -> tests/path::test_name
  [x] ...

Pipeline
  spec ✓  explore ✓  design: arch –|✓ db –|✓ api –|✓  plan ✓ (approved)  implement ✓
  review: code ✓ security ✓ (fix loops: 1)  e2e ✓|skipped  verify ✓  docs ✓|skipped  infra ✓|skipped

Verification
  build OK | types OK | lint OK | tests 214/214 | security OK

Files changed: N (+A / -D)

Deferred (not blocking)
  - [MEDIUM] ... (reason)

Out-of-scope observations
  - ...

Follow-ups for you
  - Set secrets: FOO_API_KEY (staging, prod)   [from devops]
  - Deploy / infra actions needing approval     [from devops]

Status: READY FOR PR | PR OPENED <url> | STOPPED AT <stage> - <reason>
```

## Rules

- **Two gates, never skipped:** plan approval (Gate 1) and publish approval (Gate 2). Everything in between runs without stopping unless something is blocked.
- **Keep the user posted:** one short line when each stage starts or finishes ("Stage 5: review running, code + security in parallel").
- **Pass handoffs, not transcripts.** Each agent gets the spec plus only the handoffs it needs.
- **Never fake progress.** If a stage was skipped, failed, or produced warnings, say so in the report.
- **Stop conditions:** design blocker that needs re-planning, 2 fix loops exhausted, verification still red, or anything touching production data or credentials not covered by the plan.

## When to Use Something Smaller

- Typo or one-line fix: just make the change and run `/verify`
- Only want a plan: `/plan`
- Only want a review: `/code-review`
- Custom agent sequence: `/orchestrate custom ...`
- Package merged features into a version: `/release`
- CI, Docker, or deployment setup on its own: ask for the `devops` agent
