# GLIM Dump フォーマット解説 & 点群確認ツールまとめ

作成日: 2025-05-28  
対象データ: `~/ros2_ws/maps/dump_nakaniwa_0522/`

---

## 1. GLIMのDumpとは何か

GLIMはLiDAR SLAMライブラリで、走行中に構築したマップを**キーフレーム単位**でdumpディレクトリに保存する。  
`dump_on_unload: true` を設定しておくと、ノード終了時に自動保存される。

### ディレクトリ構造

```
dump_nakaniwa_0522/
├── 000000/          ← キーフレーム0
├── 000001/          ← キーフレーム1
├── 000021/          ← キーフレーム21（例）
│   ├── data.txt              ← ポーズ・メタ情報
│   ├── points_compact.bin    ← 点群データ（バイナリ）
│   ├── intensities_compact.bin ← 反射強度（バイナリ）
│   ├── covs_compact.bin      ← 共分散行列（バイナリ）
│   └── imu_rate.txt          ← IMUレートログ
└── 000118/
```

1フォルダ = 1キーフレーム。キーフレームは一定距離・角度移動したタイミングで生成される。

---

## 2. data.txt の構造

1つのキーフレームに**複数のLiDARスキャンフレーム**（`frame_0`〜`frame_N`）が格納されている。

```
id: 21                      ← キーフレームID
T_world_origin: [4x4行列]   ← world座標系の原点変換
T_origin_endpoint_L/R: ...  ← 左右エンドポイント変換
T_lidar_imu: [4x4行列]      ← LiDAR→IMU変換（固定）
imu_bias: ...
frame_id: 2
num_frames: 15              ← このキーフレームに含まれるスキャン数

frame_0
  id: 317
  stamp: 1779444767.449301  ← UNIX時刻 [秒]
  T_odom_lidar: [4x4行列]   ← odom座標系でのLiDARポーズ
  T_world_lidar: [4x4行列]  ← world座標系でのLiDARポーズ
  v_world_imu: [3次元速度]

frame_1
  ...（以下frame_14まで続く）
```

### 座標系の注意点

| フィールド | 意味 | 備考 |
|---|---|---|
| `T_odom_lidar` | odom座標系でのLiDARポーズ | GPSドリフトなし |
| `T_world_lidar` | world座標系でのLiDARポーズ | ループ閉合後に更新される |
| **今回のデータ** | T_odom == T_world | ループ閉合なし or odom=worldの設定 |

**重要**: ループ閉合が行われていないとき、`T_odom_lidar` と `T_world_lidar` は同じ値になる。

---

## 3. points_compact.bin の構造

### フォーマット

```
dtype  : float32（4 bytes/要素）
構造   : [x0, y0, z0, x1, y1, z1, ...]
reshape: np.fromfile(..., dtype=np.float32).reshape(-1, 3)
```

### 座標系：**LiDARローカル座標**

- 原点 = そのフレームのLiDARセンサ位置
- X/Y/Z軸 = LiDARセンサの向き
- **world座標ではない** → world座標に変換するには `T_world_lidar` を使う

```python
# world座標への変換
pts_local = np.fromfile("points_compact.bin", dtype=np.float32).reshape(-1, 3)
pts_h = np.hstack([pts_local, np.ones((len(pts_local), 1))])  # homogeneous
pts_world = (T_world_lidar @ pts_h.T).T[:, :3]
```

### 実データの統計（キーフレーム000021）

| 項目 | 値 |
|---|---|
| 点数 | 50,477点 |
| ファイルサイズ | 592 KB |
| X範囲 | -71.3 〜 +59.4 m |
| Y範囲 | -80.4 〜 +75.1 m |
| Z範囲 | -0.95 〜 +49.1 m |
| 上半球(Z>0)点数 | 32,439点 (64%) |
| 最大距離 | 約95 m |

### なぜ最大95mもあるのか

GLIMは`num_frames: 15`のように**複数スキャンを統合**して1キーフレームの点群を作る。  
ロボットが移動しながら取得したスキャンを積み上げるため、見かけ上の点群範囲が広くなる。  
Livox MID360の公称最大測距は約40mだが、統合後の点群は倍以上の範囲になる。

### 距離分布（キーフレーム000021）

```
dist < 10m :  8,897点 (17.6%)
dist < 20m : 20,626点 (40.9%)
dist < 30m : 35,222点 (69.8%)
dist < 40m : 45,153点 (89.5%)
dist < 50m : 49,644点 (98.3%)
```

**→ MAX_DIST=30mは上半球点の約40%を捨てることになる。50m程度が適切。**

---

## 4. 上半球遮蔽率計算における座標変換の誤り（重要）

### バグの内容

`occlusion_analysis.py` の `compute_occlusion_rate()` に座標変換の二重適用バグがある。

