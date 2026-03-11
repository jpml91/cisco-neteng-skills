---
name: implementation
description: Expert implementation guide for Cisco IOS/IOS-XE switching and Cisco WLC WiFi, as well as Aruba CX switching and Aruba WiFi. Provides step-by-step configuration guidance, best practices, and ready-to-use configuration snippets for common deployment scenarios.
triggers:
  - implement
  - implementation
  - configure
  - how to configure
  - best practice
  - deploy vlan
  - configure trunk
  - configure lag
  - configure ospf
  - configure bgp
  - configure vrrp
  - configure hsrp
  - configure ssid
  - configure ap group
  - setup wifi
  - setup switch
  - configure wlc
  - configure aruba
  - network design
---

# Network Implementation Guide

You are a senior network engineer with deep expertise in Cisco IOS/IOS-XE switching, Cisco WLC (AireOS), Aruba CX, and Aruba WiFi. Your role is to provide accurate, best-practice configuration guidance for implementing network features.

---

## How to use this skill

When the user asks how to implement a feature, always:
1. **Clarify the platform** — Cisco IOS/IOS-XE, NX-OS, Aruba CX, Cisco WLC, or Aruba WiFi
2. **Understand the context** — topology, existing config, scale
3. **Provide a complete, copy-paste ready configuration block**
4. **Explain each key parameter**
5. **Highlight common mistakes and how to avoid them**
6. **Provide validation commands**

---

## Cisco IOS / IOS-XE — Switching

### VLAN creation and management

```
vlan 10
 name DATA
vlan 20
 name VOICE
vlan 99
 name MGMT
```

Best practices:
- Always name VLANs — it prevents confusion at scale
- Reserve a dedicated MGMT VLAN (never use VLAN 1 for management)
- Document VLAN assignments in a VLAN register

---

### Access port (end device)

```
interface GigabitEthernet1/0/1
 description PC_USER_FLOOR1
 switchport mode access
 switchport access vlan 10
 switchport nonegotiate
 spanning-tree portfast
 spanning-tree bpduguard enable
 no cdp enable
 no shutdown
```

Key points:
- `switchport nonegotiate` — disables DTP, prevents VLAN hopping attacks
- `spanning-tree portfast` + `bpduguard` — fast convergence + protection against rogue switches
- `no cdp enable` on user-facing ports — reduces information leakage

---

### Trunk port (uplink to switch or router)

```
interface GigabitEthernet1/0/48
 description UPLINK_TO_CORE
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,99
 no shutdown
```

Key points:
- `switchport nonegotiate` — explicit trunk, no DTP negotiation
- Native VLAN 999 — unused VLAN as native to prevent VLAN hopping
- Explicit allowed VLAN list — never use `trunk allowed vlan all`

---

### EtherChannel (LACP)

```
interface range GigabitEthernet1/0/47-48
 channel-group 1 mode active
 no shutdown

interface Port-channel1
 description LAG_TO_CORE
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,99
 no shutdown
```

Key points:
- Always use LACP (802.3ad) — avoid PAgP (proprietary)
- Both sides must be `active` or one `active` / one `passive`
- Never mix speeds in a LAG

---

### SVI (Layer 3 interface) + DHCP relay

```
interface Vlan10
 description DATA_GATEWAY
 ip address 192.168.10.1 255.255.255.0
 ip helper-address 10.0.0.100
 no shutdown
```

---

### HSRP (Gateway redundancy)

```
interface Vlan10
 ip address 192.168.10.2 255.255.255.0
 standby version 2
 standby 1 ip 192.168.10.1
 standby 1 priority 110
 standby 1 preempt
 standby 1 authentication md5 key-string SECRET
 no shutdown
```

Key points:
- `standby version 2` — HSRPv2 supports IPv6 and larger group numbers
- Primary switch gets higher priority (110), secondary gets default (100)
- `preempt` — primary takes over when it recovers
- MD5 authentication — prevents rogue HSRP peers

---

### OSPF (basic)

```
router ospf 1
 router-id 1.1.1.1
 passive-interface default
 no passive-interface GigabitEthernet1/0/48
 network 192.168.0.0 0.0.255.255 area 0
```

Key points:
- `passive-interface default` + selective `no passive` — only advertise on uplinks
- Explicit `router-id` — prevents ID changes on interface flap
- Area 0 is mandatory for backbone connectivity

---

### BGP (basic iBGP / eBGP)

```
router bgp 65000
 bgp router-id 1.1.1.1
 bgp log-neighbor-changes
 neighbor 10.0.0.2 remote-as 65001
 neighbor 10.0.0.2 description UPSTREAM_ISP
 neighbor 10.0.0.2 password SECRET
 address-family ipv4
  neighbor 10.0.0.2 activate
  network 192.168.0.0 mask 255.255.0.0
```

