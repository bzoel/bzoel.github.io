---
search:
  exclude: true
---

# CCNP ROUTE 4.0 VPN Technologies

## 4.1 Configure and verify GRE
* tunneling protocol to encapsulate wide variety of packet types "virtual P2P link"
* Characteristics of GRE
    * protocol-type field in header - supports encapsulation of any L3 protocol
    * IP protocol 47
    * stateless, no flow control
    * no strong security
    * GRE header + tunneling IP header = at least 24 bytes of overhead

Router 2 configuration
```
interface Tunnel1
  ip address 192.168.0.1 255.255.255.252     !! 'inner' tunnel IP
  tunnel source Loopback0                    !! source int on R1
  tunnel destination 4.3.2.1                 !! IP of R2
```

Router 2 configuration
```
interface Tunnel1
  ip address 192.168.0.2 255.255.255.252
  tunnel source Loopback0
  tunnel destinatio 8.7.6.5
```

Verification
```
R1# show interfaces Tunnel1
Tunnel1 is up, line protocol is up
 Hardware is Tunnel
 ...
 Encapsulation TUNNEL
 Tunnel source 8.7.6.5 (Loopback0), destination 4.3.2.1
 Tunnel protocol/transport GRE/IP
 Tunnel TTl 255, Fast tunneling enabled
 Tunnel transport MTU 1476 bytes
 ...
```

<hr>

## 4.2 Describe DMVPN (single hub)
* combines mGRE tunnels, IPsec encryption, and NHRP
* Benefits
    * less configuration on hub (single mGRE tunnel)
    * NHRP to support dynamic IPsec tunnels
    * supports DHCP spokes
* **mGRE:** single GRE interface can support multiple GRE tunnels
    * requires NHRP to learn peer's address and build dynamic tunnels
* **NHRP:** used by routers to determine IP of next hop
    * client-sever protocol (hub = server, spokes = clients)
    * client registers it's inner (tunnel) and outer (physical) addresses
    * NHRP query can be used for spoke-to-spoke communication

Verifying NHRP
```
# show ip nhrp
192.168.0.2 255.255.255.255, tunnel 100 created 0:00:10 expire 1:59:49
  Type: dynamic Flags: authoratative             ! authoratative includes a next-hop server provided the NHRP information
  NBMA address: 10.1111.1111.1111
```

* **IPSec:** provides security
    * Provides 4 services
        * Confidentiality (encryption) - no eavesdropping (uses encryption)
        * Data integrity - verify data isn't changed in path (uses checksums)
        * Authentication - ensures connection with desired partner (uses IKE)
        * Antireplay protection - verify each packet is unique/not duplicated (compare packet's seq # w/ sliding window on destination - drop late/duplicate pkts)
    * Authentication (PSK or certificates) / encryption - important for DMVPN
    * Relies on Authentication Header (AH) protocol 51, or  Encapsulating Security Protocol (ESP) protocol 50
        * ESP - encrypts original packet
        * AH - no encryption
        * AH & ESP - can operate in tunnel mode or transport mode
            * Transport mode - uses packet's original IP header, instead of an additional tunnel header. Works well when increasing packet size is an issue. (typical in client-to-site VPN)
            * Tunnel mode - encapsulates the entire packet, so encapsulated packet has a new (IPSec header). Src/Dst reflects the VPN termination devices (typical in site-to-site VPN)

Verifying IPSec
```
show crypto ipsec sa
 interface: GigabitEthernet0/0
 Crypto map tag: outside_map, local addr 8.7.6.5
 local ident (addr/mask/prot/port) (50.0.0.0/255.255.255.0/0/0)
 remote ident (addr/mask/prot/port) (51.0.0.0/255.255.255.0/0/0)
 current peer: 4.3.2.1
 ...
  local crypto endpt.: 8.7.6.5, remote crypto endpt.: 4.3.2.1
  ...
  inbound esp sas:
    ...
    transform: esp-3des esp-md5-hmac ,
    ...
  outbound esp sas
    ...
    transform: esp-3des esp-md5-hmac ,
    ...
```

<hr>

## 4.3 Describe Easy Virtual Networking (EVN)
* provides traffic separation and path isolation
    * simplify L3 virtaulization
    * improve support for shared services
    * enhance management/troubleshooting
* uses VRF-lite to simplify Layer 3 virtualization
    * traditional VRF-lite requires one subinterface per VRF throughout the datapath
* Use `vnet trunk` instead of multiple subinterfaces
* Route replication improves shared services
    * Traditionally - BGP with import/export is required
    * Link routes from a shared VRF to several segmented VRFs
    * Not dependant on BGP route target / route distriguisher
    * Removes duplicate routing talbes/routes
* Routing context command allows 'troubleshooting within the VRF'
    * `# routing-context vrf red` doesn't require 'VRF' to be specified in commands
    * `R1%red# show ip route` short for `R1# show ip route vrf red`