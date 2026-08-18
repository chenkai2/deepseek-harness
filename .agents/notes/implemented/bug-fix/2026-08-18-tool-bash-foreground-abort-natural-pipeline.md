# Agent Note: Tool-bash foreground abort returns result through the normal pipeline

Status: implemented

English | [中文](2026-08-18-tool-bash-foreground-abort-natural-pipeline.zh.md)

## Problem

When a foreground bash command was aborted (caller signal, not executor timeout), the
`tool-bash` execute function threw an `AbortError` instead of returning the `ShellRunResult`
with `aborted: true`. The tool runtime's `dispatchToolBody` already handles cancellation
correctly: after the body returns, `isAborted(signal)` routes to `toolAbortedResult()`, which
commits a proper error result to the session log. But the throw bypassed that path, entering
the `catch` block instead.

The `catch` block calls `toolErrorResult()`, which also produces an `isError` result. However,
the throw changes the control flow through the scheduler: the dispatch promise resolves
differently, and in some edge cases (e.g., a tight race between signal and tool output
collection) the `tool/result` event could be omitted from the session log. The next request
would then send an `assistant` message with `tool_calls` but no corresponding `tool` message,
and the DeepSeek API rejects it with `400001: insufficient tool messages following tool_calls
message`. The session is stuck — every subsequent turn fails the same way.

## Decision

Remove the `if (result.aborted) { throw ... }` block from the foreground execute path. When
the caller's signal aborts a command, the `LocalBashExecutor.run()` method returns a
`ShellRunResult` with `aborted: true` (and `timedOut: false` — the two are mutually exclusive
by the deadline library's first-cause rule). The execute function returns that result normally,
and `dispatchToolBody`'s `isAborted(signal)` check after the `await` routes to the correct
`toolAbortedResult()` path. The result is a `ToolExecutionResult` with `isError: true` and
`content: "Error: tool call aborted"`, which `appendToolResult` records as a `tool/result`
event in the session log. The session surface is complete, and the next request proceeds
without the API error.

The background path (`run_in_background`) is not affected: it already has its own pre-flight
abort check that throws before any process starts, and the scheduler handles that through the
prepare-stage `toolAbortedBeforeDispatchResult` path, which correctly appends the synthetic
`tool/result` event.

## Verification

All 108 existing unit tests and 4 integration tests pass. The change is a straight deletion
of four lines (the `if` block); no new logic is introduced, so the existing timeout test
(`reports timeout kills with both markers`) and the pre-aborted background test
(`a pre-aborted call is skipped before the process starts`) continue to cover the abort
contract.

## Alternatives considered

**Keep the throw and fix the scheduler.** The scheduler's `commitReady` path already works
correctly for every other tool; the bug is specific to `tool-bash` injecting a throw into the
one path the tool runtime expects to return normally. Fixing the caller is simpler and removes
noise from the error classification.

**Catch in `dispatchToolBody` and force `toolAbortedResult`.** The catch block already routes
to `toolErrorResult`, which produces a similar `isError` result with the same content. The
difference is subtle (the `toolAbortedResult` path preserves `additionalContexts` from the
prior result, which the bash tool never sets), so this would not change the model-visible
output. But keeping the throw means the `isAborted(signal)` branch in `dispatchToolBody` is
dead code for the bash tool, and the scheduler's `toolAbortedResult` logic is never exercised
by the most common abort scenario.

## Consequences

Foreground bash commands that are aborted by the caller signal always produce a `tool/result`
event in the session log. The `tool-call aborted` error is visible to the model, and the
session remains usable for subsequent turns. The `dispatchToolBody` `isAborted(signal)` branch
is now exercised by foreground bash, matching the contract documented in the scheduler's
cancellation-result selection.
