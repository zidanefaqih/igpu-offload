# igpu-offload

**Offload lightweight apps to the Intel iGPU on Arch Linux + Hyprland hybrid-GPU laptops (Intel iGPU + NVIDIA dGPU).**

> Case study: Acer Nitro / TigerLake-H (Intel UHD) + NVIDIA RTX 3050 Mobile.

## Why

On hybrid laptops the compositor (Hyprland) usually renders on the NVIDIA dGPU.
That means lightweight always-running apps — browser, Discord, Telegram, Spotify —
keep the dGPU awake 24/7 (~5–17 W idle, VRAM consumed) while the iGPU sits at
**0 ns render time** doing literally nothing.

This repo flips the workload split:

```
dGPU (NVIDIA) : compositor + games + CUDA/AI + heavy GPU apps
iGPU (Intel)  : native-toolkit apps (Qt/GTK), video decode, UI
```

Result: dGPU can drop to low power when you launch a game, VRAM stays free,
and the iGPU finally earns its keep.

## What's inside

| File | Purpose |
|---|---|
| `bin/igpu-run` | Wrapper: `igpu-run <app>` launches any app on the iGPU |
| `desktop/*.desktop` | App launchers that auto-offload (Brave, Discord, Telegram, Spotify) |
| `docs/nvidia-crash-fix.md` | Full write-up: diagnosing & fixing repeated GSP crashes (Xid 62 → 154 → 44, `RmInitAdapter failed 0x62:0x55`) caused by suspend/resume + runtime D3 on nvidia-open |

## The GLVND trap (read this!)

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

## ⚠️ Which apps actually offload

| App | Toolkit | Offloads? |
|---|---|---|
| Telegram | Qt (system EGL/Mesa) | ✅ **yes** — verified 350 ms iGPU engine time, holds only `renderD128` |
| GTK apps (nautilus, gnome-*) | GTK4/GL (Mesa) | ✅ yes |
| mpv, games with Vulkan/GL via Mesa | Mesa | ✅ yes |
| **Chromium-family** (Brave, Chrome, Discord, Spotify/CEF) | **bundles its own ANGLE** (`libGLESv2.so` inside the app dir) | ❌ **no** — the bundled ANGLE ignores system EGL vendor env and picks the default device (dGPU) |

For Chromium-family apps, options are: keep them on the dGPU (fine), or run
them with `--use-angle=swiftshader` (software rendering — not recommended).

## Install

```bash
git clone https://github.com/zidanefaqih/igpu-offload.git
cd igpu-offload
install -m 755 bin/igpu-run ~/.local/bin/
cp desktop/*.desktop ~/.local/share/applications/
update-desktop-database ~/.local/share/applications/ 2>/dev/null
```

## Usage

```bash
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

## ⚠️ Do NOT do this

**Never move the compositor (Hyprland) to the iGPU** via
`AQ_DRM_DEVICES=/dev/dri/card1:/dev/dri/card0` (iGPU-first) on nvidia-open.

On this hardware that caused repeated full GPU crashes:

```
Xid 62 (GSP/PMU halted) → Xid 154 (PF FLR) → Xid 44 (MMU fault)
→ external display drops → RmInitAdapter failed (0x62:0x55)
→ requires cold power cycle
```

App-level offload (this repo) is safe because the compositor never moves.
Full story and the PM fix: [`docs/nvidia-crash-fix.md`](docs/nvidia-crash-fix.md).

## License

MIT
