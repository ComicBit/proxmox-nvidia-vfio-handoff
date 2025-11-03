# Proxmox NVIDIA VFIO Handoff

Seamless NVIDIA GPU hot handoff between Proxmox host and VM — bind/unbind nvidia ⇆ vfio-pci safely, no reboots.

---

## 🚀 Overview

This project provides a **production-grade shell script** and optional Proxmox hook to safely switch your NVIDIA GPU between the **host** and a **virtual machine** without rebooting. It performs a reliable driver handoff between `nvidia` and `vfio-pci`, ensuring the GPU works for both host and VM use cases.

It solves one of the biggest pain points in single-GPU Proxmox setups.

Tested on **Proxmox VE 9.x** with **latest available kernel**.

---

## ⚙️ Features

* Seamless NVIDIA ⇆ VFIO driver rebinding (host <-> VM)
* Automatic `fbcon` detachment to avoid TTY lockups
* Restores console display on host when VM stops
* Integrated with Proxmox VM lifecycle via hook script
* Clear logs for each operation
* Safe, idempotent, and timeboxed
* Supports both single-GPU and dual-GPU setups

---

## 🧩 Installation

1. **Copy scripts:**

   ```bash
   sudo cp gpu-handoff.sh /usr/local/bin/gpu-handoff.sh
   sudo chmod +x /usr/local/bin/gpu-handoff.sh
   ```

2. **Add hook script:**

   ```bash
   sudo cp vm-hook.sh /var/lib/vz/snippets/vm111-hook.sh
   sudo chmod +x /var/lib/vz/snippets/vm111-hook.sh
   ```

3. **Link hook to VM:**

   Edit your VM config:

   ```bash
   sudo nano /etc/pve/qemu-server/111.conf
   ```

   Add this line:

   ```
   hookscript: local:snippets/vm111-hook.sh
   ```

4. **Rebuild initramfs:**

   ```bash
   sudo update-initramfs -u
   ```

---

## 🧱 Configuration sanity check

GPU driver conflicts can easily occur if your system carries leftover modprobe snippets from previous driver installs (e.g., `nvidia.conf`, `pve-blacklist.conf`, `blacklist-fb.conf`).

This repository includes a [**BLACKLISTS.md**](./BLACKLISTS.md) guide that explains how to inspect, clean, and standardize your `modprobe.d` rules so the NVIDIA and VFIO drivers behave predictably across boots.

It’s strongly recommended to go through that section before reporting issues.

---

## 🧠 How it works

### Host boot

At boot, the host owns the GPU. The NVIDIA driver stack (`nvidia`, `nvidia_drm`, `nvidia_modeset`, `nvidia_uvm`) loads early, giving the host both display and CUDA access.

### VM start

When the VM starts, the Proxmox hook triggers:

```bash
/usr/local/bin/gpu-handoff.sh to_vfio
```

The script:

* Stops NVIDIA-related userland services
* Detaches framebuffer console (`fbcon`)
* Unloads NVIDIA kernel modules
* Binds the GPU and HDMI audio function to `vfio-pci`
* Starts the VM cleanly

### VM stop

When the VM stops, Proxmox calls:

```bash
/usr/local/bin/gpu-handoff.sh to_nvidia
```

The script:

* Unbinds GPU from `vfio-pci`
* Reloads NVIDIA modules
* Rebinds GPU to `nvidia` and HDMI audio to `snd_hda_intel`
* Restores console output to the host

### Lifecycle integration

The hook integrates automatically, pinning QEMU threads to specific CPUs and logging each handoff phase.

---

## 🔍 Status check

At any time, you can inspect bindings:

```bash
/usr/local/bin/gpu-handoff.sh status
```

Example output:

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

When the VM is **off**, a monitor connected to the GPU displays the host shell (thanks to NVIDIA DRM KMS).
When the VM is **running**, the monitor automatically switches to the VM’s output.

---

## 🔧 Requirements

* Proxmox VE 9.0.x (kernel 6.14.11-4-pve)
* NVIDIA proprietary driver (Tested on 580.82.07-1)
* `vfio-pci` kernel module enabled
* VM configured with GPU + audio PCIe passthrough
* IOMMU enabled (`amd_iommu=on iommu=pt`)

---

## 🧱 Troubleshooting

* **Device busy errors** → Caused by framebuffer leftovers or modprobe conflicts. See [BLACKLISTS.md](./BLACKLISTS.md).
* **No console after reboot** → Ensure `options nvidia_drm modeset=1` and disable simpledrm/efifb.
* **VM start hang** → Check `dmesg` for VFIO errors; NVIDIA services may still be running.
* **SSH freeze during flip** → Avoid flipping under heavy I/O; run handoff non-interactively via hooks.

---

## 🧩 Credits

Built and tested on real Proxmox deployments with single-GPU passthrough setups.
Created by **Comicbit** — refined for open-source release.

---

## 📜 License

MIT License — free to use, modify, and share.
If this project saves you time, star the repo and share it with the community.
