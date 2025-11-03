# gpu-handoff — Proxmox NVIDIA VFIO hot handoff (host ⇆ VM) without reboot

> Production-grade shell script and hook examples to **hot swap** an NVIDIA GPU between the Proxmox **host** and a **Windows/Linux VM** cleanly — no reboots, no manual driver juggling. Works with modern NVIDIA open kernel modules and legacy branches.

---

## TL;DR
- **What**: A deterministic handoff script that switches an NVIDIA GPU between `nvidia` (host) and `vfio-pci` (VM) drivers, with proper resets, `driver_override`, fbcon detach, and module orchestration.
- **Why**: So you can use the same GPU for **host compute/desktop** when the VM is off, and for **full passthrough** when the VM is on.
- **Where**: Proxmox VE 8.x (tested on kernel `6.14.11-4-pve`).
- **How**: One script, optional `qm` hook, optional systemd units.
- **Result**: You can connect a monitor to the server and get a local console when the VM is stopped; when the VM starts, the card cleanly flips to VFIO.

```bash
# Install (host)
wget -O /usr/local/bin/gpu-handoff.sh https://raw.githubusercontent.com/<your-gh-user>/gpu-handoff/main/gpu-handoff.sh
chmod +x /usr/local/bin/gpu-handoff.sh

# On host for testing
gpu-handoff.sh to_vfio   # hand GPU to VFIO (for VM)
gpu-handoff.sh to_nvidia # bring GPU back to host (NVIDIA)
gpu-handoff.sh status    # show bindings
```

---

## Why this exists
Using one GPU for both host and a passthrough VM sounds simple, but the Linux GPU stack is **stateful** and **opaque** — especially with NVIDIA’s proprietary components. Most guides either dedicate the GPU to the VM **or** to the host and tell you to **reboot** to switch. That’s not acceptable when you want:

- **Host-side GUI/compute** while the VM is stopped
- **Zero reboot** flip to the VM for gaming/workloads
- **Deterministic recovery** when `nvidia-smi` or DRM gets wedged

This project provides the missing building block: a single, auditable script with correct sequencing and timeboxed fallbacks.

---

## Features
- **Deterministic hot handoff**: `nvidia → vfio-pci` and back
- **Safe module choreography**: stop `persistenced`, remove/insert modules in the right order (`nvidia_uvm`, `nvidia_drm`, `nvidia_modeset`, `nvidia`)
- **fbcon/DRM detach**: releases text console and framebuffers cleanly
- **Function-level reset** where supported; **remove+rescan** fallback
- **`driver_override` hygiene**: sets and clears so next boot is sane
- **Idempotent & timeboxed** operations with readable logs
- **Hook-integrations** for `qm` pre-start/post-stop (VM auto-handoff)
- **LXC niceties**: optional LXC reboot hooks if your containers use CUDA

Tested with:
- **Proxmox VE**: 8.x (`6.14.11-4-pve`)
- **NVIDIA drivers**: 580 (open kernel module) and 470 legacy
- **GPU**: GeForce RTX 5070 (others should work similarly)

> ⚠️ If you run Docker/NVIDIA Container Toolkit, ensure no containers are using `/dev/nvidia*` when handing off to VFIO.

---

## Requirements & assumptions
- IOMMU enabled: add kernel cmdline: `amd_iommu=on iommu=pt`
- Simple framebuffers disabled to avoid console grabs: `module_blacklist=simplefb,simpledrm` and `video=efifb:off video=vesafb:off`
- Proxmox host shell access as root
- Your GPU and its HDMI/DP audio device are on the same IOMMU group (typical: `0000:05:00.0` and `0000:05:00.1`)

> These defaults can be overridden with env vars; see **Configuration**.

---

## Installation

### 1) Install the script
```bash
curl -fsSL -o /usr/local/bin/gpu-handoff.sh \
  https://raw.githubusercontent.com/<your-gh-user>/gpu-handoff/main/gpu-handoff.sh
chmod +x /usr/local/bin/gpu-handoff.sh
```

### 2) (Optional) Add to PATH & bash completion
```bash
ln -s /usr/local/bin/gpu-handoff.sh /usr/local/bin/gpu-handoff
```

