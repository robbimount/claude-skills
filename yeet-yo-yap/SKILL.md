---
name: yeet-yo-yap
description: >-
  Write and review code comments to industry best practice: short, accurate, and only where they
  help. Use when writing code that needs comments, when reviewing or cleaning up comments (a diff,
  a branch, a file), when the user says comments are "yappy", too long, over-explained or
  "explainy", or asks to "yeet yo yap" (or "yeet the yap"). Removes comments that restate code,
  explain approaches not taken, describe earlier versions of the code, carry session or review
  chatter, or reference tickets; adds missing contracts on public interfaces; flags comments that
  are hard to write as a design smell; keeps JSDoc/TSDoc/docstring @param, @returns and @throws
  succinct; keeps `ponytail:` comments that mark deliberate shortcuts.
---

# Yeet yo yap

A comment earns its place by telling the reader something the code can't. Everything else is yap:
it costs reading time, goes stale, and buries the comments that matter.

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

## The gate

Before writing or keeping any comment, ask one question:

> **What does this tell a first-time reader that the code doesn't?**

Nothing → no comment. "Obvious" means obvious to someone new to the code, not to its author.

Before reaching for a comment, try in order: a better name, a smaller function, a type or named
constant, a test whose name states the rule. Comment only what none of those can carry, typically
a why, a precondition, or a constraint that lives somewhere else.

## The rules

### Say less

1. **Don't restate the code.** `// increment the counter` above `count++` goes. So does a doc
   comment that rephrases the name (`/** Fetches the user profile */ fetchUserProfile()`).
2. **Short and plain.** A one-line summary; two at most. Fragments are fine if the file uses them.
   Stop when the reader has what they need.
3. **Prefer clearer code to a comment.** If a comment explains what a name or expression means,
   rename it or extract a well-named variable or function instead, and drop the comment.
4. **No emphasis, humor, apologies or noise.** No `IMPORTANT!!!`, `// hack lol`, `// sorry`,
   `// magic happens here`, or banner art.
5. **Hard to describe means badly designed.** If a comment needs to be long, hedged, or has to
   explain the internals for a caller to use the thing, the abstraction is shallow or leaky. Fix
   the design (or flag it) rather than polishing the prose. A block over about ten lines belongs in
   a README, design doc, test name or commit message.

### Say only what's true now

6. **No history.** Never describe an earlier version of the code: "used to", "no longer", "now
   does", "the old X", "moved from", "switched to", "after the refactor". Git holds the history.
   State the current rule in the present tense instead.