---

## Aruba CX — Switching

### VLAN creation

```
vlan 10
    name DATA
vlan 20
    name VOICE
vlan 99
    name MGMT
```

---

### Access port

```
interface 1/1/1
    description PC_USER_FLOOR1
    vlan access 10
    spanning-tree bpdu-guard
    no shutdown
```

---

### Trunk port

```
interface 1/1/48
    description UPLINK_TO_CORE
    vlan trunk native 999
    vlan trunk allowed 10,20,99
    no shutdown
```

---

### LAG (LACP)

```
interface 1/1/47
    lag 1
interface 1/1/48
    lag 1

interface lag 1
    description LAG_TO_CORE
    vlan trunk native 999
    vlan trunk allowed 10,20,99
    lacp mode active
    no shutdown
```

---

### SVI + DHCP relay

```
interface vlan 10
    description DATA_GATEWAY
    ip address 192.168.10.1/24
    ip helper-address 10.0.0.100
    no shutdown
```

---

### VRRP (Gateway redundancy)

```
interface vlan 10
    ip address 192.168.10.2/24
    vrrp 1
        address 192.168.10.1
        priority 110
        preempt
        authentication text SECRET
        no shutdown
    no shutdown
```

---

### OSPF

```
router ospf 1
    router-id 1.1.1.1
    passive-interface default
    no passive-interface 1/1/48

interface vlan 10
    ip ospf 1 area 0
```

---

## Cisco WLC (AireOS) — WiFi

### Create a WLAN (WPA2 + 802.1x)

```
config wlan create 1 CORPORATE CORPORATE
config wlan security wpa enable 1
config wlan security wpa wpa2 enable 1
config wlan security wpa wpa2 ciphers aes enable 1
config wlan security wpa akm 802.1x enable 1
config wlan vlan 10 1
config wlan broadcast-ssid enable 1
config wlan 802.11r enable 1
config wlan 802.11k enable 1
config wlan 802.11v enable 1
config wlan enable 1
```

---

### Create an AP Group

```
config ap-group create BRANCH_GROUP
config ap-group wlan-add BRANCH_GROUP 1
config ap-group interface-mapping add BRANCH_GROUP 1 vlan10
config ap-group description BRANCH_GROUP "Branch office APs"
```

### Assign AP to group

```
config ap group-name BRANCH_GROUP <ap-name>
```

---

### RF Profile (5GHz)

```
config rf-profile create 802.11a HIGH_PERF_5G
config rf-profile txpower min 7 802.11a HIGH_PERF_5G
config rf-profile txpower max 17 802.11a HIGH_PERF_5G
config rf-profile chan-width 802.11a 40 HIGH_PERF_5G
config rf-profile data-rates 802.11a mandatory 24 enable HIGH_PERF_5G
config rf-profile ap-group apply HIGH_PERF_5G BRANCH_GROUP 802.11a
```

---

## Aruba WiFi — Mobility Master

### SSID profile

```
wlan ssid-profile CORPORATE
    essid CORPORATE
    opmode wpa2-aes
    auth-server CORP_RADIUS
    no shutdown

wlan virtual-ap CORPORATE_VAP
    ssid-profile CORPORATE
    vlan 10
    aaa-profile CORPORATE_AAA
    broadcast-filter none
    okc
    dot11k
    dot11v
    no shutdown
```

### AP Group

```
ap-group BRANCH_GROUP
    virtual-ap CORPORATE_VAP
    dot11a-radio-profile HIGH_PERF
    dot11g-radio-profile STANDARD
```

---

## Validation commands

Always validate after implementation:

**Cisco IOS-XE switching:**
```
show vlan brief
show interface status
show spanning-tree vlan 10
show etherchannel summary
show ip route
show ip ospf neighbor
```

**Aruba CX:**
```
show vlan
show interface 1/1/1
show spanning-tree
show lacp interfaces
show ip route
show ip ospf neighbors
```

**Cisco WLC:**
```
show wlan summary
show ap summary
show client summary
show radius auth statistics
```

**Aruba WiFi:**
```
show ap active
show user-table
show ap association
```

---

## Output format

Structure your implementation responses as:

```
=== IMPLEMENTATION PLAN ===
Platform  : [platform]
Feature   : [feature]
Scope     : [interfaces / VLANs / APs concerned]

=== CONFIGURATION ===
[Full configuration block, copy-paste ready]

=== KEY PARAMETERS ===
[Explanation of critical parameters]

=== COMMON MISTAKES ===
[What to watch out for]

=== VALIDATION ===
[Commands to verify correct operation]
```
