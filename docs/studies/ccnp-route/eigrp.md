---
search:
  exclude: true
---

# EIGRP

## Summary
EIGRP is an **advanced distance vector** protocol, designed by Cisco. Combines the best of link-state and distance vector.

- Supports VLSM (classless protocol, advertises a subnet mask for each network)
- No special configurations for different layer 2 protocols
- **32-bit metric** provides granularity and support for unequal cost load balancing

## Features
- rapid convergence - uses DUAL (diffusing update algorithm). Stores its neighbors' routing tables to it can quickly adapt
- higher scalability - sends partial triggered updates rather than periodic updates. Sends changes only, not whole routing table
- supports multiple routed protocols - IPv4 and IPv6 support

## Communications
- Uses RTP (Reliable Transport Protocol) for guaranteed, ordered delivery of EIGRP packets
    - 'Ack' can be required for important packets like HELLO
- EIGRP = IP protocol 88
- Supports unicast and multicast
    - Multicast IPv4: 224.0.0.10
    - Multicast IPv6: FF02::A

## Neighbor table
- Contains each neighbor that has an established adjacency
- Primary IP of the neighbor and directly connected interface

## Neighborship process
- Steps to become neighbors
	1. R1 and R2 send *HELLO* to all of their configured interfaces
    2. R1/R2 recieve each other's *HELLO* and send an *UPDATE* to each other which includes all routes EXCEPT for the ones learned on the same interface the *HELLO* came from (split-horizon). These *UPDATE* packets have the initialization bit set to indicate this is the initial exchange.
    3. Once the *UPDATE* has been recieved, each router sends an *ACK* to the other to indicate the *UPDATE* was recieved.
    4. The data from the *UPDATE* packets is placed in the topology table.
    5. *successor* routes from the topology table are placed into the routing table
- Routers must belong to the same AS - identifies the routing process and EIGRP domain

Configure interfaces within 192.168.1.0/24 to participate in EIGRP
```
R1(config)# router eigrp 250
R1(config-router)# network 172.16.12.0 [0.0.0.255] <-- optional wildcard mask
```
* If a wildcard isn't specified, classful subnet will be used

Set EIGRP RID `R1(config-router)# eigrp router-id 1.2.3.4`
* EIGRP RID (Router ID) is a 32-bit value configured like an IPv4 address
* Doesn't change unless the EIGRP process is cleared or router ID is manually configured
* Automatic selection process
    1. Highest address of it's loopbacks
    2. If no loopbacks, highest IPv4 address on an active interface

Verify neighbors with `show ip eigrp neighbors [detail]`
```
R1#show ip eigrp ne
EIGRP-IPv4 Neighbors for AS(250)
H   Address                 Interface              Hold Uptime   SRTT   RTO  Q  Seq
                                                   (sec)         (ms)       Cnt Num
0   172.16.12.2             Gi0/1                    13 00:01:02    8   150  0  2
```
* H - order the sessions were formed in
* Address - IP of the peer
* Interface - interface the peer is on
* Hold - amt of time the router will wait to hear a *HELLO*
* Uptime - amt of time since the neighborship was formed
* SRTT - time in milliseconds required for the router to send an EIGRP packet to it's neighbor and recieve an ACK
* RTO - amt of time before sending a packet from the retransmission queue
* Q - shows the number of packets waiting in the queue (greater than 0 = network congestion)
```
R1#show ip ei n d
EIGRP-IPv4 Neighbors for AS(250)
H   Address                 Interface              Hold Uptime   SRTT   RTO  Q  Seq
                                                   (sec)         (ms)       Cnt Num
0   172.16.12.2             Gi0/1                    14 00:01:45    8   150  0  2
   Version 23.0/2.0, Retrans: 1, Retries: 0, Prefixes: 1
   Topology-ids from peer - 0 
   Topologies advertised to peer:   base

Max Nbrs: 0, Current Nbrs: 0
```
* Retrans - # of times a pakcet has been retransmitted
* Retries - # of times an attempt was made to retransmit
* Prefixes - # of prefixes recieved

## Topology table
- Contains all destination routes advertised by neighbors
- each entry is assocated with a list of neighbors advertising that entry
- Each neighbor+destination combination has an *advertised metric* - the metric that the neighbor stores in their routing table to the desination
- advertised metric* + *link cost* = *metric* (i.e. this routers metric to the destination)
- best *metric* = *successor* and is placed into the routing table and advertised to other neighbors
