# Proxmox NVIDIA VFIO Handoff

Seamless NVIDIA GPU hot handoff between Proxmox host and VM — safely bind/unbind nvidia ⇆ vfio-pci **without rebooting**.

---

## 🚀 Overview

This project delivers a **production-grade handoff script** and optional **Proxmox hook integration** that lets you switch your NVIDIA GPU between the **host** and a **VM** dynamically — **no reboot required**.

It performs a clean driver rebinding process between `nvidia` (host) and `vfio-pci` (VM), orchestrating resets, fbcon detaches, and module sequencing. This enables you to use your GPU for **host compute**, **CUDA workloads**, or **desktop output** when idle, and for **full passthrough performance** when the VM runs.

Tested on **Proxmox VE 9.x** with kernel `6.14.11-4-pve` and modern NVIDIA drivers (580+).

---

## ⚙️ Key Features

* **Deterministic hot handoff** between `nvidia` and `vfio-pci`
* **Automatic framebuffer detachment** to avoid TTY lockups
* **Host console restore** after VM stops
* **Full integration with VM lifecycle** via Proxmox hooks
* **Safe module unloading/loading sequence** (`nvidia_uvm`, `nvidia_drm`, `nvidia_modeset`, `nvidia`)
* **Function-level resets** with fallback to `remove+rescan`
* **Readable logs** for every phase
* **Idempotent & timeboxed** actions to prevent hangs
* **Support for single- and multi-GPU environments**

> ⚠️ Designed for single-GPU passthrough setups but safe to extend to multi-GPU systems.

---

## 🧩 Installation

### 1. Install the handoff script

```bash
wget -O /usr/local/bin/gpu-handoff.sh \
  https://raw.githubusercontent.com/ComicBit/proxmox-nvidia-vfio-handoff/main/gpu-handoff.sh
sudo chmod +x /usr/local/bin/gpu-handoff.sh
```

This is the main control script that handles driver switching, module resets, and state recovery. It can be invoked manually or automatically via a VM hook.

### 2. Create the Proxmox VM hook

This hook automatically flips the GPU during VM lifecycle events.

```bash
sudo nano /var/lib/vz/snippets/vm111-hook.sh
```

Paste this content:

```bash
#!/usr/bin/env bash
set -euo pipefail
VMID="$1"; PHASE="$2"
HANDOFF="/usr/local/bin/gpu-handoff.sh"
LOG_TAG="[vm${VMID}-hook]"

case "$PHASE" in
  pre-start)
    echo "$LOG_TAG handoff → vfio"
    "$HANDOFF" to_vfio
    ;;
  post-stop)
    echo "$LOG_TAG handoff → nvidia"
    "$HANDOFF" to_nvidia
    ;;
  post-start)
    echo "$LOG_TAG pinning CPU threads (optional)"
    ;;
  *) echo "$LOG_TAG phase $PHASE ignored" ;;
esac
```

Then make it executable:

```bash
sudo chmod +x /var/lib/vz/snippets/vm111-hook.sh
```

### 3. Attach the hook to your VM

Edit your VM configuration:

```bash
sudo nano /etc/pve/qemu-server/111.conf
```

Add this line:

```
hookscript: local:snippets/vm111-hook.sh
```

Replace `111` with your actual VMID.

### 4. Rebuild initramfs (recommended)

```bash
sudo update-initramfs -u
```

---

## 🧱 Configuration sanity check

Old NVIDIA or framebuffer configs can interfere with the handoff process.

Check [**BLACKLISTS.md**](./BLACKLISTS.md) for guidance on:

* Removing legacy modprobe rules (`nouveau`, `simplefb`, `efifb`)
* Ensuring `nvidia_drm.modeset=1` is enabled
* Confirming consistent module behavior across boots

A single bad modprobe line can break driver switching — cleaning this is essential.

---

## 🧠 How it works

### 🖥️ Host boot phase

At boot, the host owns the GPU. NVIDIA modules load normally, providing console display, CUDA, and OpenGL acceleration.

### ▶️ VM start

The hook runs:

```bash
/usr/local/bin/gpu-handoff.sh to_vfio
```

Steps:

* Stops `nvidia-persistenced` and CUDA MPS servers
* Detaches framebuffer (`fbcon`)
* Unloads NVIDIA modules
* Binds the GPU (`0000:05:00.0`) and audio function (`0000:05:00.1`) to `vfio-pci`
* Starts the VM cleanly

### ⏹️ VM stop

When the VM shuts down:

```bash
/usr/local/bin/gpu-handoff.sh to_nvidia
```

Steps:

* Unbinds GPU/audio from `vfio-pci`
* Reloads NVIDIA modules in correct order
* Rebinds to `nvidia` and `snd_hda_intel`
* Restores the console display to the host

### 🔁 Recovery & retries

All operations are timeboxed with retries. If a module fails to unbind or reload, the script logs the issue and continues safely.

---

## 🔍 Status command

Check current GPU bindings anytime:

```bash
gpu-handoff.sh status
```

Outputs:

```
[gpu-handoff] 0000:05:00.0 driver=nvidia
[gpu-handoff] 0000:05:00.1 driver=snd_hda_intel
```

When VM is running:

```
[gpu-handoff] 0000:05:00.0 driver=vfio-pci
[gpu-handoff] 0000:05:00.1 driver=vfio-pci
```

---

## 🖥️ Monitor behavior

* When the **VM is off**, your monitor connected to the GPU displays the Proxmox host console (DRM KMS).
* When the **VM starts**, it switches automatically to the VM output.
* When stopped again, the host regains the display — no manual input needed.

---

## 🔧 Requirements

* Proxmox VE 9.x (kernel ≥ 6.14)
* NVIDIA drivers 580+ (tested with 580.82.07-1)
* `vfio-pci` kernel module enabled
* IOMMU support enabled (`amd_iommu=on iommu=pt`)
* Proper GPU + audio PCIe passthrough in VM config

---

## 🧱 Troubleshooting

* **Device busy errors:** Usually caused by framebuffer remnants or incorrect blacklists → see [BLACKLISTS.md](./BLACKLISTS.md)
* **No console after reboot:** Ensure `options nvidia_drm modeset=1` and disable `simpledrm`/`efifb`
* **VM start hang:** Check `dmesg` for VFIO or NVIDIA conflicts
* **`Unknown Error`**** in nvidia-smi:** GPU may still be half-bound; rerun `to_nvidia` or reboot modules
* **SSH freezes:** Avoid running handoff interactively during high I/O — hooks handle it safely

---

## 🧩 Contributing

PRs, improvements, and issue reports welcome.
Please include:

* `pveversion -v`
* `uname -a`
* `journalctl -k -b -g 'nvidia|vfio|drm|fb|reset|remove|rescan'`
* `lspci -nnk | grep -A3 -E 'VGA|Audio|NVIDIA|vfio'`

---

## 🧩 Credits

Built and tested on real single-GPU Proxmox systems.
Created by **ComicBit **— open-sourced for the community.

---

## 📜 License

MIT License — use, modify, and share freely.
If this project saves you time, star the repo and share it with others.
