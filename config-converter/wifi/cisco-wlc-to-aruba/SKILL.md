---
name: cisco-wlc-to-aruba
description: Converts Cisco WLC (AireOS) WiFi configurations to Aruba Mobility Master or Aruba Instant format. Handles SSIDs, AP Groups, RF Profiles, VLAN mapping, roaming parameters and generates a detailed conversion report.
triggers:
  - cisco wlc to aruba
  - aireos to aruba
  - cisco wifi to aruba
  - migrate wlc to aruba
  - convert wlc aruba
---

# Cisco WLC (AireOS) → Aruba WiFi Conversion Ruleset

You are an expert wireless network engineer. Apply the following conversion rules precisely when converting Cisco WLC (AireOS) configurations to Aruba (Mobility Master or Instant).

Before starting, ask the user:
> "Is the target platform **Aruba Mobility Master** (controller-based, large enterprise) or **Aruba Instant** (controller-less, SMB/branch)?"

Adapt output syntax accordingly. Differences are noted per section.

---

## WLAN / SSID definitions

**Cisco WLC (AireOS):**
```
config wlan create 1 CORPORATE CORPORATE
config wlan security wpa akm 802.1x enable 1
config wlan security wpa wpa2 enable 1
config wlan security wpa wpa2 ciphers aes enable 1
config wlan vlan 10 1
config wlan broadcast-ssid enable 1
config wlan enable 1
```

**Aruba Mobility Master:**
```
wlan ssid-profile CORPORATE
    essid CORPORATE
    wpa-passphrase <not applicable — 802.1x>
    opmode wpa2-aes
    auth-server <aaa-profile reference>

wlan virtual-ap CORPORATE_VAP
    ssid-profile CORPORATE
    vlan 10
    aaa-profile CORPORATE_AAA
    broadcast-filter none
    no shutdown
```

**Aruba Instant:**
```
wlan ssid-profile CORPORATE
    essid CORPORATE
    opmode wpa2-aes
    vlan 10
    auth-server <server profile reference>
    no shutdown
```

⚠️ ADAPTED — Cisco WLAN ID + profile approach mapped to Aruba SSID profile + Virtual AP. 802.1x auth-server reference must be configured separately in AAA profile.

---

## Security modes mapping

| Cisco WLC | Aruba | Status |
|---|---|---|
| `security wpa wpa2 ciphers aes` | `opmode wpa2-aes` | ✅ Direct |
| `security wpa wpa3 ciphers aes` | `opmode wpa3-aes` | ✅ Direct |
| `security wpa akm 802.1x` | `auth-server <profile>` | ⚠️ Adapted — AAA profile must be configured |
| `security wpa akm psk` + passphrase | `wpa-passphrase <key>` | ✅ Direct |
| Open (no security) | `opmode opensystem` | ✅ Direct |
| `security web-auth` | `captive-portal` profile | ⚠️ Adapted — captive portal profile required, out of scope for ISE |

---

## AP Groups

**Cisco WLC:**
```
config ap-group create BRANCH_GROUP
config ap-group wlan-add BRANCH_GROUP 1
config ap-group interface-mapping add BRANCH_GROUP 1 vlan10
```

**Aruba Mobility Master:**
```
ap-group BRANCH_GROUP
    virtual-ap CORPORATE_VAP
    dot11a-radio-profile <rf-profile>
    dot11g-radio-profile <rf-profile>
```

**Aruba Instant:**
```
# Aruba Instant uses "Sites" instead of AP Groups
# Assign APs to the same site and apply SSID profiles globally or per-site
```

⚠️ ADAPTED — Cisco AP Groups mapped to Aruba AP Groups (MM) or Sites (Instant). WLAN-to-VLAN mapping preserved via Virtual-AP VLAN assignment.

---

## RF Profiles

**Cisco WLC:**
```
config rf-profile create 802.11a HIGH_PERF
config rf-profile data-rates 802.11a supported 6 enable HIGH_PERF
config rf-profile data-rates 802.11a supported 9 enable HIGH_PERF
config rf-profile data-rates 802.11a mandatory 24 enable HIGH_PERF
config rf-profile txpower min 7 802.11a HIGH_PERF
config rf-profile txpower max 17 802.11a HIGH_PERF
config rf-profile chan-width 802.11a 40 HIGH_PERF
```

