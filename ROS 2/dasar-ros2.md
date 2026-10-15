# Dasar ROS 2

Materi ini mengasumsikan kalian sudah bisa memprogram, dan sudah menyelesaikan `instalasi-ros2.md`. Yang belum tentu kalian punya adalah pengalaman dengan sistem terdistribusi dan dengan perangkat keras yang berjalan terus-menerus. Dua hal itulah yang membentuk hampir semua keputusan desain di ROS.

Kerjakan sambil mengetik. Membaca materi ini tanpa menjalankan perintahnya tidak akan banyak membantu.

## Peta materi

```mermaid
graph TD
    A["1-3. Konsep<br/>apa itu ROS, kenapa banyak node"] --> B["4. Melihat graf<br/>CLI dan rqt_graph"]
    B --> C["5-6. Node dan topic"]
    C --> D["7-10. Package pertama<br/>publisher dan subscriber"]
    D --> E["11-12. Service dan action"]
    E --> F["13. QoS"]
    F --> G["14-16. Parameter, launch, bag"]

    style B fill:#2d6a4f,color:#fff
    style F fill:#9d0208,color:#fff
```

Bagian 4 dan 13 yang paling sering menyelamatkan kalian saat macet.

## 1. Apa itu ROS, dan apa yang bukan

Nama ROS adalah singkatan dari Robot Operating System, dan nama itu menyesatkan. ROS bukan sistem operasi. ROS berjalan di atas Linux.

| Yang kalian dapat | Wujudnya | Gunanya |
|---|---|---|
| Middleware publish-subscribe | DDS di lapisan bawah | Program kecil saling berkirim pesan tanpa tahu lawan bicaranya |
| Sistem build dan konvensi paket | `colcon`, `ament` | Kode dari ribuan orang bisa dipasang berdampingan tanpa saling merusak |
| Kesepakatan bersama | REP, tipe pesan standar | Nama frame, satuan, tata nama topic seragam lintas pembuat |

Kesepakatan di baris ketiga yang membuat driver lidar buatan orang Jerman bisa langsung dipakai algoritma navigasi buatan orang Jepang.

Yang tidak diberikan ROS: ROS tidak membuat robot kalian pintar, tidak menyediakan algoritma, dan tidak menjamin apa pun soal waktu. ROS adalah pipa dan konvensi. Isi pipanya kalian yang tentukan.

## 2. Kenapa robot dibangun dari banyak program kecil

Robot paling sederhana pun mengerjakan beberapa hal sekaligus, dan masing-masing punya kecepatan sendiri.

| Bagian | Kecepatan wajar | Kalau terlambat |
|---|---|---|
| Loop kendali motor | 100 Hz | Robot oleng atau bergetar |
| Lidar | 10 Hz | Rintangan terlihat terlambat |
| Kamera | 30 Hz | Frame basi, tidak berguna |
| Path planner | 0.5 Hz, kadang lebih lambat | Tidak apa-apa, asal yang lain jalan terus |
| Rem darurat | secepat mungkin | Fatal |

Kalian bisa saja menulis semuanya dalam satu program dengan satu loop besar. Untuk tugas pertama itu bahkan lebih mudah. Masalahnya, loop tersebut akan berjalan pada kecepatan komponen paling lambat. Robot kalian menabrak dinding sambil menunggu path planner selesai berpikir.

Masalah kedua, kalau satu bagian crash, seluruh robot mati. Padahal path planner yang crash masih bisa ditoleransi, sedangkan rem darurat tidak.

Karena itu robot dibangun sebagai kumpulan program kecil yang berjalan sendiri-sendiri, masing-masing dengan kecepatannya sendiri, saling berkirim pesan. Di ROS, program kecil itu disebut node.

```mermaid
graph LR
    L(["node lidar<br/>10 Hz"]) --> S["/scan"]
    S --> P(["node planner<br/>0.5 Hz"])
    P --> C["/cmd_vel"]
    C --> M(["node motor<br/>100 Hz"])
    M --> O["/odom"]
    O --> P

    style L fill:#1b4332,color:#fff
    style P fill:#1b4332,color:#fff
    style M fill:#1b4332,color:#fff
    style S fill:#495057,color:#fff
    style C fill:#495057,color:#fff
    style O fill:#495057,color:#fff
```

Kotak lonjong adalah node, kotak persegi adalah topic. Perhatikan bahwa node planner yang lambat tidak memperlambat node motor yang cepat. Masing-masing berjalan sendiri.

## 3. Pergeseran cara berpikir yang paling penting

Ini bagian yang paling sering membuat mahasiswa dengan latar belakang pemrograman tersandung, jadi baca pelan-pelan.

| | Program biasa | Sistem ROS |
|---|---|---|
| Programnya apa | kode yang kalian tulis | graf node yang sedang berjalan |
| Titik masuk | satu, `main` | banyak, satu per node |
| Alur eksekusi | call stack yang bisa ditelusuri | tidak ada call stack lintas node |
| Urutan startup | ditentukan kode | tidak dijamin, bisa berbeda tiap kali |
| Cara debug | pasang breakpoint | periksa graf dari luar |
| Bug tersering | logika salah | nama topic salah, tipe beda, QoS bentrok |

Node bisa sempurna secara internal dan sistemnya tetap tidak jalan, karena node itu mengirim ke `/cmd_vel` sementara yang lain mendengarkan `/robot/cmd_vel`.

Karena itu materi ini mengajarkan alat inspeksi lebih dulu, sebelum mengajarkan cara menulis node. Di ROS, kemampuan melihat isi graf yang sedang berjalan lebih berharga daripada kemampuan menulis kode node.

## 4. Melihat graf yang sedang berjalan

Jalankan simulator kura-kura bawaan ROS. Kalian akan butuh tiga terminal.

```
┌─────────────────────────┬─────────────────────────┐
│ Terminal 1              │ Terminal 2              │
│                         │                         │
│ ros2 run turtlesim \    │ ros2 run turtlesim \    │
│   turtlesim_node        │   turtle_teleop_key     │
│                         │                         │
│ jendela kura-kura       │ klik di sini,           │
│ muncul                  │ tekan tombol panah      │
├─────────────────────────┴─────────────────────────┤
│ Terminal 3                                        │
│                                                   │
│ semua perintah inspeksi di bawah dijalankan       │
│ dari sini                                         │
└───────────────────────────────────────────────────┘
```

Terminal 1:

```bash
ros2 run turtlesim turtlesim_node
```

Terminal 2:

```bash
ros2 run turtlesim turtle_teleop_key
```

Klik terminal 2, lalu tekan tombol panah. Kura-kuranya bergerak. Kalian baru saja menjalankan graf berisi dua node yang saling berkirim pesan lewat topic dan service:

