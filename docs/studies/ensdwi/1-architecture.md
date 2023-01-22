# 1.0 Architecture
## 1.1 Describe Cisco SD-WAN architecture and components

| Device      | Plane			| Use  								 |
| ----------- | ----------- 	| ------------------------			 |
| vManage     | Management  	| Creation of policy, Edge config    |
| vSmart      | Control     	| Calc/deploy control/data policy	 |
| vBond		  | Orchestration 	| Initial fabric discovery/auth 	 |
| WAN Edge	  | Data			| Routing/forwarding of user traffic |

### 1.1.a Orchestration plane (vBond, NAT)
- Provides initial authentication for the fabric, discovery of components
- Multiple vBond can be dpeployed for HA
	- Use single DNS record to point to all vBond IPs, WAN Edge will try each sequentially

When a WAN Edge joins the overlay, it only knows vBond
- Discovered from PNP, ZTP, Bootstrap, or manually
- Once connectivity is established to vManage/vSmart, the vBond->Edge connection is torn down
- Communicates using DTLS (Datagram TLS)


#### NAT

- vBond rovides STUN server (RFC 5389) for NAT discovery
	- Informs WAN edges if they are behind NAT
- vBond must be publicly addressable (or 1:1 static NAT)
	- vManage/vSmart can be behind PAT, using STUN like WAN edge

| Type 						| Behavior								| Notes										|
| Static 1:1 "full cone"	| External can initiate, always open 	| Works well with SD-WAN 					|
| Dynamic PAT "symmetric"	| Many hosts, inside initiates		 	| Other side can't be symmetric				|
| Address restricted cone 	| Inside init, then any external		| Other side can't be symmetric				|
| Port restricted cone 		| Inside inits, then filtered external	| Other side can't be symmetric				|

### 1.1.b Management plane (vManage)
- 3+ vManage can be clustered, always an odd number
	- Cluster can manage 6K WAN Edges (2K per node)

#### Tenancy options
| Type 			| Description																	|
| -------------	| -----------------------------------------------------------------------------	|
| Dedicated		| Each tenant has dedicated components, data plane is segmented					|
| VPN tenancy	| Segments data plane only, shares SD-WAN components							|
| Enterprise	| Orchestration/management in multi-tenancy mode, control plane is per-tenant	|

#### vAnalytics
- Provides insight into the WAN
- Global application performance, capacity planning
- Ingests data from the network and uses ML to predict trends
- vManage = realtime, raw data view
- vAnalytics = historical performance, forward-looking

### 1.1.c Control plane (vSmart, OMP)
- vSmart can handle 5400 connections per server
- Should be geographically dispersed, each vSmart is autonomous (no sync)
- Up to 20 vSmarts supported in a deployment
- Implements control plane policies, centralized data policies, service chaining, VPN topologies
- Handles security and encryption (key management)
	- WAN edge will compute it's own keys per transport and distribute to vSmart, which will re-distribute them back to other WAN edges (per policy)
	- Handles IPsec SA rekeying
	- WAN Edge doesn't need to handle key negotiation/distribution
- Similar to an iBGP route reflector
	- Routing/topology info is recieved from clients, best-path calculated (based on policy), and advertised to other clients

#### Control Plane Tunnels
- Encrypted/Authenticated via DTLS or TLS
	- DTLS preferred (UDP 12346)
	- TLS uses TCP
	- out of order/lost packets can be handled
	- vSmart/vManage ports match up to cores
		- 0: 12346, 1: 12446, 2: 12546, 3: 12646, 4: 12746, 5: 12846, 6: 12946, 7: 13046
- DTLS/TLS connections are maintained between all devices in SD-WAN overlay
- Negotiated using SSL certificates (each component auths the other, creates one way tunnel)
	- Device validates recieved cert is signed by a trusted root and has valid serial number w/ matching org name
- Carry OMP, SNMP, Netconf data

#### OMP
- Overlay Management Protocol
- Runs inside of a DTLS/TLS control plane tunnel, exchanges routing, key management, config updates
- *Example:* when a policy is configured vManage distributes to vSmart (NETCONF), vSmart distributes to WAN edge (OMP)
- Provides services like:
	- Facilitates network communication on the fabic (data plane, service chaining, multi-VPN topology)
	- Advertises services
	- Security information (encryption keys)
	- Best path selection and routing policy advertisement
- Enabled by default
	- Can interact with OSPF, BGP, EIGRP
	- Peering between Edge<->vSmart **only**
- Graceful restart
	- Enabled by default
	- WAN edges cache forwarding information
		- Default timer 12 hours (min: 1s, max 7d)
		- Still requires valid data plane tunnels (IPsec keys)
			- Set IPSEC rekey timer to 2x graceful restart timer
- Admin distance of 250 (vEdge) and 251 (cEdge)

OMP route types
- OMP route (or vRoute)
- TLOC
- Service route: Network service such as firewall, IDS

##### Service Route
Enables service chaining (sending data traffic through a service before going to destination)

Contains these attributes:

- **VPN ID** VPN the service belongs to
- **Service ID** FW (svc-id 1), IDS (svc-id 2), IDP (svc-id 3), netsvc1-4 (svc-id 4-7)
- **Label** OMP routes with traffic that must flow through the service will have their label field replaced with this label
- **Originator ID** system IP
- **TLOC** TLOC where service is located
- **Path ID** Identifier for OMP path

