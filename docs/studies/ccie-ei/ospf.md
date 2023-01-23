# OSPF

## Overview
### Areas
* Backbone area (0) is required any time you have more than a single area

### Router's Role/Type
`Internal router`
: All interfaces in same area

`Backbone router`
: Has an interface in area 0

`Area border router (ABR)`
: Has an interface in area 0 and another area. *Must also be a backbone router*
: Use `show ip ospf border` to determine ABR/ASBR

`Autonomous System Border Router (ASBR)`
: Performs redistribution to/from another routing domain
: Originates LSA Type-5 or LSA Type-7

### Virtual-links and GRE Tunnels **DA#1**
* Area 0 **cannot** be discontiguous
* Use virtual-link to allow backbone area transit over another area
* Is *not* data plane, just allows establishment of **control plane**
    * i.e. packets are not moving *over* the virtual link

Virtual link configuration:
```
router ospf <pid>
 area <transit-area> virtual-link <target-rid>
```

Show virtual links: `show ip ospf virtual-links`

#### GRE tunnel vs virtual-link
- Virtual link: control plane only, not found in routing table
    - "GRE tunnel for only OSPF traffic"
- GRE tunnel: control and data plane
    - By default, OSPF cost 1000

### OSPF and BGP RID. BGP synchornization **DA#2**
`OSPF RID`
: 32-bit dotted-decimal value, unique in entire routing domain. *Is not an IP address*
: Numerically highest IP address will be chosen if not defined

`BGP synchronization`
: Requires a route to be seen via IGP to be valid in BGP
: OSPF and BGP RID must also match at point of EGP/IGP redistribution

### Split-Horizon Rule
Areas must use the backbone area to communicate between one another

#### Capability Transit **DA#3**
* Enabled by default
* Overrides the split horizon rule, so **data plane** traffic may transit directly between non-zero areas
* If disabled, traffic would try to transit through area-0

To turn off:
```
router ospf <pid>
 no capability transit
```

## LSAs
Advertisements regarting the states of links

### LSA Types

| LSA       | Advertising Rtr       | Rt Tbl        | Database                             |
| -------   | --------------------- | ------------- | ------------------------------------ |
| 1         | All routers           | `O`           | `show ip ospf database router`       |
| 2         | DR                    | `O`           | `show ip ospf database network`      |
| 3         | ABR                   | `O IA`        | `show ip ospf database summary`      |
| 4         | ABR                   |               | `show ip ospf database asbr-summary` |
| 5         | ASBR                  | `O E2` `O E1` | `show ip ospf database external`     |
| 7         | ASBR                  | `O N2` `O N1` | `show ip ospf database nssa-external`|

`Type-1`
: **Router LSA** intra-area routers

`Type-2`
: **Network LSA** intra-area multi-access networks

`Type-3`
: **Summary LSA** inter-area routes

`Type-4`
: **Summary ASBR LSA** Reachability to the ASBR

    * "If you want to connect to the router in the LSID, go to the router in the Adv_R"
    * LSID = RID of external router
    * Adv_R = 
    * Metric = cost from my area to redistributed route

`Type-5`
: External routes (from another routing domain)

    * Contains an LSID (route I distribute)
    * Contains an Adv_R (route originating LSA)
    * Contains an FA (next hop)=0.0.0.0 *suppressed* - found by looking up Type-1 LSA for Adv_R

`Type-7`
: **NSSA LSA**
    
    * LSID=network
    * Adv_R=RID of advertiser
    * FA=IP of ASBR *not suppressed in NSSA*

##### Type-7 to Type-5 Conversion
N2 route converted to E2 route
Type-7 routes converted into Type-5 going to area 0
FA is not suppressed, no Type-4 is advertised

##### External route metrics
`E2/N2`
: Cost of network from advertising ASBR (cost=20)

`E1/N1`
: Cost of network from advertising router + cost of me to ASBR

### Bit types
`E-bit`
: I have capability of having external routes in my table. *Normal area E is set*, *Stub area E is cleared*

`V-bit`
: I termimnate a virtual link.

`B-bit`
: I am an ABR

### Traffic types
`intra-area`
: Traffic within same area

`inter-area`
: Traffic that enters one area from another area (same routing domain)

`external`
: Traffic that was redistributed into my routing domain

### OSPF DR/BDR
DR/BDR is elected on multi-access network.

* Prevents full-mesh of adjacencies
    * each router establishes adjaceny with only DR
    * DR floods updates to `DROthers`
* OSPF priority=1 by default, 0=don't participate
* Tie-breaker: highest OSPF RID

### RFC5185

#### DA#4
*Note:* intra-area routes (stay in same area) preferred over inter-area routers

Allows a link to be configured in multiple areas

* Must be a point to point link from OSPF's perspective

`ip ospf multi-area <area-id>`

Must configure link as point-to-point:
```
R2#show ip ospf multi-area 
OSPF_MA0 is down, line protocol is down 
  Primary Interface GigabitEthernet0/1, Area 1
  Interface ID 3
  MTU is 1500 bytes
  Interface DOWN as link is not P2P
```

