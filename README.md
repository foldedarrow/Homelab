# Cybersecurity Homelab

A self-built, segmented cybersecurity lab for hands-on practice with Active Directory attacks, network defense, and SIEM monitoring. This repo documents the lab's design and its full rebuild on a KVM/QEMU + libvirt hypervisor.

The lab is intentionally vulnerable by design: a Windows Server 2025 domain (`lab.local`) seeded with realistic misconfigurations, fronted by a pfSense firewall that segments the network into isolated zones, with Kali Linux as the attack platform.

## Build writeups

📄 **[Homelab-KVM-Rebuild.md](Homelab-KVM-Rebuild.md)** — the complete, step-by-step build log: host preparation, virtual networking, pfSense, Kali, and the Windows Server 2025 domain controller, including the gotchas hit along the way and how they were resolved.

📄 **[Wazuh-SIEM-Deployment.md](Wazuh-SIEM-Deployment.md)** — standing up the all-in-one Wazuh manager on the LAN segment and enrolling the first agent on the domain controller: manager VM, static networking, the LVM disk trap, the firewall path, and the agent quoting/`ossec.conf` quirks that broke enrollment.

## Architecture at a glance

| Layer | Detail |
| --- | --- |
| **Host** | i9 desktop, Ubuntu + NetworkManager, KVM/QEMU via libvirt |
| **Firewall / router** | pfSense CE — routing, DHCP, DNS, and the segmentation firewall |
| **Attack machine** | Kali Linux on the management LAN |
| **Target domain** | Windows Server 2025 DC hosting the vulnerable `lab.local` Active Directory domain |
| **SIEM** | Wazuh 4.14 all-in-one (manager + indexer + dashboard) on the management LAN |
| **DNS filtering** | Pi-hole upstream of the lab |

### Network segmentation

| Segment | Subnet | Role |
| --- | --- | --- |
| WAN | 192.168.4.0/22 | Bridged to the home network via `br0` |
| LAN | 192.168.10.0/24 | Management + attack (Kali) + Wazuh manager |
| TARGET | 10.10.10.0/24 | Isolated victim domain (AD) |
| SANDBOX | 10.30.30.0/24 | Fully air-gapped detonation zone |

pfSense default-denies between segments: TARGET cannot reach the management LAN or the internet, and SANDBOX is completely isolated.

## Skills demonstrated

- **Virtualization & infrastructure** — KVM/QEMU, libvirt, virtual networking (isolated L2 networks, Linux bridging with `nmcli`/netplan), UEFI + TPM 2.0 guests
- **Network security** — firewall rule design, network segmentation, DNS architecture, default-deny policy
- **Active Directory** — forest/domain setup, OU and user design, and deliberate misconfigurations (Kerberoasting, AS-REP roasting, overprivileged accounts, weak password policy) for offensive practice
- **Offensive security** — using Kali to validate the attack surface (e.g. `nmap` against the DC's Kerberos/LDAP/SMB ports)
- **Detection & monitoring** — Wazuh SIEM deployment, agent enrollment across segmented networks, and Windows security-event collection from the domain controller
- **Documentation** — reproducible, command-level build logs

## Intentional vulnerabilities

> ⚠️ This domain is **deliberately insecure** for training purposes. Weak passwords, missing pre-authentication, overprivileged accounts, and a permissive password policy are all by design. Nothing here represents production configuration.

## Status

The core lab (host, networking, pfSense, Kali, DC) is rebuilt and verified, and the Wazuh SIEM is now live with the domain controller reporting as its first agent. Still pending: a domain-joined Windows 11 client (and its agent), Metasploitable, and WireGuard remote access. See the [homelab writeup](Homelab-KVM-Rebuild.md#6-current-state--next-steps) and the [Wazuh writeup](Wazuh-SIEM-Deployment.md#9-current-state--next-steps) for the current state and next actions.