**Aruba Mobility Master:**
```
rf dot11a-radio-profile HIGH_PERF
    min-tx-power 7
    max-tx-power 17
    channel-width 40MHz
    arm
        min-tx-power 7
        max-tx-power 17
```

**Aruba Instant:**
```
rf-band all
arm
    min-tx-power 7
    max-tx-power 17
    channel-width 40
```

⚠️ ADAPTED — RF profile parameters mapped. ARM (Aruba's RRM equivalent) handles dynamic channel/power selection. Data rate filtering is managed via ARM optimization.

---

## RRM / ARM (Radio Resource Management)

| Cisco WLC (RRM) | Aruba (ARM) | Status |
|---|---|---|
| `config 802.11a channel global auto` | `arm` with `channel-quality-aware-arm` | ⚠️ Adapted — ARM replaces RRM, behavior similar |
| `config 802.11a txpower global auto` | `arm` with `min/max-tx-power` | ⚠️ Adapted |
| `config 802.11a cleanair enable` | Not applicable — ARM handles interference | ⚠️ Adapted — no CleanAir equivalent, ARM provides interference avoidance |

---

## Fast Roaming (802.11r / k / v)

| Cisco WLC | Aruba | Status |
|---|---|---|
| `config wlan mobility anchor add 1 <ip>` | `mobility-domain` configuration | ⚠️ Adapted — Aruba uses mobility domains |
| `config wlan 802.11r enable 1` | `okc enable` or `dot11r` | ✅ Aruba supports 802.11r natively |
| `config wlan 802.11k enable 1` | `dot11k` | ✅ Direct |
| `config wlan 802.11v enable 1` | `dot11v` | ✅ Direct |

---

## FlexConnect → Split Tunnel / Local Mode

**Cisco WLC (FlexConnect):**
```
config ap flexconnect native-vlan-id 1 <ap-name>
config wlan flexconnect local-switching 1 enable
```

**Aruba Mobility Master equivalent (split-tunnel):**
```
wlan virtual-ap CORPORATE_VAP
    forward-mode split-tunnel
    vlan 10
```

**Aruba Instant equivalent:**
```
# Aruba Instant operates natively in local switching mode
# No specific FlexConnect equivalent needed
wlan ssid-profile CORPORATE
    forward-mode local
```

⚠️ ADAPTED — FlexConnect local switching mapped to Aruba split-tunnel (MM) or local forward mode (Instant). Verify VLAN availability at AP location.

---

## VLAN mapping per SSID

**Cisco WLC:**
```
config wlan vlan 10 1
config wlan interface 1 vlan10
```

**Aruba:**
```
wlan virtual-ap CORPORATE_VAP
    vlan 10
```

✅ Direct — VLAN assignment preserved at Virtual-AP level.

---

## Management / OOB

| Cisco WLC | Aruba MM | Status |
|---|---|---|
| Management IP | `ip address <mgmt-ip>` on mgmt interface | ✅ Direct |
| NTP `config time ntp server <ip>` | `ntp server <ip>` | ✅ Direct |
| Syslog `config logging syslog host <ip>` | `logging <ip>` | ✅ Direct |
| SNMP | `snmp-server community` | ✅ Direct |

---

## Features not converted (❌)

| Feature | Reason | Action required |
|---|---|---|
| ISE / AAA integration | AAA/RADIUS server config depends on deployment | Manual RADIUS profile configuration required |
| Guest portals (Cisco ISE) | Aruba uses ClearPass — completely different platform | Manual ClearPass policy design required |
| Cisco DNA / Catalyst Center wireless policies | Proprietary — no equivalent | Manual redesign required |
| Anchor controller mobility tunnels | Complex multi-controller topology | Manual architect review required |
| 802.1x certificates | Certificates must be re-enrolled on Aruba | Manual PKI operation required |

---

## Final output instructions

1. Output the complete converted configuration for each SSID/AP-Group, in a clean copy-paste ready block.
2. Clearly separate Mobility Master syntax from Instant syntax if both are relevant.
3. Output the full conversion report with ✅ / ⚠️ / ❌ per element.
4. Always append this disclaimer:

> ⚠️ DISCLAIMER: Always validate this configuration in a lab environment before any production deployment. AAA/RADIUS server profiles and ClearPass integration must be configured separately.
