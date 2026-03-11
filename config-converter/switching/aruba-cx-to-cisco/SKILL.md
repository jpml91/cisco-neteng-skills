---
name: aruba-cx-to-cisco
description: Converts Aruba CX (AOS-CX) switch configurations to Cisco IOS / IOS-XE format. Handles VLANs, interfaces, trunks, LAG, STP, routing, VRRP, OSPF, BGP, VRF, SNMP and generates a detailed conversion report.
triggers:
  - aruba cx to cisco
  - aruba to cisco
  - aos-cx to cisco
  - convert aruba switch to cisco
  - migrate aruba to ios
---

# Aruba CX (AOS-CX) → Cisco IOS/IOS-XE Conversion Ruleset

You are an expert network engineer. Apply the following conversion rules precisely when converting Aruba CX (AOS-CX) switching configurations to Cisco IOS/IOS-XE.

---

## VLAN definitions

| Aruba CX | Cisco IOS-XE | Status |
|---|---|---|
| `vlan 10` / `name DATA` | `vlan 10` / `name DATA` | ✅ Direct |

---

## Interface naming

| Aruba CX | Cisco IOS-XE | Notes |
|---|---|---|
| `interface 1/1/1` | `interface GigabitEthernet1/0/1` | ✅ Map port number |
| `interface 1/1/1` (10G) | `interface TenGigabitEthernet1/0/1` | ⚠️ Depends on hardware — ask user if unsure |
| `interface vlan 10` | `interface Vlan10` | ✅ Direct |
| `interface lag 1` | `interface Port-channel1` | ✅ Direct |

Ask the user to confirm interface types (GigabitEthernet vs TenGigabitEthernet) if the hardware model is not specified in the config.

---

## Switchport — Access mode

**Aruba CX:**
```
interface 1/1/1
    description USER_PORT
    vlan access 10
    spanning-tree bpdu-guard
    no shutdown
```

**Cisco IOS-XE:**
```
interface GigabitEthernet1/0/1
 description USER_PORT
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
```

✅ Direct.

---

## Switchport — Trunk mode

**Aruba CX:**
```
interface 1/1/2
    description UPLINK
    vlan trunk native 1
    vlan trunk allowed 10,20,30
    no shutdown
```

**Cisco IOS-XE:**
```
interface GigabitEthernet1/0/2
 description UPLINK
 switchport mode trunk
 switchport trunk native vlan 1
 switchport trunk allowed vlan 10,20,30
 no shutdown
```

✅ Direct.

---

## Link Aggregation (LAG → EtherChannel)

**Aruba CX:**
```
interface 1/1/3
    lag 1

interface lag 1
    no shutdown
    vlan trunk allowed 10,20
    lacp mode active
```

**Cisco IOS-XE:**
```
interface GigabitEthernet1/0/3
 channel-group 1 mode active

interface Port-channel1
 switchport mode trunk
 switchport trunk allowed vlan 10,20
 no shutdown
```

✅ Direct — LACP mode active preserved.

---

## Spanning Tree

| Aruba CX | Cisco IOS-XE | Status |
|---|---|---|
| `spanning-tree mode mstp` | `spanning-tree mode rapid-pvst` | ⚠️ Adapted — MSTP converted to Rapid-PVST+ (Cisco default). Ensure STP topology is reviewed post-migration |
| `spanning-tree mode rpvst` | `spanning-tree mode rapid-pvst` | ✅ Direct |
| `spanning-tree bpdu-guard` | `spanning-tree bpduguard enable` | ✅ Direct |
| `spanning-tree instance X priority Y` | `spanning-tree vlan X priority Y` | ⚠️ Adapted — verify VLAN-to-instance mapping |

---

## VRRP → HSRP

Cisco environments typically use HSRP. However, Cisco IOS-XE also supports VRRP natively. Two options are provided — prefer VRRP for a safer, standard conversion.

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

**Option A — Convert to VRRP (recommended, safe, RFC standard):**
```
interface Vlan10
 ip address 192.168.10.2 255.255.255.0
 vrrp 1 ip 192.168.10.1
 vrrp 1 priority 110
 vrrp 1 preempt
 no shutdown
```

**Option B — Convert to HSRP (if Cisco-only environment):**
```
interface Vlan10
 ip address 192.168.10.2 255.255.255.0
 standby 1 ip 192.168.10.1
 standby 1 priority 110
 standby 1 preempt
 no shutdown
```

⚠️ ADAPTED — Default output is VRRP (Option A). Mention Option B to user. Always flag for review.

---

## IP Routing on SVIs

**Aruba CX:**
```
interface vlan 10
    ip address 192.168.10.1/24
    ip helper-address 10.0.0.1
    no shutdown
```

**Cisco IOS-XE:**
```
interface Vlan10
 ip address 192.168.10.1 255.255.255.0
 ip helper-address 10.0.0.1
 no shutdown
```

✅ Direct — CIDR converted to subnet mask.

---

## VRF

**Aruba CX:**
```
vrf MGMT
    rd 65000:1
```

**Cisco IOS-XE:**
```
vrf definition MGMT
 rd 65000:1
 address-family ipv4
 exit-address-family
```

✅ Direct — expanded syntax for Cisco.

---

## OSPF

**Aruba CX:**
```
router ospf 1
    router-id 1.1.1.1
    passive-interface default
    no passive-interface 1/1/1
interface vlan 10
    ip ospf 1 area 0
```

**Cisco IOS-XE:**
```
router ospf 1
 router-id 1.1.1.1
 network 192.168.10.0 0.0.0.255 area 0
 passive-interface default
 no passive-interface GigabitEthernet1/0/1
```

⚠️ ADAPTED — Interface-level OSPF binding converted to network statements. Wildcard mask derived from interface subnet. Functionally equivalent.

---

## BGP

**Aruba CX:**
```
router bgp 65000
    bgp router-id 1.1.1.1
    neighbor 10.0.0.2 remote-as 65001
    neighbor 10.0.0.2 description PEER_ISP
    address-family ipv4 unicast
        network 192.168.0.0/24
```

**Cisco IOS-XE:**
```
router bgp 65000
 bgp router-id 1.1.1.1
 neighbor 10.0.0.2 remote-as 65001
 neighbor 10.0.0.2 description PEER_ISP
 network 192.168.0.0 mask 255.255.255.0
```

⚠️ ADAPTED — address-family block flattened, CIDR converted to mask. Functionally identical.

---

## Static routes

**Aruba CX:**
```
ip route 0.0.0.0/0 10.0.0.1
ip route 192.168.100.0/24 10.0.0.2
```

**Cisco IOS-XE:**
```
ip route 0.0.0.0 0.0.0.0 10.0.0.1
ip route 192.168.100.0 255.255.255.0 10.0.0.2
```

✅ Direct — CIDR converted to subnet mask.

---

## SNMP

**Aruba CX:**
```
snmp-server community PUBLIC
    access-right ro
snmp-server host 10.0.0.10 community PUBLIC
```

**Cisco IOS-XE:**
```
snmp-server community PUBLIC ro
snmp-server host 10.0.0.10 version 2c PUBLIC
```

✅ Direct.

---

## Syslog

**Aruba CX:**
```
logging 10.0.0.20
logging severity info
```

**Cisco IOS-XE:**
```
logging host 10.0.0.20
logging trap informational
```

✅ Direct.

---

## NTP

**Aruba CX:**
```
ntp server 10.0.0.30
```

**Cisco IOS-XE:**
```
ntp server 10.0.0.30
```

✅ Direct.

---

## Features not converted (❌)

| Feature | Reason | Action required |
|---|---|---|
| VSX (Virtual Switching Extension) | Cisco equivalent is VSS or StackWise — topology and cabling are fundamentally different | Manual architect review required |
| Aruba CX QoS | QoS architecture (MQC) is fundamentally different on Cisco | Manual redesign required |
| Aruba CX policy-based routing | Syntax and capabilities differ significantly | Manual review required |
| SPAN (Aruba mirroring) | Syntax differs | Manual configuration required |
| 802.1x / NAC | AAA/RADIUS integration differs | Manual review required |

---

## Final output instructions

1. Output the complete converted configuration first, in a clean copy-paste ready block.
2. Then output the full conversion report with ✅ / ⚠️ / ❌ per element.
3. Always append this disclaimer:

> ⚠️ DISCLAIMER: Always validate this configuration in a lab environment before any production deployment. Interface naming (GigabitEthernet vs TenGigabitEthernet) must be confirmed against actual hardware.
