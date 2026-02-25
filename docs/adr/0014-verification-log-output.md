# 0014. Verification hook output in phase logs

**Date:** 2026-02-25
**Status:** Accepted

## Context

ADR 0013 introduced a generic verification hook (`VERIFY_COMMAND`) that runs
after each logically successful phase and normalizes its exit codes into
ClaudeLoop’s existing failure/retry semantics. In real-world use with tools
like `gate`, the verification command often emits structured, machine-readable
output (typically JSON) that is useful for:

- Debugging why a phase failed verification.
- Surfacing concrete test failures to the user alongside Claude’s own logs.
- Feeding rich verifier results into subsequent Claude retries (via the
  existing “retry context in prompt” mechanism, which reads from `phase-N.log`).

The initial implementation only interpreted the verification command’s exit
code. Its stdout/stderr were visible in the terminal but were not captured in
`phase-N.log`, so they were:

- Lost between runs (no durable record in `.claudeloop/logs`).
- Unavailable to retry prompts that are built from phase logs.

## Decision

ClaudeLoop now captures the verification command’s stdout and stderr into the
per-phase log in a structured, append-only section, without changing the
existing state model or config precedence rules.

### Capture semantics

- `run_verification_command "<phase_num>"` is extended to:
  - Create per-phase temporary files:
    - `.claudeloop/logs/phase-<N>.verify.out`
    - `.claudeloop/logs/phase-<N>.verify.err`
  - Run the verification hook via:
    - `CLAUDELOOP_PHASE_NUM` and `CLAUDELOOP_PHASE_TITLE` exported as before.
    - `sh -c "$VERIFY_COMMAND" >"$verify_out" 2>"$verify_err"`
  - Replay any captured output back to the user:
    - `stdout` is re-`cat`’d to stdout.
    - `stderr` is re-`cat`’d to stderr.
  - Preserve the original exit-code contract from ADR 0013:
    - `0` → verification passed.
    - `1` → gate/verification failure.
    - `2` → verification config/runtime error.
    - other non-zero → mapped to `3` (unexpected code).
  - Short-circuit as before when:
    - `DRY_RUN=true`; or
    - `VERIFY_COMMAND` is empty.

This keeps observable behaviour at the terminal boundary essentially unchanged
while giving ClaudeLoop a durable copy of the verifier’s bytes.

### Phase log integration

- A new helper `append_verification_log_section "<phase_num>" "<log_file>" "<rc>"`
  appends the captured verification output to the corresponding phase log
  (`.claudeloop/logs/phase-<N>.log`) if any output was produced.
- The section format is:

  - A header line with metadata:
    - `=== VERIFICATION START phase=<N> exit_code=<rc> time=<ISO8601> ===`
  - Optional `stdout` block when non-empty:
    - `--- verification stdout ---`
    - Raw verifier stdout bytes (e.g. JSON lines).
  - Optional `stderr` block when non-empty:
    - `--- verification stderr ---`
    - Raw verifier stderr bytes.
  - A closing marker:
    - `=== VERIFICATION END phase=<N> ===`

- `execute_phase` calls `append_verification_log_section` immediately after each
  `run_verification_command` invocation, in both of its “logical success”
  branches:
  - Claude exited `0` without permission errors.
  - Claude exited non-zero but `has_successful_session` returned true.

### State model

- The core state model remains unchanged:
  - `PHASE_STATUS_*`, `PHASE_ATTEMPTS_*`, `PHASE_START_TIME_*`,
    `PHASE_END_TIME_*`, `PHASE_DEPENDENCIES_*`, `PHASE_COUNT`,
    and `PHASE_NUMBERS` are still the only persisted per-phase state.
  - `PROGRESS.md` continues to encode just status, attempts, timestamps, and
    dependency info.
  - The JSON state file continues to track only `plan_file`, `progress_file`,
    `current_phase`, and an `interrupted` flag.
- Verification output is treated purely as **log data**, not as additional
  structured state:
  - It is appended to `phase-N.log` only.
  - It does not alter how `read_progress`, `read_old_phase_list`, or
    `detect_plan_changes` work.

## Consequences

**Positive:**

- Verification tools like `gate` can emit rich, machine-readable diagnostics
  that are:
  - Persisted in `.claudeloop/logs/phase-N.log`.
  - Available to subsequent Claude retries via the existing “retry context”
    prompt injection, without any new parsing hooks.
- Developers get a single, consolidated log per phase that includes:
  - Claude prompt and response.
  - Stream-processor events.
  - Verification command output and exit code markers.
- The change is backwards compatible with the ADR 0013 contract:
  - Exit-code mapping and retry semantics are unchanged.
  - `VERIFY_COMMAND` remains optional and is still disabled by default.

**Negative:**

- Phase logs grow slightly larger due to the additional verification sections,
  especially for verbose verification tools.
- The relative interleaving of Claude output and verification output is now
  serialized: verification output always appears after the
  `=== EXECUTION END ... ===` footer, rather than streaming live into the same
  pipe.
- Implementations that parse `phase-N.log` must be aware of the new
  `VERIFICATION START/END` markers if they previously assumed only
  prompt/response content; however, existing tools that simply treat the log as
  free-form text remain unaffected.

