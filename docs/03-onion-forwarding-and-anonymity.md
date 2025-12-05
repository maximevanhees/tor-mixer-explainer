4.a. Explain layered onion encryption and hop behavior.
4.b. Describe anonymity guarantees and their limits.
4.c. Include a Mermaid diagram (`onion-routing-flow.mermaid`)

## 4.a
- Layer onion encrpytion:
    - Client has a symmetric key with EACH relay on the circuit
    - (TODO: talk about how key-exchange work and how symmetric keys are established between client and each relay).
    - Basically really like an onion: each exchange of a relay cell adds a layer of encryption. Chain them together and we get onion-like structure where each exchange (between client and it's selected relays) is encapsulated with it's own layer of encryption.


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
        