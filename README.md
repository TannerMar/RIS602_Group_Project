# Capsulecorp Attack Chain

Automated internal network penetration testing framework targeting MSSQL servers in the Capsulecorp Pentest lab environment. Based on the methodology from *The Art of Network Penetration Testing* (RIS602 coursework).

## Overview

This framework automates the full internal network penetration testing lifecycle across 10 scripts. It starts with host and service discovery, identifies MSSQL servers through banner analysis, brute-forces credentials, gains OS command execution through xp_cmdshell, dumps NTLM hashes, establishes persistence, performs lateral movement via Pass-the-Hash and credential reuse, pivots through dual-homed hosts into new network segments using netsh portproxy, cleans up all remote artifacts, and generates a PDF report following the book's Chapter 12 deliverable structure.

The framework supports unlimited pivot depth — Kali to Host A to Host B to Host C and beyond. Every hop is tracked in a proxy chain map and all tool calls are automatically routed through the correct chain of port forwarding rules.

## Scripts

| Script | Phase | Description |
|--------|-------|-------------|
| `00_runAll` | Runner | Master script that executes the full chain in sequence with pass/fail gating. Accepts a CIDR as argument or prompts for one. |
| `00_cleanup` | Utility | Removes all local recon data (`./recon/` directory). Does NOT touch remote targets. |
| `01_recon` | Information Gathering | Host discovery via ICMP ping sweep, full TCP port scan (`-p-`), service/version detection with banner grabbing, MSSQL identification from banners (not just port 1433). |
| `02_bruteforce` | Focused Penetration | Dictionary-based credential brute-force against identified MSSQL instances. Configurable username and password wordlist paths. |
| `03_xpcmdshell` | Focused Penetration | Enables xp_cmdshell on compromised MSSQL instances, runs initial OS reconnaissance (hostname, ipconfig, local admins). |
| `04_hashdump` | Post-Exploitation | Extracts SAM and SYSTEM registry hives via anonymous SMB share, recovers NTLM password hashes with secretsdump. |
| `05_persist` | Post-Exploitation | Establishes reboot-persistent backdoor via scheduled task executing a PowerShell reverse shell on boot as SYSTEM. |
| `06_pivot` | Lateral Movement | Pass-the-Hash spraying, MSSQL credential reuse, network interface enumeration, recursive cross-network pivoting via netsh portproxy with multi-hop chaining. |
| `07_cleanup` | Cleanup | Reverts ALL remote artifacts across every compromised host — scheduled tasks, payload files, hive copies, SMB shares, registry modifications, portproxy rules, xp_cmdshell. Follows Chapter 11's five cleanup categories. |
| `08_report` | Reporting | Generates a PDF penetration test report following Chapter 12's eight-component structure. Dynamically built from all recon, loot, and pivot data. |

## Usage

### Full Chain

```bash
chmod +x 0*
./00_runAll 10.0.10.0/24
```

Or run each script individually:

```bash
./01_recon 10.0.10.0/24
./02_bruteforce
./03_xpcmdshell
./04_hashdump
./05_persist
./06_pivot
./07_cleanup
./08_report
```

### Prerequisites

- Kali Linux (tested on 2024.x)
- nmap
- Impacket (mssqlclient.py, secretsdump.py)
- NetExec (nxc)
- smbclient
- pandoc + texlive-xetex (08_report installs these automatically if missing)

## Configurable Values

Each script has clearly marked `# CONFIGURABLE` blocks at the top. Key values:

| Value | Script(s) | Default | Description |
|-------|-----------|---------|-------------|
| Wordlist paths | `02_bruteforce`, `06_pivot` | Metasploit common_users/passwords | Username and password files for MSSQL brute-force |
| `TASK_NAME` | `05_persist`, `06_pivot`, `07_cleanup` | `WindowsHealthCheck` | Scheduled task name — must match across all three scripts |
| `PROXY_BASE_PORT` | `06_pivot` | `33000` | Starting port for portproxy forwarding rules |
| `LPORT` | `05_persist`, `06_pivot` | `4444` | Callback port for reverse shell persistence |

## Cross-Network Pivoting

When 06_pivot discovers that a compromised host has a network interface on a segment Kali can't reach, it:

1. Runs a PowerShell-based host discovery and port scan FROM the compromised dual-homed host (since Kali has no route to the new segment)
2. Sets up `netsh interface portproxy` rules on the pivot host to forward traffic from high ports (33000+) to discovered services on the remote segment
3. Executes the full attack chain (brute-force, xp_cmdshell, hashdump, persistence, PtH) through the forwarded ports
4. Checks newly compromised remote hosts for additional NICs and recursively pivots deeper
5. Prompts the operator before pivoting into each new segment

