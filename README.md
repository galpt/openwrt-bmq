# Adding the BMQ CPU Scheduler to OpenWrt

What happens when you take OpenWrt — the gold standard of open-source router
firmware — and swap its CPU scheduler for something completely different? That
is exactly what this repository is here to find out.

This is an experiment. We are replacing the vanilla Linux CFS (Completely Fair
Scheduler) inside an OpenWrt x86/64 build with **BMQ (BitMap Queue)**, an
alternative CPU scheduler authored by Alfred Chen and maintained in the
[`linux-prjc`](https://gitlab.com/alfredchen/linux-prjc) kernel tree. The whole
thing is wired up with a GitHub Actions pipeline so every deliberate build
produces a ready-to-flash firmware image, no manual intervention required.

---

## Table of Contents
- [Status](#status)
- [Features](#features)
- [Requirements](#requirements)
- [Local Build](#local-build)
- [CI Pipeline](#ci-pipeline)
- [Artifact Release](#artifact-release)
- [Usage](#usage)
- [Design Notes](#design-notes)
- [Contributing](#contributing)
- [License](#license)

## Status

**Current Branch**: `main`

> [!NOTE]
> This is an active experiment. The CI pipeline targets **OpenWrt v24.10.5** (the latest stable release) with **Linux 6.6.x** on the `x86/64` platform.
> Builds are triggered manually to keep things deliberate.

## Features

- Pins to **OpenWrt v24.10.5** — no surprises from rolling upstream changes.
- Uses `CONFIG_KERNEL_GIT_CLONE_URI` to pull the kernel directly from the
  `linux-6.6.y-prjc-lts` branch of `linux-prjc` — the official OpenWrt
  mechanism for custom kernels, no tarball hacks needed.
- Enables BMQ via a kernel config fragment appended to
  `target/linux/x86/config-6.6` (`CONFIG_SCHED_ALT=y` + `CONFIG_SCHED_BMQ=y`).
- Supports dropping extra `*.patch` files into `bmq-patches/` for ad-hoc
  kernel customisation without touching the workflow.
- Produces `squashfs` and `ext4` combined images (both BIOS and EFI variants)
  as `.img.gz` artifacts.
- Publishes every successful build to GitHub Releases with a `SHA256SUMS.txt`.
- Build log is always uploaded as an artifact — even on failure — so there is
  something to look at when things go wrong.

## Requirements

You will need a Debian/Ubuntu machine with the standard OpenWrt build
dependencies installed:

```bash
sudo apt-get update
sudo apt-get install -y --no-install-recommends \
  build-essential ccache clang flex bison g++ gawk \
  gcc-multilib g++-multilib gettext git libfuse-dev \
  libncurses5-dev libssl-dev python3-setuptools \
  rsync subversion swig unzip util-linux wget \
  xsltproc zlib1g-dev file python3 python3-pip \
  squashfs-tools xz-utils quilt
```

Other distributions should provide equivalent packages. The CI runner uses
`ubuntu-latest`.

## Local Build

The steps below mirror exactly what the CI pipeline does. If it builds
locally, it will build in CI.

1. **Clone this repository**
   ```bash
   git clone https://github.com/galpt/openwrt-bmq.git
   cd openwrt-bmq
   ```

2. **Clone OpenWrt v24.10.5**
   ```bash
   git clone --depth 1 --branch v24.10.5 \
     https://github.com/openwrt/openwrt.git openwrt
   ```

3. **Install feeds**
   ```bash
   cd openwrt
   ./scripts/feeds update -a
   ./scripts/feeds install -a
   ```

4. **Write the build config**

   The key here is `CONFIG_KERNEL_GIT_CLONE_URI`. When this is set, OpenWrt
   clones the kernel from the given git URL instead of downloading its own
   tarball, and the hash check is bypassed automatically.

   ```bash
   # inside the openwrt/ directory
   printf '%s\n' \
     'CONFIG_TARGET_x86=y' \
     'CONFIG_TARGET_x86_64=y' \
     'CONFIG_TARGET_MULTI_PROFILE=y' \
     'CONFIG_TARGET_DEVICE_x86_64_DEVICE_generic=y' \
     'CONFIG_KERNEL_GIT_CLONE_URI="https://gitlab.com/alfredchen/linux-prjc.git"' \
     'CONFIG_KERNEL_GIT_REF="linux-6.6.y-prjc-lts"' \
     > .config

   make defconfig

   # Append BMQ scheduler options to the x86 kernel config fragment
   printf '\n# BMQ (BitMap Queue) CPU Scheduler\nCONFIG_SCHED_ALT=y\nCONFIG_SCHED_BMQ=y\n# CONFIG_SCHED_PDS is not set\n' \
     >> target/linux/x86/config-6.6
   ```

5. **Add optional patches** (skip if you have none):
   Place any `*.patch` files under `bmq-patches/` at the root of this repo.
   They get copied into `openwrt/target/linux/x86/patches-6.6/` automatically.

6. **Build**
   ```bash
   # inside openwrt/
   make -j$(nproc) V=s
   # fall back if the parallel build races
   make -j1 V=s
   ```

7. **Collect the images**
   ```bash
   mkdir -p ../dist
   cp bin/targets/x86/64/*squashfs-combined*.img.gz ../dist/
   cp bin/targets/x86/64/*ext4-combined*.img.gz     ../dist/
   ls -lh ../dist
   ```

> [!TIP]
> OpenWrt compresses all output images with gzip. The files you are looking for are `*.img.gz`, not bare `*.img`.

## CI Pipeline

The workflow lives in `.github/workflows/build-and-release.yml` and is
**manually triggered only** (`workflow_dispatch`). We made this choice
deliberately — every build is a conscious decision, not an automatic reaction
to a commit.

Here is the high-level sequence:

1. Install build dependencies on a fresh `ubuntu-latest` runner.
2. Clone OpenWrt v24.10.5.
3. Update and install all feeds.
4. Write `.config` with `CONFIG_KERNEL_GIT_CLONE_URI` pointing to `linux-prjc`,
   run `make defconfig`, and append the BMQ Kconfig options.
5. Copy any patches from `bmq-patches/` if present.
6. Run `make -j$(nproc) V=s`, falling back to `-j1` on failure.
7. Collect `*squashfs-combined*.img.gz` and `*ext4-combined*.img.gz` from the
   output directory.
8. Pass the images to the `create-release` job which publishes them as a
   timestamped GitHub Release.

The build log is always uploaded as an artifact — even when the build fails —
so there is always something to read when debugging.

> [!NOTE]
> The release tag is derived from the Git ref if you push a tag, or auto-generated as `openwrt-bmq-YYYYMMDD-HHmmss` when triggered manually against a branch.

## Artifact Release

Every successful run publishes a GitHub Release containing:

| File | Boot | Root filesystem |
|------|------|-----------------|
| `*squashfs-combined.img.gz`     | Legacy BIOS | squashfs — read-only |
| `*squashfs-combined-efi.img.gz` | UEFI        | squashfs — read-only |
| `*ext4-combined.img.gz`         | Legacy BIOS | ext4 — writable      |
| `*ext4-combined-efi.img.gz`     | UEFI        | ext4 — writable      |
| `SHA256SUMS.txt`                | —           | checksums for all of the above |

The squashfs variants are the recommended choice for most use cases — they are
smaller, read-only by design, and behave exactly like a typical OpenWrt flash
image.

## Usage

Once you have grabbed an image from the Releases page, here is how to put it
to use:

```bash
# Decompress
gunzip openwrt-*-squashfs-combined.img.gz

# Write to a USB stick or disk — replace /dev/sdX carefully
dd if=openwrt-*-squashfs-combined.img of=/dev/sdX bs=4M status=progress
sync

# Or skip the hardware and test immediately with QEMU
qemu-system-x86_64 \
  -drive file=openwrt-*-squashfs-combined.img,format=raw \
  -m 256m -nographic
```

Once it boots you land in a standard OpenWrt shell. Package management,
network configuration, and everything else works the same as upstream — the
only difference is the CPU scheduler running underneath.

```bash
# Verify BMQ is active after booting
cat /sys/kernel/debug/sched/features
# or check dmesg for scheduler init messages
dmesg | grep -i sched
```

## Design Notes

- **Why `CONFIG_KERNEL_GIT_CLONE_URI` and not a tarball?** OpenWrt verifies
  the SHA256 of every kernel tarball it downloads. Renaming a custom tarball
  to match the expected filename does not fool the hash check — the build
  silently downloads the real kernel anyway and BMQ never gets compiled in.
  `CONFIG_KERNEL_GIT_CLONE_URI` is the mechanism OpenWrt officially supports
  for this exact use case, and it disables the hash check by design.

- **Why Linux 6.6?** The `linux-prjc` BMQ patches are maintained for the
  6.6 LTS kernel family (`linux-6.6.y-prjc-lts`). OpenWrt v24.10.5 targets
  `KERNEL_PATCHVER=6.6` for x86, which makes them a natural fit. Newer kernel
  versions do not yet have an actively maintained BMQ port.

- **Why x86/64?** It is the easiest platform to test — you can spin up a QEMU
  VM in seconds without needing physical hardware. The same approach could, in
  theory, be adapted for other architectures if the scheduler patches are
  compatible.

- **The `bmq-patches/` directory** is intentionally left empty in the
  repository. Drop your own `*.patch` files there if you want to layer
  additional changes on top of `linux-prjc` without modifying the workflow.

- **Build timeouts** in CI are set to 12 hours. A cold build with kernel
  compilation can take a long time on a shared GitHub runner.

## Contributing

This is an experiment and contributions are very welcome — patches, ideas,
bug reports, all of it.

1. Fork the repository and create a feature branch.
2. If you are adding kernel patches, place them in `bmq-patches/` and verify
   they apply cleanly against `linux-6.6.y-prjc-lts`.
3. If you are changing the workflow, test it with `workflow_dispatch` before
   opening a pull request.
4. Open the pull request and describe what you changed and why.

> [!NOTE]
> Please do not add triggers that auto-run the workflow on every push. Builds are expensive and we want them to be intentional.

## License

This project is provided under the **MIT License**. See the [LICENSE](LICENSE)
file for details.
