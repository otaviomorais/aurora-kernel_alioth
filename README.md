# Aurora Kernel - alioth (4.19, Evolution X A16)

Current branch: `aurora-root` — builds for **Evolution X A16** (and any A16-based ROM on the PocoF3Releases `aosp-16` kernel line).

## What this is
- Base: [PocoF3Releases/kernel_xiaomi_sm8250](https://github.com/PocoF3Releases/kernel_xiaomi_sm8250) `aosp-16`
- Kept in sync by plain git merges with:
  - `https://github.com/LineageOS/android_kernel_xiaomi_sm8250` @ `lineage-23.2` (remote `lx`)
  - `https://github.com/LineageOS/android_kernel_xiaomi_sm8250` @ `lineage-20` (remote `lq`)
- Root stack: KernelSU-Next (raystef66/InfiniR design, `KSU_VERSION=12890`) + SUSFS v1.5.7 + Droidspaces config + e404 attributes.
- MM: LE9EC working-set protection (le9-patch, hakavlad) + zRAM writeback (LZ4HC).

## Critical build gotchas (read before touching)
1. **`CONFIG_KSU_LSM_SECURITY_HOOKS=y` must be present** — it is `default y` but only picked up if `.config` is regenerated AFTER the `drivers/kernelsu` port. A stale `out/.config` silently disables manager registration → "signature not found". Always run the `alioth_defconfig` target once after Kconfig changes. The CI enforces this with a config gate (see Build).
2. **Exec hooks**: A16 bionic execs via `execveat`. `fs/exec.c` hooks `execve`, compat `execve`, AND a wrapper hook inside `do_execveat_common`. `drivers/kernelsu/sucompat.c` additionally accepts any path ending in `/su` (PATH-resolved execvpe).
3. **`CONFIG_KPROBES` stays OFF** — panics on this tree. All hooks are manual/in-tree LSM (the `ksu_hooks` SukiSU-4.19 patch: `exec/open/read_write/stat/input/pty`).
4. **SUSFS**: full set since v3.8, including `SUS_MOUNT` (fs_context port in `fs/namespace.c`: `alloc_vfsmnt` spoof / `clone` / `create` / `copy_mnt_ns` / loopback / mount auto), `sus_path`+`namei` hooks, `try_umount`, `open_redirect`, overlayfs kstat, `HIDE_KSU_SUSFS_SYMBOLS`. Any future upstream merge touching `fs/namei.c`, `fs/namespace.c`, `fs/exec.c`, `fs/read_write.c`, `fs/stat.c` must keep these hooks' semantics — a merge can compile fine and still break root/hiding silently.
5. Manager: any KernelSU(-Next) APK signed with the expected cert works; `com.rifsxd.ksunext` (raystef's KSUN, cert `79e590...`, size `0x3e6`) matches the baked-in constants. `apk_sign.c` also accepts v3-only signing blocks.
6. **EEVDF was evaluated and rejected** (d1aznr branch): boots ~30s but dies on A16 `netbpfload` (no BTF in `common-r0.111`). CFS stays.

## LE9EC (working-set protection)
Compiled defaults for the **6GB alioth** (in KB), set in `arch/arm64/configs/alioth_defconfig`:

| sysctl | default | role |
|---|---|---|
| `vm.anon_min_kbytes` | `204800` (200M) | hard floor: anon below this is never reclaimed (anti swap-thrash) |
| `vm.clean_low_kbytes` | `393216` (384M) | best-effort: keep clean pagecache around this target |
| `vm.clean_min_kbytes` | `131072` (128M) | hard floor for clean pagecache |

All three are runtime-tunable without recompiling — persist via any init/`sysctl` mechanism:
```sh
echo 204800 > /proc/sys/vm/anon_min_kbytes
echo 393216 > /proc/sys/vm/clean_low_kbytes
echo 131072 > /proc/sys/vm/clean_min_kbytes
```
Tuning method: `anon_min` = summed anon RSS of your 5–6 most-reused apps (`dumpsys meminfo`); raise `clean_low` in 64M steps while watching `pgpgin` in `/proc/vmstat` on app re-open; if LMK starts killing background apps too early, the floors are too high for the device's effective free RAM.

## PROC_CHILDREN / v3.14 data-wipe incident
v3.14 shipped a field-reported **data wipe**. `CONFIG_PROC_CHILDREN` was reverted, then **re-enabled deliberately** after re-testing; root cause is **still open**. Treat v3.14/v3.15 as test builds: flash on a device you can wipe. Releases carry warnings; do not mirror this branch into a ROM's default kernel until the cause is known.

## Build
CI: `.github/workflows/kernel-build.yml` (ZyC-Clang 16) builds `Image dtbs dtbo.img`, runs the **config gate** (KSU/SUSFS/LSM-hooks/LE9EC/PROC_CHILDREN/FHANDLE/LTO/CFI must be present in `out/.config` or the job fails), assembles the boot `dtb` (cat of `out/arch/arm64/boot/dts/vendor/qcom/*.dtb`, same recipe as `AndroidKernel.mk`), packages a **flashable AnyKernel3 zip** using the template from the pinned release tag (`AK3_TEMPLATE_TAG`, default `aurora-v3.13` — bumped via `workflow_dispatch` input) and uploads `AURORA-<sha>-EVOX.zip` + `kernel-config` as artifacts.

Local (same flags CI uses):
```
PATH="$HOME/kbin:$PATH" make O=out ARCH=arm64 SUBARCH=arm64 \
  CC=clang AS=clang LD=ld.lld AR=llvm-ar NM=llvm-nm OBJCOPY=llvm-objcopy \
  STRIP=llvm-strip LLVM=1 LLVM_IAS=1 CROSS_COMPILE=aarch64-linux-gnu- \
  CROSS_COMPILE_ARM32=arm-linux-gnueabi- alioth_defconfig   # once, after Kconfig edits
PATH="$HOME/kbin:$PATH" make O=out ARCH=arm64 SUBARCH=arm64 \
  CC=clang AS=clang LD=ld.lld AR=llvm-ar NM=llvm-nm OBJCOPY=llvm-objcopy \
  STRIP=llvm-strip LLVM=1 LLVM_IAS=1 CROSS_COMPILE=aarch64-linux-gnu- \
  CROSS_COMPILE_ARM32=arm-linux-gnueabi- -j16 Image dtbs dtbo.img
```
Toolchain: **ZyC-Clang 16** (promoted from proton-clang-13; Image 3.8% smaller, validated on `ci-clang16-test`).

## Package (AK3, EvoX layout)
Boot image is header v3 with **raw `Image`** + **`dtb` inside boot** + separate **`dtbo.img`** partition. Zip `Image`, `dtb` (concat of `out/arch/arm64/boot/dts/vendor/qcom/*.dtb`), `dtbo.img` into AnyKernel3 (`is_slot_device=1`). `dtbo.img` via `make dtbo.img` (in-tree py3-compatible `scripts/ufdt/.../mkdtboimg.py`). The CI job automates exactly this; the AK3 template (tools/, META-INF/, anykernel.sh) is not in-tree — it is pulled from the release asset pinned by `AK3_TEMPLATE_TAG`.

## Syncing with upstream
```
git fetch lx lineage-23.2; git merge lx/lineage-23.2   # resolve: keep device files ours
git fetch lq lineage-20;   git merge lq/lineage-20     # historically zero conflicts
# re-verify: drivers/kernelsu fs hooks still apply, defconfig KSU/SUSFS/DROIDSPACES lines intact
# then rebuild (CI config gate catches silently dropped configs) + flash-test
```

## Known security tradeoffs (intentional, documented)
- `CONFIG_UNMAP_KERNEL_AT_EL0` (KPTI) is off (perf).
- ksud self-extract kernel thread + relaxed v3-only manager APK signing (`apk_sign.c`).
- SUSFS hooks in core VFS paths for root hiding.
Revisit if this kernel ever ships outside personal/opt-in use.

## Releases
Flashable zips + changelogs: https://github.com/otaviomorais/aurora-kernel_alioth/releases