![Dua node saling berkomunikasi lewat topic dan service](media/ros2-docs/Nodes-TopicandService.gif)

### Tujuh perintah yang wajib hafal

Semua dijalankan dari terminal 3.

| Perintah | Menjawab pertanyaan |
|---|---|
| `ros2 node list` | Node apa saja yang hidup sekarang |
| `ros2 topic list` | Topic apa saja yang ada |
| `ros2 topic info /nama --verbose` | Siapa mengirim, siapa menerima, QoS-nya apa |
| `ros2 topic echo /nama` | Isi pesannya apa, secara langsung |
| `ros2 topic hz /nama` | Seberapa sering pesannya datang |
| `ros2 interface show tipe/msg/Nama` | Bentuk tipe pesannya seperti apa |
| `ros2 topic pub` | Kirim pesan manual tanpa menulis kode |

Coba satu per satu:

```bash
ros2 node list
ros2 topic list
ros2 topic info /turtle1/cmd_vel --verbose
ros2 topic echo /turtle1/cmd_vel
```

Perintah `echo` akan diam sampai kalian menekan tombol panah lagi di terminal 2. Coba tekan, dan perhatikan isinya muncul.

```bash
ros2 topic hz /turtle1/pose
ros2 interface show geometry_msgs/msg/Twist
```

Terakhir, kirim pesan langsung dari terminal, tanpa menulis kode sama sekali:

```bash
ros2 topic pub --once /turtle1/cmd_vel geometry_msgs/msg/Twist "{linear: {x: 2.0}, angular: {z: 1.0}}"
```

Kura-kuranya bergerak melengkung. Node teleop tidak tahu-menahu, dan node turtlesim juga tidak peduli pesan itu datang dari mana.

### Melihat grafnya sebagai gambar

ROS bisa menggambar sendiri graf yang sedang berjalan:

```bash
rqt_graph
```

Sebuah jendela terbuka dan menampilkan node beserta koneksinya:

![Jendela rqt_graph menampilkan node teleop dan turtlesim](media/ros2-docs/rqt_graph.png)

Secara bawaan sebagian elemen disembunyikan. Pilih `Nodes/Topics (all)` di kiri atas supaya topic ikut terlihat:

![Pilihan Nodes/Topics (all) di rqt_graph](media/ros2-docs/unhide.png)

Lalu matikan centang `Debug` supaya tampilannya bersih:

![Centang Debug di rqt_graph](media/ros2-docs/debug.png)

Biasakan membuka `rqt_graph` setiap kali sistem kalian tidak berperilaku sesuai harapan. Dua node yang seharusnya tersambung tapi tergambar terpisah adalah petunjuk paling cepat yang bisa kalian dapat.

Sebagian besar sesi debugging di kelas ini berakhir di salah satu dari tujuh perintah tabel di atas, atau di `rqt_graph`.

## 5. Node

Node adalah satu program yang ikut dalam graf. Satu node biasanya bertanggung jawab atas satu hal: membaca satu sensor, mengendalikan satu aktuator, menjalankan satu algoritma.

Node punya nama, dan nama itu harus unik dalam graf. Kalau dua node punya nama sama, keduanya tetap jalan tapi perilakunya jadi membingungkan.

Node bukan proses. Satu proses bisa memuat beberapa node. Untuk sekarang anggap saja satu node sama dengan satu proses, karena itu yang akan kalian tulis.

## 6. Topic

Topic adalah kanal bernama yang dilalui aliran pesan. Sifatnya satu arah dan anonim.

Bentuk paling sederhana, satu publisher ke satu subscriber:

![Satu publisher mengirim ke satu subscriber lewat sebuah topic](media/ros2-docs/Topic-SinglePublisherandSingleSubscriber.gif)

Tapi jumlahnya tidak dibatasi. Satu topic bisa punya banyak publisher dan banyak subscriber sekaligus:

![Beberapa publisher dan beberapa subscriber pada topic yang sama](media/ros2-docs/Topic-MultiplePublisherandMultipleSubscriber.gif)

Publisher tidak tahu siapa yang mendengarkan. Subscriber tidak tahu siapa yang mengirim. Bisa ada nol publisher, bisa banyak. Bisa ada nol subscriber, bisa banyak.

Sifat anonim ini sengaja, dan konsekuensinya besar. Kalian bisa menyisipkan node perekam pada topic mana pun tanpa mengubah satu baris kode pun di node yang sudah ada. Kalian juga bisa mengganti driver lidar asli dengan lidar simulasi, dan sisa sistem tidak menyadarinya.

Topic dipakai untuk data yang mengalir terus: bacaan sensor, perintah kecepatan, estimasi posisi. Kalau pertanyaannya "berapa nilainya sekarang", jawabannya topic.

Setiap topic punya satu tipe pesan, dan tipe itu tidak boleh berubah. Publisher yang mengirim tipe berbeda ke topic yang sama tidak akan tersambung.

## 7. Workspace dan package

Sebelum menulis node, kalian butuh tempat menaruhnya. Kode ROS hidup di dalam package, dan package hidup di dalam workspace.

```bash
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_python --license Apache-2.0 latihan_ros
```

Hasilnya:

```
ros2_ws/                      workspace, tempat kalian menjalankan colcon build
├── src/                      satu-satunya folder yang kalian tulis sendiri
│   └── latihan_ros/          package
│       ├── package.xml       metadata dan dependensi
│       ├── setup.py          daftar program yang bisa dijalankan
│       ├── setup.cfg
│       ├── resource/
│       └── latihan_ros/      kode Python kalian di sini
│           └── __init__.py
├── build/                    dibuat colcon, jangan disentuh
├── install/                  dibuat colcon, di sini setup.bash berada
└── log/                      dibuat colcon, berguna saat build gagal
```

Dua file yang akan sering kalian sentuh adalah `package.xml`, tempat mendaftarkan dependensi, dan `setup.py`, tempat mendaftarkan program yang bisa dijalankan.

## 8. Node pertama: publisher

Buat file `~/ros2_ws/src/latihan_ros/latihan_ros/talker.py`:

