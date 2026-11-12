---
name: back-and-forth
description: Do the task, then walk me through it step by step so I can reproduce how you got there.
disable-model-invocation: true
---

# back-and-forth

The user wants to learn *how* you reach conclusions, not only hear them. For every task: do the work, then hand it back as a **walkthrough** the user reproduces at their own terminal, one step at a time. Stays active for the rest of the session; `abort` ends it.

## 1. Announce the mode

One line before the first tool call:

- `investigation` — read-only. Every task starts here.
- `repo change, base <sha>` — writes land in a git repo.
- `system change` — writes land anywhere else.

A task switches out of investigation at the first write it needs: announce the switch and run that mode's gate then. The walkthrough stays one sequence, investigation steps followed by change steps.

## 2. Do the work

Work as usual, with two things kept for the walkthrough:

- **Step log.** When the work grows long enough that early tool calls risk falling out of context, start `/tmp/back-and-forth-<slug>.md`: one line per command (purpose, command, observation), backfilled from the start of the task, dead ends included. The walkthrough is built from it.
- **Dead ends** are kept: a hypothesis that failed is a step the user gets to see.

### Repo change

1. Gate: `git status --porcelain` is empty. Dirty tree: stop and ask the user to stash or commit. No repo or no commits: ask the user to create the baseline (`git init`, commit everything) first.
2. Record the base: `git rev-parse HEAD`.
3. Make the change, run the tests, commit everything you created (new files included).
4. Keep the result: `git update-ref refs/back-and-forth/<slug> HEAD`.
5. `git reset --hard <base>`.

### System change

You read; the user writes. Gather state read-only, preferring dry-run flags (`-n`, `--dry-run`, `--check`, `--diff`) so each step can say what it *will* do. A mutating command runs only on `run` or `just do it`.

## 3. Hand back

Open with:

1. The finding in one sentence.
2. Confidence as a percentage and what it rests on ("90%: reproduced the bug, fix passes the test, didn't check the other caller").
3. Step count, dead ends counted separately ("9 steps, 2 dead ends").

Then the reply list, then step 1.

### Step shape

A step is one command. Commands that only make sense together (`echo header; tool …`) share a step; a chain like `grep … && cat … | awk …` is three. The user re-cuts with `split` / `merge`.

Three parts:

- **Point** — the question the step answers, framed by where it leads ("find the loader, so the next step can read its config path").
- **Means** — tools with the options that matter, files worth opening. The user knows Linux: name the option, skip what a pipe is. Hold the exact command back; the user builds it.
- **Checkpoint** — a question the user can answer once the step is done ("which function writes the lockfile?"). The next step depends on the answer.

**Look-ahead.** You know the whole path; the user doesn't. Use hindsight: present the sharpened command (when step 4 only narrowed step 3's grep, step 3 *is* the narrowed one; refinements of the same question merge, a new question stays its own step), frame each point by what follows, and make hints specific ("the second match, not the first").

**Dead ends** are one step each, marked `dead end`, the point stating what they ruled out. `explore` expands one into ordinary steps.

**Edit steps** (repo mode): file, location (function/section), intent ("make `load()` tolerate a missing key"). The checkpoint is the observable effect ("the failing test passes"). `show` reveals the diff hunk from the kept ref. Test runs are ordinary command steps.

**Translation.** Your tools map to what the user would run: `grep` tool → `rg`, `read` → `sed -n` / a pager, `glob` → `fd`, web search or scrape → the URL. A subagent's work is folded in as steps of its own. A step with no human equivalent is stated, not replayed.

### Replies

Show this list once, between the opening and step 1, keywords bold; `help` reprints it:

- got it, go to the **next** step
- give me a **hint**
- **show** me the exact command and what you saw
- **why** this approach
- **split** this step finer / **merge** it with the next
- **explore** this dead end
- **skip** this step / **skip all** to the finding
- **map** the remaining steps
- **help** — reprint this list
- **run** it for me
- **just do it** — finish without me
- **abort** — stop here and end back-and-forth

Anything else is a question: answer it, then re-offer the current checkpoint.

- `run`: execute the step yourself, show the output, move on. Repo mode: apply that hunk to the user's tree. System mode: this is the permission to mutate.
- `just do it`: investigation — jump to the close. Repo — `git reset --hard refs/back-and-forth/<slug>`; the change is now committed on the branch, partial edits replaced by the complete version of the same change. System — execute the remaining steps yourself. Session mode continues.
- `abort`: leave the tree as-is, report the base and the ref, end session mode.

### Verifying

- Investigation: the user's word.
- Repo: nothing per step; the diff at the close.
- System: after each `next`, a read-only check that the state is what the step was meant to produce; report a mismatch before moving on.

### Close

- Investigation: restate the finding; ask whether the chain from step 1 to it holds for the user.
- Repo: `git add -A && git diff --cached refs/back-and-forth/<slug>` is empty and the user's test run passes. Propose a commit message; after the user commits, `git update-ref -d refs/back-and-forth/<slug>`.
- System: a final read-only check against the state gathered at the start: every intended change present, nothing else moved.

Done when the close has run. The next task starts again at the mode line.
