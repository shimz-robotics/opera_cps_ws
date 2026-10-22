# Usage

ホストから `scripts/exec.sh` 1 本で container 内のコマンドを叩く運用を基本とする。インタラクティブに探索したいときは `scripts/shell.sh` で対話 shell に入る。

## 動作確認: `task_id=1` 完走

### Terminal 1: Unity ↔ ROS 2 ブリッジ

```bash
./scripts/exec.sh ros2 launch ros_tcp_endpoint endpoint.py
```

`Starting server on 0.0.0.0:10000` が出たら待機状態。ここで Unity 側を再生し、`Connection from <Unity-IP>` のログを確認。

### Terminal 2: bringup（zx200 / tms_if / tms_ts を 3 段連鎖起動）

Terminal 1 起動 → Unity 再生 → 接続確認後に Terminal 2 を起動:

```bash
./scripts/exec.sh ros2 launch /workspace/src/ros2_tms_for_construction/docker/launch/bringup.launch.yaml
```

別の task を試す場合は `bringup.launch.yaml task_id:=2` のように渡す。

詳細手順（Unity 設定、RViz の初期姿勢回避、緑ボタン押下）は上流 README を参照: <https://github.com/irvs/ros2_tms_for_construction/tree/main/docker>

### 対話 shell で操作したい場合

```bash
./scripts/shell.sh
# プロンプトが ros@<container> になる、ROS env も source 済
# あとは普通に ros2 launch / ros2 node list 等を実行
```

## 停止

```bash
docker compose down         # コンテナ停止（named volume は保持）
docker compose down -v      # named volume ごと削除（DB・build キャッシュも消える）
```

`src.repos` / `Dockerfile` を更新したときは `down -v` してから rebuild する（`down` だけでは old artifact が再利用される）。詳細は [development.md](development.md) 参照。

## RMW 切替

デフォルトは Cyclone DDS (`rmw_cyclonedds_cpp`)。`rmw_fastrtps_cpp` / `rmw_cyclonedds_cpp` を image に同梱済。host 側に DDS を別途 install する必要はない (RMW プラグインは container 内 ROS 2 プロセスに linked-in)。

### Cyclone DDS (default)

```bash
docker compose up -d
```

これだけで OK。compose.yaml で `CYCLONEDDS_URI=/opt/opera-cps/cyclonedds.conf` を hardcode、repo 直下の [`cyclonedds.conf`](../cyclonedds.conf) (localhost-only multicast + `<DontRoute>true</DontRoute>`) を bind mount しているため、起動時から nodered と同一 profile で動く。env override は不要。

### Fast DDS に戻す

`.env` で:

```
RMW_IMPLEMENTATION=rmw_fastrtps_cpp
```

または一時的に:

```bash
RMW_IMPLEMENTATION=rmw_fastrtps_cpp docker compose up -d
```

### env 注入の確認

```bash
docker inspect opera_cps_dev \
  --format '{{range .Config.Env}}{{println .}}{{end}}' \
  | grep -E "RMW|CYCLONE|DOMAIN"
```

期待値:
- `RMW_IMPLEMENTATION=rmw_cyclonedds_cpp`
- `CYCLONEDDS_URI=/opt/opera-cps/cyclonedds.conf`
- `ROS_DOMAIN_ID=0`

### Zenoh (WAN / 複数現場集約)