```python
import rclpy
from rclpy.node import Node
from std_msgs.msg import String


class Talker(Node):
    def __init__(self):
        super().__init__('talker')
        self.pub = self.create_publisher(String, 'chatter', 10)
        self.timer = self.create_timer(0.5, self.timer_callback)
        self.count = 0

    def timer_callback(self):
        msg = String()
        msg.data = f'halo {self.count}'
        self.pub.publish(msg)
        self.get_logger().info(f'mengirim: {msg.data}')
        self.count += 1


def main():
    rclpy.init()
    node = Talker()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

| Baris | Kenapa begitu |
|---|---|
| `create_publisher(String, 'chatter', 10)` | Angka 10 adalah queue depth, bagian dari QoS. Lihat bagian 13 |
| `create_timer(0.5, self.timer_callback)` | Jangan pernah pakai `while True` dengan `sleep` di dalam node. Timer yang mengatur ritme |
| `rclpy.spin(node)` | Menyerahkan kendali ke ROS. Tidak kembali sampai node dimatikan. Semua pekerjaan terjadi di dalam callback |
| `self.get_logger().info(...)` | Bukan `print()`. Log ini punya stempel waktu, punya level, bisa disaring, dan ikut terlihat saat dijalankan lewat launch file |

## 9. Node kedua: subscriber

Buat file `~/ros2_ws/src/latihan_ros/latihan_ros/listener.py`:

```python
import rclpy
from rclpy.node import Node
from std_msgs.msg import String


class Listener(Node):
    def __init__(self):
        super().__init__('listener')
        self.sub = self.create_subscription(String, 'chatter', self.listener_callback, 10)

    def listener_callback(self, msg):
        self.get_logger().info(f'menerima: {msg.data}')


def main():
    rclpy.init()
    node = Listener()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

Perhatikan bahwa subscriber tidak pernah menyebut node publisher. Yang disebut cuma nama topic dan tipe pesannya. Itulah yang dimaksud anonim di bagian 6.

## 10. Mendaftarkan dan membangun

Buka `setup.py`, cari bagian `entry_points`, isi seperti ini:

```python
    entry_points={
        'console_scripts': [
            'talker = latihan_ros.talker:main',
            'listener = latihan_ros.listener:main',
        ],
    },
```

Buka `package.xml`, tambahkan di bawah baris `<license>`:

```xml
  <depend>rclpy</depend>
  <depend>std_msgs</depend>
```

Build dari akar workspace, bukan dari dalam folder package:

```bash
cd ~/ros2_ws
colcon build --symlink-install
source install/setup.bash
```

`--symlink-install` membuat `install/` berisi tautan ke file sumber kalian, bukan salinannya. Hasilnya, mengubah isi file `.py` langsung berlaku tanpa build ulang. Build ulang tetap wajib kalau kalian mengubah `setup.py` atau `package.xml`, atau menambah file baru seperti launch file. Kalau sebelumnya pernah build tanpa flag ini, hapus dulu `build/` dan `install/` sebelum memakainya.

### Dua setup script yang berbeda

Ini sumber kebingungan yang paling sering muncul di minggu pertama.

| Script | Isinya | Kapan dijalankan |
|---|---|---|
| `/opt/ros/jazzy/setup.bash` | ROS itu sendiri | Otomatis, sudah ada di `~/.bashrc` |
| `~/ros2_ws/install/setup.bash` | Package buatan kalian | Manual, di setiap terminal baru, setiap kali |

Yang kedua sering terlupakan. Gejalanya `Package 'latihan_ros' not found` padahal build barusan sukses.

Sekarang jalankan.

```
┌──────────────────────────────┬──────────────────────────────┐
│ Terminal 1                   │ Terminal 2                   │
│                              │                              │
│ cd ~/ros2_ws                 │ cd ~/ros2_ws                 │
│ source install/setup.bash    │ source install/setup.bash    │
│ ros2 run latihan_ros \       │ ros2 run latihan_ros \       │
│   talker                     │   listener                   │
│                              │                              │
│ mengirim: halo 0             │ menerima: halo 0             │
│ mengirim: halo 1             │ menerima: halo 1             │
├──────────────────────────────┴──────────────────────────────┤
│ Terminal 3                                                  │
│ ros2 topic echo /chatter                                    │
│ ros2 topic hz /chatter                                      │
│ ros2 node info /talker                                      │
│ rqt_graph                                                   │
└─────────────────────────────────────────────────────────────┘
```

Kalau listener diam saja, jangan langsung membaca ulang kode kalian. Jalankan `ros2 topic list` dan `ros2 topic info /chatter --verbose` lebih dulu, atau buka `rqt_graph`. Sembilan dari sepuluh kali jawabannya kelihatan di situ.

## 11. Service

Topic cocok untuk data yang mengalir. Sebagian hal tidak berbentuk aliran, melainkan permintaan yang butuh jawaban. Contohnya: reset odometri, nyalakan lampu, hitung nilai ini.

![Satu klien mengirim request dan menerima response dari server](media/ros2-docs/Service-SingleServiceClient.gif)

Satu server bisa melayani banyak klien, tapi setiap pasangan tetap berbentuk satu permintaan dan satu jawaban:

![Beberapa klien memanggil service yang sama](media/ros2-docs/Service-MultipleServiceClient.gif)

Coba dengan turtlesim yang masih berjalan:

```bash
ros2 service list
ros2 service type /spawn
ros2 interface show turtlesim/srv/Spawn
ros2 service call /spawn turtlesim/srv/Spawn "{x: 2.0, y: 2.0, theta: 0.0, name: 'kura2'}"
```

Kura-kura kedua muncul.

### Menulis service server

Setiap tipe service terdiri dari dua bagian, request dan response, dipisah `---`. Lihat tipe yang akan kita pakai:

```bash
ros2 interface show std_srvs/srv/Trigger
```

```
---
bool success   # indicate successful run of triggered service
string message # informational, e.g. for error messages
```

Bagian request, yang ada di atas `---`, kosong. Pemanggil cukup bilang "kerjakan" tanpa membawa data apa pun.

Kita beri talker dari bagian 8 sebuah service yang mengembalikan hitungannya ke nol. Ganti isi `~/ros2_ws/src/latihan_ros/latihan_ros/talker.py` dengan:

```python
import rclpy
from rclpy.node import Node
from std_msgs.msg import String
from std_srvs.srv import Trigger


class Talker(Node):
    def __init__(self):
        super().__init__('talker')
        self.pub = self.create_publisher(String, 'chatter', 10)
        self.timer = self.create_timer(0.5, self.timer_callback)
        self.srv = self.create_service(Trigger, 'reset_counter', self.reset_callback)
        self.count = 0

    def timer_callback(self):
        msg = String()
        msg.data = f'halo {self.count}'
        self.pub.publish(msg)
        self.get_logger().info(f'mengirim: {msg.data}')
        self.count += 1

    def reset_callback(self, request, response):
        self.count = 0
        response.success = True
        response.message = 'hitungan kembali ke 0'
        return response


def main():
    rclpy.init()
    node = Talker()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

| Baris | Kenapa begitu |
|---|---|
| `create_service(Trigger, 'reset_counter', self.reset_callback)` | Tipe, nama, dan callback. Mirip subscriber, bedanya callback ini wajib memberi jawaban |
| `reset_callback(self, request, response)` | ROS menyerahkan isi permintaan dan objek jawaban yang masih kosong. Tugas kalian mengisi jawabannya |
| `return response` | Jawaban baru terkirim saat callback mengembalikannya. Kalau lupa, node server crash dengan `TypeError` begitu service dipanggil |

Tambahkan satu baris di `package.xml`, di bawah dua baris `<depend>` dari bagian 10:

```xml
  <depend>std_srvs</depend>