#### OMP Path selection
Only OMP route with valid TLOCs (BFD session is up) are selected

1. Valid OMP route? (TLOC is valid? BFD session is up?)
2. Prefer locally originated route
3. Prefer lower AD
4. Prefer higher OMP pref
5. Prefer higher TLOC pref
6. Prefer origin (connected, static, eBGP, EIGRP, OSPF intra-area, OSPF inter-area, OSPF external, EIGRP external, iBGP, Unknown)
7. Lowest origin metric
8. Highest system IP
9. Highest TLOC private address

vSmart can advertise up to 16 equal cost routes (default is 4)
```
omp
 ecmp-limit 4
```

#### OMP Loop prevention
- **OSPF** When a route is redistributed from OMP->OSPF, down bit is set
	- WAN edge will drop advertisement with down bit set
- **BGP** site of origin (SoO) BGP extended community is set to OMP site ID
	- WAN edge will drop advertisements with SoO == site ID
	- All BGP peers must send BGP extended communities and have same site ID
- **EIGRP** external protocol field is set to *OMP-Agent*
	- WAN edge will recieve advertisement into EIGPR topology table, but set `SD-WAN-Down` bit and AD to 252 (makes OMP route preferred)

### 1.1.d Data plane (WAN Edge)
- Where the SD-WAN overlay resides, forwards network traffic
- Enforcement of data policies (QoS, AAR)
- Establishes connections to other components
	- data plane connections with other routers, only connects to other data plane devices
	- control plane connections across each transport to up to 3 vSmart (only needs one)
		- *max-control-connections* has ability to disable control plane on a specific transport
		- if control plane connection is lost, WAN edge will forward data plane traffic for 12 hours
	- management plane connection to vManage (only one)
- Firewall - only allows explicit inbound traffic (like SSH, NETCONF, NTP, OSPF, BGP, and STUN)

#### 1.1.d (i) TLOC
Transport Location Identifier

- Identifier that ties an OMP route to a physical location
- TLOC is only IP known and reachable from underlying network
- Routable to the underlay, endpoint of data plane tunnels (like GRE tunnel source/destination)
- Advertised for each transport when a WAN edge has multiple

Contains these attributes:

- **system IP of WAN edge**
- **Color** defines the transport, 22 predefined colors (also defines public/private)
- **encapsulation type** IPSec or GRE, must match both sides
- **TLOC private address** private IP of WAN edge
- **TLOC public address** public IP of WAN edge
- **Preference** Prefer higher - can prefer one TLOC when comparing OMP route
- **Site ID** originator of the TLOC
- **Tag** similar to route tag
- **Weight** locally significant, prefer higher

##### Color
- 22 predefined colors
	- Private colors: only used when no NAT between devices in the overlay
		- metro-ethernet, mpls, private1-6
	- Public colors: use when NAT exists between overlay devices
		- 3g, biz-internet, public-internet, lte, blue, bronze, custom1-3, gold, green, red, silver
	- *Default* used when no color is defined
- WAN edges can only have one interface of each color
- WAN edges attempt to build data plane tunnels to every other site using every color
- Use *restrict* to restrict a color to only form tunnels with same color
	- restrict is an OMP attribute inside the TLOC route
	- Can also use tunnel groups
		- Tunnels only form with matching tunnel groups

#### 1.1.d (ii) IPsec and GRE

#### 1.1.d (iii) vRoute (aka OMP route)
- Synonym for OMP route
- Network prefixes, must resolve their next hop to a TLOC route
- Can advertise connected, static routes as well as redistribution from OSPF, EIGRP, BGP

Contains these attributes:

- **TLOC** next hop of the OMP route
- **System IP Address** similar to RID, unique across WAN edges
- **Color** mark a specific WAN connection
- **Origin** Source of the route (BGP, EIGRP, Connected, Static) and original metric
- **Originator** Where the route was learned from (system IP of advertiser)
- **Preference** OMP preference (prefers higher)
- **Service** If a service is associated
- **Site ID** Similar to BGP ASN, unique site ID
- **Tag** Optional, transitive - similar to route tag
- **VPN** VPN/VRF

#### 1.1.d (iv) BFD
- Used inside of IPsec tunnels between WAN Edges.
- Sends *Hello* packets to measure link liveness, packet loss, jitter, delay
- Echo mode (neighbor only echoes message back to sender)
	- Full round-trip view of the transport
- Can adjust timers, but cannot turn off

##### Path MTU discovery
- Provided by BFD
- BFD header is padding with PMTU information
- Each tunnel is probed every minute, and MTU is calculated periodically per TLOC (IPSec session)
- Suggested to turn off PMTU-D on low bandwidth links to prevent large packets from consuming all bandwidth

### 1.2 Describe Cisco SD-WAN Edge platforms and capabilities

#### XE-SDWAN
ISR1K, ISR4K, ASR1K, ENCS, CSP

- Support for additional interface types (voice, serial)
- Support for advanced security use cases
	- DNS Security (Umbrella), AMP for Endpoints, App-Aware Firewall, IDS/IPS, URL filtering
- Runs IOS-XE SD-WAN

#### vEdge
vEdge 100, 1000, 2000, 5000

- Runs Viptela OS

#### Cloud
CSR1Kv, vEdge Cloud, ISRv

## 1.3 Describe Cisco SD-WAN Cloud OnRamp
### 1.3.a SaaS
### 1.3.b IaaS
### 1.3.c Colocation