# arch-hybrid-gpu

**Notes & tools for running Arch Linux on Intel + NVIDIA hybrid-GPU laptops.**

> Case study: Acer Nitro / TigerLake-H (Intel UHD iGPU + NVIDIA RTX 3050 Laptop GPU),
> Hyprland (uwsm), kernel 7.2.6.

Two things live here, both learned the hard way on the same machine:

| | |
|---|---|
| **[1. Offload apps to the iGPU](#1-offload-apps-to-the-igpu)** | `igpu-run` — make native-toolkit apps (Qt/GTK) render on the Intel iGPU instead of keeping the dGPU awake 24/7, **including the GLVND trap** that makes plain `DRI_PRIME` silently do nothing. |
| **[2. Fix NVIDIA GSP crashes](#2-fix-nvidia-gsp-crashes-xid-62)** | Repeated `Xid 62` ("GSP PMU has halted") hangs — and the non-obvious fallout: EGL/GLX silently break, which makes **Steam segfault on launch**. Fix: proprietary `nvidia-580xx` with `NVreg_EnableGpuFirmware=0`. |

---

## 1. Offload apps to the iGPU

On hybrid laptops the compositor (Hyprland) usually renders on the NVIDIA dGPU.
That means lightweight always-running apps — browser, Discord, Telegram, Spotify —
keep the dGPU awake 24/7 (~5–17 W idle, VRAM consumed) while the iGPU sits at
**0 ns render time** doing literally nothing.

This flips the workload split:

```
dGPU (NVIDIA) : compositor + games + CUDA/AI + heavy GPU apps
iGPU (Intel)  : native-toolkit apps (Qt/GTK), video decode, UI
```

### What's inside

| File | Purpose |
|---|---|
| `bin/igpu-run` | Wrapper: `igpu-run <app>` launches any app on the iGPU |

### The GLVND trap (read this!)

The NVIDIA EGL vendor (which owns the display) **intercepts every EGL device
index**. So plain `DRI_PRIME=...` silently does nothing for GL/EGL apps —
you end up on the dGPU anyway, with 0 ns of iGPU engine time.

`igpu-run` therefore does **three** things:

```bash
export DRI_PRIME="$IGPU_DEVICE"                # Mesa device selection
export __GLX_VENDOR_LIBRARY_NAME=mesa         # GLX → Mesa, not NVIDIA
export __EGL_VENDOR_LIBRARY_FILENAMES=...mesa # EGL → Mesa, not NVIDIA
export VK_LOADER_DRM_DEVICE_SELECT=0          # Vulkan → iGPU
```

Verified working:

```
$ igpu-run glxinfo -B | grep renderer
OpenGL renderer string: Mesa Intel(R) UHD Graphics (TGL GT1)
$ igpu-run eglinfo -B | grep vendor
EGL vendor string: Mesa Project
```

### ⚠️ Which apps actually offload

| App | Toolkit | Offloads? |
|---|---|---|
| Telegram | Qt (system EGL/Mesa) | ✅ **yes** — verified 2 s+ iGPU engine time, holds only `renderD128` |
| GTK apps (nautilus, gnome-*) | GTK4/GL (Mesa) | ✅ yes |
| mpv, games with Vulkan/GL via Mesa | Mesa | ✅ yes |
| **Chromium-family** (Brave, Chrome, Discord, Spotify/CEF) | **bundles its own ANGLE** (`libGLESv2.so` inside the app dir) | ❌ **no** — the bundled ANGLE ignores system EGL vendor env and picks the default device (dGPU) |

For Chromium-family apps, options are: keep them on the dGPU (fine), or run
them with `--use-angle=swiftshader` (software rendering — not recommended).

### Install & usage

```bash
git clone https://github.com/zidanefaqih/arch-hybrid-gpu.git
cd arch-hybrid-gpu
install -m 755 bin/igpu-run ~/.local/bin/
```

Then, to make an app always launch on the iGPU, override its original
`.desktop` file in `~/.local/share/applications/` (higher priority than
`/usr/share/applications/`) by wrapping `Exec=` with `igpu-run`:

```bash
cp /usr/share/applications/org.telegram.desktop.desktop \
   ~/.local/share/applications/
sed -i 's|^Exec=Telegram|Exec=igpu-run Telegram|' \
   ~/.local/share/applications/org.telegram.desktop.desktop
```

Same name, same icon — the launcher just works, no duplicate entries.

```bash
# ad-hoc
igpu-run telegram
igpu-run mpv video.mkv
```

Verify which GPU a running app actually *renders* on:

```bash
for fd in /proc/<pid>/fdinfo/*; do
  node=$(readlink /proc/<pid>/fd/$(basename $fd) 2>/dev/null)
  case "$node" in
    *renderD128) echo "iGPU: $(grep drm-engine-render $fd)";;
    *renderD129) echo "dGPU: $(grep drm-engine-render $fd)";;
  esac
done
```

### ⚠️ Do NOT do this

**Never move the compositor (Hyprland) to the iGPU** via
`AQ_DRM_DEVICES=/dev/dri/card1:/dev/dri/card0` (iGPU-first) on nvidia.

On this hardware that caused a full GPU hang requiring a cold boot
(`Xid 62` / `RmInitAdapter failed 0x62:0x55`).

App-level offload (this repo) is safe because the compositor never moves.

---

## 2. Fix NVIDIA GSP crashes (`Xid 62`)

The open kernel modules (`nvidia-open`) **require GSP firmware**. On this
laptop the GSP crashed three times with `Xid 62: PMU has halted`, each time
requiring a **cold power cycle**.

Worse, the hang silently breaks the GL stack: EGL falls back to llvmpipe and
Xwayland GLX stops creating contexts, so any app calling
`glGetString(GL_EXTENSIONS)` gets `NULL`. That is why **Steam segfaults on
launch** (upstream bug:
[steam-for-linux#13269](https://github.com/ValveSoftware/steam-for-linux/issues/13269) /
[#13627](https://github.com/ValveSoftware/steam-for-linux/issues/13627)) —
a symptom, not the cause.

### The fix

Use the proprietary 580xx modules (they still support the legacy RM init
path) and turn GSP off — driver README chapter *"44B. DISABLING GSP MODE"*:

```bash
paru -S nvidia-580xx-dkms nvidia-580xx-utils lib32-nvidia-580xx-utils
```

```bash
sudo tee /etc/modprobe.d/nvidia-pm-fix.conf <<'EOF'
# Disable GSP: the GSP firmware crashed repeatedly on this laptop with the
# open modules (Xid 62 "PMU has halted" -> Xid 154 -> Xid 16 -> RmInitAdapter
# failed 0x62:0x55), each time requiring a cold power cycle.
# The proprietary module can run without GSP (driver README 44B).
options nvidia NVreg_EnableGpuFirmware=0 NVreg_PreserveVideoMemoryAllocations=1 NVreg_TemporaryFilePath=/var/tmp
EOF

sudo mkinitcpio -P && sudo reboot
```

Verify:

```console
$ nvidia-smi -q | grep 'GSP Firmware Version'
    GSP Firmware Version                               : N/A
$ grep EnableGpuFirmware /proc/driver/nvidia/params
EnableGpuFirmware: 0
$ eglinfo -B | grep 'EGL vendor'
EGL vendor string: NVIDIA
```

**📖 Full write-up with kernel logs, the Steam crash backtrace, what did *not*
work, and a rollback recipe: [`docs/nvidia-crash-fix.md`](docs/nvidia-crash-fix.md)**

---

## License

MIT
