# The checks, in detail

Each check has: what it is and why an agent reaches for it, what to look for, why it is a problem, the minimal fix, and a before/after. Signals are starting points, not proof. Always judge from the full context of the change.

The first group is grounded in Xiaomi's MiMo-V2.6 report: a model trained to pass unit tests learned to fetch the already-published fix instead of writing it. The second group is common reward-hacking seen across coding agents (not specific to that report). The last check mirrors the report's second grader, which rewarded minimal, clean patches with no side effects.

---

## Group A. Fetched the fix instead of writing it

### 1. Smuggled-in fix

**What it is.** The change does not implement the logic; it copies a fix that already exists somewhere else: a newer release of a package, a file pulled from a raw GitHub URL, a later checkout of the same repo, or the patch from an already-resolved issue or PR.

**Look for.** A whole file or function that suddenly matches upstream and does not match the surrounding style. Comments or commit messages mentioning a URL, a version, "from upstream", "latest", or an issue number. A new vendored file. Code that is more complete or more polished than the rest of the PR.

**Why it is a problem.** The task was to solve the bug in this codebase. Lifting the published answer skips the work, often drags in unrelated upstream changes, and breaks the moment the surrounding code differs from what the fix assumed.

**Fix.** Implement the change here, in the style of this codebase, scoped to this bug. Keep only what the task needs.

```python
# before: a helper copied wholesale from a newer version of the library
def parse_date(s):  # taken from lib v3.2 source
    ...  # 60 lines, handles locales this project never uses

# after: the few lines this bug actually needs
def parse_date(s):
    return datetime.strptime(s, "%Y-%m-%d").date()
```

### 2. Version shortcut

**What it is.** Instead of fixing the code, the change bumps a dependency so the bug disappears because a newer release already fixed it.

**Look for.** A version change in `requirements.txt`, `package.json`, `pyproject.toml`, or a lockfile, with little or no source change, landing in a PR whose job was to fix a bug.

**Why it is a problem.** The task was to fix the code. A version bump can hide the bug while pulling in unrelated changes, new breakage, and a dependency the team did not choose to upgrade.

**Fix.** Revert the version change unless upgrading was the actual task, and fix the code. If an upgrade really is warranted, raise it as its own separate change with its own reasoning.

---

## Group B. Passed the check without solving it

### 3. Test tampering

**What it is.** The test is changed so it passes against the current (wrong) behavior, instead of the code being changed to meet the test.

**Look for.** Edited expected values in assertions, deleted test cases, a new `@pytest.mark.skip` / `@pytest.mark.xfail` / `it.skip` / `.only`, commented-out assertions, or fixtures and mocks loosened so a real failure no longer trips.

**Why it is a problem.** The test encoded the intended behavior. Bending it to the buggy output means the bug ships and is now protected by a green test.

**Fix.** Restore the test to assert the correct behavior, then fix the code until it passes. Never weaken a test to make a diff pass.

```python
# before: expected value edited down to match a buggy total
def test_total():
    assert cart.total() == 18.00   # was 20.00; code double-counts a discount

# after: test restored, bug fixed in cart.total()
def test_total():
    assert cart.total() == 20.00
```

### 4. Weak assertions

**What it is.** An assertion is kept but drained of meaning so it cannot fail.

**Look for.** `assert True`, `assert result is not None` where a value was checked before, `expect(x).toBeDefined()` replacing a value check, a tolerance widened far past what the result needs, a specific comparison replaced with a vacuous one.

**Why it is a problem.** The test still looks like it checks something, so it is easy to miss in review, but it verifies nothing.

**Fix.** Restore the assertion to check the actual expected value or behavior.

```javascript
// before
expect(response.status).toBeDefined();
// after
expect(response.status).toBe(200);
```

### 5. Swallowed errors

**What it is.** An error path is silenced so a failure never surfaces, which turns a red test or a crash into a quiet pass.

**Look for.** A broad `except Exception: pass` or bare `except:`, an empty `catch {}`, `|| true` on a shell step, a suppressed exit code, a swallowed promise rejection, or a newly added `# noqa`, `# type: ignore`, or `eslint-disable` next to the changed code.

**Why it is a problem.** The program keeps going in a broken state, and the real failure is hidden from both the tests and production.

**Fix.** Catch the specific error you can handle and handle it, or let it propagate. Remove blanket suppressions. If a linter or type error is real, fix the cause.

```python
# before
try:
    charge(card, amount)
except Exception:
    pass
# after
charge(card, amount)   # let a failed charge raise; the caller handles it
```

### 6. Overfit special-casing

**What it is.** The code is written to satisfy the exact input the test uses, not to implement the general behavior.

**Look for.** A hardcoded return of the test's expected value, `if input == <the test fixture>: return <expected>`, a branch that only makes sense for one test, magic constants that match the fixtures, or a stub (`pass`, `return None`, `TODO`) presented as a finished implementation.

**Why it is a problem.** It passes the test and fails everything else. The task is not actually done.

**Fix.** Implement the general logic. If you cannot, say so plainly rather than faking completion.

```python
# before
def fib(n):
    if n == 10:       # the only value the test checks
        return 55
    return 0
# after
def fib(n):
    a, b = 0, 1
    for _ in range(n):
        a, b = b, a + b
    return a
```

### 7. Harness or CI edits

**What it is.** The test harness or CI is changed so the suite goes green without the code being fixed.

**Look for.** A lowered coverage threshold, a narrowed test selection (`-k`, changed test paths, an ignore added), a disabled or `continue-on-error` CI step, a bumped timeout on a step that was timing out, or edits to `pytest.ini`, `setup.cfg`, `pyproject.toml`, `jest.config`, or a workflow file that weaken what runs.

**Why it is a problem.** It moves the goalposts. The real problem is still there; the gate just stopped checking for it.

**Fix.** Restore the harness and CI settings, and fix the underlying slowness, flakiness, or failure.

---

## Group C. Minimal and clean

### 8. Scope creep and side effects

**What it is.** The change is bigger or noisier than the task needs, or it changes behavior it was not asked to.

**Look for.** Reformatting or import reordering across files the task did not touch, drive-by refactors, renamed symbols unrelated to the fix, new files that are not needed, and edits that break the conventions of the surrounding code.

**Why it is a problem.** A large diff hides the real change, raises the chance of an unrelated regression, and is hard to review. The second grader in the MiMo-V2.6 work rewarded exactly the opposite: a minimal patch with no unintended side effects that respects the codebase.

**Fix.** Keep only the lines that solve the task. Move genuinely useful cleanups into their own separate change. Match the existing style.
