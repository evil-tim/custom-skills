---
name: nmap-scan
description: >
  Planning and execution skill for a network scanning agent using a restricted
  nmap MCP tool. Use this skill whenever the agent needs to perform network
  reconnaissance, host discovery, port scanning, service detection, or OS
  fingerprinting tasks. Triggers on any request involving network scanning,
  target enumeration, open port discovery, service identification, or
  multi-step scan campaigns. Always consult this skill before constructing
  nmap tool calls or planning a sequence of scans — it defines the allowed
  flag subset, correct token format, scan sequencing strategy, and result
  interpretation guidance.
version: 1.0.0
metadata:
  hermes:
    tags: [cybersecurity, analysis, networking, nmap]
    category: red-teaming
---

# Nmap Agent Skill

This skill governs how the agent plans and executes network scans through the
nmap MCP tool. The tool accepts a **list of string tokens** — each token is
either an allowed flag, a flag argument value, or a valid IP/CIDR target.

---

## Token Format Rules

- Each flag is its own token: `"-sS"` not `"-sS -sV"`
- Flag values are separate tokens: `"-p", "22,80,443"` not `"-p 22,80,443"`
- Targets must be valid IP addresses or CIDR ranges — no hostnames
- Multiple targets are separate tokens: `"10.0.0.1", "10.0.0.2"`

**Correct:** `["-sS", "-sV", "-p", "22,80,443", "-T3", "192.168.1.0/24"]`

**Incorrect:** `["-sS -sV", "-p 22,80,443", "192.168.1.0/24"]`

**Flags that require a following value token:** `-p`, `--port-ratio`, `--version-intensity`

---

## Flag Reference

### Port Scanning

| Flag | Name | Notes |
|------|------|-------|
| `-sS` | TCP SYN | Stealthy, fast, requires root. Never completes TCP handshake. Default choice. |
| `-sT` | TCP Connect | No root needed. Slower, more detectable. Fallback when unprivileged. |
| `-sA` | TCP ACK | Maps firewall rules, not open ports. Open/closed both show "unfiltered". |
| `-sW` | TCP Window | Like ACK but exploits TCP window field. OS-dependent reliability. |
| `-sM` | TCP Maimon | FIN+ACK probe. Works on some BSD systems. |
| `-sU` | UDP | Slow due to ICMP rate limiting. Always use with `-p` to limit scope. |
| `-sN` | TCP Null | No flags set. Bypasses some stateless firewalls. Not reliable on Windows. |
| `-sF` | TCP FIN | FIN flag only. Same evasion properties as Null. |
| `-sX` | TCP Xmas | FIN+PSH+URG flags. Same evasion properties as Null. |

### Port Specification

| Flag | Argument | Notes |
|------|----------|-------|
| `-p` | `<ranges>` | e.g. `22`, `1-1024` |

### Host Discovery

| Flag | Notes |
|------|-------|
| `-sL` | List targets only — no packets sent. Useful for verifying CIDR expansion. |
| `-sn` | Ping scan only, no port scan. Fast subnet sweep. |
| `-Pn` | Skip host discovery, assume all hosts up. Use on firewalled targets. |
| `-PS[ports]` | TCP SYN ping. Port list optional, e.g. `-PS80,443`. |
| `-PA[ports]` | TCP ACK ping. Useful when SYN is blocked. |
| `-PU[ports]` | UDP ping. Good for hosts that block TCP probes. |
| `-PY[ports]` | SCTP INIT ping. |
| `-PE` | ICMP echo request ping. Blocked by many firewalls. |
| `-PP` | ICMP timestamp request. Sometimes gets through when echo is blocked. |
| `-PM` | ICMP netmask request. |
| `-PO[protos]` | IP Protocol ping. |

### Version Detection

| Flag | Argument | Notes |
|------|----------|-------|
| `-sV` | — | Probes open ports for service/version info. |
| `--version-intensity` | `0–9` | 0 = lightest, 9 = try all probes. Default is 7. |

