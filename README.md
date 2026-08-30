# dead-mans-machine

An **agentic digital executor**: if you go silent for long enough, a machine wakes up, follows a playbook you wrote while alive, and tries to finish the job — publish projects, send letters, ask people for help when it gets stuck.

This repo is the idea and the first plan. There is no working software here yet.

## The gap

Existing dead man's switches (LastSignal, Aeterna, Google Inactive Account Manager, Sarcophagus, …) mostly **release a message or a file** after missed check-ins.

Existing agents (Hermes, OpenClaw, browser-use, …) can **do things**, but they assume you are still around to pay the bill, ping the box, and unstick them.

Nobody really combines the two: *inactivity → decrypt a sealed playbook → an agent executes it, and escalates to humans when APIs and browsers fail.*

That is the bet.

## What we are aiming at

Not “an immortal AI.” A **bounded execution window** after you disappear.

Typical jobs:

- email people you named
- make selected git repos public
- drop a diary / ideas / farewell notes
- ask a trusted human to finish what the machine cannot (the important part)
- optionally, later: post something you already drafted

What we are *not* aiming at in v1: undetectable Instagram/X bots, CAPTCHA farms, or a volunteer P2P network that runs a full LLM for you. Relays move bytes; they do not run your process. Inference is just an API key (OpenRouter and friends). The hard problems are **clock that does not live on your VPS** and **compute that is prepaid**.

## Shape (current thinking)

Three layers that should not trust each other:

1. **Clock** — check-in / inactivity detection on infrastructure that is not yours (Healthchecks.io looks like the default; deadcheck + PagerDuty as a harder backup).
2. **Vault** — encrypted playbook. The watcher should not be able to read it until fire time (`age` is enough to start).
3. **Executor** — a small webhook receiver that decrypts, then hands the playbook to an agent (Hermes is the likely runtime). Failures fall through to “email a human, ask them to do X.”

Check-in UX is a thin wrapper: Telegram `/alive`, a weekly nudge, a signed `/abort` for you or a trusted contact. The actual timer should not be a cron job on the same box as the agent.

## Status

Research notes and a sketch. See [PLAN.md](./PLAN.md).

Not a will. Not legal advice. False positives (hospital, travel, depression) are the scary failure mode — grace periods and a trusted-contact abort are part of the design on purpose.

## License

MIT (intended). Confirm when the first code lands.
