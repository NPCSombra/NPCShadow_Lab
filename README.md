# Home Lab

A self-hosted virtual lab built from scratch in VirtualBox: an OPNsense firewall segmenting
three isolated networks, an Active Directory domain, a Kali attacker VM, and a Cowrie SSH
honeypot for capturing and reviewing simulated intrusion activity.

Built as a hands-on complement to CompTIA A+ and Network+, focused on real networking,
Windows Server administration, and basic offensive/defensive security — not just lab exercises,
but a working environment that gets broken, diagnosed, and fixed like a real one would.

## Topology

```
                         ┌─────────────────────────┐
                         │        OPNsense          │
                         │   (firewall / router)    │
                         │                           │
              WAN ───────┤  em0          em1   em2  ├──────┐
           (NAT/Internet) │               │     │    │      │
                         └───────┼─────┼───────┘
                                 │     │
                      ┌──────────┘     └──────────┐
                      │                            │
              LAN — 10.10.10.0/24         ATTACKNET — 10.10.20.0/24
              (domain network)             (isolated attack segment)
                      │                            │
        ┌─────────────┼─────────────┐    ┌─────────┼─────────┐
        │                           │    │                   │
 Windows Server 2022        Windows 11    Kali Linux     Ubuntu Server
   (AD Domain Controller)   (domain-joined (attacker VM)  (Cowrie honeypot)
    10.10.10.10              client)       DHCP            10.10.20.50
                              10.10.10.15
```

- **LAN** and **Attacknet** are firewalled from each other by default — the attacker segment
  cannot reach the domain network unless a rule is deliberately added for a specific exercise.
- **DHCP** on both internal segments is served by **Dnsmasq** (not Kea — see notes below).

## Components

| Component | Role |
|---|---|
| **OPNsense** | Firewall/router; segments the network into WAN, LAN, and Attacknet |
| **Windows Server 2022** | Active Directory domain controller, DNS |
| **Windows 11** | Domain-joined client |
| **Kali Linux** | Attacker VM — recon, scanning, exploitation practice |
| **Ubuntu Server + Cowrie** | SSH honeypot, isolated on its own segment |

## Skills demonstrated

- VirtualBox networking: NAT, Internal Networks, adapter configuration
- OPNsense firewall administration: interface assignment, DHCP (Kea and Dnsmasq), firewall
  rules, NAT, packet capture / traffic analysis
- Windows Server 2022: Active Directory Domain Services, DNS, domain join
- Linux administration: Ubuntu Server, static networking via netplan, Python virtual
  environments, systemd-style service management
- Security tooling: Kali Linux recon basics, Cowrie honeypot deployment and log analysis
- Methodical troubleshooting: packet captures, log correlation, isolating variables across
  multiple network segments and services

## Docs

- [Cowrie Honeypot Setup Guide](Cowrie_Honeypot_Setup_Guide.pdf) — full build steps, day-to-day
  start/stop/login commands, and a "lessons learned" section covering real issues hit during
  the build (case-sensitive network names, DHCP conflicts, DNS routing, etc.)

## Notable issues solved along the way

- Diagnosed a completely silent network failure down to a **case-sensitive mismatch** in a
  VirtualBox Internal Network name — two VMs that looked identically configured were actually
  on two different virtual switches.
- Traced a VM settings dialog that silently refused to save **any** change, caused by an
  unrelated invalid Display setting blocking the whole dialog.
- Resolved a DHCP service conflict (Kea vs. Dnsmasq both bound to the same interface) causing
  intermittent connectivity loss.
- Fixed a DNS resolution failure caused by a port conflict between OPNsense's Unbound resolver
  and Dnsmasq's DNS component.
- Isolated a "DNS works, nothing else does" failure down to the firewall's **WAN interface
  landing on the wrong internal subnet**, re-pointing it correctly.

## Example Exercise: Metasploit vs. Cowrie

A quick case study showing the honeypot and the attacker VM actually interacting, with the
full picture visible from both sides.

**From Kali**, used Metasploit's SSH credential scanner against the honeypot:

```
msf6 > use auxiliary/scanner/ssh/ssh_login
msf6 auxiliary(scanner/ssh/ssh_login) > set RHOSTS 10.10.20.50
msf6 auxiliary(scanner/ssh/ssh_login) > set RPORT 2222
msf6 auxiliary(scanner/ssh/ssh_login) > set USERNAME root
msf6 auxiliary(scanner/ssh/ssh_login) > set PASSWORD toor
msf6 auxiliary(scanner/ssh/ssh_login) > run

[*] 10.10.20.50:2222      - Starting bruteforce
[+] 10.10.20.50:2222      - Success: 'root:toor' ''
[-] 10.10.20.50:2222      - Failed to setup the session - Net::SSH::Exception Unknown platform: unknown
[*] Scanned 1 of 1 hosts (100% complete)
```

The credential scan succeeded, but Metasploit couldn't turn it into a usable interactive
session — a small, interesting tell that it had hit a honeypot rather than a real host.

**On the Cowrie side**, the log shows exactly why:

```
Command found: help
Command not found: ?
exitCode: 127
remote close
Closing TTY Log
Timeout reached in HoneyPotSSHTransport
connection lost
```

After authenticating, Metasploit probes the shell to fingerprint the OS (`help`, then `?`).
Cowrie only emulates a specific set of real Linux commands, so the unrecognized probe
returned "command not found" — which is exactly why Metasploit's platform detection failed
and reported "Unknown platform." Full session visibility on both ends: the attacker's tool
output and the honeypot's TTY-level record of every command it tried.

## Status

Actively maintained —  a basic SIEM/log pipeline, and documented attack scenarios run from Kali against the domain network.