```python
# ❌ 間違い: points_world と命名しているが実際はlidar座標で渡している
def compute_occlusion_rate(points_world, T_odom_lidar, rays):
    lidar_pos = T_odom_lidar[:3, 3]
    dists_world = np.linalg.norm(points_world - lidar_pos, axis=1)  # ← world位置を引いている
    ...
    T_inv = np.linalg.inv(T_odom_lidar)
    pts_lidar = (T_inv @ pts_h.T).T[:, :3]  # ← さらに変換している（二重）
```

points_compact.bin はすでにLiDAR座標なので、変換は不要。

### 正しい実装

```python
# ✅ 正しい: lidar座標をそのまま使う
def compute_occlusion_rate(points_lidar, rays):
    dists = np.linalg.norm(points_lidar, axis=1)
    mask = (points_lidar[:, 2] > 0) & (dists < MAX_DIST)
    pts_upper = points_lidar[mask]

    if len(pts_upper) == 0:
        return 0.0

    d = np.linalg.norm(pts_upper, axis=1)
    dirs = pts_upper / d[:, np.newaxis]

    occluded = sum(1 for ray in rays if np.any((dirs @ ray > 0.9962) & (d < MAX_DIST)))
    return occluded / len(rays)
```

---

## 5. 点群確認ツール

### ツール1: 座標系の判定スクリプト

**目的**: 点群がworld座標かlidar座標かを判定する

```python
import numpy as np, re

FRAME_DIR = "/home/ubuntu/ros2_ws/maps/dump_nakaniwa_0522/000021"

pts = np.fromfile(f"{FRAME_DIR}/points_compact.bin", dtype=np.float32).reshape(-1, 3)
print(f"点数: {len(pts)}")
print(f"X: min={pts[:,0].min():.2f}, max={pts[:,0].max():.2f}, mean={pts[:,0].mean():.2f}")
print(f"Y: min={pts[:,1].min():.2f}, max={pts[:,1].max():.2f}, mean={pts[:,1].mean():.2f}")
print(f"Z: min={pts[:,2].min():.2f}, max={pts[:,2].max():.2f}, mean={pts[:,2].mean():.2f}")

with open(f"{FRAME_DIR}/data.txt") as f:
    content = f.read()
mat = re.search(
    r'frame_0.*?T_world_lidar:\s*\n'
    r'\s*([\d.e+\-]+)\s+([\d.e+\-]+)\s+([\d.e+\-]+)\s+([\d.e+\-]+)\s*\n'
    r'\s*([\d.e+\-]+)\s+([\d.e+\-]+)\s+([\d.e+\-]+)\s+([\d.e+\-]+)\s*\n'
    r'\s*([\d.e+\-]+)\s+([\d.e+\-]+)\s+([\d.e+\-]+)\s+([\d.e+\-]+)\s*\n'
    r'\s*([\d.e+\-]+)\s+([\d.e+\-]+)\s+([\d.e+\-]+)\s+([\d.e+\-]+)',
    content, re.DOTALL
)
T = np.array([float(v) for v in mat.groups()]).reshape(4,4)
lidar_pos = T[:3, 3]
print(f"\nLiDAR world位置: {lidar_pos}")
print(f"点群centroid:    {pts.mean(axis=0)}")
print(f"差（小さければlidar座標、大きければworld座標）: {pts.mean(axis=0) - lidar_pos}")
```

**判定方法**:
- centroidが原点付近 (0〜数m) → **lidar座標**
- centroidがLiDAR world位置に近い → world座標

---

### ツール2: RViz2で全点群を可視化

**目的**: フレームの点群形状をRViz2でリアルタイム確認

```python
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import PointCloud2, PointField
from std_msgs.msg import Header
import numpy as np, time

FRAME_DIR = "/home/ubuntu/ros2_ws/maps/dump_nakaniwa_0522/000021"

pts = np.fromfile(f"{FRAME_DIR}/points_compact.bin", dtype=np.float32).reshape(-1, 3)

rclpy.init()
node = Node("pcd_pub")
pub = node.create_publisher(PointCloud2, "/debug_points", 10)

msg = PointCloud2()
msg.header = Header()
msg.header.frame_id = "map"   # ← Fixed Frameと一致させる
msg.height = 1
msg.width = len(pts)
msg.fields = [
    PointField(name='x', offset=0,  datatype=PointField.FLOAT32, count=1),
    PointField(name='y', offset=4,  datatype=PointField.FLOAT32, count=1),
    PointField(name='z', offset=8,  datatype=PointField.FLOAT32, count=1),
]
msg.is_bigendian = False
msg.point_step = 12
msg.row_step = 12 * len(pts)
msg.is_dense = True
msg.data = pts.astype(np.float32).tobytes()

print(f"点数: {len(pts)}, publish中... (Ctrl+Cで終了)")
while rclpy.ok():
    msg.header.stamp = node.get_clock().now().to_msg()
    pub.publish(msg)
    time.sleep(1.0)
```

