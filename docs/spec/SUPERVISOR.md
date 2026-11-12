# Supervisor protocol

> How one supervisor session turns [`TASKS.md`](TASKS.md) into merged code by delegating single tasks to implementer
> sub-sessions. The supervisor owns decisions, contracts, briefs, reviews and merges. An implementer owns one task.
> Models, the way sub-sessions are started, the parallel limit and the autonomy level come from
> [`DECISIONS.md`](DECISIONS.md) D-14–D-18. This file does not repeat them.

## 1. Roles

| Role | Owns | Never does |
|---|---|---|
| **Supervisor** | Task selection, briefs, `Executor: supervisor` tasks, SPEC amendments, shared files (§4), reviews, merges, `PROGRESS.md` | Merging a red PR; weakening, skipping or disabling a test to get green; stating an inference as a fact |
| **Implementer** | Exactly one `Executor: implementer` task, on its own branch, ending in one PR | Editing files outside the task's `Files:` list; changing SPEC or TASKS; adding packages; merging |
| **Owner** | `DECISIONS.md`, `Executor: owner` tasks, anything outside the repository | — |

## 2. Before the first task

The supervisor reads `DECISIONS.md`. If its status is not `decided`, the supervisor asks the owner the open rows, **all at
once, before starting**, and writes the answers into the file. Working "fully autonomously" starts after this, not instead
of it: a missing decision found later costs a blocked gate.

## 3. The loop

```
repeat until the milestone gate passes:
  1. ready = tasks whose Depends are all merged
  2. batch = ready tasks with disjoint Files: sets, at most D-16 of them
  3. for each task: branch task/<id>-<slug> from the latest default branch; start an implementer with the brief (§5)
  4. for each PR:
       red   -> §6
       green -> review against §7; accept: squash-merge and update PROGRESS.md with the run link
                                    reject: one revision round with the findings, then §6
  5. after each merge: bring the default branch into open branches ONE AT A TIME (see §4)
```

Phase 0 of `TASKS.md` is never parallel and never skipped.

## 4. Parallelism and CI load

- **Disjoint files.** Two tasks run together only if their `Files:` sets do not overlap.
- **Shared files are supervisor-only:** `Directory.Packages.props`, the solution file, `.github/workflows/**`, the AppHost,
  shared contracts. An implementer requests a change in the PR description; the supervisor applies it.
- **CI is a shared resource.** On self-hosted runners every running pull request competes for the same network line, the
  same Docker daemon and the same disk. Therefore:
  - at most D-16 pull requests have CI running at once; when the pipeline is unstable, exactly one;
  - after a fix lands on the default branch, open branches are updated one after another, each waiting for green;
  - a pull request that only waits is not pushed to "to see if it helps".
- **Contracts first.** DTOs shared between services are defined by the supervisor before the services are built in parallel.

## 5. The implementer brief

Everything the implementer needs is in the brief or in files the brief names. It has no memory of other tasks.

```text
You are implementing task <ID> of <owner>/<repo>.

Load first, in this order:
  1. The architecture-standards skill implementation-phase, then <skills from the task's Read line>.
  2. docs/spec/SPEC.md sections <from the task's Read line>.
  3. AGENTS.md.

The task (verbatim from docs/spec/TASKS.md):
<the whole task block>

Rules:
- Work on branch task/<id>-<slug>. Touch only the files under Files. If another file must change, stop and write the
  reason in the PR description under "Out-of-scope changes needed".
- Package versions live only in Directory.Packages.props. Request a new package in the PR description; do not add it.
- Fixtures are synthetic. No secrets in any file.
- A test that cannot fail is not a test. A wait in a test waits on a condition with a generous limit.
- Before opening the PR run the task's Verify command and the common verify.
- Use .github/pull_request_template.md: each Accept criterion with its proof (test name, command and output, or run link).
- Report what you measured as measured and what you concluded as a conclusion. Do not claim a criterion you did not prove.
- If the task is ambiguous or contradicts SPEC or the standards: stop, do not guess, and write the question in the PR
  description under "Blocked: question for supervisor".
- You do not merge. Do not re-run a failed job more than once.
```

## 6. When something fails

| Situation | Action |
|---|---|
| A job fails | Read the log of the failing step first. Name the cause before changing anything. "Flaky" is not a cause |
| The failure is in the test or the code | Fix it on the branch; one fix attempt by the implementer, then the supervisor reads the diff |
| The failure is the pipeline (a download timed out, a runner disappeared, the Docker host ran out of a resource) | Re-run the failed jobs **once**. If it repeats, stop pushing, reduce to one pull request at a time, and report to the owner what the host needs. It is not fixed by changing the code under test |
| Several jobs die in the same second, or jobs stay queued with no runner | Runners are gone. Do not re-run and do not push; tell the owner |
| A wait in a test was too short under load | Lengthen the wait on its condition. Never skip, disable or quarantine the test |
| A call between services timed out | The time budget in SPEC §5/§6 was wrong: fix the budget and the retry rule in the specification and in the client |
| "Blocked: question for supervisor" | Answer it; if the answer changes the specification, a SPEC PR lands first |
| Task too big (more than about 400 changed lines) | Split it into new task ids before the next attempt |
| Standards conflict | Comply, or record a deviation (ADR and SPEC §12.2) before merge |

## 7. Review checklist (every PR)

- Each Accept criterion is proven by a named test, a pasted command output or a run link.
- At least one test would fail if the feature were removed.
- No file outside `Files:`; no package version in a `.csproj`; no secret; no absolute path.
- Every call to another service has the time budget and retry rule of SPEC §6.
- `PROGRESS.md` and the README still say what is true after this PR (P14).