```

`package.xml` berubah, jadi build ulang dan source ulang. Lalu uji servernya dengan CLI, sebelum ada satu baris kode klien pun.

Terminal 1:

```bash
ros2 run latihan_ros talker
```

Terminal 2:

```bash
ros2 run latihan_ros listener
```

Terminal 3:

```bash
ros2 service call /reset_counter std_srvs/srv/Trigger
```

```
requester: making request: std_srvs.srv.Trigger_Request()

response:
std_srvs.srv.Trigger_Response(success=True, message='hitungan kembali ke 0')
```

Listener di terminal 2 kembali mencetak `menerima: halo 0`, lalu menghitung naik lagi.

Urutan ini disengaja. Server sudah terbukti benar lewat CLI, jadi kalau klien yang kalian tulis sebentar lagi tidak jalan, kesalahannya pasti ada di klien. Selalu uji satu sisi dengan lawan bicara yang sudah pasti benar.

### Menulis service client

Buat file `~/ros2_ws/src/latihan_ros/latihan_ros/reset_client.py`:

```python
import rclpy
from rclpy.node import Node
from std_srvs.srv import Trigger


class ResetClient(Node):
    def __init__(self):
        super().__init__('reset_client')
        self.client = self.create_client(Trigger, 'reset_counter')
        while not self.client.wait_for_service(timeout_sec=1.0):
            self.get_logger().info('menunggu service reset_counter...')

    def send_request(self):
        return self.client.call_async(Trigger.Request())


def main():
    rclpy.init()
    node = ResetClient()
    future = node.send_request()
    rclpy.spin_until_future_complete(node, future)
    response = future.result()
    node.get_logger().info(f'jawaban: {response.message}')
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

| Baris | Kenapa begitu |
|---|---|
| `create_client(Trigger, 'reset_counter')` | Tipe dan nama harus persis sama dengan milik server |
| `wait_for_service(timeout_sec=1.0)` | Menunggu paling lama satu detik, lalu kembali. Loop di sekelilingnya memberi kabar tiap detik, supaya klien tidak diam tanpa penjelasan |
| `call_async(Trigger.Request())` | Mengirim permintaan lalu langsung kembali tanpa menunggu jawaban. Yang dikembalikan adalah future, wadah yang nanti terisi jawaban |
| `rclpy.spin_until_future_complete(node, future)` | Menjalankan node sampai future terisi. Aman di sini karena dipanggil dari `main`, bukan dari dalam callback |

Daftarkan di `entry_points`, di bawah baris talker dan listener:

```python
            'reset_client = latihan_ros.reset_client:main',
```

`setup.py` berubah, jadi build ulang. Dengan talker masih berjalan:

```bash
ros2 run latihan_ros reset_client
```

```
[INFO] [1792029472.513369445] [reset_client]: menunggu service reset_counter...
[INFO] [1792029474.266488223] [reset_client]: jawaban: hitungan kembali ke 0
```

Baris `menunggu` bisa muncul sekali atau dua kali walaupun talker sudah hidup. Node yang baru lahir butuh beberapa detik untuk menemukan node lain di graf. Kalau barisnya muncul terus tanpa henti, servernya memang tidak ada. Periksa dengan `ros2 service list`.

### Memanggil service dari dalam callback

Satu peringatan penting. Jangan memanggil service dari dalam callback lalu menunggu hasilnya secara sinkron. Node kalian akan macet, karena thread yang seharusnya memproses balasan sedang kalian tahan di dalam callback. Gejalanya membingungkan, yaitu program menggantung tanpa pesan error apa pun. Kalau perlu memanggil service dari callback, pakai `call_async` dan tangani hasilnya lewat future.

Klien di atas tidak kena masalah ini karena menunggunya di `main`. Node sungguhan lebih sering memanggil service dari dalam callback, misalnya dari timer atau saat sebuah pesan datang. Di sana aturannya cuma satu: jangan menunggu apa pun. Itu termasuk `wait_for_service` dan `spin_until_future_complete`.

Contohnya node yang me-reset hitungan talker setiap lima detik. Buat file `~/ros2_ws/src/latihan_ros/latihan_ros/reset_berkala.py`:

```python
import rclpy
from rclpy.node import Node
from std_srvs.srv import Trigger


class ResetBerkala(Node):
    def __init__(self):
        super().__init__('reset_berkala')
        self.client = self.create_client(Trigger, 'reset_counter')
        self.timer = self.create_timer(5.0, self.timer_callback)

    def timer_callback(self):
        if not self.client.service_is_ready():
            self.get_logger().warn('service reset_counter belum ada')
            return
        future = self.client.call_async(Trigger.Request())
        future.add_done_callback(self.response_callback)

    def response_callback(self, future):
        self.get_logger().info(f'jawaban: {future.result().message}')


def main():
    rclpy.init()
    node = ResetBerkala()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

| Baris | Kenapa begitu |
|---|---|
| `service_is_ready()` | Mengecek tanpa menunggu. Kalau server belum ada, callback langsung selesai, dan timer mencoba lagi lima detik kemudian |
| `call_async(...)` lalu `add_done_callback(...)` | Callback timer selesai saat itu juga. Ketika jawaban datang, ROS memanggil `response_callback` dengan future yang sudah terisi |

Daftarkan `'reset_berkala = latihan_ros.reset_berkala:main',` di `entry_points`, build ulang, lalu jalankan bersama talker dan listener. Hitungan di listener kembali ke nol kira-kira setiap sepuluh pesan.

Penjelasan kenapa pemanggilan sinkron macet, beserta cara lain mengatasinya, ada di bagian 1 `lanjutan-ros2.md`.

## 12. Action

Service punya kelemahan: pemanggil menunggu tanpa kabar. Kalau permintaannya "jalan ke koordinat itu" dan butuh 30 detik, pola permintaan-balasan tidak memadai. Kalian butuh feedback, dan kalian butuh kemampuan membatalkan.

Perhatikan tiga aliran terpisah pada gambar berikut: goal yang dikirim sekali, feedback yang mengalir terus selama pengerjaan, dan result di akhir.

![Klien mengirim goal, menerima feedback berkala, lalu menerima result](media/ros2-docs/Action-SingleActionClient.gif)

Turtlesim punya satu action. Coba:

```bash
ros2 action list
ros2 action info /turtle1/rotate_absolute
ros2 interface show turtlesim/action/RotateAbsolute
ros2 action send_goal --feedback /turtle1/rotate_absolute turtlesim/action/RotateAbsolute "{theta: 3.14}"
```

Perhatikan aliran feedback yang muncul selama kura-kura berputar. Itu yang tidak bisa diberikan service.

### Menulis action server

Tipe action terdiri dari tiga bagian: goal, result, dan feedback. ROS menyediakan satu tipe contoh:

```bash
ros2 interface show example_interfaces/action/Fibonacci
```

```
# Goal
int32 order
---
# Result
int32[] sequence
---
# Feedback
int32[] sequence
```

Server berikut menghitung deret Fibonacci sepanjang `order`. Setiap detik ia mengirim deret yang sudah jadi sebagai feedback, lalu deret lengkapnya sebagai result. Isi hitungannya sengaja remeh, supaya perhatian kalian tertuju ke bentuk action-nya.

Buat file `~/ros2_ws/src/latihan_ros/latihan_ros/fibonacci_server.py`:

```python
import time

