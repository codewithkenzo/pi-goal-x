# Stopped Continuation Guard

## Problem

A queued Goal-x continuation/checkpoint can survive after the goal has already been stopped in the same turn. When that stale checkpoint starts, the agent may begin work (including spawning a builder) before the tool guard blocks later tool calls with:

> The goal was already stopped earlier in this turn ... Do not call more tools

This creates confusing "zombie checkpoint" behavior and can leave orphaned subagent work.

## Goal

Ensure queued continuations for stopped, paused, completed, aborted, or replaced goals never start a new work turn.

## User-visible behavior

- If a queued checkpoint targets a goal that is no longer the active auto-continuing goal, Goal-x should treat it as stale before work begins.
- No follow-up turn should be triggered for an inactive/stale goal.
- No subagent/tool work should start from a stale checkpoint.
- The user may still see a concise stale/invalid checkpoint notice when useful, but the agent should not try to recover in the same turn.

## Non-goals

- Redesign auto-continue.
- Change drafting flow.
- Change auditor approval/rejection behavior.
- Add new commands.

## Acceptance criteria

- Stopping or pausing a goal clears pending continuation timer/queue state.
- Replacing a goal prevents old queued messages from becoming actionable continuations.
- A queued checkpoint message for a non-active goal is rewritten/staled before it can trigger work.
- Tests cover stopped/paused/replaced stale continuation cases.
