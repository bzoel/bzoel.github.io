
## RD, CD, S, FC, FS, FD (DA#1)
- **Reported Distance** (aka advertised distance) A router's best path, which it will advertise to other neighbors
- **Compouted Distance** *My* distance through neighbor(s) to a network
- **Successor** The lowest CD path **routing table**
- **Feasibility Condition** A routers reported distance must be *lower* than my best path to be considered
- **Feasible successor** Any *other* paths that meet the feasibility condition, **topology table**
- **Feasible distance** Initially equal to distance of successor - lowest value in history of link, used for loop avoidance. Can cause DUAL algorithm to run again, vs trying to converge on a looped path


EIGPR classic metric calulation
Delay + Bandwidth * 256

via <neighbor> (CD / RD), <interface>

Show S and FS
```
R1#show ip eigrp topology | beg 10.1.1.0
P 10.1.1.0/24, 1 successors, FD is 2560
        via 12.1.1.2 (2560/256), GigabitEthernet0/2
        via 13.1.1.3 (5120/1280), GigabitEthernet0/3
```

Show all routes, even those that do not meet the FC
```
R1#show ip eigrp topology all-links | beg 10.1.1.0
P 10.1.1.0/24, 1 successors, FD is 2560, serno 4
        via 12.1.1.2 (2560/256), GigabitEthernet0/2
        via 14.1.1.4 (4096/3072), GigabitEthernet0/4 ! Not shown becasue RD > CD of best path
        via 13.1.1.3 (5120/1280), GigabitEthernet0/3
```

## Finding ASN (DA#2)
Neighbor Adj doesn't come up? Either ASN mismatch, or neighbor is unicasting and I am multicasting
1. Try a `neighbor` command to unicast
2. Try to find ASN using debug command

```
access-list 100 per eigrp host <neighbor-ip> host 224.0.0.10
debug ip packet detail 100 dump

*Nov 19 18:04:06.744: IP: s=56.1.1.5 (GigabitEthernet0/5), d=224.0.0.10, len 60, rcvd 0, proto=88
08E1D1C0:                       0100 5E00000A            ..^...
08E1D1D0: 50000001 00060800 45C0003C 00720000  P.......E@.<.r..
08E1D1E0: 01589F28 38010105 E000000A 0205E66E  .X.(8...`.....fn
08E1D1F0: 00000000 00000000 00000000 00000064  ...............d
08E1D200: 0001000C 01000000 0000000F 00040008  ................
08E1D210: 14000200

undebug all
```

### Find ASN
1. Look for *E000000A*, this is 224.0.0.10 in hex
2. Go 5 blocks after that block *00000064* this is ASN in hex
3. 64 hex to decimal = 100
4. ASN is 100

### Find K values
1. Look at 7th block *01000000*, and first two digits in 8th block *00*
2. K1 = 01, K2 = 00, K3 = 00, K4 = 00, K5 = 00

`metric weight 0 1 0 0 0 0` (ToS, K1, K2, K3, K4, K5)

## Summarization (DA#3)
Unsummarized routes with different delays
```
R2#show ip route eigrp | beg Ga
Gateway of last resort is not set

      1.0.0.0/24 is subnetted, 4 subnets
D        1.1.0.0 [90/3072] via 12.1.1.1, 00:01:38, GigabitEthernet0/1
D        1.1.1.0 [90/28416] via 12.1.1.1, 00:01:38, GigabitEthernet0/1
D        1.1.2.0 [90/54016] via 12.1.1.1, 00:01:38, GigabitEthernet0/1
D        1.1.3.0 [90/79616] via 12.1.1.1, 00:01:38, GigabitEthernet0/1
```

After summarization
```
R2#show ip route eigrp | beg Ga
Gateway of last resort is not set

      1.0.0.0/22 is subnetted, 1 subnets
D        1.1.0.0 [90/3072] via 12.1.1.1, 00:00:07, GigabitEthernet0/1
```

*Note* cost of a summary is based on the lowest cost of a specific route within the summary.

### Leaking routes
- Just use a more specific `ip summary-address`
- Use a leak map 
```
ip summary-address eigrp 100 <subnet> <mask> leak-map <route-map-name>

route-map tst permit 10
 match ip address 1

