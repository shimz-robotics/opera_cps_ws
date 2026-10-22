# Setup

## 前提

- Ubuntu ホスト（ROS 2 humble の Tier 1 プラットフォーム要件）
- Docker Engine + docker-compose v2
- Python 3 + `vcstool` (`sudo apt install python3-vcstool`)
- X サーバが稼働中（GUI を使う場合）
- `shimz-robotics` メンバー / 各 collaborator 権限（`src.private.repos` の取り込みに必要、無くても public 部分だけで build は通る）

## ワークスペース構造

```
opera-cps-ws/                        # this repo (meta)
├── Dockerfile
├── compose.yaml
├── requirements.txt                 # 上流から vendor、bump 時は手動同期
├── src.repos                        # public 依存
├── src.private.repos                # private 依存
├── launch/bringup.launch.yaml       # task_id=1 用 3 launch 連鎖
├── scripts/
│   ├── entrypoint.sh                # container: 初回 colcon build + gosu drop
│   ├── restore-db.sh                # container: mongorestore + parameter フィールド整形
│   ├── exec.sh                      # host: docker compose exec で ROS env 付き 1 コマンド実行
│   └── shell.sh                     # host: container に対話 bash で入る
├── docs/                            # 本ドキュメント群
└── src/                             # vcs import 先 (.gitignore'd)
    ├── ros2_tms_for_construction/
    ├── Groot/
    ├── tms_if_for_opera/
    ├── opera/{common,simulator,zx200,ic120}/...
    └── ...
```

コンテナ内では `./src/` がそのまま `/workspace/src/` に bind mount される。`build/` `install/` `log/` は named volume。

## 起動手順

### 1. clone & 依存取得

```bash
git clone https://github.com/shimz-robotics/opera-cps-ws.git
cd opera-cps-ws

mkdir -p src
vcs import src/ < src.repos                       # 全員必須 (public)
vcs import src/ < src.private.repos || true       # アクセス権ある人のみ成功
```

`src.private.repos` の `|| true` は意図的: vcstool には skip フラグが無いため、認証に失敗した entry を許容するための wrap。アクセス権が無い entry はスキップ扱いになり、ある entry のみ clone される。

### 2. ビルド

```bash
UID=$(id -u) GID=$(id -g) docker compose build
```

`UID` / `GID` を build args で渡すのは、コンテナ内 `ros` ユーザーの UID/GID をホストと揃えて bind mount の権限不一致を防ぐため。初回は BehaviorTree.CPP / mongo-c-driver / mongo-cxx-driver などのソースビルドが走るため 20〜30 分かかる想定。

### 3. X サーバーへのアクセス許可（GUI を使うセッションごとに 1 回）

```bash
xhost +local:
```

### 4. 起動

```bash
docker compose up -d
```

初回起動時にコンテナ内で `colcon build --symlink-install` が自動実行される（`/workspace/install/.colcon_build_succeeded` sentinel でガード、2 回目以降は skip）。

### 5. MongoDB シードデータの投入（初回のみ）

```bash
docker compose exec tms restore-db.sh
```

`ros2_tms_for_construction/demo/rostmsdb_collections.zip` を毎回上書き展開して `mongorestore`、続いて `parameter` collection から `description` (string) フィールドを除去（subtask 側の型不整合 workaround）。

完了すると `rostmsdb` に task 5 件・parameter 20 件が投入される。`task_id=1`（zx200 掘削積込タスク、6 サブタスク。`task_id=2` は同じ内容を 4 サイクル）はシードに含まれているため別途登録不要。

## 次のステップ

- 動作確認（task_id=1 完走）: [usage.md](usage.md)
- src 編集 → 再ビルドの運用: [development.md](development.md)
