---
name: check-inbox
description: "Check for tasks from the hub or external producers. Run at session start to see pending work."
command: /check-inbox
invocable: both
allowed-tools: Bash, Read
---

# Check Inbox for Pending Tasks

When invoked, follow these steps precisely:

## 1. Read Bus Configuration

Read `.claude/bus-config.json` in the current project directory to get the `session_name`.

If the file does not exist, report an error:
"No bus-config.json found. This project is not configured as an AI bus client. See the bus client setup guide."

## 2. Read Inbox Messages

Run:
```
bash ~/.ai-bus/lib/bus-read.sh {session_name} --format table
```

## 3. Handle Empty Inbox

If no messages are returned:
"Inbox empty. No pending tasks."

Stop here.

## 4. Display Messages

If messages exist, show a numbered list:

```
INBOX ({count} messages)
════════════════════════════════════════════════════════
  # │ Type     │ From │ Subject              │ Priority
────┼──────────┼──────┼──────────────────────┼─────────
  1 │ task     │ hub  │ Fix form validation  │ normal
  2 │ question │ hub  │ What ES version?     │ normal
```

## 4b. Triage & claim stale (inbox hygiene — do not skip)

The inbox is not auto-pruned: handled/old messages linger and turn it into a
graveyard where stale items outnumber real ones. Before presenting, claim the dead ones.

For each distinct `correlation_id` in the inbox, decide if its work is already done:
- TheLink task `status: completed`, or the `feature_branch` is merged / PR closed, or the plan file says DONE/QA-passed → **stale**.

Reap every stale correlation (moves ALL its envelopes to `processed/` in one call):
```bash
bash ~/.ai-bus/lib/bus-reap.sh {session_name} --corr {correlation_id} --reason "stale: already completed/merged"
```
Also `--claim` any empty / ack-only / plain-text noise.

Then re-read the inbox and present ONLY what remains. Report e.g. "claimed 16 stale,
2 open." **Never present (or let the user pick) a `proceed-to-implementation` whose
task is already done** — reap it instead; acting on it re-implements completed work.

## 5. Ask User to Select

Ask the user which of the **remaining open** tasks to work on:
- Enter a number to select a specific task
- Enter "all" to work through them sequentially

## 6. Claim the Selected Message

When a task is selected, claim it:
```
bash ~/.ai-bus/lib/bus-read.sh {session_name} --claim {msg-id}
```

## 7. Present the Task

Show the full task description from the message payload, including:
- The subject
- The full description/body
- Any context provided
- The correlation ID (needed for reporting back)

## 8. Begin Work

After presenting the task, begin working on it. The user can use `/report-back` at any time to send results to the hub.

## 9. Reconcile the reply-watch schedule (relayable-messaging)

Claiming messages in step 6 already updates the reply-correlation ledgers automatically
(`bus-read.sh --claim` closes an `awaiting/` debt when you claim a reply, and opens an
`owing/` debt when you claim a `reply_expected` request). After triage, let the
**agent-owned reply-watch schedule** self-adjust — arm if you now owe/are-owed a reply,
back off while quiet, and **tear itself down** once both ledgers are empty:

```bash
python3 ~/.ai-bus/lib/reply-watch.py reconcile
```

This is what makes an idle worker keep checking *only while a reply is in flight* and
stop (zero idle cost) the moment the conversation is settled. Safe to run every time;
it's idempotent. If you OWE a reply (an `owing/` marker exists), send it with
`/report-back` (or `bus-send.sh … --in-reply-to <correlation_id>`) — that closes the debt.

## 10. Consume the TheLink plane too (not just the bus)

The steps above drain the **filesystem bus**. The hub also reaches you over **TheLink**
(`send_message` + task comments), and an idle session never auto-pulls those. So also run:

```
/thelink-task-protocol consume
```

It pulls your unread TheLink messages (`list_messages` → `get_message` → `mark_message_read`)
and checks your current task comments for a hub response (latest comment not authored by you =
your turn). Without this, hub TheLink messages sit unread until someone types into your terminal.
Run it here, at session start, and on every self-wake. (Only you can read your own message
inbox — the hub cannot pull it for you, which is why this is worker-side.)
