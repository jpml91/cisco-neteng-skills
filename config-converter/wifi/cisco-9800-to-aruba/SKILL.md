---
name: cisco-9800-to-aruba
description: Converts Cisco Catalyst 9800 (IOS-XE wireless) configurations to Aruba Mobility Master or Aruba Instant format. Handles WLANs, policy profiles, policy tags, site tags, RF tags, AP profiles, Flex profiles, roaming parameters and generates a detailed conversion report.
triggers:
  - cisco 9800 to aruba
  - catalyst 9800 to aruba
  - ios-xe wireless to aruba
  - 9800 to aruba
  - convert 9800 aruba
  - migrate 9800 to aruba
  - c9800 to aruba
---

# Cisco Catalyst 9800 (IOS-XE Wireless) → Aruba WiFi Conversion Ruleset

You are an expert wireless network engineer with deep knowledge of both Cisco Catalyst 9800 (IOS-XE wireless) and Aruba (Mobility Master / Instant) architectures.

## Architecture mapping overview

The 9800 uses a **tag-based architecture** fundamentally different from AireOS:

| 9800 (IOS-XE) | Aruba equivalent |
|---|---|
| WLAN profile | SSID profile |
| Policy profile | Virtual-AP profile |
| Policy tag (WLAN → Policy binding) | Virtual-AP (SSID + AAA + VLAN binding) |
| Site tag | AP Group / Site |
| RF tag | RF radio profile |
| AP profile (join parameters) | AP provisioning profile |
| Flex profile | Forward mode split-tunnel |

Before starting, ask the user:
> "Is the target platform **Aruba Mobility Master** (controller-based) or **Aruba Instant** (controller-less)?"

---

## WLAN profile → SSID profile

**Cisco 9800:**
```
wlan CORPORATE 1 CORPORATE
 security wpa psk set-key ascii 0 MyPassphrase
 security wpa akm psk
 security wpa wpa2
 security wpa wpa2 ciphers aes
 no shutdown
```

**Aruba Mobility Master:**
```
wlan ssid-profile CORPORATE
    essid CORPORATE
    opmode wpa2-aes
    wpa-passphrase MyPassphrase
    no shutdown
```

**Aruba Instant:**
```
wlan ssid-profile CORPORATE
    essid CORPORATE
    opmode wpa2-aes
    wpa-passphrase MyPassphrase
    no shutdown
```

✅ Direct — WLAN profile maps cleanly to Aruba SSID profile.

---

## WLAN with 802.1x

**Cisco 9800:**
```
wlan CORPORATE_DOT1X 2 CORPORATE_DOT1X
 security wpa akm dot1x
 security wpa wpa2
 security wpa wpa2 ciphers aes
 security dot1x authentication-list RADIUS_LIST
 no shutdown
```

**Aruba Mobility Master:**
```
wlan ssid-profile CORPORATE_DOT1X
    essid CORPORATE_DOT1X
    opmode wpa2-aes
    auth-server CORP_RADIUS
    no shutdown

wlan virtual-ap CORPORATE_DOT1X_VAP
    ssid-profile CORPORATE_DOT1X
    aaa-profile CORPORATE_AAA
    no shutdown
```

⚠️ ADAPTED — 802.1x auth-list mapped to Aruba AAA profile. RADIUS server must be defined separately in the AAA profile.

---

## Policy profile → Virtual-AP (VLAN + QoS binding)

**Cisco 9800:**
```
wireless profile policy CORPORATE_POLICY
 vlan 10
 no shutdown
```

**Aruba Mobility Master:**
```
wlan virtual-ap CORPORATE_VAP
    ssid-profile CORPORATE
    vlan 10
    aaa-profile CORPORATE_AAA
    no shutdown
```

**Aruba Instant:**
```
wlan ssid-profile CORPORATE
    essid CORPORATE
    vlan 10
    opmode wpa2-aes
    no shutdown
```

✅ VLAN binding preserved. Policy profile and SSID profile are merged into Virtual-AP on Aruba.

---

## Policy tag (WLAN ↔ Policy binding)

**Cisco 9800:**
```
wireless tag policy BRANCH_POLICY_TAG
 wlan CORPORATE policy CORPORATE_POLICY
 wlan CORPORATE_DOT1X policy CORP_DOT1X_POLICY
```

**Aruba:**
```
ap-group BRANCH_GROUP
    virtual-ap CORPORATE_VAP
    virtual-ap CORPORATE_DOT1X_VAP
```

⚠️ ADAPTED — Policy tag (WLAN-to-policy binding) maps to Aruba AP Group (Virtual-AP assignment). One AP Group per set of SSIDs.

---

## Site tag → AP Group / Site

**Cisco 9800:**
```
wireless tag site BRANCH_SITE_TAG
 ap-profile BRANCH_AP_PROFILE
 flex-profile BRANCH_FLEX_PROFILE
 local-site
```

**Aruba Mobility Master:**
```
ap-group BRANCH_GROUP
    dot11a-radio-profile HIGH_PERF
    dot11g-radio-profile STANDARD
```

**Aruba Instant:**
```
# Site tag maps to Aruba Instant "Site" — assign APs to the same site
```

⚠️ ADAPTED — Site tag concept (AP join parameters, flex profile) maps to Aruba AP Group with radio profiles. `local-site` (FlexConnect local switching) maps to split-tunnel forward mode.

