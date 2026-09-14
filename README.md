# Aurora Kernel — alioth (4.19, Evolution X A16)

Current branch: `aurora-root` → builds for **Evolution X A16** (and any A16-based ROM on the PocoF3Releases `aosp-16` kernel line).

## What this is
- Base: [PocoF3Releases/kernel_xiaomi_sm8250](https://github.com/PocoF3Releases/kernel_xiaomi_sm8250) `aosp-16`
- Kept in sync by plain git merges with:
  - `https://github.com/LineageOS/android_kernel_xiaomi_sm8250` @ `lineage-23.2` (remote `lx`)
  - `https://github.com/LineageOS/android_kernel_xiaomi_sm8250` @ `lineage-20` (remote `lq`)
- Root stack: KernelSU-Next (raystef66/InfiniR design, `KSU_VERSION=12890`) + SUSFS v1.5.7 + Droidspaces config + e404 attributes.

## Critical build gotchas (read before touching)
1. **`CONFIG_KSU_LSM_SECURITY_HOOKS=y` must be present** — it is `default y` but only picked up if `.config` is regenerated AFTER the `drivers/kernelsu` port. A stale `out/.config` silently disables manager registration → "signature not found". Always run the `alioth_defconfig` target once after Kconfig changes.
2. **Exec hooks**: A16 bionic execs via `execveat`. `fs/exec.c` hooks `execve`, compat `execve`, AND a wrapper hook inside `do_execveat_common`. `drivers/kernelsu/sucompat.c` additionally accepts any path ending in `/su` (PATH-resolved execvpe).
3. **`CONFIG_KPROBES` stays OFF** — panics on this tree. All hooks are manual/in-tree LSM (the `ksu_hooks` SukiSU-4.19 patch: `exec/open/read_write/stat/input/pty`).
4. **SUS_MOUNT is OFF**: infinir's `namespace.c` targets pre-fs_context mounts and cannot merge into the `aosp-16` tree. Other SUSFS features (kstat, spoof uname/cmdline, magic mount) are on. `susfs_is_mnt_devname_ksu` lives at the end of `fs/namespace.c`, `ksu_is_zygote` rename applied in `core_hook.c`.
5. Manager: any KernelSU(-Next) APK signed with the expected cert works; `com.rifsxd.ksunext` (raystef's KSUN, cert `79e590…`, size `0x3e6`) matches the baked-in constants. `apk_sign.c` also accepts v3-only signing blocks.

## Build
```
PATH="$HOME/kbin:$PATH" make O=out ARCH=arm64 SUBARCH=arm64 \
  CC=clang AS=clang LD=ld.lld AR=llvm-ar NM=llvm-nm OBJCOPY=llvm-objcopy \
  STRIP=llvm-strip LLVM=1 LLVM_IAS=1 CROSS_COMPILE=aarch64-linux-gnu- \
  CROSS_COMPILE_ARM32=arm-linux-gnueabi- alioth_defconfig   # once, after Kconfig edits
PATH="$HOME/kbin:$PATH" make O=out ARCH=arm64 SUBARCH=arm64 ... -j16
```
Toolchain: proton-clang-13 (`kbin/ld` → host `ld` so kconfig builds work).

## Package (AK3, EvoX layout)
Boot image is header v3 with **raw `Image`** + **`dtb` inside boot** + separate **`dtbo.img`** partition. Zip `Image`, `dtb` (concat of `out/arch/arm64/boot/dts/**/*.dtb`), `dtbo.img` into AnyKernel3 (`is_slot_device=1`). `dtbo.img` via in-tree `scripts/ufdt/.../mkdtboimg.py` (py3-patched).

## Syncing with upstream
```
git fetch lx lineage-23.2; git merge lx/lineage-23.2   # resolve: keep device files ours
git fetch lq lineage-20;   git merge lq/lineage-20     # historically zero conflicts
# re-verify: drivers/kernelsu fs hooks still apply, defconfig KSU/SUSFS/DROIDSPACES lines intact
make ... alioth_defconfig && make ... -j16             # then package + flash-test
```

## EEVDF experiment (closed)
`d1aznr/kernel_xiaomi_alioth@eevdf` ships EEVDF on android14-common r0.111. Two findings:
1. The series does not rebase onto our aosp-16 Lineage base (the kernel/sched
   there expects the full common tree: tlb_migrate_finish, PF_PERF_CRITICAL, etc.).
2. Flashed the branch as-is (built with proton-13, BPF_JIT stub for their MODULES=n):
   boots to 30s, then A16 `netbpfload` fails: `BTF loading error: -22` →
   bpfloader `reboot_on_failure`. The common tree lacks CONFIG_DEBUG_INFO_BTF
   and the A16 bpf backports that Lineage aosp-16 carries.
=> EEVDF on this device requires the A16-userspace-compatible base we already
   use (or porting the whole bpf/BTF stack). Kept as CFS.
