# DMVPN

## Philosophy of P2P vs MP
- Nonbroadcast, multiaccess network (NBMA)
    - Like ISDN, ATM, X25, Frame Relay (if configured in multipoint manner)
    - Diagram != P2P or MP (Ethernet always MP, PPP, HDLC always P2P), encapsulation indicates type
    - P2P network - no mapping needed
    - MP network, requires mapping (ARP = L2:L3 mapping)
    - DMVPN mapping L3:L3
- Requires a *full mesh* underlay transport (i.e. internet, MPLS)
- *NBMA IP* - registered IP address from provider (underlay IP)
- *Tunnel IP* - doesn't mean private/public, just means inside of tunnel (overlay IP)
- Best routing protocols: EIGRP or BGP - OSPF, not as reliable in DMVPN
## Phase 1, 2, 3 designs
- None of them are obsolete, just separate designs
- Hub router alwaysconfigured in mutlipoint manner

Hub config
```
interface Tunnel100
    ip address <tunnel.ip> <tunnel.mask>
    tunnel source <nbma.ip>
    tunnel mode gre multipoint
    ip nhrp network-id 100
    ip nhrp map multicast dynamic ! Look into NHRP table for all NBMA IPs
    no ip split-horizon eigrp 100
```
Routing protocol config
```
router eigrp 100
    network <tunnel.ip> 0.0.0.0 ! better than 0.0.0.255, more specific
    network <other.networks>
```

- Hub is configured in a multipoint manner, can only unicast by default. Must use `ip nhrp map multicast dynamic` to create a table of NBMA addresses to multicast to.
    - EIGRP needs to be able to multicast hellos to spokes
    - `show ip nhrp multicast` Lists all registered multicast (NBMA) IPs.
    - Could also just map statically:
    ```
    ip nhrp map multicast <spoke.2.nbma.ip>
    ip nhrp map multicast <spoke.3.nbma.ip>
    ```

`tunnel mode gre multipoint` (mGRE or multipoint GRE) no tunnel desination specified, but requires a **mapping**

i.e. to get to tunnel IP *x.x.x.x*, go to NBMA IP *y.y.y.y*

NHRP (Next Hop Resolution Protocol) used to create this mapping
- NHRP network-id is *locally significant*, similar to a PID.

- NHRP **must** run on hub to provide mapping, but *not requied* on spoke. Running on spoke is common and provides scalability/zero touch on hub by sending their mapping to hub. Could just use static mappings...
```
interface Tunnel100
    ip nhrp map <tunnel.ip> <nbma.ip> ! Spoke 2
    ip nhrp map <tunnel.ip> <nbma.ip> ! Spoke 3
```

- NHRP has *servers* and *clients*
    - Configured on spokes (client points to server)
    - Could just leave everything as a server and it would *work*, but not efficient


### Phase 1
- Data plane = hub and spoke
- Spokes configured in P2P manner
- Spoke to spoke communication through hub router

Spoke config
```
interface Tunnel100
    ip address <tunnel.ip> <tunnel.mask>
    tunnel source <nbma.ip> ! OR <nbma.intf>
    tunnel desination <hub.nbma.ip>
    ip nhrp network-id 100
    ip nhrp nhs <hub.tunnel.ip>
    ip nhrp registration no_unique ! Optional: If we are DHCP client
    ip nhrp registration timeout 5 ! Optional: For troubleshooting
```

`ip nhrp registration no_unique` "The IP I am registering with you is not unique and might change, I'm a DHCP client". Hub would discard new registrations with different IP.

`ip nhrp registration timeout X` "Send NHRP registration every X seconds, can clear issues if hub loses NHRP registrations

### Phase 2
- Data plane = full mesh
- Is basically a glorified phase 1 **unless you manipulate the IGP**
- Really better to move to phase 3

Hub configuration changes
- Manipulate IGP to not change next hop on recieved routes
```
interface Tunnel100
    no ip next-hop-self eigrp 100
```

Spoke configuration changes
- Change to mGRE
- Loses multicast capability, need to use NHRP for multicast capability
```
interface Tunnel100
    ! Change to mGRE
    no tunnel destination <hub.nbma.ip>
    tunnel mode gre multipoint

    ! Use NHRP for multicast resolution
    ip nhrp map multicast <hub.nbma.ip>

    ! Statically map hub tunnel.ip:nbma.ip
    ip nhrp map <hub.tunnel.ip> <hub.nbma.ip>
```

*Note:* Can also use `ip nhrp nhs <hub.tunnel.ip> nbma <hub.nbma.ip> multicast`
to configure NHS, static map, and multicast at same time

Why is first traceroute through hub, followed by the rest through spoke?
- spoke didn't know remote spoke's NBMA address, and had to send resolution request to NHRP NHS

