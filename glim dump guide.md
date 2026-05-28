#!/usr/bin/env python3
"""
GLIM Inspector - GLIMダンプデータの確認・可視化ツール

使い方:
    python3 glim_inspector.py
    python3 glim_inspector.py --dump ~/ros2_ws/maps/dump_nakaniwa_0522
"""

# ============================================================
# 設定（ここだけ変える）
# ============================================================
DEFAULT_DUMP = "~/ros2_ws/maps/dump_nakaniwa_0522"
MAX_DIST     = 50.0   # 上半球フィルタの距離閾値 [m]
# ============================================================

import argparse
import os
import re
import sys
import time
import threading

import numpy as np


# ============================================================
# ユーティリティ
# ============================================================

def get_folders(dump_dir):
    return sorted([
        f for f in os.listdir(dump_dir)
        if re.match(r'^\d{6}$', f) and os.path.isdir(os.path.join(dump_dir, f))
    ])


def parse_frame0_pose(data_txt):
    with open(data_txt) as f:
        content = f.read()
    mat44 = (
        r'\s*([\d.e+\-]+)\s+([\d.e+\-]+)\s+([\d.e+\-]+)\s+([\d.e+\-]+)\s*\n'
        r'\s*([\d.e+\-]+)\s+([\d.e+\-]+)\s+([\d.e+\-]+)\s+([\d.e+\-]+)\s*\n'
        r'\s*([\d.e+\-]+)\s+([\d.e+\-]+)\s+([\d.e+\-]+)\s+([\d.e+\-]+)\s*\n'
        r'\s*([\d.e+\-]+)\s+([\d.e+\-]+)\s+([\d.e+\-]+)\s+([\d.e+\-]+)'
    )
    m = re.search(r'frame_0.*?T_world_lidar:\s*\n' + mat44, content, re.DOTALL)
    if not m:
        return None
    return np.array([float(v) for v in m.groups()]).reshape(4, 4)


def load_points(dump_dir, folder):
    return np.fromfile(
        os.path.join(dump_dir, folder, "points_compact.bin"),
        dtype=np.float32
    ).reshape(-1, 3)


def make_cloud_msg(pts, frame_id="map"):
    from sensor_msgs.msg import PointCloud2, PointField
    from std_msgs.msg import Header
    msg = PointCloud2()
    msg.header = Header()
    msg.header.frame_id = frame_id
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
    return msg


def generate_rviz_config(dump_dir, topics):
    type_map  = {
        "PointCloud2": "rviz_default_plugins/PointCloud2",
        "Path":        "rviz_default_plugins/Path",
    }
    color_map = {
        "/glim_map":     "r: 1.0\n          g: 1.0\n          b: 1.0",
        "/debug_points": "r: 0.0\n          g: 1.0\n          b: 0.5",
        "/debug_upper":  "r: 1.0\n          g: 0.8\n          b: 0.0",
        "/glim_path":    "r: 1.0\n          g: 0.2\n          b: 0.2",
    }
    displays = ""
    for dtype, topic in topics:
        plugin = type_map.get(dtype, f"rviz_default_plugins/{dtype}")
        col    = color_map.get(topic, "r: 1.0\n          g: 1.0\n          b: 1.0")
        extra  = "\n        Color Transformer: AxisColor\n        Axis: Z\n        Size (m): 0.05" \
                 if dtype == "PointCloud2" else ""
        displays += f"""
    - Class: {plugin}
      Enabled: true
      Name: {topic}
      Topic:
        Value: {topic}
      Color:
        {col}{extra}
"""
    dump_name = os.path.basename(dump_dir.rstrip('/'))
    rviz_path = os.path.join(dump_dir, f"{dump_name}.rviz")
    with open(rviz_path, 'w') as f:
        f.write(f"""Visualization Manager:
  Class: ""
  Displays:{displays}
  Enabled: true
  Global Options:
    Fixed Frame: map
  Name: root
  Tools:
    - Class: rviz_default_plugins/MoveCamera
Window Geometry:
  Height: 900
  Width: 1400
""")
    print(f"RViz設定: {rviz_path}")
    print(f"起動:     rviz2 -d {rviz_path}\n")


