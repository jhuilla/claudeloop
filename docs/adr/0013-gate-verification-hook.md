# 0013. Gate-compatible verification hook

**Date:** 2026-02-25
**Status:** Accepted

## Context

ClaudeLoop currently trusts each successful phase execution as long as the Claude
CLI session itself looks healthy. For larger models this is often sufficient, but
smaller or cheaper models sometimes fail to run all the checks they're asked to
run (tests, typecheck, lint, build, etc.), even when prompted explicitly.

For users who already have a structured verification tool such as
[`gate`](https://github.com/jhuilla/gate), it is desirable to make that tool a
mandatory harness-level gate between phases, without changing the default
behavior for users who do not use such a tool.

Gate defines a stable, Unix-friendly exit-code contract
([PLAN.md](https://github.com/jhuilla/gate/blob/main/PLAN.md)):

- `0` — all gates passed
- `1` — one or more gates failed
- `2` — config or runtime error in Gate itself (infrastructure problem)

ClaudeLoop needs a way to optionally call Gate (or any similar tool) after each
phase and interpret these exit codes without hard-coding Gate-specific logic or
breaking existing workflows.

## Decision

Introduce a generic, optional verification hook command that runs after each
logically successful phase and participates in the existing config precedence
chain.

### VERIFY_COMMAND setting

A new string setting `VERIFY_COMMAND` is added with the following semantics:

- When **empty** (the default), verification is disabled and ClaudeLoop behaves
  exactly as today.
- When **non-empty**, `VERIFY_COMMAND` is treated as a shell command string to
  execute as a mandatory verification step **after each successful phase**.

It participates in the same four-layer precedence chain defined in ADR 0008:

1. **Defaults** — in `claudeloop`, `VERIFY_COMMAND` defaults to the empty string.
2. **Config file** — `.claudeloop/.claudeloop.conf` may contain:
   - `VERIFY_COMMAND=...`
3. **Environment variables** — `VERIFY_COMMAND` can be set in the environment:
   - `VERIFY_COMMAND="npx gate claude bundle pr" claudeloop --plan PLAN.md`
4. **CLI arguments** — highest precedence via a new flag:
   - `--verify-command "<shell command>"`

The config file is updated following the existing pattern:

- On first run (non–dry-run), the active `VERIFY_COMMAND` value is written to
  `.claudeloop/.claudeloop.conf` alongside other persisted keys.
- On subsequent runs, when `--verify-command` is passed, only the
  `VERIFY_COMMAND=` line is updated.

### CLI surface

ClaudeLoop adds a new persistent CLI option:

- `--verify-command <cmd>` — set the verification command to run after each
  successful phase (e.g. `--verify-command "npx gate claude bundle pr"`).

This flag is:

- Parsed in `parse_args` similar to `--max-retries`.
- Included in `usage()` help text with a short explanation and Gate example.
- Persisted to `.claudeloop/.claudeloop.conf` when provided, unless running with
  `--dry-run`.

### Verification hook execution

A helper function `run_verification_command` is introduced in `claudeloop`:

- Signature: `run_verification_command "<phase_num>"`
- Behavior:
  - **Dry run**: if `DRY_RUN=true` it returns `0` without executing anything.
  - **Disabled**: if `VERIFY_COMMAND` is empty, it returns `0`.
  - **Enabled**: otherwise it:
    - Exports lightweight context variables for the child process:
      - `CLAUDELOOP_PHASE_NUM` — the current phase number (e.g. `2.5`)
      - `CLAUDELOOP_PHASE_TITLE` — the phase title string
    - Invokes the command via:
      - `sh -c "$VERIFY_COMMAND"`
    - Captures the exit code and maps it as described below.
    - Prints a short summary via `print_success` / `print_warning` /
      `print_error` in addition to whatever the verification tool prints.

The hook is called from `execute_phase` **only in branches where ClaudeLoop
considers the phase logically successful**, i.e.:

- The Claude CLI exited 0 and no permission error was detected; or
- The Claude CLI exited non-zero but `has_successful_session` determined that
  the session completed successfully.

In both cases, the phase is temporarily treated as a candidate for completion;
the verification hook then decides whether it is actually accepted.

### Exit-code mapping (Gate contract)

`run_verification_command` interprets the verification command’s exit code
according to Gate’s contract and returns a small set of normalized codes to
its caller (`execute_phase`):

- **0** — verification passed:
  - Raw exit code 0 from the command, or verification disabled/dry-run.
  - `run_verification_command` returns `0`.
- **1** — gate/verification failure:
  - Raw exit code 1 from the command.
  - Treated as a **phase failure** at the harness level.
  - `run_verification_command` returns `1`.
- **2** — verification config/runtime error:
  - Raw exit code 2 from the command (Gate’s “config/runtime error” case).
  - Treated as an **infrastructure error**.
  - `run_verification_command` returns `2`.
- **3** — unexpected exit code:
  - Any other non-zero exit code is mapped to `3`.
  - ClaudeLoop logs a clear error (including the raw code) and treats this as
    a generic verification failure. For now, `execute_phase` maps this back
    into the normal failure/retry path (equivalent to `1` from the caller’s
    perspective), but the distinct `3` return value is available if we need to
    refine behavior later.

### Integration into execute_phase and main_loop

`execute_phase` is updated so that:

- After a logically successful Claude session, it calls
  `run_verification_command "$phase_num"`.
- Only when the verification hook returns `0` does it:
  - Mark the phase as `completed` via `update_phase_status`.
  - Write progress via `write_progress`.
  - Print “Phase N completed successfully”.
  - Return `0`.
- When the verification hook returns `1` (gate failure) or `3` (unexpected
  exit code):
  - It logs an appropriate error.
  - Marks the phase as `failed`.
  - Writes progress.
  - Returns `1` so that `main_loop` applies the existing retry/backoff logic.
- When the verification hook returns `2` (infrastructure error):
  - It logs an infrastructure-specific error message explaining that the
    external verification tool or its config is broken.
  - It leaves the phase status as-is (the last attempt is recorded as
    `in_progress`; on a subsequent run, `read_progress` normalizes stale
    `in_progress` back to `pending`).
  - It returns `2` to signal an unrecoverable verification failure.

`main_loop` is updated to distinguish these cases explicitly:

- It captures the exact return code from `execute_phase` into a local variable
  (e.g. `phase_rc`).
- When `phase_rc == 0`:
  - Behavior is unchanged (move on to the next phase).
- When `phase_rc == 2`:
  - It prints a clear message explaining that the verification hook failed with
    an infrastructure/config error (exit 2) and that the user must fix the
    external tool (e.g. Gate) or its configuration.
  - It returns `2` from `main_loop`, which propagates up as the ClaudeLoop
    process exit code `2`.
- For all other non-zero codes:
  - It follows the existing error-handling path: quota detection,
    empty-log detection, permission errors, exponential backoff, and retry
    budget, eventually returning `1` when retries are exhausted.

This yields the following top-level process exit semantics for ClaudeLoop when
verification is enabled:

- `0` — all phases and their verification hooks passed.
- `1` — at least one phase (or its verification hook) failed after exhausting
  retries.
- `2` — verification tooling/config itself is broken; user intervention is
  required before retrying.

## Consequences

**Positive:**

- Makes it easy to enforce that critical checks (tests, typechecking, linting,
  builds, etc.) are actually run, by delegating to a single verification
  command such as `gate claude bundle pr`.
- Aligns with Gate’s stable exit-code contract without hard-coding Gate
  specifics, allowing other tools to be plugged in as long as they respect the
  same `0/1/2` semantics.
- Preserves existing behavior for users who do not set `VERIFY_COMMAND` — the
  default remains unchanged.
- Keeps configuration consistent with the rest of ClaudeLoop via the existing
  defaults → config → env → CLI precedence chain.

**Negative:**

- Adds another moving part to the execution pipeline; misconfigured verification
  commands can now block progress entirely (by design).
- Some users may be surprised that a non-zero exit code from the verification
  hook causes retries and failures even when the Claude phase itself succeeded.
- The mapping for unexpected exit codes (mapped to a generic failure) is a
  policy choice; if different tools adopt different conventions, we may need to
  extend the contract or allow more customization in the future.

