# GPU / Framebuffer Blacklist Reference

This document describes how to inspect, clean, and standardize your Proxmox host’s
`/etc/modprobe.d` configuration so the GPU handoff between host and VM works
consistently.

---

## 🧠 Why this matters

NVIDIA and VFIO kernel modules are picky about load order and conflicting rules.
Over time, driver packages, DKMS, and Proxmox updates scatter small snippets such as:

* `/etc/modprobe.d/nvidia.conf`
* `/etc/modprobe.d/pve-blacklist.conf`
* `/etc/modprobe.d/blacklist-generic-fb.conf`
* `/etc/modprobe.d/nvidia-nofb.conf`

Many of them set contradictory flags like:

* `options nvidia_drm modeset=0` vs `modeset=1`
* `NVreg_PreserveVideoMemoryAllocations=1` vs `=0`
* Blacklisting `nvidia*` while the host is meant to use NVIDIA as console

These inconsistencies cause unreliable handoffs and sometimes full system hangs
when unbinding or rebinding the GPU.

---

## 🔍 Inspect your active blacklist configuration

Run this diagnostic block to review all GPU-related modprobe files and boot state:

```bash
# Inspect graphics/vfio config and boot state
echo "=== /proc/cmdline ==="
cat /proc/cmdline
echo

echo "=== modprobe.d snippets mentioning nvidia/nouveau/fb/vfio ==="
grep -HnE '(^|[^a-z])(nvidia|nouveau|vfio|fb|framebuffer)' /etc/modprobe.d/*.conf 2>/dev/null | sed 's/^/• /' || true
echo

echo "=== Full contents of those snippets (sorted) ==="
for f in $(grep -lE '(^|[^a-z])(nvidia|nouveau|vfio|fb|framebuffer)' /etc/modprobe.d/*.conf 2>/dev/null | sort); do
  echo "----- $f -----"
  sed -n '1,200p' "$f"
  echo
done

echo "=== Modules listed for initramfs preloading ==="
grep -HnE 'nvidia|vfio|fb' /etc/initramfs-tools/modules 2>/dev/null || echo "(none)"
echo

echo "=== What’s actually inside the current initramfs ==="
lsinitramfs /boot/initrd.img-$(uname -r) | grep -E '(^|/)(nvidia|vfio|simpledrm|efifb|vesafb)\b' || echo "(no matches)"
echo

echo "=== Current driver bindings & overrides for GPU + HDA ==="
for d in /sys/bus/pci/devices/0000:05:00.{0,1}; do
  [ -e "$d" ] || continue
  echo "== $d =="
  if [ -L "$d/driver" ]; then
    echo "driver=$(basename \"$(readlink -f \"$d/driver\")\")"
  else
    echo "driver=(none)"
  fi
  if [ -f "$d/driver_override" ]; then
    printf "override="; cat "$d/driver_override"
  else
    echo "override=(none)"
  fi
done
```

---

## 🧹 Standardize your modprobe rules

We recommend keeping **exactly one canonical file** defining your driver behavior.

Create or edit `/etc/modprobe.d/10-nvidia-proxmox.conf`:

```conf
# Canonical NVIDIA settings for Proxmox + GPU handoff

# Disable the open-source driver
blacklist nouveau
options nouveau modeset=0

# Enable KMS so the host can show a console on the NVIDIA driver
options nvidia_drm modeset=1

# NVIDIA runtime options
options nvidia NVreg_PreserveVideoMemoryAllocations=0
options nvidia NVreg_EnableS0ixPowerManagement=1
options nvidia NVreg_TemporaryFilePath=/var/tmp
```

Optional VFIO file (`/etc/modprobe.d/10-vfio.conf`):

```conf
# Leave commented unless you want static binding
# options vfio-pci ids=10de:2f04,10de:2f80 disable_idle_d3=1
```

---

## 🚫 Disable conflicting files

Back up and disable redundant snippets:

```bash
sudo bash -c 'for f in \
  /etc/modprobe.d/00-nvidia-clean.conf \
  /etc/modprobe.d/nvidia.conf \
  /etc/modprobe.d/nvidia-modeset.conf \
  /etc/modprobe.d/nvidia-nofb.conf \
  /etc/modprobe.d/blacklist-generic-fb.conf \
  /etc/modprobe.d/pve-blacklist.conf \
; do [ -f "$f" ] && cp -n "$f" "${f}.bak" && mv -v "$f" "${f}.disabled"; done'
```

---

## 🛡 Rebuild initramfs

Once your configuration is clean:

```bash
sudo grep -q '^nvidia$' /etc/initramfs-tools/modules || \
  printf "%s\n" nvidia nvidia_modeset nvidia_drm nvidia_uvm | sudo tee -a /etc/initramfs-tools/modules

sudo update-initramfs -u
```

If you use `systemd-boot` with ZFS, refresh:

```bash
sudo proxmox-boot-tool refresh
```

---

## ✅ Expected result

After reboot:

* `nvidia`, `nvidia_modeset`, `nvidia_drm`, and `nvidia_uvm` load early.
* No duplicate or disabled framebuffer drivers (`simpledrm`, `efifb`, `vesafb`).
* `/usr/local/bin/gpu-handoff.sh status` shows `nvidia` when VM is off, and `vfio-pci` while VM is running.
* Monitor plugged into the server shows the host shell when VM is stopped.

---

## 🧩 FAQ

**Q:** Why not just blacklist `nvidia*` in `pve-blacklist.conf` like other guides?
**A:** Because we want the host to own the GPU when the VM is off.
Our handoff handles rebinding automatically when the VM stops.

**Q:** Do I need to blacklist `simplefb` or `efifb` manually?
**A:** No. They’re already disabled at boot via kernel cmdline:
`module_blacklist=simplefb,simpledrm video=efifb:off video=vesafb:off`.

**Q:** Why is `NVreg_PreserveVideoMemoryAllocations=0` used?
**A:** It ensures reliable resets and avoids “device busy” errors when rebinding the card.

---

### Summary

| Component                 | Purpose                                                 |
| ------------------------- | ------------------------------------------------------- |
| `10-nvidia-proxmox.conf`  | Single source of truth for NVIDIA configuration         |
| `initramfs-tools/modules` | Ensures drivers load early                              |
| `kernel cmdline`          | Disables firmware framebuffers for clean NVIDIA console |
| `gpu-handoff.sh`          | Safely switches the GPU between host and VM             |
| `vm-hook.sh`              | Integrates the switch with Proxmox VM lifecycle         |