Multi-area established w/ 1 neighbor:
```
R2#show ip ospf multi-area 
OSPF_MA0 is up, line protocol is up 
  Primary Interface GigabitEthernet0/1, Area 1
  Interface ID 3
  MTU is 1500 bytes
  Neighbor Count is 1
```

Two neighborships shown:
```
R2#show ip ospf neighbor 

Neighbor ID     Pri   State           Dead Time   Address         Interface
0.0.0.1           0   FULL/  -        00:00:30    12.1.1.1        GigabitEthernet0/1
0.0.0.1           0   FULL/  -        00:00:32    12.1.1.1        OSPF_MA0
```

Virtual interface is created:
```
R2#show ip ospf int br
Interface    PID   Area            IP Address/Mask    Cost  State Nbrs F/C
Gi0/1        1     0               12.1.1.2/24        1     P2P   1/1
MA0          1     1               Unnumbered Gi0/1   1     P2P   1/1
```

#### DA#5
Use RFC5185 multi-area to give the ASBR a link in another area

- Routes are not redistributed into a given area, just into the OSPF routing domain

ASBR cannot exist in a stub area:
`*Dec 17 17:26:15.631: %OSPF-4-ASBR_WITHOUT_VALID_AREA: Router is currently an ASBR while having only one area which is a stub area`

### OSPF Recursion Process (DA#6)
```
R1#show ip ospf database external 

            OSPF Router with ID (0.0.0.1) (Process ID 1)

                Type-5 AS External Link States

  LS age: 90
  Options: (No TOS-capability, DC, Upward)
  LS Type: AS External Link
  Link State ID: 4.4.4.0 (External Network Number )
  Advertising Router: 0.0.0.4
  LS Seq Number: 80000001
  Checksum: 0x3F51
  Length: 36
  Network Mask: /24
        Metric Type: 2 (Larger than any link state path)
        MTID: 0 
        Metric: 20 
        Forward Address: 0.0.0.0
        External Route Tag: 0
```
Suppressed *Fwd_Address*? Go to router with RID of *Adv_R*

Look for Adv_R:
```
R1#show ip ospf database router adv-router 0.0.0.4

            OSPF Router with ID (0.0.0.1) (Process ID 1)
```
Can't find this *Adv_R* RID in own area?

Go look for LSA Type-4:
```
R1#show ip ospf database asbr-summary 0.0.0.4

            OSPF Router with ID (0.0.0.1) (Process ID 1)

                Summary ASB Link States (Area 1)

  LS age: 212
  Options: (No TOS-capability, DC, Upward)
  LS Type: Summary Links(AS Boundary Router)
  Link State ID: 0.0.0.4 (AS Boundary Router address)
  Advertising Router: 0.0.0.2
  LS Seq Number: 80000001
  Checksum: 0x4C69
  Length: 28
  Network Mask: /0
        MTID: 0         Metric: 128 
```

Go look for *this* *Adv_R*
```
R1#show ip ospf database router adv-router 0.0.0.2

            OSPF Router with ID (0.0.0.1) (Process ID 1)

                Router Link States (Area 1)

  LS age: 294
  Options: (No TOS-capability, DC)
  LS Type: Router Links
  Link State ID: 0.0.0.2
  Advertising Router: 0.0.0.2
  LS Seq Number: 80000002
  Checksum: 0x725B
  Length: 36
  Area Border Router
  Number of Links: 1

    Link connected to: a Transit Network
     (Link ID) Designated Router address: 12.1.1.2
     (Link Data) Router Interface address: 12.1.1.2
      Number of MTID metrics: 0
       TOS 0 Metrics: 64
```

How do I get to 12.1.1.2?
```
R1#show ip route 12.1.1.2
Routing entry for 12.1.1.0/24
  Known via "connected", distance 0, metric 0 (connected, via interface)
  Routing Descriptor Blocks:
  * directly connected, via GigabitEthernet0/2
      Route metric is 0, traffic share count is 1
```

We are directly connected, so just go directly to 12.1.1.2.
```
R1#show ip route 4.4.4.0
Routing entry for 4.4.4.0/24
  Known via "ospf 1", distance 110, metric 20, type extern 2, forward metric 192
  Last update from 12.1.1.2 on GigabitEthernet0/2, 00:05:14 ago
  Routing Descriptor Blocks:
  * 12.1.1.2, from 0.0.0.4, 00:05:14 ago, via GigabitEthernet0/2
      Route metric is 20, traffic share count is 1
```

#### Same thing in an NSSA area
```
R4#show ip ospf database nssa-external 

            OSPF Router with ID (0.0.0.4) (Process ID 1)

                Type-7 AS External Link States (Area 2)

  LS age: 49
  Options: (No TOS-capability, Type 7/5 translation, DC, Upward)
  LS Type: AS External Link
  Link State ID: 4.4.4.0 (External Network Number )
  Advertising Router: 0.0.0.4
  LS Seq Number: 80000001
  Checksum: 0x73EA
  Length: 36
  Network Mask: /24
        Metric Type: 2 (Larger than any link state path)
        MTID: 0 
        Metric: 20 
        Forward Address: 34.1.1.4
        External Route Tag: 0
```