# ============================================================
# フレーム切り替えpublishループ
# ============================================================

def interactive_publish(dump_dir, folders, start_idx, build_msgs_fn):
    """
    build_msgs_fn(node, idx) -> [(publisher, cloud_msg), ...]
    publishしながら [n]/[p]/数字 でフレームを切り替える
    """
    import rclpy

    rclpy.init()
    node = rclpy.create_node("glim_interactive_pub")

    state = {
        "idx":  start_idx,
        "msgs": build_msgs_fn(node, start_idx),
        "stop": False,
    }

    def publish_worker():
        while not state["stop"] and rclpy.ok():
            now = node.get_clock().now().to_msg()
            for pub, msg in state["msgs"]:
                msg.header.stamp = now
                pub.publish(msg)
            time.sleep(0.5)

    t = threading.Thread(target=publish_worker, daemon=True)
    t.start()

    print("publish開始 — rviz2 を別ターミナルで起動してください")
    while True:
        folder = folders[state["idx"]]
        print(f"\n現在: [{state['idx']:3d}] {folder}   "
              f"[n]次  [p]前  [数字]直接指定  [q]終了")
        cmd = input("> ").strip().lower()

        if cmd == 'q':
            state["stop"] = True
            break
        elif cmd == 'n':
            new_idx = min(state["idx"] + 1, len(folders) - 1)
        elif cmd == 'p':
            new_idx = max(state["idx"] - 1, 0)
        elif cmd.isdigit() and 0 <= int(cmd) < len(folders):
            new_idx = int(cmd)
        else:
            print("無効な入力")
            continue

        state["idx"]  = new_idx
        state["msgs"] = build_msgs_fn(node, new_idx)
        print(f"→ フレーム {folders[new_idx]} に切り替え")

    rclpy.shutdown()


# ============================================================
# 各メニュー機能
# ============================================================

def menu_summary(dump_dir, folders):
    print(f"\n--- データ概要: {os.path.basename(dump_dir)} ---")
    positions, total_pts = [], 0
    for folder in folders:
        T = parse_frame0_pose(os.path.join(dump_dir, folder, "data.txt"))
        if T is not None:
            positions.append(T[:3, 3])
        pts_bin = os.path.join(dump_dir, folder, "points_compact.bin")
        if os.path.exists(pts_bin):
            total_pts += os.path.getsize(pts_bin) // 12
    positions = np.array(positions)
    travel = np.sum(np.linalg.norm(np.diff(positions, axis=0), axis=1))
    print(f"総キーフレーム数: {len(folders)}")
    print(f"取得ポーズ数:     {len(positions)}")
    print(f"総移動距離:       {travel:.1f} m")
    print(f"X範囲: {positions[:,0].min():.1f} 〜 {positions[:,0].max():.1f} m")
    print(f"Y範囲: {positions[:,1].min():.1f} 〜 {positions[:,1].max():.1f} m")
    print(f"Z範囲: {positions[:,2].min():.1f} 〜 {positions[:,2].max():.1f} m")
    print(f"総点数（概算）:   {total_pts:,} 点")


def menu_coord_check(dump_dir, folders, frame_idx):
    folder    = folders[frame_idx]
    pts       = load_points(dump_dir, folder)
    T         = parse_frame0_pose(os.path.join(dump_dir, folder, "data.txt"))
    lidar_pos = T[:3, 3]
    centroid  = pts.mean(axis=0)
    diff      = np.linalg.norm(centroid - lidar_pos)
    print(f"\n--- 座標系判定: {folder} ---")
    print(f"点数:            {len(pts)}")
    print(f"点群centroid:    {centroid.round(2)}")
    print(f"LiDAR world位置: {lidar_pos.round(2)}")
    print(f"差（ノルム）:    {diff:.2f} m")
    print("→ LiDARローカル座標（正常）" if diff < 5.0 else "→ world座標の可能性あり（要確認）")


