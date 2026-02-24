> 🇺🇸 [English](README.md) · 🇮🇩 [Indonesian](README_ID.md) · 🇨🇳 [Chinese](README_ZH.md) · 🇯🇵 [Japanese](README_JA.md)

# 为 OpenWrt 添加 BMQ CPU 调度器

如果我们把 OpenWrt——开源路由器固件领域的黄金标准——的 CPU 调度器换成完全不同的东西，会发生什么？这正是本仓库想要探索的问题。

这是一个实验。我们将 OpenWrt x86/64 构建中的原版 Linux EEVDF（最早截止期限优先调度器）替换为 **BMQ（位图队列）**——一个由 Alfred Chen 编写并在 [`linux-prjc`](https://gitlab.com/alfredchen/linux-prjc) 内核树中维护的替代 CPU 调度器。整个流程通过 GitHub Actions 管道自动化，让每一次有意为之的构建都能产出可直接烧录的固件镜像，无需任何手动干预。

---

## 目录
- [状态](#状态)
- [为什么要更换 CPU 调度器？](#为什么要更换-cpu-调度器)
- [特性](#特性)
- [环境要求](#环境要求)
- [本地构建](#本地构建)
- [CI 流水线](#ci-流水线)
- [产物发布](#产物发布)
- [使用方法](#使用方法)
- [设计说明](#设计说明)
- [参与贡献](#参与贡献)
- [许可证](#许可证)

## 状态

**当前分支**：`main`

> [!NOTE]
> 这是一个持续进行中的实验。CI 流水线针对 `x86/64` 平台上的 **OpenWrt v24.10.5**（最新稳定版）与 **Linux 6.6.x** 进行构建。构建需要手动触发，以确保每次构建都是经过深思熟虑的决定。

## 为什么要更换 CPU 调度器？

先说实话：如果你的 OpenWrt 跑在一台 RAM 只有 128 MB、单核 MIPS CPU 的小型嵌入式路由器上，你大概率不需要这个。默认的 EEVDF 调度器完全够用。

但 x86 硬件是另一回事了。当你用一台小型 PC 或闲置台式机运行 OpenWrt 的时候，你开始做一些嵌入式路由器从未被设计用于完成的事——Docker 容器、使用 fq_codel 或 CAKE 进行流量整形的 SQM，以及可能运行的几个自托管服务。突然间，你的路由器变成了一台*同时*负责转发数据包的通用计算机，CPU 需要应对极为多样的混合工作负载。

这正是 BMQ 这类替代调度器能悄然带来实质性改变的地方。BMQ（位图队列）的设计目标就是在 CPU 占用率达到 100% 的情况下，依然保持延迟敏感型任务的响应性。在高负载下，像 CAKE 这样的 SQM 算法需要在不被后台任务抢光 CPU 时间片的情况下运行其包调度逻辑；Docker 容器需要稳定、持续地获得 CPU 分配。在默认的 EEVDF 下，所有这些任务会以一种在高压下容易引入抖动的方式竞争资源。BMQ 用不同的方式处理这种竞争——其基于位图的优先级队列在后台有任务猛烈占用 CPU 的情况下，依然能保持交互式和延迟敏感型工作负载的流畅性。

那为什么偏偏选 BMQ，而不是更新的调度器？问得好——外面有不少有趣的调度器。BORE 正备受关注，LAVD 在交互式工作负载方面颇具潜力，LFBMQ 则是 BMQ 本身的直接演进版本。我们选择 BMQ 的真实原因是：这是我们*第一次*尝试将自定义调度器注入 OpenWrt 构建，我们不想在一个尚未能够充分测试的东西上冒险。BMQ 在 linux-prjc 树上有着漫长的历史，6.6 LTS 补丁成熟且维护良好，它感觉是最稳妥的起点。

还有一个我们目前无法绕过的硬性约束：OpenWrt v24.10.5 仍然使用 Linux 6.6.x。BORE、LAVD 和 LFBMQ 都需要 6.10 或更新版本的内核才能干净地打入补丁。在 OpenWrt 升级其内核版本之前，这些选项根本无从实现。我们现有的就是 6.6 上的 BMQ——对于上述使用场景，它是一个完全值得折腾的实验对象。

## 特性

- 锁定至 **OpenWrt v24.10.5**——不受上游滚动更新带来的意外影响。
- 使用 `CONFIG_KERNEL_GIT_CLONE_URI` 直接从 `linux-prjc` 的 `linux-6.6.y-prjc-lts` 分支拉取内核——这是 OpenWrt 官方支持的自定义内核机制，无需 tarball 技巧。
- 通过追加到 `target/linux/x86/config-6.6` 的内核配置片段启用 BMQ（`CONFIG_SCHED_ALT=y` + `CONFIG_SCHED_BMQ=y`）。
- 支持将额外的 `*.patch` 文件放入 `bmq-patches/` 进行即兴内核定制，无需修改工作流。
- 以 `.img.gz` 产物的形式生成 `squashfs` 和 `ext4` combined 镜像（含 BIOS 和 EFI 两种变体）。
- 每次成功构建均发布至 GitHub Releases，并附有 `SHA256SUMS.txt`。
- 构建日志始终作为产物上传——即使构建失败也不例外——确保出问题时总有东西可以查看。

## 环境要求

你需要一台安装了标准 OpenWrt 构建依赖的 Debian/Ubuntu 机器：

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

其他发行版应提供同等的软件包。CI Runner 使用 `ubuntu-latest`。

## 本地构建

以下步骤与 CI 流水线的执行过程完全一致。本地能构建成功，CI 中就能构建成功。

1. **克隆本仓库**
   ```bash
   git clone https://github.com/galpt/openwrt-bmq.git
   cd openwrt-bmq
   ```

2. **克隆 OpenWrt v24.10.5**
   ```bash
   git clone --depth 1 --branch v24.10.5 \
     https://github.com/openwrt/openwrt.git openwrt
   ```

3. **安装 feeds**
   ```bash
   cd openwrt
   ./scripts/feeds update -a
   ./scripts/feeds install -a
   ```

4. **写入构建配置**

   关键在于 `CONFIG_KERNEL_GIT_CLONE_URI`。设置此项后，OpenWrt 会从给定的 git URL 克隆内核，而不是下载自己的 tarball，哈希校验也会自动跳过。

   ```bash
   # 在 openwrt/ 目录内执行
   printf '%s\n' \
     'CONFIG_TARGET_x86=y' \
     'CONFIG_TARGET_x86_64=y' \
     'CONFIG_TARGET_MULTI_PROFILE=y' \
     'CONFIG_TARGET_DEVICE_x86_64_DEVICE_generic=y' \
     'CONFIG_KERNEL_GIT_CLONE_URI="https://gitlab.com/alfredchen/linux-prjc.git"' \
     'CONFIG_KERNEL_GIT_REF="linux-6.6.y-prjc-lts"' \
     > .config

   make defconfig

   # 将 BMQ 调度器选项追加到 x86 内核配置片段
   printf '\n# BMQ (BitMap Queue) CPU Scheduler\nCONFIG_SCHED_ALT=y\nCONFIG_SCHED_BMQ=y\n# CONFIG_SCHED_PDS is not set\n' \
     >> target/linux/x86/config-6.6
   ```

5. **添加可选补丁**（如无则跳过）：
   将任意 `*.patch` 文件放置在本仓库根目录的 `bmq-patches/` 下，它们会被自动复制到 `openwrt/target/linux/x86/patches-6.6/`。

6. **开始构建**
   ```bash
   # 在 openwrt/ 内执行
   make -j$(nproc) V=s
   # 若并行构建出现竞争条件则回退
   make -j1 V=s
   ```

7. **收集镜像**
   ```bash
   mkdir -p ../dist
   cp bin/targets/x86/64/*squashfs-combined*.img.gz ../dist/
   cp bin/targets/x86/64/*ext4-combined*.img.gz     ../dist/
   ls -lh ../dist
   ```

> [!TIP]
> OpenWrt 会用 gzip 压缩所有输出镜像。你要找的文件是 `*.img.gz`，而不是裸露的 `*.img`。

## CI 流水线

工作流文件位于 `.github/workflows/build-and-release.yml`，**仅支持手动触发**（`workflow_dispatch`）。这是经过深思熟虑的选择——每次构建都应该是一个有意识的决定，而不是对某次提交的自动反应。

高层执行序列如下：

1. 在全新的 `ubuntu-latest` Runner 上安装构建依赖。
2. 克隆 OpenWrt v24.10.5。
3. 更新并安装所有 feeds。
4. 写入 `.config`（含指向 `linux-prjc` 的 `CONFIG_KERNEL_GIT_CLONE_URI`），运行 `make defconfig`，并追加 BMQ Kconfig 选项。
5. 如有补丁，从 `bmq-patches/` 复制。
6. 运行 `make -j$(nproc) V=s`，失败则回退至 `-j1`。
7. 从输出目录收集 `*squashfs-combined*.img.gz` 和 `*ext4-combined*.img.gz`。
8. 将镜像传递给 `create-release` job，发布为带时间戳的 GitHub Release。

即使构建失败，构建日志也始终作为产物上传，确保调试时总有迹可循。

> [!NOTE]
> 如果你推送了标签，release tag 从 Git ref 中派生；否则在手动触发时自动生成为 `openwrt-bmq-YYYYMMDD-HHmmss` 格式。

## 产物发布

每次成功运行均会发布包含以下内容的 GitHub Release：

| 文件 | 引导固件 | 根文件系统 |
|------|----------|------------|
| `*squashfs-combined.img.gz`     | Legacy BIOS | squashfs — 只读 |
| `*squashfs-combined-efi.img.gz` | UEFI        | squashfs — 只读 |
| `*ext4-combined.img.gz`         | Legacy BIOS | ext4 — 可写     |
| `*ext4-combined-efi.img.gz`     | UEFI        | ext4 — 可写     |
| `SHA256SUMS.txt`                | —           | 以上所有文件的校验和 |

squashfs 变体是大多数使用场景的推荐选择——体积更小，只读特性由设计保证，行为与典型的 OpenWrt 烧录镜像完全一致。

## 使用方法

从 Releases 页面下载镜像后，可以按以下方式使用：

```bash
# 解压缩
gunzip openwrt-*-squashfs-combined.img.gz

# 写入 USB 闪存盘或磁盘——请谨慎替换 /dev/sdX
dd if=openwrt-*-squashfs-combined.img of=/dev/sdX bs=4M status=progress
sync

# 或者跳过硬件，直接用 QEMU 测试
qemu-system-x86_64 \
  -drive file=openwrt-*-squashfs-combined.img,format=raw \
  -m 256m -nographic
```

启动后你将进入标准的 OpenWrt shell。包管理、网络配置及其他所有操作与上游完全一致——唯一的区别是底层运行的 CPU 调度器。

```bash
# 启动后验证 BMQ 是否激活
cat /sys/kernel/debug/sched/features
# 或检查 dmesg 中的调度器初始化信息
dmesg | grep -i sched
```

## 设计说明

- **为什么用 `CONFIG_KERNEL_GIT_CLONE_URI` 而不是 tarball？** OpenWrt 会验证其下载的每个内核 tarball 的 SHA256。把自定义 tarball 改名成预期的文件名并不能骗过哈希校验——构建系统会悄悄下载真正的内核，BMQ 永远不会被编译进去。`CONFIG_KERNEL_GIT_CLONE_URI` 正是 OpenWrt 为此用例官方支持的机制，它在设计上就禁用了哈希校验。

- **为什么是 Linux 6.6？** `linux-prjc` 的 BMQ 补丁是为 6.6 LTS 内核家族（`linux-6.6.y-prjc-lts`）维护的。OpenWrt v24.10.5 在 x86 上使用 `KERNEL_PATCHVER=6.6`，两者天然契合。更新的内核版本目前还没有积极维护的 BMQ 移植版本。

- **为什么是 x86/64？** 这是最容易测试的平台——无需任何物理硬件，几秒钟内就能启动 QEMU 虚拟机。同样的思路理论上也可以适配其他架构，前提是调度器补丁兼容。

- **`bmq-patches/` 目录**在仓库中故意保持为空。如果你想在 `linux-prjc` 之上叠加额外更改而不修改工作流，把自己的 `*.patch` 文件放进去就行。

- **apt 镜像源选择**在 `apt-get update` 运行之前完成。流水线用 3 秒 `curl` 计时探测 `archive.ubuntu.com`、`azure.archive.ubuntu.com` 和 `us.archive.ubuntu.com`，然后将 apt 源重写为响应最快的那个。同时还配置了 10 秒的短连接超时和 5 次重试，确保问题节点能被快速放弃，而不是让 job 卡住好几分钟。

- **CI 构建超时**设置为 5 小时。包含内核编译的冷启动构建在 GitHub 共享 Runner 上可能需要相当长的时间。

## 参与贡献

这是一个实验，非常欢迎各种形式的贡献——补丁、想法、bug 报告，统统来者不拒。

1. Fork 本仓库并创建特性分支。
2. 如果要添加内核补丁，将其放入 `bmq-patches/`，并验证它能干净地应用于 `linux-6.6.y-prjc-lts`。
3. 如果要修改工作流，请先用 `workflow_dispatch` 测试，再提交 Pull Request。
4. 提交 Pull Request，说明你修改了什么以及为什么这样修改。

> [!NOTE]
> 请不要添加在每次推送时自动运行工作流的触发器。构建是有代价的，我们希望每次构建都是经过深思熟虑的。

## 许可证

本项目遵循 **MIT 许可证**。详情请参阅 [LICENSE](LICENSE) 文件。
