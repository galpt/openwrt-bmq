> 🇺🇸 [English](README.md) · 🇮🇩 [Indonesian](README_ID.md) · 🇨🇳 [Chinese](README_ZH.md) · 🇯🇵 [Japanese](README_JA.md)

# Menambahkan CPU Scheduler BMQ ke OpenWrt

Apa yang terjadi kalau kita ambil OpenWrt — standar emas firmware router open-source — lalu kita ganti CPU scheduler-nya dengan sesuatu yang sama sekali berbeda? Nah, itu persis yang sedang coba kita jawab di repositori ini.

Ini adalah sebuah eksperimen. Kita mengganti vanilla Linux EEVDF (Earliest Eligible Virtual Deadline First) di dalam build OpenWrt x86/64 dengan **BMQ (BitMap Queue)**, sebuah CPU scheduler alternatif yang dibuat oleh Alfred Chen dan dipelihara di kernel tree [`linux-prjc`](https://gitlab.com/alfredchen/linux-prjc). Semuanya diotomasi dengan pipeline GitHub Actions sehingga setiap build yang kita jalankan menghasilkan firmware image yang siap di-flash, tanpa perlu campur tangan manual.

---

## Daftar Isi
- [Status](#status)
- [Mengapa Mengganti CPU Scheduler?](#mengapa-mengganti-cpu-scheduler)
- [Fitur](#fitur)
- [Persyaratan](#persyaratan)
- [Build Lokal](#build-lokal)
- [CI Pipeline](#ci-pipeline)
- [Artifact Release](#artifact-release)
- [Penggunaan](#penggunaan)
- [Catatan Desain](#catatan-desain)
- [Kontribusi](#kontribusi)
- [Lisensi](#lisensi)

## Status

**Branch Aktif**: `main`

> [!NOTE]
> Ini adalah eksperimen yang sedang berjalan. CI pipeline menargetkan **OpenWrt v24.10.5** (rilis stabil terbaru) dengan **Linux 6.6.x** pada platform `x86/64`. Build dijalankan secara manual agar setiap build merupakan keputusan yang disengaja.

## Mengapa Mengganti CPU Scheduler?

Jawaban jujurnya dulu: kalau kamu menjalankan OpenWrt di router embedded kecil dengan RAM 128 MB dan CPU MIPS single-core, kamu mungkin tidak butuh ini. EEVDF bawaan sudah lebih dari cukup.

Tapi hardware x86 itu cerita lain. Ketika kamu punya mini PC atau desktop bekas yang menjalankan OpenWrt, kamu mulai melakukan hal-hal yang tidak pernah dirancang untuk dilakukan embedded router — container Docker, SQM dengan fq_codel atau CAKE untuk traffic shaping, mungkin beberapa layanan self-hosted di sampingnya. Tiba-tiba routermu adalah komputer serba guna yang *kebetulan* juga melakukan routing paket, dan CPU-nya harus menangani beban kerja yang sangat beragam.

Di sinilah scheduler alternatif seperti BMQ bisa membuat perbedaan nyata secara diam-diam. BMQ (BitMap Queue) dirancang untuk menjaga task yang sensitif terhadap latensi tetap responsif bahkan saat CPU sedang di angka 100%. Di bawah beban berat, algoritma SQM seperti CAKE perlu menjalankan logika packet scheduling-nya tanpa kehabisan jatah CPU karena task latar belakang. Container Docker butuh jatah CPU yang dikasih secara konsisten. Dengan EEVDF bawaan, semua task ini bersaing dengan cara yang bisa menyebabkan jitter di bawah tekanan. BMQ menangani persaingan itu dengan cara berbeda — antrian prioritas berbasis bitmap miliknya cenderung membuat workload yang interaktif dan sensitif terhadap latensi tetap mulus meski ada sesuatu yang sedang membebani CPU di latar belakang.

Lalu kenapa BMQ secara spesifik dan bukan yang lebih baru? Pertanyaan bagus — ada banyak scheduler menarik di luar sana. BORE sedang banyak dibicarakan, LAVD terlihat menjanjikan untuk workload interaktif, dan LFBMQ adalah evolusi langsung dari BMQ itu sendiri. Alasan jujurnya kita memilih BMQ adalah karena ini adalah *percobaan pertama* kita dalam menyuntikkan scheduler kustom ke dalam build OpenWrt, dan kita tidak mau gegabah dengan sesuatu yang belum bisa kita uji secara menyeluruh. BMQ punya rekam jejak panjang di tree linux-prjc, patch LTS 6.6-nya sudah matang dan terawat dengan baik, dan terasa seperti titik awal yang paling bertanggung jawab.

Ada juga kendala teknis yang saat ini tidak bisa kita akali: OpenWrt v24.10.5 masih menggunakan Linux 6.6.x. BORE, LAVD, dan LFBMQ semuanya membutuhkan kernel 6.10 atau lebih baru untuk bisa diterapkan dengan bersih. Sebelum OpenWrt menaikkan versi kernel-nya, opsi-opsi itu memang tidak bisa digunakan. BMQ di 6.6 adalah yang kita punya — dan untuk use case yang disebutkan di atas, ini adalah sesuatu yang sangat layak untuk dieksperimentasi.

## Fitur

- Menyematkan ke **OpenWrt v24.10.5** — tidak ada kejutan dari perubahan upstream yang rolling.
- Menggunakan `CONFIG_KERNEL_GIT_CLONE_URI` untuk menarik kernel langsung dari branch `linux-6.6.y-prjc-lts` milik `linux-prjc` — mekanisme resmi OpenWrt untuk kernel kustom, tanpa hack tarball.
- Mengaktifkan BMQ via kernel config fragment yang ditambahkan ke `target/linux/x86/config-6.6` (`CONFIG_SCHED_ALT=y` + `CONFIG_SCHED_BMQ=y`).
- Mendukung penambahan file `*.patch` ke dalam `bmq-patches/` untuk kustomisasi kernel ad-hoc tanpa menyentuh workflow.
- Menghasilkan image `squashfs` dan `ext4` combined (varian BIOS dan EFI) sebagai artifact `.img.gz`.
- Mempublikasikan setiap build yang sukses ke GitHub Releases dengan `SHA256SUMS.txt`.
- Build log selalu diunggah sebagai artifact — bahkan saat gagal — sehingga selalu ada sesuatu untuk dilihat ketika ada masalah.

## Persyaratan

Kamu butuh mesin Debian/Ubuntu dengan dependensi build OpenWrt standar yang terinstal:

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

Distribusi lain seharusnya menyediakan paket yang setara. CI runner menggunakan `ubuntu-latest`.

## Build Lokal

Langkah-langkah di bawah ini mencerminkan persis apa yang dilakukan CI pipeline. Kalau berhasil di build lokal, pasti berhasil juga di CI.

1. **Clone repositori ini**
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

4. **Tulis build config**

   Kuncinya ada di `CONFIG_KERNEL_GIT_CLONE_URI`. Ketika ini diset, OpenWrt akan meng-clone kernel dari URL git yang diberikan alih-alih mengunduh tarball miliknya sendiri, dan pengecekan hash dilewati secara otomatis.

   ```bash
   # di dalam direktori openwrt/
   printf '%s\n' \
     'CONFIG_TARGET_x86=y' \
     'CONFIG_TARGET_x86_64=y' \
     'CONFIG_TARGET_MULTI_PROFILE=y' \
     'CONFIG_TARGET_DEVICE_x86_64_DEVICE_generic=y' \
     'CONFIG_KERNEL_GIT_CLONE_URI="https://gitlab.com/alfredchen/linux-prjc.git"' \
     'CONFIG_KERNEL_GIT_REF="linux-6.6.y-prjc-lts"' \
     > .config

   make defconfig

   # Tambahkan opsi scheduler BMQ ke kernel config fragment x86
   printf '\n# BMQ (BitMap Queue) CPU Scheduler\nCONFIG_SCHED_ALT=y\nCONFIG_SCHED_BMQ=y\n# CONFIG_SCHED_PDS is not set\n' \
     >> target/linux/x86/config-6.6
   ```

5. **Tambahkan patch opsional** (lewati jika tidak ada):
   Taruh file `*.patch` apa pun di bawah `bmq-patches/` di root repositori ini. Mereka akan disalin ke `openwrt/target/linux/x86/patches-6.6/` secara otomatis.

6. **Build**
   ```bash
   # di dalam openwrt/
   make -j$(nproc) V=s
   # fallback kalau build paralel race condition
   make -j1 V=s
   ```

7. **Kumpulkan image-nya**
   ```bash
   mkdir -p ../dist
   cp bin/targets/x86/64/*squashfs-combined*.img.gz ../dist/
   cp bin/targets/x86/64/*ext4-combined*.img.gz     ../dist/
   ls -lh ../dist
   ```

> [!TIP]
> OpenWrt mengompresi semua output image dengan gzip. File yang kamu cari adalah `*.img.gz`, bukan `*.img` biasa.

## CI Pipeline

Workflow-nya ada di `.github/workflows/build-and-release.yml` dan **hanya bisa dipicu secara manual** (`workflow_dispatch`). Pilihan ini kita buat dengan sengaja — setiap build adalah keputusan yang disadari, bukan reaksi otomatis terhadap sebuah commit.

Ini urutan high-level-nya:

1. Install dependensi build di runner `ubuntu-latest` yang bersih.
2. Clone OpenWrt v24.10.5.
3. Update dan install semua feeds.
4. Tulis `.config` dengan `CONFIG_KERNEL_GIT_CLONE_URI` yang mengarah ke `linux-prjc`, jalankan `make defconfig`, dan tambahkan opsi Kconfig BMQ.
5. Salin patch dari `bmq-patches/` jika ada.
6. Jalankan `make -j$(nproc) V=s`, fallback ke `-j1` jika gagal.
7. Kumpulkan `*squashfs-combined*.img.gz` dan `*ext4-combined*.img.gz` dari direktori output.
8. Serahkan image ke job `create-release` yang mempublikasikannya sebagai GitHub Release dengan timestamp.

Build log selalu diunggah sebagai artifact — bahkan ketika build gagal — sehingga selalu ada sesuatu untuk dibaca saat debugging.

> [!NOTE]
> Tag release diturunkan dari Git ref kalau kamu push tag, atau di-generate otomatis sebagai `openwrt-bmq-YYYYMMDD-HHmmss` ketika dipicu manual terhadap sebuah branch.

## Artifact Release

Setiap run yang sukses mempublikasikan GitHub Release yang berisi:

| File | Boot | Root filesystem |
|------|------|-----------------|
| `*squashfs-combined.img.gz`     | Legacy BIOS | squashfs — read-only |
| `*squashfs-combined-efi.img.gz` | UEFI        | squashfs — read-only |
| `*ext4-combined.img.gz`         | Legacy BIOS | ext4 — writable      |
| `*ext4-combined-efi.img.gz`     | UEFI        | ext4 — writable      |
| `SHA256SUMS.txt`                | —           | checksum untuk semua file di atas |

Varian squashfs adalah pilihan yang direkomendasikan untuk sebagian besar use case — ukurannya lebih kecil, read-only by design, dan berperilaku persis seperti image flash OpenWrt pada umumnya.

## Penggunaan

### Opsi 1: Fresh Install (dd / QEMU)

Mulai dari nol di bare metal? Unduh image dari halaman Releases lalu tulis langsung ke disk atau USB stick.

```bash
# Dekompresi image terlebih dahulu
gunzip openwrt-*-squashfs-combined.img.gz

# Tulis ke disk atau USB stick — pastikan /dev/sdX sudah benar sebelum dijalankan
dd if=openwrt-*-squashfs-combined.img of=/dev/sdX bs=4M status=progress
sync
```

Mau test dulu sebelum menyentuh hardware? Pakai QEMU:

```bash
qemu-system-x86_64 \
  -drive file=openwrt-*-squashfs-combined.img,format=raw \
  -m 256m -nographic
```

### Opsi 2: Flash via LuCI Web UI (Sysupgrade)

Sudah punya OpenWrt yang berjalan dan mau upgrade ke build BMQ ini? Kamu tidak perlu akses fisik atau `dd` — LuCI web UI menanganinya dengan bersih.

Buka **LuCI → System → Backup / Flash Firmware**, scroll ke bawah ke bagian **Flash new firmware image**, lalu upload file kamu.

> [!IMPORTANT]
> **Upload file `.img.gz` langsung — jangan didekompresi dulu.** LuCI menangani gzip secara transparan.

**Memilih file yang tepat itu penting banget. Gunakan tabel ini:**

| Mode boot kamu | File yang diupload |
|---|---|
| Legacy BIOS / MBR | `openwrt-x86-64-generic-squashfs-combined.img.gz` |
| UEFI / EFI | `openwrt-x86-64-generic-squashfs-combined-efi.img.gz` |

Tidak yakin mode boot mesinmu? Jalankan ini di shell OpenWrt:

```bash
ls /sys/firmware/efi 2>/dev/null && echo "EFI" || echo "Legacy BIOS"
```

> [!WARNING]
> **Jangan pernah gunakan image `ext4` untuk sysupgrade.** Varian ext4 sama sekali tidak mendukung jalur sysupgrade — mereka hanya untuk fresh install via `dd`. Mekanisme sysupgrade OpenWrt bergantung pada struktur SquashFS + overlayfs untuk menyimpan konfigurasi kamu saat upgrade. Format ext4 tidak memiliki overlay filesystem, jadi mem-flash-nya lewat web UI akan gagal total atau membuat perangkat kamu tidak bisa boot.

Setelah kamu upload file yang tepat, pilih apakah mau **Keep settings** (biarkan dicentang untuk mempertahankan konfigurasi yang ada), klik **Flash image**, verifikasi checksum di layar konfirmasi, lalu klik **Proceed**. Perangkat akan reboot ke build BMQ yang baru secara otomatis.

### Verifikasi BMQ Aktif

Baik setelah fresh flash maupun sysupgrade, konfirmasi scheduler berjalan setelah boot:

```bash
# Cek fitur scheduler yang aktif
cat /sys/kernel/debug/sched/features

# Atau cari pesan inisialisasi scheduler
dmesg | grep -i sched
```

## Catatan Desain

- **Kenapa `CONFIG_KERNEL_GIT_CLONE_URI` dan bukan tarball?** OpenWrt memverifikasi SHA256 dari setiap tarball kernel yang diunduhnya. Mengganti nama tarball kustom agar cocok dengan nama file yang diharapkan tidak mengecoh pengecekan hash — build akan diam-diam mengunduh kernel asli dan BMQ tidak pernah ikut dikompilasi. `CONFIG_KERNEL_GIT_CLONE_URI` adalah mekanisme yang secara resmi didukung OpenWrt untuk use case ini, dan ia menonaktifkan pengecekan hash by design.

- **Kenapa Linux 6.6?** Patch BMQ `linux-prjc` dipelihara untuk keluarga kernel LTS 6.6 (`linux-6.6.y-prjc-lts`). OpenWrt v24.10.5 menargetkan `KERNEL_PATCHVER=6.6` untuk x86, yang membuatnya sangat cocok. Versi kernel yang lebih baru belum memiliki port BMQ yang aktif dipelihara.

- **Kenapa x86/64?** Ini adalah platform yang paling mudah untuk ditest — kamu bisa menjalankan VM QEMU dalam hitungan detik tanpa membutuhkan hardware fisik. Pendekatan yang sama bisa, secara teori, diadaptasi untuk arsitektur lain jika patch scheduler-nya kompatibel.

- **Direktori `bmq-patches/`** sengaja dibiarkan kosong di repositori. Taruh file `*.patch` milikmu di sana kalau mau menambahkan perubahan di atas `linux-prjc` tanpa mengubah workflow.

- **Pemilihan mirror apt** dilakukan sebelum `apt-get update` dijalankan. Pipeline melakukan probe ke `archive.ubuntu.com`, `azure.archive.ubuntu.com`, dan `us.archive.ubuntu.com` dengan pengecekan timing `curl` 3 detik lalu menulis ulang sumber apt agar mengarah ke yang merespons paling cepat. Timeout per-koneksi singkat (10 detik) dan 5 retry juga dikonfigurasi sehingga host yang bermasalah diabaikan dengan cepat alih-alih membuat job hang selama bermenit-menit.

- **Timeout build** di CI diset ke 5 jam. Build pertama dengan kompilasi kernel bisa memakan waktu lama di runner GitHub yang shared.

## Kontribusi

Ini adalah eksperimen dan kontribusi sangat disambut — patch, ide, laporan bug, semuanya.

1. Fork repositori ini dan buat feature branch.
2. Kalau mau menambahkan kernel patch, taruh di `bmq-patches/` dan verifikasi bahwa patch tersebut bisa diterapkan dengan bersih terhadap `linux-6.6.y-prjc-lts`.
3. Kalau mengubah workflow, test dulu dengan `workflow_dispatch` sebelum membuka pull request.
4. Buka pull request dan jelaskan apa yang kamu ubah dan mengapa.

> [!NOTE]
> Mohon jangan menambahkan trigger yang menjalankan workflow secara otomatis di setiap push. Build itu mahal dan kita ingin setiap build dilakukan dengan sengaja.

## Lisensi

Proyek ini disediakan di bawah **MIT License**. Lihat file [LICENSE](LICENSE) untuk detailnya.
