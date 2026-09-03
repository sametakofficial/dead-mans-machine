# Architecture (working)

How the pieces might fit. Not a freeze.

## What this repo is (and is not)

**Not a Hermes fork.** Hermes is a moving agent runtime (skills, plugins, cron, Telegram, browser). Forking it means we inherit 200k+ lines and lose the upgrade path.

**Not “Hermes, but with extra keep-alive hacks.”** A hidden patch that keeps someone else’s agent process alive is fragile and unreviewable.

**This repo is the trigger + vault + YAML contract.** Everything else is a neighbor we call.

- **NewAPI** is not in-tree. It is an external model depot — the same way OpenAI or Anthropic would be. We speak OpenAI-compat to it. Users who already run NewAPI (or OpenRouter directly) just paste a base URL.
- **Brains** (Hermes, WhyCodes, later others) are external runtimes. We start them at fire time, or ping a Hermes that already lives on Telegram. We do not fork them.
- If every brain is missing, prepaid credits are empty, or the agent hangs: core still warns, waits for abort, sends the emails, opens the repos, asks a human. YAML guarantees. Agents improvise.

```
┌─────────────────────────────────────────────────────────┐
│ dead-mans-machine          (this repo = trigger)        │
│  protocol  vault  yaml-runner  plugin slots             │
└───────────────┬───────────────────────────┬─────────────┘
                │ ping / FIRE               │ OpenAI-compat
                ▼                           ▼
         Healthchecks.io              NewAPI   (external depot)
         (or deadcheck)               = "our OpenAI" for this box
                                            ├ user’s keys / OpenRouter
                                            └ OmniRoute (optional cheap channel)
                │
                ▼
         brain slot (pick zero or one)
            WhyCodes   default-ish on a tiny VPS (one Rust binary)
            Hermes     if they already run it (Telegram gateway, skills)
            none       YAML only
```

Three processes, not one binary:

| Process | Stays up how | Needs GPU/LLM? |
| --- | --- | --- |
| **protocol** (thin watcher) | several copies ping the same Healthchecks URL | no |
| **depot** (NewAPI, *their* compose) | prepaid VPS + prepaid upstream credits | no (it proxies) |
| **executor** (our YAML runner, then maybe a brain) | same prepaid box, sleeps until webhook | only after FIRE |

## Core vs plugins

**Core** is the part that must work on a fresh clone + Healthchecks + a Telegram bot token. No NewAPI, no OmniRoute, no Hermes required.

Core jobs:

1. Check-in wrapper (`/alive`, `/status`, `/dead-now`, `/abort`)
2. Ping an external clock (Healthchecks ping URL)
3. Ladder: warn → trusted contact → abort window → FIRE
4. HMAC webhook in
5. Decrypt playbook (`age`)
6. Run **deterministic** steps from YAML (email, GitHub public, Telegram, `human_help`)
7. Dry-run flag

**Plugins** are anything we might swap or skip:

| Slot | Default | Other options |
| --- | --- | --- |
| Clock | `healthchecks` | `deadcheck`, later `nostr` |
| Nudge | `telegram` + `ntfy` | email only |
| Vault | `age` file | later Shamir / tlock / Arweave |
| Actions | `email`, `github`, `telegram`, `human_help` | later `browser`, `nostr_note` |
| Brain | none (YAML runner) | `whycodes` (light), `hermes` (if already installed) |
| Depot | none (brain’s own keys) | `newapi` as a neighbor, not a submodule |
| Cheap router | off | `omniroute` *behind* NewAPI |

A plugin here is a directory with a small manifest (`plugin.yaml`) and a runner that speaks a boring interface: `ping()`, `notify()`, `decrypt()`, `run_step()`. We can keep this in-process Python/Go at first. Hermes has its *own* plugin system — we use that only for the brain pack, not for the clock.

## AI depot: NewAPI in, OmniRoute as a plugin

Hermes already speaks many providers. That is not enough for “the owner is dead”:

- keys live in Hermes’ config (one box, one leak)
- no shared quota / prepaid pool
- no single place to say “this fire may spend at most $X”
- swapping cheap fallbacks is manual

