---
search:
  exclude: true
---

# ROUTE 5.0 Infrastructure Security

## 5.1 Describe IOS AAA using local database

* AAA = Authentication, authorization, and accounting
    * who is permitted to access = Authenticate
    * what can they access = Authorize
    * audit what they did = Accounting

Create a user
```
(config)# username billyz privilege <privilege-level> <secret/password> mypassword
```

Enable AAA - this command immediately applies local authentication to all lines except the console
```
(config)# aaa new-model
```

Define an authentication list (default applies when another list is not specified)
```
! use local authentication database
(config)# aaa authentication login default local
```

----

## 5.2 Describe device security using IOS AAA with TACACS+ and RADIUS
* a. AAA with TACACS+ and RADIUS
    * RADIUS
        * Open standard RFC 2865/2866
        * Uses UDP 1812 (or 1645) for Authentication/Authorization
        * Uses UDP 1813 (or 1646) for Accounting
        * Only encrypts the password portion of the message
    * TACACS
        * Cisco proprietary
        * Uses TCP port 49
        * Encrypts the entire message between device and TACACS+ server
* b. Local privilege authorization fallback

Use RADIUS server, unless it is unavailable
```
(config)# radius server RADIUS1
(config-radius-server)# address ipv4 1.2.3.4
(config-radius-server)# key myS3cretPassword

(config)# aaa group server radius RADIUS-GROUP
(config-sg-radius)# server name RADIUS1

(config)# aaa authentication login default group RADIUS-GROUP local
```

Configure a TACACS+ server with local backup for VTY lines only
```
(config)# tacacs server TACACS1
(config-server-tacacs)# address ipv4 4.3.3.3
(config-server-tacacs)# key mySup3rs3cretKey

(config)# aaa group server tacacs TACACS-GROUP
(config-sg-tacacs+)# server name TACACS1

(config)# aaa authentication login VTY-LINES group TACACS-GROUP local

(config)# line vty 0 4
(config-line)# login authentication VTY-LINES
```

----

## 5.3 Configure and verify device access control
#### a. Lines (VTY, AUX, console)
* VTY (virtual TTY) - used for SSH/Telnet access
* console/auxiliary - used to gain access with a physical terminal connection


Set a password for a specific line
```
(config-line)# password mypassword
```

#### b. Management plane protection

Enforce a minimum password length `security passwords min-length <length>`

Use SSH (encrypted) instead of Telnet (clear text)
```
! These values are used to create an RSA key pair
(config)# hostname myrouter
(config)# ip domain-name zoellers.us

(config)# username bz privilege 15 secret mysecretpw

(config)# crypto key generate rsa modulus 2048

! version 1.99 is default, 'compatiblity mode'
(config)# ip ssh version 2

(config)# line vty 0 15
! only allow SSH connections to the vty line
(config-line)# transport input ssh
```

To verify SSH `show ip ssh`
To verify RSA key `show crypto key mypubkey rsa`
To replace existing key pairs - overwrite them `crypto key zeroize rsa`
#### c. Password encryption
* Passwords that must be stored in a router's configuration should be encrypted so someone reading the configuration cannot decipher them
* Cisco routers have different options for encrypting passwords

`enable secret 5 md5-hash`
`enable secret 4 sha256-hash`

* Passwords (such as line passwords) are clear text - can be encrypted using `service password-encryption` - uses a weak encryption (Vigenere cipher) aka 'Type 7'

----

## 5.4 Configure and verify router security features
#### a. IPv4 access control lists (standard, extended, time-based)

##### Standard ACLs
?

##### Extended ACLs
?

##### Time-based ACLs
* Can be used to change which traffic is filtered based on a range of time
* Relies on the router's clock - important to have valid time (use NTP)
* Two types:
    * Periodic- becomes active/inactive at specified times/days
    * Absolute- fixed starting and stopping date/time

Create a periodic time range
```
(config)# time-range Biz-Hours
(config-time-range)# periodic weekdays 08:00 to 17:00
```

Create an absolute time range
```
(config)# time-range Christmas-Holiday
(config-time-range)# absolute start 17:00 22 Dec 2019 end 08:00 2 Jan 2020
```

Apply a time range to an ACL
```
(config)# access-list 100 deny ip any host 1.2.3.4 time-range Biz-Hours
! or
(config)# ip access-list extended myacl
(config-ext-nacl)# deny ip any any time-range Christmas-Holiday
```

Verify
```
# show access-list 100
Extended IP access list 100
    10 deny ip any host 1.2.3.4 time-range Biz-Hours (inactive)

# show time-range Biz-Hours
time-range entry: Biz-Hours (inactive)
   periodic weekdays 8:00 to 17:00
   used in: IP ACL entry
```

#### b. IPv6 traffic filter
?

#### c. Unicast reverse path forwarding (uRPF)
* Used to block packets with a spoofed IP address
* Checks the source IP of a packet arriving on an interface, and determines if that IP address is reachable
* Can also check to see whether the packet is arriving on the same interface that would be used to send return traffic
* 3 modes:
    * Loose mode: verify that the source IP is reachable based on the FIB
    * Strict mode: Loose mode + packet must arrive on same interface that would be used to send return traffic
    * VRF mode: *not tested on CCNP* Also known as uRPFv3 - operates like Loose mode, but the packets are checked against the FIB for a specific VRF
* Strict mode could cause traffic to be dropped in an asymettric routing scenerio
* If a default route is used in the routing table, that does NOT count as a match since there is no specific routing entry for a packet's address. Use `allow-default` option to allow the default route to match

```
! Loose mode
(config-if)#ip verify unicast source reachable-via any <allow-default>

! Strict mode
(config-if)#ip verify unicast source reachable-via rx <allow-default>
```
Verify
```
#show cef interface gi0/1 | inc RPF
  IP unicast RPF check is enabled
  Input features: uRPF
```