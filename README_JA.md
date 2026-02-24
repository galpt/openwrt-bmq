> 🇺🇸 [English](README.md) · 🇮🇩 [Indonesian](README_ID.md) · 🇨🇳 [Chinese](README_ZH.md) · 🇯🇵 [Japanese](README_JA.md)

# OpenWrt に BMQ CPU スケジューラを追加する

オープンソースルーターファームウェアの最高峰である OpenWrt の CPU スケジューラを、まったく別のものに交換したらどうなるのか？それがまさに、このリポジトリで明らかにしようとしていることです。

これは実験です。OpenWrt x86/64 ビルドに搭載されているバニラ Linux の EEVDF（Earliest Eligible Virtual Deadline First）スケジューラを、Alfred Chen によって作成され、[`linux-prjc`](https://gitlab.com/alfredchen/linux-prjc) カーネルツリーで保守されている代替 CPU スケジューラ **BMQ（BitMap Queue）** に置き換えます。すべての工程は GitHub Actions パイプラインで自動化されており、意図的に実行したビルドは毎回、手動の介入なしにフラッシュ可能なファームウェアイメージを生成します。

---

## 目次
- [ステータス](#ステータス)
- [なぜ CPU スケジューラを変えるのか？](#なぜ-cpu-スケジューラを変えるのか)
- [機能](#機能)
- [必要な環境](#必要な環境)
- [ローカルビルド](#ローカルビルド)
- [CI パイプライン](#ci-パイプライン)
- [成果物のリリース](#成果物のリリース)
- [使い方](#使い方)
- [設計上のメモ](#設計上のメモ)
- [コントリビューション](#コントリビューション)
- [ライセンス](#ライセンス)

## ステータス

**現在のブランチ**：`main`

> [!NOTE]
> これは進行中の実験です。CI パイプラインは `x86/64` プラットフォーム上の **OpenWrt v24.10.5**（最新の安定版）と **Linux 6.6.x** を対象としています。ビルドは手動で実行します。すべてのビルドを意図的な判断のもとで行うためです。

## なぜ CPU スケジューラを変えるのか？

正直なところから始めましょう。RAM 128 MB のシングルコア MIPS CPU を搭載した小型組み込みルーターで OpenWrt を動かしているのなら、おそらくこれは必要ありません。デフォルトの EEVDF スケジューラで十分です。

しかし、x86 ハードウェアは話が別です。ミニ PC や古い PC で OpenWrt を動かし始めると、組み込みルーターが本来想定していなかったことをやり始めます——Docker コンテナ、fq_codel や CAKE を使った SQM によるトラフィックシェーピング、さらにはいくつかのセルフホストサービスなど。気づけばあなたのルーターは、*片手間にパケットを転送する* 汎用コンピュータへと変貌しており、CPU は非常に多様な混合ワークロードを処理しなければなりません。

これこそが、BMQ のような代替スケジューラが静かに大きな違いをもたらせる場面です。BMQ（BitMap Queue）は、CPU が 100% に達しているときでもレイテンシに敏感なタスクの応答性を保つように設計されています。高負荷時、CAKE のような SQM アルゴリズムはバックグラウンドのタスクに CPU リソースを奪われることなくパケットスケジューリングロジックを実行する必要があります。Docker コンテナは安定して CPU の割り当てを受け続ける必要があります。デフォルトの EEVDF では、これらすべてのタスクがプレッシャー下でジッターを生じさせる形で競合します。BMQ はその競合を別の方法で処理します——ビットマップベースの優先度キューにより、バックグラウンドで何かが CPU を激しく使用しているときでも、インタラクティブかつレイテンシ重視のワークロードをスムーズに保ちます。

では、なぜ特に BMQ なのか、もっと新しいものではなく？良い質問です——外には興味深いスケジューラがたくさんあります。BORE は注目を集めており、LAVD はインタラクティブなワークロードで有望で、LFBMQ は BMQ の直接の進化版です。私たちが BMQ を選んだ正直な理由は、OpenWrt ビルドにカスタムスケジューラを組み込む*初めての試み*であり、十分にテストできないものに冒険したくなかったからです。BMQ は linux-prjc ツリーで長い実績があり、6.6 LTS パッチは成熟して適切にメンテナンスされており、最も安全な出発点に感じられました。

また、現時点では回避できないハードな制約もあります：OpenWrt v24.10.5 はまだ Linux 6.6.x を使用しています。BORE、LAVD、LFBMQ はいずれもクリーンに適用するためにカーネル 6.10 以降が必要です。OpenWrt がカーネルバージョンを上げるまで、それらの選択肢は単純に手が届きません。6.6 上の BMQ が今持っているものです——そして上記のユースケースにとっては、実験する価値が十分にあります。

## 機能

- **OpenWrt v24.10.5** に固定——アップストリームのローリング変更による予期せぬ問題なし。
- `CONFIG_KERNEL_GIT_CLONE_URI` を使って `linux-prjc` の `linux-6.6.y-prjc-lts` ブランチからカーネルを直接取得——tarball ハックなしで機能する OpenWrt 公式のカスタムカーネル機構。
- `target/linux/x86/config-6.6` に追記されるカーネル設定フラグメントで BMQ を有効化（`CONFIG_SCHED_ALT=y` + `CONFIG_SCHED_BMQ=y`）。
- `bmq-patches/` に `*.patch` ファイルを追加するだけで、ワークフローを変更せずにアドホックなカーネルカスタマイズが可能。
- `squashfs` および `ext4` の combined イメージ（BIOS 版と EFI 版の両方）を `.img.gz` 成果物として生成。
- ビルド成功のたびに `SHA256SUMS.txt` 付きで GitHub Releases へ公開。
- ビルドログは失敗時も含め常にアーティファクトとしてアップロード——問題が起きたときに必ず何か調べられるものが残ります。

## 必要な環境

標準的な OpenWrt ビルド依存関係がインストールされた Debian/Ubuntu マシンが必要です：

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

他のディストリビューションでも同等のパッケージを提供しています。CI ランナーは `ubuntu-latest` を使用しています。

## ローカルビルド

以下の手順は CI パイプラインが行うこととまったく同じです。ローカルでビルドできれば、CI でもビルドできます。

1. **このリポジトリをクローン**
   ```bash
   git clone https://github.com/galpt/openwrt-bmq.git
   cd openwrt-bmq
   ```

2. **OpenWrt v24.10.5 をクローン**
   ```bash
   git clone --depth 1 --branch v24.10.5 \
     https://github.com/openwrt/openwrt.git openwrt
   ```

3. **フィードをインストール**
   ```bash
   cd openwrt
   ./scripts/feeds update -a
   ./scripts/feeds install -a
   ```

4. **ビルド設定を記述**

   肝心なのは `CONFIG_KERNEL_GIT_CLONE_URI` です。これを設定すると、OpenWrt は自前の tarball をダウンロードするのではなく、指定した git URL からカーネルをクローンし、ハッシュチェックも自動的にスキップされます。

   ```bash
   # openwrt/ ディレクトリ内で実行
   printf '%s\n' \
     'CONFIG_TARGET_x86=y' \
     'CONFIG_TARGET_x86_64=y' \
     'CONFIG_TARGET_MULTI_PROFILE=y' \
     'CONFIG_TARGET_DEVICE_x86_64_DEVICE_generic=y' \
     'CONFIG_KERNEL_GIT_CLONE_URI="https://gitlab.com/alfredchen/linux-prjc.git"' \
     'CONFIG_KERNEL_GIT_REF="linux-6.6.y-prjc-lts"' \
     > .config

   make defconfig

   # BMQ スケジューラオプションを x86 カーネル設定フラグメントに追記
   printf '\n# BMQ (BitMap Queue) CPU Scheduler\nCONFIG_SCHED_ALT=y\nCONFIG_SCHED_BMQ=y\n# CONFIG_SCHED_PDS is not set\n' \
     >> target/linux/x86/config-6.6
   ```

5. **任意のパッチを追加**（ない場合はスキップ）：
   `*.patch` ファイルをこのリポジトリのルートにある `bmq-patches/` に置いてください。ビルド時に自動的に `openwrt/target/linux/x86/patches-6.6/` にコピーされます。

6. **ビルド実行**
   ```bash
   # openwrt/ 内で実行
   make -j$(nproc) V=s
   # 並列ビルドで競合が起きた場合はフォールバック
   make -j1 V=s
   ```

7. **イメージを収集**
   ```bash
   mkdir -p ../dist
   cp bin/targets/x86/64/*squashfs-combined*.img.gz ../dist/
   cp bin/targets/x86/64/*ext4-combined*.img.gz     ../dist/
   ls -lh ../dist
   ```

> [!TIP]
> OpenWrt はすべての出力イメージを gzip で圧縮します。探しているファイルは `*.img.gz` であり、裸の `*.img` ではありません。

## CI パイプライン

ワークフローは `.github/workflows/build-and-release.yml` に定義されており、**手動でのみ実行可能**（`workflow_dispatch`）です。この選択は意図的なものです——すべてのビルドは意識的な決断であるべきで、コミットへの自動的な反応であってはなりません。

大まかな実行順序は次のとおりです：

1. クリーンな `ubuntu-latest` ランナーにビルド依存関係をインストール。
2. OpenWrt v24.10.5 をクローン。
3. すべてのフィードを更新・インストール。
4. `linux-prjc` を指す `CONFIG_KERNEL_GIT_CLONE_URI` を含む `.config` を作成、`make defconfig` を実行、BMQ の Kconfig オプションを追記。
5. `bmq-patches/` にパッチがあればコピー。
6. `make -j$(nproc) V=s` を実行、失敗時は `-j1` にフォールバック。
7. 出力ディレクトリから `*squashfs-combined*.img.gz` と `*ext4-combined*.img.gz` を収集。
8. イメージを `create-release` ジョブに渡し、タイムスタンプ付きの GitHub Release として公開。

ビルドログはビルドが失敗した場合でも常にアーティファクトとしてアップロードされるため、デバッグ時に必ず参照できるものが残ります。

> [!NOTE]
> タグをプッシュした場合はその Git ref から release タグが派生します。ブランチに対して手動で実行した場合は `openwrt-bmq-YYYYMMDD-HHmmss` の形式で自動生成されます。

## 成果物のリリース

ビルドが成功するたびに、以下の内容を含む GitHub Release が公開されます：

| ファイル | ブートファームウェア | ルートファイルシステム |
|---------|---------------------|----------------------|
| `*squashfs-combined.img.gz`     | Legacy BIOS | squashfs — 読み取り専用 |
| `*squashfs-combined-efi.img.gz` | UEFI        | squashfs — 読み取り専用 |
| `*ext4-combined.img.gz`         | Legacy BIOS | ext4 — 書き込み可能     |
| `*ext4-combined-efi.img.gz`     | UEFI        | ext4 — 書き込み可能     |
| `SHA256SUMS.txt`                | —           | 上記すべてのチェックサム |

squashfs バリアントがほとんどのユースケースで推奨される選択肢です——サイズが小さく、読み取り専用の特性が設計によって保証されており、典型的な OpenWrt フラッシュイメージと同じように動作します。

## 使い方

Releases ページからイメージを入手したら、以下の方法で使用できます：

```bash
# 解凍
gunzip openwrt-*-squashfs-combined.img.gz

# USB スティックまたはディスクに書き込む——/dev/sdX は慎重に確認してから変更
dd if=openwrt-*-squashfs-combined.img of=/dev/sdX bs=4M status=progress
sync

# またはハードウェアなしで QEMU でテスト
qemu-system-x86_64 \
  -drive file=openwrt-*-squashfs-combined.img,format=raw \
  -m 256m -nographic
```

起動後は標準の OpenWrt シェルに入ります。パッケージ管理、ネットワーク設定、その他すべてはアップストリームと同様に動作します——唯一の違いは、その下で動作する CPU スケジューラです。

```bash
# ブート後に BMQ が有効か確認
cat /sys/kernel/debug/sched/features
# またはスケジューラの初期化メッセージを dmesg で確認
dmesg | grep -i sched
```

## 設計上のメモ

- **tarball ではなく `CONFIG_KERNEL_GIT_CLONE_URI` を使う理由は？** OpenWrt はダウンロードするカーネル tarball の SHA256 を検証します。カスタム tarball を期待するファイル名にリネームしてもハッシュチェックは騙せません——ビルドシステムは黙って本物のカーネルをダウンロードしてしまい、BMQ は一切コンパイルされません。`CONFIG_KERNEL_GIT_CLONE_URI` は、まさにこのユースケースのために OpenWrt が公式にサポートするメカニズムであり、設計上ハッシュチェックを無効化します。

- **なぜ Linux 6.6 なのか？** `linux-prjc` の BMQ パッチは 6.6 LTS カーネルファミリー（`linux-6.6.y-prjc-lts`）向けに保守されています。OpenWrt v24.10.5 は x86 向けに `KERNEL_PATCHVER=6.6` を使用しており、自然にマッチします。より新しいカーネルバージョンには、現在積極的にメンテナンスされている BMQ 移植版がありません。

- **なぜ x86/64 なのか？** テストが最も容易なプラットフォームだからです——物理ハードウェアなしで数秒のうちに QEMU 仮想マシンを起動できます。スケジューラパッチに互換性があれば、同じアプローチを理論的には他のアーキテクチャにも適用できます。

- **`bmq-patches/` ディレクトリ**は意図的にリポジトリ内で空のままにしています。ワークフローを変更せずに `linux-prjc` の上に追加の変更を重ねたい場合は、自分の `*.patch` ファイルをそこに置くだけで対応できます。

- **apt ミラーの選択**は `apt-get update` の実行前に行われます。パイプラインは 3 秒の `curl` タイミングチェックで `archive.ubuntu.com`、`azure.archive.ubuntu.com`、`us.archive.ubuntu.com` を測定し、最も応答が速いものを指すように apt ソースを書き換えます。接続ごとに 10 秒の短いタイムアウトと 5 回のリトライも設定されており、問題のあるホストは数分間ジョブを止めるのではなく素早く切り捨てられます。

- **CI のビルドタイムアウト**は 5 時間に設定されています。カーネルのコンパイルを含むコールドビルドは、GitHub の共有ランナー上では相当な時間がかかることがあります。

## コントリビューション

これは実験であり、あらゆる形での貢献を歓迎します——パッチ、アイデア、バグ報告、何でもどうぞ。

1. このリポジトリをフォークして機能ブランチを作成する。
2. カーネルパッチを追加する場合は `bmq-patches/` に配置し、`linux-6.6.y-prjc-lts` に対してクリーンに適用されることを確認する。
3. ワークフローを変更する場合は、プルリクエストを開く前に `workflow_dispatch` でテストする。
4. プルリクエストを開き、何をどのように変更したか、その理由を説明する。

> [!NOTE]
> プッシュのたびにワークフローを自動実行するトリガーを追加しないでください。ビルドはコストがかかるものであり、すべてのビルドを意図的に行いたいと考えています。

## ライセンス

このプロジェクトは **MIT ライセンス**のもとで提供されています。詳細は [LICENSE](LICENSE) ファイルを参照してください。