import rclpy
from rclpy.action import ActionServer
from rclpy.node import Node
from example_interfaces.action import Fibonacci


class FibonacciServer(Node):
    def __init__(self):
        super().__init__('fibonacci_server')
        self.server = ActionServer(self, Fibonacci, 'fibonacci', self.execute_callback)

    def execute_callback(self, goal_handle):
        order = goal_handle.request.order
        self.get_logger().info(f'goal diterima, order {order}')
        sequence = [0, 1]
        for i in range(1, order):
            sequence.append(sequence[i] + sequence[i - 1])
            feedback = Fibonacci.Feedback()
            feedback.sequence = sequence
            goal_handle.publish_feedback(feedback)
            time.sleep(1)
        goal_handle.succeed()
        result = Fibonacci.Result()
        result.sequence = sequence
        return result


def main():
    rclpy.init()
    node = FibonacciServer()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

| Baris | Kenapa begitu |
|---|---|
| `ActionServer(self, Fibonacci, 'fibonacci', self.execute_callback)` | Tipe, nama, dan callback yang mengerjakan goal |
| `goal_handle.request` | Isi goal yang dikirim klien |
| `goal_handle.publish_feedback(feedback)` | Kabar sementara. Boleh dikirim berkali-kali selama goal berjalan |
| `goal_handle.succeed()` | Menandai goal berhasil. Kalau lupa, ROS mencetak `Goal state not set, assuming aborted` dan klien menerima status `ABORTED`, walaupun result tetap dikembalikan |
| `time.sleep(1)` | Pura-pura kerja berat. Lihat catatan di bawah |

Bagian 8 melarang `sleep` di dalam node, dan di sini kita melanggarnya dengan sengaja. Akibatnya, selama satu goal dikerjakan, node ini tidak bisa mengerjakan hal lain. Server ini juga tidak bisa dibatalkan, karena tanpa pengaturan tambahan ROS menolak semua permintaan pembatalan. Untuk contoh sekecil ini tidak masalah. Cara yang benar dibahas di bagian 3 `lanjutan-ros2.md`.

Tambahkan `<depend>example_interfaces</depend>` di `package.xml` dan `'fibonacci_server = latihan_ros.fibonacci_server:main',` di `entry_points`, lalu build ulang. Uji dengan CLI, sama seperti service tadi.

Terminal 1:

```bash
ros2 run latihan_ros fibonacci_server
```

Terminal 2:

```bash
ros2 action send_goal --feedback /fibonacci example_interfaces/action/Fibonacci "{order: 5}"
```

Feedback muncul satu per detik dengan deret yang makin panjang, lalu ditutup result. Keluarannya dipotong di sini:

```
Goal accepted with ID: b09d5c0c9c604119b608b5169199fa7e

Feedback:
    sequence:
- 0
- 1
- 1

...

Result:
    sequence:
- 0
- 1
- 1
- 2
- 3
- 5

Goal finished with status: SUCCEEDED
```

### Menulis action client

Server tadi kalian uji dengan CLI. Klien diuji dengan cara sebaliknya, yaitu dengan server yang sudah pasti benar: milik turtlesim. Pastikan `turtlesim_node` sedang berjalan.

Buat file `~/ros2_ws/src/latihan_ros/latihan_ros/rotate_client.py`:

