3.a. Describe guard, middle, and exit relay nodes.
3.b. Explain what each node can and cannot observe.
3.c. Discuss why these design constraints aid anonymity

## 3.a

In the Tor network, circuits are constructed using three primary types of relay nodes: guard (entry), middle, and exit relays. These roles are determined by flags in the consensus document (e.g., "Guard", "Exit", "Fast", "Stable"), bandwidth weights, and exit policies. Relays advertise capabilities in their descriptors, and authorities vote on flags. A typical 3-hop exit circuit is: Client → Guard → Middle → Exit → Destination. Internal circuits (e.g., for onion services) omit the exit. Relays can serve multiple roles if flagged appropriately, but path selection enforces separation for anonymity.

### Guard (Entry) Relays
- **Role**: Guards are the first hop in a circuit, directly connected to the client. They mitigate predecessor attacks by persisting across sessions. Clients maintain a sampled set {SAMPLED_GUARDS} from consensus relays with the "Guard" flag.
- **Requirements**: Must have "Guard", "Fast", "Stable", "V2Dir" flags. Support ntor-v3 handshakes. Handle CREATE2/EXTEND2 cells. Bridges (special guards) are user-configured for censorship evasion.
- **Behavior**: Decrypt outermost layer of relay cells using per-hop keys (Kf/Kb from ntor-v3). Forward to next hop. Test reachability via self-circuits (tor-spec §5.3.3).

### Middle Relays
- **Role**: Middles are intermediate hops, relaying encrypted traffic without exiting the network or directly connecting to clients. Selected randomly from "Fast" flagged relays. Constraints: no family/subnet overlap with other hops. 
- **Requirements**: "Fast" flag. Support relay cell forwarding. No exit policy checks needed.
- **Behavior**: Decrypt/encrypt relay cells with hop-specific keys (layered onion encryption). Forward without inspecting payload.

### Exit Relays
- **Role and Selection**: Exits are the final hop, connecting to external destinations. Selected first in path building, from "Exit" flagged relays supporting the stream's port/IP.
- **Requirements**: "Exit", "Fast" flags; permissive exit policy (accept/reject patterns).
- **Behavior**: Decrypt innermost layer, connect externally (TCP), relay data via RELAY_DATA. Enforce exit policies. Vulnerable to traffic inspection if unencrypted.

These roles ensure layered anonymity, with guards persistent for security and exits handling external risks.

## 3.b

Tor's layered encryption (onion routing) and leaky-pipe topology limit observations at each node, preventing any single relay from seeing the full path or both endpoints. Relay cells are encrypted end-to-end but unwrapped hop-by-hop using per-hop keys. Observations depend on position, with traffic analysis possible but confirmation attacks mitigated.

### Guard (Entry) Relay Observations
- **Can Observe**:
  - Client's IP address, port, and TLS handshake metadata (e.g., client ciphersuites). Identifies the source (client or bridge).
  - Circuit IDs (CircID, 2-4 bytes, channel-local) and cell types (e.g., CREATE2 for circuit init).
  - Outermost encrypted relay cell headers. After decryption: next hop's address (in EXTEND2 cells), but not full path.
  - Traffic patterns: timing, volume (cell counts), directionality. Can infer stream multiplexing via StreamIDs.
- **Cannot Observe**:
  - Inner payload (multi-layer encrypted). Cannot see destinations, data, or onion service descriptors.
  - Full circuit path (only knows immediate next hop; telescoping hides extensions).
  - Client identity beyond IP.
  - Encrypted handshakes (ntor-v3) prevent key recovery without private keys.

### Middle Relay Observations
- **Can Observe**:
  - Previous and next hop IPs/ports. Knows circuit topology locally (e.g., CircID mappings).
  - Decrypted relay header (after unwrapping its layer): StreamID, digest (Df/Db for integrity), command (e.g., RELAY_DATA).
  - Traffic metadata: cell timing/volume, but payload remains encrypted (inner layers intact).
  - Circuit extensions (EXTEND2 cells) reveal next hop's link specifiers (IP, port, Ed25519 ID), but not beyond.
- **Cannot Observe**:
  - Original client IP or full path (no end-to-end visibility; only adjacent hops).
  - Decrypted payload (inner encryption layers protect content; cannot read streams).
  - Stream origins/destinations (StreamIDs are end-to-end, but middle sees multiplexed anonymized streams).

### Exit Relay Observations
- **Can Observe**:
  - Destination IP/port (from RELAY_BEGIN) and unencrypted payload (if not end-to-end encrypted, e.g., plain HTTP).
  - Previous hop (middle relay's IP/port), but not full path or client.
  - Innermost decrypted relay cells: full StreamID, data (RELAY_DATA).
  - External traffic: exit sees cleartext requests/responses, enabling content inspection (e.g., HTTP headers, if not HTTPS).
  - Traffic patterns: correlates inbound cells to outbound connections.
- **Cannot Observe**:
  - Client IP or earlier hops (only sees middle relay; path obscured by telescoping).
  - Encrypted content (if TLS/HTTPS used externally; Tor doesn't proxy end-to-end encryption).
  - Internal circuit details (e.g., cannot link streams to specific guards without correlation).

All nodes observe TLS metadata (e.g., packet sizes) but not decrypted cells without keys. Global passive adversaries can correlate via timing/volume, but single nodes cannot.

## 3.c