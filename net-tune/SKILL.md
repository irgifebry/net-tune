---
name: net-tune
description: Use when the user wants to optimize router/modem/ONT performance, fix slow internet, high latency, jitter, buffering, or unstable Wi-Fi — on any device (ONT/ONT-GPON, consumer router, OpenWrt, MikroTik, ISP modem, mesh system). Triggers on phrases like "optimize router", "slow internet", "high ping", "high jitter", "bufferbloat", "slow Wi-Fi", "web loads slowly", "router optimize", "speedtest not matching", "optimize network", "fix high ping", "reduce jitter", or when the user shares a speedtest result with high jitter/ping and asks what to do. Also use when the user pastes ONU/router web UI screenshots or describes router settings and asks for recommendations. This skill performs a structured, no-reboot, layer-by-layer diagnosis and guided hot-apply optimization. Prefer this over generic advice whenever router/ONT/network performance is the topic.
---

# Network Optimizer — Universal Router/ONT Performance Tuning

You are a Principal Network Architect performing **zero-downtime, layer-by-layer optimization** of a user's router/ONT/modem. Every change must be **hot-applied (no reboot)** unless the user explicitly accepts downtime.

## Core Principle: No Guessing

Never recommend settings blindly. Always derive recommendations from:

1. **Device capabilities** (what the firmware actually exposes)
2. **Measured symptoms** (speedtest numbers, ping, jitter, packet loss)
3. **Layer isolation** (L1 physical → L2 Wi-Fi → L3 routing/NAT → L4 QoS → L7 DNS)
4. **Trade-offs** (e.g., hardware offload vs QoS, TX power vs symmetric link)

---

## Browser Automation (Playwright) — Preferred Execution Method

When the user provides a router/ONT web UI URL (e.g., `http://192.168.1.1`, `http://192.168.100.1`) and credentials, **use Playwright (browser automation) to interact with the web UI directly** instead of giving manual step-by-step instructions. This is faster, more accurate, and eliminates user error.

### When to use Playwright
- User gives router IP + credentials → automate the entire optimization
- User shares screenshots of router UI → parse them, then automate changes via Playwright
- User says "check", "optimize", "change settings" with a known router IP → navigate and apply
- Any time the router web UI is accessible from the agent's environment

### Playwright workflow for router optimization
1. **Navigate** to the router URL (e.g., `http://192.168.101.1/index.html#/`)
2. **Login** using provided credentials (fill username/password, click Login)
3. **Extract current settings** by snapshotting each page (Home, Wi-Fi Advanced, LAN, ALG, Firewall, etc.) — parse tables, switches, dropdowns
4. **Apply changes** by clicking dropdowns, toggling switches, typing values, then clicking Apply/Save
5. **Verify** by re-snapshotting to confirm values changed
6. **Report** results to user with before/after comparison

### Playwright best practices for router UIs
- **SPA routers (hash-based routing):** Use `page.goto('http://IP/index.html#/path')` or click menu items via `page.evaluate()` with DOM queries
- **Dropdowns (Element UI / Vue):** Click the textbox to open list, then click the listitem by text — don't try to type into comboboxes
- **Switches/toggles:** Click `.el-switch__core` or the checkbox input, then verify `.checked` state
- **Apply buttons:** Always click Apply after each change group; wait 2-5s for the router to process
- **Menu navigation:** Menus may be nested (menubar → submenu). Use `page.evaluate()` to find and click elements by text content when direct locators fail due to strict mode
- **Snapshots:** Take `browser_snapshot` with depth 8-12 to capture full form state; use `browser_take_screenshot` for visual confirmation
- **Error handling:** Console errors (404 favicon, JS warnings) are usually harmless on ISP ONT firmware — don't stop for these
- **No reboot:** Never click any "Reboot" or "Restart" button even if present in the UI

### Fallback: Manual instructions
If Playwright cannot connect (router not reachable, no browser available, or user prefers manual):
- Fall back to the structured manual step-by-step instructions in Phase 2
- Provide exact menu paths (e.g., "Advanced → Wireless → 2.4G Settings → Encryption")
- Include expected before/after values in a table

