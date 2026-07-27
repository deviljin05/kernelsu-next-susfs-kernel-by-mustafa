# Custom GKI Kernel Builder (Wild KSU + SUSFS + BBRv3)

GitHub Actions workflow to build a custom **Android GKI 5.10 (android12-5.10)** kernel with root hiding, root management, networking, memory, storage, and gaming-oriented tuning baked in.

## What's included in this build

| Feature | Detail |
|---|---|
| **Root** | KernelSU-Next (Wild fork, `dev-susfs` branch) |
| **Root Hiding** | SUSFS4KSU — path/mount/kstat spoofing, uname spoof, log enable, KSU symbol hiding, cmdline/bootconfig spoof, open-redirect, sus map |
| **Baseband Protection** | Baseband Guard (BBG) LSM — protects modem/bootloader partitions from malicious root modules (optional, default ON) |
| **TCP Congestion Control** | BBRv3 backport + BBR/CUBIC/BIC/Westwood/HTCP fallback options, FQ/FQ-CoDel/CAKE qdiscs |
| **ZRAM** | LZ4HC compression |
| **IO Scheduler** | `none` only (deadline/kyber/BFQ disabled) |
| **Memory Management** | KSM, ZRAM writeback, compaction, migration, page reporting, tuned dirty ratios, swappiness 30 |
| **Storage** | F2FS (LZ4/LZ4HC/ZSTD compression) + EXT4 |
| **NTSync** | Wine/Proton sync primitives support |
| **Scheduler/Gaming** | 300Hz tick, SCHED_MC, schedutil governor + custom **Zenith** governor (aggressive up-scale / debounced down-scale) |
| **SuiTune driver** | Custom in-kernel misc driver: 60s boot-time CPU boost, screen-off min-freq cap, and touch-input boost (250ms MIN freq_qos bump) |

## Inputs (workflow_dispatch)

| Input | Required | Default | Meaning |
|---|---|---|---|
| `os_patch_level` | Yes | `lts` | Kernel branch tag to track (`lts` = rolling, always current) |
| `susfs_commit` | No | *(empty)* | Specific SUSFS commit hash. Leave blank to use the latest commit on the `gki-android12-5.10` branch |
| `device_name` | No | `gki-custom` | Label used for artifact/zip naming |
| `enable_bbg` | Yes | `true` | Enable/disable the Baseband Guard LSM |

## Build Pipeline (high-level steps)

1. **Setup** — install dependencies, set up the `repo` tool, configure ccache
2. **Source sync** — sync GKI android12-5.10 kernel from Google's manifest via `repo sync`
3. **SUSFS integration**
   - Clone SUSFS4KSU, copy configs + patches
   - Apply sublevel-based compatibility fixes → apply SUSFS patch → revert fixes
   - Fix `set_nameidata` argument mismatch
   - Apply `show_pad` fix for older sublevels
4. **KernelSU-Next setup** — Wild fork's setup script + static.patch fix
5. **Baseband Guard** (if enabled) — setup, register LSM in Kconfig, verify, enable config
6. **BBRv3** — backport from WildKernels patches, check `proc_dou8vec_minmax` sysctl dependency
7. **Networking, ZRAM, IO scheduler, memory, storage configs** — appended to defconfig
8. **NTSync patches** — lockdep fix (older sublevels) + compat + base patch
9. **Gaming configs** — 300Hz HZ, SCHED_MC, schedutil
10. **Zenith governor** — writes a custom `cpufreq_zenith.c` driver and wires it in (registers in Kconfig + Makefile)
11. **SuiTune driver** — writes a custom `suitune.c` misc driver (boot boost, screen-off cap, touch boost) and registers it
12. **Cleanup/branding** — clean dirty flag, set custom local-version string (`-BLAZE-KSUNv1-STABLE`), disable strict defconfig check
13. **Build** — `build/build.sh` with `LTO=thin`, ccache enabled
14. **Package** — built `Image` is packaged into an AnyKernel3 (`gki-2.0` branch) flashable zip
15. **Upload** — uploaded as a GitHub Actions artifact (14-day retention)

## Output

You'll get a flashable ZIP named like:
```
<device_name>-Blaze-android12-5.10-susfs<commit8>.zip
```
This is an AnyKernel3-based zip — flashable from any custom recovery (TWRP/OrangeFox) or a fastboot-based flasher.

## Requirements / Notes

- Target: **GKI android12-5.10** kernels only (configs/patches won't match other GKI branches)
- Runner: `ubuntu-22.04`, 150-minute timeout
- Some patches (BBRv3, static.patch, ntsync) are kept non-fatal with `|| echo warning` — if they fail, the build continues but that feature may be missing/partial. Always check the Actions log for warnings.
- Leaving `susfs_commit` blank means the latest SUSFS commit is used — pin a specific commit hash if you need a reproducible build.

## Disclaimer

A custom kernel with root hiding and baseband protection can void your device warranty, and misuse can brick your device. Only flash on your own device, with full understanding, and at your own risk.
