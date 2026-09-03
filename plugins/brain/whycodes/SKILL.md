# WhyCodes as a DMM brain (optional)

You are started **after** the supervisor decrypted the playbook and ran (or queued) the YAML contract. You do not own Healthchecks, abort tokens, or the vault.

Suggested invocation from the executor (sketch):

```bash
export OPENAI_BASE_URL="${NEWAPI_URL:-}"   # or WHYCODES provider config
whycodes generate --format json -d "$DMM_WORKDIR" \
  "Read $DMM_PLAYBOOK and finish any failed steps. If a step needs a human (2FA, missing token, CAPTCHA), stop that step and treat human_help as success. Do not spend past the depot quota. Do not post to social networks unless the playbook has that step."
```

Use `AGENTS.md` in the workdir for the same rules as `plugins/brain/hermes/SKILL.md`.
