# Cybersecurity Homelab — Full Rebuild on KVM / libvirt

_Hyper-V (laptop) → KVM/QEMU + libvirt (i9 desktop, Ubuntu) — complete build log, start to finish._

| **Host** | i9 desktop (hostname fa-pc, user foldedarrow), Ubuntu with NetworkManager |
| --- | --- |
| **Hypervisor** | KVM / QEMU via libvirt — replaces Hyper-V |
| **Host RAM** | 31 GB (down from 64 GB on the retired laptop) |
| **Host NIC** | enp3s0 (wired) → bridged as br0 |
| **Home network** | 192.168.4.0/22 — gateway 192.168.4.1 — Pi-hole at 192.168.4.2 |
| **Built this session** | pfSense, Kali, Windows Server 2025 DC (lab.local) |
| **Not yet rebuilt** | Wazuh SIEM + agents, Win11 client, Metasploitable, SANDBOX VMs |
| **Data migrated** | None — clean rebuild from documentation |

> **About this document**
> This is a complete record of rebuilding the homelab from scratch on the new i9 host. The old Hyper-V lab on the ThinkPad is fully decommissioned and gone; this build is now the only homelab. The lab topology (segmented subnets, the intentionally-vulnerable AD domain) is unchanged in design — only the hypervisor and its virtual networking are new, expressed in KVM/libvirt terms.

## Contents

