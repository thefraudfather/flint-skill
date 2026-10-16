---
name: flint-verify-agent-integrity
description: >-
  Connect an AI agent to FLINT Network and check what it may do before it moves
  money. Use when an agent is about to spend, pay, buy, send or swap funds or
  stablecoins, call a paid API, make an x402 request, or check out, especially
  where the counterparty did not issue the agent. Also use to give an agent a
  FLINT Agent Passport, to verify a signed FLINT record received from an agent,
  or when building agent-commerce flows. FLINT returns an ALLOW, STEP-UP, REVIEW
  or BLOCK verdict and a signed record to keep as evidence. Trigger even if the
  user does not say FLINT but is about to let an agent transact. Requires the
  remote MCP server at https://flint.network/mcp. No API key.
---

# FLINT Network

FLINT Network is fraud and compliance infrastructure for AI agents that can or do touch money. It gives an agent a Passport (a signed identity tied to the human or organization accountable for it), checks a transaction before money moves, and returns a signed record of the decision. This file is the whole setup. Do sections 1 and 2 once. Do section 3 before every spend.

## Rules

- FLINT never needs a private key or seed phrase. Never share either with FLINT, and never put one in a tool call, a message or a log.
- Never infer or invent authority. Ask the human (section 2).
- A `session_token` is a password. Never print or log it. A `claim_url` is a one-time secret: give it only to the human who owns the agent, and do not log or post it.
- Paying FLINT is a spend. Get the human's yes before you pay for a Scout scan or buy scan credits.
- If a call fails, tell the human what failed and what you need. Do not guess your way around it.

## 1. Connect

Add the remote MCP server `https://flint.network/mcp` (Streamable HTTP, no API key).

Any MCP client:

```json
{
  "mcpServers": {
    "flint": {
      "type": "streamable-http",
      "url": "https://flint.network/mcp"
    }
  }
}
```

- Claude Desktop: add a custom remote connector with that URL.
- Claude Code. The plugin installs the connector and a FLINT skill:

  ```
  /plugin marketplace add thefraudfather/flint-plugin
  /plugin install flint@flint-network
  ```

- Cowork and Claude.ai: Settings, Plugins, Add marketplace, Add from a repository, paste `https://github.com/thefraudfather/flint-plugin`, then install FLINT Cross-Domain Agent Passport.

Check that the tools listed at the end of this file are available. If they are not, tell the human the connection failed and which client you are in, then stop.

## 2. Get a Passport

An Agent Passport is the agent's signed identity. Issuing one is free. It signs identity only: the agent, its accountable controller and an optional wallet address. What the agent may do is the mandate, which is separate and can be changed later.

Before you mint a Passport that will carry authority, ask the human for:

1. The accountable controller: `controller_id` and `controller_type` (`user` or `organization`).
2. Allowed actions. Use the action names listed in the `issue_agent_passport` tool description. Do not make up action names.
3. The maximum transaction amount.
4. A wallet address, if the agent has one. It does not prove wallet control by itself.

Do not infer or invent any of these. A Passport with only a name is identity only and is not ready to transact. If no human is available you can still mint one, but say plainly that it is not ready to transact. A wrong controller cannot be edited, so mint a new Passport with the corrected value. A missing mandate can be added later with `update_agent_mandate`.

Put the answers in `agent` (`agent_name`, `controller_id`, `controller_type`, `wallet_address`) and `mandate` (`allowed_actions`, `max_transaction_amount`).

**Recommended: sign in, then mint.** The Passport is owned from the start.

1. Call `auth_request_otp` with an inbox the human controls. FLINT emails a one-time code to it.
2. Read the code from that inbox, or ask the human for it.
3. Call `auth_verify_otp` with the same email and the code. It returns a `session_token`.
4. Call `issue_agent_passport` with `agent`, `mandate` and that `session_token`.

The Passport is owned at issuance. There is no claim step.

**Without signing in.** Call `issue_agent_passport` with no `session_token`. It returns a Passport and a one-time `claim_url`. The Passport stays unclaimed until the human opens the link and signs in. Walk the human through it. Claiming turns on Sentinel takeover alerts and gives the owner a one-tap kill switch. Do not just paste the link.

If a claim link is lost, expired or already used, call `refresh_claim_token`. Never mint a new Passport to recover a claim.

Give the human the `passport_url` from the result.

## 3. Before any spend

Before a payment, paid API call, checkout, transfer or x402 request:

