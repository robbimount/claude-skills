---
name: check-yo-self
description: >-
  Use when the user says "check yo self" (or "check yourself"), asks for a self-review of their
  own changes before a PR, merge or hand-off, asks to review someone else's PR and post the
  findings as comments, or wants a diff, branch, PR or set of files checked for code smells,
  architectural smells, language anti-patterns, logic bugs, semantic overloading, temporal
  coupling, representable illegal states, unhandled edge cases, security holes, test gaps,
  resource leaks, breaking changes, missing logs or metrics, cyclomatic complexity, code
  cleanliness or yappy comments.
---

# Check yo self

The author of a change shares its blind spots. Every lens below runs in its own fresh subagent
that sees the code and the facts around it, never your conclusions about it. You orchestrate,
verify and report.

## Stay current

This skill is maintained at https://github.com/robbimount/claude-skills. Before following it,
pull updates if the last check was more than a day ago, replacing `<skill dir>` with this skill's
base directory:

```bash
d="<skill dir>"
g=$(git -C "$d" rev-parse --absolute-git-dir 2>/dev/null) &&
  [ -z "$(find "$g/FETCH_HEAD" -mmin -1440 2>/dev/null)" ] &&
  git -C "$d" pull --ff-only --quiet
```

If the pull changed this file, re-read it before continuing. If the directory isn't a git
checkout, or the pull fails (offline, local edits), carry on with the local copy and mention it
once.

## 0. Pick the mode

| Mode | Chosen when | For | What gets reported |
|---|---|---|---|
| `self` (default) | Anything else | Code the user owns, usually written in this session | Every verified finding, minors included, plus complexity over 10 |
| `review` | The arguments start with `review` or `--review`, or the user asks for review mode or for findings posted as PR comments | Someone else's change | Only findings that pass the review bar (step 4); drafted as PR comments |

## 1. Resolve the scope and start a run file

- The user named a target (paths, a PR number, a commit range): use it. `review` mode with no
  target: ask for the PR.
- Otherwise: the current branch against its merge base with the default branch
  (`git merge-base HEAD origin/HEAD`, falling back to `main`/`master`), plus uncommitted changes and
  untracked source files. Skip ignored files and binaries.
- A PR whose head isn't checked out: `git fetch origin pull/<n>/head` and
  `git worktree add <scratch>/pr-<n> FETCH_HEAD`, and point the reviewers there, so they read the
  PR's code rather than whatever branch is checked out. Remove the worktree when done.
- If there are no changed files, say so and stop.

Create a run directory in the scratchpad (`<scratch>/check-yo-self-<timestamp>/`) holding
`run.md`: the mode, repo path, exact diff command, changed files, each lens with its status
(pending, done, failed) and the Context section from step 2. Update the statuses as reviewers
finish. If the conversation is compacted mid-run, re-read `run.md` and carry on from it.

## 2. Gather facts, not opinions

Fill the Context section of `run.md` with facts only:

- **External systems** the change calls or is called by: HTTP APIs, queues and topics, databases,
  caches, third-party SDKs, other services. For each, where its contract can be read: an
  OpenAPI, proto, GraphQL or JSON schema; migrations; the SDK's types or source at the pinned
  version (`node_modules`, `site-packages`, the Go module cache); docs in the repo.
- **Outside consumers** of anything the change exposes (endpoints, events, schemas, packages),
  where you can find them, e.g. `gh search code --owner <org> "<name>"`.
- `review` mode: the PR description, the linked ticket if a connected tool can read it, and the
  existing review comments (`gh api repos/{owner}/{repo}/pulls/<n>/comments`).

No suspicions and no "look closely at X": a reviewer handed a hypothesis tends to confirm it.

### Cyclomatic complexity (`self` mode)

Measure it with a tool. Run the first that works on the changed files:

1. `lizard -C 10 -w <files>` (or `python3 -m lizard`, `pipx run lizard`).
2. The language's own rule: ESLint `complexity: ["warn", 10]`,
   `ruff check --select C901 --config "lint.mccabe.max-complexity = 10"`, `gocyclo -over 10`.
