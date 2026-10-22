# Development — 編集・再ビルドの運用

ホスト側で `src/` を編集して container 内に反映するフロー。コマンドはホストから `scripts/exec.sh` 経由で叩く前提で書く（対話 shell でやりたい時は `scripts/shell.sh` で入ってから後半部分を実行）。

## 再ビルドのまとめ

| やりたいこと | コマンド | 所要時間 |
|---|---|---|
| Python 編集 | ノード再起動だけ (`--symlink-install` で symlink 経由、build 不要) | 秒 |
| C++ 編集 | `./scripts/exec.sh colcon build --symlink-install --packages-up-to <pkg>` | 分 |
| 全部 build し直し | sentinel 削除 + `docker restart` (entrypoint が初回扱いで実行) | 10 分前後 |
| Dockerfile 変更 | `docker compose down -v` + `compose build` + `up -d` | 25 分（cache 効けば短縮） |

## ① Python 編集 → 即反映

`--symlink-install` のおかげで `/workspace/install/.../foo.py` はホスト側ファイルへの symlink。**再 build 不要、ノード再起動だけ**:

```bash
# Host: src/ 配下で編集
vim src/ros2_tms_for_construction/tms_ts/tms_ts_subtask/scripts/foo.py
# ノードが走っているターミナル: Ctrl-C → 再 launch
./scripts/exec.sh ros2 launch <pkg> <launch.py>
```

## ② C++ 編集 → 部分 build

```bash
# Host: 編集
vim src/.../my_node.cpp

# Host から 1 行で部分 build
./scripts/exec.sh colcon build --symlink-install --packages-up-to my_pkg

# ノード再起動
./scripts/exec.sh ros2 launch <pkg> <launch.py>
```

`--packages-up-to <pkg>` は「指定パッケージ + その依存」だけを build する。

## ③ 新パッケージを `src.repos` に追加

```bash
# Host
vcs import src/ < src.repos             # 新規 entry のみ clone される (既存 dir は skip)
./scripts/exec.sh colcon build --symlink-install --packages-up-to new_pkg
```

## ④ `src.repos` の pin を bump (commit を新しくしたい)

```bash
# Host
vcs import --force src/ < src.repos     # 既存 dir を destroy して再 clone
./scripts/exec.sh colcon build --symlink-install --packages-up-to bumped_pkg
```

`ros2_tms_for_construction` の pin を上げて DB のシード (`demo/rostmsdb_collections.zip`) が変わった時は `docker compose exec tms restore-db.sh` を実行し直す。

特定 repo だけ branch 切替したい時はホスト側で `cd src/<repo> && git checkout <branch>` で OK。

## ⑤ 「初回 colcon build」をやり直す

entrypoint は sentinel `/workspace/install/.colcon_build_succeeded` でガードされているので、sentinel を消して restart すれば初回扱いで再実行:

```bash
docker exec opera_cps_dev rm -f /workspace/install/.colcon_build_succeeded
docker restart opera_cps_dev
docker logs -f opera_cps_dev             # build 進捗
```

named volume の `build/` `install/` `log/` は残ったまま増分 build なので速い。完全クリーンしたければ:

```bash
docker exec opera_cps_dev rm -rf /workspace/build /workspace/install /workspace/log
docker restart opera_cps_dev
```

## ⑥ Dockerfile / requirements.txt 変更

```bash
# Host
docker compose down -v                                # named volume も削除 (build/install/log/db 全部消える)
UID=$(id -u) GID=$(id -g) docker compose build
docker compose up -d                                  # 初回起動扱いで entrypoint が全 colcon build
docker compose exec tms restore-db.sh                 # DB seed も投入し直し
```

## requirements.txt 同期

`requirements.txt` は本リポジトリ直下に vendor している。`src.repos` の `ros2_tms_for_construction` pin を bump する時は、同 commit の `requirements.txt` とこちらの `requirements.txt` を手動で diff して同期させること（CI チェックは未導入、必要性が見えたら再検討）。

```bash
# bump 後の同期チェック例
diff requirements.txt src/ros2_tms_for_construction/requirements.txt
```
