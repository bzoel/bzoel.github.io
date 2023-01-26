# Multicast

## Overview

### What is Multicast?
- One source, multiple receivers
- Different from broadcast: multicast only recieves packets if receiver asks for them
- Best utilization of resources because data is only transmitted through the network where it is needed

### Multicast alternatives

#### Replicated Unitcast
Source sends a unicast stream to each client. Bad because of multiple similar streams from source

*Downside:* source can become overwhelemed, inefficient bandwidth usage

#### Directed Broadcast
Broadcast to a particular subnet (i.e. 172.16.1.255), and is routable. Standard broadcast (255.255.255.255) is non-routable.

``` title="Enable/disable directed broadcast"
interface <intf-name>
 no ip directed-broadcast
```

*Downside*: recievers that don't want/need the data still have to listen and choose to ignore

### Layer 3 Multicast
- Src IP: still unicast
- Dst IP: multicast group
    - Class D IP
    - 224.0.0.0 - 239.255.255.255 (IPs beginning with `1110`)
    - Class D is flat, no subnetting
- Some ranges for specific purposes
    - 224.0.0.0/24 - local network control block (routing protocols)
        - TTL of 1 or 2
    - 232.0.0.0/8 - source-specific multicast block, streams with known sources
    - 239.0.0.0/8 - organization-local scope, comparible to RFC1918, use within an AS

### Layer 2 Multicast
- Dst MAC: can't use ARP to determine
    - Sender/receiver should agree on a single destination MAC
    - Receivers accept frames with the dest MAC

#### Mapping L3 to L2
Mapping procedure generates a well-known multicast MAC from a multicast IP

- Uses MAC OUI `01:00:5E` (first 3 bytes)
    - `01` 
        - First bit `0` means assigned by IEEE, `1` means made up
        - Second bit `1` means broadcast/multicast, `0` means unicast
- Remaining 24 bits (3 bytes)
    - bit #1 = `0` for multicast
    - bits #2-24 = 23 bits for mapping IP to MAC
        - This causes ambiguity because not enough bits to represent all IPs
            - If two multicast L3 IPs are used that share the same MAC, the end hosts will still receive the frame, review the L3 IP, and process **OR** drop
        - Lower 23 IP address bits map to these 23 bits
        - 32 (2^5) IP addresses all map to same MAC
            - *ex:* 230.1.2.3, 230.129.2.3, 239.1.2.3 map to `01:00:5E:01:02:03`
    
### Multicast Routers
- Guide packets away from a source, *not* towards a destination.
- Track upstream interfaces (closest to source), and destination interfaces (with interested receivers)

### Multicast Routes
- Tracks multicast groups and sources

`(Src,Group) or (S,G)`
: Forwarding state with specific source. "Sorce specific multicast"

`(*,Group) or (*,G)`
: Forwarding state with any/unknown source. "Any source multicast"

``` title="Example mroute"
# show ip mroute 239.255.255.253
(*, 239.255.255.253), 4w1d/00:03:26, RP 10.250.0.115, flags: SJC
  Incoming interface: Null, RPF nbr 0.0.0.0
  Outgoing interface list:
    TenGigabitEthernet1/1/8, Forward/Sparse, 2w0d/00:03:26
    Vlan200, Forward/Sparse-Dense, 2w6d/00:02:22
    TenGigabitEthernet2/1/8, Forward/Sparse, 4w1d/00:03:22

(10.250.32.230, 239.255.255.253), 00:00:07/00:02:52, flags: 
  Incoming interface: TenGigabitEthernet2/1/8, RPF nbr 10.250.255.130
  Outgoing interface list:
    Vlan200, Forward/Sparse-Dense, 00:00:07/00:02:52
```

- Incoming interface: the upstream interface
    - No ECMP in the multicast table, there is a tie breaker
        - For multiple groups, even and odd groups are 'load split'
- Outgoing interface list: the downstream interface(s)

#### Multicast Routing Protocol (PIM)
`PIM`
: Protocol independent multicast

##### Primary Responsibilities per route
1. Identify upstream interface (IIF)
2. Identify downstream interfaces (OIL)
3. Maintain dynamic multicast trees

### Multicast Trees
- Root depends on type of group
    - (S,G) trees are rooted at source
    - (*,G) trees are shared trees

### Loop Prevention
`RPF Check`
: Does source IP of packet correspond to IIF? If not, drop it.

**Multicast only receives packets from the IIF**

### IGMP
`IGMP`
: Internet Group Management Protocol

- Receiver (host) signals it's interest in a multicast stream to the closest routers on the network
- Closest routers to the receiver are LHR: last hop routers
- Closest router to the source are FHR: first hop router

#### IGMP v1/v2
- Routers are signaled for ASM (*,G). SSM (S,G) not supported in v1/v2.

``` title="Join a multicast group on IOS"
interface <intf>
 ip igmp join-group <multicast-ip>
```

##### IGMP Messages
1. IGMP Membership report (JOIN)
    - Host -> LHR "I would like to join this group"
    - Src: Host IP, Dst: Multicast group IP
2. IGMP Membership query, general (QUERIER)
    - ? -> Downstream routers "Which groups are you interested in?"
    - Src: Router IP, Dst: 224.0.0.1 (All IP Hosts)

``` title="Enable Multicast routing on Cisco router"
! Enable multicast globally
ip multicast-routing <distributed>

! Enable multicast on all multicast interfaces (On LHR/FHR facing sources/receivers)
interface <intf>
 ip pim <sparse-mode|sparse-dense-mode> ! (sparse-mode if not using AutoRP)
```

**Designated Querier** always the lowest IP

### Source Specific Multicast (SSM)
- Requires IGMPv3

``` title="Join a multicast SSM group on IOS"
interface <intf>
 ip igmp join-group <multicast-ip> source <source-ip>
```

#### IGMPv3
- Completely new protocol, not backwards compatible

``` title="Enable IGMPv3"
ip igmp version 3
```