access-list 1 permit <subnet> <inverse-mask>
```
1. Leak map refs non-existant route-map? Only summary advertised
2. Leak map refs route-map that doesn't have an ACL? Summary + specific routes advertised
3. Leak map refs route-map that refs ACL that doesn't exist? Summary + specific routes advertised

## Default Routes
Options for injecting default route in EIGRP?

1. (best?) `ip summary-address eigrp <asn> 0.0.0.0 0.0.0.0`
    - Allows 'per neighbor' default route injection
    - Will surpress all specific routes
2. `ip route 0.0.0.0 0.0.0.0 Null0`, then `network 0.0.0.0` in EIGRP
    - Works because Null0 becomes a directly connected interface in routing table
    - Will not suppress other specific routes
3. `ip route 0.0.0.0 0.0.0.0 <next-hop> `, then `redistribute static` in EIGRP
    - Will not suppress other specific routes
    - Default route will have AD of 170, external route
4. `ip default-network <network>`, then `redistribute static` in EIGRP
    - If subnetted IP is used, static route for network will be injected
    - Cisco says being removed soon (classfull command)

## Authentication
1. Classic Mode - supports MD5
    - Keychain based authentication

```
key chain <keychain-name>
 key 1
  key-string <password>

interface <intf>
 ip authentication key-chain eigrp <asn> tst
 ip authentication mode eigrp <asn> md5
```

See password in double quotes
`show key-chain <keychain-name>`

2. Named Mode - supports MD5 or SHA-256

MD5
```
key chain <keychain-name>
 key 1
  key-string <password>

router eigrp <name>
 address-family ipv4 as <asn>
  af-interface <intf | default>
   authentication mode md5
   authentication key-chain <keychain-name>
```

SHA-256 *does not use key chain*
```
router eigrp <name>
 address-family ipv4 as <asn>
  af-interface <intf | default>
   authentication mode hmac-sha-256 <password>
```


## Metric Calculation (DA#4)

- Before composite metric is calculated, vector metric is calulated
    - Vector metric "worst case scenerio"
        - Advertised to neighbors
        - Bandwidth = worst bandwidth along the path
        - Delay = sum of delay along the path

### K values
Default, K1 and K3 = 1, others = 0
- K1 = bandwidth
- K2 = load
- K3 = delay
- K4 = reliability
- K5 = MTU

### Classic Mode metric calculation
[(K1 x BW) + (K2 x BW 256-load) + (K3 x DLY)] x [K5/K4 + reliability] x 256

Bandwidth = Slowest BW / 10,000,000
Delay = Sum of Delays / 10

So by default:
**(BW + DLY) x 256**

### Named Mode metric calculation
- When is wide metric in use?

EIGRP version 9 or above

```
show eigrp tech-support
eigrp-release      :  20.00.00 : Portable EIGRP Release  
```

 Metric version 64bit = wide metric

```
R8#show eigrp protocols | inc rib-scale|version
Metric rib-scale 128
Metric version 64bit
```

- Composite metric - 64 bits long
- Rib-scale - routing table can only hold 32 bit number, so metric gets divided by rib-scale 
    - **Need to change on all routers on same AS** If you don't, it won't cause loop, but will cause suboptimal routing


FD is 13189120, RIB is 103040
13189120 / rib-scale (128) = 103040

```
R7#show ip ei topology 8.8.8.0/24
...
State is Passive, Query origin flag is 1, 1 Successor(s), FD is 13189120, RIB is 103040
Descriptor Blocks:
78.1.1.8 (GigabitEthernet0/8), from 78.1.1.8, Send flag is 0x0
    Composite metric is (13189120/163840), route is Internal
    Vector metric:
        Minimum bandwidth is 100000 Kbit
        Total delay is 101250000 picoseconds
        ...
```

    Nonchangable values from EIGRP spec
    - EIGRP_BW = 10,000,000 (10 million)
    - EIGRP_DLY = 1,000,000 (1 million)
    - Multiplier = 65536

    Metric calculation
    - wide metric = throughput + latency
        - throughput = (EIGRP_BW x Multiplier) / Minimum bandwidth
        - latency = (Total delay x Multiplier) / EIGRP_DLY

## Advanced Tagging (DA#5)
Using dotted decimal notation with tagging

`(config)#route-tag notation dotted-decimal`

Filter for odd tagged routes using DDN route tags
```
route-tag list RT seq 5 permit 0.0.0.1 0.0.0.254

route-map tst permit 10
 match tag list RT

router eigrp <name>
 !
 address-family ipv4 unicast autonomous-system <asn>
  !
  topology base
   distribute-list route-map tst in
```

### Filter for an odd number on 4th octet in an ACL
This demonstrates same thing using an ACL that you can do with route tag.

    `access-list 1 permit 1.1.1.1 0.0.0.254`
    - Matches 1.1.1.1/32, 1.1.1.3/32, etc.
    - Works due to "AND-ing" process (binary)

1.1.1.1/32 match
```
Last octet .1 = 0000 0001
Wildcard .254 = 0000 0001
AND             _________
                0000 0001
```

