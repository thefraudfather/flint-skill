# FLINT: Verify Agent Integrity Before Payment

An Agent Skill that teaches Claude to verify an AI agent's integrity and authority
with [FLINT](https://flint.network) before it moves money, so the agent can be
trusted to transact and a signed evidence trail exists for every decision.

Identity answers *who an agent is*. FLINT also answers *is it authorized for this
action* and *is it still uncompromised* (not taken over, diverted, or
impersonated). This skill runs FLINT's verify-before-pay loop at the moment an
agent is about to spend: mint a Cross-Domain Agent Passport if needed, check the
transaction, gate on an allow / step-up / review / block verdict, and report the
outcome.

FLINT is rail-neutral: it verifies the agent, it does not move the money. The
payment rail still executes the transfer; FLINT decides whether it should proceed.

## What's in here

- `SKILL.md` — the skill (trigger description and workflow).
- `references/flint-tools.md` — argument detail and response shapes for the FLINT
  MCP tools.
- `evals/trigger-evals.json` — should-trigger / should-not-trigger queries for
  tuning the description.

## Requirements

The skill uses the **FLINT MCP connector** at `https://flint.network/mcp`
(Streamable HTTP). It is public: no account, no API key, no setup. Add it as a
custom remote MCP connector in your client, then the skill can run the loop.

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