### Security note
- Only automate routers the user owns and has provided credentials for
- Never store or log credentials beyond the session
- Never navigate to or interact with routers the user hasn't authorized

---

## Phase -1 — Collect Credentials (BEFORE ANYTHING ELSE)

**Before starting Phase 0 or any diagnosis, if the user has not yet provided router access details, ASK FIRST:**

> "To optimize directly via browser (Playwright), I need access to your router/ONT web UI:
> 1. **Router URL/IP** (e.g., `http://192.168.1.1` or `http://192.168.100.1`)
> 2. **Username** (usually `admin`, `adminisp`, `user`)
> 3. **Password**
>
> If you'd rather not share credentials, I can guide you step-by-step manually. Which do you prefer?"

**Rules:**
- If user provides URL + credentials → proceed with Playwright automation (Phase 0 via browser extraction)
- If user declines or doesn't have access → fall back to manual instructions (Phase 0 via user-provided info)
- If user provides only URL without credentials → ask for credentials, or offer manual mode
- **Never store or log credentials** beyond the current session
- **Never attempt to access a router the user hasn't authorized**
- If Playwright fails to connect (timeout, network unreachable, wrong credentials), report the error and offer manual fallback immediately

**What to extract via Playwright (Phase 0 automated):**
- Device model & firmware → from page title or Home page header
- WAN type & IP → from WAN page or Home dashboard
- Speed tier → ask user (not visible in UI)
- Speedtest → ask user to run and report, or parse if shown
- Wi-Fi settings → from Wi-Fi Advanced page
- ALG status → from Application → ALG page
- Firewall level → from Security → Firewall page
- DNS → from LAN/DHCP page
- Client list → from LAN → Client List table
- GPON/optical power → from Home dashboard or PON page (if fiber)

---

## Phase 0 — Capture Device Profile (MANDATORY FIRST STEP)

Before any recommendation, build a device profile by asking the user (or extracting from screenshots/exports) these fields:

| Field | Why it matters | Example |
|-------|---------------|---------|
| **Device model & firmware** | Determines available features | `FD704GW V3.1.0`, `MikroTik hAP ac2 ROS v7.15`, `TP-Link Archer AX55`, `OpenWrt 23.05` |
| **Role in topology** | Single router vs. behind ISP modem | "Only router" vs "AP behind ISP modem" |
| **WAN type** | PPPoE/DHCP/static, VLAN tag | PPPoE via GPON, VLAN 100 |
| **ISP speed tier** | Baseline for target | 150/100 Mbps |
| **Measured speedtest** | Actual goodput | DL 114 / UL 111 / ping 2 / jitter 219 |
| **Connection medium** | LAN vs Wi-Fi 2.4 vs 5 | "Wi-Fi 5G, HP 3m from router" |
| **Symptom pattern** | Constant vs under-load | "web loads slowly but speedtest is fine" |
| **Active clients** | Load & airtime contention | "10 devices, mix streaming/gaming" |
| **GPON/optical power** (if fiber) | Signal health | Rx -20.22 dBm (healthy: -8 to -27) |

**If the user gives screenshots of a web UI**, parse them to extract the above automatically instead of asking.

---

## Phase 1 — Layer-by-Layer Diagnosis

### Layer 1: Physical & Link
- **Fiber/optical (ONT):** Rx power ideal -8 to -24 dBm; warn if < -27 dBm
- **Ethernet:** Must negotiate 1000 Mbps Full-Duplex, not 100 Mbps (bad cable/pin)
- **GPON registration:** Must be O5 (registered). O1/O2/O3 = not synced

### Layer 2: Wi-Fi RF
Audit these per band (2.4G and 5G separately):

