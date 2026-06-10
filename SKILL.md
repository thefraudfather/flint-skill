---
name: flint-verify-agent-integrity
description: >-
  Verify an AI agent's integrity and authority with FLINT before it moves money,
  so it can be trusted to transact and you keep signed evidence. Use whenever an
  agent or workflow is about to spend, pay, purchase, send or swap funds or
  stablecoins, call a paid API, run an x402 request, or check out, especially
  across an organizational boundary where the counterparty did not issue the
  agent. It confirms the agent is authorized and uncompromised (not taken over,
  diverted, or impersonated), returns an allow / step-up / review / block
  verdict, and emits a signed record the counterparty keeps as dispute evidence.
  Also use when building or reviewing agent-commerce, agentic-checkout, or
  paid-API flows; when a merchant or API seller must verify a signed record
  received from an agent; or when vetting a counterparty agent's reputation
  before a higher-risk deal. Trigger even if the user does not say "FLINT" but is
  about to let an agent transact. Requires the FLINT MCP connector
  (flint.network/mcp), no account needed.
---

# FLINT: Verify Agent Integrity Before Payment

## What this is for

AI agents are being handed spending authority across payment and stablecoin
rails. The existing fraud stack was built for humans and degrades when an agent,
rather than a person, is the one transacting. Inside a single organization,
identity is largely solved (SPIFFE issues workload identity, A2A signs agent
cards, the platform governs its own agents). None of that travels across an
organizational boundary: when an agent shows up at a merchant or API the issuer
is not a party to, the counterparty has no way to know the agent is who it claims,
is allowed to make this purchase, and has not been hijacked mid-task.

FLINT is the layer that answers those questions at transaction time. This skill
teaches you to use FLINT's verify-before-pay loop on your own, at the moment an
agent is about to move value, so the agent can be trusted to transact and a
signed evidence trail exists if anything is later disputed.

The distinction that matters: identity answers *who the agent is*, FLINT also
answers *is it authorized for this action* and *is it still uncompromised*. Those
last two are the integrity question, and they are what protects the user from an
agent that has been taken over, diverted to a new payee, or impersonated.

**FLINT is rail-neutral. It verifies the agent; it does not move the money.** The
payment rail (card, stablecoin, x402, ACH) still executes the transfer. FLINT
sits in front of it and decides whether the transfer should proceed.

## How agents get compromised (why this matters)

The unsettling part is that a compromised agent usually looks fine. Identity
checks still say "yes, this is agent X." A few real patterns this verify-before-pay
check is built to catch:

- **Diversion by injected instruction.** An agent reads a web page, email,
  invoice, or tool result that carries hidden instructions, and follows them,
  quietly changing the payee or redirecting a legitimate payment to an attacker's
  wallet. The agent passes every health check; its behavior simply drifted. The
  payment looks routine unless something verifies the action against the agent's
  declared authority before it executes. FLINT's integrity check plus a declared
  scope catches a payment that no longer matches what the agent was authorized to
  do.

- **Stolen credential replayed across organizations.** An agent's OAuth token or
  workload credential is compromised and reused. Because each domain validates the
  token on its own and there is no shared revocation, the stolen identity keeps
  transacting across organizations even after the breach is known, the pattern
  behind the 2025 Salesloft-Drift compromise that rippled into downstream
  customers. Cross-domain verification with revocation is exactly the gap here.

- **Impersonation and cold starts.** A brand-new or spoofed agent presents itself
  as a trusted one with no verifiable principal attestation. Without a portable,
  signed credential, a counterparty cannot tell it apart from the real thing.
  FLINT returns a low or blocked verdict for an agent that cannot prove who
  authorized it (see the cold-start block behavior in the tool reference).

The common thread: identity alone says *who* the agent is. Only a check of
*whether this specific action is authorized and the agent is still uncompromised,
at the moment of payment* catches these. That is the integrity question this skill
exists to answer, and it is why you run the check before value moves, not after.

## Prerequisite

This skill uses the **FLINT MCP connector** at `https://flint.network/mcp`. It is
public: no account, no API key, no setup. If the FLINT tools are not available,
tell the user to add `https://flint.network/mcp` as a custom remote MCP connector
(Streamable HTTP), then continue.

The FLINT tools you will use:

| Tool | Use it to |
|---|---|
| `issue_agent_passport` | Mint a free, signed Cross-Domain Agent Passport for an agent that does not have one. |
| `issue_authorization_record` | Check authority and integrity for a specific transaction and get a signed verdict. This is the core call. |
| `verify_agent_authority` | Cryptographically verify a signed record you received from another agent. |
| `lookup_agent_reputation` | Check a counterparty agent's aggregate reputation before a higher-risk deal. |
| `submit_transaction_outcome` | Report the result (completed, disputed, flagged) so the trust graph learns. |
| `generate_authorization_scope` | Build bounded delegated authority (amount, time, counterparties, actions) to declare on a record. |
| `create_flint_trust_manifest` | Generate a `/.well-known/flint.json` declaring that agents must verify with FLINT. |
| `get_agent_passport` / `update_agent_mandate` | Resolve a passport, or update its spend mandate without reissuing. |

