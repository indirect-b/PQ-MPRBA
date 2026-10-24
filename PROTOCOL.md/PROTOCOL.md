# PQ-MPRBA Protocol Specification

**PQ-MPRBA** *(Post-Quantum Multi-Party Ratchet with Blinded Authentication)* — designed by Mohammad Taha Azizi Ahad.

As implemented in chat-app v6.0.0. Experimental — not independently audited.

## Design goals

- End-to-end encryption that runs entirely in the browser.
- Confidentiality against both classical and future quantum attackers (hybrid design).
- Authentication that is itself post-quantum, not just the key exchange.
- Group chats where membership changes are safe and do not break messages.
- A server that learns as little as possible — ideally nothing but ciphertext.

## Primitives

| Purpose | Primitive |
| :--- | :--- |
| Post-quantum key encapsulation | ML-KEM-768 (NIST FIPS 203) |
| Classical key agreement | ECDH P-256 |
| Post-quantum identity signatures | ML-DSA-65 (NIST FIPS 204) |
| Key derivation | HKDF-SHA256 |
| Message encryption | AES-256-GCM (fresh key per message) |
| Blinded authentication | Ristretto255-VOPRF (RFC 9497) with DLEQ proofs |
| Ratchet authentication | HMAC chain |

## Layer 1 — Hybrid ratchet

Each session combines ML-KEM-768 with ECDH P-256; both shared secrets feed HKDF-SHA256 into a symmetric ratchet chain. Every message gets a fresh one-time AES-256-GCM key.

- **Forward secrecy:** old keys are never stored. A later compromise cannot rewind past messages.
- **Post-compromise recovery:** as new messages flow, the chain advances past a compromised state and locks the attacker out again.
- **Out-of-order messages:** a bounded skipped-key cache (at most 100 keys, 10-minute lifetime) lets early messages decrypt without keeping old keys alive. Anything older is gone.
- **Zeroization:** used keys are explicitly wiped from RAM, not merely freed.

## Layer 2 — Identity signatures

Each identity bundle publishes an ML-KEM-768 public key together with an ML-DSA-65 signature over it. The other side verifies the signature before trusting the key.

- A contact earns the PQ badge only with a valid key **and** a valid signature.
- A failed signature is shown as a clear warning, never silently accepted.
- Contacts without PQ material fall back to the classical state, labeled as such — never disguised as post-quantum.

## Layer 3 — Group epochs

Group keys move through epochs under a small state machine: STABLE → ROTATING → STABLE.

- Rotation is triggered by membership changes (join/leave).
- A 3-epoch grace buffer keeps in-flight messages from the previous epochs decryptable during the transition.
- Once the transition completes, old epoch keys are wiped.
- Result: a departed member cannot read future messages; a new member cannot read past ones.

## Layer 4 — Blinded authentication

Login uses Ristretto255-VOPRF: the client blinds its credential, the server operates on the blinded value, and the client unblinds the result. The server proves correct operation with DLEQ proofs without revealing its key.

The server is a zero-knowledge forwarder: it relays ciphertext and blinded tokens only. It never sees plaintext and does not learn identities from authentication.

## Honest limits

- Experimental, single-author work. No independent audit has been performed.
- Hybrid by design: a classical ECDH half is still present and labeled as such. "Fully post-quantum" describes the direction, not the current state.
- The 3-epoch grace window and the 100-key / 10-minute skipped-key bounds are judgment calls, chosen to balance reliability against key lifetime — open to criticism.

## Implementation

The reference engine is [`crypto.ts`](crypto.ts) (Web Crypto API + `@noble/post-quantum` / `@noble/curves`). The messenger using it: https://github.com/indirect-b/chat-app
