# MPLS
- What/Why: Multiprotocol
    - IP/Eth/ATM/FR
- How: Label Switching

## Data Plane (DP)
- Data plane remains mostly unchanged over time; defined in RFC3031, RFC3032
- No concept of src/dst address - replaced by MPLS label (routers generally only look at dst address anyway)
- `MPLS Label`: Instruction for packet treatment
    - Instructions are not carried in the packet, instead programmed in device (control plane)
- `Lookup`: Router process to locate the instructions for a labeled packet in the control plane

### MPLS Label
- Defined in RFC3032
- Stored in TCAM
- MPLS header comes after L2 header (always encapsulated within L2), can encapsulate L3 (IP) or L2 (Eth)
    - Sometimes called "L2.5", "L2+" maybe more accurate

#### Fields
* Label: Label Value, 20 bits
    * Lowest: 0, Highest: 2^20 (1,048,575)
* Exp: Experimental Use, 3 bits [used for QoS, CoS bits; RFC5462: Traffic Class]
* S: Bottom of Stack, 1 bit
    * Indicates that the following header is either MPLS (0) or non-MPLS (1) (when multiple MPLS headers are 'stacked')
* TTL: Time to Live, 8 bits

#### Label Opacity
When a router recieves an MPLS packet (has MPLS header), it uses the first header to determine the forwarding instructions. Any further MPLS headers are ignored (hidden)

#### Instructions
- Stored in MPLS LBL FIB
    - MPLS LBL FIB conains `MPLS Label Value` -> `Exit Ifc, Next Hop Addr, MPLS Actions/Operation`

##### MPLS Actions
`Add the label (Push)`
: Adds a label to the top of the stack of labels, increasing stack depth

`Replace the label (Swap)`
: Swaps the label at the top of the stack to a new label, stack depth remains the same

`Remove the label (Pop)`
: Removes the label at the top of the stack, stack depth decreases

#### Reserved Labels
- `0`: IPv4 Explicit Null (aka, "pop the label" Pop + perform an IPv4 lookup)
- `1`: Router Alert
- `2`: IPv6 Explicit Null
- `3`: Implicit Null (aka, "pop the label")
- `4-6`: Unassigned
- `7`: Entropy Label Indicator (ELI)
- `8-12`: Unassigned
- `13`: Generic Associated Channel
- `14`: OAM Alert
- `15`: Extension Label (XL)

### Packet Forwarding
`Limited IP Core`
: MPLS network does not know IP routes for packets it is forwarding

#### Tables
**IP FIB (w/o MPLS)**
    - Contents: IP Subnets
    - Key: IP Addresses
    - Results: Exit Ifc, Next Hop Info (Adj table)

**IP FIB (w/ MPLS)**
    - Contents: (same)
    - Key: (same)
    - Results: Exit Ifc, Next Hop Info (Adj table), **Optional: MPLS Action**

**MPLS Label FIB**
    - Contents: MPLS Labels
    - Key: MPLS Label
    - Results: Exit Ifc, Next Hop Info (Adj table), MPLS Actions

## Control Plane (CP)
- `TDP`: Prestandard version of LDP (Cisco specific)
- `LDP`: Dynamic protocol to distribute MPLS paths; this tech is on it's way out
- `RSVP`: 
- `BGP-LU`:
- `SR`: Segment routing; latest and greatest MPLS control plane

### Static Label Switched Paths (LSP)
- Take a destination IP (FIB entry) and bind an MPLS label

Create static MPLS action

"Binding": Bind an IP CEF entry to an MPLS action
```
mpls static binding ipv4 <dest-prefix> <dest-mask> <input|output> <dest-nexthop> <label-id>
```

Create a static MPLS label
```
mpls static labelswitch <incoming-label-id>
  moi out-interface <output-intf> ipv4 <nexthop-ip> <outgoing-label-id>
```

### BGP-Free Core
- Only edge routers run BGP, core is MPLS packets destined between edge routers