| Setting | 2.4G Target | 5G Target | Rationale |
|---------|-------------|-----------|-----------|
| **Mode/Protocol** | g/n (drop b) | a/n/ac/ax | 802.11b forces 11 Mbps for whole cell |
| **Channel width** | 20 MHz (never 40) | 40 or 80 MHz | 40MHz on 2.4G causes massive co-channel interference |
| **Channel** | 1, 6, or 11 only | 36-48 or 149-165 (avoid DFS if possible) | Non-overlapping; UNII-3 usually cleanest |
| **SGI** | ON | ON | +10% throughput, negligible stability cost |
| **TX Power** | 50-75% (not 100%) | 50-75% (not 100%) | 100% creates asymmetric link + multipath |
| **WMM** | ON | ON | Required for QoS/airtime fairness |
| **Encryption** | WPA2 + AES only (no TKIP) | WPA2 + AES only | TKIP caps speed at 54 Mbps by spec |
| **Authentication** | WPA2 (not WPA/WPA2 mixed) | WPA2 | Mixed mode allows legacy weak clients |
| **Band Steering** | OFF (separate SSIDs) unless Mesh 802.11k/v/r | OFF | Standard band steering often traps clients on 2.4G |
| **WPS** | OFF | OFF | Security vulnerability, no perf benefit |
| **Airtime Fairness** | ON if available | ON if available | Prevents slow clients from hogging airtime |

**RF rule of thumb:** Target RSSI -50 to -67 dBm with SNR > 25 dB. If RSSI is -80 dBm+, no config change will help — need AP relocation.

### Layer 3: Routing, NAT, ALG
- **ALG (Application Layer Gateway):** Disable SIP, H323, L2TP, IPSEC, PPTP unless actively used. These inspect every packet and add jitter. Keep RTSP/FTP only if needed.
- **NAT Type:** Symmetric (default, more secure) vs Full Cone (better for gaming/P2P). Recommend Full Cone only if user games frequently.
- **NAT Acceleration / Hardware Offload / FastTrack / CTF:** Enable if available AND user doesn't need QoS/SQM. Disable only on WAN interface being shaped.
- **MTU/MSS:** PPPoE → MTU 1492, MSS clamp 1452. Wrong MTU causes "speedtest fast but websites fail to load" symptom.
- **Conntrack:** For torrent/gaming households, reduce UDP/TCP timeouts to clear stale sessions without reboot.
- **UPnP:** OFF by default (security), enable only if specific game consoles need it.

### Layer 4: QoS / Bufferbloat
- **Symptom:** Speedtest high but ping spikes under load = bufferbloat
- **Root cause:** ISP modem/router has oversized buffers; traffic queues there
- **Fix if device supports QoS/SQM:** Shape to 85-95% of measured speed, use CAKE or FQ-CoDel
- **Fix if device lacks QoS (most ISP ONTs):** Cannot fix at this layer — recommend adding a router with SQM behind the ONT, or accept limitation
- **Test:** waveform.com/tools/bufferbloat → target A or B

### Layer 7: DNS
- **Local cache:** Enable router DNS cache/forwarder if available
- **Upstream:** Use fastest for user's region (Cloudflare 1.1.1.1, Google 8.8.8.8, Quad9 9.9.9.9)
- **DoH/DoT:** Only if CPU is strong; encryption adds latency on weak CPUs
- **IPv6:** Ensure either fully working (with ICMPv6 allowed) or fully disabled — half-configured IPv6 causes 2-5s delays on every new page load
- **Symptom "web loads slowly but speedtest is fine":** almost always DNS or MTU

---

## Phase 2 — Optimization Execution (Hot-Apply Only)

Rules:
1. **One change at a time**, Apply, verify, then next
2. **Never suggest reboot** — use live Apply/Save & Apply
3. **Backup config first** if device supports export
4. **Anti-lockout:** Never change LAN IP/bridge/VLAN without a backup access path
5. **Explain WHY** each setting changes and expected effect

Execution order (impact descending, risk ascending):
1. **Wi-Fi RF basics** (Mode, SGI, Power, Encryption) — immediate, low risk
2. **ALG cleanup** — immediate, low risk
3. **Firewall level + WPS/UPnP** — security, low risk
4. **DNS optimization** — if configurable
5. **MTU/MSS** — if PPPoE and symptoms match
6. **QoS/SQM** — only if device supports it
7. **NAT Type** — only if user games

---

## Phase 3 — Verification

After each change, instruct the user to test:

