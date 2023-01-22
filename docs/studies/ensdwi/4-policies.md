# 4.0 Policies
## 4.1 Configure control policies

## 4.2 Configure data policies

## 4.3 Configure end-to-end segmentation

### 4.3.a VPN segmentation
- VPNs are segmented (same as VRF), each data packet carries a VPN ID

3 Types of VPN:

- **Service VPN**: User traffic, multiple can exist with different topologies
	- Overlay
	- Values of 1-511
- **Transport VPN**: Physical underlay, WAN connectivity "VPN 0"
- **Management VPN**: OOB management, "VPN 512"

### 4.3.b Topologies
- TLOC routes infuence how the data plane is built, filtering routes and TLOCs accomplishes topologies

## 4.4 Configure Cisco SD-WAN application-aware routing

## 4.5 Configure direct Internet access