1. [Host Preparation](#1-host-preparation)
2. [Virtual Networking](#2-virtual-networking)
3. [pfSense Firewall / Router](#3-pfsense-firewall--router)
4. [Kali Linux (Attack Machine)](#4-kali-linux-attack-machine)
5. [Windows Server 2025 Domain Controller](#5-windows-server-2025-domain-controller)
6. [Current State & Next Steps](#6-current-state--next-steps)

## 1. Host Preparation

The new host is an i9 desktop already running Ubuntu, which we kept as the host OS rather than going bare-metal. KVM/QEMU with libvirt is the natural in-place replacement for Hyper-V. Before installing anything, the hardware was verified.

### 1.1 Hardware & virtualization check

```bash
egrep -c '(vmx|svm)' /proc/cpuinfo     # -> 64  (VT-x on all logical cores)
free -h                                # -> 31Gi total
ip -br link                            # -> enp3s0 UP, wired (d8:43:ae:6b:75:aa)
```

> **Capacity note — 31 GB, not 64 GB**
> The retired laptop had 64 GB; this host has 31 GB, with ~8 GB already used by the desktop. The full lab footprint is ~20 GB of VM allocations, so running everything at once would be uncomfortably tight (~29/31 GB). Working habit going forward: run VMs in scenario groups. AD attack practice needs pfSense + DC + Win11 + Kali + Wazuh; Metasploitable and SANDBOX can stay powered off during that work.

### 1.2 Virtualization stack

Installed the KVM/QEMU + libvirt toolchain, plus swtpm and ovmf (needed later for the Windows guests), and added the user to the libvirt and kvm groups so VMs can be managed without sudo.

```bash
sudo apt update
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients \
                    virtinst virt-manager bridge-utils swtpm ovmf cpu-checker
sudo usermod -aG libvirt,kvm $USER     # then log out / back in

kvm-ok            # -> 'KVM acceleration can be used'
virsh list --all  # -> empty table, no permission error
```

**ovmf** supplies UEFI firmware for the Generation-2 Windows guests; **swtpm** provides the emulated TPM 2.0 that Windows requires. The **virtio-win** ISO (downloaded later) supplies storage/network drivers during the Windows install.

## 2. Virtual Networking

The Hyper-V vSwitches were re-created as libvirt constructs. The three internal segments become isolated layer-2 networks; the WAN segment becomes a Linux bridge on the physical NIC.

### 2.1 vSwitch → libvirt mapping

| **Hyper-V vSwitch** | **Segment** | **libvirt network** | **Bridge** |
| --- | --- | --- | --- |
| vSwitch-LAN | 192.168.10.0/24 | lan (isolated L2) | virbr-lan |
| vSwitch-TARGET | 10.10.10.0/24 | target (isolated L2) | virbr-target |
| vSwitch-SANDBOX | 10.30.30.0/24 | sandbox (isolated L2) | virbr-sandbox |
| vSwitch-WAN | 192.168.4.0/22 | wan (bridge to br0) | br0 |

### 2.2 Isolated segments (LAN / TARGET / SANDBOX)

Each internal network is defined with **no `<ip>` block and no `<forward>` element** — a bare L2 switch. libvirt runs no DHCP, no NAT, and gives the host no address on these segments. pfSense remains the sole gateway and DHCP server for each, which is what enforces the segmentation.

```bash
cat > /tmp/target.xml <<'EOF'
<network>
  <name>target</name>
  <bridge name='virbr-target' stp='on' delay='0'/>
</network>
EOF

for net in lan target sandbox; do
  sudo virsh net-define /tmp/$net.xml
  sudo virsh net-autostart $net
  sudo virsh net-start $net
done
```

> **Gotcha — system vs session libvirt**
> After defining the networks with sudo virsh (which targets qemu:///system), a plain virsh net-list showed an empty table because it defaulted to the per-user qemu:///session instance — a separate, empty world. Fixed permanently by pinning the default URI so plain virsh and virt-manager always target the system instance:

```bash
echo 'export LIBVIRT_DEFAULT_URI=qemu:///system' >> ~/.bashrc
source ~/.bashrc
virsh net-list --all   # -> default, lan, sandbox, target  (active, autostart)
```

### 2.3 WAN bridge (br0)

The WAN segment is the only part that touches the host's real NIC. pfSense's WAN attaches to a Linux bridge (br0) over enp3s0, landing the firewall directly on the home network. A bridge is used deliberately — libvirt NAT would double-NAT the whole lab.

**Resolving a dual-netplan conflict**

Two netplan files claimed enp3s0 under different renderers: 01-network-manager-all.yaml (NetworkManager) and a stray cloud-init file 50-cloud-init.yaml (defaulting to networkd). Layering a bridge onto that ambiguity risks a lockout, so it was consolidated to a single renderer first, with cloud-init told to stop managing the network.

```bash
sudo cp /etc/netplan/50-cloud-init.yaml ~/50-cloud-init.yaml.bak   # backup

sudo bash -c 'cat > /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg' <<'EOF'
network: {config: disabled}
EOF

sudo rm /etc/netplan/50-cloud-init.yaml
sudo netplan try        # self-reverts after 120s unless confirmed
sudo chmod 600 /etc/netplan/01-network-manager-all.yaml   # silence perms warning
```

> **Home network changed during the rebuild**
> The live host came up on 192.168.4.0/22 (gateway 192.168.4.1), not the 192.168.8.0/24 recorded in the old docs — a different home router is now in use, with Pi-hole at 192.168.4.2. This only affects the WAN edge: pfSense's WAN now pulls an address in 192.168.4.0/22, and the old .140 reservation / WireGuard inbound no longer apply. The isolated LAN / TARGET / SANDBOX segments are unaffected.

**Building br0 via NetworkManager**

Because NetworkManager owns the NIC, the bridge was built with nmcli, which applies the slave swap atomically. The old standalone profile is deleted only after br0 is confirmed up, so the NIC cannot be orphaned.

```bash
nmcli connection add type bridge ifname br0 con-name br0 ipv4.method auto
nmcli connection add type ethernet ifname enp3s0 master br0 \
      con-name br0-slave-enp3s0
nmcli connection up br0 && nmcli connection delete "Wired connection 1"

ip -br addr show br0      # -> br0 UP 192.168.5.22/22
ping -c 3 192.168.4.1    # -> 0% packet loss
```

> **Result — WAN bridge live**
> br0 holds the DHCP lease (192.168.5.22/22), enp3s0 is enslaved as br0-slave-enp3s0, the old profile is gone, and the gateway pings cleanly. Tailscale and libvirt's default virbr0 rode through untouched. The host is bridge-ready for pfSense's WAN.

The thin wan network that attaches VMs to br0 was created later (it was initially missed, causing the first pfSense build to fail with “no network with matching name 'wan'”):

```bash
cat > /tmp/wan.xml <<'EOF'
<network>
  <name>wan</name>
  <forward mode='bridge'/>
  <bridge name='br0'/>
</network>
EOF
virsh net-define /tmp/wan.xml && virsh net-autostart wan && virsh net-start wan
```

## 3. pfSense Firewall / Router

pfSense CE was installed fresh and configured by hand from the project documentation (the old config.xml was not migrated). It provides routing, DHCP, DNS, and the segmentation firewall for all four networks.

### 3.1 Create the VM

Four virtio NICs with pinned MACs make interface assignment unambiguous later. pfSense boots on legacy BIOS — no UEFI/TPM needed (unlike the Windows guests).

```bash
virt-install \
  --name pfSense \
  --memory 2048 --vcpus 2 --cpu host \
  --os-variant freebsd14.0 \
  --disk path=/var/lib/libvirt/images/pfSense.qcow2,size=20,format=qcow2,bus=virtio \
  --cdrom /var/lib/libvirt/images/pfSense-CE.iso \
  --network network=wan,model=virtio,mac=52:54:00:b0:00:01 \
  --network network=lan,model=virtio,mac=52:54:00:b0:00:02 \
  --network network=target,model=virtio,mac=52:54:00:b0:00:03 \
  --network network=sandbox,model=virtio,mac=52:54:00:b0:00:04 \
  --graphics spice --video qxl --noautoconsole
```

### 3.2 Interface assignment (by MAC)

On first boot pfSense drops into interface assignment (VLANs: n). Interfaces are matched by MAC, not by vtnet number:

| **pfSense role** | **MAC** | **vtnet** | **Network** |
| --- | --- | --- | --- |
| WAN | 52:54:00:b0:00:01 | vtnet0 | wan / br0 |
| LAN | 52:54:00:b0:00:02 | vtnet1 | lan |
| OPT1 | 52:54:00:b0:00:03 | vtnet2 | target |
| OPT2 | 52:54:00:b0:00:04 | vtnet3 | sandbox |

### 3.3 Interface addressing & DHCP

Set via the console (option 2). Each interface gets its gateway address and a DHCP pool of .100–.199, leaving low addresses free for statics.

| **Interface** | **IP address** | **DHCP pool** |
| --- | --- | --- |
| WAN (vtnet0) | 192.168.5.23/22 (DHCP from home network) | — |
| LAN (vtnet1) | 192.168.10.1/24 | 192.168.10.100–199 |
| OPT1 / TARGET (vtnet2) | 10.10.10.1/24 | 10.10.10.100–199 |
| OPT2 / SANDBOX (vtnet3) | 10.30.30.1/24 | 10.30.30.100–199 |

### 3.4 Setup wizard & DNS

The WebGUI (https://192.168.10.1) was reached from Kali once it was built. Key wizard choices:

| **Setting** | **Value** |
| --- | --- |
| Hostname / domain | pfSense / home.arpa (NOT lab.local — that belongs to the DC) |
| Primary DNS | 192.168.4.2 (Pi-hole) |
| Override DNS by WAN DHCP | Unticked — use Pi-hole, not the home router's servers |
| WAN | DHCP; Block private networks + Block bogon both UNTICKED |
| DNS Resolver | Forwarding mode enabled → forwards to Pi-hole |

> **DNS routing design**
> LAN clients (Kali) resolve via pfSense → Pi-hole → internet, so lab DNS is filtered and visible in Pi-hole's query log. TARGET domain machines are the exception: they must use the DC (10.10.10.10) for DNS so they can resolve lab.local and AD SRV records — set per-scope in TARGET's DHCP, independent of Pi-hole. Note pfSense's resolver defaults to recursion and ignores upstream DNS until forwarding mode is enabled.

### 3.5 Firewall rules

The segmentation matrix. pfSense default-denies anything not explicitly passed, so SANDBOX needs no rules at all to be air-gapped, and TARGET needs only the narrow Wazuh exception above its blocks. Rules evaluate top-down, so order matters on TARGET.

| **Interface** | **Order** | **Rule** | **Action** | **Purpose** |
| --- | --- | --- | --- | --- |
| LAN | — | LAN net → any | Pass | Kali/Wazuh reach TARGET + internet |
| OPT1 / TARGET | 1 | TARGET net → 192.168.10.12 : 1514,1515 | Pass | Wazuh agents → manager (only exception) |
| OPT1 / TARGET | 2 | TARGET net → LAN net | Block | No lateral movement to management net |
| OPT1 / TARGET | 3 | TARGET net → any | Block | Victims cannot reach internet / call home |
| OPT2 / SANDBOX | — | (no rules) | Default deny | Fully air-gapped |

> **Verified**
> From Kali: nmap -p 88,389,445 10.10.10.10 returned all three open (kerberos-sec, ldap, microsoft-ds), proving LAN→TARGET attack traffic passes and the DC is reachable, while TARGET remains unable to initiate back to LAN or the internet.

## 4. Kali Linux (Attack Machine)

Kali is the LAN management and attack box. It was installed from the 2026.1 installer ISO and attached to the lan network, where it validates the LAN path by pulling a pfSense DHCP lease.

```bash
virt-install \
  --name kali \
  --memory 4096 --vcpus 2 --cpu host \
  --os-variant debiantesting \
  --disk path=/var/lib/libvirt/images/kali.qcow2,size=40,format=qcow2,bus=virtio \
  --cdrom /var/lib/libvirt/images/kali.iso \
  --network network=lan,model=virtio,mac=52:54:00:b0:10:01 \
  --graphics spice --video qxl --channel spicevmc --noautoconsole
```

Notes: os-variant debiantesting is used because the 2026.1 release is newer than the host's osinfo-db. SPICE + qxl gives a proper accelerated desktop console (the old Hyper-V Enhanced Session Mode problems don't apply on KVM).

> **LAN validated end to end**
> After install, ip a on Kali showed eth0 (MAC 52:54:00:b0:10:01) holding 192.168.10.100/24 marked dynamic — the first address from pfSense's LAN pool. That single lease proved the lan libvirt network, pfSense's LAN interface, and its DHCP server all working together. apt update / full-upgrade then confirmed outbound internet and Pi-hole DNS via pfSense.

Recommended follow-up: pin Kali to its documented static **192.168.10.50** via a pfSense DHCP reservation so it stops floating in the pool.

## 5. Windows Server 2025 Domain Controller

The DC anchors the TARGET subnet and hosts the intentionally-vulnerable lab.local domain. It was activated on the WAN first (needs internet), then moved to the isolated TARGET segment and promoted.

### 5.1 Create the VM (UEFI + TPM)

Unlike pfSense and Kali, the Gen-2 Windows guest needs UEFI firmware (OVMF) and an emulated TPM 2.0 (swtpm), plus the virtio-win ISO for storage/network drivers during install. It was built on the wan network for activation.

```bash
virt-install \
  --name WinSrv2025-AD \
  --memory 4096 --vcpus 2 --cpu host \
  --os-variant win2k22 \
  --boot uefi \
  --tpm backend.type=emulator,backend.version=2.0,model=tpm-crb \
  --disk path=/var/lib/libvirt/images/winsrv2025.qcow2,size=60,format=qcow2,bus=virtio \
  --disk path=/var/lib/libvirt/images/virtio-win.iso,device=cdrom \
  --cdrom /var/lib/libvirt/images/winserver2025.iso \
  --network network=wan,model=virtio,mac=52:54:00:b0:00:10 \
  --graphics spice --video qxl --channel spicevmc --noautoconsole
```

> **Three install-time gotchas**
> - The Windows ISO must be passed as `--cdrom` (the install method), not as a `--disk`, or virt-install errors.
> - The TPM model is `tpm-crb` — `tpm-crd` is rejected by libvirt.
> - At "Where do you want to install Windows?" the disk is invisible until the virtio storage driver is loaded from the virtio-win CD (`vioscsi\2k22\amd64`). Likewise the NIC shows no network until the NetKVM driver is installed from Device Manager after first boot.

### 5.2 Activate, then move to TARGET

With the virtio NIC driver installed, Windows pulled internet over the WAN and was activated (Settings → System → Activation). It was then shut down and its NIC moved from wan to target, reusing the same MAC:

```bash
virsh detach-interface WinSrv2025-AD network --mac 52:54:00:b0:00:10 --config
virsh attach-interface WinSrv2025-AD network target --model virtio \
      --mac 52:54:00:b0:00:10 --config
virsh start WinSrv2025-AD
```

On TARGET it received a DHCP lease (10.10.10.100), confirming TARGET DHCP works, then was set static to the DC address. DNS points at itself (127.0.0.1) ahead of the AD DNS role — the “DNS server incorrect” warning at this stage is expected.

```powershell
netsh interface ip set address "Ethernet" static 10.10.10.10 255.255.255.0 10.10.10.1
netsh interface ip set dns "Ethernet" static 127.0.0.1
```

### 5.3 Promote to Domain Controller

AD DS installed and a new forest created. Two DNS delegation warnings during promotion are expected and harmless (no parent zone above lab.local). The server auto-reboots into domain mode.

```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

Install-ADDSForest -DomainName "lab.local" -DomainNetbiosName "LAB" \
  -InstallDns:$true \
  -SafeModeAdministratorPassword (ConvertTo-SecureString "P@ssw0rd123!" -AsPlainText -Force) \
  -Force:$true
```

### 5.4 OUs and users

A corporate OU structure under Corp (IT, HR, Finance, Managers, Workstations, ServiceAccounts) and five users were created. All users share the weak password Password123! for spraying practice.

| **Name** | **Username** | **Dept / OU** | **Note** |
| --- | --- | --- | --- |
| John Smith | jsmith | IT | Standard user |
| Tom Wilson | twilson | IT | Standard user |
| Sarah Jones | sjones | HR | AS-REP roastable (misconfig) |
| Mike Chen | mchen | Finance | Standard user |
| Emma Davis | edavis | Managers | Domain Admin (overprivileged) |

### 5.5 Intentional misconfigurations

The four deliberate weaknesses that make the domain a realistic attack target were recreated:

| **#** | **Misconfiguration** | **Detail** | **Attack** |
| --- | --- | --- | --- |
| 1 | Kerberoastable svc-backup | SPN HTTP/backup.lab.local; pwd Summer2024! | Kerberoasting |
| 2 | AS-REP roastable sjones | Pre-auth not required | AS-REP roasting |
| 3 | Overprivileged edavis | Manager added to Domain Admins | BloodHound path to DA |
| 4 | Weak password policy | 4-char min, no complexity, no lockout, no expiry | Password spraying |

The SPN and password policy were set via PowerShell; the other two via the GUI:

```powershell
setspn -A HTTP/backup.lab.local svc-backup   # svc-backup must exist first

Set-ADDefaultDomainPasswordPolicy -Identity lab.local -ComplexityEnabled $false \
  -MinPasswordLength 4 -PasswordHistoryCount 0 -MaxPasswordAge 0 -LockoutThreshold 0
```

> **Typo watch**
> The password-policy command uses -Identity (not -Identify), and copy-paste can substitute an en-dash for a hyphen — both cause parameter-binding errors. Retype the dashes if a paste throws a strange error.

## 6. Current State & Next Steps

The core lab is rebuilt and verified: pfSense segments four networks, Kali is live on LAN with internet, and the DC anchors a fully attackable domain on the isolated TARGET subnet.

| **Component** | **State** |
| --- | --- |
| Host (KVM/libvirt) | Complete — stack installed, default URI pinned to system |
| Virtual networks | Complete — wan (br0) + isolated lan/target/sandbox |
| pfSense | Complete — routing, DHCP, DNS (Pi-hole forwarding), firewall rules |
| Kali | Complete — 192.168.10.x on LAN, updated, internet OK |
| Windows Server 2025 DC | Complete — lab.local, OUs, users, 4 misconfigs |
| Wazuh SIEM + agents | Pending |
| Win11 client (domain-joined) | Pending |
| Metasploitable / SANDBOX VMs | Pending |
| WireGuard external access | Pending (rebuild for 192.168.4.0/22) |

### Recommended next actions

- Pin Kali static to 192.168.10.50 via pfSense DHCP reservation.
- Rebuild Wazuh manager (192.168.10.12) and redeploy agents to the DC and Win11 client.
- Build the Win11 client on TARGET, join to lab.local (DNS → 10.10.10.10).
- Add Metasploitable 2 — on KVM use an emulated e1000 NIC (no Legacy Adapter workaround needed).
- Rebuild WireGuard on pfSense for remote Kali access, updated for the 192.168.4.0/22 WAN.
- (Optional) Pin br0 to a static host address outside the router's DHCP pool.

Build log generated from the rebuild session. The old Hyper-V homelab on the ThinkPad is fully decommissioned; this KVM/libvirt build is the current and only homelab.