### 3) Verify prerequisites
```bash
# Expect to see your GPU and audio function bindings
gpu-handoff.sh status
```
If status shows your GPU on `nvidia` and the audio on `snd_hda_intel`, you’re good.

---

## Quick start

### Manual handoff
```bash
# To VM (bind to vfio-pci)
gpu-handoff.sh to_vfio

# Back to host (bind to nvidia + snd_hda_intel)
gpu-handoff.sh to_nvidia

# Status
gpu-handoff.sh status
```

### Proxmox VM hook (automatic)
Create a VM hook to hand off at **pre-start** and restore at **post-stop**.

`/var/lib/vz/snippets/vm111-hook.sh`:
```bash
#!/usr/bin/env bash
set -euo pipefail
VMID="$1"; PHASE="$2"
LOG_TAG="[vm${VMID}-hook]"

case "$PHASE" in
  pre-start)
    echo "$LOG_TAG handoff → vfio"
    /usr/local/bin/gpu-handoff.sh to_vfio
    ;;
  post-stop)
    echo "$LOG_TAG handoff → nvidia"
    /usr/local/bin/gpu-handoff.sh to_nvidia
    ;;
  *) ;;
esac
```

Make it executable and attach to your VM config:
```bash
chmod +x /var/lib/vz/snippets/vm111-hook.sh
# In /etc/pve/qemu-server/111.conf
# hookscript: local:snippets/vm111-hook.sh
```

> Replace `111` with your VMID.

### Optional: systemd units (host)
You can provide convenience targets to flip manually via `systemctl`. See `contrib/` in this repo for examples.

---

## Configuration
Defaults auto-detect the first NVIDIA GPU and its paired HDA function. You can override via env vars or edit the header in `gpu-handoff.sh`.

- `GPU_PCI` — GPU function (default auto-detected, e.g., `0000:05:00.0`)
- `GPU_AUDIO_PCI` — HDMI/DP audio function (e.g., `0000:05:00.1`)
- `RETRIES` — attempts on sensitive ops (default: `5`)
- `SLEEP_SHORT` / `SLEEP_MED` — backoff timings

Example:
```bash
GPU_PCI=0000:03:00.0 GPU_AUDIO_PCI=0000:03:00.1 gpu-handoff.sh to_vfio
```

---

## How it works (high level)
1. **Stop host consumers**: terminate `nvidia-persistenced`/MPS and sanity-check `/dev/nvidia*`
2. **Detach framebuffers**: unbind fbcon/DRM so the tty lets go of the GPU
3. **Module orchestration**: unload `nvidia_*` when moving to VFIO; load when moving back
4. **Reset**: attempt FLR; otherwise, do `remove + rescan` to get a fresh device node
5. **Bind**: set `driver_override` and bind to target (`vfio-pci` or `nvidia`/`snd_hda_intel`)
6. **Hygiene**: clear `driver_override`, re-enable fbcon as needed

This sequence fixes common wedges like `Device or resource busy`, `Xid 119` GSP timeouts, and DRM flip timeouts.

---

## Troubleshooting
### “Device or resource busy” on unbind
- Ensure **no Docker containers** or services have `/dev/nvidia*` open
- Stop `nvidia-persistenced` and MPS (`nvidia-cuda-mps-server`)
- The script already does timeboxed `pkill -9` and retries

### `nvidia-smi` shows `ERR!` columns or `Unknown Error`
- Run `gpu-handoff.sh to_nvidia` again — it performs **remove+rescan** and rebind
- Check `journalctl -k -b -g 'nvidia|vfio|drm|fb|reset|remove|rescan'`

### Xid 119 / GSP RPC timeouts
- Happens when the device is **half-initialized**; our sequence forces a reset
- Persistent issues can indicate a very old vBIOS or buggy driver. Try another driver branch

### `no NVIDIA GPU found`
- Handoff mid-flight can make `lspci` transiently miss devices. Re-run status a second later

### Flip event timeout on head 0 (DRM)
- Typical of a grabbed fbcon. Our fb detach step handles this; ensure boot params disable simpledrm/efifb

### Can’t open new SSH sessions during a flip
- Very busy hosts can spike CPU/IO during detach/reset. Prefer running handoff **non-interactively** via hooks or systemd; avoid flipping under high disk pressure

