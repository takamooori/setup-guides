# ROS2 bag トピック名変更手順

## 概要

ROS2のbagファイル（`.db3`）はSQLiteデータベースであるため、
`sqlite3`コマンドで直接トピック名を書き換えることができる。

---

## 前提条件

```bash
# sqlite3のインストール（未インストールの場合）
sudo apt install sqlite3 -y
```

---

## 手順

### 1. bagディレクトリに移動

```bash
cd ~/ros2_ws/bag/<bag名>/
ls
# metadata.yaml と *.db3 が存在することを確認
```

---

### 2. バックアップを作成

```bash
cp <ファイル名>.db3 <ファイル名>.db3.bak
```

> **確認方法：** `ls` で `.bak` ファイルが作成されていればOK  
> **元に戻す方法：** `cp <ファイル名>.db3.bak <ファイル名>.db3`

---

### 3. db3のトピック名を書き換え

```bash
sqlite3 <ファイル名>.db3 "UPDATE topics SET name = '/新しい名前' WHERE name = '/古い名前';"
```

**例：**
```bash
sqlite3 rosbag2_2026_05_21-08_42_17_0.db3 "UPDATE topics SET name = '/imu/spresense' WHERE name = '/imu_corrected';"
```

---

### 4. metadata.yamlも書き換え

```bash
sed -i 's|/古い名前|/新しい名前|g' metadata.yaml
```

**例：**
```bash
sed -i 's|/imu_corrected|/imu/spresense|g' metadata.yaml
```

---

### 5. 確認

```bash
ros2 bag info .
```

`Topic information` に新しいトピック名が表示されていればOK。

---

## トピックが重複していた場合

変更先のトピック名が既にbagに存在していた場合、同じ名前が2つ表示される。
古い方（不要な方）を削除する。

### 5-1. どちらが古いか確認（idが小さい方が古い）

```bash
sqlite3 <ファイル名>.db3 "SELECT id, name FROM topics WHERE name = '/重複したトピック名';"
```

出力例：
```
1|/imu/spresense   ← 古い（元からあった）
2|/imu/spresense   ← 新しい（今回変更した）
```

---

### 5-2. 古い方のメッセージとトピックをdb3から削除

> ⚠️ `topic_id = <古いid>` の数字を確認してから実行すること

```bash
sqlite3 <ファイル名>.db3 "DELETE FROM messages WHERE topic_id = 1; DELETE FROM topics WHERE id = 1;"
```

---

### 5-3. metadata.yamlから古いエントリを削除

`ros2 bag info` で古い方のメッセージ数を確認してから実行。

```bash
python3 -c "
import yaml

with open('metadata.yaml', 'r') as f:
    data = yaml.safe_load(f)

topics = data['rosbag2_bagfile_information']['topics_with_message_count']

# 削除対象：トピック名と古いmessage_countで特定
remove_count = <古い方のmessage_count>  # ← ここを変更
topics = [t for t in topics if not (t['topic_metadata']['name'] == '/重複したトピック名' and t['message_count'] == remove_count)]

data['rosbag2_bagfile_information']['topics_with_message_count'] = topics
data['rosbag2_bagfile_information']['message_count'] -= remove_count

with open('metadata.yaml', 'w') as f:
    yaml.dump(data, f, allow_unicode=True, sort_keys=False)

print('Done')
"
```

---

### 5-4. 再確認

```bash
ros2 bag info .
```

重複がなくなっていればOK。

---

## 後片付け

動作確認が取れたらバックアップを削除。

```bash
rm <ファイル名>.db3.bak
```

---

## 注意事項

- 複数のbagディレクトリがある場合、**それぞれのディレクトリで独立して作業**する（互いに影響しない）
- `sqlite3`コマンドでは**正しいdb3ファイル名**を指定すること（ディレクトリ内に複数ある場合は特に注意）
- `metadata.yaml` と `.db3` は**両方とも変更**が必要（どちらか一方だけでは不完全）