`rmw_zenoh` は **container には install しない** 方針。WAN / NAT 越しや複数現場集約には、host で **[zenoh-bridge-ros2dds](https://github.com/eclipse-zenoh/zenoh-plugin-ros2dds)** を立てて Cyclone DDS の通信を Zenoh network に bridge する。container 側はそのまま Cyclone DDS で OK (`network_mode: host` で host の bridge と localhost multicast 経由でやり取り)。

参考: [shimz-robotics/nodered-ros2-opera](https://github.com/shimz-robotics/nodered-ros2-opera) でも同じ運用 (container = Cyclone DDS、host = `zenoh-bridge-ros2dds --config launch_content/config-nodered.json5`)。config ファイルは nodered repo の `launch_content/config-*.json5` (router / nodered / dump / pit / reference) を参考に role 別に作成。

host install (Eclipse Zenoh の deb repo):

```bash
echo "deb [trusted=yes] https://download.eclipse.org/zenoh/debian-repo/ /" \
  | sudo tee /etc/apt/sources.list.d/zenoh.list
sudo apt update && sudo apt install -y zenoh-bridge-ros2dds
```

起動例:

```bash
# host 側 (別 terminal)
zenoh-bridge-ros2dds --config /path/to/config.json5

# container 側はデフォルトで Cyclone DDS なのでそのまま up
docker compose up -d
```

詳細運用 (config 設計、複数現場 topology、認証) は別途 docs を整備する想定 (TODO)。

## Node-RED (nodered-ros2-opera) との並列運用

[`shimz-robotics/nodered-ros2-opera`](https://github.com/shimz-robotics/nodered-ros2-opera) は Cyclone DDS hardcoded (`rmw_cyclonedds_cpp` + `launch_content/cyclone_profile.xml` を `CYCLONEDDS_URI` で指す)。本リポも repo 直下の [`cyclonedds.conf`](../cyclonedds.conf) に同等内容を tracked で置いて bind mount するので、両者の profile は揃っており、同 host で立ち上げるだけで DDS discovery が両方向に通る。

### 起動

```bash
# TMS 側 (このリポ)
cd opera-cps-ws
docker compose up -d

# Node-RED 側 (別リポ)
cd ../nodered-ros2-opera
docker compose up -d
```

両 container 共に `network_mode: host` + `ROS_DOMAIN_ID=0` (デフォルト) で立ち上がる。nodered と TMS の `<DontRoute>true</DontRoute>` + `<NetworkInterface address="127.0.0.1" multicast="true"/>` profile が一致しているので localhost multicast 経由で peer-to-peer 通信する。

### discovery 確認

```bash
# TMS 側から
./scripts/exec.sh ros2 node list
./scripts/exec.sh ros2 topic list

# Node-RED 側から
docker exec -it shimz-node-red \
  bash -lc 'source /opt/ros/humble/setup.bash && ros2 node list'
```

両方から相手側の node が見えれば接続成功。

### つまずきポイント

- 相手の node が見えない → 両 container の `RMW_IMPLEMENTATION` と `ROS_DOMAIN_ID` を `docker inspect` で確認 (上述 env 注入の確認を両 container に対して実行)
- 片方向のみ → profile の `<DontRoute>true</DontRoute>` が片方欠けていないか、両 container `network_mode: host` で立っているか
- Node-RED 側 flow が topic を出さない → http://localhost:1880 admin で flow を deploy 必要

## 既知の制約 / トラブルシュート

- ROS 2 humble の Tier 1 サポートは Ubuntu 22.04 のみ
- GPU アクセラレーションは未設定
- moveit / nav2 / tf 系は wildcard を避け、`src.repos` の `<depend>` を grep した実依存ベースで explicit list 化（image を 12 GB → 5-6 GB を目標）。colcon build で「未 install」と言われたら Dockerfile のコメント済み apt-get install ブロックにそのパッケージを足す
- **ポート衝突**: `network_mode: host` のため `27017`（MongoDB）が他で使われていると起動失敗。`sudo systemctl stop mongod` で native mongod を停止、または `docker ps --filter "publish=27017"` で別 container を確認して `docker stop <name>` で空ける
- **`sequence size exceeds remaining buffer` 警告が大量に出る**: 共有マシン上で他人の ROS 2 publish が混入している。`ROS_DOMAIN_ID` を被らない番号に変更（`ROS_DOMAIN_ID=42 docker compose up -d`）
- **`xhost +local:` は閉じ忘れに注意**: 作業後 `xhost -local:`
- **xrdp 等で `$DISPLAY` が変わる環境**: `scripts/exec.sh` / `scripts/shell.sh` は呼び出し時の `$DISPLAY` を毎回 container に渡すので、ホスト側 shell の DISPLAY が正しければ OK。compose で記録された値を強制更新したい場合は `docker compose down && up`
