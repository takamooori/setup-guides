# GLIM GPU セットアップマニュアル

## 前提条件

| 項目 | バージョン |
|------|-----------|
| OS | Ubuntu 22.04 (Jammy) |
| Docker ベースイメージ | `nvidia/cuda:12.2.0-cudnn8-devel-ubuntu22.04` |
| ホストドライバ対応CUDA | 12.2 |
| GPU | NVIDIA GeForce GTX 1650 |
| ROS2 | Humble |

> ⚠️ CUDA=ON でビルドするには CUDA 12.x 環境が必須。CUDA 11.8 では `thrust::cuda::par_nosync` エラーが発生する。

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

## 1. 依存パッケージのインストール

```bash
sudo apt update
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

## 2. Iridescence（可視化ライブラリ）のインストール

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

## 3. GTSAM のインストール（v4.3a0）

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

✅ 確認：checkout 後に `git log --oneline -1` で `4.3a0` タグのコミットが表示されること
```bash
git log --oneline -1
# 例: abc1234 (HEAD, tag: 4.3a0) ...
```

---

## 4. gtsam_points のインストール

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

---

## 5. 共有ライブラリをシステムに反映

```bash
sudo ldconfig
```

✅ 確認：エラーなく完了すること

---

## 6. GLIM ROS2 パッケージの clone

> `glim` / `glim_ros2` は `ros2_ws/src` に clone する

```bash
cd ~/ros2_ws/src
git clone https://github.com/koide3/glim
git clone https://github.com/koide3/glim_ros2
```

✅ 確認：2つのディレクトリが作成されていること
```bash
ls ~/ros2_ws/src | grep glim
# glim
# glim_ros2
```

---

## 7. ビルド

```bash
cd ~/ros2_ws
source /opt/ros/humble/setup.bash
colcon build
source ~/.bashrc
```

✅ 確認：ビルド完了後に以下が表示されること
```
Summary: XX packages finished
```
`Failed` や `Aborted` がないこと

---

## 備考

- iridescence / gtsam / gtsam_points は `sudo make install` でシステムにインストールされるため、clone 場所はどこでもよい（`ros2_ws` 内への clone は非推奨）
- CUDA=OFF でビルドした場合は GPU 高速化なしで動作可能（CPU モード）
- CUDA=ON でビルドに失敗する場合は Docker ベースイメージの CUDA バージョンを確認する（12.x 必須）
