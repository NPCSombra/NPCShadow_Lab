# NPCShadow_Lab   

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

## Status

Actively maintained — next planned additions: Metasploitable target on Attacknet, a basic
SIEM/log pipeline, and documented attack scenarios run from Kali against the domain network.
