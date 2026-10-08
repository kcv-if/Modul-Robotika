# Panduan Instalasi ROS 2 Jazzy

Panduan ini untuk memasang ROS 2 Jazzy Jalisco di laptop kalian. Gazebo dipasang terpisah, lihat `../Gazebo/instalasi-gazebo.md` setelah bagian ini selesai.

Kerjakan sebelum pertemuan pertama. Kalau ada yang gagal, catat pesan errornya persis seperti yang muncul di layar.

## Kenapa Jazzy

ROS 2 rilis versi baru tiap tahun, tapi tidak semua paket ikut pindah secepat itu. Versi LTS terbaru belum menyediakan `navigation2`, `nav2-bringup`, dan `turtlebot3` di repositori apt-nya, padahal ketiganya kita pakai di paruh kedua semester.

Jazzy versi terbaru yang ekosistem perkuliahannya sudah lengkap, dan pasangan resminya adalah Gazebo Harmonic. Satu kelas memakai kombinasi yang sama: Jazzy dan Harmonic di atas Ubuntu 24.04. Kalau kalian pasang versi lain, contoh di kelas tidak akan jalan dan kami sulit membantu.

## Pilih jalur

Cari baris pertama yang cocok dengan mesin kalian.

| Sistem | Jalur | Bagian |
|---|---|---|
| Ubuntu 24.04, Linux Mint 22, Pop!_OS 24.04 | apt langsung | [Bagian 1](#bagian-1-ubuntu-2404) |
| Windows 10 build 19045 ke atas, atau Windows 11 | WSL2 dan Ubuntu | [Bagian 2](#bagian-2-windows-lewat-wsl2) |
| macOS, Apple Silicon maupun Intel | pixi dan RoboStack | [Bagian 3](#bagian-3-macos) |
| Linux selain Ubuntu | Distrobox | [Bagian 4](#bagian-4-linux-selain-ubuntu) |
| NixOS | nix-ros-overlay | [Bagian 4](#bagian-4-linux-selain-ubuntu) |

Urutannya sengaja: instalasi langsung ke sistem operasi dulu, container paling akhir.

Alasannya sederhana. Kalau ROS terpasang langsung di sistem kalian, file ada di tempat kalian menaruhnya, `code .` membuka workspace yang barusan di-build, dan `colcon build` menulis ke folder yang kelihatan di file manager. Dengan container, pertanyaan seperti "workspace saya ke mana", "kenapa VS Code tidak menemukan package saya", dan "kenapa hasil build saya hilang" punya jawaban yang tidak ada hubungannya dengan robotika. Kalian sedang belajar robotika. Container tetap kami dukung, tapi posisinya cadangan.

## Bagian 1: Ubuntu 24.04

Siapkan sekitar 2,5 GB ruang disk kosong. Unduhan paket ROS sendiri sekitar 73 MB, sisanya dependensi Ubuntu.

### 1.1 Pastikan locale UTF-8

```bash
locale  # cek apakah sudah UTF-8

sudo apt update && sudo apt install locales
sudo locale-gen en_US en_US.UTF-8
sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
export LANG=en_US.UTF-8

locale  # verifikasi
```

Perintah `locale` yang kedua harus menampilkan `LANG=en_US.UTF-8`. Locale UTF-8 lain juga boleh.

### 1.2 Aktifkan repositori ROS

Cara lama yang memakai `apt-key` sudah tidak berlaku dan akan gagal di Ubuntu 24.04. Sekarang ROS menyediakan paket `.deb` bernama `ros2-apt-source` yang memasang kunci dan file sumbernya sekaligus.

```bash
sudo apt install software-properties-common
sudo add-apt-repository universe

sudo apt update && sudo apt install curl -y
export ROS_APT_SOURCE_VERSION=$(curl -s https://api.github.com/repos/ros-infrastructure/ros-apt-source/releases/latest | grep -F "tag_name" | awk -F'"' '{print $4}')
curl -L -o /tmp/ros2-apt-source.deb "https://github.com/ros-infrastructure/ros-apt-source/releases/download/${ROS_APT_SOURCE_VERSION}/ros2-apt-source_${ROS_APT_SOURCE_VERSION}.$(. /etc/os-release && echo ${UBUNTU_CODENAME:-${VERSION_CODENAME}})_all.deb"
sudo dpkg -i /tmp/ros2-apt-source.deb
```

Cek hasilnya:

```bash
cat /etc/apt/sources.list.d/ros2.sources    # harus ada, isinya menyebut packages.ros.org
sudo apt update && apt-cache policy ros-jazzy-desktop | head -3
```

Baris `Candidate:` harus berisi nomor versi, bukan `(none)`.

Kalau kampus memblokir GitHub atau kalian di balik proxy, perintah `curl` ke `api.github.com` gagal tanpa pesan apa pun. Akibatnya `ROS_APT_SOURCE_VERSION` kosong dan unduhannya 404. Isi manual:

```bash
export ROS_APT_SOURCE_VERSION=1.2.0
```

lalu ulangi perintah `curl -L` dan `dpkg -i` di atas.

### Catatan untuk pengguna Mint, Pop!_OS, Zorin, elementary

Perintah di 1.2 mengandung `${UBUNTU_CODENAME:-${VERSION_CODENAME}}`. Jangan disederhanakan.

Mint 22 melaporkan dirinya sebagai `wilma`, baik lewat `lsb_release -cs` maupun lewat `VERSION_CODENAME`. Mint 22 sebenarnya dibangun di atas Ubuntu Noble, tapi `packages.ros.org` tidak punya suite bernama `wilma`, jadi URL-nya akan 404. Hanya variabel `UBUNTU_CODENAME` yang berisi `noble`. Salin perintahnya apa adanya.

Mint 22 dan Pop!_OS 24.04 sudah kami pastikan berbasis Noble. Zorin 18 dan elementary OS 8 belum. Kalau kalian pakai keduanya, jalankan `. /etc/os-release && echo $UBUNTU_CODENAME` dan pastikan hasilnya `noble` sebelum lanjut.

### 1.3 Pasang ROS 2

```bash
sudo apt update
sudo apt upgrade
sudo apt install ros-jazzy-desktop
```

Paket `ros-jazzy-desktop` sudah mencakup ROS, RViz2, demo, dan tutorial. Jangan pakai `ros-jazzy-ros-base`, itu versi tanpa GUI.

Kalau muncul konflik dependensi, kemungkinan besar apt sources kalian cuma mengaktifkan suite `noble` saja. Ini sering terjadi di image minimal dan di WSL. Cek dengan:

```bash
grep Suites /etc/apt/sources.list.d/ubuntu.sources
```

Kalau `noble-updates` atau `noble-backports` tidak ada, edit file itu sehingga barisnya menjadi:

```
Suites: noble noble-updates noble-backports
```

lalu jalankan:

```bash
sudo apt clean && sudo apt update && sudo apt full-upgrade -y
```

### 1.4 Aktifkan environment

```bash
source /opt/ros/jazzy/setup.bash
```

Perintah ini cuma berlaku untuk terminal yang sedang terbuka. Supaya otomatis di setiap terminal baru:

```bash
echo 'source /opt/ros/jazzy/setup.bash' >> ~/.bashrc
```

Ganti dengan `setup.zsh` kalau shell kalian zsh.

Setelah ini lompat ke [Bagian 5](#bagian-5-pengaturan-wajib-untuk-semua-orang).

## Bagian 2: Windows lewat WSL2

### 2.1 Ini instalasi langsung, bukan container

Banyak yang mengira WSL itu sejenis VM sehingga sama saja dengan container. Bedanya cukup besar dan akan terasa sepanjang semester.

| | WSL2 dan apt (bagian ini) | Docker Desktop |
|---|---|---|
| Cara ROS terpasang | `sudo apt install ros-jazzy-desktop`, paket resmi yang sama persis dengan Ubuntu | image jadi yang dibuat orang lain |
| Lokasi workspace | `~/ros2_ws`, filesystem Linux milik kalian, bisa dibuka dari Windows Explorer di `\\wsl$\Ubuntu-24.04\home\namakalian` | di dalam container, hilang saat container dihapus |
| Hasil `colcon build` | bertahan, seperti mesin Linux biasa | bertahan hanya kalau mount volume-nya benar |
| VS Code | Remote-WSL langsung membuka workspace | butuh Dev Containers dan mount yang tepat |
| Pasang paket tambahan | `sudo apt install ros-jazzy-<nama>` | edit Dockerfile lalu rebuild |

Di bawahnya WSL2 memang menjalankan VM ringan, tapi kalian tidak pernah mengurusnya, dan tidak ada file, build, atau environment kalian yang masuk ke dalam container.

### 2.2 Cek dulu apakah laptopnya bisa

Buka PowerShell biasa:

```powershell
winver
```

Syarat minimum adalah Windows 10 versi 2004 (build 19041) ke atas, atau Windows 11. Khusus untuk GUI, yang di WSL disebut WSLg, minimalnya build 19044 ke atas atau Windows 11.

Virtualisasi harus aktif. Buka Task Manager, masuk ke Performance, pilih CPU, lalu cari baris Virtualization. Nilainya harus `Enabled`. Kalau `Disabled`, kalian perlu menyalakannya lewat BIOS atau UEFI, biasanya di bagian pengaturan CPU.

Pasang juga driver GPU dari situs vendornya, jangan mengandalkan Windows Update. Driver inilah yang menentukan apakah rendering memakai GPU asli atau CPU. Rendering CPU membuat Gazebo berjalan sekitar 20 kali lebih lambat. Ambil driver sesuai kartu grafis kalian dari Intel, AMD, atau NVIDIA, dan pasang sebelum melanjutkan.

Kalau laptop dikelola IT kampus dan WSL diblokir, ada tiga hal yang bisa dicoba. Pertama, minta ke IT. WSL2 fitur pengembang standar dan permintaan untuk mengaktifkan Virtual Machine Platform biasanya disetujui. Kedua, kalau yang diblokir cuma distribusi WSL-nya, Docker Desktop mungkin masih bisa, walaupun Docker Desktop butuh virtualisasi yang sama. Ketiga, kalau hypervisor-nya sendiri yang diblokir, tidak ada opsi lokal yang bisa dipakai di laptop itu.

### 2.3 Pasang WSL2 dan Ubuntu

Buka PowerShell sebagai Administrator, lewat klik kanan lalu Run as administrator:

```powershell
wsl --install -d Ubuntu-24.04
```

Restart laptop. Setelah itu buka Ubuntu dari Start menu dan buat username serta password Linux kalian.

Kalau WSL sudah pernah terpasang sebelumnya, perbarui saja:

```powershell
wsl --update
wsl --shutdown
```

### 2.4 Pastikan WSL2 dan WSLg aktif

Di PowerShell:

```powershell
wsl --list --verbose
```

Kolom VERSION untuk Ubuntu-24.04 harus berisi 2. WSLg tidak tersedia di WSL 1, jadi GUI tidak akan muncul sama sekali. Kalau tertulis 1:

```powershell
wsl --set-version Ubuntu-24.04 2
wsl --set-default-version 2
```

Lalu:

```powershell
wsl --version
```

Harus ada baris `WSLg version:`. Kalau perintahnya tidak dikenali, kalian masih memakai WSL bawaan yang lama, jalankan `wsl --update`.

Sekarang masuk ke Ubuntu dan cek:

```bash
ls /mnt/wslg          # harus ada
ls -l /dev/dxg        # harus ada, ini perangkat GPU virtualnya
echo "$DISPLAY"       # biasanya :0
```

Berikutnya buktikan GPU-nya benar-benar terpakai:

```bash
sudo apt update && sudo apt install -y mesa-utils
glxinfo -B | grep -iE "OpenGL renderer|OpenGL version"
```

Baris renderer harus menyebut nama kartu grafis kalian. Kalau yang muncul `llvmpipe`, berarti rendering dikerjakan CPU. Kembali ke langkah 2.2, pasang driver vGPU dari vendor, jalankan `wsl --shutdown` di PowerShell, lalu buka Ubuntu lagi.

Perhatikan bahwa rendering CPU tidak memunculkan pesan error. Semuanya terlihat normal, cuma sangat lambat. Tiap semester ada yang melaporkannya sebagai laptop yang kurang kuat, padahal instalasinya yang belum benar.

### 2.5 Kerjakan Bagian 1 di dalam Ubuntu

Seluruh langkah 1.1 sampai 1.4 berlaku tanpa perubahan. Tidak ada perbedaan antara Ubuntu biasa dan Ubuntu di WSLg.

Satu hal yang perlu dihindari: jangan pasang `LIBGL_ALWAYS_SOFTWARE=1`. Banyak petunjuk lama di internet menyuruh begitu. Sejak Gazebo Garden itu tidak diperlukan lagi, dan efeknya justru memaksa rendering CPU.

### 2.6 Simpan file di filesystem Linux

Taruh workspace di `~/ros2_ws`, jangan di `/mnt/c/...`. Akses lintas filesystem lewat lapisan 9p jauh lebih lambat dan membuat `colcon build` merangkak.

Setelah ini lompat ke [Bagian 5](#bagian-5-pengaturan-wajib-untuk-semua-orang).

## Bagian 3: macOS

Jalur ini memasang ROS langsung ke sistem kalian. Tidak perlu container, VM, akses root, Xcode, atau kompilasi dari sumber. Workspace-nya folder biasa di home directory, dan VS Code membukanya seperti proyek lain. Berlaku untuk Apple Silicon maupun Intel.

Ada satu batasan yang perlu kalian ketahui sejak awal. Di macOS, Gazebo Harmonic crash kalau dunia simulasinya memakai sensor yang perlu di-render, yaitu kamera, depth camera, RGBD, dan lidar GPU. Ini bug yang sudah dikenal di hulu dan belum diperbaiki untuk Harmonic.

Yang tetap berjalan normal cukup banyak: node ROS, topic, service, action, `colcon build`, RViz2, tf2, URDF, Gazebo server, Gazebo GUI, controller, `ros2_control`, dan Nav2 dengan lidar tipe ray. Untuk praktikum yang memakai kamera kalian akan pindah ke container, dan instruksinya diberikan menjelang pertemuan tersebut. Jumlahnya cuma beberapa pertemuan di akhir semester.

### 3.1 Pasang pixi

```bash
curl -fsSL https://pixi.sh/install.sh | bash
```

Tutup terminal, lalu buka lagi.

### 3.2 Buat environment

```bash
pixi init ros_ws --channel https://prefix.dev/robostack-jazzy
cd ros_ws
pixi add ros-jazzy-desktop
pixi add ros-dev-tools
```

Perhatikan URL channel-nya, harus `robostack-jazzy`. Contoh di dokumentasi resmi RoboStack sekarang memakai channel versi yang lebih baru. Kalau kalian menyalinnya mentah-mentah, yang terpasang adalah versi ROS yang berbeda dengan kelas.

### 3.3 Aktifkan dan uji

```bash
pixi shell
ros2 --help
ros2 run demo_nodes_cpp talker    # Ctrl-C setelah beberapa pesan muncul
rviz2
```

ROS hanya tersedia di dalam `pixi shell`, dan kalian harus berada di folder `ros_ws` saat menjalankannya. Setiap kali buka terminal baru: `cd ros_ws && pixi shell`.

### 3.4 Jangan source ROS sistem di dalam pixi

Kalau di `~/.zshrc` atau `~/.bash_profile` ada baris semacam `source /opt/ros/.../setup.bash`, sisa instalasi lama atau kebiasaan dari dual-boot, hapus dulu.

`PYTHONPATH` dari skrip itu bentrok dengan environment conda. Gejalanya `rclpy` gagal di-import, dengan pesan yang menyebut path Python dari prefix yang salah.

### 3.5 Kalau sudah terbiasa dengan conda, mamba, atau micromamba

pixi tetap jalur yang kami sarankan, tapi RoboStack juga bisa dipasang lewat conda, mamba, atau micromamba karena paketnya sama-sama berasal dari channel conda. Contoh di bawah memakai `micromamba`. Untuk `mamba` atau `conda`, ganti nama perintahnya saja.

```bash
micromamba create -n ros_env -c conda-forge -c robostack-jazzy ros-jazzy-desktop
micromamba activate ros_env
micromamba install -c conda-forge -c robostack-jazzy compilers cmake pkg-config make ninja colcon-common-extensions rosdep
```

Ada tiga hal yang perlu dijaga:

- Jangan pasang ROS ke environment `base`, dan jangan campur dengan channel `defaults` milik Anaconda. Paket dari `defaults` tidak kompatibel dengan conda-forge dan membuat solver menghasilkan kombinasi yang rusak.
- Channel-nya harus `robostack-jazzy`, sama seperti di 3.2.
- Setiap buka terminal baru, jalankan `micromamba activate ros_env`. Uji di 3.3 tetap berlaku, tinggal ganti `pixi shell` dengan perintah ini.

pip, uv, poetry, dan `venv` tidak bisa dipakai untuk memasang ROS. Paket seperti `rclpy` tidak tersedia di PyPI. Kalau kalian memakai `venv` untuk proyek Python lain, jangan aktifkan bersamaan dengan environment ROS, karena Python dari `venv` akan menutupi Python milik environment ROS dan `rclpy` gagal di-import.

### 3.6 Menambah library Python untuk node kalian

Node kalian nanti butuh library tambahan seperti `numpy` atau `opencv`. Pasang ke environment ROS yang sama, jangan ke Python sistem atau Python dari Homebrew. Node yang dijalankan lewat `ros2 run` cuma melihat Python milik environment ROS.

Urutan pilihannya:

1. Ambil dari conda-forge kalau tersedia. Paket di sana dibuat supaya cocok dengan paket ROS dari RoboStack.

   ```bash
   pixi add numpy opencv                        # jalur pixi, dari folder ros_ws
   micromamba install -c conda-forge numpy opencv   # jalur 3.5, environment sudah aktif
   ```

2. Kalau paketnya cuma ada di PyPI, baru pakai pip atau uv, tetap ke dalam environment ROS.

   ```bash
   pixi add --pypi nama-paket                   # jalur pixi, tercatat di pixi.toml

   # jalur 3.5, environment sudah aktif
   which python                                 # harus menunjuk ke dalam ros_env
   python -m pip install nama-paket
   uv pip install nama-paket                    # uv otomatis memakai environment conda yang aktif
   ```

Selalu pakai `python -m pip`, jangan `pip` saja. Perintah `pip` bisa saja milik Python lain yang kebetulan ada lebih dulu di `PATH`, dan paketnya terpasang di tempat yang tidak dilihat ROS.

Jangan pasang ulang lewat pip paket yang sudah dibawa RoboStack, misalnya `numpy`. Kalau versi dari pip menimpa versi conda, paket ROS yang dikompilasi terhadap versi lama bisa crash saat di-import.

Setelah ini lompat ke [Bagian 5](#bagian-5-pengaturan-wajib-untuk-semua-orang).

## Bagian 4: Linux selain Ubuntu

Kami sudah memeriksa repositori resmi masing-masing distro. Hasilnya, cuma NixOS yang punya jalur instalasi langsung yang lengkap.

| Distro | Paket ROS 2 Jazzy | Jalur |
|---|---|---|
| NixOS | ada, lengkap | `nix-ros-overlay` |
| RHEL 9, Rocky 9, Alma 9 (x86-64) | ada, 1455 paket RPM, tapi tanpa Gazebo | ROS langsung di host, Gazebo di container |
| Fedora | tidak ada, cuma paket tooling seperti `colcon` dan `rosdep` | Distrobox |
| Debian 12 dan 13 | tidak ada sama sekali | Distrobox |
| Arch, Manjaro | ada di AUR, tapi kompilasi dari sumber dan makan waktu berjam-jam | Distrobox |
| openSUSE | tidak ada | Distrobox |
| Gentoo | overlay tidak dirawat sejak 2024 | Distrobox |

Ada satu jebakan yang perlu diketahui soal Debian. `packages.ros.org` memang punya folder `bookworm` dan `trixie`, jadi sekilas Debian terlihat didukung. Isinya cuma 63 dan 78 paket, semuanya perkakas build seperti `colcon`, `rosdep`, dan `bloom`. Tidak ada satu pun paket `ros-jazzy-*`. Sebagai pembanding, suite `noble` berisi lebih dari 8000 paket.

### 4.1 Distrobox

Distrobox menjalankan image kuliah sebagai container yang sengaja tidak diisolasi. Home directory, GPU, socket X11 atau Wayland, dan terminal kalian semuanya ikut masuk ke dalamnya. Untuk yang belum pernah memakai container, ini membuat pengalamannya jauh lebih dekat ke instalasi biasa.

GPU langsung bekerja tanpa flag tambahan. Masalah label SELinux di Fedora juga tidak muncul, karena Distrobox mematikan konfinemen SELinux untuk container tersebut.

Ada satu hal yang perlu diwaspadai. Flag `--home` tidak mengisolasi home kalian. Dokumentasinya menyatakan bahwa flag itu tidak mencegah home host ikut ter-mount, gunanya cuma supaya dotfile container tidak mengotori home asli. Artinya isi `~/.bashrc`, variabel `PYTHONPATH`, atau blok `conda init` yang nyangkut di sistem kalian tetap bocor ke dalam container. Error yang muncul karena hal ini biasanya cuma terjadi di satu laptop dan sulit ditelusuri.

Perintah pembuatan container dan nama image-nya diberikan di kelas.

### 4.2 NixOS

Pakai `nix-ros-overlay`. Kalian sudah paham cara kerja sistem kalian sendiri, jadi ikuti README overlay-nya. ROS dan Gazebo dua-duanya tersedia di sana.

Setelah ini lompat ke [Bagian 5](#bagian-5-pengaturan-wajib-untuk-semua-orang).

## Bagian 5: Pengaturan wajib untuk semua orang

Dua pengaturan berikut berlaku di semua sistem operasi dan keduanya wajib.

### 5.1 ROS_DOMAIN_ID

Node ROS 2 yang berada pada domain yang sama akan saling menemukan dan saling berkirim pesan secara otomatis. Nilai bawaannya 0 untuk semua orang. Kalau banyak laptop berada di satu jaringan yang sama, artinya node semua orang saling terhubung. Mendebug robot yang bergerak sendiri karena menerima perintah dari laptop orang lain bukan pengalaman yang menyenangkan.

Pakai nomor yang diberikan dosen, atau nomor kursi kalian:

```bash
echo 'export ROS_DOMAIN_ID=42' >> ~/.bashrc   # ganti 42 dengan nomor kalian
```

Rentang aman untuk Linux adalah 0 sampai 101. Untuk macOS dan Windows, 0 sampai 166.

### 5.2 Middleware

Satu kelas memakai Cyclone DDS.

Alasannya, dengan middleware bawaan (Fast DDS), setiap perintah ROS di macOS memunculkan banyak pesan `[SYSTEM Error]` soal thread affinity. Fungsinya tetap berjalan, tapi tampilannya merah semua dan membingungkan. Cyclone tidak punya masalah tersebut. Karena satu kelas harus memakai middleware yang sama supaya bisa saling berkomunikasi, semuanya ikut Cyclone.

Untuk Ubuntu, WSL2, dan Linux:

```bash
sudo apt install ros-jazzy-rmw-cyclonedds-cpp
echo 'export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp' >> ~/.bashrc
```

Untuk macOS, jalankan dari dalam folder `ros_ws`:

```bash
pixi add ros-jazzy-rmw-cyclonedds-cpp
```

Lalu buat file `activate.sh` di sebelah `pixi.toml` dengan isi:

```bash
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
```

dan daftarkan di `pixi.toml`:

```toml
[activation]
scripts = ["activate.sh"]
```

Kalau memakai jalur 3.5, jalankan dari dalam environment yang sudah aktif:

```bash
micromamba install -c conda-forge -c robostack-jazzy ros-jazzy-rmw-cyclonedds-cpp
mkdir -p "$CONDA_PREFIX/etc/conda/activate.d"
echo 'export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp' > "$CONDA_PREFIX/etc/conda/activate.d/rmw.sh"
```

Skrip di folder `activate.d` dijalankan otomatis setiap kali environment diaktifkan, baik lewat conda, mamba, maupun micromamba.

## Bagian 6: Verifikasi

Buka terminal baru, lalu jalankan berurutan.

### 6.1 Pemeriksaan bawaan

```bash
ros2 doctor
```

Harusnya muncul `All <n> checks passed`. Peringatan soal "new distribution available" normal dan bisa diabaikan.

### 6.2 Talker dan listener

Uji ini memeriksa API C++ dan Python sekaligus.

Terminal 1:

```bash
ros2 run demo_nodes_cpp talker
```

Terminal 2:

```bash
ros2 run demo_nodes_py listener
```

Terminal 1 harus mencetak `Publishing: 'Hello World: 1'` dan seterusnya. Terminal 2 harus mencetak `I heard: [Hello World: 1]`.

Di macOS, jalankan `cd ros_ws && pixi shell` di kedua terminal, atau `micromamba activate ros_env` kalau memakai jalur 3.5.

Kalau talker jalan tapi listener diam saja, penyebabnya hampir selalu `ROS_DOMAIN_ID` yang berbeda di dua terminal itu, atau firewall yang memblokir traffic lokal.

### 6.3 Turtlesim

Uji ini memeriksa GUI dan input keyboard, mengikuti tutorial resmi "Using turtlesim". Cek dulu:

```bash
ros2 pkg executables turtlesim
```

Harus muncul empat baris: `draw_square`, `mimic`, `turtle_teleop_key`, dan `turtlesim_node`.

Paket ini sudah ikut di `ros-jazzy-desktop`. Kalau muncul `Package 'turtlesim' not found` di macOS, pasang manual dengan `pixi add ros-jazzy-turtlesim` dari folder `ros_ws`, atau `micromamba install ros-jazzy-turtlesim` di dalam environment kalian kalau memakai jalur 3.5.

Di macOS, aktifkan environment di kedua terminal di bawah: `cd ros_ws && pixi shell`, atau `micromamba activate ros_env`.

Terminal 1:

```bash
ros2 run turtlesim turtlesim_node
```

Jendela biru berisi seekor kura-kura harus muncul. Terminal 1 mencetak `Spawning turtle [turtle1] at x=[5.544445], y=[5.544445], theta=[0.000000]`.

Terminal 2:

```bash
ros2 run turtlesim turtle_teleop_key
```

Klik terminal 2 supaya fokus keyboard ada di sana, lalu tekan tombol panah. Kura-kuranya harus bergerak dan meninggalkan jejak garis. Tekan `q` untuk keluar dari teleop, dan Ctrl-C di terminal 1.

Kalau perintah di terminal 1 jalan tapi jendelanya tidak muncul, masalahnya di tampilan, bukan di ROS. Di WSL kembali ke 2.4. Kalau kura-kura diam saat tombol ditekan, pastikan yang sedang fokus terminal 2, bukan jendela kura-kura.

### 6.4 Cek nilai environment

```bash
echo $ROS_DOMAIN_ID          # nomor yang diberikan ke kalian
echo $RMW_IMPLEMENTATION     # rmw_cyclonedds_cpp
```

## Instalasi dianggap selesai kalau

```bash
ros2 doctor                     # All N checks passed
ros2 run demo_nodes_cpp talker  # Publishing: 'Hello World: 1'
ros2 run turtlesim turtlesim_node  # jendela kura-kura muncul, bisa digerakkan dengan teleop
echo $ROS_DOMAIN_ID             # nomor kalian, tidak kosong
echo $RMW_IMPLEMENTATION        # rmw_cyclonedds_cpp
```

## Masalah yang sering muncul

| Yang terlihat | Penyebab | Solusi |
|---|---|---|
| `ros2: command not found` | Belum di-source | `source /opt/ros/jazzy/setup.bash`. Di macOS: `cd ros_ws && pixi shell`, atau `micromamba activate ros_env` |
| `apt-cache policy` menampilkan `Candidate: (none)` | Repositori belum aktif | Ulangi langkah 1.2, cek isi `/etc/apt/sources.list.d/ros2.sources` |
| Unduhan `.deb` 404 | `ROS_APT_SOURCE_VERSION` kosong karena GitHub diblokir, atau codename salah di distro turunan Ubuntu | Isi versinya manual, dan pastikan memakai `UBUNTU_CODENAME` |
| `ros-dev-tools` gagal, dependensi tidak terpenuhi | `noble-updates` atau `noble-backports` tidak ada di apt sources | Lihat langkah 1.3 |
| Talker jalan, listener tidak menerima apa-apa | `ROS_DOMAIN_ID` berbeda antar terminal, atau firewall | Samakan nilainya, lalu cek firewall |
| macOS: `rclpy` gagal di-import, path Python aneh | Ada `source .../setup.bash` di file startup shell | Hapus baris itu, buka terminal baru |
| macOS: banyak `[SYSTEM Error]` soal thread affinity | Masih memakai Fast DDS | Pindah ke Cyclone, lihat 5.2 |
| WSL: GUI tidak muncul sama sekali | Distro masih WSL 1 | `wsl --set-version Ubuntu-24.04 2` |
| WSL: `glxinfo` menyebut `llvmpipe` | Driver vGPU vendor belum terpasang | Pasang driver Intel, AMD, atau NVIDIA, lalu `wsl --shutdown` |

Kalau masalah kalian tidak ada di tabel ini, catat pesan error persis seperti yang muncul beserta perintah yang dijalankan, lalu sertakan saat bertanya.
