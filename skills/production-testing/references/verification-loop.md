# The Verification Loop — run → report bug → fix → rerun

The core operating procedure of this skill: **you do not claim a test passes, or that an API/feature
works, until you have run it and observed the result.** Reading code is not verification. This file
is the command playbook and the bug-report contract.

## The loop

```
┌─ IDENTIFY target + test type (API / unit / integration / component / hook / E2E)
│
├─ RUN the real command (Bash), capture stdout+stderr+exit code
│
├─ EVALUATE
│    GREEN + new test  → PROVE-IT-CAN-FAIL: break the assertion → rerun → confirm RED
│                        → restore → rerun → confirm GREEN → DONE
│    GREEN + existing  → DONE (report the command + result)
│    RED               → RAISE A BUG (template below). Triage: test-bug or product-bug?
│
├─ FIX the correct side (or hand the bug off if out of scope)
│
└─ RERUN the exact same command ──► back to EVALUATE   (repeat until GREEN)

If still RED after a reasonable fix attempt → STOP, escalate with the bug report.
Never disable / skip / .only() / loosen the assertion to force green.
```

**When a subagent does the fixing:** after it reports "fixed", **re-run the same verification command
yourself** (or in a fresh verification step) — trust the observed result, not the claim. This is the
"rerun the agent" step: the loop re-enters at RUN with the identical command until it is genuinely
green.

## Running each test type

Adapt the commands to the repo; the shape is what matters. Always run the **narrowest** command that
covers the change (full suites are too slow), then widen if needed.

**Backend — a single test / an API test** (run from the directory that owns its `pytest.ini`):
```bash
cd <component-dir>
pytest --reuse-db path/to/test_records.py::test_records_endpoint_permissions -q
# DB errors after a schema change? run once with --create-db, then go back to --reuse-db.
```

**Backend — a live API call** (when verifying a running endpoint, not just its test): call it with
the correct auth and assert the contract by inspection:
```bash
curl -sS -i -H "Authorization: Bearer <token>" "http://localhost:8000/api/records?segment=3"
# check: status line, JSON body shape, and that unauthorized/invalid-param variants behave.
```

**Frontend — unit/component/hook** (from the frontend project dir):
```bash
npx jest -c ./jest.config.js path/to/Component.test.tsx -t "renders results"
```

**E2E — one spec** against a target environment:
```bash
APP_NAME=<app> APP_URL=<env-url> <e2e-runner> --spec 'path/to/feature/*.test.js'
```

**Prove-it-can-fail** (new tests only): make the assertion wrong (or point the selector at something
absent), rerun, confirm the run goes **RED**, then restore and confirm **GREEN**. A test you have
never seen fail is not yet a test.

## The bug report (what "raise the bug" means)

On a RED result, do **not** loosen the test. Produce a structured report so the failure is
actionable and the fix is targeted:

```
BUG: <one-line symptom>
Test type: API | unit | integration | component | hook | E2E
Command:   <exact command you ran>
Expected:  <the contract — status/body/rendered output/DB state>
Actual:    <what happened>
Evidence:  <the relevant traceback / assertion diff / response body excerpt — trimmed, not dumped>
Repro:     <minimal steps or the smallest failing input>
Triage:    TEST BUG (assertion/fixture/selector wrong) | PRODUCT BUG (code under test wrong)
Suspected cause: <your best hypothesis>
```

Then act on the triage:
- **Product bug** → fix the product code (or, if out of scope, hand off the report and stop — do not
  weaken the test to make CI green).
- **Test bug** → fix the test/fixture/selector so it correctly asserts the intended behavior.
- **Flaky (E2E)** → do not `.skip()` globally; use a branch-scoped override to test the fix, address
  the root cause (wait/selector/timing), and only mark it resolved once the fix is in production.

## Triage: is it a test bug or a product bug?

- The assertion encodes a **wrong expectation** (stale value, wrong status, bad fixture) → **test bug**.
- The code returns something that violates a **real, intended** contract (wrong status, missing/renamed
  field, N+1, crash) → **product bug**. Fix the product; keep the test.
- **Uncertain?** Re-derive the intended behavior from the spec/requirements, not from the current
  output. Do not "fix" the test to match buggy output — that bakes the bug in.

## Anti-patterns this loop exists to prevent

- Declaring "the tests pass" / "the endpoint works" from reading the diff. **Run it.**
- Loosening an assertion, adding `.skip()`/`.only()`, an env-gated skip, or a `try/except pass` to
  turn RED into green.
- Reflexively `--snapshot-update`ing a failing snapshot (that commits the regression as truth).
- Accepting a fixer subagent's "done" without re-running the verification command.
- Marking an E2E flake resolved before the fix ships to production (resets the counter while it's
  still flaky on the main branch).

## Doing this at scale (optional, agent-driven)

For a batch of endpoints/tests, drive the loop per item: run → on RED emit the bug report → fix or
hand off → **rerun the same command** → collect only the results that are genuinely green. Report the
survivors and the still-open bugs; never report an item as passing on the strength of a fix you did
not re-verify.
