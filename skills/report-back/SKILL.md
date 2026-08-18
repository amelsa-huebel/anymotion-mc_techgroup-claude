---
name: report-back
description: "Send results, status, or findings back to the hub. Usage: /report-back <type> <subject> [details]"
command: /report-back
invocable: both
allowed-tools: Bash, Read
---

# Report Back to Hub

When invoked, follow these steps precisely:

## 1. Read Bus Configuration

Read `.claude/bus-config.json` in the current project directory to get `session_name` and `hub` name.

If the file does not exist, report an error:
"No bus-config.json found. This project is not configured as an AI bus client."

## 2. Parse Arguments

Parse `$ARGUMENTS`:
- First word: **type** — must be one of: `completion`, `status`, `finding`, `error`
- Second word: **subject** — short summary
- Remaining words: **details** — additional information

If type is not one of the valid types, show the valid options and ask the user to retry.

## 3. Build Details

Construct a JSON payload:

- For **completion**: Include a summary of the work done in the current session. Gather key changes, files modified, and outcomes.
- For **status**: Include current progress, what's done, what's remaining.
- For **finding**: Include the discovery, its implications, and any recommendations.
- For **error**: Include the error details, what was attempted, and suggested remediation.

## 4. Find Correlation ID

Check the `processed/` directory for the most recently claimed task message:
```
ls -t ~/.ai-bus/sessions/{session_name}/processed/ | head -1
```

If a processed message exists, read it and extract the `id` field to use as the correlation ID. This links the response back to the original task.

## 5. Send the Message

If a correlation ID was found:
```
bash ~/.ai-bus/lib/bus-send.sh --from {session_name} --to {hub} --type {type} --subject "{subject}" --body '{details_json}' --correlation-id {correlation_id}
```

If no correlation ID (spontaneous report):
```
bash ~/.ai-bus/lib/bus-send.sh --from {session_name} --to {hub} --type {type} --subject "{subject}" --body '{details_json}'
```

## 6. Confirm

Output:
"Reported to hub: {type} -- {subject}"

If this was a completion report, suggest:
"Run `/check-inbox` to see if there are more tasks waiting."
