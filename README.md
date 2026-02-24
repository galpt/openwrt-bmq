> 🇺🇸 [English](README.md) · 🇮🇩 [Indonesian](README_ID.md) · 🇨🇳 [Chinese](README_ZH.md) · 🇯🇵 [Japanese](README_JA.md)

# Adding the BMQ CPU Scheduler to OpenWrt

What happens when you take OpenWrt — the gold standard of open-source router
firmware — and swap its CPU scheduler for something completely different? That
is exactly what this repository is here to find out.

This is an experiment. We are replacing the vanilla Linux EEVDF (Earliest Eligible Virtual Deadline First) scheduler inside an OpenWrt x86/64 build with **BMQ (BitMap Queue)**, an
alternative CPU scheduler authored by Alfred Chen and maintained in the
[`linux-prjc`](https://gitlab.com/alfredchen/linux-prjc) kernel tree. The whole
thing is wired up with a GitHub Actions pipeline so every deliberate build
produces a ready-to-flash firmware image, no manual intervention required.

---

## Table of Contents
- [Status](#status)
- [Why Change the CPU Scheduler?](#why-change-the-cpu-scheduler)
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

## Why Change the CPU Scheduler?

Honest answer first: if you are running OpenWrt on a small embedded router with 128 MB of RAM and a single-core MIPS CPU, you probably do not need this. The default EEVDF (Earliest Eligible Virtual Deadline First) scheduler will serve you just fine.

But x86 hardware is a different story. When you have a mini PC or a repurposed desktop running OpenWrt, you start doing things that embedded routers were never meant to do — Docker containers, SQM with fq_codel or CAKE for traffic shaping, maybe a few self-hosted services on the side. Suddenly your router is a general-purpose computer that *also* happens to route packets, and the CPU is juggling a very mixed workload.

This is where an alternative scheduler like BMQ can quietly make a real difference. BMQ (BitMap Queue) was designed to keep latency-sensitive tasks responsive even when the CPU is pegged at 100%. Under heavy load, SQM algorithms like CAKE need to run their packet scheduling logic without getting starved by background tasks. Docker containers need their CPU slices delivered consistently. With the default EEVDF, all of these tasks compete in a way that can introduce jitter under pressure. BMQ handles that competition differently — its bitmap-based priority queuing tends to keep interactive and latency-sensitive workloads snappy even when something else is hammering the CPU in the background.

So why BMQ specifically and not something newer? Excellent question — there are plenty of interesting schedulers out there. BORE is getting a lot of attention, LAVD looks promising for interactive workloads, and LFBMQ is the direct evolution of BMQ itself. The honest reason we picked BMQ is that this is our *first attempt* at injecting a custom scheduler into an OpenWrt build, and we did not want to be adventurous with something we could not test thoroughly. BMQ has a long track record on the linux-prjc tree, the 6.6 LTS patches are mature and well-maintained, and it felt like the most responsible starting point.

There is also a hard constraint we cannot work around right now: OpenWrt v24.10.5 still ships with Linux 6.6.x. BORE, LAVD, and LFBMQ all require kernel 6.10 or newer to apply cleanly. Until OpenWrt bumps its kernel version, those options are simply out of reach. BMQ on 6.6 is what we have — and for the use case described above, it is a perfectly reasonable thing to experiment with.

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

### Option 1: Fresh Install (dd / QEMU)

Starting from scratch on bare metal? Grab the image from the Releases page and write it directly to a disk or USB stick.

```bash
# Decompress the image first
gunzip openwrt-*-squashfs-combined.img.gz

# Write to a disk or USB stick — double-check /dev/sdX before running this
dd if=openwrt-*-squashfs-combined.img of=/dev/sdX bs=4M status=progress
sync
```

Want to test it before touching any hardware? QEMU is your friend:

```bash
qemu-system-x86_64 \
  -drive file=openwrt-*-squashfs-combined.img,format=raw \
  -m 256m -nographic
```

### Option 2: Flash via LuCI Web UI (Sysupgrade)

Already running OpenWrt and want to upgrade to this BMQ build? You do not need physical access or `dd` — the LuCI web UI handles it cleanly.

Navigate to **LuCI → System → Backup / Flash Firmware**, scroll down to the **Flash new firmware image** section, and upload your file.

> [!IMPORTANT]
> **Upload the `.img.gz` file directly — do not decompress it first.** LuCI handles the gzip transparently.

**Picking the right file matters a lot. Use this table:**

| Your boot mode | File to upload |
|---|---|
| Legacy BIOS / MBR | `openwrt-x86-64-generic-squashfs-combined.img.gz` |
| UEFI / EFI | `openwrt-x86-64-generic-squashfs-combined-efi.img.gz` |

Not sure which boot mode your machine uses? Run this in your OpenWrt shell:

```bash
ls /sys/firmware/efi 2>/dev/null && echo "EFI" || echo "Legacy BIOS"
```

> [!WARNING]
> **Never use the `ext4` images for sysupgrade.** The ext4 variants do not support the sysupgrade path at all — they are for initial `dd` installs only. OpenWrt's sysupgrade mechanism depends on the SquashFS + overlayfs structure to save your configuration across upgrades. The ext4 format has no overlay filesystem, so flashing it through the web UI will either fail outright or leave your device in an unbootable state.

Once you have uploaded the right file, choose whether to **Keep settings** (leave this checked to preserve your existing config), click **Flash image**, verify the checksum on the confirmation screen, then click **Proceed**. The device reboots into the new BMQ build automatically.

### Verify BMQ is Active

Whether you did a fresh flash or a sysupgrade, confirm the scheduler is running after boot:

```bash
# Check active scheduler features
cat /sys/kernel/debug/sched/features

# Or look for scheduler init messages
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

- **Apt mirror selection** is done before `apt-get update` runs. The pipeline
  probes `archive.ubuntu.com`, `azure.archive.ubuntu.com`, and
  `us.archive.ubuntu.com` with a 3-second `curl` timing check and rewrites
  the apt sources to point at whichever responds fastest. A short per-connection
  timeout (10 s) and 5 retries are also configured so a flaky host is abandoned
  quickly instead of hanging the job for minutes.

- **Build timeouts** in CI are set to 5 hours. A cold build with kernel
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
