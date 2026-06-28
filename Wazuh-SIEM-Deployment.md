# Wazuh SIEM — Deployment Log

_Standing up the all-in-one Wazuh manager on the LAN segment and enrolling the first agent on the Active Directory domain controller — KVM/libvirt rebuild (fa-pc, 28 June 2026)._

| **Session goal** | Rebuild Wazuh (manager + indexer + dashboard) and deploy the DC agent on the new KVM lab |
| --- | --- |
| **Manager VM** | wazuh-manager — Ubuntu Server 22.04.5, 8 GB RAM / 4 vCPU / 48 GB disk |
| **Manager IP** | 192.168.10.12 (static) on the lan segment — reconciled from conflicting legacy docs |
| **Wazuh version** | 4.14 all-in-one (manager 4.14, agent 4.14.5) |
| **First agent** | WinSrv2025-AD (10.10.10.10, TARGET) — enrolled and Active |
| **Outcome** | Manager live, dashboard reachable from Kali, DC agent reporting — SIEM loop closed |

> **About this document**
> A build log for today's Wazuh deployment on the rebuilt KVM/libvirt host. The legacy project docs (Wazuh_Setup, Wazuh_Agent_Deployment) describe the old Hyper-V environment; this log supersedes them for the new setup, including the corrected static manager IP and the KVM-specific steps. pfSense segmentation, the AD domain, and Kali are unchanged from the [migration build log](Homelab-KVM-Rebuild.md).

## Contents

