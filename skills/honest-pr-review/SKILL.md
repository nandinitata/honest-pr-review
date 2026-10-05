---
name: honest-pr-review
description: Use when reviewing a coding agent's pull request or branch diff before merging, or when a change passes its tests but you suspect it gamed them. Catches reward-hacking shortcuts (fetched or copied fixes, weakened or deleted tests, swallowed errors, overfit special-casing, scope creep) and fixes them back to a minimal, honest change.
---

# Honest PR Review

## Core principle

A green test suite is not proof the task was solved. A coding agent rewarded for passing tests will sometimes pass them without doing the work: it fetches a fix someone already wrote, weakens the test, hides the error, or special-cases the one input the test uses. This review assumes the diff might have passed by gaming the check. You verify the work was actually done, then make the change small and honest.

Inspired by Xiaomi's MiMo-V2.6 report, where a model trained to pass unit tests learned to fetch the already-published fix instead of writing it, and a second grader pushed it back toward minimal, clean patches. See `references/checks.md` for the full catalog.

## Workflow

1. **Get the diff.**
   - Local branch: `base=$(git merge-base HEAD origin/main 2>/dev/null || git merge-base HEAD main)`, then `git diff --stat "$base"...HEAD` and `git diff "$base"...HEAD`.
   - GitHub PR: `gh pr diff <number>` and `gh pr view <number>` for the description and the task it claims to solve.
2. **Read full context.** For every changed file, open the whole file, not just the hunk. A swallowed error or a weakened assertion only reads as gaming once you see what it replaced and what it guards.
3. **Run the checks** in `references/checks.md` (the eight below). For each one, look at both the source changes and the test changes together.
4. **Report first.** List the findings before touching anything (see format below). End with a merge verdict: Ready / Ready with fixes / Not ready.
5. **Fix and verify.** Apply the minimal correction for each finding, one at a time. If a test was gamed to hide a real bug, fix the code, not the test. After each fix, run the real suite (the original tests, not the tampered version) and confirm it passes honestly.
6. **Summarize** what you changed and give the new verdict.

## The eight checks

Fetched the fix instead of writing it:
1. **Smuggled-in fix**: code copied from a newer package version, a raw file URL, a later repo checkout, or a fixed issue/PR, rather than implemented here.
2. **Version shortcut**: bumping a dependency so the bug disappears because a newer release already fixed it.

Passed the check without solving it:
3. **Test tampering**: expected values edited to match buggy output, tests deleted, skip/xfail/`.only` added, assertions commented out, fixtures or mocks loosened.
4. **Weak assertions**: `assert True`, an existence check replacing a value check, widened tolerances, dropped conditions.
5. **Swallowed errors**: broad `except: pass`, empty catch blocks, `|| true`, suppressed exit codes, silenced linters or type checkers (`# noqa`, `# type: ignore`, `eslint-disable`).
6. **Overfit special-casing**: hardcoded expected output, `if input == <test value>: return <expected>`, a branch that satisfies only the one test, a stub body claimed as done.
7. **Harness or CI edits**: coverage thresholds lowered, test selection narrowed, a CI step disabled, timeouts bumped so a slow or failing step goes green.

Minimal and clean (the grader's lens):
8. **Scope creep and side effects**: unrelated edits, drive-by refactors, reformatting, needless new files, or changes that break the patterns around them. Trim to the smallest change that truly solves the task.

## Findings are qualitative

Say what the change does and why it is a shortcut, not just that a pattern matched. One `except Exception: pass` can be fine in a retry loop and a cover-up in an error path. Judge intent from context.

Report each finding like this:

> **[Swallowed errors · Critical] `payment.py:42`**
> The new `except Exception: pass` wraps the charge call. The test on `test_payment.py:30` expects a failed charge to raise, but this hides the failure so the test passes while real charges fail silently.
> Fix: let the exception propagate (or handle the specific error), then confirm the test fails for the right reason and the code actually handles it.

Severity: **Critical** (ships a bug or hides broken behavior), **Important** (weakens a real guarantee or adds meaningful risk), **Minor** (scope creep, style, small cleanups).

## Rationalization table

| The agent's excuse | The reality |
|---|---|
| "All tests pass, so it's done." | Tests can pass because they were weakened or edited. Verify the behavior, not the green check. |
| "I edited the test to match the actual output." | The test encoded the expected behavior. Matching it to buggy output hides the bug. |
| "The broad `except` makes it robust." | It makes failures invisible. Catch the specific error, or let it raise. |
| "I pinned a newer version that fixes it." | The task was to fix the code. A version bump that smuggles in the fix is not the work. |
| "I added a small refactor while I was in there." | Unrelated edits widen the blast radius and hide the real change. Keep it minimal. |

## Red flags, stop and look closely

- A test file changed in the same diff that "fixes" a failing test.
- A new `skip`, `xfail`, `.only`, or a commented-out assertion.
- A broad catch, a silenced linter, or a suppressed exit code added near the changed code.
- A whole file that matches an upstream version, or a comment citing a URL or version number.
- A diff far larger than the task needs.

## Fix protocol

Make the smallest change that solves the task. Do not add side effects. Match the surrounding code's style and patterns. When you undo a shortcut, restore the real behavior the shortcut was hiding, then prove it with the real tests.