```python
import rclpy
from rclpy.action import ActionClient
from rclpy.node import Node
from turtlesim.action import RotateAbsolute


class RotateClient(Node):
    def __init__(self):
        super().__init__('rotate_client')
        self.client = ActionClient(self, RotateAbsolute, '/turtle1/rotate_absolute')

    def send_goal(self, theta):
        goal = RotateAbsolute.Goal()
        goal.theta = theta
        self.client.wait_for_server()
        return self.client.send_goal_async(goal, feedback_callback=self.feedback_callback)

    def feedback_callback(self, feedback_msg):
        self.get_logger().info(f'sisa putaran: {feedback_msg.feedback.remaining:.2f} rad')


def main():
    rclpy.init()
    node = RotateClient()

    goal_future = node.send_goal(1.57)
    rclpy.spin_until_future_complete(node, goal_future)
    goal_handle = goal_future.result()

    if not goal_handle.accepted:
        node.get_logger().info('goal ditolak')
    else:
        result_future = goal_handle.get_result_async()
        rclpy.spin_until_future_complete(node, result_future)
        result = result_future.result().result
        node.get_logger().info(f'selesai, berputar {result.delta:.2f} rad')

    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

| Baris | Kenapa begitu |
|---|---|
| `ActionClient(self, RotateAbsolute, '/turtle1/rotate_absolute')` | Tipe dan nama action milik turtlesim, sama dengan yang tadi dipakai di CLI |
| `wait_for_server()` | Menunggu sampai server ada, tanpa batas waktu. Kalau turtlesim mati, klien diam selamanya |
| `send_goal_async(goal, feedback_callback=...)` | Mengirim goal. Future yang dikembalikan berisi jawaban pertama server: goal diterima atau ditolak |
| `goal_handle.get_result_async()` | Future kedua, berisi result setelah goal selesai dikerjakan |
| `feedback_callback` | Dipanggil setiap kali feedback datang, selama `spin_until_future_complete` berjalan |

Ada dua future karena action punya dua tahap. Goal diterima dulu, baru selesai belakangan. Server juga boleh menolak goal sejak awal, misalnya karena sedang sibuk.

Tambahkan `<depend>turtlesim</depend>` di `package.xml` dan `'rotate_client = latihan_ros.rotate_client:main',` di `entry_points`, build ulang, lalu jalankan:

```bash
ros2 run latihan_ros rotate_client
```

```
[INFO] [1792029523.386395103] [rotate_client]: sisa putaran: 1.57 rad
[INFO] [1792029523.393580223] [rotate_client]: sisa putaran: 1.55 rad
...
[INFO] [1792029524.929938270] [rotate_client]: sisa putaran: 0.02 rad
[INFO] [1792029524.930237698] [rotate_client]: selesai, berputar -1.55 rad
```

Feedback datang puluhan kali per detik, mengikuti kecepatan turtlesim memperbarui posisinya. Ada dua hal di sini yang sering membingungkan:

- Tanda minus pada result bukan kesalahan. Definisinya di `ros2 interface show turtlesim/action/RotateAbsolute` berbunyi "angular displacement in radians to the starting position", yaitu sudut untuk kembali ke posisi awal. Arahnya berlawanan dengan putaran yang barusan terjadi.
- Kalau dijalankan dua kali, kura-kura nyaris tidak bergerak di percobaan kedua. `theta` adalah arah mutlak, bukan tambahan, dan kura-kuranya sudah menghadap ke sana.

Dokumentasi resmi menulis klien yang sama dengan rantai callback, yaitu pola `add_done_callback` seperti di `reset_berkala`. Pakai pola itu kalau goal dikirim dari dalam callback, karena aturan di bagian 11 berlaku juga untuk action: jangan menunggu di dalam callback.

### Memilih di antara ketiganya

| | Topic | Service | Action |
|---|---|---|---|
| Arah | satu arah | bolak-balik | bolak-balik, berkali-kali |
| Pemanggil menunggu | tidak | ya | tidak, tapi dapat kabar |
| Bisa dibatalkan | tidak relevan | tidak | ya |
| Feedback | tidak | tidak | ya |
| Cocok untuk | data mengalir terus | permintaan cepat | tugas panjang |
| Contoh | `/scan`, `/cmd_vel`, `/odom` | reset odometri, spawn objek | navigasi ke titik, angkat lengan |

Kalian akan banyak memakai action nanti saat masuk ke navigasi. Perintah "pergi ke titik ini" di Nav2 adalah sebuah action.

## 13. QoS

Bagian ini sering dilewati mahasiswa, lalu membuang waktu berjam-jam karena topic yang kelihatannya benar tapi tidak menyambung.

QoS singkatan dari Quality of Service, yaitu sekumpulan pengaturan yang menentukan bagaimana pesan dikirim. Publisher punya pengaturan QoS, subscriber juga. Kalau keduanya tidak kompatibel, koneksinya tidak terbentuk sama sekali. Tidak ada node yang crash. Yang muncul cuma satu baris `[WARN]` di terminal masing-masing node, dan baris itu gampang tenggelam di antara log lain. Topic tetap muncul di `ros2 topic list`, dan data tidak pernah sampai.

### Tiga pengaturan yang perlu dikenal sekarang

| Pengaturan | Pilihan | Artinya | Dipakai untuk |
|---|---|---|---|
| Reliability | `RELIABLE` | Pesan dikirim ulang sampai sampai | Perintah kecepatan, hal yang tidak boleh hilang |
| | `BEST_EFFORT` | Pesan hilang ya sudah | Kamera resolusi tinggi, frame basi tidak berguna |
| Durability | `VOLATILE` | Subscriber baru tidak dapat apa-apa sampai pesan berikutnya | Data yang terus mengalir |
| | `TRANSIENT_LOCAL` | Publisher menyimpan pesan terakhir, langsung dikirim ke subscriber baru | Peta, data yang jarang berubah |
| Depth | angka | Berapa pesan ditahan di antrean | Ini angka `10` di kode kalian |

### Matriks kompatibilitas

Subscriber tidak boleh menuntut jaminan yang lebih ketat daripada yang ditawarkan publisher.

| Publisher | Subscriber | Tersambung |
|---|---|---|
| `RELIABLE` | `RELIABLE` | ya |
| `RELIABLE` | `BEST_EFFORT` | ya |
| `BEST_EFFORT` | `BEST_EFFORT` | ya |
| `BEST_EFFORT` | `RELIABLE` | **tidak** |
| `TRANSIENT_LOCAL` | `VOLATILE` | ya |
| `VOLATILE` | `TRANSIENT_LOCAL` | **tidak** |

Dua baris bertanda tebal itu yang akan memakan waktu kalian. Cara memeriksanya:

```bash
ros2 topic info /nama_topic --verbose
```

Perintah itu menampilkan profil QoS kedua sisi. Bandingkan. Kalau publisher dan subscriber sama-sama ada tapi data tidak mengalir, di sinilah jawabannya.

### Membuat QoS bentrok dengan sengaja

`ros2 topic pub` dan `ros2 topic echo` menerima pengaturan QoS lewat flag. Pakai itu untuk membuat ulang baris keempat matriks di atas: publisher `BEST_EFFORT`, subscriber `RELIABLE`.

Terminal 1:

```bash
ros2 topic pub /uji_qos std_msgs/msg/String "{data: halo}" --qos-reliability best_effort
```

Terminal 2:

```bash
ros2 topic echo /uji_qos --qos-reliability reliable
```

Terminal 2 tidak menampilkan data apa pun. Yang muncul cuma satu baris ini:

```
[WARN] [1792029563.379378941] [_ros2cli_91124]: New publisher discovered on topic '/uji_qos', offering incompatible QoS. No messages will be received from it. Last incompatible policy: RELIABILITY
```

Terminal 1 mencetak pasangannya: `New subscription discovered on topic '/uji_qos', requesting incompatible QoS. No messages will be sent to it.` Ujung kedua baris itu menyebut pengaturan yang bentrok, di sini `RELIABILITY`.

Periksa juga dari terminal 3 dengan `ros2 topic info /uji_qos --verbose`. Cari baris `Reliability` di bawah masing-masing `Endpoint type`:

```
Endpoint type: PUBLISHER
...
  Reliability: BEST_EFFORT
...
Endpoint type: SUBSCRIPTION
...
  Reliability: RELIABLE
```

Hentikan terminal 2, lalu jalankan ulang dengan `--qos-reliability best_effort`. Data langsung mengalir.

Satu catatan. Tanpa flag QoS, `ros2 topic echo` menyesuaikan diri dengan publisher yang sudah ada, jadi hampir selalu tersambung. Node buatan kalian tidak begitu. QoS-nya persis seperti yang tertulis di kode, sehingga bisa saja `echo` menerima data sementara node kalian tidak.

### Mengatur QoS di kode

Angka `10` di bagian 8 adalah singkatan. ROS mengartikannya sebagai profil dengan depth 10, `RELIABLE`, dan `VOLATILE`. Kalau butuh pengaturan lain, tulis profilnya lengkap:

```python
from rclpy.qos import QoSProfile, ReliabilityPolicy, DurabilityPolicy

