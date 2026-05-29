# OpenClaw Restart Resume

`gateway-restart-resume` is a small OpenClaw skill for one practical failure mode: an agent is asked to restart the gateway, but the restart kills the same runtime that would normally send the follow-up reply.

The skill gives agents a safe protocol:

1. Acknowledge the restart request in the original medium.
2. Capture only the routing metadata needed to reply later.
3. Create a durable resume callback before restarting.
4. Restart the gateway only after the callback exists.
5. Verify gateway, health, and original medium recovery.
6. Send a structured post-restart status report back to the original conversation.

## Use Case

Use this when OpenClaw is controlled from Discord, Slack, Matrix, SMS, email, web chat, CLI relay, ACP, cron, or another gateway-backed medium and the user expects an answer after the gateway comes back.

Without this protocol, an in-gateway agent can truthfully start a restart but never return to say whether the restart worked.

## Install

```bash
clawhub install gateway-restart-resume
```

ClawHub:

```text
https://clawhub.ai/udaymanish6/gateway-restart-resume
```

## What It Does Not Do

- It does not restart OpenClaw by itself.
- It does not read secrets or env files.
- It does not replace an OpenClaw admin/update runbook.
- It does not assume Discord; Discord is only one supported medium.

## Community

If this solves a real restart-follow-up problem in your OpenClaw setup, please star it on ClawHub or GitHub so other operators can find it.
