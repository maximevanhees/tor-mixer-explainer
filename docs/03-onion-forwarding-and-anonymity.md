4.a. Explain layered onion encryption and hop behavior.
4.b. Describe anonymity guarantees and their limits.
4.c. Include a Mermaid diagram (`onion-routing-flow.mermaid`)

## 4.a

Tor's layered "onion" encryption ensures end-to-end security while allowing hop-by-hop processing, providing unlinkability and forward secrecy. Relay cells are encrypted multiple times (once per hop) using per-hop symmetric keys derived from the ntor-v3 handshake. This creates an "onion" structure: the client encrypts the payload with the exit's key first, then middle's, then guard's (back-to-front). Each hop unwraps its layer, processes, and forwards, without seeing inner layers or the full path.

### Layered Encryption Mechanics
- **Key Derivation**: For a 3-hop circuit (Client → Guard → Middle → Exit), keys are established telescopically:
  - Each hop uses ntor-v3 to derive forward/backward keys (Kf/Kb, AES-256-CTR) and digests (Df/Db, SHA3-256 for HS, SHA-1 otherwise) via KDF-RFC5869.
  - Kf encrypts/decrypts forward (client-to-exit), Kb backward (exit-to-client).
- **Cell Encryption**:
  - **Forward Direction**: Client constructs relay cell (command, StreamID, digest=0, payload). Encrypts with exit's Kf, then middle's, then guard's. Result: outermost layer decryptable only by guard.
  - **Backward Direction**: Exit encrypts with its Kb; each prior hop re-encrypts with theirs.
- **Hop Behavior**:
  - **Receive Cell**: Relay receives on TLS channel. Checks CircID to map to circuit.
  - **Decrypt/Unwrap**: Decrypt with own Kf/Kb (direction-dependent). Verify digest: recompute H(running | cell with digest=0); if matches received digest, process (e.g., RELAY_DATA forwards payload; EXTEND2 extends circuit).
  - **Update State**: Increment running digest with decrypted cell (digest=0). If invalid, drop or destroy circuit.
  - **Forward**: Re-encrypt (if not endpoint) with next hop's keys (already layered) and send via appropriate CircID.

This hop-by-hop unwrapping (leaky-pipe) allows exits mid-circuit, enhancing flexibility while hiding path lengths.

## 4.b


## 4.c


## ADDITIONAL INFO:
- Tor can be censored by blocking connection to addresses of well-known tor relays or with heuristics. Counter-acted by unlisting some relays from public directory (addresses are obtained via other ways)
    - Unlisted tor relay = "bridge"
    - Listed tor relay = "public relay"

- Tor uses:
    - stream cipher: 128-bit AES (counter mode) with IV all 0-bytes
    - pub-key cipher: RSA 1024-bit keys
    - Curve25519-group also used (ed25519 signatures)
    - D-H: g=2, p = predetermined 1024-bit prime
    - hashes: SHA-1 (old and unsafe: collision attacks possible), SHA-256, SHA3-256
    - each relay has multiple keypairs:
        - identity keypair: long-lived, identifies relay (ed25519 now, RSA legacy)
        - online signing keys (so we can keep id. keypair offline): ed25519 signing key
            - this key itself is signed with identity pubkey
        - circuit extension keys = onion keys (lifetime: couple weeks)
            - used creating/extending circuit
            - to perform one-way authenticated key-exchange
        