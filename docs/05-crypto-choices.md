6.a. Explain chosen asymmetric cryptography.
6.b. Explain chosen symmetric cryptography.
6.c. Justify choices for performance and security.

## 6.a - Asymmetric Cryptography Choices

For a rebuilt Tor-like system, I'd use **Ed25519** for identity keys and **X25519** for key exchange. This builds on what Tor has been moving toward while addressing the slow crypto evolution.

### How Tor Currently Works

Tor uses RSA-1024 for relay identity keys (with newer Ed25519 support) and the **ntor protocol** (X25519-based) for circuit key exchange. The problems: RSA-1024 is weak by modern standards, supporting both RSA and Ed25519 creates complexity, and there's no clear path to post-quantum resistance.

### Why Ed25519 and X25519?

**Ed25519** (signatures) and **X25519** (key exchange) are part of the Curve25519 family, designed for security and performance:

**Performance advantages**:
- Ed25519 signing: ~0.04ms vs RSA-2048's ~1-2ms
- Compact: 32-byte keys, 64-byte signatures vs RSA's 256 bytes each
- X25519 key exchange: ~0.05ms per operation

**Security benefits**:
- 128 bits of security (equivalent to RSA-3072)
- Deterministic signatures (no RNG vulnerabilities like ECDSA)
- Forward secrecy through ephemeral keys

**Simplicity**: Using the same curve family for both operations simplifies the codebase and reduces attack surface.

### Integration with Noise Protocol

As mentioned in section 5.a, we'd use the **Noise protocol framework** for handshakes. Noise provides clean, well-specified authentication and key exchange patterns, and is algorithm-agnostic.

### Post-Quantum Preparation

Use a **hybrid approach**: combine X25519 with **Kyber** (NIST's ML-KEM standard). An adversary must break both to compromise the session:
- X25519: security against classical computers (today)
- Kyber: security against quantum computers (future)

Kyber-768 adds ~1KB to handshakes and ~0.05ms latency. We could start with pure X25519, and add support for Kyber later.

## 6.b - Symmetric Cryptography Choices

For symmetric encryption, I'd use **ChaCha20-Poly1305** as the primary cipher, with **AES-256-GCM** as a fallback for hardware-accelerated environments.

### How Tor Currently Works

Tor uses **AES-128-CTR** for encrypting relay cells, creating "onion" layers. The problems: no built-in authentication (relies on protocol-level integrity) and not the fatest in software implementations.

### Why ChaCha20-Poly1305?

**ChaCha20-Poly1305** combines the ChaCha20 stream cipher with Poly1305 authentication (used by Google, OpenSSH, WireGuard):

**Performance**:
- ChaCha20: ~1-2 GB/s per core (software)
- AES-128 (software): ~0.5-1 GB/s per core
- AES-128 (hardware with AES-NI): ~5-10 GB/s per core

For devices without AES-NI (many ARM processors, mobile devices), ChaCha20 is 2-3× faster.

**Security**:
- Built-in authentication via Poly1305
- Constant-time implementation
- 256-bit keys for long-term security
- Simple nonce handling (counter per cell)

**Integration**: libp2p supports ChaCha20-Poly1305 natively, providing consistent crypto across the stack.

### AES-256-GCM as Fallback

Support **AES-256-GCM** for hardware acceleration. On CPUs with AES-NI, AES-GCM reaches 5-10 GB/s which is crucial for high-bandwidth relays. Clients and relays negotiate during the Noise handshake: mobile devices prefer ChaCha20, servers with AES-NI prefer AES-GCM.

### Key Derivation

Use **HKDF-SHA256** to derive circuit keys from the handshake shared secret. HKDF is standardized and well-analyzed, expanding the shared secret into forward/backward encryption and authentication keys.

## 6.c - Performance and Security Justification

The crypto choices balance security, performance across diverse hardware, and practical deployability.

### Security Comparison

| Aspect | Current Tor | Proposed System | Improvement |
|--------|-------------|-----------------|-------------|
| Identity keys | RSA-1024 + Ed25519 | Ed25519 only | Simpler, smaller, faster |
| Key exchange | ntor (X25519) | Noise + X25519 | More flexible, easier to upgrade |
| Circuit encryption | AES-128-CTR | ChaCha20-Poly1305 | Authenticated, faster on mobile |
| Post-quantum | None | Hybrid X25519+Kyber | Future-proof |
| Key derivation | Custom KDF | HKDF-SHA256 | Standardized |

**Security levels**:
- Ed25519/X25519: ~128 bits (sufficient against classical computers)
- ChaCha20-256: 256 bits (future-proof)
- Hybrid Kyber: quantum-resistant key exchange
- Constant-time implementations prevent side-channel attacks