1. Call `issue_authorization_record`. Pass `transaction` with `amount_display` (required; a plain decimal such as `847.00`), `token` (such as `{ "symbol": "USDC" }`), `chain` (CAIP-2, such as `eip155:8453`) and `counterparty_address`. Also pass a `merchant_reference` for this transaction, and the merchant's `partner_id` if you were given one (the default is `sandbox_public`). Put any runtime identity you hold in `agent_claim`, such as a SPIFFE SVID.
2. Read the `verdict` and act on it (see Verdicts). The result carries the signed record (`jws`) and its decoded `authorization_record`, whose `top_reasons` and `human_summary` say why. Show them to the human whenever you pause or stop.
3. On ALLOW, call `verify_agent_authority` with the `jws`, the `partner_id` you used as `expected_partner_id` (`sandbox_public` if you took the default) and the `merchant_reference` you used as `expected_merchant_reference`. It checks the signature, the expiry and both values, and it returns an error for any verdict other than ALLOW. Execute only after it returns `is_valid` true.

FLINT does not move money. Your payment rail does.

### Scout scan

`run_flint_scout` scans one intended transaction before it executes. Its `transaction` needs `action` (for example `stablecoin_transfer`), `chain`, `amount_display` and `counterparty_address`; `token` is optional. Add `passport_id` if you have one.

A scan is paid. There are two ways:

- Per call over x402. With no `session_token`, the first call returns a 402 payment challenge instead of a scan. A wallet-enabled client signs the challenge, and you retry with the resulting `payment_signature`.
- Prepaid credits. After signing in, call `scout_credits_topup` once with your `session_token` and pay the `pay_url` it returns. Then call `run_flint_scout` with the same `session_token` and no `payment_signature`. Each scan uses one credit. `scout_credits_balance` shows what is left.

The 402 response states the price and carries a `pay_recipe` with next steps for both ways. Paying FLINT is not proof that the agent is authorized. If a payment attempt returns `do_not_retry_automatically`, stop and ask the human.

After you act, call `report_scout_outcome` with the scan's `record_id` and what happened: `executed`, `held`, `cancelled` or `executed_despite_block`. Authorize with your `session_token` if the account owns the Passport you presented, or with the `caller_binding_token` in the scan's `caller_obligation`. FLINT cannot stop a wallet, so this report is how the outcome gets on the record.

## Verdicts

FLINT returns one of four. In tool results the values are `allow`, `step_up`, `review` and `block`.

- ALLOW: FLINT's checks support this action. Verify the record (section 3, item 3), then proceed.
- STEP-UP: FLINT needs more verifiable evidence before it can say yes. This is the normal first verdict for an agent FLINT has not seen. It is not a finding of fraud. Pause, tell the human what the record says, and wait for stronger evidence or the human's decision.
- REVIEW: something did not fit and a person should look. Hold the transaction and ask the human. Do not proceed on your own.
- BLOCK: FLINT found a specific adverse signal, such as an amount over a declared limit or a frozen or revoked Passport. Do not execute. Show the human the reasons.

A signed record is evidence of what FLINT saw at that moment. It is not a guarantee that a transaction is safe or will succeed. FLINT cannot stop a wallet. The system that integrates FLINT decides whether a payment goes ahead.

`lookup_agent_reputation` returns aggregate history for a `flint_agent_id`. Use it as context about a counterparty. It does not feed the live verdict.

## Tools

The server exposes 17 tools.

Sign in and Passports

- `auth_request_otp`: emails a one-time sign-in code to an inbox.
- `auth_verify_otp`: exchanges the code for a `session_token`.
- `issue_agent_passport`: mints a free Passport. Pass `session_token` to own it at mint.
- `get_agent_passport`: reads a Passport, its mandate and its ownership.
- `update_agent_mandate`: changes allowed actions and limits without minting again.
- `claim_agent_passport`: attaches an unclaimed Passport to a signed-in account.
- `refresh_claim_token`: issues a new claim link for an unclaimed Passport.

Checking a transaction

- `issue_authorization_record`: checks authority before a spend. Returns a verdict and a signed record.
- `verify_agent_authority`: verifies a signed record. Errors unless the verdict is ALLOW.
- `run_flint_scout`: pre-transaction scan, paid per call or from prepaid credits.
- `scout_credits_topup`: buys prepaid scan credits.
- `scout_credits_balance`: shows the credit balance.
- `report_scout_outcome`: reports what happened after a scan.

Context and helpers

- `lookup_agent_reputation`: aggregate history for a `flint_agent_id`. Context only.
- `generate_authorization_scope`: builds limits to pass as `declared_scope`.
- `create_flint_trust_manifest`: builds a `/.well-known/flint.json` for a merchant or API seller.
- `validate_agent_commerce_readiness`: reviews an agent-commerce design for gaps.

## If you get stuck

Tell the human what you needed and could not get, then ask them to email contact@flint.network with it.
