# Aurora Kernel — Xiaomi Poco F3 / Redmi K40 (alioth)

Custom 4.19 kernel for **alioth** (Poco F3 / POCO F3 Pro "aliothin" / Redmi K40, SM8250),
built for **Evolution X A16** (validated) and Android 13–16 AOSP-based ROMs on the same
lineage-4.19 base. Ships **KernelSU-Next + SUSFS** and **Droidspaces** support, kept in
lock-step with the LineageOS 4.19 msm-8250 trees.

## Branches

| Branch | Base | Status |
|---|---|---|
| `aurora-v3` | EvoX/`PocoF3Releases` `aosp-16` kernel + full LineageOS sync (`lineage-23.2` Jul/2026 + `qcom lineage-20` **Sep/2026 batch**: clone3, fdtable, rtmutex, net, mm) + KSU/SUSFS/Droidspaces | **current** — v3.1 release |
| `main` | `raystef66/InfiniR_kernel_alioth` (Feb/2026) + partial LineageOS syncs (ext4/mbcache/eventpoll + Sep batch via content-patch) | legacy (v1.0), kept for history |

## What the v3 branch includes

- **Up-to-date with LineageOS**: zero commits missing vs `LineageOS/android_kernel_qcom_sm8250@lineage-20` (tip 09-Sep-2026) and `@android_kernel_xiaomi_sm8250 lineage-23.2` (tip 13-Jul-2026).
- **KernelSU-Next + SUSFS v1.5.7** — all hooks (mount, path, stat, kstat, fd, statfs, cmdline, proc_namespace, open_redirect ready) adapted to the `fs_context` mount API.
- **Droidspaces** — full non-GKI config set (namespaces, cgroups, seccomp, NAT/veth/bridge/nft/iptables stack) + upstream cgroup-prefix fix. Verified: `droidspaces check` green.
- EvoX device parity: same source layout, defconfig, LTO (ThinLTO) + CFI as shipped by the ROM.

## Building

Toolchain: proton-clang 13 (20210522) — or AOSP clang `r416183b` used by the org CI.
Compat tweaks already in-tree: `scripts/as-version.sh` (clang ≤13 fallback), in-tree
`scripts/ufdt/.../mkdtboimg.py` (python3), `-D__KERNEL__` ordering.

```bash
make O=out ARCH=arm64 SUBARCH=arm64 CC=clang AS=clang LD=ld.lld AR=llvm-ar \
     NM=llvm-nm OBJCOPY=llvm-objcopy STRIP=llvm-strip LLVM=1 LLVM_IAS=1 \
     CROSS_COMPILE=aarch64-linux-gnu- CROSS_COMPILE_ARM32=arm-linux-gnueabi- \
     alioth_defconfig
make O=out …same flags… -j$(nproc)
```

Artifacts consumed by the flashable package: `out/arch/arm64/boot/Image`,
`dtb` (written into **vendor_boot** — header v3 devices), `dtbo.img`.

## Flashing

AnyKernel3 zip in releases (`AURORA-vX.Y-EVOX.zip`): TWRP/OrangeFox → Install → reboot.
It writes boot, vendor_boot (dtb) and dtbo — **all three must stay in sync with the
kernel tree**; flashing a kernel built from a different DT source is the classic
silent no-boot on this platform.

## Maintenance pipeline (how Aurora stays on tip)

```bash
git fetch lx lineage-23.2 lq lineage-20          # LineageOS tips
git merge lq/lineage-20                          # usually zero conflicts
git merge lx/lineage-23.2                        # resolve device dirs to ours/org
# SUSFS/KSU layer: re-inject hooks on any touched fs/ file (see fs/namespace.c,
# fs/proc/task_mmu.c patterns guarded by CONFIG_KSU_SUSFS*)
# droidspaces: keep defconfig block (Documentation: ravindu644/Droidspaces-OSS)
# build → flash-test → tag + release
```

## Credits

GPLv2. Base source: [PocoF3Releases/kernel_xiaomi_sm8250](https://github.com/PocoF3Releases/kernel_xiaomi_sm8250) (EvoX A16 kernel) ← [LineageOS](https://github.com/LineageOS). KSU/SUSFS layer: [raystef66/InfiniR_kernel_alioth](https://github.com/raystef66/InfiniR_kernel_alioth) + [tiann/KernelSU](https://github.com/tiann/KernelSU) / [simonpunk/susfs4ksu](https://github.com/sidex15/susfs4ksu-module). Droidspaces guide: [ravindu644/Droidspaces-OSS](https://github.com/ravindu644/Droidspaces-OSS).
