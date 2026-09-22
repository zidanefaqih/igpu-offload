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
iGPU (Intel)  : browser, Discord, Telegram, Spotify (UI + video decode)
```

Result: dGPU can drop to low power when you launch a game, VRAM stays free,
and the iGPU finally earns its keep. Video decode (AV1/VP9/H.264) on TigerLake
iGPU is excellent, so YouTube in Brave is actually *better* on the iGPU.

## What's inside

| File | Purpose |
|---|---|
| `bin/igpu-run` | Wrapper: `igpu-run <app>` launches any app on the iGPU (`DRI_PRIME`) |
| `desktop/*.desktop` | App launchers that auto-offload (Brave, Discord, Telegram, Spotify) |
| `docs/nvidia-crash-fix.md` | Full write-up: diagnosing & fixing repeated GSP crashes (Xid 62 → 154 → 44, `RmInitAdapter failed 0x62:0x55`) caused by suspend/resume + runtime D3 on nvidia-open |

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
# any app, ad-hoc:
igpu-run brave
igpu-run discord

# or just use the .desktop launchers (they call igpu-run internally)
```

Verify which GPU a running app uses:

```bash
# list render nodes an app holds (r128 = iGPU, r129 = dGPU on this laptop)
ls -l /proc/$(pgrep -x brave | head -1)/fd | grep -o "renderD[0-9]*" | sort -u

# check iGPU is actually rendering (engine time should be > 0ns)
cat /proc/<pid>/fdinfo/<fd> | grep drm-engine-render
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