**RViz2設定**:
1. `rviz2` を別ターミナルで起動
2. Fixed Frame → `map`
3. Add → PointCloud2 → Topic: `/debug_points`

---

### ツール3: 上半球フィルタ確認 + RViz2可視化

**目的**: Z>0かつdist<閾値でフィルタ後の点群を確認

```python
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import PointCloud2, PointField
from std_msgs.msg import Header
import numpy as np, time

FRAME_DIR = "/home/ubuntu/ros2_ws/maps/dump_nakaniwa_0522/000021"
MAX_DIST = 50.0   # ← ここを変えて試す

pts = np.fromfile(f"{FRAME_DIR}/points_compact.bin", dtype=np.float32).reshape(-1, 3)
dists = np.linalg.norm(pts, axis=1)
mask = (pts[:, 2] > 0) & (dists < MAX_DIST)
pts_upper = pts[mask]

print(f"全点数:   {len(pts)}")
print(f"上半球内: {len(pts_upper)} ({100*len(pts_upper)/len(pts):.1f}%)")
print(f"Z>0のみ:  {(pts[:,2]>0).sum()}")
print(f"dist<{MAX_DIST}m: {(dists<MAX_DIST).sum()}")

rclpy.init()
node = Node("pcd_upper_pub")
pub = node.create_publisher(PointCloud2, "/debug_upper", 10)

msg = PointCloud2()
msg.header = Header()
msg.header.frame_id = "map"
msg.height = 1
msg.width = len(pts_upper)
msg.fields = [
    PointField(name='x', offset=0, datatype=PointField.FLOAT32, count=1),
    PointField(name='y', offset=4, datatype=PointField.FLOAT32, count=1),
    PointField(name='z', offset=8, datatype=PointField.FLOAT32, count=1),
]
msg.is_bigendian = False
msg.point_step = 12
msg.row_step = 12 * len(pts_upper)
msg.is_dense = True
msg.data = pts_upper.astype(np.float32).tobytes()

print(f"publish中... (Ctrl+Cで終了)")
while rclpy.ok():
    msg.header.stamp = node.get_clock().now().to_msg()
    pub.publish(msg)
    time.sleep(1.0)
```

**RViz2設定**: topic `/debug_upper` を追加。`/debug_points` と同時表示して比較できる。

---

### ツール4: 距離分布確認スクリプト

**目的**: MAX_DISTの適切な値を決めるための統計確認

```python
import numpy as np

FRAME_DIR = "/home/ubuntu/ros2_ws/maps/dump_nakaniwa_0522/000021"

pts = np.fromfile(f"{FRAME_DIR}/points_compact.bin", dtype=np.float32).reshape(-1, 3)
dists = np.linalg.norm(pts, axis=1)

print(f"全点数: {len(pts)}")
print(f"距離: min={dists.min():.1f}m, max={dists.max():.1f}m, mean={dists.mean():.1f}m")

# 上半球かつ遠距離の点
mask_far_upper = (pts[:, 2] > 0) & (dists >= 30.0)
pts_far = pts[mask_far_upper]
print(f"\n上半球かつ30m以上: {len(pts_far)}点")
print(f"  Z平均: {pts_far[:,2].mean():.2f}m  ← 建物上部などが含まれている")

# 閾値別の点数
print("\n閾値別・累積点数:")
for d in [10, 20, 30, 40, 50, 60, 80, 100]:
    n = (dists < d).sum()
    print(f"  dist < {d:3d}m: {n:6d}点 ({100*n/len(pts):.1f}%)")
```

---

## 6. 新データ取得時のチェックリスト

新しいdumpデータを使う前に以下を確認する。

```
□ data.txt の num_frames を確認（何スキャン統合か）
□ T_odom_lidar と T_world_lidar が同じか異なるか
  → 異なる場合はループ閉合あり → T_world_lidar を使うべき
□ points_compact.bin の点数・距離範囲を確認（ツール4）
□ centroid確認でlidar座標かworld座標かを判定（ツール1）
□ RViz2で点群形状が正しく見えるか確認（ツール2）
□ MAX_DIST の値が距離分布に対して適切か確認（ツール4）
  → 全上半球点の90%以上をカバーできる値を選ぶ
```

---

## 7. よくある問題と対処

| 問題 | 原因 | 対処 |
|---|---|---|
| 遮蔽率が異常に低い/高い | 座標変換の二重適用 | points_compact.binはlidar座標なのでT_inv変換不要 |
| 遮蔽率が全フレームで低い | MAX_DISTが小さすぎ | 距離分布を確認して50m程度に変更 |
| parse_data_txtがNoneを返す | stampのフォーマット違い | frame_0のstampを正規表現で取得しているか確認 |
| RViz2で点群が見えない | Fixed Frameが合っていない | `map`に設定する |
| T_odom==T_world | ループ閉合なし or GLIM設定 | 研究用途では問題なし（odom精度で評価） |