---

## Security & safety
- Script runs as root; it only touches GPU-related sysfs, kernel modules and service daemons
- No external network calls; pure local operations
- Designed to be **idempotent** and safe to re-run if a step fails

---

## FAQ
**Q: Will this work with AMD?**  
Parts of the flow do, but AMD resets are simpler and often don’t need fancy override hygiene. A future `amd-handoff.sh` could be added.

**Q: Can I keep the host Xorg/Wayland session while flipping?**  
No. You must release DRM/fbcon. The host can return to a text console while the VM owns the card.

**Q: Does this survive kernel/driver updates?**  
Yes. The script avoids hardcoding module paths and cleans `driver_override`. Re-run after updates.

**Q: Can I use multiple GPUs?**  
Yes — run one VM per GPU or parameterize `GPU_PCI/GPU_AUDIO_PCI`.

---

## SEO: help others find this
- **Repository name**: `proxmox-nvidia-vfio-handoff` (includes core keywords)
- **Title (H1)**: `Proxmox NVIDIA VFIO hot handoff: switch GPU between host and VM without reboot`
- **Meta description (for GitHub Pages/docs)**: `Production-ready script for Proxmox to switch an NVIDIA GPU between host (nvidia) and VM (vfio-pci) on the fly. Handles driver overrides, resets, DRM/fbcon detach, and module orchestration.`
- **Keywords to naturally include**: `Proxmox GPU passthrough`, `NVIDIA VFIO`, `hot handoff`, `bind vfio-pci`, `unbind nvidia`, `driver_override`, `remove+rescan`, `function level reset`, `fbcon detach`, `nvidia-persistenced`, `Xid 119`, `Unknown Error`, `flip event timeout`, `hypervisor`, `Windows 11 VM`, `Linux VM`, `single GPU`, `no reboot`, `Proxmox VE 8`.
- **Search intent Q&A to include in README**:
  - “How do I switch an NVIDIA GPU between Proxmox host and VM without reboot?”
  - “Why does `nvidia-smi` show Unknown Error after vfio bind?”
  - “How to fix Xid 119 GSP timeout when doing GPU passthrough?”
  - “Proxmox vfio-pci handoff script for NVIDIA”

> Tip: Create a short blog post linking to the repo with the same keywords. Cross-post to the Proxmox forum and Reddit (/r/Proxmox, /r/VFIO) — that’s where operators search.

---

## Contributing
PRs and issues welcome! Please include:
- `pveversion -v`
- `uname -a`
- `journalctl -k -b -g 'nvidia|vfio|drm|fb|reset|remove|rescan'`
- `lspci -nnk | grep -A3 -E "VGA|Audio|NVIDIA|vfio"`

---

## License
MIT — see `LICENSE`.

---

## Acknowledgements
Shoutout to the VFIO, Proxmox and Nouveau communities for years of reverse-engineering and docs archeology.

---

## Appendix: Kernel cmdline example (Proxmox)
Edit `/etc/default/grub` (or Proxmox kernel cmdline equivalent) to include:
```
amd_iommu=on iommu=pt \
module_blacklist=simplefb,simpledrm \
video=efifb:off video=vesafb:off \
default_hugepagesz=2M hugepagesz=2M hugepages=0 \
kvm_amd.nested=1
```
Then update grub and reboot.

---

## Appendix: Example systemd units
See `contrib/systemd/`:
- `gpu-to-vfio.service` — `ExecStart=/usr/local/bin/gpu-handoff.sh to_vfio`
- `gpu-to-nvidia.service` — `ExecStart=/usr/local/bin/gpu-handoff.sh to_nvidia`

Enable for quick flips:
```bash
systemctl enable --now gpu-to-nvidia.service
# later
systemctl start gpu-to-vfio.service
```

---

## Screens & expected output
- `status` shows current drivers, e.g. `0000:05:00.0 driver=nvidia`, `0000:05:00.1 driver=snd_hda_intel`
- After `to_vfio`, both become `vfio-pci`
- After `to_nvidia`, GPU back on `nvidia`, audio on `snd_hda_intel`, `nvidia-smi` healthy

If you see wedges, re-run the respective handoff; the script is idempotent and includes retries.