1.1.1.1/32 no match
```
Last octet .2 = 0000 0010
Wildcard .254 = 0000 0001
AND             _________
                0000 0000
```

1.1.1.5/32 match
```
Last octet .5 = 0000 0101
Wildcard .254 = 0000 0001
AND             _________
                0000 0001
```

## Stub Routing
- When a path is lost, EIGRP takes active role to find new path.
- EIGRP sends *QUERY* to ask neighbors.
    - *QUERY* gets propogated downstream until someone answers
    - Neighbors must respond within 3 mins
        - No response? Path is SIA (stuck in active) until reply recieved

### Stub router
- Sets stub flag, *QUERY* is never sent to stub router

Only advertise a specific type of routes
```
router eigrp <asn>
 eigrp stub (connected | summary | static | redistributed | receive-only)
```

Leak some routes through a stub?
```
router eigrp <asn>
 eigrp stub connected summary leak-map <map>
```

Leak specific routes to specific neighbors?
```
route-map <tst> permit 10
 match ip address 2
 match interface Gi0/2
route-map <tst> permit 20
 match ip address 3
 match interface Gi0/3
route-map <tst> permit 90
```

## ADD_PATH (DA#6)
Can advertise multiple paths to the same network, so that there is no wait for convergence.

```
router eigrp <name>
 address-family ipv4 as <asn>
  af-interface <intf>
   add-paths 2
```

## FRR LFA (DA#7)
"BFD on Demand"

- Requires named mode
- Creates a BFD session

Configuration:
```
router eigrp <name>
 address-family ipv4 as <asn>
 topology base
  fast-reroute ...
```

FRR options:
```
R9(config-router-af-topology)#fast-reroute ?
  load-sharing  Distributes repair paths equally among links and prefixes
  per-prefix    Enable Fast-Reroute Per-Prefix
  tie-break     Set repair path preference
```

When using *per-prefix*:
```
R9#show ip route 11.1.1.1 | inc Repair
      Repair Path: 10.9.12.12, via GigabitEthernet12

R9#show ip cef 11.1.1.1
11.1.1.1/32
  nexthop 10.9.10.10 GigabitEthernet10
    repair: attached-nexthop 10.9.12.12 GigabitEthernet12
```

- **per-prefix** calculate repair path on a per prefix basis
- **load-sharing** treat the paths as equal
- **tie-break** example: interface-disjoint, chose a repair path on a different interface for redundancy purposes

```
R9(config-router-af-topology)#fast-reroute tie-break ?
  interface-disjoint         Prefer Interface disjoint repair path
  linecard-disjoint          Prefer line card disjoint repair path
  lowest-backup-path-metric  Prefer repair path with lowest total metric
  srlg-disjoint              Prefer SRLG disjoint repair path
```

**Order of priority to select repair path** SRLG (prio 10), Interface (prio 20), Lowest metric (prio 30), line card (prio 40)

**SRLG** - Shared risk link group
```
interface <intf>
 srlg gid <gid>
```

## EIGRP Filtering (DA #10)

**Filter using maximum-hops**
```
router eigrp <asn>
 metric maximum-hops <max-hops>
```

**Filter by setting AD**
```
! Identify network
access-list 1 permit 2.0.0.0 0.255.255.255

! Set admin distance to 255 when advertised by router 100.1.1.3
router eigrp 100
 distance 255 <neighbor-ip> <neigh-wildcard> <acl>
 ! distance 255 100.1.1.3 0.0.0.0 1
```

**Using distribute-list**
```
! Identify network
access-list 1 deny 4.0.0.0 0.255.255.255
access-list 1 permit any

router eigrp 100
 distribute-list 1 in Gi0/2
 ! distribute-list <acl> <direction> <intf>
```

**Using extended ACL**
```
! 100.1.1.2 = Advertising router
! 3.0.0.0/8 = Network to be filtered
access-list 100 deny ip host 100.1.1.2 3.0.0.0 0.255.255.255
access-list 100 per ip any any

router eigrp 100
 distribute-list 100 in Gi0/0
```

**Filter by metric range**
Example - filter metrics between 7936-10496

*Calculate a deviation*
7936 + 10496 = 18432 / 2 = 9216 (middle of range)
10496 - 7936 = 2560 / 2 = 1280 (deviation from middle of range)

Route map says "If route metric is within range of 9216 +- 1280, deny"
```
route-map tst deny 10
 match metric 9216 +- 1280
route-map tst permit 20

router eigrp 100
 distribute-list route-map tst in 
```

**Filter by internal/external route**
```
router eigrp 100
 distance eigrp 90 255
 ! distance eigrp <internal-routes> <external-routes>