---

## RF tag → RF radio profile

**Cisco 9800:**
```
wireless tag rf HIGH_PERF_RF_TAG
 dot11 5ghz rf-profile HIGH_PERF_5G
 dot11 24ghz rf-profile STANDARD_24G
```

**Aruba Mobility Master:**
```
ap-group BRANCH_GROUP
    dot11a-radio-profile HIGH_PERF_5G
    dot11g-radio-profile STANDARD_24G
```

✅ Direct — RF tag radio profile references map to Aruba AP Group radio profile assignments.

---

## RF profile → Aruba RF radio profile

**Cisco 9800:**
```
ap dot11 5ghz rf-profile HIGH_PERF_5G
 channel width 40
 tx-power min 7
 tx-power max 17
 rate RATE_54M mandatory
 no shutdown
```

**Aruba Mobility Master:**
```
rf dot11a-radio-profile HIGH_PERF_5G
    min-tx-power 7
    max-tx-power 17
    channel-width 40MHz
    arm
        min-tx-power 7
        max-tx-power 17
```

⚠️ ADAPTED — RRM bounds converted to Aruba ARM bounds. ARM handles dynamic channel/power within those limits.

---

## AP profile → AP provisioning

**Cisco 9800:**
```
ap profile BRANCH_AP_PROFILE
 description "Branch APs"
 mgmtuser username admin password 0 admin privilege 15
 ntp ip 10.0.0.30
```

**Aruba Mobility Master:**
```
ap provisioning-profile BRANCH_APS
    master <mobility-master-ip>
    ntp-server 10.0.0.30
```

⚠️ ADAPTED — AP profile management parameters mapped to Aruba AP provisioning profile. Admin credentials must be reconfigured per Aruba requirements.

---

## Flex profile → Forward mode

**Cisco 9800:**
```
wireless profile flex BRANCH_FLEX
 vlan-name DATA vlan 10
 vlan-name VOICE vlan 20
```

```
wireless profile policy CORPORATE_POLICY
 flex-profile BRANCH_FLEX
 vlan 10
```

**Aruba Mobility Master:**
```
wlan virtual-ap CORPORATE_VAP
    forward-mode split-tunnel
    vlan 10
```

**Aruba Instant:**
```
wlan ssid-profile CORPORATE
    forward-mode local
    vlan 10
```

⚠️ ADAPTED — FlexConnect local switching (flex-profile) maps to Aruba split-tunnel (MM) or local mode (Instant). VLAN must be available at AP location.

---

## Fast roaming (802.11r / k / v)

**Cisco 9800:**
```
wlan CORPORATE 1 CORPORATE
 dot11r ft-over-ds
 dot11k neighbor-report
 dot11v bss-transition
```

**Aruba:**
```
wlan virtual-ap CORPORATE_VAP
    dot11r
    dot11k
    dot11v
    okc
```

✅ Direct — 802.11r/k/v natively supported on both platforms.

---

## Security modes mapping

| Cisco 9800 | Aruba | Status |
|---|---|---|
| `security wpa wpa2 ciphers aes` | `opmode wpa2-aes` | ✅ Direct |
| `security wpa wpa3` | `opmode wpa3-aes` | ✅ Direct |
| `security wpa akm dot1x` | `auth-server <profile>` | ⚠️ Adapted |
| `security wpa akm psk` | `wpa-passphrase <key>` | ✅ Direct |
| `security wpa akm sae` (WPA3) | `opmode wpa3-aes` + `sae` | ✅ Direct |
| Open SSID | `opmode opensystem` | ✅ Direct |

---

## Management

| Cisco 9800 | Aruba | Status |
|---|---|---|
| `ntp server <ip>` | `ntp server <ip>` | ✅ Direct |
| `logging host <ip>` | `logging <ip>` | ✅ Direct |
| `snmp-server community` | `snmp-server community` | ✅ Direct |

---

## Features not converted (❌)

| Feature | Reason | Action required |
|---|---|---|
| Cisco ISE / RADIUS policy | AAA platform differs — Aruba uses ClearPass | Manual ClearPass policy design required |
| Cisco DNA Center / Catalyst Center wireless policies | Proprietary | Manual redesign required |
| 9800 High Availability (SSO) | Aruba uses VRRP-based MM redundancy | Manual HA design required |
| CleanAir / Spectrum Intelligence | No equivalent — ARM handles interference | ARM profile tuning recommended |
| mDNS gateway / service discovery | Different approach on Aruba | Manual review required |
| Cisco Spaces / CMX (location) | Aruba uses ALE / Meridian | Manual integration required |

---

## Final output instructions

1. Map each 9800 construct (WLAN → Policy → Tag chain) to the corresponding Aruba construct before outputting config.
2. Output configuration grouped by: SSID profiles → Virtual-APs → AP Groups → RF profiles.
3. Clearly separate Mobility Master and Instant syntax where they differ.
4. Output the full conversion report with ✅ / ⚠️ / ❌ per element.
5. Always append this disclaimer:

> ⚠️ DISCLAIMER: Always validate this configuration in a lab environment before any production deployment. The 9800 tag-based architecture must be fully understood before migration — map all WLAN/Policy/Site/RF tag combinations before converting.
