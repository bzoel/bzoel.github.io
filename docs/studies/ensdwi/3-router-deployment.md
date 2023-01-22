# 3.0 Router Deployment
## 3.1 Describe WAN Edge deployment

### 3.1.a On-boarding

#### Manual bootstrap
- Minimal configuration including IP addressing, vBond address, system identification
```
vEdge# config
vEdge(config)# system host-name *hostname*
vEdge(config-system)# system-ip *ip-address*
vEdge(config-system)# site-id *site-id*
vEdge(config-system)# organization-name *organization-name*
vEdge(config-system)# vbond *(dns-name | ip-address)*
vEdge(config)# vpn 0
vEdge(config-vpn-0)# interface *interface-name*
vEdge(config-interface)# (ip dhcp-client | ip *address* *prefix/length*)
vEdge(config-interface)# no shutdown
vEdge(config-interface)# tunnel-interface
vEdge(config-tunnel-interface)# color *color*
vEdge(config-vpn-0)# ip route 0.0.0.0/0 *next-hop*
vEdge(config)# commit and-quit
```

```
cEdge# config-transaction
cEdge(config)# system host-name *hostname*
cEdge(config-system)# system-ip *ip-address*
cEdge(config-system)# site-id *site-id*
cEdge(config-system)# vbond *(dns-name | ip-address)*
cEdge(config-system)# organization-name *organization-name*
cEdge(config)# interface Tunnel *intf-number*
cEdge(config-if)# ip unnumbered *wan-phys-interface*
cEdge(config-if)# tunnel source *wan-phys-interface*
cEdge(config-if)# tunnel mode sdwan
cEdge(config)# interface GigabitEthernet *intf-number*
cEdge(config-if)# ip address *ip-address* *mask*
cEdge(config-if)# no shutdown
cEdge(config)# sdwan
cEdge(config-sdwan)# interface GigabitEthernet *intf-number*
cEdge(config-interface-interface-name)# tunnel-interface
cEdge(config-tunnel-interface)# color *color*
cEdge(config-tunnel-interface)# encapsulation ipsec
cEdge(config)# ip route 0.0.0.0 0.0.0.0 *next-hop*
cEdge(config)# ip domain-lookup
cEdge(config)# ip name-server *dns-server-ip*
cEdge(config)# commit 
```

### 3.1.b Orchestration with zero-touch provisioning and plug-and-play
- Two methods of auto-provisioning WAN Edges
	- PNP: uses HTTPS to connect to Cisco PNP servers (XE SD-WAN)
		- HTTPS connection
		- Device connects to devicehelper.cisco.com
	- ZTP: uses UDP 12346 to connect (vEdge)
		- DTLS tunnel
		- Device connects to ztp.viptela.com
		- Devices use specific interface for ZTP
- Before automatic provisioning, a device template must be attached in vManage

### 3.1.c Data center and regional hub deployments

## 3.2 Configure Cisco SD-WAN data plane

### 3.2.a Circuit termination and TLOC-extension

### 3.2.b Dynamic tunnels

### 3.2.c Underlay-overlay connectivity

## 3.3 Configure OMP

## 3.4 Configure TLOCs

## 3.5 Configure CLI and vManage feature configuration templates
- Configure templates at **vManage > Configuration > Templates**
	- Feature templates **Feature (tab) > Add template**
		- Select multple device types
	- Device templates **Device (tab) > Create template > From feature template or CLI template**
		- Single devie type
		- Then apply to devices
- Device templates can be applied to multiple devices of the same type
	- 4 types of info:
		- Basic info: System, Logging, AAA, BFD, OMP
		- Transport/Management VPN: configuration for VPN 0 and VPN512
		- Service VPN: Configuration for LAN facing interfaces and user VPNs
		- Additional: local policies, security policies, SNMP, etc.
- Feature templates are used to build device templates, can be reused across multiple device templates
	- Allow the option to use variables within configuration parameters
	- Types of values in a feature template:
		- Default: factory default and cannot be changed *example: BFD timers*
		- Global: same configuration any time the template is used
		- Device specific: user defined variables
			- Populate via vManage workflow or from CSV
- CLI templates can be used
- If WAN edge loses control plane connection during config push, 5 minute rollback timer is started

### 3.5.a VRRP

### 3.5.b OSPF

### 3.5.c BGP

### 3.5.d EIGRP

## 3.6 Describe multicast support in Cisco SD-WAN
