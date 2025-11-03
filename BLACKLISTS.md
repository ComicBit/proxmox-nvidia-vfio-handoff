# GPU / Framebuffer Blacklist Reference

This document describes how to inspect, clean, and standardize your Proxmox host’s
`/etc/modprobe.d` configuration so the GPU handoff between host and VM works
consistently.

---

## 🧠 Why this matters

NVIDIA and VFIO kernel modules are picky about load order and conflicting rules.
Over time, driver packages, DKMS, and Proxmox updates scatter small snippets such as:

- `/etc/modprobe.d/nvidia.conf`
- `/etc/modprobe.d/pve-blacklist.conf`
- `/etc/modprobe.d/blacklist-generic-fb.conf`
- `/etc/modprobe.d/nvidia-nofb.conf`

Many of them set contradictory flags like:
- `options nvidia_drm modeset=0` vs `modeset=1`
- `NVreg_PreserveVideoMemoryAllocations=1` vs `=0`
- Blacklisting `nvidia*` while the host is meant to use NVIDIA as console

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
    echo "driver=$(basename "$(readlink -f "$d/driver")")"
  else
    echo "driver=(none)"
  fi
  if [ -f "$d/driver_override" ]; then
    printf "override="; cat "$d/driver_override"
  else
    echo "override=(none)"
  fi
done