## 8. Reporting to the owner

- **State, then evidence.** "Merged", "open and green", "open and red", "running", "not started" are different states;
  each is reported with the commit and the run it was read from.
- **Measured or inferred.** What was read from a log or a run is a fact. What was concluded from symptoms is labelled as a
  conclusion, with what would confirm it. The supervisor cannot see the runner host, the owner's machine or anything else
  outside the repository, and says so.
- **What is not shown.** Every status report lists what is still unproven (for example "three green runs on one commit:
  not shown").
- **Owner actions** are listed separately from agent work, with the exact setting or command.

## 9. Checkpoint after every phase

Before the next phase starts the supervisor compares what the documents claim with what can be measured now
(the `session-status` skill), corrects `PROGRESS.md` and the README in one pull request, and lists the gaps as new tasks.
No new feature work starts while a gate that was due is neither claimed with proof nor reported as failed with evidence.

## 10. Implementers on Claude Haiku 5.5 (this project)

DECISIONS D-14 puts the implementers on `claude-haiku-5-5`. A fast, small model does well on a narrow, explicit task and badly on
an open one, so the protocol adds these rules to §3 to §7. They override nothing; they make the brief stricter.

1. **Everything the task needs is in the brief or in a file it names.** The brief quotes the whole task block, the contract
   excerpts it needs (copied from `CONTRACTS.md`, not summarised), and the exact test vectors. An implementer is never told
   "see the spec" for a number.
2. **Smaller tasks.** Aim for about 300 changed lines. A task that grows past 400 is split by the implementer, who stops and
   reports the split in the PR description; the supervisor writes the new task ids.
3. **Tests first, shown failing.** The brief asks for the Accept tests to be written first, run to see them fail for the right
   reason, then the code. The PR description pastes the first failing run's one-line summary.
4. **Port, do not invent.** Where a task names a reference (the chess project's `Chess.Llm`, an authservice endpoint, a guide), the
   implementer attaches or reads it and copies the shape. A new abstraction not named in the task is a finding.
5. **One revision round, then the supervisor decides.** After one review round with findings, if the PR is still wrong the
   supervisor either rewrites the brief with the missing facts and starts a **new** implementer session on the same branch, or
   takes the task over. It does not keep a session in a loop of small corrections.
6. **The supervisor verifies, it does not trust the report.** It re-runs the Verify command (or reads its CI run), reads the
   test bodies for the "can this fail" bar (§7), and greps the diff for the guards in SPEC §5 (clock, logging of free text, SDK
   namespace, package versions in a `.csproj`). A claim without a link is not a claim.
7. **Stop conditions the brief states:** a contradiction between SPEC, CONTRACTS and the task; a needed change outside `Files:`;
   an unknown behaviour of a real dependency; any wish to change a constant of CONTRACTS §9. The implementer stops and writes
   "Blocked: question for supervisor". It does not choose.
8. **Constants and formulas are the owner's.** An implementer who finds a number that seems wrong reports it; it never adjusts it.
9. **Language of the brief:** English. User-facing Polish text goes through `pl.ts` and the template files only.

## 11. Starting and tracking sub-sessions

How D-15 is carried out with the `claude-code-remote` tools (the director has them in its session; if a tool is not listed,
load it with ToolSearch):

- **Start:** `create_session` with `model: "claude-haiku-5-5"`, `source_url` of this repository, `source_revision` the latest
  default branch, `title` `<task id> <title>`, `outcome_branch` `task/<id>-<slug>` unless the environment prescribes a branch
  pattern (then use it and note the mapping in `PROGRESS.md` Notes), and `prompt` the brief of §5 plus §10. Pass `effort` only if the owner asked.
- **Track:** `list_sessions` and `get_session` (`status_bucket`: working, blocked, review_ready, completed, failed) and
  `list_events` with `kinds: ["assistant","result"]` to read what it did. Do not poll in a loop: when nothing else remains, end
  the turn and let events wake you (PR events through `subscribe_pr_activity`).
- **Talk:** `send_message` for an answer to a "Blocked: question for supervisor"; `interrupt_session` for a session that has
  gone off the task.
- **A session can stop mid-task** (a usage limit, a restart). Before resuming or replacing it, read its branch, its pull request
  and its working tree, and continue from what was pushed. Never restart the task from the default branch: that is how work is
  duplicated (a lesson of the chess project).
- **Finish:** after the merge and the `PROGRESS.md` update, `archive_session` (D-17 allows it).
- **In-process agents** (the Agent tool) are for read-only research only: reading the standards, the chess project or authservice
  and returning a summary the supervisor then checks.
- **Branch names and PRs:** one branch and one PR per task. The PR is opened ready for review, with the pull request template of the repository.

## 12. The director's first hour

1. Read `README.md`, `AGENTS.md`, `DECISIONS.md`, then `SPEC.md`, `CONTRACTS.md`, `COPILOT-SDK.md`, `TASKS.md`, `PROGRESS.md`.
2. Run `scripts/check-spec.sh && scripts/check-open-markers.sh` and report the output.
3. Verify that the standards are readable (the plugins of `.claude/settings.json`; otherwise attach `architecture-standards` read-only).
4. Ask the owner the open rows of `DECISIONS.md` in **one** message (§2), and which model the director itself runs on (D-14).
5. Only then start T-00.02, T-01.01 and T-01.02, within D-16.

