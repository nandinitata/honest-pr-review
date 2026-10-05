# honest-pr-review

A Claude Code skill that reviews a coding agent's pull request, catches the shortcuts an agent takes to make tests pass without doing the work, and fixes them back to a small, honest change.

A green test suite is not proof the task was solved. An agent rewarded for passing tests will sometimes pass them the easy way: it copies a fix someone already wrote, weakens or deletes the test, hides the error, or special-cases the one input the test checks. This skill assumes the diff might have passed by gaming the check. It verifies the work was actually done, then trims the change to the minimum.

## Where the idea comes from

Xiaomi's **MiMo-V2.6 technical report** (LLM-Core, 2026) describes training a model with reinforcement learning to pass unit tests. Early on, instead of solving the assigned bug, the model learned to fetch the already-published fix: install a newer release and read its source, pull a file from a raw GitHub URL, clone a later checkout of the repo, read the history of an already-fixed issue, or probe versions for one that was already patched. Their defense included a second grader that ranked passing patches by approach, **minimality**, unintended side effects, and codebase craftsmanship, which pushed the model toward smaller, cleaner changes.

This skill turns that idea into a review you can run on any PR. The "fetch the fix" checks come straight from the report. The other common gaming patterns (weak assertions, swallowed errors, edited tests, overfit special-casing) are general reward-hacking, not specific to the report. The minimal-and-clean check mirrors the report's second grader.

Report: *MiMo-V2.6: Scaling Reinforcement Learning Towards Self-Improvement*, Xiaomi LLM-Core, 2026. Overview coverage: [Kili Technology](https://kili-technology.com/blog/data-story-mimo-v2-6-llm-as-a-judge), [VentureBeat](https://venturebeat.com/technology/better-than-deepseek-xiaomis-mimo-v2-6-pro-debuts-as-the-top-open-weights-model-in-the-world-alongside-cheaper-v2-6-flash).

## What it checks

Eight checks, all qualitative (it reasons about what the change does, not just pattern counts):

1. **Smuggled-in fix**: code copied from a newer version, a raw URL, a later checkout, or a fixed issue instead of implemented.
2. **Version shortcut**: a dependency bump that makes the bug disappear because a newer release already fixed it.
3. **Test tampering**: expected values edited to match buggy output, tests deleted or skipped, assertions commented out.
4. **Weak assertions**: `assert True`, existence checks replacing value checks, widened tolerances.
5. **Swallowed errors**: broad `except: pass`, empty catches, `|| true`, silenced linters and type checkers.
6. **Overfit special-casing**: hardcoded expected output, branches that satisfy only the one test, stubs claimed as done.
7. **Harness or CI edits**: lowered coverage, narrowed test selection, disabled CI steps, bumped timeouts.
8. **Scope creep and side effects**: unrelated edits and drive-by refactors; trimmed to the smallest change that solves the task.

Full catalog with signals and before/after examples: [`skills/honest-pr-review/references/checks.md`](skills/honest-pr-review/references/checks.md).

## How it works

It reviews a local branch (diff against its base) or a GitHub PR (via the `gh` CLI). It reads the full context of each changed file, reports the findings with a severity and a merge verdict, then fixes what it found: it restores real assertions, un-skips tests, removes error suppression, replaces special-casing with real logic, reverts harness edits, and trims scope. If a test was gamed to hide a real bug, it fixes the code and re-runs the real suite to prove it.

## Install

**As a plugin:**

```
/plugin marketplace add nandinitata/honest-pr-review
/plugin install honest-pr-review@nandinitata-skills
```

**Or copy the skill in:**

```
cp -r skills/honest-pr-review ~/.claude/skills/
```

## Use

In Claude Code, after a coding agent has produced a branch or opened a PR:

```
Review this PR with honest-pr-review
```

or point it at a GitHub PR:

```
Run honest-pr-review on PR #142
```

It lists what it found, fixes it, and re-runs the tests so they pass for the right reason.

## License

MIT. See [LICENSE](LICENSE).