*Type 7/5 translation* means P-bit is set (propagate bit)

When LSA is converted from type7/5 on `0.0.0.3`, it becomes the Adv_R, Fwd_Addr is propagated:
```
R1#show ip ospf database external 

            OSPF Router with ID (0.0.0.1) (Process ID 1)

                Type-5 AS External Link States

  LS age: 127
  Options: (No TOS-capability, DC, Upward)
  LS Type: AS External Link
  Link State ID: 4.4.4.0 (External Network Number )
  Advertising Router: 0.0.0.3
  LS Seq Number: 80000001
  Checksum: 0xE5B
  Length: 36
  Network Mask: /24
        Metric Type: 2 (Larger than any link state path)
        MTID: 0 
        Metric: 20 
        Forward Address: 34.1.1.4
        External Route Tag: 0
```

**Example** The 34.1.1.4 network (Fwd_Addr) is suppressed upstream. 4.4.4.0 network is removed from the routing table since ASBR is not accessible to R1.

Solution: suppress the FA when type7 is converted to type4. Now normal recursion will be used on R1 to find Type-4 LSA to provide reachability to the ASBR (R3).
R3#show run | sec router ospf
router ospf 1
 router-id 0.0.0.3
 area 2 nssa translate type7 suppress-fa
 area 2 range 34.1.1.0 255.255.255.0 not-advertise

### Multiple ABR in NSSA (DA#7)

In NSSA with multiple ABR's, there is an election and the highest RID becomes the advertising router (and performs type7 to type5)

Override higher RID with:
`R3(config-router)#area 1 nssa translate type7 always`

Do this on multiple ABR's and both will be the advertising router.

### Network type impact on redistribution (DA#8)

Show all ABR/ASBR and cost to routers
```
R5#show ip ospf border-routers 

            OSPF Router with ID (0.0.0.5) (Process ID 1)


                Base Topology (MTID 0)

Internal Router Routing Table
Codes: i - Intra-area route, I - Inter-area route

i 0.0.0.3 [1] via 35.1.1.3, GigabitEthernet0/3, ASBR, Area 0, SPF 3
i 0.0.0.4 [1] via 45.1.1.4, GigabitEthernet0/4, ASBR, Area 0, SPF 3
```

When redistributing, a P2P network type will result in a supressed FA. Broadcast network type will set the FA to the specific address (since multiple routers could exist on the segment)

### (DA#9)
Fwd_Addr must be an IA route in OSPF, otherwise route will be ignored.

### RFC2328 (DA#10)
RFC2328 "When the network numbers (in an LSID) are identical, there must be a way to differentiate one network from another."

* Least specific route: gets the normal LSID
* More specific routes: uses broadcast address of that network

```
R2#show ip ospf d external 

            OSPF Router with ID (0.0.0.2) (Process ID 1)

                Type-5 AS External Link States

  LS age: 232
  Options: (No TOS-capability, DC, Upward)
  LS Type: AS External Link
  Link State ID: 2.0.0.0 (External Network Number )
  Advertising Router: 0.0.0.2
  LS Seq Number: 80000001
  Checksum: 0xC1DA
  Length: 36
  Network Mask: /8
        Metric Type: 2 (Larger than any link state path)
        MTID: 0 
        Metric: 20 
        Forward Address: 0.0.0.0
        External Route Tag: 0

  LS age: 232
  Options: (No TOS-capability, DC, Upward)
  LS Type: AS External Link
  Link State ID: 2.0.255.255 (External Network Number )
  Advertising Router: 0.0.0.2
  LS Seq Number: 80000001
  Checksum: 0xC1DA
  Length: 36
  Network Mask: /16
        Metric Type: 2 (Larger than any link state path)
        MTID: 0 
        Metric: 20 
        Forward Address: 0.0.0.0
        External Route Tag: 0
```

### RFC1583 & RFC2328 (DA#11)
**By default RFC1583 is used**

`RFC1583`
: "The cost of a summary should be based on the lowest cost of a specific route within that summary."

`RFC2328`
: "The cost of a summary should be based on the **highest** cost of a specific route within that summary."

To implement RFC2328:
```
router ospf <pid>
    no compatible rfc 1583
```

### Route precedence (DA#12)

* Default: N is preferred over E (RFC3101)
    * RFC1587 E is preferred over N
* OSPF lower PID is preferred when running two PIDs

#### Preference of routes:
1. Intra area `O` routes
    - More than one entry? Use lower OSPF PID
2. Inter area `O IA` routes
3. External NSSA Type 1 `O N1`
    - E1 and N1 route competing? If Fwd_Addr is different, cost will be referenced before RFC3101
4. External Type 1 `O E1`
5. External NSSA Type 2 `O N2`
6. External Type 2 `O E2`