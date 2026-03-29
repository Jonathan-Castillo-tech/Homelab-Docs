# DNS Hardening & Browser Privacy Audit
**Date:** 2026-03-29  
**System:** Fedora (primary workstation)  
**Scope:** DNS-over-TLS enforcement, resolver leak analysis, browser fingerprint hardening  

---

## Summary

Identified and resolved three security misconfigurations on a Fedora workstation connected to a self-hosted AdGuard Home DNS server (Raspberry Pi 4B, Rocky Linux 9.7). The system appeared to be leaking DNS queries despite a DoT configuration being in place. Root cause was traced to a NetworkManager per-connection DNS override stripping the TLS hostname, causing silent fallback. Browser fingerprint hardening was also performed on Brave.

---

## Environment

| Component | Details |
|---|---|
| Workstation OS | Fedora (Linux x86_64) |
| Browser | Brave 145 |
| DNS Resolver | Self-hosted AdGuard Home on Raspberry Pi 4B |
| Pi OS | Rocky Linux 9.7 Minimal |
| Pi IP | 192.168.8.110 |
| DoT Hostname | kukimonji55.duckdns.org |
| AdGuard Upstreams | Quad9, Mullvad, ControlD (all via TLS) |

---

## Issue 1 — DNSOverTLS Set to Opportunistic

### Problem
`/etc/systemd/resolved.conf` had `DNSOverTLS=opportunistic`. Opportunistic mode attempts TLS but silently falls back to plaintext UDP on failure — no warning, no error, no indication in logs under normal conditions.

### Discovery
DNS leak test at browserleaks.com showed ControlD, Quad9, and i3d.net resolvers instead of only the expected AdGuard upstreams. Initial assumption was a leak; later confirmed to be expected upstream behavior, but the opportunistic setting was still a real misconfiguration worth fixing.

### Fix
```ini
# /etc/systemd/resolved.conf
[Resolve]
DNS=192.168.8.110#kukimonji55.duckdns.org
DNSOverTLS=yes
DNSSEC=no
```

```bash
sudo systemctl restart systemd-resolved
```

### Result
`resolvectl status` confirmed `DNS over TLS: yes` globally.

---

## Issue 2 — NetworkManager Per-Connection DNS Override

### Problem
NetworkManager had `192.168.8.110` hardcoded in the "Team Jesus" WiFi connection profile (`ipv4.dns`) **without** the TLS hostname (`#kukimonji55.duckdns.org`). This per-link setting overrode the global `resolved.conf` configuration, pushing a bare IP to the interface. Without the hostname, TLS certificate verification could not be performed against the DoT endpoint.

### Discovery
```bash
resolvectl status
# Global showed: 192.168.8.110#kukimonji55.duckdns.org (correct)
# wlp0s20f3 showed: 192.168.8.110 (bare IP — no hostname)

nmcli connection show "Team Jesus" | grep dns
# ipv4.dns: 192.168.8.110  ← overriding global config
```

### Fix
**Step 1 — Tell NetworkManager to hand off DNS to systemd-resolved:**
```bash
sudo bash -c 'cat > /etc/NetworkManager/conf.d/dns.conf << EOF
[main]
dns=systemd-resolved
EOF'
```

**Step 2 — Remove the bare IP from the connection profile:**
```bash
nmcli connection modify "Team Jesus" ipv4.dns ""
nmcli connection up "Team Jesus"
```

### Verification
```bash
resolvectl status | grep -E "DNS Server|DNS over TLS"
# Output: 192.168.8.110#kukimonji55.duckdns.org (single entry, hostname intact)
```

### Result
Per-link DNS now inherits from global resolved config with full TLS hostname. NM no longer overrides DNS resolution.

---

## Issue 3 — Browser Fingerprint Leaks (Brave)

### Problem
creepjs fingerprint test revealed two significant leaks:

**WebGL Renderer exposed (high confidence):**
```
Google Inc. (Intel)
ANGLE (Intel, Mesa Intel(R) UHD Graphics (TGL GT2), OpenGL 4.6)
```
13.24 bits of identifying information — only 1 in ~9,600 browsers share this exact value.

**WebRTC STUN leak:**
```
ip: [REDACTED]  (real public IP exposed via STUN)
```
WebRTC's STUN protocol was bypassing all DNS/privacy controls and exposing the real public IP directly to any page that queried it.

### Fix

**WebRTC:**  
`brave://settings/privacy` → WebRTC IP handling policy → **Disable non-proxied UDP**

**Fingerprinting:**  
`brave://flags` → Enable `#brave-farbling` and `#brave-show-strict-fingerprinting-mode`  
`brave://settings/shields` → Fingerprinting → **Strict**

### Verification (creepjs before/after)

| Metric | Before | After |
|---|---|---|
| WebRTC | ip: 2.142.188.54 | blocked |
| WebGL | Intel UHD (confidence: high) | UKkaNteP/SJjRnTJj (confidence: low) |
| Canvas | 14% rgba noise | 13% rgba noise |
| Resistance mode | allow | strict |
| FP ID | 6521175c... | 906a970d... (different — good) |

---

## AdGuard Configuration Note

DNS leak tests initially appeared alarming — showing Quad9, Mullvad, and ControlD resolvers. These are **not leaks**. They are AdGuard Home's configured upstream DoT providers. The leak test sees public resolvers that AdGuard queries on behalf of the client. The full chain is:

```
Fedora (DoT) → Pi AdGuard Home → Quad9 / Mullvad / ControlD (DoT)
```

All hops are encrypted. ISP cannot see DNS queries.

**AdGuard upstream config (DNS Settings):**
```
tls://dns.quad9.net
tls://base.dns.mullvad.net
tls://p2.freedns.controld.com
```

**Bootstrap DNS (plaintext — required to resolve DoT hostnames initially):**
```
9.9.9.10
149.112.112.10
```

Note: Upstream list had duplicates — each provider listed twice. Cleaned up to single entries.

---

## Residual Exposures (Known / Accepted)

| Item | Details | Mitigation |
|---|---|---|
| System fonts | DejaVu Sans, Liberation Mono, Noto (Linux-specific) | No practical fix without breaking rendering |
| Timezone | Europe/Madrid | Accurate — not a concern |
| Screen resolution | 1920x1080 | Extremely common, low risk |
| User-Agent | Linux x86_64, Chrome 145 | UA reduction active via Brave |

---

## Skills Demonstrated

- `systemd-resolved` configuration and DoT enforcement
- NetworkManager DNS override diagnosis and remediation  
- `resolvectl`, `nmcli` diagnostic workflow
- DNS resolver leak analysis and interpretation
- Browser fingerprint surface analysis (creepjs, EFF Cover Your Tracks, ipleak.net)
- WebRTC attack surface mitigation
- Brave privacy hardening (Farbling, Strict fingerprinting mode)

---

## References

- [browserleaks.com](https://browserleaks.com)
- [dnsleaktest.com](https://dnsleaktest.com)
- [abrahamjuliot.github.io/creepjs](https://abrahamjuliot.github.io/creepjs/)
- [coveryourtracks.eff.org](https://coveryourtracks.eff.org)
- [systemd-resolved man page](https://www.freedesktop.org/software/systemd/man/resolved.conf.html)
