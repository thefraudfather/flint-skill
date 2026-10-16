# FLINT: Verify Agent Integrity Before Payment

An Agent Skill that connects an AI agent to [FLINT Network](https://flint.network)
and checks what it may do before it moves money, so a signed evidence trail
exists for every decision.

Identity answers *who an agent is*. FLINT also checks *is it authorized for this
action*. The skill connects the agent to FLINT, gets it an Agent Passport after
asking the human for the accountable controller and the mandate, checks each
transaction before money moves, and acts on the allow / step-up / review / block
verdict.

FLINT is rail-neutral: it checks the agent, it does not move the money. The
payment rail still executes the transfer; the system that integrates FLINT
decides whether it goes ahead.

## One paste

Any agent that can read a URL and use remote MCP can start from one line:

```
Read https://flint.network/skill.md and follow the steps.
```

https://flint.network/skill.md is the canonical copy. `SKILL.md` in this repo is
the same text, kept in sync with it.

## What's in here

- `SKILL.md`: the skill (trigger description and steps).
- `evals/trigger-evals.json`: should-trigger and should-not-trigger queries for
  tuning the description.

## Requirements

The skill uses the **FLINT MCP connector** at `https://flint.network/mcp`
(Streamable HTTP). No API key is needed. Add it as a custom remote MCP connector
in your client, then the skill can run. An email sign-in code (OTP) is used to
mint a Passport the human owns from the start.

Documentation: https://flint.network/connect

## Privacy

The FLINT connector collects only what an agent submits to mint or verify (agent
name, optional controller, optional wallet, transaction context). Passports are
signed and publicly resolvable by design. FLINT uses submitted verification data
to train and improve its own fraud-detection models, does not sell data, does not
share raw identifiers, and does not use it to train third-party or general-purpose
models. Full policy: https://flint.network/privacy

## Operated by

KillChain, Inc. (d/b/a FLINT Network) · contact@flint.network
