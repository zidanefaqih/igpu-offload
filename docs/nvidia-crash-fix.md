# NVIDIA GSP crash on hybrid laptops (Arch + Hyprland): diagnosis, symptoms & fix

**Hardware/software this was diagnosed on:**

| | |
|---|---|
| Laptop | Acer Nitro (TigerLake-H i7-11400H) |
| iGPU | Intel UHD Graphics (TGL GT1) — `i915` |
| dGPU | NVIDIA GeForce RTX 3050 Laptop GPU (GA107) |
| OS | Arch Linux, kernel 7.2.6-arch2-1 |
| Compositor | Hyprland (uwsm) |
| Broken driver | `nvidia-open` / `nvidia-open-dkms` **615.71.09** |
| Working driver | `nvidia-580xx-*` **580.178.04** (proprietary) with **GSP disabled** |

---

## TL;DR

The open kernel modules **require GSP firmware**, and on this laptop GSP
crashed three times (`Xid 62: PMU has halted`), each time requiring a cold
power cycle. The crash also silently broke the whole GL stack, which made
unrelated apps (Steam!) crash.

**Fix:** switch to the proprietary 580xx modules and disable GSP:

```
options nvidia NVreg_EnableGpuFirmware=0 NVreg_PreserveVideoMemoryAllocations=1 NVreg_TemporaryFilePath=/var/tmp
```

---

## 1. The primary problem: `Xid 62` (GSP PMU halt)

Three crashes, ~2 days apart. Kernel log pattern, always the same:

```
NVRM: GPU0 _kgspRpcGspEventPmuHalted: Received signal from GSP that PMU has halted.
NVRM: Xid (PCI:0000:01:00): 62, 224f811e ...
NVRM: Xid (PCI:0000:01:00): 154, GPU recovery action changed from 0x0 (None) to 0x1 (PF FLR)
... either of:
NVRM: Xid (PCI:0000:01:00): 16, Head 00000003 Count ...   (display head dies)
NVRM: krcWatchdogCallbackVblankRecovery_IMPL: ... 7 Seconds without a Vblank Counter Update on head:D0
NVRM: Xid (PCI:0000:01:00): 44, MMU Fault: ENGINE GRAPHICS ...
...
NVRM: nvCheckOkFailedNoLog: Check failed: Reset required [NV_ERR_RESET_REQUIRED] (0x00000062)
NVRM: RmInitAdapter failed! (0x62:0x55:2130)
```

Recovery from this state requires a **cold power cycle** (full power off).
A warm reboot may not reset the dGPU.

Crashes #1 and #2 happened shortly after suspend/resume. Crash #3 happened
**spontaneously** ~1h50m into a session with no suspend and runtime D3
already disabled. So runtime-PM hardening alone is **not** sufficient.

## 2. The side effect nobody expects: GL stack breaks

After the GPU locks up, the driver cannot re-initialise, and this is what
the system looks like:

```console
$ nvidia-smi -q | grep -i 'video memory'     # may still answer...
$ eglinfo -B | grep 'EGL vendor'
EGL vendor string: Mesa Project              # ← falls back to llvmpipe!

$ LIBGL_DEBUG=verbose glxinfo32 -B
X Error of failed request:  BadValue (integer parameter out of range for operation)
  Major opcode of failed request:  150 (GLX)
  Minor opcode of failed request:  24 (X_GLXCreateNewContext)
```

So:

- **EGL** silently falls back to **software rendering (llvmpipe)** — everything
  gets slow, CPU load rises.
- **Xwayland GLX is broken** for both 32-bit and 64-bit clients.
- Any app that calls `glGetString(GL_EXTENSIONS)` now receives **NULL**.

## 3. The visible casualty: Steam segfaults on launch

Steam (32-bit, `vgui2_s.so`) passes the result of `glGetString()` straight
into `strchr()`/`strlen()` without a NULL check, so it crashes before the UI
appears:

```
#0  __GI_strchr (libc.so.6+0xc7b51)   <- called with NULL
#1  vgui2_s.so+0xfab92
#2  vgui2_s.so+0x14f06b
#3  vgui2_s.so+0x16151f
#4  steamui.so+0x1f3179a
...
```

This is an **upstream Steam bug**, not a local misconfiguration:

