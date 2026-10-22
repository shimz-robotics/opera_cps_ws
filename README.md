# opera-cps-ws

ROS 2 TMS for Construction の **開発用 meta-workspace**。Docker + vcstool で `ros2_tms_for_construction` 本体と全外部依存を 1 ファイルで管理し、ホスト側で全 src を編集しながらコンテナ内で colcon build できる構成。

- **ベース image**: `osrf/ros:humble-desktop-full` (Ubuntu 22.04 jammy)
- **GUI**: X11 forward（VNC なし）
- **MongoDB**: 別サービス (`mongo:6.0`)
- **外部依存**: `src.repos` (public) / `src.private.repos` (private) で commit を固定して vcstool 管理

## 上流 PR との関係

「最小限の Docker テンプレート」として `irvs/ros2_tms_for_construction` の `docker/` 配下にも別の Dockerfile セットが存在する:

| 役割 | 場所 |
|---|---|
| 誰でもまず動かせる reference | `irvs/ros2_tms_for_construction:docker/` |
| 全依存を src.repos で管理する開発・実験環境 | このリポジトリ |

両方を並存させる。本リポジトリの `Dockerfile` / `entrypoint.sh` / `restore-db.sh` / `launch/bringup.launch.yaml` は上流の `docker/` をベースにしているため、上流が更新されたら追従が必要。

## 環境構築

```bash
git clone https://github.com/shimz-robotics/opera-cps-ws.git
cd opera-cps-ws

# per-host 設定 (UID/GID 等) を .env に固定 — 初回のみ
cp .env.example .env
sed -i "s/^UID=.*/UID=$(id -u)/; s/^GID=.*/GID=$(id -g)/" .env

mkdir -p src
vcs import src/ < src.repos
vcs import src/ < src.private.repos || true

xhost +local:
docker compose build                 # 初回 20-30 分 (.env から UID/GID を自動読込)
docker compose up -d                 # 初回起動時にコンテナ内 colcon build (10 分)
docker compose exec tms restore-db.sh # DB seed 投入 (初回のみ)
```

前提条件（ホスト OS / vcstool / X サーバ 等）と各ステップの解説は [docs/setup.md](docs/setup.md) 参照。`.env` は host 固有なので gitignored、`.env.example` を雛形に編集する。

## 動作確認 (task_id=1)

Unity ([shimz-robotics/OperaSim-PhysX](https://github.com/shimz-robotics/OperaSim-PhysX)) を別途起動した状態で、以下を 2 ターミナルで実行:

### Terminal 1: Unity ↔ ROS 2 ブリッジ

```bash
./scripts/exec.sh ros2 launch ros_tcp_endpoint endpoint.py
```

Unity の **再生ボタンを押して接続を確認した後**、Terminal 2 を起動する（順序を守らないと `zx200_bringup` の `ros2_control` が初期姿勢を取得できず不安定になる）。

### Terminal 2: bringup (zx200 / tms_if / tms_ts を 3 段連鎖起動)

```bash
./scripts/exec.sh ros2 launch /workspace/src/ros2_tms_for_construction/docker/launch/bringup.launch.yaml
```

RViz の初期姿勢回避（`boom_joint` を下げて Plan & Execute）→ `tms_ur_button` の緑ボタン押下までの詳細手順は [docs/usage.md](docs/usage.md) 参照。

### 終了

```bash
docker compose down     # コンテナ停止
```

## RMW 切替

デフォルトは Cyclone DDS (`rmw_cyclonedds_cpp`)。compose.yaml で `CYCLONEDDS_URI` を repo 直下の [`cyclonedds.conf`](cyclonedds.conf) (localhost-only multicast + DontRoute) に固定済なので、Node-RED ([shimz-robotics/nodered-ros2-opera](https://github.com/shimz-robotics/nodered-ros2-opera)) の `launch_content/cyclone_profile.xml` と同一 profile になり、同 host 内 container 間で discovery が両方向に通る。Fast DDS が必要なら `.env` で `RMW_IMPLEMENTATION=rmw_fastrtps_cpp` に上書き。

### LAN 上の別 PC と直接 DDS 接続したい場合

`cyclonedds.conf` は **同 host 並列運用に最適化** された localhost-only profile のため、別 PC 上の ROS 2 ノードとは直接 discover しない。次のいずれかを選択:

1. **host で [`zenoh-bridge-ros2dds`](https://github.com/eclipse-zenoh/zenoh-plugin-ros2dds) を立てる** (推奨、WAN / NAT / 複数現場でもそのまま使える、詳細 [docs/usage.md](docs/usage.md))
2. **`compose.override.yaml`** で LAN 向け interface 指定の profile (`<NetworkInterface address="<host-IP>"/>` 等) を bind mount し直す
3. **Fast DDS に切り替え** (`.env` で `RMW_IMPLEMENTATION=rmw_fastrtps_cpp`)、LAN 内自動 discovery を利用

詳細 (Node-RED 並列運用手順、host zenoh-bridge-ros2dds 起動手順) は [docs/usage.md](docs/usage.md) 参照。

## ドキュメント

- [docs/setup.md](docs/setup.md) — 前提・ワークスペース構造・起動手順（詳細）
- [docs/usage.md](docs/usage.md) — `task_id=1` 完走手順 / 停止 / 既知の制約・トラブルシュート
- [docs/development.md](docs/development.md) — src 編集 → 再ビルドの運用（Python / C++ / pin bump / 初回 build やり直し / Dockerfile 変更）

## 関連

- 上流: [irvs/ros2_tms_for_construction](https://github.com/irvs/ros2_tms_for_construction)
- Unity 側シミュレータ: [shimz-robotics/OperaSim-PhysX](https://github.com/shimz-robotics/OperaSim-PhysX) — zx200 link の DeadTime tuning で制御の振動挙動を緩和済み（`shimz-main` branch）。
