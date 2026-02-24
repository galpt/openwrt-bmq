# OpenWrt BMQ x86 Build Repository

This repository automates the process of building a custom OpenWrt firmware image for x86
aarch64 hardware using the **BMQ (BitMap Queue)** kernel tree.

Unlike the stock OpenWrt project, we pull a patched Linux kernel from the
[`linux-prjc`](https://gitlab.com/alfredchen/linux-prjc) repository and apply any
local BMQ (BitMap Queue) scheduler patches before compiling. The goal is to provide a reproducible,
continuous integration pipeline that produces fully‑built `squashfs` images ready
to flash or run under a virtual machine.

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
> This project is primarily an automation wrapper around the upstream OpenWrt
> build system.  The pipeline has been tested on Ubuntu 22.04 LTS and via
> GitHub Actions.

## Features

- Automatically clones the latest OpenWrt sources.
- Downloads and packages the `linux-6.6.y-prjc` kernel tree.
- Applies local `bmq-patches/` into the kernel.
- Builds `x86/64` images with `make defconfig` and parallel make.
- Continuous build and release using GitHub Actions.
- Releases images as GitHub Release assets with SHA256 checksums.

## Requirements

The following tools are required to perform a local build:

```bash
# Debian/Ubuntu
sudo apt update
sudo apt install -y \
  build-essential ccache git python3 python3-pip subversion \
  libncurses5-dev gawk gettext unzip rsync squashfs-tools libssl-dev \
  file xz-utils
```

Other Unix‑like distributions should provide equivalent packages.

## Local Build

1. **Clone the repository**
   ```bash
   git clone https://github.com/<your‑org>/openwrt-bmq.git
   cd openwrt-bmq
   ```

2. **Fetch OpenWrt sources**
   (the CI workflow performs a `--depth 1` clone by default):
   ```bash
   git clone --depth 1 https://git.openwrt.org/openwrt/openwrt.git openwrt
   ```

3. **Obtain the BMQ kernel tree**
   ```bash
   git clone --depth 1 --branch linux-6.6.y-prjc-lts \
     https://gitlab.com/alfredchen/linux-prjc.git linux-prjc-src
   mkdir -p openwrt/dl
   tar -C linux-prjc-src -cJf openwrt/dl/linux-6.6.119-prjc.tar.xz .
   ```

4. **Provide local patches** (optional):
   Place any `*.patch` files under `bmq-patches/` and they will be copied into
   `openwrt/target/linux/x86/patches-6.6/` during the build.

5. **Build the firmware**
   ```bash
   cd openwrt
   ./scripts/feeds update -a
   ./scripts/feeds install -a
   make defconfig           # use default x86/64 config
   make -j$(nproc) V=s      # or `make V=s` to see verbose output
   ```
   If the parallel build fails you can fall back to a single job:
   ```bash
   make -j1 V=s
   ```

6. **Collect the image**
   ```bash
   mkdir -p ../dist
   cp bin/targets/x86/64/*.img ../dist/
   ls -lh ../dist
   ```

## CI Pipeline

The GitHub Actions workflow defined in `.github/workflows/build-and-release.yml`
performs the same steps as above in a clean Ubuntu environment.  It is triggered
on pushes to `main` or manually (`workflow_dispatch`).

### Workflow Highlights
- Installs required build dependencies.
- Clones OpenWrt and the BMQ kernel tree.
- Applies patches from `bmq-patches/` if present.
- Runs `make` and packages the resulting `.img` files into a `dist/` folder.
- Uploads artifacts and automatically creates a GitHub Release with checksums.

> [!TIP]
> A new release tag is generated automatically from the Git reference or the
> current UTC timestamp when the workflow runs on `main`.

## Artifact Release

After a successful build, the `create-release` job collects the artifacts and
publishes a GitHub Release using [softprops/action-gh-release]. Each release
contains:

- `openwrt-x86-squashfs-*.img` files
- `SHA256SUMS.txt` with checksums

These files can be downloaded manually or used in automation pipelines for
flashing devices or testing under virtualization.

## Usage

Flash the generated image to a USB stick or load it in QEMU/VMware/VirtualBox.
For example:

```bash
# run under QEMU
qemu-system-x86_64 -drive file=openwrt-x86-64-combined-squashfs.img,format=raw
```

Once booted you will have a minimal OpenWrt system with the BMQ kernel.
Configuration and package installation works the same as upstream OpenWrt.

## Design Notes

* The kernel tarball is shipped into `openwrt/dl/` instead of relying on
  OpenWrt's download servers to ensure build reproducibility.
* The `bmq-patches/` directory is intentionally left empty in the repository so
  users can add their own patches without polluting the tree.
* Build timeouts in CI are set to 12 hours to accommodate slower runners.

## Contributing

Contributions are welcome!

1. Fork the repository and create a feature branch.
2. Add or update patches in `bmq-patches/` if needed.
3. Modify the workflow or documentation as necessary.
4. Open a pull request.

> [!NOTE]
> Please ensure that new patches apply cleanly against `linux-6.6.y-prjc` and
> that the workflow still completes within the timeout.

## License

This project is provided under the **MIT License**.  See the [LICENSE](LICENSE)
file for details.
