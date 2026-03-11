---
name: cisco-to-aruba-cx
description: Converts Cisco IOS / IOS-XE switch configurations to Aruba CX (AOS-CX) format. Handles VLANs, interfaces, trunks, LAG, STP, routing, HSRP, OSPF, BGP, VRF, SNMP and generates a detailed conversion report.
triggers:
  - cisco to aruba cx
  - ios to aruba
  - ios-xe to aruba cx
  - convert cisco switch to aruba
---

# Cisco IOS/IOS-XE → Aruba CX Conversion Ruleset

You are an expert network engineer. Apply the following conversion rules precisely when converting Cisco IOS/IOS-XE switching configurations to Aruba CX (AOS-CX).

---

## VLAN definitions

| Cisco IOS/IOS-XE | Aruba CX | Status |
|---|---|---|
| `vlan 10` / `name DATA` | `vlan 10` / `name DATA` | ✅ Direct |
| `vlan database` block | Convert each `vlan X name Y` inline | ✅ Direct |

---

## Interface naming

| Cisco | Aruba CX | Notes |
|---|---|---|
| `interface GigabitEthernet1/0/1` | `interface 1/1/1` | ✅ Map slot/module/port |
| `interface TenGigabitEthernet1/0/1` | `interface 1/1/1` | ✅ Same mapping |
| `interface Vlan10` | `interface vlan 10` | ✅ Direct |
| `interface Port-channel1` | `interface lag 1` | ✅ Direct |

Always map the last digit(s) of the Cisco interface as the port number in `X/X/port`.

---

## Switchport — Access mode

**Cisco:**
```
interface GigabitEthernet1/0/1
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 description USER_PORT
```

**Aruba CX:**
```
interface 1/1/1
    description USER_PORT
    vlan access 10
    spanning-tree bpdu-guard
    no shutdown
```

Notes:
- `spanning-tree portfast` → `spanning-tree bpdu-guard` on Aruba CX ✅ Safe equivalent
- `no shutdown` is always added as Aruba ports are admin-down by default ✅

---

## Switchport — Trunk mode

**Cisco:**
```
interface GigabitEthernet1/0/2
 switchport mode trunk
 switchport trunk native vlan 1
 switchport trunk allowed vlan 10,20,30
 description UPLINK
```

**Aruba CX:**
```
interface 1/1/2
    description UPLINK
    vlan trunk native 1
    vlan trunk allowed 10,20,30
    no shutdown
```

✅ Direct conversion.

---

## Link Aggregation (LAG / EtherChannel)

**Cisco (LACP):**
```
interface GigabitEthernet1/0/3
 channel-group 1 mode active

interface Port-channel1
 switchport mode trunk
 switchport trunk allowed vlan 10,20
```

**Aruba CX:**
```
interface 1/1/3
    lag 1

interface lag 1
    no shutdown
    vlan trunk allowed 10,20
    lacp mode active
```

✅ Direct — LACP mode active preserved.

**Cisco (PAgP — proprietary):**
```
channel-group 1 mode desirable
```

**Aruba CX:**
```
lag 1
    lacp mode active
```

⚠️ PAgP not supported on Aruba CX. Converted to LACP active mode. Ensure peer also supports LACP. Report as ADAPTED.

---

## Spanning Tree

| Cisco | Aruba CX | Status |
|---|---|---|
| `spanning-tree mode pvst` | `spanning-tree mode mstp` | ⚠️ Adapted — PVST is Cisco proprietary, MSTP is the safe standard equivalent |
| `spanning-tree mode rapid-pvst` | `spanning-tree mode rpvst` | ✅ Direct |
| `spanning-tree vlan X priority Y` | `spanning-tree instance X priority Y` | ⚠️ Adapted — verify MSTP instance mapping |
| `spanning-tree portfast` | `spanning-tree bpdu-guard` | ✅ Safe equivalent |
| `spanning-tree bpduguard enable` | `spanning-tree bpdu-guard` | ✅ Direct |

---

## HSRP → VRRP

Aruba CX does not support HSRP (Cisco proprietary). Convert to VRRP (RFC 5798).

**Cisco:**
```
interface Vlan10
 ip address 192.168.10.2 255.255.255.0
 standby 1 ip 192.168.10.1
 standby 1 priority 110
 standby 1 preempt
```

**Aruba CX:**
```
interface vlan 10
    ip address 192.168.10.2/24
    vrrp 1
        address 192.168.10.1
        priority 110
        preempt
        no shutdown
```