3. No tool works: count each changed function by hand (1, plus 1 per `if`, `else if`, loop,
   `case`, `catch`, `&&`, `||` and `?:`) and mark the score as an estimate.

Keep functions that overlap the diff's hunks and score over 10. Each is an Important finding
under the lens `complexity`, with its score and the branches to extract. The tool's output is its
evidence; it skips step 4's verification.

## 3. Dispatch every lens at once

One message, one Agent call per lens, all in parallel: `general-purpose` agents with the most capable
model passed explicitly (e.g. `model: "opus"`), never inherited. All lenses run whatever the
diff's size, except that `review` mode skips `conventions` unless the user names it. If the user
named lenses (e.g. `bugs, edges`), run only those. Over about 2,000 changed lines, split the files
into areas and give each lens one agent per area.

| Lens | Looks for |
|---|---|
| **design** | Dependency direction and layering, import cycles, logic in the wrong module, leaky abstractions, tight coupling, state owned in two places; duplication within the change or of a helper the codebase already has (search for one before accepting new utility code); long functions, deep nesting, flag arguments, primitive obsession, feature envy, speculative generality and one-implementation interfaces |
| **conventions** | The idioms of the changed files' languages and frameworks, judged by the project's own lint config and conventions (e.g. TS: `any`, non-null `!`, floating promises, mutable exports); naming, magic values, dead code, unused imports and parameters, leftover debug output and TODOs, inconsistency with the surrounding code; and the comments the change adds or touches, judged by the `yeet-yo-yap` skill's rules (load it) |
| **logic-bugs** | Wrong conditions, off-by-one, inverted checks, races, state-machine holes, wrong error propagation, silent failures (errors caught and ignored, logged and carried on past, or hidden behind a fallback or default value). Traces every caller of a changed function |
| **invariants** | Semantic overloading (one name, field, flag or return value carrying two meanings, e.g. `null` meaning both "absent" and "failed", a missing field doubling as a signal); temporal coupling (calls that must happen in an unstated order, init-before-use, setters that must precede a method, state valid only between two calls); representable illegal states (types or data shapes that allow combinations the code must never see, e.g. `{ok: true, error}`, two booleans that can't both be true, optional fields that are required together), and whether a discriminated union, constructor or narrower type would make them unrepresentable |
| **edge-cases** | Empty, null, huge or malformed input; dependency failures, timeouts and missing cancellation, partial failure, concurrency, retries |
| **security** | Injection (SQL, shell, HTML, template), missing authentication or authorization checks, secrets in code, config or logs, path traversal, SSRF, unsafe deserialization, personal data written to logs or errors, input trusted across a trust boundary without validation |
| **tests** | Changed behavior with no test; tests that would still pass if the logic broke; tests asserting internals rather than behavior; mocks that hide the real failure; flaky timing, sleeps and order dependence; missing negative and boundary cases |
| **resources** | Listeners, timers, handles, connections and subscriptions never released; caches, maps and queues that grow without bound; N+1 calls; repeated work or quadratic loops on hot paths |
| **compatibility** | Breaking changes to public APIs, schemas, config, CLI flags or message formats, including for the outside consumers in Context; data written by an older version; missing or irreversible migrations; old and new versions running side by side during a rollout; whether a rollback is safe |
| **observability** | Where the change needs a log line or a metric (New Relic, or whatever monitoring the project uses), judged by the project's existing logging and metrics conventions: failures, retries, fallbacks and dropped work that leave no trace; new features, jobs or paths whose use, outcome or latency nobody can see; logs missing the ids needed to trace a run; wrong log levels; noisy logging in hot paths or loops; metrics with unbounded attribute values (ids, URLs) |

Each brief is the shared header below plus that lens's row. Add nothing else: no opinion of the
code, no suspicions, no list of what you already checked.

```
Mode: <self|review>. Review <diff command> in <repo path>; changed files: <list>.
Facts about the change and the systems it touches: the Context section of <run.md path>.
Lens: <lens name>: <the lens's "Looks for" cell>.
Read the surrounding code and callers, not just the hunks. Before stating how an external system
behaves, read its contract from Context; if none is listed, end the finding with
"assumes: <what>". Report only issues in the change, or issues the change causes elsewhere.
Edit nothing in the repo, commit nothing, spawn no agents.
A finding names the caller, input or path that triggers it, or the concrete future change it
makes harder. A preference, a "could be cleaner", or a rule the project's linter or formatter
already enforces is not a finding. In review mode, report only critical and important findings.
Write your findings to <run dir>/<lens>.md, one per line:
  severity | file:line | the issue in one sentence |
  a concrete scenario where it bites | the fix in one line [| assumes: ...]
Severity: critical = wrong results, data loss, a crash or a security hole in normal use;
important = breaks on a realistic input or path, or makes the code meaningfully harder to change;
minor = polish.
End the file with "Checked:" and one line on what you covered. Reply with only the file's path
and the count of findings per severity. No findings is a valid answer.
```

A reviewer that fails or writes no usable file is dispatched once more with the same brief. If it
fails again, mark it failed in `run.md`; the report says that lens did not run.

## 4. Merge, verify, rank

1. **Merge.** Read the lens files. Merge findings with the same location and cause into one,
   keeping the highest severity and naming every lens that found it. In `review` mode, drop
   anything the existing PR comments already raise.
2. **Verify.** With five or fewer findings, open each cited location yourself. Otherwise group
   the findings by file and verify each group in a fresh Agent, all in parallel, so the
   evidence never fills your context. Its brief is the findings verbatim, the repo path and the
   run directory: "Try to refute each finding. Read the cited code and its callers. When a
   scenario is cheap to prove, prove it with a test or one-line script in the run directory,
   never in the repo. Reply one line per finding: CONFIRMED, REFUTED or UNPROVEN, then the
   evidence in one sentence." Keep CONFIRMED. REFUTED goes to **Dropped** with the evidence.
   UNPROVEN stays, marked unproven, in `self` mode, and goes to Dropped in `review` mode.
3. **Re-grade** by what a user of the software would get if it shipped, using the scale in the
   brief, not by the reviewer's label.
4. **Review bar** (`review` mode). Keep a finding only if one of these holds:
   - **High risk**: it is critical.
   - **Likely**: it is important, and what triggers it happens in normal use: a real caller, a
     real input, a real deploy.
   - **Maintainability**: it is important, and it will cost future changes in this area:
     duplicated logic that must change together, logic in the wrong layer, a broken contract,
     a function too tangled to change safely.

   A finding with an unchecked `assumes:` becomes a question to the author if it would be
   critical. Everything else goes to Dropped as "below the review bar".

## 5. Report, then wait

### `self` mode

Make no edits until the user picks what to fix. The report, in the terminal:

- **Critical** / **Important** / **Minor**: `file:line`, the issue, the scenario, the fix. Name the
  lens(es) that found it; mark unproven ones.
- **Dropped**: one line each, with why.
- One closing line: ready to ship, or what blocks it.

Then offer to fix the chosen findings. Each fix gets a failing test first where behavior changes.

### `review` mode

Post nothing until the user picks. The report, in the terminal:

- **Blocking** (high risk, likely) and **Non-blocking** (maintainability): each as the comment
  it would post, anchored at `file:line`: the issue, the scenario and the suggested fix in a few
  plain sentences. Questions to the author are listed with them.
- **Dropped**: the count per reason, then one line each.
- One closing line: the review action you'd suggest (comment, approve, request changes).

When the user picks the comments and the action, post them as one review: write
`{"commit_id": "<head sha>", "event": "COMMENT", "body": "...", "comments": [{"path", "line",
"side": "RIGHT", "body"}]}` to the run directory and send it with
`gh api repos/{owner}/{repo}/pulls/<n>/reviews --input <file>`. Use `APPROVE` or
`REQUEST_CHANGES` only when the user chose it. A finding on a line outside the diff goes in the
review body. Then remove the PR worktree.
