# Technical Plan

## Current evidence

Relevant code in `extensions/goal.ts`:

- `queueContinuation(ctx, force)` schedules a timeout and sets `continuationScheduledFor`.
- `sendQueuedContinuation(ctx, goalId)` checks active status before sending a follow-up checkpoint.
- `filter_context` rewrites older checkpoint messages to stale when the queued goal is not current/latest.
- `tool_call` blocks later tool calls after `turnStoppedFor` is set.
- `clearGoalTurnRuntimeState()` now clears continuation timer/queue, accounting, runningGoalId, and checkpointGoalId on stop/replace/focus-loss transitions.
- Stale checkpoint tool blocking should cover full goal work-tool set except `get_goal`.

The reported issue suggests a checkpoint/follow-up can be delivered after stop/replacement and still start a turn before the guard has enough context to prevent first work.

## Minimal implementation direction

1. Add a small helper such as `isActionableContinuationGoal(goalId)`:
   - current `state.goal` exists
   - ids match
   - status is `active`
   - `autoContinue` is true

2. Use helper in:
   - `sendQueuedContinuation`
   - `queueContinuation`
   - `filter_context` staleness check, if needed
   - turn start or continuation prompt handling if a current-message checkpoint for an inactive goal can still appear

3. On stop-ish transitions, ensure `clearContinuationState()` runs:
   - pause
   - abort
   - complete
   - replace/tweak paths where old goal should not continue

4. Add logging/diagnostic event only if existing test patterns support it.

## Test plan

Add/extend tests around `goal.ts` behavior:

- active autoContinue goal can queue continuation.
- paused/stopped goal does not deliver queued continuation.
- completed/aborted goal does not deliver queued continuation.
- replaced goal makes old queued checkpoint stale/non-actionable.
- stale checkpoint must not cause progress tool calls/subagent starts.
- mid-turn stale checkpoints must block all work-capable goal tools except safe read-only inspection (`get_goal`).

Run:

```sh
npm test
npm run check
```

## PR shape

Small PR title:

`fix(goal): ignore queued continuations for stopped goals`

PR body should be concise:

- explain zombie checkpoint race
- mention clear continuation state on stop/replacement
- mention tests
