# Test Loop Protocol (harness-neutral)

The portable core of autonomous, tested implementation. Both `/implement` (Claude Code)
and `$implement` (Codex) read and follow THIS file verbatim for the testing phase, so the
impl⇄test loop behaves identically across harnesses. The harness-specific wrappers own
project setup, task splitting, and the spawn/wait mechanics; this file owns everything from
"a single task's implementation is committed" onward.

## Vocabulary

- **task-runner** — the per-task agent (one spawn per task). Owns the loop below. Its context
  is disposable: only its compact final summary propagates up to the orchestrator.
- **impl agent / impl-fix agent** — sub-agents the task-runner spawns to write or fix the
  implementation. They never write test code.
- **test-author agent** — sub-agent that writes the ad-hoc test overlay and builds.
- **overlay** — the throwaway `#ifdef _DEBUG` test code for the current task. Never part of an
  implementation commit. Lives as a patch under the task folder between rounds.
- **golden tdata** — a read-only backup of the authed test account. Tests only ever copy FROM
  it; they never write to it.

## Inputs the wrapper passes in

- `TASK_DIR` — `.ai/<project>/<letter>/` for this task.
- `TASK_ID` — stable id used in commit trailers (e.g. the project + letter).
- **TASK SPEC** — the task's full description block (from `implementing.md`) and its referenced
  images (`images/<file>` design mockups / screenshots / graphic resources for this isolated task).
  This is half of what the tests are designed against (the diff is the other half); the design READS
  the images — they show what the result should look like.
- Config: `BUILD` (build command), `EXE` (built binary path), `MAX_ATTEMPTS` (default 4). The test
  account lives in `out/Debug/` as the portable-data folders described under "Test account" below;
  the wrapper has already confirmed the golden one exists (launch gate). All paths are relative to
  the current checkout — no worktrees are created; the run happens in whatever repository slot it
  was launched from.

## State machine (run by the task-runner)

Precondition: the implementation for this task is committed in the current checkout (impl agents
commit; they do not stash). Record that commit's SHA as **IMPL_SHA** — the reset after each test run
returns the checkout to exactly it. The runner tracks the attempt number as its own state (`attempt`
starts at 1); the commit message carries no attempt marker. Commits follow "Commit message" below.

```
TEST_AUTHOR -> RUN -> ASSESS (adversarial — see "Assessing"):
  APPROVED       -> reset to the impl commit (drop overlay); delete the test binary; return DONE up.
  TEST_FLAW      -> fix the overlay only; back to RUN. Does NOT cost an impl attempt.
  IMPL_BUG       -> spawn impl-fix agent (input = test.md, latest attempt's Root cause / Fix hint);
                    it commits a NEW attempt; re-apply overlay (--3way, else re-author); RUN. attempt++
  UNRECOVERABLE  -> delete the test binary; return BLOCKED up with the reason. Stop.
  attempt > MAX  -> delete the test binary; return BLOCKED up with test.md + "improve" notes. Stop.

On every TERMINAL exit (APPROVED / BLOCKED / UNRECOVERABLE / cap) "delete the test binary" means the
step in "Leave no test binary behind" below.
```

Early-escalation rule: if two consecutive ASSESS rounds produce the **same failure signature**
(same step fails the same way after a fix), stop and return BLOCKED — do not burn the rest of
the attempt budget chasing it.

UNRECOVERABLE conditions: the app reaches a login screen / `AUTH_KEY_DUPLICATED` and re-copying the
test account does not recover it; a file-lock build error (`LNK1104`, `C1041`) that persists after
the path-scoped kill; `test_TelegramForcePortable` missing when SETUP runs; or a crash with no usable
diagnostic after