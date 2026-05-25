# Paused Resume Guard

## Problem

A paused Goal-x goal should remain paused across Pi session resume, reload, and tree restore. Current behavior can still let old checkpoint context or stale in-memory state treat the goal as runnable, which causes a bad resume experience: stale checkpoint tool guards fire, normal inspection can be blocked, and the user loses the graceful paused state they expected.

## Goal

Make paused resume deterministic: paused goals stay paused, do not auto-continue, and old checkpoint messages for them are staled before work begins.

## User-visible behavior

- Resuming a Pi session with a paused goal shows the goal as paused, not running.
- No auto-continue checkpoint is queued or delivered for paused goals.
- Old checkpoint prompts for paused/stopped/replaced goals are stale before agent work starts.
- `/goal`, `/goal-resume`, and `/goal-clear` remain usable after resume.
- Explicit `/goal-resume` resumes cleanly and then follows the existing continuation behavior.

## Non-goals

- Redesign auto-continue.
- Change command names or drafting UX.
- Change auditor behavior.
- Introduce persistence migrations unless tests prove unavoidable.

## Acceptance criteria

- A paused goal loaded via `session_start` remains paused and does not queue continuation.
- A paused goal loaded via `session_tree` remains paused and does not queue continuation.
- A stale checkpoint for a paused resumed goal is aborted/staled before work starts.
- Tool-call blocking remains a fallback only; normal paused-state commands/inspection do not enter a stale cascade.
- Tests and local Pi resume QA verify the behavior.
