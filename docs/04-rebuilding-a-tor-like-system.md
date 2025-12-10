5.a. Choose and justify a P2P or networking library stack.
5.b. Describe discovery design.
5.c. Decide whether to keep telescoping circuits.
5.d. Highlight intentional design trade-offs.

## 5.a - Choosing a Networking Stack

For rebuilding a Tor-like system today, I would choose **libp2p** (from the IPFS project) as the core networking library, supplemented by Rust's **tokio** for asynchronous I/O and **noise-protocol** for secure handshakes. This isn't about throwing away everything Tor does well, but rather about addressing specific architectural limitations that have become increasingly problematic over the years.

### How Tor's Current Stack Works

Tor's networking layer is built around a fairly traditional approach: custom TLS connections over TCP. When a Tor client wants to communicate with a relay, it establishes a TLS connection, performs a handshake, and then multiplexes multiple circuits over that single encrypted channel. The system uses RSA and SHA-1 for some legacy operations (though these are being phased out), and relies on the ntor handshake protocol for establishing circuit keys.

This approach worked well when Tor was designed in the early 2000s, but it has some notable limitations:

- The stack is **monolithic**. TLS and TCP are tightly coupled, making it hard to swap out components or add new transport options
- **Head-of-line blocking** occurs when multiplexing circuits: if one TCP packet is lost, all circuits on that connection must wait for retransmission, even if their own data arrived successfully
- The handshake patterns and packet sizes can be **fingerprinted** by censors, making Tor traffic identifiable
- **NAT traversal** is difficult, preventing many users behind firewalls from running relays
- Upgrading cryptography has been slow. Ed25519 adoption took years.

### How libp2p Improves the Situation

libp2p takes a fundamentally different approach: it's **protocol-agnostic** and **modular**. Instead of hardcoding TLS-over-TCP, it provides a flexible framework where you can plug in different components for transport, encryption, multiplexing, and discovery.

#### Transport Flexibility

libp2p supports multiple transport protocols: TCP, QUIC, and WebRTC which you can use interchangeably. This solves several of Tor's problems at once:

**QUIC** eliminates head-of-line blocking by giving each multiplexed stream its own flow control. If one circuit loses a packet, the others keep flowing. This could significantly reduce latency for Tor users, especially on unreliable connections. The switch wouldn't require rewriting the entire system, just swapping the transport module.

**WebRTC** brings excellent NAT traversal through built-in hole-punching techniques. This means more users could run relays from behind home routers, expanding the network without requiring port forwarding or VPN setups. It also works natively in browsers, opening up possibilities for browser-based Tor clients.

The beauty is that libp2p handles the protocol negotiation automatically. Two peers can figure out which transports they both support and pick the best one, with fallbacks if needed.

#### Modern Cryptography

libp2p comes with modern crypto primitives out of the box. The **Noise protocol** provides forward-secure handshakes similar to Tor's ntor. Instead of being stuck with aging RSA-1024 keys in legacy code, you get Ed25519 and secp256k1 by default.

#### Decentralized Discovery

Tor's semi-centralized model relies on directory authorities to maintain the network consensus. If these authorities are DDoSed or censored, the entire network can struggle. libp2p's **Kademlia DHT** offers a decentralized alternative where relay information is distributed across the network.

This doesn't mean abandoning trust entirely (see anwser 5.b), but it removes single points of failure. The DHT also handles peer discovery naturally, making it easier for nodes to find each other without central coordination.

#### Performance and Efficiency

libp2p was built for high-throughput systems like IPFS and Filecoin, where millions of nodes transfer large amounts of data. Combined with tokio's async runtime, it handles concurrent operations efficiently.

#### Cross-Platform Development

Unlike Tor's monolithic C codebase, libp2p has mature implementations in Go, Rust, and JavaScript. This makes it easier to build Tor clients for different platforms reimplementing the entire networking stack each time. The ecosystem is battle-tested. (e.g. see IPFS).

### Trade-offs and Remaining Challenges

libp2p isn't perfect. Its gossip protocols can leak metadata if not carefully anonymized. The DHT introduces some latency compared to Tor's direct authority queries. And while the modular design is powerful, it also adds complexity.

But these are manageable challenges. The key insight is that libp2p provides a modern, flexible foundation that addresses Tor's most pressing architectural limitations: transport rigidity, crypto aging, centralization bottlenecks, etc. without a complete redesign of the anonymity model. It's an incremental improvement that makes the system more adaptable to future threats and opportunities.

## 5.b - Discovery Design

The discovery system, which is how clients find relays and learn about the network, is one of Tor's most centralized components. I'd propose a **hybrid approach** that combines Kademlia DHT (distributed hash table) with lightweight directory authorities. This gives us the resilience of decentralization while keeping the trust guarantees that make Tor work.

### How Tor's Current Discovery Works

Tor uses a small set of **directory authorities** (currently about 10 hardcoded servers run by trusted operators). Here's the process:

