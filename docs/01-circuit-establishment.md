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