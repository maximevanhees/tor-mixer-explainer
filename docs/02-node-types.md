3.a. Describe guard, middle, and exit relay nodes.
3.b. Explain what each node can and cannot observe.
3.c. Discuss why these design constraints aid anonymity


## 3.a 

- **Guard Relay**: The first node in the circuit. Follows a strict selection process to ensure that the same guard is not used for all circuits (TODO: explain why). 
- **Middle Relay**: The second node in the circuit. It knows the previous and next hop in the circuit.
- **Exit Relay**: The last node in the circuit. It knows the IP address of the destination server and the previous hop in the circuit.


## 3.b 

General rule is that each node can only observe the previous and next hop in the circuit.

- **Guard Relay**:
  - Can observe: The IP address of the client and the first hop of the circuit.
  - Cannot observe: The content of the traffic, the final destination, or any other hops in the circuit.

- **Middle Relay**:
  - Can observe: The previous and next hop in the circuit.
  - Cannot observe: The IP address of the client, the content of the traffic, or the final destination.

- **Exit Relay**:
  - Can observe: The IP address of the destination server and the previous hop in the circuit.
  - Cannot observe: The IP address of the client or any other hops in the circuit.


## 3.c 


## ADDITIONAL INFO: