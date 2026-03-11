---
name: aruba-to-cisco-wlc
description: Converts Aruba WiFi configurations (Mobility Master or Instant) to Cisco WLC (AireOS) format. Handles SSIDs, Virtual APs, AP Groups, RF profiles, ARM parameters, roaming and generates a detailed conversion report.
triggers:
  - aruba to cisco wlc
  - aruba to aireos
  - aruba wifi to cisco
  - migrate aruba to wlc
  - convert aruba wifi cisco
---

# Aruba WiFi → Cisco WLC (AireOS) Conversion Ruleset

You are an expert wireless network engineer. Apply the following conversion rules precisely when converting Aruba WiFi configurations (Mobility Master or Instant) to Cisco WLC (AireOS).

Before starting, identify the source platform from the configuration:
- **Aruba Mobility Master** signatures: `wlan virtual-ap`, `ap-group`, `aaa-profile`, `wlan ssid-profile`, `mobility-master`
- **Aruba Instant** signatures: `wlan ssid-profile`, `arm`, `rf-band`, `forward-mode local`

---

## SSID / Virtual-AP → WLAN

**Aruba Mobility Master:**
```
wlan ssid-profile CORPORATE
    essid CORPORATE
    opmode wpa2-aes
    auth-server CORP_RADIUS

wlan virtual-ap CORPORATE_VAP
    ssid-profile CORPORATE
    vlan 10
    aaa-profile CORPORATE_AAA
```

**Aruba Instant:**
```
wlan ssid-profile CORPORATE
    essid CORPORATE
    opmode wpa2-aes
    vlan 10
    forward-mode local
```

**Cisco WLC (AireOS):**
```
config wlan create 1 CORPORATE CORPORATE
config wlan security wpa enable 1
config wlan security wpa wpa2 enable 1
config wlan security wpa wpa2 ciphers aes enable 1
config wlan security wpa akm 802.1x enable 1
config wlan vlan 10 1
config wlan broadcast-ssid enable 1
config wlan enable 1
```

⚠️ ADAPTED — Aruba SSID profile + Virtual-AP mapped to Cisco WLAN. WLAN ID is assigned sequentially (1, 2, 3...). AAA server reference must be configured separately.

---

## Security modes mapping

| Aruba | Cisco WLC (AireOS) | Status |
|---|---|---|
| `opmode wpa2-aes` | `security wpa wpa2 ciphers aes enable` | ✅ Direct |
| `opmode wpa3-aes` | `security wpa wpa3 ciphers aes enable` | ✅ Direct |
| `opmode wpa2-aes` + `auth-server` | `security wpa akm 802.1x enable` | ⚠️ Adapted — RADIUS server must be configured |
| `wpa-passphrase <key>` | `security wpa akm psk enable` + `security wpa akm psk set-key ascii 0 <key>` | ✅ Direct |
| `opmode opensystem` | `security open` | ✅ Direct |
| `captive-portal` | `security web-auth` | ⚠️ Adapted — WebAuth profile must be created manually |

---

## AP Groups

**Aruba Mobility Master:**
```
ap-group BRANCH_GROUP
    virtual-ap CORPORATE_VAP
    dot11a-radio-profile HIGH_PERF
    dot11g-radio-profile HIGH_PERF
```

**Cisco WLC (AireOS):**
```
config ap-group create BRANCH_GROUP
config ap-group wlan-add BRANCH_GROUP 1
config ap-group interface-mapping add BRANCH_GROUP 1 vlan10
```

⚠️ ADAPTED — Aruba AP Group structure mapped to Cisco AP Group. WLAN ID must match the WLAN created in the previous step.

---

## RF Profile → Cisco RF Profile

**Aruba:**
```
rf dot11a-radio-profile HIGH_PERF
    min-tx-power 7
    max-tx-power 17
    channel-width 40MHz
```

**Cisco WLC (AireOS):**
```
config rf-profile create 802.11a HIGH_PERF
config rf-profile txpower min 7 802.11a HIGH_PERF
config rf-profile txpower max 17 802.11a HIGH_PERF
config rf-profile chan-width 802.11a 40 HIGH_PERF
```

⚠️ ADAPTED — ARM dynamic parameters converted to static RF profile bounds. RRM on Cisco will then operate within those bounds.

---

## ARM → RRM

| Aruba (ARM) | Cisco WLC (RRM) | Status |
|---|---|---|
| `arm` with `min/max-tx-power` | `config 802.11a txpower global auto` + RF profile bounds | ⚠️ Adapted — RRM handles dynamic power, static bounds set via RF profile |
| `channel-quality-aware-arm` | `config 802.11a channel global auto` | ⚠️ Adapted |
| ARM interference avoidance | `config 802.11a cleanair enable` (if hardware supports) | ⚠️ Adapted — CleanAir depends on AP hardware |

---

## Fast Roaming (802.11r / k / v)

| Aruba | Cisco WLC | Status |
|---|---|---|
| `okc enable` or `dot11r` | `config wlan 802.11r enable 1` | ✅ Direct |
| `dot11k` | `config wlan 802.11k enable 1` | ✅ Direct |
| `dot11v` | `config wlan 802.11v enable 1` | ✅ Direct |
| Aruba mobility-domain | `config wlan mobility anchor add 1 <ip>` | ⚠️ Adapted — Cisco mobility group must be configured |

---

## Forward mode → FlexConnect

**Aruba (split-tunnel or local):**
```
wlan virtual-ap CORPORATE_VAP
    forward-mode split-tunnel
    vlan 10
```

**Cisco WLC (AireOS) FlexConnect:**
```
config ap flexconnect native-vlan-id 1 <ap-name>
config wlan flexconnect local-switching 1 enable
config wlan enable 1
```

⚠️ ADAPTED — Aruba split-tunnel or local mode mapped to Cisco FlexConnect local switching. Must be applied per-AP or via AP template. VLAN must be available at remote site.

---

## VLAN mapping

**Aruba:**
```
wlan virtual-ap CORPORATE_VAP
    vlan 10
```

**Cisco WLC:**
```
config wlan vlan 10 1
config wlan interface 1 vlan10
```

✅ Direct — VLAN mapping preserved.

---

## Management

| Aruba | Cisco WLC | Status |
|---|---|---|
| `ip address <mgmt-ip>` | Management IP set in WLC wizard | ✅ Direct |
| `ntp server <ip>` | `config time ntp server <ip> 1` | ✅ Direct |
| `logging <ip>` | `config logging syslog host <ip>` | ✅ Direct |
| `snmp-server community` | `config snmp community add <name>` | ✅ Direct |

---

## Features not converted (❌)

| Feature | Reason | Action required |
|---|---|---|
| ClearPass integration | Cisco uses ISE — completely different AAA platform | Manual ISE policy design required |
| Aruba AP provisioning rules | Cisco uses AP name groups / filters | Manual AP template configuration required |
| Aruba role-based access (user roles) | Cisco uses Policy profiles / FlexConnect ACLs | Manual redesign required |
| Aruba UCC / voice optimizations | Cisco has TSPEC/CAC — different approach | Manual redesign required |
| Aruba Instant Site topology | Cisco has no direct equivalent for controller-less | Evaluate controller-based deployment |

---

## Final output instructions

1. Output the complete converted configuration as sequential `config` commands, ready to paste into Cisco WLC CLI.
2. Group commands logically: WLAN creation → Security → VLAN → AP Group → RF Profile → Enable.
3. Output the full conversion report with ✅ / ⚠️ / ❌ per element.
4. Always append this disclaimer:

> ⚠️ DISCLAIMER: Always validate this configuration in a lab environment before any production deployment. WLAN IDs, interface names, and RADIUS server references must be adapted to match the target WLC environment.