def menu_dist_stats(dump_dir, folders, frame_idx):
    folder = folders[frame_idx]
    pts    = load_points(dump_dir, folder)
    dists  = np.linalg.norm(pts, axis=1)
    upper  = (pts[:,2] > 0).sum()
    print(f"\n--- 距離分布: {folder} ---")
    print(f"全点数: {len(pts)}  max={dists.max():.1f}m  mean={dists.mean():.1f}m\n")
    for d in [10, 20, 30, 40, 50, 60, 80, 100]:
        n   = (dists < d).sum()
        bar = '█' * int(30 * n / len(pts))
        print(f"  dist < {d:3d}m: {n:6d}点 ({100*n/len(pts):5.1f}%) {bar}")
    print(f"\n上半球(Z>0)={upper}点 に対するMAX_DIST別カバー率:")
    for d in [30, 40, 50, 60]:
        n    = ((pts[:,2] > 0) & (dists < d)).sum()
        mark = " ← 現在の設定" if d == MAX_DIST else ""
        print(f"  MAX_DIST={d}m: {100*n/upper:.1f}%{mark}")


def menu_frame_rviz(dump_dir, folders, start_idx):
    """[3] 点群 → RViz（全点 + 上半球、フレーム切り替えあり）"""
    from sensor_msgs.msg import PointCloud2

    print(f"MAX_DIST = {MAX_DIST} m（変更はスクリプト冒頭の設定を編集）")
    generate_rviz_config(dump_dir, [
        ("PointCloud2", "/debug_points"),
        ("PointCloud2", "/debug_upper"),
    ])
    print("RViz2 でトピックのチェックを切り替えて表示を選択してください")
    print(f"  /debug_points : 全点群")
    print(f"  /debug_upper  : 上半球(Z>0, dist<{MAX_DIST}m)のみ\n")

    pubs = {}

    def build_msgs(node, idx):
        if "all" not in pubs:
            pubs["all"]   = node.create_publisher(PointCloud2, "/debug_points", 10)
            pubs["upper"] = node.create_publisher(PointCloud2, "/debug_upper",  10)
        pts   = load_points(dump_dir, folders[idx])
        dists = np.linalg.norm(pts, axis=1)
        upper = pts[(pts[:,2] > 0) & (dists < MAX_DIST)]
        print(f"  全{len(pts):,}点 / 上半球{len(upper):,}点 ({100*len(upper)/len(pts):.1f}%)")
        return [
            (pubs["all"],   make_cloud_msg(pts)),
            (pubs["upper"], make_cloud_msg(upper)),
        ]

    interactive_publish(dump_dir, folders, start_idx, build_msgs)


