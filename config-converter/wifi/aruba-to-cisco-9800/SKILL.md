---
name: aruba-to-cisco-9800
description: Converts Aruba WiFi configurations (Mobility Master or Instant) to Cisco Catalyst 9800 (IOS-XE wireless) format. Handles SSID profiles, Virtual-APs, AP Groups, RF profiles, ARM parameters, roaming and generates a detailed conversion report.
triggers:
  - aruba to cisco 9800
  - aruba to catalyst 9800
  - aruba to ios-xe wireless
  - aruba to 9800
  - convert aruba to 9800
  - migrate aruba to 9800
  - aruba to c9800
---

# Aruba WiFi → Cisco Catalyst 9800 (IOS-XE Wireless) Conversion Ruleset

You are an expert wireless network engineer with deep knowledge of both Aruba (Mobility Master / Instant) and Cisco Catalyst 9800 (IOS-XE wireless) architectures.

## Architecture mapping overview

| Aruba | 9800 (IOS-XE) equivalent |
|---|---|
| SSID profile | WLAN profile |
| Virtual-AP (SSID + VLAN + AAA binding) | Policy profile + Policy tag |
| AP Group | Site tag + Policy tag |
| RF radio profile | RF profile + RF tag |
| AP provisioning profile | AP profile |
| Forward mode split-tunnel / local | Flex profile + Site tag |

Before starting, identify the Aruba source platform:
- **Mobility Master** signatures: `wlan virtual-ap`, `ap-group`, `aaa-profile`, `wlan ssid-profile`
- **Aruba Instant** signatures: `wlan ssid-profile`, `forward-mode`, `rf-band`, `arm`

---

## SSID profile → WLAN profile

**Aruba:**
```
wlan ssid-profile CORPORATE
    essid CORPORATE
    opmode wpa2-aes
    wpa-passphrase MyPassphrase
    no shutdown
```

**Cisco 9800:**
```
wlan CORPORATE 1 CORPORATE
 security wpa psk set-key ascii 0 MyPassphrase
 security wpa akm psk
 security wpa wpa2
 security wpa wpa2 ciphers aes
 no shutdown
```

✅ Direct — WLAN ID assigned sequentially (1, 2, 3...).

---

## SSID profile with 802.1x → WLAN with dot1x

**Aruba:**
```
wlan ssid-profile CORPORATE_DOT1X
    essid CORPORATE_DOT1X
    opmode wpa2-aes
    auth-server CORP_RADIUS
    no shutdown
```

**Cisco 9800:**
```
wlan CORPORATE_DOT1X 2 CORPORATE_DOT1X
 security wpa akm dot1x
 security wpa wpa2
 security wpa wpa2 ciphers aes
 security dot1x authentication-list RADIUS_LIST
 no shutdown
```

⚠️ ADAPTED — RADIUS auth-server reference mapped to Cisco authentication-list. AAA list must be configured separately (`aaa authentication dot1x RADIUS_LIST group <server-group>`).

---

## Security modes mapping

| Aruba | Cisco 9800 | Status |
|---|---|---|
| `opmode wpa2-aes` | `security wpa wpa2 ciphers aes` | ✅ Direct |
| `opmode wpa3-aes` | `security wpa wpa3` | ✅ Direct |
| `opmode wpa2-aes` + `auth-server` | `security wpa akm dot1x` + auth-list | ⚠️ Adapted |
| `wpa-passphrase <key>` | `security wpa akm psk` + `psk set-key ascii 0 <key>` | ✅ Direct |
| `opmode opensystem` | `security open` | ✅ Direct |

---

## Virtual-AP → Policy profile (VLAN + QoS binding)

**Aruba Mobility Master:**
```
wlan virtual-ap CORPORATE_VAP
    ssid-profile CORPORATE
    vlan 10
    aaa-profile CORPORATE_AAA
    no shutdown
```

**Cisco 9800:**
```
wireless profile policy CORPORATE_POLICY
 vlan 10
 no shutdown
```

✅ VLAN binding preserved in policy profile. AAA list referenced in WLAN profile.

---

## AP Group → Policy tag + Site tag

**Aruba:**
```
ap-group BRANCH_GROUP
    virtual-ap CORPORATE_VAP
    virtual-ap CORPORATE_DOT1X_VAP
    dot11a-radio-profile HIGH_PERF_5G
    dot11g-radio-profile STANDARD_24G
```

**Cisco 9800:**
```
wireless tag policy BRANCH_POLICY_TAG
 wlan CORPORATE policy CORPORATE_POLICY
 wlan CORPORATE_DOT1X policy CORP_DOT1X_POLICY

wireless tag site BRANCH_SITE_TAG
 ap-profile BRANCH_AP_PROFILE
 local-site

wireless tag rf BRANCH_RF_TAG
 dot11 5ghz rf-profile HIGH_PERF_5G
 dot11 24ghz rf-profile STANDARD_24G
```

⚠️ ADAPTED — Aruba AP Group split into three 9800 tags: policy tag (WLAN bindings), site tag (AP join parameters), RF tag (radio profiles). All three must be assigned to APs.

