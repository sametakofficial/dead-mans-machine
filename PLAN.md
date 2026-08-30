# PLAN

Working notes from the first research pass. Direction, not a spec. Things will move.

## Goal

If the owner stops checking in, **something still runs** that is not just “send this email.”

A sealed playbook gets opened. An agent tries the steps. When a step needs a human (GitHub 2FA, a site with no API, a confused relative), it **asks people**, instead of pretending a browser can impersonate the owner forever.

Success for a first cut: missed check-ins → warnings → abort window → mail + “please publish these repos” to named humans. Browser automation and social posting are later, optional, and easy to get banned for.

## What already exists (and what we should steal)

**Do not rebuild the clock.**

| Piece | Examples | Takeaway |
| --- | --- | --- |
| Email / file dead man's switch | [LastSignal](https://github.com/giovantenne/lastsignal), [Aeterna](https://github.com/alpyxn/aeterna), [dead-man-hand](https://github.com/bkupidura/dead-man-hand) | Grace, reminders, trusted contact, encrypted payload. Their *timer* dies if *your* VPS dies. |
| Clock independent of your box | [Healthchecks.io](https://healthchecks.io), [deadcheck](https://github.com/adamdecaf/deadcheck) + PagerDuty snooze | This is the interesting pattern. You only ping. They own the alarm. |
| Nudges | Telegram, [ntfy](https://ntfy.sh) | One-tap `/alive` beats “remember the magic link.” |
| Passive proof of life | [nostr-dead-man-switch](https://github.com/AusDavo/nostr-dead-man-switch) | Activity as check-in; “Uncle Jim” federation = friends run the watcher. Early, but the *shape* is right for a thin protocol. |
| Sealed payload | `age`, Shamir, drand [tlock](https://github.com/drand/tlock), Arweave (PingVaults) | v1: `age` + copies on two machines + USB. Permanent storage / social key-split later. |
| Agent runtime | [Hermes](https://github.com/NousResearch/hermes-agent) | Cron, Telegram, skills, optional cloud browser. Likely executor, not something to reimplement. |
| Browser agents | browser-use, Skyvern, Steel | Useful *after* fire, for sites with no API. Not the product. CAPTCHA solvers in an OSS core is a losing arms race. |

“Dead man switch + AI agent” on GitHub today is almost all **kill-switches for runaway agents**, not executors for dead owners. The niche looks empty.

## Two problems that actually matter

### 1. Liveness of the *process*

Public Syncthing / Iroh **relays are pipes**. They do not run your binary. If every machine that was running the daemon is off, the switch is off.

So:

- Put the **timer** on Healthchecks (or deadcheck/PagerDuty). Several local daemons can ping the *same* check URL; any one ping keeps it green.
- Run a **thin watcher** in more than one place (home VPS, a friend’s Pi, a second cheap box).
- Run the **fat executor** (Hermes + API keys) on prepaid compute, not on random volunteer relays.

Volunteer/P2P is plausible for the *thin* watcher (Uncle Jim style), not for “please run my 8GB agent.”

### 2. Liveness of *inference*

The model is an API. OpenRouter-style prepaid credits are the obvious fuel.

Caveats from current terms (they can change): unused credits may expire on the order of a year; credits are not transferable; auto-recharge wants a card, and a card is a bad posthumous dependency. Practical approach: prepaid burst bucket + a tiny keep-alive spend while alive so the balance does not go stale. Cheap open models as fallback for “send this email.” Do not design v1 around “the agent buys its own subscription with a card.”

Compute (the VM) is a separate bill. Two prepaid VPS in different places, or a prepaid Akash-style escrow, is more honest than “the mesh will host me.” A **finite window** (e.g. 12–24 months of prepaid run) is a feature, not a failure.

## Proposed split

```
protocol/    thin: /alive, warnings, abort, ping Healthchecks
vault/       encrypt/verify playbook (age). watcher cannot read it
executor/    HMAC webhook → decrypt → Hermes skill → steps + human_help
```

Fire path (sketch):

```
owner  --/alive-->  protocol × N  --ping-->  Healthchecks
                                              |
                                         down + nags
                                              |
                                    trusted contact abort?
                                              |
                                         webhook FIRE
                                              |
                                         executor + agent
                                              |
                              email / github / telegram / "please do X"
```

Healthchecks (or PagerDuty) is the clock. We own the ladder (warn → trusted contact → abort window → fire) and the playbook runner.

## Playbook (v1, no browser)

Something like: emails, `github: make_public`, a Telegram note, and `human_help` (“here are three repos and a letter; please publish”). Encrypted at rest. Dry-run mode that logs and sends nothing.

## Rough sequence

**First slice** — clock + UX, no secrets opened  
Healthchecks check; Telegram `/alive` `/status` `/dead-now`; ntfy reminder; a second pinger (cron curl) so the bot is not a single point of failure; fake a down and see who gets notified.

**Second slice** — ladder + abort  
Warn / trusted-contact mail / signed abort URL; webhook receiver that only verifies HMAC and logs.

**Third slice** — executor  
`age` playbook; mail + GitHub + Telegram + human_help; Hermes skill; dry-run; prepaid API credits on the executor host only.

**Later, maybe**  
deadcheck/PagerDuty as a second clock; Shamir shares; Arweave copy of ciphertext; a friend-run watcher; browser-use for one or two sites, with CAPTCHA escalating to a human, not to a solver API in-tree.

**Out of v1 on purpose**  
Vision-Chrome-as-the-product, Nopecha in core, volunteer relays as compute, auto-buying subscriptions, “never patched / never banned” social posting.

## Open questions

- Hosted Healthchecks vs self-host vs deadcheck-first — hosted is less work; company-risk is real but slow.
- How aggressive the ladder should be (weeks vs months). False fire is worse than a late fire.
- Whether Hermes is a hard dependency or the executor stays a boring script with an optional agent.
- Legal/ToS: auto-posting to Instagram/X/YouTube as the owner is likely against their rules even if the owner is dead. Prefer APIs you control and humans.

## Non-goals (for now)

A replacement for a will. A stealth scraping framework. A decentralized LLM. An always-on social media ghost.
