# Linux VMs with QEMU/KVM Manual

## Overview

This manual condenses my learning path for running virtual machines natively on Linux with **QEMU/KVM, libvirt and virt-manager**, instead of the VirtualBox workflow I used on Windows. It covers a desktop install, a headless Ubuntu Server install managed remotely over SSH, first-VM defaults, and the pitfalls hit along the way.

The full, detailed write-up lives in its own repository: [A-DW/Linux_VMs_with_QEMU-KVM](https://github.com/A-DW/Linux_VMs_with_QEMU-KVM) (`README.md`, `ServerEnvironmentInstallation.md`, `Concepts-FAQ.md`). This file is the quick-reference version.

---

## The stack

On Linux, "running a VM" is a layered set of tools rather than a single `.exe`:

```text
virt-manager / virsh   → interface (GUI / CLI)
        ↓
libvirt                → manages VMs, networks and storage
        ↓
QEMU                   → provides the virtual machine and virtual hardware
        ↓
KVM                    → kernel module exposing hardware virtualization to QEMU
        ↓
Intel VT-x / AMD-V     → CPU virtualization extensions
```

| Component | Role | Use directly? |
|-----------|------|---------------|
| KVM | Linux kernel facility exposing CPU virtualization | No, QEMU uses it |
| QEMU | Runs the VM and supplies virtual CPU, memory, disk, network | Rarely, only for specialist setups |
| libvirt | Management API/service for VMs, networks, storage | Indirectly, via a frontend |
| virt-manager | Graphical frontend for libvirt (local or remote) | Yes, primary desktop app |
| virsh | libvirt command-line manager | Yes, after learning the GUI |
| virt-install | CLI VM-creation tool | Yes, for repeatability and scripting |
| GNOME Boxes | Simplified GUI over the same backend | Optional, for disposable test VMs |
| Proxmox VE | Dedicated virtualization platform (its own distro) | Not for a desktop workstation |

QEMU/KVM was chosen because KVM is part of the kernel, and VirtualBox on Linux felt heavier on resources.

---

## Why this approach was chosen

The libvirt-managed stack keeps GUI and CLI working against the **same** VMs, so learning `virt-manager` and `virsh` reinforces each other. It avoids hand-writing large `qemu-system-x86_64` command lines and avoids a separate user-session setup.

The trade-off is a longer initial setup (packages, group membership, networking concepts) compared to VirtualBox's install-and-run experience.

---

## Exact procedure: desktop

### 1. Install the packages

```bash
sudo apt update
sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients virt-manager virtinst ovmf swtpm swtpm-tools
```

`ovmf` provides UEFI firmware for guests; `swtpm` provides an emulated TPM (needed for Windows 11).

### 2. Add the user to the `libvirt` group

```bash
sudo usermod -aG libvirt "$USER"
```

Log out completely and back in for the group to apply.

> **Security note:** `libvirt` group membership is effectively administrative (near-root). A member can define arbitrary system VMs that may expose host disks and devices. Only add trusted accounts. The `kvm` group is not needed for VMs managed via `qemu:///system`.

### 3. Validate the host

```bash
virt-host-validate qemu
virsh --connect qemu:///system list --all
```

If validation fails, confirm VT-x/AMD-V (SVM) is enabled in UEFI/BIOS and that `/dev/kvm` exists.

### 4. Launch virt-manager

```bash
virt-manager
```

Use the **QEMU/KVM system connection** (`qemu:///system`), the conventional choice for server-like guests, autostart and managed networking.

### 5. First VM defaults

| Setting | Value |
|---------|-------|
| Connection | QEMU/KVM system |
| Source | Local ISO |
| Firmware | UEFI |
| Disk | QCOW2, dynamically allocated |
| Disk bus | VirtIO or VirtIO-SCSI |
| Network | `default` virtual network (NAT), VirtIO model |
| Display | SPICE |
| CPU | Default first, `host-passthrough` later |

NAT is the right first choice: the VM gets outbound access behind a libvirt-managed network. Use a Linux bridge only when a guest must appear directly on the physical LAN.

### 6. Guest tools

Ubuntu guests:

```bash
sudo apt update
sudo apt install qemu-guest-agent spice-vdagent
sudo systemctl enable --now qemu-guest-agent
```

Windows guests need the **virtio-win** drivers (not shipped with Windows). Attach both the Windows ISO and `virtio-win.iso`, use UEFI plus a TPM 2.0 for Windows 11, load the storage driver from the VirtIO ISO if Setup cannot see the disk, then install the VirtIO guest tools and QEMU guest agent.

---

## Exact procedure: headless Ubuntu Server

On a server the virtualization stack stays the same; only **where the graphical interface runs** changes.

```text
Your workstation                 Ubuntu Server
├── virt-manager   ── qemu+ssh ─► ├── libvirt
└── SSH client                   ├── QEMU / KVM
                                 ├── VM storage
                                 └── Virtual networks
```

### 1. Check hardware support

```bash
sudo apt update
sudo apt install cpu-checker
sudo kvm-ok
lscpu | grep -E 'Virtualization|Hypervisor'
```

If the server is itself a VM, the outer hypervisor must expose nested virtualization, otherwise `/dev/kvm` is missing.

### 2. Install the server stack

```bash
sudo apt install qemu-kvm qemu-utils libvirt-daemon-system libvirt-clients virtinst ovmf swtpm swtpm-tools
sudo adduser "$USER" libvirt
```

Reconnect SSH afterwards. `virt-manager` is **not** installed on the server.

### 3. Verify

```bash
id
virt-host-validate qemu
virsh -c qemu:///system list --all
ls -l /dev/kvm
virsh -c qemu:///system net-list --all
```

An empty VM list is normal. The `default` network should be active with autostart; if not:

```bash
virsh -c qemu:///system net-start default
virsh -c qemu:///system net-autostart default
```

### 4. Manage remotely

On the workstation:

```bash
sudo apt install virt-manager
ssh-keygen -t ed25519
ssh-copy-id youruser@server-ip
virsh -c qemu+ssh://youruser@server-ip/system list --all
virt-manager -c qemu+ssh://youruser@server-ip/system
```

Once key login works, consider disabling password logins (`PasswordAuthentication no`), testing a second session before closing the first to avoid lockout.

| Method | Runs on | Best use |
|--------|---------|----------|
| virt-manager over SSH | Linux workstation | Best overall GUI |
| Cockpit Machines | Web service on the server | Browser-based management |
| virsh + virt-install | SSH terminal | Learning, automation, troubleshooting |

### 5. Optional: Cockpit Machines

```bash
sudo apt install cockpit cockpit-machines
sudo systemctl enable --now cockpit.socket
```

Open `https://SERVER-IP:9090`. Never expose port 9090 to the public internet; restrict it to the LAN, a management VLAN, a VPN, or an SSH tunnel (`ssh -L 9090:localhost:9090 youruser@server-ip`), and enforce it with a firewall rule.

### 6. Create the first VM from the CLI

```bash
sudo mkdir -p /var/lib/libvirt/boot
sudo cp ~/ubuntu-server.iso /var/lib/libvirt/boot
virt-install --osinfo list | grep -i ubuntu

sudo virt-install \
  --connect qemu:///system \
  --name lab-ubuntu-01 \
  --memory 4096 \
  --vcpus 2 \
  --cpu host \
  --disk path=/var/lib/libvirt/images/lab-ubuntu-01.qcow2,size=30,format=qcow2,bus=virtio \
  --cdrom /var/lib/libvirt/boot/ubuntu-server.iso \
  --osinfo ubuntu24.04 \
  --network network=default,model=virtio \
  --graphics spice \
  --boot uefi \
  --noautoconsole
```

Replace `ubuntu24.04` with the identifier matching the ISO. A specific `--osinfo` lets virt-install choose proper virtual hardware defaults.

---

## `qemu:///system` vs `qemu+ssh://`

`qemu:///system` connects to the system libvirt instance on the **local** machine. `qemu+ssh://youruser@server-ip/system` connects over SSH to the system libvirt instance on **another** machine. The VMs keep running on the server; virt-manager only displays and controls them.

---

## Practical command sheet

```bash
virt-host-validate qemu
virsh -c qemu:///system list --all
virsh -c qemu:///system start VM_NAME
virsh -c qemu:///system shutdown VM_NAME
virsh -c qemu:///system autostart VM_NAME
virsh -c qemu:///system dumpxml VM_NAME
virsh -c qemu:///system net-list --all
virsh -c qemu:///system pool-list --all
```

---

## Common mistakes to avoid

| Mistake | Why it breaks things |
|---------|----------------------|
| Writing raw `qemu-system-*` commands from day one | Storage, networking, firmware, display and lifecycle all become manual work |
| Treating KVM as a standalone app | It is a kernel facility used by QEMU |
| Installing virt-manager on the headless server | Unnecessary; run it on the workstation instead |
| Mixing URIs between tools | virt-manager may show VMs that `virsh list --all` does not; always pass `-c` explicitly |
| Leaving the default storage path on the root filesystem | Growing QCOW2 images can fill `/var/lib/libvirt/images` and hurt the whole server; use a dedicated pool for real labs |
| Changing the server's network for a bridge over SSH | A malformed Netplan config can disconnect the server; have console access first |
| Adding untrusted users to `libvirt` | Membership is effectively near-root |
| Exposing Cockpit on port 9090 publicly | Management surface should stay on LAN, VPN or tunnel |

---

## Troubleshooting

| Symptom | Likely cause | What to check |
|---------|--------------|---------------|
| `/dev/kvm` missing | Virtualization disabled in UEFI/BIOS, or nested virtualization off | `sudo kvm-ok`, `lscpu`, UEFI settings |
| `virsh` permission denied | Not in `libvirt` group, or no re-login | `id`, then log out and back in |
| `qemu+ssh` connection fails | SSH key/user issue or libvirt not running | `ssh youruser@server-ip`, `systemctl status libvirtd` |
| VM has no network | `default` network inactive | `net-list --all`, `net-start default`, `net-autostart default` |

---

## Final working state

```text
Workstation:   virt-manager  ── qemu+ssh://youruser@server-ip/system
Ubuntu Server: libvirt + QEMU/KVM (headless)
               /var/lib/libvirt/images/   ← VM disks (QCOW2)
               /var/lib/libvirt/boot/     ← ISO images
               default NAT network        ← 192.168.122.0/24
```

Full write-up and extras: [A-DW/Linux_VMs_with_QEMU-KVM](https://github.com/A-DW/Linux_VMs_with_QEMU-KVM)