1. Relays upload their descriptors (public keys, IP addresses, bandwidth, policies) to the authorities
2. Every hour, authorities vote on a consensus document that lists all active relays
3. Clients download this consensus from authorities or mirror servers
4. Clients use the consensus to select relays for building circuits

This works well for reliability because you always know where to get the latest network state. But it creates some vulnerabilities:

- **Single point of failure**: If authorities are DDoSed, the network can't update its consensus
- **Censorship target**: Authorities have known IP addresses that censors can block (solved partially with use of bridges)

### The Hybrid DHT Approach

Instead of relying solely on central authorities, we can distribute relay information across the network using a Kademlia DHT, which is also used by BitTorrent and IPFS. But we don't abandon authorities entirely; we use them for validation and trust anchoring.

#### How the DHT Works

In a Kademlia DHT, every relay gets a unique 160-bit ID (hash of its public key), and relay descriptors are stored on nodes whose IDs are "close" to the relay's ID (using XOR distance). When a client wants to find a relay:

1. Start with a few known nodes (bootstrap seeds)
2. Ask them "who do you know closest to relay X?"
3. Query those closer nodes iteratively
4. Find the relay descriptor

**Key benefit**: The relay information is now distributed across thousands of nodes. No single server failure breaks discovery, and censors can't block a handful of IPs to disable the system.

#### Preventing Sybil Attacks

The obvious concern with DHTs is **sybil attacks** where an adversary creating thousands of fake nodes to control parts of the DHT and serve malicious relay descriptors. We address this through:

**Proof-of-work requirements**: Publishing a relay descriptor requires solving a computational puzzle. This makes it expensive to flood the DHT with fake relays. The difficulty can be tuned based on network conditions.

**Bandwidth proofs**: Alternatively, nodes could prove they've actually relayed traffic before being allowed to publish descriptors. This ties DHT participation to real network contribution.

These mechanisms don't make sybil attacks impossible, but they make them costly enough to be impractical at scale.

#### Keeping Authorities for Trust

Here's where the hybrid design shines: we keep a lightweight authority layer, but change what it does.

**Authority role**: Instead of serving all relay descriptors directly, authorities vote on and sign **consensus snapshots** which are compact summaries of the network state including:
- Shared randomness for path selection
- Network parameters (bandwidth weights, version requirements)
- Merkle tree roots of valid relay descriptors

**Client verification**: When a client fetches a relay descriptor from the DHT, it verifies the descriptor against the authority-signed Merkle root. This prevents malicious DHT nodes from serving fake descriptors (they would need to break authority signatures).

**Fallback mechanism**: If the DHT is under attack or unavailable, clients can fall back to fetching directly from authorities, just like current Tor. This ensures the system degrades gracefully rather than failing completely.

#### Guard Selection Through DHT

Tor uses **guard relays**, wich are a small set of entry nodes that clients stick with for months in order to prevent certain attacks. We preserve this in the DHT design:

1. Client queries the DHT for high-uptime relays
2. Selects a few as persistent guards
3. Uses these guards as DHT entry points for future queries

This reduces the risk of **bootstrap attacks** where an adversary controls the initial nodes you connect to. Your guards become your trusted DHT gateways.

#### Security Measures

**Encrypted queries**: All DHT queries use authenticated encryption (Noise protocol) to prevent tampering and traffic analysis. An adversary can't see what relay you're looking for.

**Rate limiting**: Nodes rate-limit DHT publications to prevent flood attacks. If someone tries to spam the DHT with fake descriptors, they'll be throttled.

**Multiple verification paths**: Clients can query multiple DHT paths and cross-verify results. If different paths return different descriptors for the same relay would mean something is wrong and we could fall back to authorities.

### Addressing the Trade-offs

**Latency**: DHT queries add some overhead compared to Tor's O(1) authority fetches. But we can mitigate this through:
- Caching of relay descriptors
- Preferring authority bootstraps for initial setup
- Parallel DHT queries to reduce bad latency

**Complexity**: The hybrid design is more complex than pure authorities or pure DHT. But this complexity can give us improved resilience where the system works even if parts fail, and it's much harder to censor or attack.

### Why This Works

The hybrid approach gives us the best of both worlds:

- **Decentralized distribution**: Relay information spreads peer-to-peer, eliminating single points of failure
- **Centralized trust**: Authorities still validate what's legitimate, preventing sybil attacks
- **Censorship resistance**: If authorities are blocked, the DHT keeps working
- **Graceful degradation**: If the DHT is attacked, fall back to authorities

It's an incremental improvement over Tor's current system. We are not throwing away the trusted authority model that works, just distributing the load and adding resilience where it's needed most.

## 5.c - Telescoping Circuits: Keep or Replace?

One of Tor's core design decisions is **telescoping circuits**: building paths incrementally, one hop at a time. I'd keep this approach but enhance it with better failure recovery and optional parallelism. Telescoping has proven itself over two decades, and the alternatives come with serious security trade-offs.

