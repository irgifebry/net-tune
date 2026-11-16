# net-tune

**Universal router/ONT optimizer skill for Claude Code** — zero-downtime, layer-by-layer network performance tuning with Playwright browser automation.

## What it does

`net-tune` turns Claude into a **Principal Network Architect** that diagnoses and optimizes any router, ONT, or modem — without rebooting. It follows a structured 4-phase workflow:

1. **Phase -1 — Collect Credentials:** Asks for router URL + username + password (or falls back to manual mode)
2. **Phase 0 — Device Profile:** Extracts device model, firmware, WAN type, speed tier, symptoms, client count (via Playwright or user input)
3. **Phase 1 — Layer-by-Layer Diagnosis:** Isolates bottlenecks across OSI layers (L1 physical → L2 Wi-Fi RF → L3 routing/NAT → L4 QoS/bufferbloat → L7 DNS)
4. **Phase 2 — Hot-Apply Optimization:** Applies changes one at a time with live Apply (no reboot), in impact-descending order
5. **Phase 3 — Verification:** Speedtest, bufferbloat test, ping, DNS test, real-world responsiveness

## Key features

- **Universal** — not hardcoded to any specific device. Recommendations are derived from the device profile, not fixed values
- **Vendor-agnostic** — translates concepts (HW offload, QoS, TX power, DNS cache) to MikroTik / OpenWrt / consumer (TP-Link, Asus) / ISP ONT (ZTE, Huawei, FiberHome) terminology
- **Playwright-first** — if credentials are provided, Claude automates the entire optimization via browser automation (navigate → login → extract → apply → verify). Falls back to manual step-by-step instructions
- **Zero-downtime** — never suggests reboot/restart. All changes are hot-applied
- **Safety guardrails** — never disables firewall, never weakens encryption below WPA2-AES, never exposes admin ports to WAN, always recommends disabling WPS
- **Honest limitations** — if the device lacks a feature (e.g., no QoS/SQM), says so and recommends adding a router with SQM behind the ONT instead of fabricating capabilities

## Trigger phrases

The skill activates automatically when the user mentions:

- `optimize router`, `slow internet`, `high ping`, `high jitter`
- `bufferbloat`, `slow Wi-Fi`, `web loads slowly`
- `router optimize`, `speedtest not matching`, `optimize network`
- `fix high ping`, `reduce jitter`
- Shares a speedtest result with high jitter/ping and asks what to do
- Pastes router/ONT web UI screenshots or describes router settings
- Asks to check/audit router configuration

## Supported devices

| Category | Examples |
|----------|----------|
| ISP ONT/ONU | ZTE, Huawei, FiberHome, Nokia (GPON) |
| Consumer routers | TP-Link, Asus, Xiaomi, D-Link |
| Open-source | OpenWrt, MikroTik RouterOS, UniFi, EdgeRouter |
| Mesh systems | Any mesh with web UI |

## Installation

### As a Claude Code skill

1. Copy the `net-tune/` folder to `~/.claude/skills/` (or `~/.agents/skills/`)
2. The skill will be available in the next session

### From .skill file

```bash
# Download net-tune.skill from releases, then:
# Place in ~/.claude/skills/ or use Claude Code skill installer
```

### From source

```bash
git clone https://github.com/irgifebry/net-tune.git
cp -r net-tune/net-tune ~/.claude/skills/
```

## File structure

```
net-tune/
└── SKILL.md    # Skill definition (frontmatter + instructions)
```

## Usage example

```
User: optimize my router, internet is slow, high jitter
Claude: [activates net-tune skill]
  → Asks for router URL + credentials
  → Navigates via Playwright, extracts device profile
  → Diagnoses layer-by-layer
  → Applies optimizations one at a time
  → Verifies with speedtest/bufferbloat/ping
```

## License

MIT
