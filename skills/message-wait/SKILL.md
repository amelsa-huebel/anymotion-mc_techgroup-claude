---
name: message-wait
description: "Drain and wait on inbound TheLink direct messages reliably. Lists the inbox and filters unread on readAt (the unread_only client filter is unreliable), acts on each, marks read. When still awaiting an expected reply, arms a schedule heartbeat as the offline fallback to the webhook event-wake."
command: /message-wait
invocable: both
allowed-tools: Read, mcp__TheLinkMcp__whoami, mcp__TheLinkMcp__list_messages, mcp__TheLinkMcp__get_message, mcp__TheLinkMcp__mark_message_read, mcp__TheLinkMcp__list_schedules, mcp__TheLinkMcp__create_schedule
---

# Message Wait — reliable inbound-message drain + await

An idle Claude session does **not** auto-pull its TheLink message inbox. Unless it runs this, direct
messages sit unread. This skill is the **read-down** path for messages (task-comment turns are handled
by `task-plan-poll`).

> **`unread_only` is unreliable.** It's a client-side filter that has missed messages in practice
> (see the hub's TheLink-DM-pull note). **Always list and filter on `readAt` yourself.**

## Modes

`/message-wait <mode> [args]`

### `drain` — one-shot: read everything unread, act, mark read (any tier)
Run at **session start, on every wake, and after any inbox check.**
1. `whoami` → your contact id.
2. `list_messages({direction:"inbox"})` — list, do **not** trust `unread_only`.
3. For each message where `readAt` is null/absent:
   - `get_message({id})` for the full body.
   - **Act on it per your role's message-handling skill** (worker: treat as a directive from your
     orchestrator; orchestrator: interpret and advance-or-escalate). The latest message is authoritative.
   - `mark_message_read({id})` — **marking read IS the completion of the drain.** Never leave an
     acted-on message unread; never mark unread-but-unacted.
4. If nothing unread → report `"no unread messages"` and stop. Cheap; safe every tick.

### `await <what> [--every N]` — drain now, then guarantee a re-check (awaiting a reply)
Use when you've sent a request and are blocked on the answer. An agent can't truly block-loop, so
"waiting" = drain now + ensure something will wake you again.
1. Run `drain`. If the awaited message arrived → act and stop.
2. If not, check the wake coverage:
   - **Webhook event-wake** is the primary path — the sender's reply should trigger your window's
     inbound webhook. If that's wired, you're covered; stop.
   - **Schedule heartbeat is the fallback** for the offline-503 case (webhook is not queued when the
     agent is offline). If no heartbeat schedule exists (`list_schedules`), arm one:
     `create_schedule(steps=[{prompt:"/message-wait drain", key:"Enter"}], interval_minutes=N or 15, status="active")`.
3. Report what you're waiting on and which wake path is armed. **Never claim to "wait" without a
   concrete re-wake path** — a wait with no wake is a silent hang.

### `teardown-wait` — remove the heartbeat once the reply landed
When the awaited message has been handled and no other await is pending, pause/delete the heartbeat
schedule you armed so ticks don't run forever. (Self-teardown keeps idle ticks rare.)

## Reply-correlation (peer-to-peer)

If your project uses the reply-correlation protocol, a request carries an expectation id and the reply
carries `in_reply_to`. On `drain`, match replies to your outstanding expectations and close the
expectation the moment you act — reply **to the sender**, not to a hub star.

## Caller-scoping note

Only **you** can read your own inbox — the hub / an external gate cannot `list_messages` on your
behalf. That's why message consumption must be agent-driven and lives in this skill. (An external
gate can still wake you on *task comments*, which the hub can read — that's `task-plan-poll`.)

## Failure modes

- **Acted but forgot `mark_message_read`** — the message re-processes next tick (duplicate action).
  Marking read is not optional.
- **`await` with no webhook and no schedule** — that's a hang. Arm the heartbeat.
- **Heartbeat never torn down** — ticks accumulate. Run `teardown-wait` when the await resolves.