**NewAPI** ([QuantumNous/new-api](https://github.com/QuantumNous/new-api)) is a self-hosted OpenAI-compatible gateway: channels, tokens, quotas, retries, format conversion. That is a **depot**, not a model. Compose it with Docker; do not vendor it (AGPL). Hermes points at `http://new-api:3000` like any OpenAI base URL.

**OmniRoute** ([diegosouzapw/OmniRoute](https://github.com/diegosouzapw/OmniRoute)) is a wide cheap/free router (many providers, auto-fallback). Useful, noisy, not something to make the default brain. Better as a **NewAPI channel** (or a Hermes `model-providers` plugin) the user can enable: when the prepaid OpenRouter bucket is empty, fall through to free/cheap models for “write this email,” not for “drive Instagram.”

Suggested wiring:

```
Hermes  -->  NewAPI
               token: dmm-executor   quota: $N for this fire
               channels:
                 1. OpenRouter (prepaid, primary)
                 2. user’s other keys
                 3. [plugin] OmniRoute   (optional, cheap)
```

Users who hate NewAPI skip it: Hermes talks to OpenRouter directly. The supervisor does not care.

## Brains: WhyCodes vs Hermes vs none

The product is not “an agent with a dead-man switch taped on.” It is a switch that *may* hire an agent.

**WhyCodes** ([whycorporation/whycodes](https://github.com/whycorporation/whycodes), [why.codes](https://why.codes)) — Rust, one native binary, ~10 MB idle, `whycodes generate "…" --format json` for headless fire, OpenAI-compat (so it can point at NewAPI), `AGENTS.md` / skills, GitHub tools, optional CDP browser. Fits a cheap always-on box. Good **default brain plugin** when we want improvisation without dragging a Python gateway.

**Hermes** — heavier, already has Telegram/Discord gateway, cron, a rich plugin/skill system. Good when the owner *already* lives in Hermes. Then DMM is just: Healthchecks + webhook + “message the Hermes bot / run this skill.” We still do not become a Hermes keep-alive daemon.

**none** — YAML runner only. This is the core demo. Ship this first.

A “plugin network” is right for **brains and actions**. It is wrong as the *identity* of the repo. If DMM is only a marketplace of other people’s agents, there is no clock, no vault, no abort, no `human_help` contract. Keep a small core; let WhyCodes/Hermes/email/github register in slots.

Do not make WhyCodes the heart either. It is a coding agent. Fire-day jobs are mostly mail + git + “ask a person.” A 10 MB binary is a nice *optional* worker, not the supervisor.

## Fire path (v1)

```
owner taps /alive
    protocol × N  ──GET hc-ping.com/<uuid>──►  Healthchecks
                                                 │
                                            down + nags
                                                 │
                                      trusted contact /abort?
                                                 │
                                         POST /trigger  (HMAC)
                                                 │
                                         executor
                                           ├ decrypt playbook
                                           ├ run YAML steps   ← always
                                           └ if brain=whycodes|hermes
                                                one-shot generate / skill
                                                NewAPI if configured
```

YAML runner is the contract. Hermes is allowed to *add* attempts, not to skip the contract. If the agent loops or the depot 402s, `human_help` still goes out.

## Repo layout (when code exists)

```
protocol/          thin watcher + telegram/ntfy + HC ping
vault/             age encrypt/decrypt helpers
executor/          webhook + yaml runner
plugins/
  clock/healthchecks/
  clock/deadcheck/         (later)
  brain/whycodes/          default-ish light brain (generate --format json)
  brain/hermes/            pack: skill + plugin.yaml (if they already run Hermes)
  depot/newapi/            compose *example* only — image stays upstream
  depot/omniroute/         optional NewAPI channel
  action/email/
  action/github/
  action/human_help/
playbook.example.yaml
compose.yml                protocol + (optional) new-api + (optional) hermes
```

v1 compose can be protocol-only. NewAPI and Hermes are profiles.

Stub manifests live under `plugins/` already so the slots are visible before any code. `playbook.example.yaml` is the fire contract.

## What “working core” means

A demo we can run without an LLM:

1. `docker compose up` protocol
2. `/alive` keeps a Healthchecks check green
3. skip pings → warn on Telegram/ntfy
4. `/dead-now` or real down → abort window
5. FIRE → dry-run prints “would email X, would public Y”
6. live mode with SMTP + a dummy GitHub token does those two things

Brain/depot come after that loop is boring.

## Why not one mega-plugin inside Hermes

Hermes plugins run **in-process** with the agent. A clock that lives only there dies with the agent. A vault that lives there is readable by every other plugin. Keep clock + vault + webhook **out of** Hermes; keep “how to improvise on this playbook” **in** a Hermes skill.

## License note

NewAPI is AGPL. We should **call** it over HTTP, not copy it into this tree. OmniRoute is MIT. Hermes is MIT. This repo can stay MIT if we only compose.
