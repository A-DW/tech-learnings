# 🧠 Tech Learnings

A living, ever-evolving knowledge base documenting what I learn along the way — AI, tools, infrastructure, automation, cybersecurity, and everything in between.

> This repo is not a course. It's a map of my technical journey.

---

## 📂 Structure

```
tech-learnings/
├── ai/                  # LLMs, prompting, local inference, APIs
├── tools/               # CLI tools, dev utilities, platforms
├── infrastructure/      # Docker, Linux, Cloudflare, cloud
├── automation/          # N8N, scripting, workflow design
├── cybersecurity/       # Techniques, CTFs, research notes
├── networking/          # Protocols, proxies, tunnels
└── resources/           # Curated links, references, cheatsheets
```

---

## 🗂️ Topics

### 🤖 AI
- [ ] Local LLM deployment with Ollama
- [ ] LLM API integration and cost optimization
- [ ] Prompt engineering patterns
- [ ] AI workflow automation (N8N + LLMs)

### 🛠️ Tools
- [ ] VS Code setup & extensions
- [ ] Docker & container orchestration
- [ ] Git advanced workflows
- [x] [Antigravity IDE — Linux tarball install, sandbox troubleshooting & GNOME launcher](tools/antigravity-ide/Antigravity-IDE-Manual.md)

### 🏗️ Infrastructure
- [x] [Linux VMs with QEMU/KVM — libvirt, virt-manager, desktop & headless server setup](https://github.com/A-DW/Linux_VMs_with_QEMU-KVM) *(standalone repo, summary below)*
- [ ] Ubuntu 24.04 server hardening
- [ ] Cloudflare tunnels & DNS
- [ ] Oracle Cloud & self-hosting
- [ ] Proxmox VE for a dedicated home-lab host

#### 🖥️ Summary: Linux VMs with QEMU/KVM

Repo: [A-DW/Linux_VMs_with_QEMU-KVM](https://github.com/A-DW/Linux_VMs_with_QEMU-KVM) — my learning path for running VMs natively on Linux instead of VirtualBox.

**The stack, in one glance**

```text
virt-manager / virsh   → interface (GUI / CLI)
libvirt                → manages VMs, networks, storage
QEMU                   → provides the virtual hardware
KVM                    → kernel module exposing CPU virtualization
Intel VT-x / AMD-V     → hardware extensions
```

**Key takeaways**
- KVM is a kernel facility, not an app; QEMU runs the VM and uses KVM for near-native speed; libvirt manages everything; virt-manager is the app you actually open.
- Chosen stack: **QEMU/KVM + libvirt + virt-manager**. Avoid hand-written `qemu-system-*` commands, GNOME Boxes as the main tool, and Proxmox on a daily-driver workstation.
- First-VM defaults: `qemu:///system` connection, UEFI, QCOW2 disk, VirtIO disk/network, SPICE display, **NAT** via the `default` network (bridging comes later).
- Windows guests need **virtio-win** drivers; Windows 11 also needs an emulated TPM (`swtpm`).
- Install `qemu-guest-agent` (and `spice-vdagent`) inside guests for host–guest integration.
- Security note: the `libvirt` group is effectively near-root on the host — only add trusted accounts.

**Headless server setup** ([ServerEnvironmentInstallation.md](https://github.com/A-DW/Linux_VMs_with_QEMU-KVM/blob/main/ServerEnvironmentInstallation.md))
- Install QEMU/KVM + system libvirt on Ubuntu Server; keep `virt-manager` on the workstation and connect with `qemu+ssh://user@server/system`.
- Alternatives: **Cockpit Machines** (browser UI on port 9090, restrict it with a firewall/VPN/SSH tunnel, never expose it publicly) or pure `virsh` + `virt-install`.
- Verify with `kvm-ok`, `virt-host-validate qemu` and `virsh -c qemu:///system list --all`.
- Watch out: default storage is `/var/lib/libvirt/images` on the root filesystem (use a dedicated pool for real labs), and changing the server's network config remotely for a bridge can lock you out.

**Also in the repo**
- Practical `virsh` command sheet (start, shutdown, autostart, dumpxml, net-list, pool-list).
- [Concepts-FAQ.md](https://github.com/A-DW/Linux_VMs_with_QEMU-KVM/blob/main/Concepts-FAQ.md): why QEMU and KVM work together and what `qemu:///system` means.
- Troubleshooting table for missing `/dev/kvm`, permission denied, failed `qemu+ssh`, and VMs without network.

### ⚙️ Automation
- [ ] N8N workflows
- [ ] Python automation scripts
- [ ] Google Cloud APIs (Sheets, Places)

### 🔐 Cybersecurity
- [ ] Privilege escalation techniques
- [ ] Network analysis
- [ ] CTF write-ups

---

## 📅 Log

| Date | Topic | Notes |
|------|-------|-------|
| 2026-10-10 | Linux VMs with QEMU/KVM | New repo [Linux_VMs_with_QEMU-KVM](https://github.com/A-DW/Linux_VMs_with_QEMU-KVM): KVM vs QEMU vs libvirt vs virt-manager, NAT-first networking, VirtIO drivers for Windows guests, headless Ubuntu Server setup |
| 2026-06-10 | Antigravity IDE on Linux | User-local tarball install, `--no-sandbox` fix, GNOME `.desktop` launcher wiring |

---

## 📌 Notes

- Notes are raw and opinionated — written for future me, not for documentation's sake.
- Each folder may contain its own `README.md` with deeper context.
