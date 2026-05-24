# GLIM セットアップマニュアル

## 前提条件

| 項目 | バージョン |
|------|-----------|
| OS | Ubuntu 22.04 (Jammy) |
| Docker ベースイメージ | `nvidia/cuda:12.2.2-cudnn8-devel-ubuntu22.04` |
| ホストドライバ対応CUDA | 12.2 |
| GPU | NVIDIA GeForce GTX 1650 |
| ROS2 | Humble |

---

## 事前確認（コンテナ起動後に必ず実施）

### CUDA バージョン確認
```bash
nvcc --version
```
✅ 期待値：`release 12.2` と表示されること

### GPU・ドライバ確認
```bash
nvidia-smi
```
✅ 期待値：`CUDA Version: 12.2`、GPU が認識されていること

### OS 確認
```bash
cat /etc/os-release | grep VERSION
```
✅ 期待値：`VERSION_ID="22.04"`

### ROS2 確認
```bash
source /opt/ros/humble/setup.bash
ros2 --version
```
✅ 期待値：`ros2 humble` と表示されること

---

## 1. システムの更新

```bash
sudo apt update
sudo apt upgrade -y
```

✅ 確認：エラーなく完了すること

---

## 2. 依存パッケージのインストール

```bash
sudo apt install -y \
  libomp-dev \
  libboost-all-dev \
  libmetis-dev \
  libfmt-dev \
  libspdlog-dev \
  libglm-dev \
  libglfw3-dev \
  libpng-dev \
  libjpeg-dev
```

✅ 確認：エラーなくインストール完了すること

---

## 3. Iridescence（可視化ライブラリ）のインストール

> `ros2_ws` の外（ホームディレクトリ直下推奨）で実行する

```bash
cd ~
git clone https://github.com/koide3/iridescence --recursive
mkdir iridescence/build && cd iridescence/build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
sudo make install
```

✅ 確認：`make install` がエラーなく完了すること

---

## 4. GTSAM のインストール（v4.3a0）

```bash
cd ~
git clone https://github.com/borglab/gtsam
cd gtsam && git checkout 4.3a0
mkdir build && cd build
cmake .. \
  -DGTSAM_BUILD_EXAMPLES_ALWAYS=OFF \
  -DGTSAM_BUILD_TESTS=OFF \
  -DGTSAM_WITH_TBB=OFF \
  -DGTSAM_USE_SYSTEM_EIGEN=ON \
  -DGTSAM_BUILD_WITH_MARCH_NATIVE=OFF
make -j$(nproc)
sudo make install
```

✅ 確認：checkout 後に以下で `4.3a0` タグのコミットが表示されること
```bash
git log --oneline -1
# 例: abc1234 (HEAD, tag: 4.3a0) ...
```

---

## 5. gtsam_points のインストール

> ⚠️ GPU / CPU でコマンドが異なる。環境に合わせて選択すること。

### 🟢 GPU モード（CUDA 12.x 環境）

```bash
cd ~
git clone https://github.com/koide3/gtsam_points
mkdir gtsam_points/build && cd gtsam_points/build
cmake .. -DBUILD_WITH_CUDA=ON
make -j$(nproc)
sudo make install
```

✅ 確認：cmake 時に以下が表示されること
```
-- Build with CUDA: ON
```

### 🔵 CPU モード（CUDA 環境がない場合）

```bash
cd ~
git clone https://github.com/koide3/gtsam_points
mkdir gtsam_points/build && cd gtsam_points/build
cmake .. -DBUILD_WITH_CUDA=OFF
make -j$(nproc)
sudo make install
```

✅ 確認：cmake 時に以下が表示されること
```
-- Build with CUDA: OFF
```

---

## 6. 共有ライブラリをシステムに反映

```bash
sudo ldconfig
```

✅ 確認：エラーなく完了すること

---

## 7. GLIM ROS2 パッケージの clone

```bash
cd ~/ros2_ws/src
git clone https://github.com/koide3/glim
git clone https://github.com/koide3/glim_ros2
```

✅ 確認：
```bash
ls ~/ros2_ws/src | grep glim
# glim
# glim_ros2
```

---

## 8. Config ファイルの設定

### センサートピックの変更（全モード共通）

ファイルパス：`ros2_ws/src/glim/config/config_ros.json`

使用しているセンサーのトピック名に合わせて変更する。

**変更前**
```json
"imu_topic": "/os_cloud_node/imu",
"points_topic": "/os_cloud_node/points"
```

**変更後（例：Sony製IMU + Livox LiDAR使用時）**
```json
"imu_topic": "/imu/spresense",
"points_topic": "/converted_pointcloud2"
```

---

### 🔵 CPU モードの場合のみ：config.json の変更

ファイルパス：`ros2_ws/src/glim/config/config.json`

デフォルトは GPU モードになっているため、CPU で動かす場合は以下に変更する。

**変更前（GPU モード・デフォルト）**
```json
"config_odometry": "config_odometry_gpu.json",
"config_sub_mapping": "config_sub_mapping_gpu.json",
"config_global_mapping": "config_global_mapping_gpu.json"
```

**変更後（CPU モード）**
```json
"config_odometry": "config_odometry_cpu.json",
"config_sub_mapping": "config_sub_mapping_passthrough.json",
"config_global_mapping": "config_global_mapping_pose_graph.json"
```

---

## 9. ビルド

```bash
cd ~/ros2_ws
source /opt/ros/humble/setup.bash
colcon build
source ~/.bashrc
```

✅ 確認：
```
Summary: XX packages finished
```
`Failed` や `Aborted` がないこと

---

## 10. 実行

### 実行コマンド

**ターミナル1：GLIMノードの起動**
```bash
ros2 run glim_ros glim_rosnode --ros-args -p use_simtime:=true
```

**ターミナル2：Livox LiDAR → PointCloud2 変換ノードの起動**
```bash
ros2 launch livox_to_pointcloud2 livox_to_pointcloud2.launch.py
```

**ターミナル3：rosbag の再生**
```bash
ros2 bag play Downloads/1005kakuninmanual/ --clock
```

### 注意事項

- tfの接続が以下の形になっている必要がある（崩れるとエラーが出る）
  ```
  map → odom → imu_link → livox_frame
  ```
- 3D-LiDARのinputトピックは **PointCloud2形式が必須**。Livox の CustomMsg（`/livox/lidar`）はそのまま使えないため `livox_to_pointcloud2` パッケージで変換が必要
- `glim_rosbag` ノードは他ターミナルとのトピック通信を行わないため、bagデータに **pointcloud2トピックとimuトピックの両方**を含める必要がある
- timeの整合性が重要なため `use_simtime:=true` と `--clock` オプションはセットで使うこと
- `acc_scale` の設定：LiDARのz軸上方向を基準として、IMUのz軸が上向きなら `9.81`、逆向きなら `0.00` に設定する

---

## 備考

- iridescence / gtsam / gtsam_points は `sudo make install` でシステムにインストールされるため、clone 場所はどこでもよい（`ros2_ws` 内への clone は非推奨）
- CUDA=ON でビルドに失敗する場合は Docker ベースイメージの CUDA バージョンを確認する（12.x 必須）
