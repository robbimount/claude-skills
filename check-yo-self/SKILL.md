---
name: check-yo-self
description: >-
  Use when the user says "check yo self" (or "check yourself"), asks for a self-review of their
  own changes before a PR, merge or hand-off, or wants a diff, branch, PR or set of files checked
  for code smells, architectural smells, language anti-patterns, logic bugs, semantic overloading,
  temporal coupling, representable illegal states, unhandled edge cases, code cleanliness or yappy
  comments.
---

# Check yo self

The author of a change shares its blind spots. Every lens below runs in its own fresh subagent
that sees the code, never your conclusions about it. You orchestrate, verify and report.

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

## 1. Resolve the scope

- The user named a target (paths, a PR number, a commit range): use it.
- Otherwise: the current branch against its merge base with the default branch
  (`git merge-base HEAD origin/HEAD`, falling back to `main`/`master`), plus uncommitted changes and
  untracked source files. Skip ignored files and binaries.
- Write down the exact `git diff` command and the changed files. If there are none, say so and stop.

## 2. Dispatch every lens at once

One message, one Agent call per lens, all in parallel: `general-purpose` agents with the most capable
model passed explicitly (e.g. `model: "opus"`), never inherited. All eight run whatever the diff's
size; if the user named lenses (e.g. `bugs, edges`), run only those. Over about 2,000 changed lines,
split the files into areas and give each lens one agent per area.

| Lens | Looks for |
|---|---|
| **code-smells** | Long functions, duplication, deep nesting, flag arguments, magic values, primitive obsession, feature envy, speculative generality |
| **architecture** | Dependency direction and layering, import cycles, logic in the wrong module, leaky abstractions, tight coupling, one-implementation interfaces, state owned in two places |
| **anti-patterns** | The idioms of the changed files' languages and frameworks, judged by the project's own lint config and conventions (e.g. TS: `any`, non-null `!`, floating promises, mutable exports) |
| **logic-bugs** | Wrong conditions, off-by-one, inverted checks, races, state-machine holes, wrong error propagation. Traces every caller of a changed function |
| **invariants** | Semantic overloading (one name, field, flag or return value carrying two meanings, e.g. `null` meaning both "absent" and "failed", a missing field doubling as a signal); temporal coupling (calls that must happen in an unstated order, init-before-use, setters that must precede a method, state valid only between two calls); representable illegal states (types or data shapes that allow combinations the code must never see, e.g. `{ok: true, error}`, two booleans that can't both be true, optional fields that are required together), and whether a discriminated union, constructor or narrower type would make them unrepresentable |
| **edge-cases** | Empty, null, huge or malformed input; dependency failures, timeouts, partial failure, concurrency, retries, data written by an older version |
| **cleanliness** | Naming, dead code, unused imports and parameters, leftover debug output and TODOs, inconsistency with the surrounding code, lint and format issues |
| **comments** | Loads the `yeet-yo-yap` skill and applies its rules to the comments the change adds or touches, reporting each change it would make without making it |

Each brief is the shared header below plus that lens's row. Never add your own opinion of the code,
your suspicions, or what you already checked: a reviewer handed a hypothesis tends to confirm it.

```
Review <diff command> in <repo path>; changed files: <list>.
Lens: <lens name>: <the lens's "Looks for" cell>.
Read the surrounding code and callers, not just the hunks. Report only issues in the change, or
issues the change causes elsewhere. Read-only: do not edit, commit or spawn agents.
Return a list; each finding is:
  severity | file:line | the issue in one sentence |
  a concrete scenario where it bites | the fix in one line
Severity: critical = wrong results, data loss, a crash or a security hole in normal use;
important = breaks on a realistic input or path, or makes the code meaningfully harder to change;
minor = polish.
End with "Checked:" and one line on what you covered. An empty list is a valid answer.
```

A reviewer that fails or returns no usable list is dispatched once more with the same brief. If it
fails again, the report says that lens did not run.

## 3. Verify, merge, rank

1. Open every cited location yourself. Keep a finding only if the code shows it; when a scenario
   is cheap to prove, prove it (a unit test, a one-line script) in a scratch location or a file you
   delete afterwards, so the working tree ends as it started. The rest go to **Dropped**, each with
   the reason.
2. Merge duplicates across lenses into one finding, keeping the highest severity.
3. Re-grade by what a user of the software would get if it shipped, using the scale in the brief,
   not by the reviewer's label.

## 4. Report, then wait

Make no edits until the user picks what to fix. The report, in the terminal:

- **Critical** / **Important** / **Minor**: `file:line`, the issue, the scenario, the fix. Name the
  lens(es) that found it.
- **Comments**: the count, plus the ones worth changing.
- **Dropped**: one line each, with why.
- One closing line: ready to ship, or what blocks it.

Then offer to fix the chosen findings. Each fix gets a failing test first where behavior changes.
