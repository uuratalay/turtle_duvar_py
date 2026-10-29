# turtle_duvar_py — Duvardan Kaçan Kaplumbağa

ROS2 Humble (Ubuntu 22.04) için `ament_python` paketi. `turtlesim` içindeki
kaplumbağayı sürekli ileri sürer; konumu (`/turtle1/pose`) sınırların
(varsayılan 1.0–10.0) dışına çıkmaya yaklaştığında dönerek yön değiştirir.
Konum saniyede bir log'a yazılır.

## Mimari

| Dosya | Görev |
|---|---|
| `turtle_duvar_py/control_logic.py` | Saf karar mantığı (`compute_cmd`) — ROS'a bağlı değil, düz `python3` ile test edilebilir |
| `turtle_duvar_py/turtle_duvar_node.py` | `rclpy` node'u: `/turtle1/pose` aboneliği, `/turtle1/cmd_vel` yayını, 1 Hz log timer'ı |
| `test/test_control_logic.py` | 9 birim testi — sınır davranışını (duvara değme, tam sınırda, özel parametreler) doğrular |

Mantığın ROS I/O'dan ayrılması bilinçli bir tercih: `compute_cmd` fonksiyonu
`rclpy` kurulu olmayan bir ortamda bile test edilebiliyor (bu paket de bu
şekilde test edildi, aşağıya bakın).

## Node Davranışı

- **Abone:** `/turtle1/pose` (`turtlesim/msg/Pose`) → güncel `x`, `y` okunur.
- **Yayıncı:** `/turtle1/cmd_vel` (`geometry_msgs/msg/Twist`) → her pose
  güncellemesinde yeniden hesaplanıp yayınlanır.
- **Karar kuralı:** `x < min_limit` veya `x > max_limit` veya `y < min_limit`
  veya `y > max_limit` ise `angular.z = angular_speed` (dönerek yön
  değiştirir); aksi halde düz gider (`angular.z = 0`). `linear.x` her zaman
  `linear_speed`.
- **Log:** 1 Hz timer, en son bilinen `x`, `y` değerini `get_logger().info()`
  ile yazar.

## Parametreler (Bonus)

| Parametre | Varsayılan | Açıklama |
|---|---|---|
| `linear_speed` | `1.5` | İleri hız (m/s) |
| `angular_speed` | `4.0` | Dönüş açısal hızı (rad/s) |
| `min_limit` | `1.0` | Alt sınır |
| `max_limit` | `10.0` | Üst sınır |

> **Neden `linear_speed=1.5`, `angular_speed=4.0`?** Dönüş yarıçapı
> `r = linear_speed / angular_speed`'tir. `min_limit`/`max_limit` ile
> turtlesim'in gerçek duvarı (0 / ~11.09) arasında yalnızca ~1.0 birimlik
> boşluk var; `r` bu boşluğun belirgin şekilde altında olmalı, yoksa
> kaplumbağa dönerken gerçek duvara çarpar (turtlesim'in kendi loglarında
> `Oh no! I hit the wall!` uyarısı olarak görünür). `1.5/4.0 = 0.375`
> güvenli marj bırakır. İlk denemede `2.0/1.0 = 2.0` ile bu tam olarak
> yaşandı — test ederken aynı uyarıyı görürsen `angular_speed`'i artır
> ya da `linear_speed`'i düşür.

## Bu Ortamda Yapılan Testler

Bu paket, ROS2 kurulu olmayan bir sandbox'ta hazırlandı, bu yüzden burada
doğrulanabilenler:

1. **Syntax kontrolü** — tüm `.py` dosyaları `python3 -m py_compile` ile
   derlendi, hata yok.
2. **Birim testleri** — `test/test_control_logic.py` içindeki 9 test
   (merkezde düz gitme, 4 duvarın her birinde dönme, tam sınırda güvenli
   kabul, özel parametrelerle davranış) `python3 -m unittest` ile koşturuldu:
   **9/9 geçti**.

`rclpy`/`colcon` bu sandbox'ta mevcut olmadığından node'un gerçek
`turtlesim` ile uçtan uca çalışması (`colcon build` + `ros2 run`) sizin
Ubuntu 22.04 + ROS2 Humble makinenizde yapılmalı — adımlar aşağıda.

## Kurulum ve Çalıştırma (Ubuntu 22.04 + ROS2 Humble)

```bash
# 1) Paketi workspace'inize yerleştirin
mkdir -p ~/robotik_ws/src
cp -r turtle_duvar_py ~/robotik_ws/src/

# 2) Bağımlılıkları kontrol edin (genelde ROS2 kurulumunda hazır gelir)
sudo apt update
rosdep install --from-paths ~/robotik_ws/src --ignore-src -r -y

# 3) Build
cd ~/robotik_ws
colcon build --packages-select turtle_duvar_py
source install/setup.bash

# 4) Birim testlerini burada da çalıştırabilirsiniz (opsiyonel)
colcon test --packages-select turtle_duvar_py
colcon test-result --verbose
```

### Çalıştırma — iki terminal

**Terminal 1** (turtlesim simülatörü):
```bash
source /opt/ros/humble/setup.bash
ros2 run turtlesim turtlesim_node
```

**Terminal 2** (bizim node):
```bash
source ~/robotik_ws/install/setup.bash
ros2 run turtle_duvar_py turtle_duvar
```

Kaplumbağa ekranda ileri gidip duvara yaklaştıkça dönerek yön değiştirmeli;
Terminal 2'de saniyede bir `Konum -> x: ..., y: ...` logu akmalı.

### Parametreleri değiştirerek çalıştırma (bonus'u göstermek için)

```bash
ros2 run turtle_duvar_py turtle_duvar --ros-args \
  -p linear_speed:=3.0 -p angular_speed:=1.5 \
  -p min_limit:=2.0 -p max_limit:=9.0
```

veya çalışırken canlı değiştirmek için:

```bash
ros2 param set /turtle_duvar linear_speed 1.0
```

## Teslim İçin Ekran Görüntüsü Önerisi

- Terminal 1 + Terminal 2 + turtlesim penceresi aynı ekranda, kaplumbağa bir
  duvara yaklaşıp dönerken.
- `colcon test-result --verbose` çıktısının ekran görüntüsü (testlerin
  geçtiğini gösterir).

## GitHub'a Yükleme

```bash
cd ~/robotik_ws/src/turtle_duvar_py
git init
git add .
git commit -m "turtle_duvar_py: duvardan kacan kaplumbaga odevi"
git branch -M main
git remote add origin <REPO_URL>
git push -u origin main
```
