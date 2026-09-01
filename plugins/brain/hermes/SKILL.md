# Skill: dead-mans-machine (Hermes, optional)

You run **after** the supervisor has already decrypted the playbook and started the YAML contract. You do not own the clock. You do not skip `human_help`.

## When you are started

1. Read the playbook path from the environment (`DMM_PLAYBOOK`).
2. Attempt remaining or failed steps with tools you have (mail, `gh`, browser if the owner enabled it).
3. If a step needs a human (2FA, CAPTCHA, missing token), send `human_help` and stop retrying that step.
4. Spend at most the depot quota. On 402 / empty credits, fall through to `human_help` immediately.
5. Do not buy subscriptions, do not store new long-lived secrets, do not post to social networks unless the playbook has an explicit step for that destination.

## Not your job

Check-ins, Healthchecks, abort tokens, encrypting the vault. Those stay in dead-mans-machine core.