**Assign tags to APs:**
```
ap <ap-mac-or-name>
 policy-tag BRANCH_POLICY_TAG
 site-tag BRANCH_SITE_TAG
 rf-tag BRANCH_RF_TAG
```

---

## RF radio profile → 9800 RF profile

**Aruba:**
```
rf dot11a-radio-profile HIGH_PERF_5G
    min-tx-power 7
    max-tx-power 17
    channel-width 40MHz
```

**Cisco 9800:**
```
ap dot11 5ghz rf-profile HIGH_PERF_5G
 channel width 40
 tx-power min 7
 tx-power max 17
 no shutdown
```

⚠️ ADAPTED — ARM dynamic bounds converted to 9800 RF profile bounds. RRM operates within those limits.

---

## Forward mode → Flex profile (FlexConnect)

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

**Cisco 9800:**
```
wireless profile flex BRANCH_FLEX
 vlan-name DATA vlan 10

wireless profile policy CORPORATE_POLICY
 flex-profile BRANCH_FLEX
 vlan 10

wireless tag site BRANCH_SITE_TAG
 ap-profile BRANCH_AP_PROFILE
 flex-profile BRANCH_FLEX
 local-site
```

⚠️ ADAPTED — Aruba split-tunnel / local mode maps to 9800 FlexConnect with flex-profile. VLAN must be available at the AP location. `local-site` keyword enables FlexConnect local switching.

---

## Fast roaming (802.11r / k / v)

**Aruba:**
```
wlan virtual-ap CORPORATE_VAP
    dot11r
    dot11k
    dot11v
    okc
```

**Cisco 9800:**
```
wlan CORPORATE 1 CORPORATE
 dot11r ft-over-ds
 dot11k neighbor-report
 dot11v bss-transition
```

✅ Direct — 802.11r/k/v natively supported on both platforms. OKC covered by 802.11r on 9800.

---

## ARM → RRM

| Aruba (ARM) | Cisco 9800 (RRM) | Status |
|---|---|---|
| `arm` with `min/max-tx-power` | RF profile `tx-power min/max` + `ap dot11 5ghz rrm txpower` | ⚠️ Adapted — RRM operates within RF profile bounds |
| `channel-quality-aware-arm` | `ap dot11 5ghz rrm channel cleanair-event` | ⚠️ Adapted |
| ARM interference avoidance | `ap dot11 5ghz rrm channel cleanair-event` (if CleanAir APs) | ⚠️ Adapted — depends on AP hardware |

---

## AP provisioning → AP profile

**Aruba:**
```
ap provisioning-profile BRANCH_APS
    master <mobility-master-ip>
    ntp-server 10.0.0.30
```

**Cisco 9800:**
```
ap profile BRANCH_AP_PROFILE
 ntp ip 10.0.0.30
 no shutdown
```

✅ Direct — core provisioning parameters preserved.

---

## Management

| Aruba | Cisco 9800 | Status |
|---|---|---|
| `ntp server <ip>` | `ntp server <ip>` | ✅ Direct |
| `logging <ip>` | `logging host <ip>` | ✅ Direct |
| `snmp-server community` | `snmp-server community` | ✅ Direct |

---

## AP tag assignment order (mandatory on 9800)

After creating all profiles and tags, always assign all three tags to each AP. This step has no equivalent on Aruba — it is mandatory on the 9800:

```
ap <ap-name>
 policy-tag BRANCH_POLICY_TAG
 site-tag BRANCH_SITE_TAG
 rf-tag BRANCH_RF_TAG
```

Or configure a default tag set that applies automatically to untagged APs:
```
wireless tag policy default-policy-tag
wireless tag site default-site-tag
wireless tag rf default-rf-tag
```

⚠️ If tags are not assigned, APs join with default tags which may not match the intended SSID/VLAN configuration. Always verify after deployment.

---

## Features not converted (❌)

| Feature | Reason | Action required |
|---|---|---|
| ClearPass / Aruba AAA | Cisco uses ISE — completely different AAA platform | Manual ISE policy design required |
| Aruba user roles | Cisco uses Policy profiles + ACLs | Manual redesign required |
| Aruba ALE / location services | Cisco uses CMX / Cisco Spaces | Manual integration required |
| Aruba UCC / voice optimization | Cisco uses TSPEC / CAC — different approach | Manual redesign required |
| Aruba Instant Site topology | 9800 is controller-based — site topology must be redesigned | Evaluate 9800 deployment model |
| Aruba dynamic segmentation | Cisco uses SGT / TrustSec | Manual redesign required |

---

## Final output instructions

1. Output configuration in this order: WLAN profiles → Policy profiles → Policy tags → Site tags → RF tags → AP profiles → Flex profiles → AP tag assignments.
2. Generate one policy tag, one site tag, and one RF tag per Aruba AP Group.
3. Output the full conversion report with ✅ / ⚠️ / ❌ per element.
4. Always append this disclaimer:

> ⚠️ DISCLAIMER: Always validate this configuration in a lab environment before any production deployment. The Catalyst 9800 tag-based architecture requires careful planning — every AP must have all three tags (policy, site, RF) assigned for correct operation.
