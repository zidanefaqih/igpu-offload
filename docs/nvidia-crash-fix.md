# Fix: repeated NVIDIA GSP crashes after suspend/resume (Xid 62 → 154 → 44)

Laptop: Acer Nitro (TigerLake-H i7, Intel UHD iGPU + NVIDIA RTX 3050 Mobile GA107)
OS: Arch Linux, kernel 7.2.6-arch2-1, Hyprland (uwsm), driver `nvidia-open 615.71.09`

## Symptoms

Twice in one day, with an identical pattern:

1. External monitor (HDMI, wired to the dGPU) suddenly disappears from DRM
2. GPU hangs; `nvidia-smi` prints `ERR!` for Fan/Power/ECC
3. After module reload: `kgspWaitForGfwBootOk failed 0x55`, `RmInitAdapter failed (0x62:0x55:2130)`
4. Only a **cold power cycle** recovers it (warm reboot sometimes not enough)

## Kernel log timeline (both crashes)

```
suspend entry (deep) → resume
   ↓ (minutes to hours later)
Xid 62  : internal error — "Received signal from GSP that PMU has halted"
Xid 154 : "GPU recovery action changed from 0x0 (None) to 0x1 (PF FLR)"
   ↓
nvidia-modeset: Failure reading maximum pixel clock for HDMI-0
Xid 44  : MMU Fault: ENGINE GRAPHICS ... FAULT_PDE ACCESS_TYPE_VIRT_WRITE
   ↓
NVRM: Reset required [NV_ERR_RESET_REQUIRED] 0x62
NVRM: kgspWaitForGfwBootOk_TU102: failed ... 0x55 (progress 0x9)
NVRM: RmInitAdapter: Cannot initialize GSP firmware RM
NVRM: RmInitAdapter failed! (0x62:0x55:2130)
   ↓ reload modules
NVRM: kgspExecuteFwsec_TU102: failed to execute FWSEC for SB
NVRM: kflcnWaitForHalt_TU102: Timeout waiting for Falcon to halt
   ↓
GPU requires full power cycle
```

## Root cause

`nvidia-open` **always runs the GPU on GSP firmware** (the closed-source RM is
replaced by a firmware running on the GSP RISC-V core). On this
RTX 3050 Mobile + kernel combination, GSP does not survive
**suspend/resume cycles combined with runtime D3 (fine-grained power
gating)**:

- `power/control = auto` → GPU enters runtime D3 during idle
- suspend/resume wakes it mid-state
- GSP PMU halts → Xid 62 → recovery fails → adapter init fails permanently

The trigger was **not** Steam / lib32 packages (crash #1 happened before
Steam was installed).

## The fix (what worked)

Since proprietary `nvidia-dkms` is no longer packaged (open-only era), we keep
`nvidia-open` and remove the two triggers:

### 1. Disable runtime D3 for the dGPU

`/etc/udev/rules.d/80-nvidia-pm.rules`:

```
# Prevent NVIDIA GPU from entering runtime D3 (fine-grained) — causes Xid 62
# after suspend/resume cycles on this laptop.
ACTION=="add", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x03[0-9]*", ATTR{power/control}="on"
```

### 2. Preserve video memory across suspend

`/etc/modprobe.d/nvidia-pm-fix.conf`:

```
options nvidia NVreg_PreserveVideoMemoryAllocations=1 NVreg_TemporaryFilePath=/var/tmp
```

### 3. Enable NVIDIA suspend/resume services

```bash
sudo systemctl enable nvidia-suspend.service nvidia-resume.service
sudo mkinitcpio -P
```

Apply immediately without reboot:

```bash
echo on | sudo tee /sys/bus/pci/devices/0000:01:00.0/power/control
```

### Trade-offs

- dGPU idles at ~17 W instead of ~0 W (runtime D3 disabled)
- Suspend takes a little longer (VRAM snapshot to `/var/tmp`)
- Cost: ~2–4 GB disk during suspend

### If it still crashes

- Try the AUR legacy driver: `paru -S nvidia-580xx-dkms` (proprietary-mode,
  allows disabling GSP with `NVreg_EnableGpuFirm=0`)
- Check for a driver downgrade in the pacman cache
- Always cold-boot (full power off) after a hang — warm reboot may not
  reset the dGPU

## Notes

- Installing Steam + lib32 packages before crash #2 was a red herring
- Reload nvidia modules only from a TTY with the compositor stopped —
  reloading while Hyprland is running kills all displays