### How Telescoping Works in Tor

When you build a circuit in Tor, you don't construct the entire path at once. Instead:

1. **First hop**: Connect to your guard relay, establish encryption
2. **Second hop**: Send an EXTEND2 cell through the guard, asking it to connect to a middle relay
3. **Third hop**: Send another EXTEND2 through both previous hops to reach the exit relay

Each hop only sees its immediate neighbors. The guard doesn't know you're connecting to an exit, and the exit doesn't know you're the original client. The EXTEND2 cells are wrapped in layers of encryption, so intermediate relays can't read the instructions meant for later hops.

This gives you **perfect forward secrecy** (each hop uses ephemeral keys) and **path unlinkability** (no single relay knows the full circuit). It's elegant and secure.

### The Problems with Current Telescoping

The sequential nature creates some practical issues:

**Latency accumulation**: Building a 3-hop circuit requires 3 round trips. If each hop has 100ms latency, you're waiting 300ms just to establish the circuit before sending any actual data. For users on slow connections, this adds up quickly.

**Failure waste**: If the third hop is unresponsive, you've wasted the work of building the first two hops. Tor has to start over with exponential backoff, which can take several seconds. This is frustrating for users and makes the network feel sluggish.

**No parallelism**: You can't probe multiple potential paths simultaneously. If you're in a high-latency region, you're stuck waiting for each sequential attempt.

### Why Not Abandon Telescoping?

Before we talk about improvements, let's consider the alternatives and why they're problematic:

**Source routing** (pre-computing the full path): The client would encrypt the entire path at once and send it to the first relay. This is faster (one round trip), but it exposes the full path to timing analysis. If an adversary can observe when you send the path and when the exit receives it, they can correlate the circuit. Telescoping prevents this by adding independent delays at each hop.

**Full pre-build** (parallel construction): You could try building multiple paths simultaneously and pick the fastest. But this requires more computation (generating keys for all paths) and more network overhead (multiple EXTEND attempts). It also makes you more fingerprintable and thus your traffic pattern becomes distinctive.

Telescoping's sequential nature actually provides security benefits. The delays between hops make timing analysis harder, and the incremental approach means each hop's decision is independent.

### Improvements Without Abandoning the Model

We can address the efficiency problems while keeping telescoping's security properties:

#### Batched Extensions for Low-Sensitivity Circuits

For circuits that don't need maximum anonymity (like fetching public website content), we could allow **batched extensions**: the client sends multiple EXTEND2 payloads in a single cell, and intermediate relays process them in parallel where possible.

This wouldn't work for all circuits. High-security users would still use sequential telescoping. But for everyday browsing, it could cut circuit build time significantly without exposing the full path to any single relay.

#### Redundant Path Building

Add **probabilistic redundancy**: when extending from hop 2 to hop 3, occasionally try two different exit relays in parallel. Whichever responds first wins, and you abandon the other. This adds some overhead but dramatically reduces the impact of unresponsive relays.

The key is making this probabilistic and random. You don't want to create a distinctive traffic pattern that makes you fingerprintable.

#### Adaptive Timeouts

Instead of fixed exponential backoff, use **adaptive timeouts** based on network conditions. If you're on a high-latency connection, don't give up on a hop after 2 seconds—maybe it just needs 5 seconds. If you're on fiber, fail fast.

This could be informed by the DHT-based discovery system from section 5.b. Nodes could publish latency statistics that clients use to set realistic timeouts.


## 5.d - Design Trade-offs

Rebuilding a Tor-like system means making hard choices. You can't optimize for everything simultaneously. Improving one aspect often means accepting compromises elsewhere. Here are the major decisions and how I'd approach them.

### Anonymity vs. Performance

This is the fundamental tension in any anonymity network.

**The problem**: Tor's 3-hop circuits provide strong unlinkability where no single relay sees both the client and destination. But each hop adds latency due to queuing, encryption overhead, and network delays. For web browsing, this is tolerable. For real-time applications like VoIP or gaming, this is less optimal.

**The trade-off**: Keep 3 hops as the default, but allow users to configure 2-hop circuits for low-threat scenarios. A 2-hop circuit is faster (roughly 33% less latency) but weaker against **predecessor attacks**. In that case if an adversary controls both relays, they can correlate traffic. 

### Decentralization vs. Trust

**The problem**: Tor's directory authorities are centralized chokepoints. DDoS them, and the network struggles. Censor them, and users in restrictive countries can't bootstrap. But they also provide reliable consensus by providing a legitimate relay list.

**The trade-off**: The hybrid DHT approach distributes the load while keeping authority signatures for validation. But DHT queries are slower and protecting against sybil attacks might require some proof-of-work, which adds computational overhead. But you gain resilience. If authorities are blocked, the DHT keeps working. If the DHT is attacked, authorities provide fallback. Neither component is a single point of failure. Going fully P2P, meaning that we have no trusted authorities would be expose vulnerability to sybil attacks.