1. [Manager VM Creation](#1-manager-vm-creation)
2. [Static IP & DNS](#2-static-ip--dns)
3. [Wazuh Install & the LVM Disk Trap](#3-wazuh-install--the-lvm-disk-trap)
4. [Dashboard Verification](#4-dashboard-verification)
5. [Firewall Path for the DC Agent](#5-firewall-path-for-the-dc-agent)
6. [Getting the Agent Installer onto the DC](#6-getting-the-agent-installer-onto-the-dc)
7. [Agent Install — the cmd.exe Quoting Bug](#7-agent-install--the-cmdexe-quoting-bug)
8. [Enrollment Failure — Stale ossec.conf](#8-enrollment-failure--stale-ossecconf)
9. [Current State & Next Steps](#9-current-state--next-steps)

## 1. Manager VM Creation

The manager was built fresh on the lan libvirt network with a virtio NIC. Specs were bumped from the legacy doc's 6 GB to 8 GB / 4 vCPU — the all-in-one runs an OpenSearch indexer that is memory-hungry, and the host has 31 GB to spare.

```bash
virt-install \
  --name wazuh-manager \
  --memory 8192 --vcpus 4 --cpu host \
  --disk path=/var/lib/libvirt/images/wazuh-manager.qcow2,size=50,format=qcow2 \
  --network network=lan,model=virtio \
  --osinfo ubuntu22.04 \
  --cdrom /var/lib/libvirt/images/ubuntu-22.04.5-live-server-amd64.iso \
  --graphics spice --noautoconsole
```

Installed via the subiquity text installer with username **wazuh**, server name **wazuh-manager**, and OpenSSH enabled. Networking left on DHCP during install so the mirror step and apt would work (LAN → internet is permitted by the pfSense rule set).

## 2. Static IP & DNS

The interface came up as **enp1s0** (virtio naming, not eth0 as in the old Hyper-V docs) with a DHCP lease of .101. It was pinned to the static **192.168.10.12**. Because the Ubuntu installer hands networking to cloud-init, cloud-init's network management was disabled first to stop it regenerating the netplan file and stomping the static config on reboot.

```bash
# disable cloud-init networking
echo 'network: {config: disabled}' | sudo tee \
  /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
```

```yaml
# /etc/netplan/01-static.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp1s0:
      dhcp4: false
      addresses: [192.168.10.12/24]
      routes: [{to: default, via: 192.168.10.1}]
      nameservers: {addresses: [192.168.10.1]}
```

```bash
sudo chmod 600 /etc/netplan/01-static.yaml
sudo rm /etc/netplan/50-cloud-init.yaml
sudo netplan try   # auto-reverts in 120s if it breaks
```

> **netplan try, and the OVS warning**
> `netplan try` was used instead of `apply` so a bad config rolls back automatically rather than locking the box out. The `Cannot call Open vSwitch` warning it prints is harmless — netplan probes for OVS, doesn't find it (standard bridges, not OVS), and carries on. DNS verified end to end: ping to Pi-hole at 192.168.4.2 (one hop, ttl 63) and name resolution via the pfSense resolver both succeeded.

## 3. Wazuh Install & the LVM Disk Trap

The first run of the all-in-one installer reached the dashboard step and failed with `No space left on device`, then auto-rolled-back and removed the partial install. The cause was not the 50 GB disk — it was the Ubuntu Server installer's default LVM behaviour: it had sized the logical volume at only **24 GB**, leaving the other **24 GB** unallocated inside the volume group where the filesystem couldn't see it.

```bash
sudo vgs   # VFree showed 24.00g idle in the volume group
sudo lvs   # ubuntu-lv was only <24.00g

# claim the free space and grow the filesystem
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
# root now 48 GB / 36 GB free

# re-run with -o (overwrite) to clear any partial remnants
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh \
  && sudo bash ./wazuh-install.sh -a -o
```

> **Watch out — Ubuntu Server under-allocates the LVM volume by default**
> A fresh Ubuntu Server install commonly leaves ~half the disk unallocated in the volume group. For anything storage-heavy (Wazuh indexer, databases) extend the LV before installing, or the install fills the undersized volume and dies with a misleading 'disk full' even though the qcow2 has room.

> **Result — installation finished**
> Second run completed cleanly through the dashboard step. The installer printed the generated `admin` password at the end (recoverable later via the `wazuh-passwords-tool.sh` or the `wazuh-install-files.tar`). All three services confirmed `active (running)`; dashboard listening on 0.0.0.0:443. The red OpenSearch/Java `setSecurityManager` warnings at startup are normal noise, not errors.

## 4. Dashboard Verification

Logged into the dashboard from the Kali browser to confirm cross-network reachability. Both Kali and the manager sit on the lan segment, so this is intra-segment traffic and needs no firewall rule.

| **Setting** | **Value** |
| --- | --- |
| URL | https://192.168.10.12 (port 443) |
| User | admin |
| Cert | Self-signed — browser warning expected, proceed past it |
| Result | Login page loaded from Kali, authenticated successfully |

## 5. Firewall Path for the DC Agent

The DC is on TARGET (10.10.10.0/24) and the manager on LAN (192.168.10.0/24); TARGET → LAN is blocked by the intentional segmentation. A single narrow Pass rule on the **OPT1 (TARGET)** interface allows agent traffic to the manager. On inspection the rule already existed (from the pfSense rebuild) _twice_ — the duplicate sitting below the block rules was dead weight and was removed. Final OPT1 order:

| **#** | **Action** | **Source** | **Destination** | **Ports** | **Note** |
| --- | --- | --- | --- | --- | --- |
| 1 | Pass | OPT1 subnets | 192.168.10.12 | 1514–1515 | Wazuh agent → manager (effective — above blocks) |
| 2 | Block | OPT1 subnets | LAN subnets | * | Block TARGET → LAN |
| 3 | Block | OPT1 subnets | * | * | Block TARGET outbound |

> **Rule order is first-match-wins**
> pfSense evaluates top-down. The Pass rule only works because it sits above both block rules — the duplicate that had drifted below 'Block TARGET → LAN' was never being hit. `Test-NetConnection 192.168.10.12 -Port 1514/1515` from the DC later confirmed both ports `TcpTestSucceeded : True`, proving the path.

## 6. Getting the Agent Installer onto the DC

TARGET has no internet, so the agent MSI can't be downloaded on the DC directly. Rather than poke a temporary HTTP hole in the firewall (the legacy doc's approach), the MSI was downloaded on the host and attached to the DC as CD media — no firewall change.

```bash
# on the host (fa-pc)
cd /var/lib/libvirt/images
sudo wget https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.5-1.msi
```

> **Watch out — a raw .msi will not mount as CD media**
> Attaching the bare `.msi` as a virtual CD produced `The volume does not contain a recognized file system` in Windows — an MSI is not an ISO9660 image, so there's no filesystem to read. Wrap it in an ISO first: `sudo genisoimage -o wazuh-agent.iso -J -r wazuh-agent-4.14.5-1.msi`, then attach that. (Also: the virt-manager storage browser hides non-ISO files and needs the refresh icon after new files appear; Browse Local fails on the root-owned images dir with Permission denied.)

## 7. Agent Install — the cmd.exe Quoting Bug

The first install used mismatched quotes (`'192.168.10.12'` single, name double). In **cmd.exe** single quotes are literal characters, not string delimiters, so the manager address was written malformed and the silent install failed without registering the service. The corrected command uses no quotes (the values have no spaces) and adds verbose logging:

```bat
msiexec /i E:\wazuh-agent-4.14.5-1.msi /q ^
  WAZUH_MANAGER=192.168.10.12 ^
  WAZUH_AGENT_NAME=WinSrv2025-AD ^
  /l*v C:\wazuh-install.log

net start WazuhSvc
sc query WazuhSvc   # STATE : 4 RUNNING
```

## 8. Enrollment Failure — Stale ossec.conf

Despite the service running and the network path open, the dashboard showed _No agents added._ The agent log and config revealed the manager address still carried a stray leading quote from the very first attempt: `<address>'192.168.10.12</address>`. The key lesson:

> **Watch out — a re-run MSI install does NOT overwrite an existing ossec.conf**
> Because the agent was already present, the second (clean) install left the broken config from the first attempt in place. The agent was trying to reach a host literally named `'192.168.10.12`, which never resolves, so enrollment (port 1515) never completed. Fix the config directly, or fully uninstall before reinstalling.

The address was corrected directly in the config, then the service restarted. (Editing in Notepad is less error-prone than nested PowerShell find-and-replace for a one-character fix.)

```powershell
$conf = "C:\Program Files (x86)\ossec-agent\ossec.conf"
Stop-Service WazuhSvc
(Get-Content $conf) -replace "<address>'192.168.10.12", `
  "<address>192.168.10.12" | Set-Content $conf
Start-Service WazuhSvc

# verify: <address>192.168.10.12</address> (no quote, single closing bracket)
Select-String -Path $conf -Pattern "address"
```

> **Result — agent Active**
> After the address was clean and the service restarted, the agent enrolled within a minute. Wazuh Agents summary: **Active (1), Disconnected (0)** — WinSrv2025-AD reporting from 10.10.10.10. The full SIEM loop is now live: attacks from Kali against the DC generate Windows security events (4768 / 4769 / 4625 and friends) that flow into Wazuh.

## 9. Current State & Next Steps

| **Component** | **State** |
| --- | --- |
| Wazuh manager + indexer + dashboard | Live on 192.168.10.12, all services running |
| Dashboard | Reachable and authenticated from Kali (https://192.168.10.12) |
| Firewall (OPT1 → manager 1514/1515) | Single clean Pass rule above the block rules |
| Agent 001 — WinSrv2025-AD (DC) | Enrolled, Active, reporting |
| Agent 002 — Win11-Client01 | Pending — client VM not yet rebuilt |

### Recommended next actions

- **Rebuild Win11-Client01** and domain-join to lab.local, then deploy agent 002 via the same MSI process — now that the quoting and ossec.conf quirks are known. Gives a second endpoint for lateral-movement and credential-harvesting detection.
- **Generate signal.** Run an attack from Kali (AS-REP roast on sjones, Kerberoast on svc-backup, or a password spray) and confirm the events surface in the dashboard — validates detection end to end and surfaces any alert tuning needed.
- **Remaining rebuilds:** Metasploitable 2 on TARGET, SANDBOX VMs, and the WireGuard tunnel for the new 192.168.4.0/22 WAN subnet.

Generated as a build log for the Wazuh deployment on the KVM/libvirt rebuild. Commands reflect the exact steps run on fa-pc and the DC; adapt addresses and drive letters if the environment changes.