qos = QoSProfile(
    depth=10,
    reliability=ReliabilityPolicy.BEST_EFFORT,
    durability=DurabilityPolicy.VOLATILE,
)
self.pub = self.create_publisher(String, 'chatter', qos)
```

`create_subscription` menerima profil di posisi yang sama. Coba pasang profil di atas ke publisher talker sambil membiarkan listener tetap memakai `10`. Listener diam, dan kedua terminal mencetak `[WARN]` yang sama seperti percobaan CLI tadi. Kembalikan talker ke `10` setelah selesai.

## 14. Parameter

Nilai yang mungkin ingin kalian ubah tanpa mengubah kode sebaiknya jadi parameter. Contohnya kecepatan maksimum, nama frame, nama port serial.

Deklarasikan di dalam konstruktor node:

```python
self.declare_parameter('max_speed', 1.0)
```

Bacanya:

```python
kecepatan = self.get_parameter('max_speed').value
```

Dari terminal:

```bash
ros2 param list
ros2 param get /nama_node max_speed
ros2 param set /nama_node max_speed 2.5
```

Nilai yang di-hardcode akan menyulitkan kalian sendiri nanti, terutama saat pindah dari simulasi ke robot nyata. Angka yang berbeda antara simulasi dan perangkat keras sebaiknya jadi parameter sejak awal.

### Kapan nilai baru dipakai

Potongan kode di atas belum memperlihatkan hal yang paling sering menjebak: `ros2 param set` mengubah nilai yang disimpan node, tapi node belum tentu membacanya lagi.

Buat file `~/ros2_ws/src/latihan_ros/latihan_ros/param_talker.py`:

```python
import rclpy
from rclpy.node import Node
from std_msgs.msg import String


class ParamTalker(Node):
    def __init__(self):
        super().__init__('param_talker')
        self.declare_parameter('pesan', 'halo')
        self.declare_parameter('period', 0.5)
        period = self.get_parameter('period').value
        self.pub = self.create_publisher(String, 'chatter', 10)
        self.timer = self.create_timer(period, self.timer_callback)

    def timer_callback(self):
        msg = String()
        msg.data = self.get_parameter('pesan').value
        self.pub.publish(msg)


def main():
    rclpy.init()
    node = ParamTalker()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

Node ini membaca dua parameter di tempat yang berbeda. `period` dibaca sekali, di konstruktor. `pesan` dibaca di dalam callback, setiap kali timer berbunyi.

Daftarkan `'param_talker = latihan_ros.param_talker:main',` di `entry_points`, build ulang, lalu jalankan.

Terminal 1:

```bash
ros2 run latihan_ros param_talker
```

Terminal 2:

```bash
ros2 topic echo /chatter
```

Terminal 3:

```bash
ros2 param list /param_talker
ros2 param set /param_talker pesan "apa kabar"
```

`param list` menampilkan `period` dan `pesan`, ditambah `start_type_description_service` dan `use_sim_time` yang dimiliki setiap node. Begitu `param set` dijalankan, terminal 2 langsung berganti mencetak `data: apa kabar`.

Sekarang ubah periodenya:

```bash
ros2 param set /param_talker period 0.1
ros2 param get /param_talker period
ros2 topic hz /chatter
```

`set` menjawab `Set parameter successful` dan `get` menjawab `Double value is: 0.1`, tapi `hz` tetap di sekitar 2. Nilainya memang berubah. Hanya saja timer sudah dibuat di konstruktor dengan nilai lama, dan tidak ada baris kode yang membaca `period` lagi.

| Parameter dibaca di | Efek `ros2 param set` |
|---|---|
| Konstruktor | Nilai tersimpan, tapi tidak terpakai sampai node dijalankan ulang |
| Callback | Terpakai di panggilan callback berikutnya |

Nilai yang dibaca di konstruktor tetap bisa diatur, asalkan sebelum node hidup:

```bash
ros2 run latihan_ros param_talker --ros-args -p period:=0.1
```

Kali ini `ros2 topic hz /chatter` menunjukkan 10. Cara membuat node bereaksi saat parameter berubah di tengah jalan dibahas di bagian 6 `lanjutan-ros2.md`.

Terakhir, tipe parameter mengikuti nilai bawaannya. `period` dideklarasikan dengan `0.5`, jadi bertipe desimal, dan perintah berikut ditolak:

```bash
ros2 param set /param_talker period 1
```

```
Setting parameter failed: Wrong parameter type, expected 'Type.DOUBLE' got 'Type.INTEGER'
```

Tulis `1.0`.

## 15. Launch file

Setelah sistem kalian berisi lima node, menjalankannya satu per satu di lima terminal jadi menyiksa.

```mermaid
graph TD
    L["latihan.launch.py"] --> A(["talker"])
    L --> B(["listener"])

    style L fill:#023e8a,color:#fff
    style A fill:#1b4332,color:#fff
    style B fill:#1b4332,color:#fff
```

Buat folder `launch` di dalam package, lalu file `latihan.launch.py`:

```python
from launch import LaunchDescription
from launch_ros.actions import Node


def generate_launch_description():
    return LaunchDescription([
        Node(
            package='latihan_ros',
            executable='talker',
            name='talker',
        ),
        Node(
            package='latihan_ros',
            executable='listener',
            name='listener',
        ),
    ])
```

Supaya file ini ikut terpasang, tambahkan ke `data_files` di `setup.py`:

```python
import os
from glob import glob

    data_files=[
        ('share/ament_index/resource_index/packages', ['resource/' + package_name]),
        ('share/' + package_name, ['package.xml']),
        (os.path.join('share', package_name, 'launch'), glob('launch/*.launch.py')),
    ],
```

Build ulang, source, jalankan:

```bash
cd ~/ros2_ws && colcon build --symlink-install && source install/setup.bash
ros2 launch latihan_ros latihan.launch.py
```

Launch file bisa jauh lebih pintar dari contoh ini. Bisa menerima argumen, memuat file parameter, mengganti nama topic, dan memanggil launch file lain. Untuk sekarang cukup tahu bahwa file ini yang akan kalian pakai untuk menyalakan seluruh robot. Semua kemampuan tadi dibahas di bagian 7 `lanjutan-ros2.md`.

## 16. Merekam dan memutar ulang

Karena semua yang penting lewat topic, seluruhnya bisa direkam.

```mermaid
graph LR
    subgraph rekam["Saat merekam"]
        A(["robot atau simulasi"]) --> T1["/scan, /odom"]
        T1 --> B(["ros2 bag record"])
        B --> F[("file bag")]
    end

    subgraph putar["Saat memutar ulang"]
        F2[("file bag")] --> C(["ros2 bag play"])
        C --> T2["/scan, /odom"]
        T2 --> D(["algoritma kalian"])
    end

    style F fill:#023e8a,color:#fff
    style F2 fill:#023e8a,color:#fff
```

