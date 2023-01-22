
# 5.0 Security and Quality of Service

## 5.1 Configure service insertion

## 5.2 Describe Cisco SD-WAN security features

#### SD-WAN fabric security
##### Crypto parameters
- 2048-bit keys with RSA encryption
- Authenticate sender with ESP or AH
- AES 256 bit key to encrypt data
- AES 256 GCM has built in hashing to verify integrity
- Anti-Replay protection is enabled

##### Key negotiation
- No ISAKMP
- Each WAN Edge generates an AES-256 bit key per transport and advertises across control plane (DTLS/TLS) tunnel to vSmart
- Keys are regenerated every 24 hours, rekey timer can be tuned
	- Symmetric keys used in an asymmetric fashion
		- *Example* Data going from cEdge1 to cEdge2. cEdge1 knows cEdge2's key from OMP, and will use that to encrypt data. cEdge2 will then use it's own key to decrypt.

##### Pairwise keys
- Additional security since the same key isn't used across all devices for encrypt/decrypt
- Each WAN edge generates a key per transport **and per peer**, which is advertised to vSmart
	- *Example* cEdge1 sending data to cEdge2. Would use the cEdge1-2 key. cEdge3->cEdge2 would use the cEdge3-2 key
- Backwards compatible, and is disabled by default

### 5.2.a. Application-aware enterprise firewall

### 5.2.b IPS

### 5.2.c URL filtering

### 5.2.d AMP

### 5.2.e SSL and TLS proxy

## 5.3 Describe Cloud security integration

### 5.3.a. DNS security

### 5.3.b. Secure Internet Gateway (SIG)

## 5.4 Configure QoS treatment on WAN Edge routers

### 5.4.a Scheduling

### 5.4.b Queuing

### 5.4.c Shaping

### 5.4.d Policing

### 5.4.e Marking

### 5.4.f Per-tunnel and adaptive QoS