def menu_full_map_rviz(dump_dir, folders):
    """[4] 全軌跡 + マップ → RViz"""
    import rclpy
    from nav_msgs.msg import Path
    from geometry_msgs.msg import PoseStamped
    from sensor_msgs.msg import PointCloud2
    from std_msgs.msg import Header

    stride_kf  = int(input("キーフレーム間引き (デフォルト3): ").strip() or "3")
    stride_pts = int(input("点群間引き        (デフォルト10): ").strip() or "10")

    print("データ読み込み中...")
    all_poses, all_pts_world = [], []

    for folder in folders:
        T = parse_frame0_pose(os.path.join(dump_dir, folder, "data.txt"))
        if T is not None:
            all_poses.append((folder, T))

    for i, (folder, T) in enumerate(all_poses):
        if i % stride_kf != 0:
            continue
        pts_bin = os.path.join(dump_dir, folder, "points_compact.bin")
        if not os.path.exists(pts_bin):
            continue
        pts   = np.fromfile(pts_bin, dtype=np.float32).reshape(-1, 3)[::stride_pts]
        pts_h = np.hstack([pts, np.ones((len(pts), 1))])
        all_pts_world.append((T @ pts_h.T).T[:, :3].astype(np.float32))

    all_pts_world = np.vstack(all_pts_world)
    print(f"表示点数: {len(all_pts_world):,}")

    generate_rviz_config(dump_dir, [
        ("PointCloud2", "/glim_map"),
        ("Path",        "/glim_path"),
    ])

    path_msg = Path()
    path_msg.header = Header()
    path_msg.header.frame_id = "map"
    for _, T in all_poses:
        ps = PoseStamped()
        ps.header.frame_id = "map"
        ps.pose.position.x = float(T[0, 3])
        ps.pose.position.y = float(T[1, 3])
        ps.pose.position.z = float(T[2, 3])
        ps.pose.orientation.w = 1.0
        path_msg.poses.append(ps)

    cloud_msg = make_cloud_msg(all_pts_world)

    rclpy.init()
    node     = rclpy.create_node("glim_map_pub")
    pub_map  = node.create_publisher(PointCloud2, "/glim_map",  10)
    pub_path = node.create_publisher(Path,        "/glim_path", 10)

    print("publish中... (Ctrl+C で終了)")
    while rclpy.ok():
        now = node.get_clock().now().to_msg()
        cloud_msg.header.stamp = now
        path_msg.header.stamp  = now
        pub_map.publish(cloud_msg)
        pub_path.publish(path_msg)
        time.sleep(1.0)


# ============================================================
# フレーム選択
# ============================================================

def select_frame(folders):
    print(f"\nフレーム選択 (総数: {len(folders)}):")
    print(f"  [a] 最初  ({folders[0]})")
    print(f"  [m] 中間  ({folders[len(folders)//2]})")
    print(f"  [e] 最後  ({folders[-1]})")
    print(f"  数字で直接指定 (0 〜 {len(folders)-1})")
    choice = input("選択 > ").strip().lower()
    if choice == 'a':
        return 0
    elif choice == 'm':
        return len(folders) // 2
    elif choice == 'e':
        return len(folders) - 1
    elif choice.isdigit() and 0 <= int(choice) < len(folders):
        return int(choice)
    else:
        print("無効な入力。最初のフレームを使用")
        return 0


# ============================================================
# メインメニュー
# ============================================================

def main():
    parser = argparse.ArgumentParser(description="GLIM Dump Inspector")
    parser.add_argument('--dump', default=os.path.expanduser(DEFAULT_DUMP))
    args = parser.parse_args()

    dump_dir = os.path.expanduser(args.dump)
    if not os.path.isdir(dump_dir):
        print(f"エラー: {dump_dir} が見つかりません")
        sys.exit(1)

    folders = get_folders(dump_dir)
    if not folders:
        print("キーフレームが見つかりません")
        sys.exit(1)

    while True:
        print(f"""
=== GLIM Inspector ===
DUMP    : {dump_dir}
フレーム: {len(folders)} 個 ({folders[0]} 〜 {folders[-1]})
MAX_DIST: {MAX_DIST} m  （変更はスクリプト冒頭）

  [1] データ概要確認
  [2] 座標系判定
  [3] 点群 → RViz  (/debug_points + /debug_upper)  ← フレーム切り替えあり
  [4] 全軌跡 + マップ → RViz (/glim_path + /glim_map)
  [5] 距離分布確認
  [q] 終了
""")
        choice = input("選択 > ").strip().lower()

        if choice == 'q':
            print("終了")
            break
        elif choice == '1':
            menu_summary(dump_dir, folders)
        elif choice == '2':
            menu_coord_check(dump_dir, folders, select_frame(folders))
        elif choice == '3':
            menu_frame_rviz(dump_dir, folders, select_frame(folders))
        elif choice == '4':
            menu_full_map_rviz(dump_dir, folders)
        elif choice == '5':
            menu_dist_stats(dump_dir, folders, select_frame(folders))
        else:
            print("無効な入力です")


if __name__ == '__main__':
    main()