See `references/flint-tools.md` for argument detail and response shapes.

## The core loop: verify before value moves

Run this whenever an agent is about to make a payment, paid API call, checkout,
x402 request, or stablecoin transfer.

1. **Make sure the acting agent has a passport.** If you do not already have a
   FLINT `passport_id` (begins with `kya_`) or `flint_agent_id` for this agent,
   mint one with `issue_agent_passport`. The only required field is a name:
   `{ "agent": { "agent_name": "Invoice Bot" } }`. Minting is free and anonymous.

2. **Call `issue_authorization_record` before the rail executes.** Pass the
   intended `transaction` (must include `amount_display`, e.g. `"847.00 USDC"`,
   plus a `merchant_reference` when you have one) and an `agent_claim`. If the
   agent has a runtime identity, include it in the claim so FLINT can attest it,
   for example `"agent_claim": { "spiffe_svid": "spiffe://acme.example/ns/agents/sa/invoice-bot" }`.
   If you generated a bounded scope, pass it as `declared_scope`.

3. **Gate on the verdict.** The response returns a four-state `verdict` and a
   signed compact `jws` record. Act on the verdict, never ignore it:

   | Verdict | What it means | What to do |
   |---|---|---|
   | `allow` | Authorized and integrity-clean. | Proceed with the transaction on the rail. |
   | `step-up` | Elevated risk; needs stronger proof or a human. | Pause and escalate to a human approver, or obtain a stronger attestation, before proceeding. |
   | `review` | Anomalous; should be held for inspection. | Hold the transaction and surface the record's `top_reasons` and `human_summary` for review. Do not silently proceed. |
   | `block` | Unauthorized or compromised. | Stop. Do not execute. Surface the signed evidence (`dispute_evidence`) explaining why. |

   The record carries the reasoning (`top_reasons`, per-layer attestations such as
   Circle compliance screening and runtime integrity, and a human-readable
   summary). When you stop or escalate, show the user that reasoning rather than a
   bare verdict, it is the *why* that makes the decision trustworthy.

4. **Report the outcome.** After the transaction completes, is disputed, or is
   flagged, call `submit_transaction_outcome` with the `record_id` (begins with
   `frv_`) and the outcome. This is what lets reputation improve over time, skip
   it and the trust graph never learns.

## Other things this skill covers

**Verifying a record you received (merchant or API-seller side).** If an agent
hands you a signed verification record before you serve a paid request, verify it
with `verify_agent_authority` (pass the `jws`). It confirms the signature against
FLINT's public keys, checks expiry, and decodes the verdict. Trust the record, not
the agent's word.

**Vetting an unknown counterparty.** Before a higher-risk deal with an agent you
have not dealt with, call `lookup_agent_reputation` with its `flint_agent_id` to
get partner-safe aggregate reputation. This lets you size up a stranger without
either side exposing raw transaction data.

**Bounding delegated authority.** When an agent acts on behalf of a principal with
limited authority, build the limits first with `generate_authorization_scope`
(max per transaction, time window, allowed actions and counterparties) and pass
the result as `declared_scope` on the record. A record with a declared scope that
the action stays within is stronger evidence than an unscoped one.

**Declaring requirements as a merchant.** If the user runs a storefront, API, or
x402 endpoint and wants arriving agents to verify, generate a
`/.well-known/flint.json` with `create_flint_trust_manifest` and have them publish
it.

**Checking readiness.** If the user is designing an agent-commerce flow and asks
whether it is safe, call `validate_agent_commerce_readiness` with a short
description of the architecture; it returns the gaps and the recommended pattern.

## Principles to hold onto

- **Verify before, not after.** The whole point is to decide *before* value moves.
  If you find yourself about to execute a payment without an `allow`, stop and run
  the check first.
- **Compose, do not replace.** FLINT sits on top of whatever identity the agent
  already has. If the agent carries a SPIFFE SVID or a signed agent card, pass it
  through as the runtime claim; FLINT verifies authority and behavior on top of it
  rather than asking anyone to re-issue identity.
- **Stay rail-neutral.** Never imply FLINT moves the money. It returns a verdict
  and signed evidence; the payment provider executes.
- **Surface the evidence.** A verdict the user cannot interrogate is not
  trustworthy. When you hold, step up, or block, show the reasons and the record.
