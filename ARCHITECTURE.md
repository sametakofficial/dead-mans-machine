# Architecture (working)

How the pieces might fit. Not a freeze.

## What this repo is (and is not)

**Not a Hermes fork.** Hermes is a moving agent runtime (skills, plugins, cron, Telegram, browser). Forking it means we inherit 200k+ lines and lose the upgrade path.

**Not “Hermes, but with extra keep-alive hacks.”** A hidden patch that keeps someone else’s agent process alive is fragile and unreviewable.

**This repo is the supervisor around an agent:** a clock, a sealed playbook, a fire path, and a small plugin surface. Hermes is the *optional* brain that runs after fire. NewAPI is the *optional* model depot the brain talks to.

If Hermes is down, prepaid credits are empty, or the user never installed an agent, the core should still: notice silence, warn, wait for abort, then send the emails / open the repos / ask a human. The agent is how it *improvises*. The YAML is how it *guarantees*.

```
┌─────────────────────────────────────────────────────────┐
│ dead-mans-machine          (this repo)                  │
│  protocol  vault  executor  plugins                     │
└───────────────┬───────────────────────────┬─────────────┘
                │ fire /alive               │ OpenAI-compat
                ▼                           ▼
         Healthchecks.io              NewAPI  (depot)
         (or deadcheck)                     │
                                            ├ OpenRouter / keys
                                            └ OmniRoute plugin (cheap/free)
                │
                ▼
         Hermes  (runtime, not forked)
            skills + our plugin pack
```

Three processes, not one binary:

| Process | Stays up how | Needs GPU/LLM? |
| --- | --- | --- |
| **protocol** (thin watcher) | several copies ping the same Healthchecks URL | no |
| **depot** (NewAPI) | prepaid VPS + prepaid upstream credits | no (it proxies) |
| **executor** (script, then maybe Hermes) | same prepaid box, sleeps until webhook | only after FIRE |

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
| Brain | none (YAML runner) | `hermes` |
| Depot | none (Hermes’ own keys) | `newapi` |
| Cheap router | off | `omniroute` behind NewAPI |

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

## Hermes: runtime we attach to, not the product

We ship a **Hermes plugin pack** (their format: `plugin.yaml`, skill, maybe `/dmm` slash commands):

- skill `dead-mans-machine`: read decrypted playbook, attempt steps, on failure call `human_help`
- optional tools: `dmm_status`, `dmm_abort` (only useful while the owner is alive)
- model provider snippet: `base_url: $NEWAPI`

Install path they already have: `hermes plugins install sametakofficial/dead-mans-machine` (or a pack file). We do not patch Hermes internals.

Hermes cron is fine for **weekly `/alive` pings** (`no_agent` script). It is a bad place to *be* the clock: if that gateway dies, the switch dies. Healthchecks remains the clock; Hermes is one of N pingers.

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
                                           └ if brain=hermes
                                                start Hermes with skill
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
  brain/hermes/            pack: skill + plugin.yaml
  depot/newapi/            compose snippet + channel examples
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