- [ValveSoftware/steam-for-linux#13269](https://github.com/ValveSoftware/steam-for-linux/issues/13269)
  — `glGetString(GL_EXTENSIONS)` NULL → `strstr()` crash (root-caused by the reporter)
- [ValveSoftware/steam-for-linux#13627](https://github.com/ValveSoftware/steam-for-linux/issues/13627)
  — same offsets, crash right after the XRandR workaround

**So if Steam suddenly segfaults on a hybrid NVIDIA laptop, check the GPU
health *before* reinstalling Steam.**

Quick health check:

```bash
sudo dmesg | grep -iE 'NVRM|Xid'          # any Xid 62/16/44 = GPU already hung
eglinfo -B | grep 'EGL vendor'            # "Mesa Project" = NVIDIA EGL is down
```

## 4. The fix: proprietary modules + GSP disabled

The open modules ship the GSP as an inseparable part of the design; you
cannot disable it there. The **proprietary** modules still support the
legacy RM initialisation path, and the switch is officially documented in
the driver README, chapter *"44B. DISABLING GSP MODE"*.

Because Arch no longer packages proprietary `nvidia-dkms`, use the AUR
`580xx` branch (supports Maxwell → Blackwell, including RTX 3050 Laptop GPU —
checked against NVIDIA's `supportedchips.html`):

```bash
paru -S nvidia-580xx-dkms nvidia-580xx-utils lib32-nvidia-580xx-utils
# conflicts with nvidia-open-dkms / nvidia-utils will be resolved by paru
```

Then write the module options:

```bash
sudo tee /etc/modprobe.d/nvidia-pm-fix.conf <<'EOF'
# Disable GSP: the GSP firmware crashed repeatedly on this laptop
# (Xid 62 "PMU has halted" -> Xid 154 -> Xid 16 -> RmInitAdapter failed 0x62:0x55)
# with the open kernel modules. The proprietary module can run without GSP.
options nvidia NVreg_EnableGpuFirmware=0 NVreg_PreserveVideoMemoryAllocations=1 NVreg_TemporaryFilePath=/var/tmp
EOF

sudo mkinitcpio -P
sudo reboot
```

> The CachyOS-maintained `nvidia-580xx` packages enable kernel modesetting by
> default (patched), so `nvidia_drm.modeset=1` is **not** required.

## 5. Verification

```console
$ nvidia-smi --query-gpu=driver_version --format=csv,noheader
580.178.04

$ nvidia-smi -q | grep 'GSP Firmware Version'
    GSP Firmware Version                               : N/A      # ← GSP is off

$ grep EnableGpuFirmware /proc/driver/nvidia/params
EnableGpuFirmware: 0

$ eglinfo -B | grep 'EGL vendor'
EGL vendor string: NVIDIA

$ DISPLAY=:1 glxinfo32 -B | grep renderer
OpenGL renderer string: NVIDIA GeForce RTX 3050 Laptop GPU/PCIe/SSE2
```

Afterwards Steam starts normally (its crash was only a symptom).

## 6. Rollback

Keep the previous packages around before switching:

```bash
mkdir -p ~/backup-nvidia
cp /var/cache/pacman/pkg/nvidia-utils-*.pkg.tar.zst \
   /var/cache/pacman/pkg/lib32-nvidia-utils-*.pkg.tar.zst \
   /var/cache/pacman/pkg/nvidia-open-dkms-*.pkg.tar.zst ~/backup-nvidia/
```

Rollback from a TTY (`Ctrl+Alt+F3`):

```bash
sudo pacman -Rdd --noconfirm nvidia-580xx-utils nvidia-580xx-dkms lib32-nvidia-580xx-utils
sudo pacman -U --noconfirm ~/backup-nvidia/*.pkg.tar.zst
sudo sed -i 's/ NVreg_EnableGpuFirmware=0//' /etc/modprobe.d/nvidia-pm-fix.conf
sudo mkinitcpio -P && sudo reboot
```

## 7. Things that were tried first and did **not** fix it

| Attempt | Result |
|---|---|
| `NVreg_PreserveVideoMemoryAllocations=1` + `nvidia-suspend/resume` services | Helps suspend stay reliable, but crash #3 happened without any suspend |
| Disabling runtime D3 (`power/control=on` udev rule) | Did not prevent crash #3 |
| `nvidia-open-dkms` rebuild instead of precompiled `nvidia-open` | Same version, same crash |
| Forcing software GL for Steam | Still crashed (the crash is in Steam's own GL query, not rendering) |
| Moving the compositor to the iGPU (`AQ_DRM_DEVICES` reordered) | **Made it worse** — caused a full GPU hang requiring cold boot. Never do this on nvidia + hybrid laptops |

## 8. Notes

- Installing Steam / lib32 packages was a red herring at first
- `steam -srt-logger-opened` segfaults left core dumps readable via
  `coredumpctl` + `gdb` — that is how the NULL `strchr` was identified
- The iGPU is unaffected by all of this; video decode can still be offloaded
  to it (see the main README)
