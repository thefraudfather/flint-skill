# FLINT MCP tool reference

Argument detail and response shapes for the FLINT tools used by this skill. The
server is `https://flint.network/mcp` (Streamable HTTP, no auth). All IDs are
stable prefixes: passports `kya_`, agent ids `faid_`, records `frv_`, outcomes
`out_`.

## issue_agent_passport (write)

Mint a free, hybrid-signed Cross-Domain Agent Passport. Only a name is required.

Input (minimum):
```json
{ "agent": { "agent_name": "Invoice Bot" } }
```
Optional: `agent.controller_id`, `agent.controller_type`, `agent.wallet_address`,
`agent.attestations`, and a `mandate` object (`allowed_actions`,
`max_transaction_amount`, `notes`). The mandate is mutable config and is NOT part
of the signed identity.

Key response fields: `passport_id` (`kya_...`), `flint_agent_id` (`faid_...`),
`passport_url` (public, resolvable), `claim_url` (one-time, to attach to an
account), `signature_valid`, `owned`. The signature is hybrid: classical ES256
plus post-quantum ML-DSA-65.

## issue_authorization_record (write) — the core call

Check authority and integrity for a specific intended transaction and get a signed
verdict. Call this BEFORE the rail executes.

Input:
```json
{
  "transaction": { "amount_display": "847.00 USDC", "merchant_reference": "order_847" },
  "agent_claim": { "spiffe_svid": "spiffe://acme.example/ns/agents/sa/invoice-bot" },
  "declared_scope": { }
}
```
`transaction.amount_display` is required. `agent_claim` carries identity and any
runtime hint (SPIFFE SVID, signed agent card, principal). `merchant_reference`,
`partner_id`, `nonce`, and `timestamp` are optional (nonce/timestamp auto-fill).

Key response fields: `record_id` (`frv_...`), `verdict` (`allow` | `step-up` |
`review` | `block`), `jws` (compact signed record), and an `authorization_record`
object containing `verdict.top_reasons`, per-source `attestations` (e.g.
`circle_compliance_engine`, `etherscan`, `flint_runtime_intelligence`, `spiffe`),
`dispute_evidence.human_summary`, and `dispute_evidence.evidence_replay_url`.

Note: a brand-new agent with no principal attestation and no declared scope will
often return `block` with reasons like `agent_cold_start` and
`no_principal_attestation`. That is correct behavior, declare a scope and pass a
runtime claim to raise the verdict.

## verify_agent_authority (read)

Verify a signed record received from an agent.

Input: `{ "jws": "<compact JWS>" }` (optional `jwks_url`, defaults to FLINT prod).
Response: `is_valid`, `is_expired`, `protected_header` (alg ES256, kid), `verdict`,
and the decoded `authorization_record`. Verifies against
`https://flint.network/.well-known/jwks.json`.

## lookup_agent_reputation (read)

Input: `{ "flint_agent_id": "faid_..." }` (pattern `^faid_[a-f0-9]{24}$`).
Response: `agent_reputation` with `global_stats` (first_seen, total_transactions,
dispute_rate, f3_labels), `runtime_diversity`, `principal_consistency`,
`recent_outcomes`. Aggregate only, no raw identifiers.

## submit_transaction_outcome (write)

Input: `{ "record_id": "frv_...", "outcome": "completed" | "disputed" | "flagged", "merchant_note": "optional" }`.
Response: `outcome_id` (`out_...`) and confirmation the trust graph will update.

## generate_authorization_scope (read)

Input (required): `max_amount_per_tx` (number), `time_window_end` (ISO-8601 Z).
Optional: `allowed_actions`, `allowed_counterparties`, `asset` (default USD),
`principal_id`, `purpose`.
Response: `scope_object` and `declared_scope` (pass the latter to
`issue_authorization_record`).

## create_flint_trust_manifest (read)

Input (all optional, sensible defaults): `name`, `domain`, `contact_email`,
`description`, `supported_actions`, `max_transaction_amount`, `accepted_principals`.
Response: `manifest` and `manifest_json` to publish at `/.well-known/flint.json`.

## get_agent_passport (read) / update_agent_mandate (write)

`get_agent_passport`: `{ "passport_id": "kya_..." }` returns identity, mandate,
status, verification, public URL.
`update_agent_mandate`: `{ "passport_id": "kya_...", "mandate": { "allowed_actions": [], "max_transaction_amount": 1000, "notes": "..." } }`.
Updates mutable config only; `passport_signature_unchanged` confirms the identity
credential is untouched.

## validate_agent_commerce_readiness (read)

Input: `{ "architecture_description": "<at least 20 chars describing the flow>" }`.
Response: `signals`, `missing_components`, `recommendations`, `is_ready`,
`correct_pattern`. Use it to audit an agent-commerce design.