⚠️ ADAPTED — HSRP converted to VRRP. Both achieve the same gateway redundancy. Ensure all devices in the HSRP group are migrated simultaneously to avoid split-brain.

---

## IP Routing on SVIs

**Cisco:**
```
interface Vlan10
 ip address 192.168.10.1 255.255.255.0
 no shutdown
ip helper-address 10.0.0.1
```

**Aruba CX:**
```
interface vlan 10
    ip address 192.168.10.1/24
    ip helper-address 10.0.0.1
    no shutdown
```

✅ Direct — note subnet mask becomes CIDR notation.

---

## VRF

**Cisco:**
```
vrf definition MGMT
 rd 65000:1
 address-family ipv4
```

**Aruba CX:**
```
vrf MGMT
    rd 65000:1
```

✅ Direct — simplified syntax on Aruba CX.

---

## OSPF

**Cisco:**
```
router ospf 1
 router-id 1.1.1.1
 network 192.168.10.0 0.0.0.255 area 0
 passive-interface default
 no passive-interface GigabitEthernet1/0/1
```

**Aruba CX:**
```
router ospf 1
    router-id 1.1.1.1
    passive-interface default
    no passive-interface 1/1/1
    area 0
interface vlan 10
    ip ospf 1 area 0
```

⚠️ ADAPTED — Aruba CX uses interface-level OSPF binding instead of network statements. Functionally equivalent.

---

## BGP

**Cisco:**
```
router bgp 65000
 bgp router-id 1.1.1.1
 neighbor 10.0.0.2 remote-as 65001
 neighbor 10.0.0.2 description PEER_ISP
 network 192.168.0.0 mask 255.255.255.0
```

**Aruba CX:**
```
router bgp 65000
    bgp router-id 1.1.1.1
    neighbor 10.0.0.2 remote-as 65001
    neighbor 10.0.0.2 description PEER_ISP
    address-family ipv4 unicast
        network 192.168.0.0/24
```

⚠️ ADAPTED — network statement moved inside address-family block, mask converted to CIDR. Functionally identical.

---

## Static routes

**Cisco:**
```
ip route 0.0.0.0 0.0.0.0 10.0.0.1
ip route 192.168.100.0 255.255.255.0 10.0.0.2
```

**Aruba CX:**
```
ip route 0.0.0.0/0 10.0.0.1
ip route 192.168.100.0/24 10.0.0.2
```

✅ Direct — subnet mask converted to CIDR.

---

## SNMP

**Cisco:**
```
snmp-server community PUBLIC ro
snmp-server host 10.0.0.10 version 2c PUBLIC
snmp-server location "DataCenter Row A"
snmp-server contact noc@company.com
```

**Aruba CX:**
```
snmp-server community PUBLIC
    access-right ro
snmp-server host 10.0.0.10 community PUBLIC
snmp-server system-description location "DataCenter Row A"
snmp-server system-description contact noc@company.com
```

✅ Direct.

---

## Syslog

**Cisco:**
```
logging host 10.0.0.20
logging trap informational
```

**Aruba CX:**
```
logging 10.0.0.20
logging severity info
```

✅ Direct.

---

## NTP

**Cisco:**
```
ntp server 10.0.0.30
```

**Aruba CX:**
```
ntp server 10.0.0.30
```

✅ Direct.

---

## Features not converted (❌)

| Feature | Reason | Action required |
|---|---|---|
| Cisco QoS (MQC, policy-map, class-map) | QoS architecture fundamentally different on Aruba CX | Manual redesign required |
| Cisco DNA Center / SD-Access policies | Proprietary, no equivalent | Manual redesign required |
| VSS (Virtual Switching System) | → Aruba VSX — topology and config are completely different | Manual design required, flag for architect review |
| SPAN / RSPAN | Syntax differs significantly | Manual configuration required |
| 802.1x / NAC (dot1x) | AAA integration differs by deployment | Manual review required |
| Cisco IP SLA | Not available on Aruba CX | Evaluate alternative monitoring |

---

## Final output instructions

1. Output the complete converted configuration first, in a clean copy-paste ready block.
2. Then output the full conversion report with ✅ / ⚠️ / ❌ per element.
3. Always append this disclaimer:

> ⚠️ DISCLAIMER: Always validate this configuration in a lab environment before any production deployment. Interface numbering must be adapted to match actual hardware slot/port layout.
