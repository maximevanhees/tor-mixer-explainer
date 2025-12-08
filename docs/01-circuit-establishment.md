2.a. Explain how a Tor client discovers relays and selects nodes for a circuit.
2.b. Describe the telescoping circuit construction process.
2.c. Explain how cryptographic keys are established for each hop.
2.d. Include and reference a Mermaid diagram (`circuit-establishment.mermaid`)


## 2.a 

A Tor client discovers relays through a distributed directory system that ensures a consistent and authenticated view of the network topology. The client's primary mechanism for relay discovery is obtaining a *consensus document* from directory authorities or caches, which aggregates signed votes from a set of semi-trusted directory authorities.

### Step 1: Bootstrapping and Directory Authorities
- **Pre-loaded Authorities**: Each Tor client binary is pre-configured with a hardcoded list of directory authorities. These are well-known, long-term servers run by independent operators. The list includes their IP addresses, ports, and identity fingerprints (digests of their long-term RSA or Ed25519 identity keys).
- **Initial Consensus Fetch**: Upon startup or when lacking a valid consensus, the client attempts to download a consensus from these authorities or fallback directory mirrors (caches).
- **Consensus Validity**: The consensus is a signed document (using RSA or Ed25519 signatures from a majority of authorities) containing a list of all valid relays, their IP addresses, ports, onion keys, bandwidth estimates, exit policies, and flags (e.g., Guard, Exit, Fast, Stable). It is valid for a specific time window. Clients verify signatures against known authority keys to prevent tampering.

### Step 2: Consensus Flavors and Microdescriptors
- **Flavors**: Clients can request different consensus flavors, such as "ns" (network status, full descriptors) or "microdesc" (microdescriptors), a stripped-down version for efficiency. Microdescriptors contain hashes (SHA-256) of relay info, reducing download size.
- **Descriptor Fetching**: Once a consensus is obtained, the client identifies relays via their digests. Descriptors include detailed info like public keys, exit policies, and family declarations (to avoid related relays in one circuit). Clients cache consensuses and descriptors and refresh them at radom intervals.

### Step 3: Node Selection for a Circuit
Once relays are discovered, selection for a circuit (typically 3 hops: entry/guard, middle, exit) follows strict rules to balance anonymity, performance, and security. Selection is probabilistic, weighted by bandwidth and flags from the consensus.

- **Guard Selection (First Hop)**: To mitigate predecessor attacks (where an adversary observes traffic over time), clients use a small set of *entry guards* (typically 3 primary). Guards are relays with the "Guard" flag (stable, fast, high uptime). Guards persist across sessions (lifetime: 2-3 months) to limit exposure. Guards are confirmed usable after successful circuit builds.
  
- **Middle Node Selection (Second Hop)**: Middle nodes are chosen from relays with the "Fast" flag (bandwidth > median or >100KB/s). Selection is random, weighted by consensus bandwidth weights. Constraints: no family overlap, distinct subnets (/16 IPv4, /32 IPv6), and not the same as guard or exit.

- **Exit Node Selection (Third Hop)**: Exits must support the stream's port/IP (checked via exit policy summaries in consensus/microdescriptors).


## 2.b

Tor's circuit construction uses a *telescoping* (incremental) process, where circuits are built one hop at a time, negotiating keys and extending the path without revealing the full path to intermediate nodes. This provides perfect forward secrecy. Circuits are multiplexed over TLS channels between relays, using fixed-size (514-byte) cells for communication.

### High-Level Overview
- **Circuit Structure**: A typical exit circuit is 3 hops: Client → Guard (Entry) → Middle → Exit. Internal circuits (e.g., for onion services) omit the exit.
- **Cell Types**: Circuits use control cells (CREATE2, CREATED2) for building and relay cells (EXTEND2, EXTENDED2) for extension. Relay cells are encrypted end-to-end but unwrapped hop-by-hop.
- **Handshake**: Modern Tor uses the ntor-v3 handshake (Curve25519 ECDH + extensions)

1. **Path Selection**: Before building, select nodes per as described in _2.a_. Client chooses exit first (based on port/policy), then guard, then middle (front-to-back for telescoping efficiency).

2. **Initial Hop (Client to Guard)**:
   - Client sends a CREATE2 cell to the guard over a TLS channel (which was established via handshake).
   - Payload: Handshake type, client's ephemeral Curve25519 public key (X) and optional extensions
   - Guard responds with CREATED2: Ephemeral key Y, authentication tag, and encrypted server message (if extensions used).

3. **Extending the Circuit (Telescoping)**:
   - To add hop N (e.g., middle), client sends RELAY_EARLY cell with EXTEND2 payload to hop N-1.
   - EXTEND2: Link specifiers (IP, port, Ed25519/RSA IDs), handshake type/data (ntor-v3).
   - Hop N-1 forwards as CREATE2 to hop N, which responds with CREATED2.
   - Hop N-1 wraps CREATED2 in EXTENDED2 and sends back to client.
   - Client performs ntor-v3 with hop N, deriving new keys (layered on previous encryptions).
   - Encryption: Each relay cell is encrypted with all hops' keys (onion-like), but built incrementally: client encrypts payload with hop N's key, then hop N-1's, etc.

4. **Completion and Multiplexing**:
   - Circuit completes when all hops respond successfully. Client assigns CircID.
   - Multiple streams multiplex over one circuit (StreamID in relay cells). Streams attach via RELAY_BEGIN.

### Teardown
- **Full Teardown**: DESTROY cell (with reason) propagates, closing all streams.
- **Partial**: RELAY_TRUNCATE to truncate from a hop, sending DESTROY forward and TRUNCATED back. Allows cannibalization (re-extending from truncation point).

This telescoping ensures no hop knows the full path.

## 2.c

Tor establishes per-hop symmetric keys using the ntor-v3 handshake, a 1-RTT ECDH protocol with extensions for efficiency and forward secrecy. Keys are derived for layered encryption/digests, ensuring traffic is onion-encrypted (unwrapped hop-by-hop but end-to-end secure).

### Per-Hop Key Types
- **Forward/Backward Keys (Kf/Kb)**: AES-256-CTR for encrypting/decrypting relay cells in each direction.
- **Digests (Df/Db)**: SHA3-256 for integrity (forward/backward).

### Layered Encryption
- **Relay Cells**: Encrypted with all hops' Kf/Kb (telescoping). Hop i decrypts its layer, checks digest (Df/Db), forwards.
- **Forward Secrecy**: Ephemeral keys per handshake; compromising one doesn't reveal past traffic.

This ensures secure, authenticated key establishment per hop without full path exposure.


## 2.d
Please see Mermaid diagram: [here](/diagrams/circuit-establishment.mermaid)