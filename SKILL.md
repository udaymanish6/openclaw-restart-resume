---
name: gateway-restart-resume
description: Use when an OpenClaw agent is asked from any gateway-backed medium to restart, stop, reload, update, or repair the gateway and the agent must reply again after the gateway comes back.
---

# OpenClaw Restart Resume

## Overview

Gateway restart is a conversation-continuity problem. An in-gateway OpenClaw agent that restarts the gateway kills the runtime that would normally send the follow-up reply. Before taking any gateway-killing action, create a durable resume path that can verify recovery and report back through the same medium, conversation, channel, thread, or user route.

## When to Use

Use this skill when:

- A user asks through Discord, Slack, Matrix, SMS, email, web chat, CLI relay, ACP, cron, or another OpenClaw-backed medium to restart, stop, reload, update, or repair the gateway.
- The action may stop the gateway, restart the service manager, reload plugins, or replace the running OpenClaw package.
- The agent is running inside OpenClaw, or may be running inside OpenClaw.
- The user expects a post-restart status reply.

Do not use this skill for external shell operators that can survive the restart themselves, except as a checklist for user-facing notifications.

## Restart Resume Protocol

1. Classify execution context.
   - If running inside OpenClaw, assume the restart will terminate the current turn.
   - If running outside OpenClaw, still use the notification and verification steps when the request came from a user-facing channel.

2. Confirm permission.
   - If the user did not explicitly ask for restart/stop/update, ask first.
   - Do not restart for vague requests such as "fix it" until the restart is explained and approved.

3. Capture reply target.
   - Record the medium, account/profile if relevant, conversation id, channel id, thread id, message id, user id if needed, and agent id.
   - Prefer the exact original route. If the medium has no thread concept, record the smallest addressable conversation target.
   - Store only routing metadata needed to reply. Do not store raw message bodies, tokens, env contents, or secrets.

4. Send pre-restart acknowledgement.
   - Tell the user the gateway is restarting and that a follow-up will be sent after verification.
   - Keep it short because the gateway may stop immediately afterward.

5. Create durable resume callback before restart.
   - Prefer a one-shot or short-delay OpenClaw cron job, task, or native gateway callback that survives the gateway restart.
   - The callback must include the reply target and verification checklist.
   - If callback creation fails, do not restart. Tell the user the restart was not attempted because no follow-up path exists.

6. Restart only after the callback exists.
   - Use the narrowest approved restart action.
   - Avoid broad update/reinstall operations unless the user requested them and the current admin runbook supports them.

7. Resume callback verifies recovery.
   - Check gateway/service status.
   - Check health endpoint or gateway probe.
   - Check connectivity or send capability for the original medium when the local OpenClaw version supports it.
   - Check plugin or task health only when relevant to the restart reason.

8. Send final reply.
   - Reply in the original medium and conversation/thread when available.
   - Use the operator report format below unless the original medium requires a shorter message.
   - State whether restart completed, what was verified, duration if known, and any remaining warning.
   - If verification fails, report the failed check and the safest next action.

## Medium-Agnostic Reply Rules

- Discord/Slack/Matrix: reply in the original thread when available; otherwise reply in the original channel/conversation.
- SMS/chat apps: keep the final message compact and avoid multi-line technical detail unless the user asked for it.
- Email: include a concise subject-like first line, then the verification report.
- CLI relay or ACP: return structured status when the caller expects JSON/tool output; include human-readable summary when routed to a person.
- Cron or automation-triggered restarts: send the final report to the configured notification target and record the result in durable run history if available.
- If the original medium is unavailable after restart, use the configured fallback notification target only when policy allows it; otherwise leave a durable ops note for the operator.

## Resume Payload

The durable callback should carry a minimal payload:

```json
{
  "reason": "user-requested gateway restart",
  "agentId": "<agent-id>",
  "reply": {
    "medium": "<discord|slack|matrix|sms|email|web|cli|acp|cron|other>",
    "account": "<account-id-if-needed>",
    "target": "<conversation-or-addressable-target>",
    "channelId": "<channel-id-if-needed>",
    "threadId": "<thread-id-if-needed>",
    "userId": "<user-id-if-needed>",
    "messageId": "<original-message-id-if-needed>"
  },
  "verify": [
    "gateway status",
    "health",
    "original medium connectivity"
  ],
  "createdAt": "<iso-timestamp>"
}
```

Sanitize or omit fields that are not needed for routing. Never include secrets, full config, env files, transcript bodies, or private message contents.

## Verification Checklist

Use the local OpenClaw version's native tools when available. Equivalent checks are acceptable.

```bash
openclaw gateway status
openclaw health
openclaw channels status --probe
```

When the restart was part of an update or plugin repair, also verify the relevant narrow checks, for example:

```bash
openclaw doctor --non-interactive --no-workspace-suggestions
openclaw plugins doctor
openclaw tasks audit
```

Do not claim the restart succeeded unless fresh verification output supports it.

## Failure Handling

- Callback creation fails: do not restart; explain that there is no durable way to report back.
- Restart command fails before shutdown: report the command failure immediately in the current medium.
- Gateway returns but the original medium is disconnected: report through any still-available approved fallback route, or leave a durable task/ops note for the operator.
- Gateway does not return: the callback should record the failure in durable state and stop retrying after a bounded number of attempts.
- Auth/token errors appear: do not read or paste secrets. Report the config path/key names only and ask for explicit operator direction.

## Common Mistakes

- Restarting first and trying to schedule the follow-up afterward.
- Treating "gateway command returned" as proof that the original medium recovered.
- Storing raw messages or secret-bearing config in the resume payload.
- Creating a repeating cron when a one-shot callback is enough.
- Retrying forever after a failed restart.
- Sending the final reply to a default destination instead of the original medium/conversation/thread.

## Output Message Format

Prefer this operator report for human-facing media that can handle multi-line messages.

Before restart:

```text
Restarting the OpenClaw gateway now.
I created a resume check and will reply here after the gateway is back and verified.
```

After success:

```text
Gateway restart complete.

Verified:
- Gateway: <running/admin-capable/version if known>
- Health: <ok/degraded>
- Original medium: <connected/reply sent/not directly testable>
- Extra checks: <plugins/tasks/update checks or "not needed">

Duration: <duration or "not recorded">
Warnings: <none or short warning>
Next action: <none or operator action>
```

After degraded recovery:

```text
Gateway restart completed, but verification is degraded.

Passed:
- <passed check>

Needs attention:
- <failed or warning check>

I did not take further action automatically. Safest next action: <next step>.
```

After failure:

```text
Gateway restart did not verify cleanly.

Failed check: <check>
Last known state: <state>
Safest next action: <next step>
```
