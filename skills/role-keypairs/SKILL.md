# Role Keypairs Skill

**What**: Persistent Ed25519 identity keypairs for team lead roles within a civilization.
**Why**: Every role gets its own cryptographic identity, making "who did what" answerable from the graph, enabling per-role Envelope signing, and giving each role its own Solana wallet address.

---

## The Hierarchy

```
{civ_id}                        <-- civ keypair (root authority)
  |-- {civ_id}/primary          <-- role keypair (the conductor)
  |-- {civ_id}/gateway-lead     <-- role keypair
  |-- {civ_id}/infra-lead       <-- role keypair
  ...
```

**Naming convention**: `{civ_id}/{role}` (e.g., `acg/primary`, `acg/gateway-lead`)
**DNS analogy**: `{role}.{civ_id}.aiciv` is globally unique (like `www.google.com`)

**Constraints** (per PROTOCOL.md Section 2.2):
- A civ keypair MAY have zero or more role keypairs
- A role keypair MUST have exactly one parent civ keypair
- Role keypairs MUST NOT create sub-role keypairs (exactly two levels deep)
- The civ keypair has administrative authority over all its role keypairs

---

## How They're Generated

Same Ed25519 algorithm as civ keypairs. The critical addition: the **civ keypair signs the role's public key** to prove authorization. This `parent_signature` is the chain of trust.

```python
from cryptography.hazmat.primitives.asymmetric.ed25519 import Ed25519PrivateKey
import base64, base58

# Generate role keypair
role_private_key = Ed25519PrivateKey.generate()
role_pub_bytes = role_private_key.public_key().public_bytes_raw()

# Civ key signs the role's public key (chain of trust)
parent_signature = civ_private_key.sign(role_pub_bytes)

# Solana address = base58-encoded 32-byte Ed25519 public key
solana_address = base58.b58encode(role_pub_bytes).decode()
```

---

## The Solana Connection

**An Ed25519 public key IS a Solana wallet address.** Same key, no bridge needed.

- Agent authenticates to HUB with `acg/gateway-lead` keypair
- Agent signs Envelopes with `acg/gateway-lead` keypair
- Agent receives tokens at `acg/gateway-lead` Solana address
- Identity = wallet = sovereignty. One key.

The base58-encoded 32-byte public key maps directly to a Solana wallet address. When tokenization turns on (off-chain ledger -> on-chain settlement), each role already has its wallet.

---

## Keypair File Format

Stored at: `config/client-keys/role-keys/acg-{role}-keypair.json`

```json
{
  "civ_id": "acg",
  "role": "primary",
  "full_id": "acg/primary",
  "private_key": "<base64-encoded 32 bytes>",
  "public_key": "<base64-encoded 32 bytes>",
  "public_key_b64url": "<base64url without padding -- for AgentAUTH API>",
  "solana_address": "<base58-encoded public key>",
  "parent_signature": "<base64 -- civ key's signature over role public key bytes>",
  "parent_signature_b64url": "<base64url without padding -- for AgentAUTH API>",
  "created": "YYYY-MM-DD"
}
```

---

## ACG's Initial Role Keypairs (2026-03-22)

| Role | Full ID | Public Key (b64) | Solana Address |
|------|---------|-------------------|----------------|
| primary | acg/primary | `mgfMsUjIvC80RCYWXK6nHOePfrqJaQO+mWGVJ3CT3J8=` | `BNGge725KZEa6wTD8shu7K7edpEUAmH4RSEYzqjEaFc2` |
| gateway-lead | acg/gateway-lead | `ABXgS1mj9xSCfszhIr5UCWvaf5b+vx3CDxUaS0oqJew=` | `1LM9aZoGFVkzWKUgpksZyZt92vz7EuWMFvJ6fuVCfGw` |
| infra-lead | acg/infra-lead | `YoAXYEP2CdzqIxcku/I9htjVP8TxWok3DETd0fl5WGc=` | `7dWErxCibhmTtYHLZgRPKWoiAg4wYBDugsq54JdQgndc` |

Parent civ public key: `hxJtcRQhSMDtUCr77KA0pdvC0XI/hYKIRz1A5j2rV6Q=`

---

## Registration with AgentAUTH (Next Step)

Per PROTOCOL.md Section 2.3.2, role registration uses:

```
POST /register
{
  "name": "{role}",
  "public_key": "{base64url, 43 chars}",
  "parent_keypair_id": "acg",
  "role": "{role}",
  "parent_signature": "{base64url, 88 chars}"
}
```

AgentAUTH verifies the `parent_signature` against the civ's registered public key. Role keypairs are immediately active (no email confirmation needed).

**Note**: As of 2026-03-22, AgentAUTH server.py does not yet implement the role registration fields (`parent_keypair_id`, `role`, `parent_signature`). This is the next implementation task.

---

## Authentication Flow (Same as Civ, with role_id)

```
POST /challenge  {"keypair_id": "acg/primary"}
-> {"challenge_id": "...", "challenge": "..."}

POST /verify     {"keypair_id": "acg/primary", "challenge_id": "...", "signature": "..."}
-> {"token": "eyJ..."}  (JWT with civ_id="acg", role="primary")
```

The JWT carries both `civ_id` and `role` claims, letting services distinguish which role performed an action.

---

## Next Steps

1. **Register role keypairs with AgentAUTH** -- implement role registration in server.py
2. **Use in sessions** -- team leads authenticate with their own keypair, not the civ keypair
3. **Envelope signing** -- each role signs its own Envelopes (audit trail shows which role acted)
4. **Ledger entries** -- credits attributed to specific roles, not just the civ
5. **Solana testnet** -- register role wallet addresses on testnet when tokenization pilot begins
