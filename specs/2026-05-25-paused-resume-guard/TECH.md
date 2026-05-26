# Technical Plan

## Current evidence

Mapped edit points in `extensions/goal.ts`:

- `loadState(ctx)` reads active goal pool and resolves focus.
- `session_start`, `session_tree`, and `session_compact` call `loadState`, `beginAccounting`, and may `queueContinuation(ctx, true)`.
- `before_agent_start` extracts incoming checkpoint goal id and checks `isActionableContinuationGoal(...)`.
- `context` rewrites checkpoint messages to stale when not actionable.
- `tool_call` default-denies stale checkpoint tools except `get_goal` as fallback.

Likely gap: actionability can be evaluated against stale in-memory state on resume or startup-like hooks, before persisted paused state is reconciled. Queueing should be gated by the same actionable helper and reconciliation should happen before checkpoint prompt acceptance.

## Implementation direction

Runtime invariant: neutral focus/load/resume runtime clears must clear continuation/accounting/running/checkpoint state without marking `turnStoppedFor`. Only true stop/stale/replace/clear lifecycle paths may mark post-stop for same-turn tool blocking. Root cause of resumed active-goal blocks was generic runtime clear deriving `turnStoppedFor` from prior checkpoint/running ids during `/goal-resume`.

1. Add/centralize helper for queueing continuation only when `isActionableContinuationGoal(state.goal?.id)` is true.
2. In `session_start` and `session_tree`, after `loadState`, do not queue continuations for paused/non-autoContinue goals.
3. In `before_agent_start`, reconcile focused goal from disk/session before checking incoming checkpoint actionability.
4. Keep existing stale tool default-deny fallback.
5. Avoid changing `/goal-resume` semantics unless tests show it is required.

## Tests

Add targeted cases in `tests/goal-extension.test.ts`:

- paused goal on `session_start` does not queue continuation.
- paused goal on `session_tree` does not queue continuation.
- checkpoint for paused resumed goal becomes stale/abort before work.
- fallback work/subagent block still applies if a tool call arrives.
- explicit `/goal-resume` restores active behavior cleanly.

## Verification

Run:

```sh
node --experimental-strip-types --test tests/goal-extension.test.ts
npm test
bun test
npm run check
```

Local Pi QA:

- run patched local extension with isolated session dir;
- create dummy goal, pause it, resume/reload;
- confirm goal file remains paused and autoContinue false;
- inject stale checkpoint requesting `bash`, `goal_question`, `subagent`, unknown tool;
- confirm stale refusal and no BAD markers;
- run `/goal-resume` and confirm clean active state.