7. **No alternatives.** Never explain why this approach was chosen over another ("instead of a
   proxy", "rather than polling", "X doesn't work here, so"). That's design rationale: it belongs
   in a README, design doc or ADR, or in the commit message. Never in code.
8. **No session or review chatter.** No `// as discussed`, `// per review`, `// fixed per
   feedback`, `// Claude: added this helper`, `// AI-generated`, `// the fake now omits X so…`.
   Describe what the code does, not the conversation that produced it.
9. **No ticket or tracker references.** No Jira keys, PR numbers or internal links. The one
   exception is a workaround for a **third-party** bug: link the upstream issue, so whoever finds it
   fixed upstream knows the workaround can go.
10. **No authorship, dates or changelogs.** No `// added by`, `// 2026-10-21`, `// v2: …`. Git
    blame and the log record these.
11. **No perishable measurements.** Timings, counts and "currently" stamps rot silently
    (`// takes ~40s`, `// no caller currently uses this`). State the durable fact (`// slow; the
    timeout is generous on purpose`). Fine: a target or budget, a platform constant, a dated
    snapshot of an external limit (`// as of 2026-08 the API caps this at 100`).
12. **Keep comments accurate.** When code changes, fix or delete the comments it makes wrong. A
    stale comment is worse than none. When moving code, move its comments with it.

### Say the useful things

13. **A short "why", only where the code would look wrong or fragile without it.** One clause,
    stated as a fact about the constraint. Good: `// The router uses the first matching route, so
    /users/me comes before /users/:id`. Bad: a paragraph on how the problem was found. Likewise a
    non-trivial regex, bit trick or external format gets a one-line gloss or a spec pointer
    (`RFC 1035 §4`).
14. **Warn about consequences elsewhere.** When a change here breaks or must be mirrored somewhere
    else, say where: `// Kept in sync with ERROR_CODES in api/errors.ts`, `// The snapshot test
    pins this shape`. Put it at the point a future editor will touch.
15. **Document the contract, not the implementation.** On a public function, type or module: a
    sentence giving the mental model, then the precise parts a caller needs without reading the
    body: what each input means, what it returns, what it throws, side effects, and any ordering,
    concurrency or lifetime rules. How it works inside belongs in the code.
16. **Parameters, returns and throws, succinctly.**
    - `@param` only when the name and type don't already say it: units, allowed range, inclusive
      or exclusive bounds, what `null` means, whether it's mutated. Skip it when it would only
      restate the name.
    - `@returns` when the value isn't obvious, especially what `null`, `undefined`, an empty
      collection or a sentinel means.
    - `@throws` whenever the function throws on purpose: the condition, in a few words.
    - Use the language's convention: JSDoc/TSDoc, Python docstrings, KDoc, Javadoc, GoDoc, Rustdoc.
17. **Fields, constants and magic values: nouns, not verbs.** Say what the value *is*, not how it's
    used: units, valid range, bounds, nullability, who owns or closes it, invariants tying it to
    other fields. `/** Upstream lookup timeout (ms) */`. A bare number in logic gets a name, or a
    short comment if a name can't carry it.

### Leave these alone

18. **`ponytail:` comments stay.** They mark a deliberate shortcut and its limit
    (`// ponytail: global lock; per-account locks if throughput matters`). Keep them to one or two
    lines: the limit, and the upgrade path if there is one. Don't invent an upgrade path that
    hasn't been agreed.
19. **License and legal headers stay** untouched.
20. **Tooling directives stay**: `eslint-disable-next-line`, `@ts-expect-error`, `// prettier-ignore`,
    `# noqa`, `#pragma`, type annotations in comments. Each should name its reason in a few words
    when the reason isn't obvious.

### Never

21. **No commented-out code.** Delete it; git has it.
22. **No secrets, credentials, internal hostnames or customer data** in comments.
23. **No TODOs without substance.** A vague `// TODO: clean up` goes. A deliberate limit is a
    `ponytail:` comment; real follow-up work belongs in the tracker, not in the code.

### Fit in

24. **Match the file.** Follow the surrounding code's comment density, voice and conventions
    (doc-comment style, sentence case, punctuation) and the project's spelling convention. Don't
    annotate every line of a sparse file; don't strip a well-commented one.
25. **Tests aren't exempt.** Let the test name say what's tested. Comment only the invariant being
    pinned or setup that would otherwise puzzle a reader (a magic coordinate, a deliberate wait, a
    fixture's quirk), never the history of the bug.

## Writing new code

For a new public function, type or module, write its contract comment **before** the body, while
the intent is fresh. If it won't fit in a line or two without describing internals, rethink the
interface before implementing it (rule 5). Then write the body and add only what the rules call
for: units on constants, a short "why" where the code would look wrong, a pointer where a change
must be mirrored elsewhere, and `ponytail:` on deliberate shortcuts. Before finishing, reread the
contract against the finished code and fix whatever drifted.

## Reviewing comments

1. **Scope.** Unless told otherwise, review the comments the change introduced:
   `git diff <base>...HEAD -- <files>`. Pre-existing comments stay as they are, even ones you'd
   write differently, **except** ones the change made false: check comments on and around changed
   code for rot and fix those.
2. **Rewrite or delete** each comment against the rules. Most fixes are cuts: drop the history,
   the alternatives, the chatter and the restatement, and keep the one clause that matters.
3. **Add what's missing.** A new or changed public interface with no contract, a constant with no
   unit, a fragile ordering with no "why", or a cross-file dependency with no pointer gets a short
   comment.
4. **Flag design smells.** Where a comment is long or convoluted because the code is (rule 5),
   report it as a design issue rather than silently trimming it.
5. **Move rationale, don't lose it.** If a comment held real design reasoning (why X, not Y) and
   nothing else records it, offer to move it to the README or design doc rather than delete it.
6. **Change comments only.** Never touch code, names, strings, or anything a test depends on (for
   example coordinates or IDs documented in a template comment). Suggest renames and refactors in
   the report instead.
7. **Prove it.** Show that only comments changed. For TypeScript/JavaScript, transpile each changed
   file at the base and in the working tree with comments removed (`ts.transpileModule` with
   `removeComments: true`) and compare. For other languages, compare with comment lines stripped,
   or run the compiler or formatter the same way. Then run the formatter, linter, type checker and
   tests.
8. **Scan for leftovers** in the comment lines the change adds, for example:

   ```bash
   added=$(git diff <base> | grep -E '^\+\s*(//|\*|/\*|#)')
   echo "$added" | grep -E '\b[A-Z]{2,}-[0-9]+\b'   # tracker keys (case-sensitive)
   echo "$added" | grep -i -E 'jira|pr ?#[0-9]|instead of|rather than|no longer|used to|previously|switched to|the old\b|anymore|currently|as discussed|per review|claude:|ai-generated|generated by'
   ```

   The key pattern also hits names like `UTF-8` and `SHA-256`. Review every hit. "Now" that describes the
   present is fine; "now" that contrasts with the past isn't.
9. **Report** per file how many comments were shortened, removed, added and left alone, with a few
   before/after examples, then list any design smells and suggested renames.

## Examples

Restates the code, and explains an alternative:

```ts
// Before
/**
 * Gets the user. We use a Map here instead of an array because lookups by id
 * were slow when we used find() before, which caused the dashboard to lag.
 */
function getUser(id: string): User | undefined { … }

// After
/** @returns undefined when no user has this id */
function getUser(id: string): User | undefined { … }
```

History and a ticket:

```ts
// Before
// PROJ-1234: this used to throw, but now it returns null so callers can retry
// After
// Callers retry on null
```

Session chatter and a perishable number:

```ts
// Before
// Per review feedback, bumped this since the export currently takes ~40s
const EXPORT_TIMEOUT_MS = 60_000;
// After
/** Export timeout (ms); large exports are slow */
const EXPORT_TIMEOUT_MS = 60_000;
```

A field, described as a noun with its bounds:

```ts
// Before
/** Used to slice the buffer when we read the next chunk */
end: number;
// After
/** Index one past the last valid byte (exclusive) */
end: number;
```

A "why" worth keeping, cut to one clause:

```ts
// Before
// We have to register /users/me before /users/:id, because the router walks the
// list top to bottom and stops at the first match, so otherwise /users/:id would
// treat "me" as an id.
// After
// The router uses the first matching route, so /users/me comes before /users/:id
```
