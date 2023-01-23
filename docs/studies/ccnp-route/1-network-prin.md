---
search:
  exclude: true
---

# CCNP Route 1.0 Network Principles

## 1.1 Identify Cisco Express Forwarding Concepts
- IOS platform has 3 methods to forward packets (process switching, fast switching, CEF)
- Layer 3 device has distributed architecture - independant control plane and data plane
    - Control plane (route processor)
        - runs routing protocol
        - conveys control info to interface module for control of data plane
        - collects data plane info (traffic stats)
        - handles certain data packets sent to the RP
    - Data plane (interface microcoded processor)
        - forwards data packets

- Process switching (slowest)
    - Each packet is individually examined by the CPU (control plane)
    - All forwarding decisions made in software
    - CPU intensive/last resort due to degradation of performance
- Fast switching (faster)
    - Initial packet in a flow is process switched
    - Forwarding decision is stored in the data plane's hardware fast-switching cache
    - Subsequent frames are forwarded based on the fast-switching cache
    - Fast-switching cache contains Address, Prefix, L2 mac of next hop, outgoing interface
- Cisco Express Forwarding (fastest)
    - Least CPU intensive
    - Control plane creates two hardware-based tables
        - Created using Layer 2/3 tables (incl. routing and ARP tables)
        - Contain all info needed to forward packet
        - Hardware based decision for all frames in flow
        - CEF caches info from routing engine before flows are encountered
        - Control plane is responsible for building FIB/adj tables
        - **1.1.a** Forwarding Information Base (FIB) `show ip cef` `show ipv6 cef`
            - Contains precomposed reverse lookups and next-hop information
            - Contains Address, Prefix, 'Adjacency pointer' (to Adj table)
            - Dest prefixes are stored from most specific to least specific (maxiumum efficiency)
        - **1.1.b** Adjacency table `show adjacency`
            - Holds Layer 2 next-hop addresses and frame header rewrite info for all FIB entries
            - Contains 'Adjacency pointer', L2 mac of next hop (from ARP table)
            - Populated as adjacency's are discovered (via ARP)
    - Some features are not compatible with CEF
        - IP header options
        - expiring TTL counters
        - forwarded to tunnel interface
        - unsupported encapsulation (in or out)
        - Exceed MTU and require fragmentation
    - Pakcets that cannot be CEF switched are 'punted' to be fast/process switched
    - Enabled by default for IPv4, enabled with `ipv6 unicast-routing` for IPv6

```
! validate that CEF is running
show interface Fa0/0
 ..
 IP CEF switching is enabled
 ..
```
`(config-if)# no ip route-cache cef` disable CEF per interface
`(config)# no ip cef` disable CEF globally

## 1.2 Explain general network challenges
**a. Unicast**
    - Traffic exchanged between one sender and one reciever
**b. Out-of-order packets**
**c. Asymmetric routing**
    - Traffic flows that use a different path for the return path than the original
    - Occurs in networks w/ redundant paths
    - Often desirable for effective bandwidth use

<hr>

## 1.3 Describe IP operations
**a. ICMP Unreachable and Redirects**
- IMCP Redirect: used by routers to notify sender that there is a better route available
    - If a router recieves a pkt from segment, and sees the destination on the same segment, it will notify the sender with an ICMP Redirect to packets can be forwarded directly

**b. IPv4 and IPv6 fragmentation**
- IPv4 packet larger than egress MTU - must fragment unless DF bit is set
- Causes CPU/memory overhead during fragmentation/reassembly
- If one fragment is dropped -> retransmit whole packet
- L4/L7 filtering has trouble processing
- - IPv6 fragmentation
    - IPv6 routers only fragment a packet if they are the source
    - Packets larger than the MTU of outgoing interface are dropped
        - Sends ICMPv6 'Packet Too Big' message back to the source, including the smaller MTU
    
**c. TTL**

<hr>

## 1.4 Explain TCP operations
**a. IPv4 and IPv6 (P)MTU**
- IPv4 packet max size: 65,535 bytes
- IPv6 packet max size: 4,294,967,295 bytes (w/ hop-by-hop extension and jumbo payload)
- Most transmission links enforce smaller sizes
    
**b. MSS**
- defines largest amount of data a recieving device can accept in a TCP segment
- manually set -> not negotiated
- MSS should = MTU - 40 bytes (20 for IPv4 header, 20 for TCP header)
- Helps avoid fragmentation but does not prevent it if there is a smaller MTU link in the path

- PMTUD (Path MTU Discovery)
    - determines the lowest MTU along a path
    - supported by TCP
    - Requires ICMP Destination Unreachable messages
        - Ensure that 'unreachable' and 'time exceeded' messages are permitted by packet filtering
    - PMTUD works for IPv6 similar to IPv4

**c. Latency**
- amount of time for a message to go from A to B
- Network Latency: amount of time for the packet to travel the network from src to dst
- Causes of latency
    - Propagation Delay
    - Serialization
    - Data Protocols
    - Routing
    - Switching
    - Queueing
    - Buffering
- TCP flow control & reliability have an affect on end-to-end latency

**d. Windowing**
- "the amount of data that can be sent before expecting an acknowledgement"
- Usually several times the MSS

**e. Bandwidth-delay product (BDP)**
- TCP can experience bottlenecks on paths with high bandwidth and long RTT (i.e. long fat pipe)
- BDP = bandwidth (bps) x RTT (seconds)
- BDP = "number of bits it takes to fill the pipe" i.e. "amount of unacknowledged TCP data to keep the pipeline full"
- BDP is used to optimize the TCP window size

**f. Global synchronization**

<hr>

## 1.5 Describe UDP operations
**a. Starvation**
- aka "TCP starvation" or "UDP dominance"
- TCP - includes reliability, flow control, congestion avoidance
- UDP - includes none of the above
- When TCP+UDP flows experience congestion, TCP backs off using 'slow start' - UDP then eats up the bandwidth that TCP gave up

**b. Latency**
- UDP has very low latency compared to TCP
- UDP - used for streaming media that requires minimal delay and can tolerate occasional packet loss

<hr>

## 1.6 Recognize proposed changes to the network
**a. Changes to routing protocol parameters**
**b. Migrate parts of the network to IPv6**
**c. Routing protocol migration**

---
## References
https://www.cisco.com/c/dam/en_us/training-events/exams/docs/300-101_route.pdf