### Phase 3

- Data plane = full mesh
- Allows for summarization from the hub (which is not possible in Phase 2, b/c of routing protocol manipulation)
- Basically uses NHRP instead of routing table for spoke-to-spoke routing

Hub configuration changes (from Phase 2):
```
interface Tunnel100
    ip nhrp redirect
```

Spoke configuration changes (from Phase 2):
```
interface Tunnel100
    ip nhrp shortcut
```

**Summarize routes on hub**:

- Uses NHRP table to resolve more specific (spoke-to-spoke) routes
- First packet traverses hub, hub notifies sender using traffic indication to notify sender of existance of better path - which triggers ender to start a resolution request
```
interface Tunnel100
    ip summary-address eigrp 100 0.0.0.0 0.0.0.0
```

## Operation
## Control and Data Plane

- Control plane is always hub and spoke
- Data plane depends on 'phase'

## Split horizon

- Enabled by default
- Router sends updates back out same interface with unreachable metric
- Disable as part of hub config `no ip split-horizon eigrp <pid>`


## DMVPN Messages
### Register Requests

- Notify NHRP server of *tunnel.ip:nbma.ip* mapping

**Packet:**

- ENET header - protocol = IP (0x0800)
- IP header - ip.src = <spoke.nbma.ip>, ip.dst = <hub.nbma.ip>, protocol = GRE (47)
- GRE header - protocol = NHRP (0x2001)
- NHRP header
    - Fixed header - Protocol-type = IPv4, Version = 1 (RFC2332), Hop-count, Packet-type= Reg_Req
    - Mandatory part - src NBMA = <nbma.ip>, src Protocol = <tunnel.ip>, dst Protocol = <hub.tunnel.ip>
    - Flag Uniqueness bit = true **|| false if we are DHCP client**

### Register Replies

- Notify NHRP client of successfuly registration request

**Packet:**

- ENET header - protocol = IP (0x0800)
- IP header - ip.src = <hub.nbma.ip>, ip.dst = <spoke.nbma.ip>, protocol = GRE (47)
- GRE header - protocol = NHRP (0x2001)
- NHRP header
    - Fixed header - Protocol-type = IPv4, Version = 1 (RFC2332), Hop-count, Packet-type= Reg_Reply
    - Mandatory part - src NBMA = <nbma.ip>, src Protocol = <tunnel.ip>, dst Protocol = <hub.tunnel.ip>

### Resolution Requests

- Ask NHRP server for NBMA IP for a specified tunnel IP
- NHRP server forwards request to destination spoke **does not respond itsself**
- Resolution request contains requestors NBMA IP and Tunnel IP

**Packet:**

- ENET header - protocol = IP (0x0800)
- IP header - ip.src = <spoke.nbma.ip>, ip.dst = <hub.nbma.ip>, protocol = GRE (47)
- GRE header - protocol = NHRP (0x2001)
- NHRP header
    - Fixed header - Protocol-type = IPv4, Version = 1 (RFC2332), Hop-count, Packet-type = Res_Req
    - Mandatory part - src NBMA = <spoke.nbma.ip>, src Protocol = <spoke.tunnel.ip>, dst Protocol = <dest.spoke.tunnel.ip>

### Resolution Replies

- Destination spoke populates it's NHRP table, then replies directly to requestor

**Packet:**

- ENET header - protocol = IP (0x0800)
- IP header - ip.src = <dest.spoke.nbma.ip>, ip.dst = <spoke.nbma.ip>, protocol = GRE (47)
- GRE header - protocol = NHRP (0x2001)
- NHRP header
    - Fixed header - Protocol-type = IPv4, Version = 1 (RFC2332), Hop-count, Packet-type = Res_Reply
    - Mandatory part - src NBMA = <spoke.nbma.ip>, src Protocol = <spoke.tunnel.ip>, dst Protocol = <dest.spoke.tunnel.ip> **same as initial request**
    - Client info entry - Client NBMA = <dest.spoke.nbma.ip>, Client Protocol = <dest.spoke.tunnel.ip>

### Traffic Indication

- Hub notifies spoke that there is a better path
- This triggers spoke to send a resolution request, then remote spoke provides the path to spoke

**Packet:**

- ...
- NHRP header
    - Fixed header
    - Mandatory part
    - Traffic indication - src NBMA <hub.nbma.ip>, src Protocol <hub.tunnel.ip>, dst Protocol <spoke.tunnel.ip>
    - Packet causing indication - IPv4, ICMP, src: <src.traffic.ip>, dst: <dst.traffic.ip>

# DMVPN Labs
- Interesting Traffic
- Per Tunnel QoS
- DMVPN Clustering
- DHCP and DMVPN

**Configure these before next class**