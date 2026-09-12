# Aurora Kernel — Xiaomi Poco F3 / Redmi K40 (alioth)

Custom kernel 4.19 for **alioth** (Poco F3, POCO F3 Pro/aliothin, Redmi K40) SM8250,
built for **Android 13–16** AOSP-based ROMs (crDroid 12 validated) and maintained
with regular security syncs from the LineageOS 4.19 msm-8250 trees.

## Base & Credits

This is a fork of [raystef66/InfiniR_kernel_alioth](https://github.com/raystef66/InfiniR_kernel_alioth)
(`16.0-alioth`, Feb/2026 state — itself based on LineageOS `android_kernel_xiaomi_sm8250`
lineage-23.0 / `android_kernel_qcom_sm8250`), kept in-tree for full history and GPLv2 credit.

Upstream thanks: @raystef66 (E404/InfiniR, KernelSU+SUSFS integration), LineageOS,
CIP 4.19 stable, Qualcomm ISA.

## What Aurora adds on top

- `fs/ext4` + `fs/mbcache` synced to LineageOS `lineage-20` tip (Sep/2026):
  `EXT4_EX_NOFAIL` ENOSPC fixes, extensible xattr-cache (reusable entries),
  fs-verity and extents/inline hardening.
- `fs/eventpoll` synced to tip (epoll fixes, adapted to the in-tree net API).
- qcom/lineage May→Sep 2026 security batch (1146 files): locking/rtmutex,
  netfilter-core bits, binder_alloc fixes, misc `UPSTREAM:` backports.
- Build reproducible with **Proton-Clang 13 (20210522)** or the AOSP clang used
  by the upstream CI; flags auto-adapted (regalloc advisor / mcpu guards).
  See `tools-build/` notes below.

## Building

```bash
# toolchain: proton-clang-20210522 OR aosp-clang (>=14 preferred by some scripts;
# proton-13 works with the compat tweaks in this tree)
export PATH="$HOME/kbin:$PATH"   # clang, ld.lld, aarch64-linux-gnu-*, arm-linux-gnueabi-*

make O=out ARCH=arm64 SUBARCH=arm64 CC=clang AS=clang LD=ld.lld AR=llvm-ar \
     NM=llvm-nm OBJCOPY=llvm-objcopy STRIP=llvm-strip LLVM=1 LLVM_IAS=1 \
     CROSS_COMPILE=aarch64-linux-gnu- CROSS_COMPILE_ARM32=arm-linux-gnueabi- \
     vendor/alioth_defconfig

make O=out ...same flags... -j$(nproc)
```

Outputs (format used by the flashable package):

- `out/arch/arm64/boot/Image.gz-dtb` → `ALIOTH-Image` / `Image.gz-dtb`
- `out/arch/arm64/boot/dtb` → written into **vendor_boot** (AK3 auto)
- `out/arch/arm64/boot/dtbo.img` → written into **dtbo** (AK3 auto)

> Important for A/B devices on Android 13+: the DTB lives in `vendor_boot` and
> the overlays in `dtbo`. Flashing only a boot-patched Image without the matching
> `dtb`/`dtbo` breaks GPU/display probing. Use the packaged AnyKernel zip
> (raystef66 AK3 layout, `is_slot_device=1`) which handles this automatically.

## Flashing

Via TWRP/OrangeFox: flash the `E404R-BPF-ALIOTH-*.zip` (or renamed `AURORA-*.zip`).
Via fastboot: flash `boot`, `vendor_boot` (dtb) and `dtbo` images from the same build.

## Root

KernelSU + SUSFS are pre-integrated (config `CONFIG_KSU=y`, `CONFIG_KSU_SUSFS=y`).
Use any KernelSU/SukiSU manager. `CONFIG_KALLSYMS` is off upstream for stealth;
enable in the defconfig if you need symbol-based debugging.

## Maintenance policy (how Aurora stays current)

Upstream refs kept as git remotes: `lx` = LineageOS/android_kernel_xiaomi_sm8250,
`lq` = LineageOS/android_kernel_qcom_sm8250. Sync cadence: per upstream merge
batch (roughly monthly CIP/qcom batches).

- **Syncable by directory replacement (current flow):** `fs/ext4`, `fs/mbcache`,
  `fs/eventpoll`, misc self-contained `UPSTREAM:` patches.
- **Blocked pending full ecosystem rebase (do not attempt piecemeal):**
  `net/` + `include/net` (needs lockdep/rwsem/xfrm/uapi-bpf set), `mm/`
  (needs locking + pgtable refactor), `drivers/android/binder` (dbitmap rework,
  removes `simple_lmk` used by this device), `clone3` series (conflicts with
  the 4.19 `kernel/fork.c` internals that the KSU hooks rely on).
- When an upstream maintainer (or Aurora) performs the full-tree ecosystem
  rebase, the blocked groups get enabled here.

## License

GPLv2. See `LICENSES/`.