For multi-hop scenarios (e.g., Kali → Host A → Host B → Host C), portproxy rules are chained through intermediate hosts automatically. The `PROXY_CHAIN_MAP` tracks how Kali reaches every compromised host and is persisted to `proxy-chain-map.txt` for use by 07_cleanup.

### Cleanup Ordering

07_cleanup follows a specific phase order to ensure remote hosts are cleaned before their tunnels are torn down:

1. **Phase 1:** Remote-segment MSSQL hosts (via portproxy — tunnels must be alive)
2. **Phase 2:** PtH hosts (remote PtH hosts also need tunnels alive)
3. **Phase 3:** Portproxy rule reset (safe now that all remote hosts are cleaned)
4. **Phase 4:** Same-segment MSSQL hosts (xp_cmdshell disabled last on each host)

## Output Structure

All data is stored under `./recon/`:

```
recon/
  scope.txt                              # Scanned CIDR(s)
  attacker-ip.txt                        # Kali IP for callbacks
  hosts/
    all-live-hosts.txt                   # Ping sweep results
    targets.txt                          # Windows hosts
    mssql-targets.txt                    # IP:PORT of MSSQL services
    mssql-creds.txt                      # IP:PORT:USER:PASS
    ping-sweep.gnmap                     # Raw nmap ping sweep
  ports/
    full-tcp.gnmap                       # Full port scan results
  services/
    full-service.nmap                    # Service/version scan
    service_*.nmap                       # Per-segment service scans (from pivots)
  mssql/
    <ip>_<port>_cmdshell.txt             # xp_cmdshell recon output
  loot/
    <ip>/
      hashes.txt                         # NTLM hashes from secretsdump
  persist/
    <ip>_persist.txt                     # Persistence deployment log with summary
  pivot/
    pth-successes.txt                    # HOST:USER:NTHASH for PtH wins
    credreuse-successes.txt              # Credential reuse results
    networks-discovered.txt              # New CIDRs found during pivoting
    portproxy-rules.txt                  # All netsh portproxy rules created
    proxy-chain-map.txt                  # HOST|CHAIN routing map for cleanup
    pivot-decisions.txt                  # PIVOTED/DECLINED per segment
    forward-map_*.txt                    # Per-segment port forwarding maps
    remote-scan_*.txt                    # Per-segment remote scan results
    <ip>_pth.txt                         # Recon from PtH'd hosts
    <ip>_mssql_pivot.txt                 # Recon from credential-reused hosts
  cleanup/
    <ip>_cleanup.log                     # Per-host MSSQL cleanup log
    <ip>_pth_cleanup.log                 # Per-host PtH cleanup log
    <ip>_portproxy_cleanup.log           # Per-host portproxy cleanup log
```

## Report

08_report generates a PDF following Chapter 12's eight-component structure:

1. **Executive Summary** — Goals, scope, dates, high-level results
2. **Engagement Methodology** — Attacker model, four-phase methodology
3. **Attack Narrative** — Chronological story of the engagement
4. **Technical Observations** — Up to 6 findings (MSSQL creds, xp_cmdshell, credential harvesting, persistence, PtH, credential reuse), each with severity, MITRE ATT&CK mapping, evidence, affected assets, and remediation
5. **Appendix A** — Severity definitions and full MITRE ATT&CK technique table
6. **Appendix B** — All discovered hosts and services (parsed from nmap output)
7. **Appendix C** — Tool list with URLs
8. **Appendix D** — Hardening references (NIST, CIS, Microsoft, MITRE, OWASP, SANS)

The report is entirely dynamic — sections are only included if that attack phase actually succeeded. Pivot narratives vary based on whether the operator chose to pivot, whether remote hosts were compromised, or whether the segments were skipped.

## MITRE ATT&CK Coverage

| ID | Technique | Script |
|----|-----------|--------|
| T1046 | Network Service Discovery | 01_recon |
| T1110.001 | Brute Force: Password Guessing | 02_bruteforce |
| T1059.001 | PowerShell | 03_xpcmdshell |
| T1505 | Server Software Component (xp_cmdshell) | 03_xpcmdshell |
| T1003.002 | OS Credential Dumping: SAM | 04_hashdump |
| T1053.005 | Scheduled Task | 05_persist |
| T1550.002 | Pass the Hash | 06_pivot |
| T1021.002 | SMB/Windows Admin Shares | 06_pivot |
| T1078 | Valid Accounts (credential reuse) | 06_pivot |
| T1090.001 | Internal Proxy (netsh portproxy) | 06_pivot |
| T1070 | Indicator Removal on Host | 07_cleanup |

## Lab Environment

Designed for the Capsulecorp Pentest lab — a Dragon Ball Z-themed Windows/Linux network used for RIS602 coursework. The framework is lab-use only and should not be used against systems without explicit written authorization.