### Label Distribution Protocol (LDP)
1. Autodiscovery
    - `HELLO` packets
        - Dest IP (Multicast): 224.0.0.2 (AllRouters)
        - Uses UDP (Src/Dst Port: 646)
        - Neighbor advertises unicast IP (XPort IP, usually same as LSR-ID - must be routable) for creation of neighbor->neighbor TCP session
    - LDP-ID (LSR-ID: LABELSPACE)
        - Highest Loopback = LSR-ID
        - Ethernet: LABELSPACE = 0 (per platform, not per interface)

2. Session (neighborship) creation
    - Created using XPort IP (Dst IP)
    - TCP session (Dst Port: 646, Src Port: ephermal) (client/server)
    - Two types of messages
        1. `Label Mapping MSG`
            - Advertises local bindings (IPv4 Addr + MPLS Label)
        2. `Address MSG`
            - ??

- Independent LSP Control (generate labels for everything) [Cisco]
- Ordered LSP Control (generate binding for local routes and any bindings that were recieved from other rtrs)
#### LDP Configuration
- LDP starts when `mpls ip` is enabled on an intf

See bindings: `show mpls ldp bindings`

See autodiscovery: `show mpls ldp discovery [detail]`

Set MPLS LDP transfer address `(config-if) mpls ldp discovery transport-address [ip|interface]`

## MPLS Configuration
See all MPLS intfs
```
show ip mpls interfaces
```

Show MPLS LFIB
```
show mpls forwarding-table
```

Enable MPLS on intf
```
interface <intf>
  mpls ip
```

## MPLS L3VPN
- SP provdes "router as a service"
    - Customer advertises routes to SP router, which advertises other customer routes

### Goals
- Separation between Customer (C) routes
- No C-routes in the core (P routers)
- C-routes separated at PE routers
    - Each customer space assigned unique identifier
        - No longer IPv4 routes, now L3VPN routes (and BGP AF)
- BGP to distribute routes between PE's

#### Separation of C-Routes
- Utilizes "route distinguisher" attached to each IPv4 route
    - 64-bits
        - NN = unique number
        - AS:NN
        - IPv4:NN
        - 4BAS:NN
    - Becomes part of the route (NLRI) within BGP, so it would be unique even if IPv4 is same
- New BGP address family
    - RD:IPv4 (64 bits + 32 bits = 96 bits)
        - *Example:* ASN:NN:IPV4ROUTE
            - 10:1:10.1.1.1
    - New address family called `VPN-IPv4` (or `VPN-IPv6`)
        - Cisco: `VPNv4`
        - Juniper: `INET-VPN`
- PE: Advertises C-Routes as VPNv4
    - VRF (on PE router) becomes associated with a VPNv4 RD
    - virtual router (on local router) + RD = VRF
    - VRF:RD is 1:1 relationship

#### Policy Administration
- Uses tags, not RD - because single tag is insufficient for complex policy
- Originating PE: attaches tags per export policy
- Receiving PE: inspects and imports tags per import policy
- BGP doesn't have tags, instead has attributes
    - Extended-Communities (64 bits)
        - Route-Target: used for tagging purposes
            - AS:NN
            - IPv4:NN
        - RD (part of VPNv4 route/NLRI) **is not same as** RT (BGP extended community)

### VPNv4 configuration
- iBGP between PE's, MPLS+LDP+OSPF across core

PE configuration
```
! Create VR for routes
vrf definiation VRF_BLUE
 rd 10:101 ! <asn:unique>
 route-target import 102:102 ! <>
 route-target export 101:101
 address-family ipv4 unicast
 exit
 
! Add customer facing interface to VR
interface <intf>
 vrf forwarding VRF_BLUE

! Move Cust->SP eBGP to VR
router bgp <asn>
 address-family ipv4 unicast vrf VRF_BLUE
  neighbor <cust-ip> remote-as <cust-as>

! Exchange VPNv4 routes
router bgp <asn>
 address-family vpnv4 unicast
  neighbor <peer-pe-ip> activate
```

! See routes inside of VRF `show bgp vpnv4 unicast <rd|vrf> <vrf-name>`

How does receiving PE know which VRF data plane traffic is placed in? Stacked MPLS labels - the VPNv4 path gets an "inner" label