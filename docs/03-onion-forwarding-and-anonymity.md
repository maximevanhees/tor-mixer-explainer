4.a. Explain layered onion encryption and hop behavior.
4.b. Describe anonymity guarantees and their limits.
4.c. Include a Mermaid diagram (`onion-routing-flow.mermaid`)

## 4.a - Layered Onion Encryption

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

## 4.b - Anonymity Guarantees and Limits

Tor provides probabilistic anonymity by making it hard to link senders and receivers, obscure traffic patterns, and distribute trust across the network. However, these protections have limits, especially against powerful or global attackers. Anonymity is based on models where partial observers cannot easily confirm connections, measured by concepts like k-anonymity (hiding among k users) and entropy (uncertainty in linking).

### Anonymity Guarantees
- **Sender/Receiver Unlinkability**: Clients blend into large groups of about 3 million daily users. Guards remain consistent over sessions (lasting 2-3 months), limiting how many entry points see a client's IP and reducing risks from repeated observations. Layered encryption means no single relay sees both the source and destination: guards spot sources but not final targets (due to encrypted payloads), exits handle destinations and cleartext but not origins, and middles see neither. Paths are chosen with bandwidth-based weights to spread selections evenly, making it tougher for attackers to control enough relays to trace traffic.
- **Resistance to Traffic Analysis**: Cells are fixed-size (514 bytes) to hide packet lengths. Multiple streams share circuits (via StreamIDs), and the leaky-pipe setup mixes traffic flows, complicating efforts to match volumes or timings. Encryption uses counter-mode AES (non-deterministic) with integrity checks, blocking changes to data. Features like padding cells and flow control add variability to patterns.
- **Forward Secrecy**: Keys for each handshake are temporary, so even if a relay is compromised later, past traffic stays secure.
- **Hidden Services**: Rendezvous points allow mutual anonymity: services keep their IPs secret through introduction points, and clients connect via these points without exposing the service's location.

### Limits of Guarantees
- **Global Passive Adversary**: If someone watches all network traffic (like a group of ISPs), they can match patterns at entry and exit points using timing or volume (e.g., comparing packet arrival times with high accuracy). Tor assumes attackers aren't everywhere; a full global view breaks protections in any fast anonymity system.
- **Traffic Confirmation**: Attackers who suspect a connection can confirm it by adding patterns (like slowing traffic) or just watching both ends. Active versions involve injecting delays; passive ones rely on natural variations. Tor reduces but doesn't eliminate this—owning entry and exit fully reveals links.
- **Intersection Attacks**: Watching over time can narrow down users by seeing who is active when (intersecting changing user groups). Guards help but don't stop this if the guard itself is compromised, allowing links across sessions.
- **Sybil and Denial-of-Service**: Attackers could flood with fake guards or exits to skew paths, or overload nodes to force rerouting. Tor counters with bandwidth requirements for flags and exit policies, but it's not foolproof.
- **Endpoint Weaknesses**: Exits view unencrypted data (if not using HTTPS), risking snooping. Clients can be tricked by malware. Tor doesn't protect beyond its edges.

Tor focuses on real-world usability over absolute anonymity, offering strong protection against common threats while recognizing theoretical vulnerabilities.


## 4.c - Onion Routing Flow Diagram
Please see Mermaid diagram: [here](/diagrams/onion-routing-flow.mermaid)