```bash
ros2 bag record -o rekaman1 /chatter
```

Hentikan dengan Ctrl-C. Lalu:

```bash
ros2 bag info rekaman1
ros2 bag play rekaman1
```

Saat diputar ulang, node lain tidak bisa membedakan data rekaman dengan data asli. Kalian bisa merekam sekali dari robot nyata, lalu mengembangkan algoritma di meja kerja dengan data yang sama persis berulang kali.

Tugas mingguan di kelas ini dikumpulkan dalam bentuk bag. Sistem penilaian memutar ulang rekaman kalian dan memeriksa topic serta isinya. Karena itu, apa pun sistem operasi yang kalian pakai, penilaiannya sama.

## 17. Kesalahan yang paling sering terjadi

| Gejala | Penyebab | Solusi |
|---|---|---|
| `Package 'latihan_ros' not found` padahal build sukses | Lupa source `install/setup.bash` di terminal baru | `cd ~/ros2_ws && source install/setup.bash` |
| Build aneh, hasil tidak berubah | `colcon build` dijalankan dari dalam folder package | Selalu build dari `~/ros2_ws` |
| Kode `.py` sudah diubah, perilaku node tetap lama | Build tanpa `--symlink-install`, jadi yang jalan salinan lama | Build ulang dengan `colcon build --symlink-install` |
| Build sukses, `ros2 run` tidak menemukan program | Lupa mendaftarkan di `entry_points` | Isi `setup.py`, build ulang |
| Node hidup, topic ada, data tidak mengalir | QoS tidak kompatibel | Cari baris `[WARN]` berisi `incompatible QoS` di log, lalu `ros2 topic info --verbose` dan bandingkan kedua sisi |
| Klien terus mencetak `menunggu service` | Server belum hidup, atau nama service berbeda | `ros2 service list`, samakan nama |
| Node server crash dengan `TypeError` begitu service dipanggil | Callback service lupa `return response` | Tambahkan `return response` di akhir callback |
| Action client diam tanpa output | `wait_for_server()` menunggu server yang tidak ada | Jalankan servernya, cek `ros2 action list` |
| `ros2 param set` sukses, perilaku node tidak berubah | Nilainya dibaca sekali di konstruktor | Baca di dalam callback, atau atur saat start dengan `--ros-args -p` |
| `Wrong parameter type, expected 'Type.DOUBLE' got 'Type.INTEGER'` | Menulis `1` untuk parameter bertipe desimal | Tulis `1.0` |
| `Node not found` saat `ros2 param set` ke node yang baru dijalankan | Node belum sempat ditemukan | Tunggu sebentar, ulangi |
| Node saling tidak melihat sama sekali | `ROS_DOMAIN_ID` berbeda antar terminal | `echo $ROS_DOMAIN_ID` di semua terminal, samakan |
| Topic tersambung di satu tempat, tidak di tempat lain | Beda `chatter` dan `/chatter`. Yang pertama relatif terhadap namespace, kedua absolut | Perhatikan garis miring di depan |
| Output tidak muncul saat pakai launch file | Memakai `print()` | Ganti dengan `self.get_logger().info()` |
| Seluruh node macet | Callback menahan eksekusi terlalu lama, atau memanggil service secara sinkron dari dalam callback | Pindahkan kerja berat ke node lain, pakai `call_async` |

## Bacaan lanjutan

Kelanjutan materi ini ada di `lanjutan-ros2.md`.

Materi ini adalah ringkasan yang disusun untuk urutan perkuliahan kita. Penjelasan resmi yang lebih panjang ada di dokumentasi ROS 2, dan berguna kalau kalian ingin menggali satu topik lebih dalam.

| Topik | Halaman resmi |
|---|---|
| Node | [Understanding nodes](https://docs.ros.org/en/jazzy/Tutorials/Beginner-CLI-Tools/Understanding-ROS2-Nodes/Understanding-ROS2-Nodes.html) |
| Topic | [Understanding topics](https://docs.ros.org/en/jazzy/Tutorials/Beginner-CLI-Tools/Understanding-ROS2-Topics/Understanding-ROS2-Topics.html) |
| Service | [Understanding services](https://docs.ros.org/en/jazzy/Tutorials/Beginner-CLI-Tools/Understanding-ROS2-Services/Understanding-ROS2-Services.html) |
| Service di kode | [Writing a simple service and client (Python)](https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Writing-A-Simple-Py-Service-And-Client.html) |
| Action | [Understanding actions](https://docs.ros.org/en/jazzy/Tutorials/Beginner-CLI-Tools/Understanding-ROS2-Actions/Understanding-ROS2-Actions.html) |
| Action di kode | [Writing an action server and client (Python)](https://docs.ros.org/en/jazzy/Tutorials/Intermediate/Writing-an-Action-Server-Client/Py.html) |
| Parameter | [Understanding parameters](https://docs.ros.org/en/jazzy/Tutorials/Beginner-CLI-Tools/Understanding-ROS2-Parameters/Understanding-ROS2-Parameters.html) |
| Parameter di kode | [Using parameters in a class (Python)](https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Using-Parameters-In-A-Class-Python.html) |
| QoS | [About Quality of Service settings](https://docs.ros.org/en/jazzy/Concepts/Intermediate/About-Quality-of-Service-Settings.html) |
| Bag | [Recording and playing back data](https://docs.ros.org/en/jazzy/Tutorials/Beginner-CLI-Tools/Recording-And-Playing-Back-Data/Recording-And-Playing-Back-Data.html) |

## Catatan teknis dan atribusi

Animasi dan screenshot di bagian 4, 6, 11, dan 12 diambil dari dokumentasi resmi ROS 2, filenya tersimpan di folder `media/ros2-docs/`. Sumbernya adalah repositori [ros2/ros2_documentation](https://github.com/ros2/ros2_documentation) cabang `jazzy`, di bawah lisensi [Creative Commons Attribution 4.0 International](https://creativecommons.org/licenses/by/4.0/). Filenya kami salin apa adanya tanpa modifikasi.

File disalin ke folder lokal, bukan ditautkan langsung ke internet, supaya materi tetap tampil utuh tanpa koneksi internet.

Diagram selebihnya, yaitu peta materi di awal, diagram beda kecepatan node di bagian 2, diagram launch file di bagian 15, dan diagram bag di bagian 16, ditulis dengan sintaks mermaid dan khusus dibuat untuk mata kuliah ini. Diagram mermaid tampil sebagai gambar kalau file dibuka lewat GitHub, GitLab, Obsidian, atau pratinjau markdown di VS Code. Di penampil markdown sederhana, isinya tampil sebagai teks dan masih terbaca.
