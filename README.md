# cisco-neteng-skills

A collection of Claude Code skills for network engineers working with Cisco and Aruba infrastructure.

## Scope

- **Switching** : Cisco IOS / IOS-XE ↔ Aruba CX
- **WiFi** : Cisco WLC ↔ Aruba Mobility Master / Instant
- **Troubleshooting** : Cisco & Aruba switching and WiFi
- **Implementation** : Configuration guidance and best practices

> Firewalling is out of scope intentionally.

## Skills available

| Skill | Description |
|---|---|
| `/config-converter` | Auto-detects vendor and converts configuration |
| `/troubleshooting` | Helps diagnose switching and WiFi issues |
| `/implementation` | Guides configuration implementation |

## Conversion report format

Every conversion outputs a structured report:

```
=== CONVERSION REPORT ===
✅ CONVERTED   (N elements) — apply directly
⚠️  ADAPTED     (N elements) — safe workaround applied, review recommended
❌ NOT CONVERTED (N elements) — manual intervention required

⚠️  DISCLAIMER: Always validate in a lab environment before production deployment.
```

## Supported conversions

### Switching
- Cisco IOS/IOS-XE → Aruba CX
- Aruba CX → Cisco IOS/IOS-XE

### WiFi
- Cisco WLC (AireOS) → Aruba Mobility Master / Instant
- Aruba Mobility Master / Instant → Cisco WLC (AireOS)

## Installation

Copy the desired skill folders into your Claude Code skills directory:

```
~/.claude/skills/
├── config-converter/
├── troubleshooting/
└── implementation/
```

## Important notice

These skills generate configuration suggestions only. **Always validate every generated configuration in a lab environment before any production deployment.** The authors assume no responsibility for misconfiguration or network outages.