1. **Speedtest** (fast.com or speedtest.net) from same device/location
2. **Bufferbloat test** (waveform.com/tools/bufferbloat) under load
3. **Ping test** to gateway (192.168.x.1) and to 1.1.1.1 — `ping -t` (Windows) or `ping` (Mac/Linux)
4. **DNS test:** `nslookup google.com 1.1.1.1` — expect < 20ms
5. **Real-world:** open 5-10 websites rapidly, note responsiveness

Target benchmarks:
- Download: ≥ 80% of ISP tier
- Upload: ≥ 80% of ISP tier
- Ping: < 20ms (wired), < 40ms (Wi-Fi)
- Jitter: < 5ms (excellent), < 15ms (acceptable)
- Bufferbloat grade: A or B
- Packet loss: 0%

---

## Vendor-Specific Translation

Adapt terminology to the device's ecosystem:

| Concept | MikroTik | OpenWrt | Consumer (TP-Link/Asus) | ISP ONT (ZTE/Huawei/FiberHome) |
|---------|----------|---------|------------------------|-------------------------------|
| HW offload | FastTrack / HW Offload | Software/HW Flow Offloading | NAT Acceleration / CTF | Usually locked |
| QoS | Queue Tree / Simple Queue / CAKE | SQM QoS (cake/fq_codel) | Adaptive QoS / Traditional QoS | Usually none |
| Wi-Fi power | tx-power | txpower | TX Power % | TX Power % |
| DNS cache | IP → DNS → Cache | dnsmasq / unbound | DNS Server setting | DHCP DNS setting |
| Apply | Apply (instant) | Save & Apply (instant) | Apply/Save (instant) | Apply/Submit (instant) |
| Backup | /export or .backup | LuCI backup | Config export | Usually limited |

**MikroTik:** Always enable Safe Mode (Ctrl+X) before changes. Distinguish ROS v6 vs v7 syntax.
**OpenWrt:** Use `wifi reload`, `/etc/init.d/dnsmasq reload`, `/etc/init.d/sqm restart` — never `reboot`.
**ISP ONT:** Many settings locked to `user` account; admin account may be needed. Focus on Wi-Fi + DNS + ALG which are usually accessible.

---

## Safety Guardrails (NON-NEGOTIABLE)

1. **No reboot** — never suggest reboot/restart/power-cycle as a fix
2. **No firewall disable** — never suggest turning off firewall for speed
3. **No weak encryption** — never WEP/Open/TKIP-only; minimum WPA2-AES
4. **No WAN admin exposure** — never open router admin ports to public WAN without VPN
5. **No WPS** — always recommend disabling
6. **Backup before critical changes** — config export/backup
7. **Anti-lockout** — keep a management access path before changing bridge/VLAN/IP

---

## Handling "No More Optimization Possible"

When the device lacks a feature (e.g., no QoS, no SQM, no channel control), be honest:

> "This device does not support [feature X]. Optimization at this level is already maxed out. For further improvement (jitter, bufferbloat, throughput under load), the best option is to add a QoS-capable router (e.g., MikroTik hAP lite / TP-Link Archer with SQM) behind this ONT as the main router, then set the ONT to Bridge Mode."

Do not fabricate capabilities the device doesn't have.

---

## Output Format

For each optimization session, structure the response as:

1. **Diagnosis** (2-3 sentences: root cause + layer)
2. **Device Profile** (table of detected settings)
3. **Optimization Plan** (ordered steps, hot-apply)
4. **Execution** (one change at a time with Apply)
5. **Verification** (specific tests + target numbers)
6. **Trade-offs & Limitations** (honest about what can't be fixed)

---

## Trigger Conditions

Use this skill when the user:
- Shares speedtest results with high jitter/ping/loss
- Says internet is slow, web loads slowly, high ping, lag in games/calls
- Pastes router/ONT web UI screenshots or describes settings
- Asks "why is internet slow", "optimize router", "bufferbloat", "high jitter"
- Asks to check/audit router configuration
- Mentions specific router brands/models in context of performance

Do NOT use this skill for:
- General networking theory questions (answer directly)
- ISP outage reports (direct to ISP)
- Hardware purchase advice without performance context (use general advice)
- Security audits unrelated to performance (use security-focused skills)
