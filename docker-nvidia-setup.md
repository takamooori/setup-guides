# Docker GPU対応セットアップマニュアル

GLIM（LiDAR SLAM）をDockerコンテナ内でNVIDIA GPUを使って動かすためのセットアップ手順。

## 環境情報

| 項目 | 内容 |
|------|------|
| PC | kbkn-RL5C-G50 |
| OS | Ubuntu 22.04 (jammy) |
| GPU | GTX 1650 Mobile (TU117M) |
| ROS2 | Humble |
| Docker repo | [orange2025_docker](https://github.com/KBKN-Autonomous-Robotics-Lab/orange2025_docker) |

## 目標

DockerコンテナでNVIDIA GPUを使えるようにして、GLIM（LiDAR SLAM）をGPUで動かす。

---

## 全体ステップ

### Step 1 ✅ GPUの存在確認

`lspci` で GTX 1650 Mobile を確認済み。

```bash
lspci | grep -i nvidia
```

---

### Step 2 ✅ NVIDIAドライバのインストール

Ubuntu 22.04 + GTX 1650 向けの推奨ドライバは `nvidia-driver-535`。

```bash
sudo apt update
sudo apt install -y nvidia-driver-535
sudo reboot
```

再起動後に確認：

```bash
nvidia-smi
```

出力右上に `CUDA Version: 12.2` のように表示されればOK。

---

### Step 3 　nvidia-container-toolkit のインストール

DockerコンテナがホストのGPUを使えるようにする橋渡し役。

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

---

### Step 4 　Dockerfileのベースイメージ変更

`nvidia-smi` の `CUDA Version` を確認してからベースイメージを決定する。

> CUDA Version が 12.x であれば `nvidia/cuda:11.8.0-cudnn8-devel-ubuntu22.04` を使用（後方互換性あり）。

```dockerfile
# 変更前
FROM ubuntu:jammy-20230522

# 変更後
FROM nvidia/cuda:11.8.0-cudnn8-devel-ubuntu22.04
```

ROS2インストール行の前に以下の環境変数を追記：

```dockerfile
ENV CUDA_HOME=/usr/local/cuda
ENV PATH=${CUDA_HOME}/bin:${PATH}
ENV LD_LIBRARY_PATH=${CUDA_HOME}/lib64:${LD_LIBRARY_PATH}
```

---

### Step 5 　runスクリプトに `--gpus` オプション追加

`livox_run.sh` と `livox_runLite.sh` の両方に以下の2行を追加する。

```bash
--gpus all \
-e NVIDIA_DRIVER_CAPABILITIES=all \
```

追加後の例：

```bash
docker run -it --rm \
  --gpus all \
  -e NVIDIA_DRIVER_CAPABILITIES=all \
  --net=host \
  ...
  kbkn202x/orange202x:latest
```

---

### Step 6 　リビルドと動作確認

```bash
bash build.sh
```

> ビルドには数時間かかる場合がある。

ビルド完了後、コンテナを起動して確認：

```bash
docker run --rm --gpus all kbkn202x/orange202x:latest nvidia-smi
```

`nvidia-smi` の出力が表示されればGPU対応完了。

---

## 注意事項

- **CUDAバージョンの選択**: Step 4のベースイメージは `nvidia-smi` の結果を見てから決める。
- **ハイブリッドGPU構成**: このPCはIntel + NVIDIAのハイブリッドGPU。CUDA計算はNVIDIA側が担当し正常に動く。
- **RViz 3D表示について**: noVNC（ブラウザアクセス）経由のRViz 3D表示はGPU描画不可。物理モニタかX11転送が別途必要。
- **GLIMのSLAM演算**: SLAM演算（本来の目的）はnoVNC環境でも問題なくGPUで動く。

---

## 参照

- [orange2025_docker リポジトリ](https://github.com/KBKN-Autonomous-Robotics-Lab/orange2025_docker)