### OS Detection

| Flag | Notes |
|------|-------|
| `-O` | Enable OS detection. Requires ≥1 open and ≥1 closed port. Needs root. |
| `--osscan-limit` | Skip OS detection on hosts unlikely to yield useful results. |
| `--osscan-guess` | Report best guess even when confidence is low. |

### Timing

| Flag | Name | Notes |
|------|------|-------|
| `-T0` | Paranoid | 5 min between probes. IDS evasion. |
| `-T1` | Sneaky | 15 sec between probes. |
| `-T2` | Polite | Reduced bandwidth. Good for production networks. |
| `-T3` | Normal | Default adaptive timing. Use this unless there's a reason not to. |
| `-T4` | Aggressive | Assumes fast, reliable network. LAN/lab only. |
| `-T5` | Insane | Very aggressive. May miss results on lossy links. |

### Output / Debug

| Flag | Notes |
|------|-------|
| `-v` / `-vv` | Increase verbosity during scan. |
| `-d` / `-dd` | Debug output. |
| `--reason` | Shows why each port is in its reported state. Useful for firewall analysis. |

### Aggressive

| Flag | Notes |
|------|-------|
| `-A` | Enables `-O` + `-sV` + default scripts + traceroute. Note: NSE scripts (`-sC`) may not be available depending on MCP server configuration. Use for deep single-host analysis only. |

---

## Scan Planning Strategy

### Phase 1 — Host Discovery
Always start with a ping scan before port scanning a range. Avoids wasting
time on dead hosts.

```json
["-sn", "10.0.0.0/24"]
```

Use `-Pn` only when hosts are known to be up but ICMP is blocked, or when
targeting a single confirmed host.

### Phase 2 — Port Scanning
Scan ports on confirmed live hosts only.

- **Default:** `-sS -T3` (stealthy, fast, requires root)
- **Unprivileged fallback:** `-sT`
- **UDP services** (DNS, SNMP, TFTP): `-sU` — always pair with `-p` to limit scope
- **Firewall evasion:** `-sN`, `-sF`, or `-sX` against stateless firewalls; `-sA` to map stateful filtering rules

Port selection:
- Common services: `-p 21,22,23,25,53,80,110,143,443,3306,3389,8080`
- Full range (slow): `-p 1-65535`

### Phase 3 — Service & OS Detection
Run only against confirmed open ports to reduce time and noise.

```json
["-sV", "--version-intensity", "5", "-p", "22,80,443", "10.0.0.5"]
```

OS detection needs ≥1 open and ≥1 closed port to work reliably. Add
`--osscan-guess` when results are ambiguous.

```json
["-O", "--osscan-guess", "10.0.0.5"]
```

---

## Result Interpretation

After each scan, reason about output before planning the next step:

- **No hosts up (phase 1):** Verify the CIDR range; try `-Pn` if ICMP is likely blocked
- **Host up, no open ports:** Firewall filtering likely — try `-sA` for stateful detection or `-sN`/`-sF`/`-sX` for stateless
- **Many open ports:** Prioritize known service ports for version detection before scanning everything
- **OS detection failed:** Needs both open and closed port — expand port range or add `--osscan-guess`
- **Version detection inconclusive:** Raise `--version-intensity` toward 9

---

## Common Patterns

**Subnet recon:**
```
1. ["-sn", "10.0.0.0/24"]                                        — find live hosts
2. ["-sS", "-T3", "--port-ratio", "0.1", "<live_host>"]           — top ports
3. ["-sV", "-p", "<open_ports>", "<live_host>"]                   — version detection
```

**Single host deep scan:**
```
1. ["-A", "-T3", "10.0.0.5"]
```

**UDP service discovery:**
```
1. ["-sU", "-p", "53,67,68,69,123,161,500", "-T3", "10.0.0.0/24"]
```
