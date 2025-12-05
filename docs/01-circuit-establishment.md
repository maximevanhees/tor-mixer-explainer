2.a. Explain how a Tor client discovers relays and selects nodes for a circuit.
2.b. Describe the telescoping circuit construction process.
2.c. Explain how cryptographic keys are established for each hop.
2.d. Include and reference a Mermaid diagram (`circuit-establishment.mermaid`)


## 2.a 

- Relays are discovered through the Tor directory protocol. The client downloads a list of relays from the directory authority servers. This is basically a list of different relays that are in the network to which the client can connect. These get cached by the client (and relays). 
- (TODO: why are directory authority servers considered "semi-trusted" (explain why "semi"))
- (TODO: info about selection procedure)


## 2.b


## 2.c

- Seperate symmetric key between client and each relay
    - used to encapsulate the onion-like encryption


## 2.d


## ADDITIONAL INFO:

- Client uses relay messages (containing in a multi-encrypted relay cell) to exchange data with different relayes on selected circuit. 
    - simple example
        1. client sends BEGIN message: tells exit relay node to create new stream. Stream gets associated with new TCP connection to target host
        2. exit node replies with CONNECTED message: confirms TCP connection is succesful
        3. now client and exit node exchange DATA messages (DATA messages = contents of anonymized stream)

- Conflux = multiple circuits from client to same exit node
    - each circuit has different relays between client and exit node
- Conflux-set = colelction of the multiple circuits
    - client can choose over which circuit in conflux set to sent messages.

Opening a channel is multi-step process:
1. The initiator opens a new TLS session with certain properties, and the responding relay checks and enforces those properties.
2. Both parties exchange cells over this TLS session in order to establish their identity or identities.
3. Both parties verify that the identities that they received are the ones that they expected. (If any expected key is missing or not as expected, the party MUST close the connection.)
Once this is done, the channel is Open, and regular cells can be exchanged.


- TLS ciphersuites: `AES_128_GCM_SHA256`, `CHACHA20_POLY1305_SHA256` (TLS 1.3)
- Most commonly used for Tor Link Protocol v4/v5 is: `TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384`
    - ECDHE (key-exchange X25519 or P-256): provides forward secrecy
    - ECDSA: for authentication (verify relay's certificate)
    - AES_256_GCM: symm encryption algo. GCM provides authenticatied encryption (confidentialty + integrity)
    - SHA384: Hash function used for PRF to derive keys
- ChaCha20-Poly1305 less popular: relays typically have AES-NI hardware support --> thus AES-256 generally preferred over ChaCha20


![negotiate_initialize_channel](/assets/negotiating-channels-1.svg)


- VERSIONS cell: negoatiate Tor channel protocol version
- CERTS cell: describes key party claims to have + certificates to authenticate that those keys actually belong to related identitiy.
    - bridges gap between TLS connection and relays identity
    - contains a list of certs:
        - link key certificate: proves that identity key has signed the key used for the (established) TLS connection
        - Onion key certificate: used for extending cirtuict proving it is owned by the identity key 
    - without CERTS --> possibility for MITM
        - Because the initial TLS handshake didn't prove the server's identity (to avoid leaking that it is a tor relay), the relay must prove its identity inside the encrypted tunnel
        - It basically makes sure that the key used for the TLS connection actually belongs to the relay you connect wih. TLS only provides encryption, but you don't trust the other party yet!
    - This means: responder's CERTS cells contains:
        - 1 CertType 4 ed25519 (self-signed): proves the master identity key owns the signing key. So this certificate contains the signing key, and the whole certficate is signed using the identity key.
        - 1 CertType 5 ed25519: proves the signing key owns the current TLS connection. This certificate is signed using the signing key and contains the sha-256 hash of the TLS certificate (that was used during the TLS handshake)
- Optional: the initiator must provide a CERTS cell when we want the authenticate:
    - initiator's CERTS cells contains CertType 6 certificate
    - because iniator is not required to provide TLS certificate during TLS handshake, the CERTS cell is not yet sufficient to authenticate the channel until the AUTHENTICATE cell is received. 
- AUTH_CHALLENGE cells: from responder to initator; contains string that initiator must signed as part of their AUTHENTICATE cell. 
- AUTHENTICATE cells: as reponse to the AUTH_CHALLENGE cell
- NETINFO cells: last step of handshake. Used for time synchronization and address verficiation.


- Creating and extending circuits:
    - Circuit are created incrementally: one hop at a time
        - to create new: client sends CREATE2 cell to first node (contains first half of handshake)
        - node reponds with CREATED2 cell (contains second half of handshake); or with DESTROY cell to shut down circuit on handshake failure
        - (TODO: more info about ntor)
    - To extend circuit: client sneds EXTEND2 message (encapsulated in RELAY_EARLY cell) to last node in the circuit. These messages contain link specifiers which describe the next node in the circuit and how to connect to it
    - (TODO?: talk about one-hop circuits (they use CREATE_FAST/CREATED_FAT) 
        - e.g. used when clients wants to download directory information
        - is this what is used for some hidden services that connect to RD point directy?)
    - ntor handshake: uses DH handshakes to create a separate shared key between client and a server (client has 1 shared key between each relay and itself)
        - modern tor versions use ntor-v3 (has support for "extensions")
   - Once circuit is established: both parties will have a shared set of circuit keys that are used to encrypt, decrypt (onion) and authenticate relay cells over that circuit.
        - So: after handshake we have a single shared secret
            - we need to turn that into mutliple keys which are needed for the encryption
                - done using KDF (takes in samll amount of input and stretches it out into massive string of bytes)
            - Tor uses separate keys for:
                - encrypting data going forward (client to relay) = forward encryption key
                - encrypting data going backwards (realy to client) = backward encryption key
                - verifying data integrity (hashing) going forward = forward integrity key
                - verifying data integrity (hashing) coming backwards = backwared integrity key
            - KDF-TOR: used by CREATE_FAST handshake
            - KDF-RFC5869 (used when using ntor and hs-ntor)