# Windows (WSL2) で動かす

Windows 11 の PC で、Windows 側の Unity (OperaSim-PhysX) の ZX200 を、WSL2 の中の本リポジトリ (CPS 一式) と Node-RED から動かす手順。Linux ホストの手順 ([setup.md](setup.md)・[usage.md](usage.md)) との違いは、WSL2 と Docker Engine の用意、GUI の出し方 (WSLg)、Unity の接続先の 3 点で、コンテナの中身は同じ。

> **社内ネットワークで使う方へ**: TLS を検査するプロキシの内側では、WSL への CA 証明書の取り込みと `certs/` への写しが要る。手順は社内向けの補足 (要アクセス権) を参照: <https://shimz.box.com/s/0kfmgk2y89pvzdn9zm6akwet2hizdt0g>

## 構成

| 置き場 | 動くもの | 役割 |
|---|---|---|
| Windows | Unity (OperaSim-PhysX) | ZX200 のシミュレータ。`127.0.0.1:10000` で WSL の endpoint に接続する |
| Windows | ブラウザ | Node-RED の画面 (`http://127.0.0.1:1880/ui`) |
| WSL | コンテナ `opera_cps_dev` | ROS 2 humble。endpoint、MoveIt、CPS、RViz、緑ボタン |
| WSL | コンテナ `opera_cps_mongodb` | CPS のタスク DB |
| WSL | コンテナ `shimz-node-red` | Node-RED ([shimz-robotics/nodered-ros2-opera](https://github.com/shimz-robotics/nodered-ros2-opera)、private) |

- Windows と WSL の間は、WSL の既定のネットワーク (NAT) のままで繋がる。NAT の localhost 転送で、Windows の `127.0.0.1:10000` が `network_mode: host` のコンテナに届く。`.wslconfig` の変更、ポートの転送設定、管理者権限は要らない
- RViz と緑ボタンの窓は WSLg で Windows の画面に出る。`xhost` は要らない
- コンテナ間の DDS は、Linux ホストと同じく Cyclone DDS の localhost multicast で繋がる
- Docker Desktop ではなく、WSL のディストリビューションの中に Docker Engine を入れた構成で確かめた

## 前提

- Windows 11、WSL 2 (WSLg が使える版)
- WSL のディストリビューション Ubuntu-24.04 (systemd 有効)
- その中に Docker Engine と Docker Compose v2 ([Docker の公式手順](https://docs.docker.com/engine/install/ubuntu/))、`git`、`vcstool` (`sudo apt install vcstool`。ROS の apt リポジトリを入れていない Ubuntu では `python3-vcstool` の名前では入らない)
- 自分のユーザーを docker グループに入れる (`sudo usermod -aG docker $USER` の後、WSL の端末を開き直す)。公式手順の本編には無く、[post-install](https://docs.docker.com/engine/install/linux-postinstall/) にある
- Windows に Unity Hub と [OperaSim-PhysX](https://github.com/shimz-robotics/OperaSim-PhysX) (`shimz-main` branch)
- Node-RED を使う場合は nodered-ros2-opera を clone できる GitHub の権限

WSL の端末で `docker run --rm hello-world` が sudo なしで通れば準備できている (`docker compose version` は daemon に繋がないので、権限が無くても通ってしまう)。

## 初回の準備

所要時間はおよそ 40 分 (大半はダウンロードとビルドの待ち時間)。

### 1. GitHub の認証を Windows と共有する (Node-RED を使う場合)

nodered-ros2-opera は private なので、clone に認証が要る。WSL の git から Windows の Git Credential Manager を使う:

```bash
git config --global credential.helper "/mnt/c/Program\ Files/Git/mingw64/bin/git-credential-manager.exe"
```

Windows で GitHub にログイン済みなら、clone の時に何も聞かれない。聞かれたら、開いたブラウザでログインする。

### 2. opera_cps_ws を用意する (約 15 分)

```bash
cd ~
git clone https://github.com/shimz-robotics/opera_cps_ws.git
cd opera_cps_ws
printf 'UID=%s\nGID=%s\n' "$(id -u)" "$(id -g)" > .env
vcs import --force src < src.repos
docker compose build
```

- TLS を検査するプロキシの内側では、`docker compose build` の前に CA 証明書を `certs/` に置く ([setup.md の該当節](setup.md#tls-を検査するプロキシの内側でビルドする場合))
- `docker compose build` は 8 分ほど (ベース image の取得を含めると 15 分ほど)

続けて、初回の起動と DB の投入をする。

```bash
touch ~/.Xauthority
docker compose up -d
docker compose logs -f tms
```

`~/.Xauthority` は compose が bind mount するので、先にファイルとして作っておく (無いと Docker がディレクトリとして作ってしまう。WSLg では中身は空でよい)。初回の起動ではコンテナの中で colcon build が自動で走る (約 3 分)。`Summary: ... packages finished` が出たら Ctrl-C で抜け、DB にタスクを入れる:

```bash
docker compose exec tms restore-db.sh
```

最後に `[restore-db] Done.` が出れば完了。

### 3. nodered-ros2-opera を用意する (Node-RED を使う場合、約 10 分)

```bash
cd ~
git clone https://github.com/shimz-robotics/nodered-ros2-opera.git
cd nodered-ros2-opera
cp .env.example .env
sed -i "s/^UID=.*/UID=$(id -u)/; s/^GID=.*/GID=$(id -g)/" .env
docker compose build
```

プロキシの内側では、こちらも `docker compose build` の前に CA 証明書を `certs/` に置く (nodered-ros2-opera の README)。

### 4. Unity の接続先を設定する

1. Unity Hub で OperaSim-PhysX を開く
2. Project で `Assets/Other 3rd Party Series/Resources/ROSConnectionPrefab` を選ぶ
3. Inspector の **ROS IP Address** を `127.0.0.1` にする。Port は `10000` のまま。前後に空白を入れない (入ると「ROS IP is invalid」と出て繋がらない)
4. Ctrl+S で保存する

## 毎回の使い方

WSL の端末を 2 つ開いて使う。

### CPS で掘削・積込タスクを流す

端末 1 (Unity との中継):

```bash
cd ~/opera_cps_ws
docker compose up -d
./scripts/exec.sh ros2 launch ros_tcp_endpoint endpoint.py
```

`Starting server on 0.0.0.0:10000` が出たら、Unity で ▶ (再生) を押す。画面左上の ROS の表示が青になれば接続できている。

端末 2 (CPS 一式):

```bash
cd ~/opera_cps_ws
./scripts/exec.sh ros2 launch /workspace/launch/bringup.launch.yaml
```

RViz と緑ボタンの窓が開く。緑ボタンを押すと task_id=1 が始まる (約 35 秒)。

RViz で ZX200 のリンクがピンク色 (接触) の時は、先に姿勢を変える。MotionPlanning パネルの **Joints** タブで `boom_joint` を少し下げ、**Planning** タブで **Plan & Execute** を押す。

もう一度流す時や、タスクが途中で止まった時は、端末 2 で Ctrl-C を押して bringup を起動し直す (Unity は再生したままでよい)。1 回終えたタスクや途中で止まったタスクが残っている間は、緑ボタンを押しても次のタスクが始まらない。

### Node-RED から操作する

端末 1・2 の CPS 一式が動いている状態で、別の端末から起動する。

```bash
cd ~/nodered-ros2-opera
docker compose up -d
```

Windows のブラウザで `http://127.0.0.1:1880/ui` を開き、「掘削開始」を押す。

### 終わる時

1. Unity の ▶ を止める
2. 端末 1・2 で Ctrl-C を押す
3. コンテナを止める

```bash
cd ~/opera_cps_ws && docker compose stop
cd ~/nodered-ros2-opera && docker compose stop
```

PC が重い時は、Windows の PowerShell で `wsl --shutdown` を実行すると WSL ごと止まる。

## 困った時

| 症状 | 原因と対処 |
|---|---|
| Unity に「ROS IP is invalid」と出る | ROS IP Address の前後に空白が入っている。入れ直す |
| Unity で再生しても接続が赤のまま | 端末 1 の endpoint が動いていない。`Starting server on 0.0.0.0:10000` が出ているか確かめる |
| Unity を再生し直したら endpoint が落ちた (Broken pipe) | endpoint の再接続の既知の不具合。端末 1 で endpoint を起動し直す |
| しばらく放っておいたらコンテナが止まっていた (exit 255) | WSL の端末を全部閉じると、WSL の VM が自動で止まる。使う間は端末を開いておく。止まったら `docker compose up -d` からやり直す |
| `docker compose build` が `CERTIFICATE_VERIFY_FAILED` で止まる | TLS を検査するプロキシの CA が `certs/` に無い ([setup.md の該当節](setup.md#tls-を検査するプロキシの内側でビルドする場合)) |
| 緑ボタンを押してもタスクが始まらない、途中で止まる | 前のタスクが残っている。端末 2 で Ctrl-C を押して bringup を起動し直してから、1 回だけ押す。RViz でリンクがピンク色なら、先に `boom_joint` を下げて Plan & Execute する |
| bringup の起動時に `Failed getting a result from calling /zx200/controller_manager/load_controller` が出て、ZX200 が動かない | Unity を再生する前に bringup を起動した。Unity を再生した状態で、端末 2 の bringup を起動し直す |
| `docker compose build` が `permission denied ... docker.sock` で止まる | docker グループに入っていない。「前提」の `usermod` の後、端末を開き直す |
| 動きがカクつく | 制御器の時刻は Unity の `/clock` で進むので、PC が重いと滑らかさが落ちる。ほかの重いアプリを閉じる |
| `http://127.0.0.1:1880/ui` に接続できない | Node-RED のコンテナが止まっている。`cd ~/nodered-ros2-opera && docker compose up -d` |
| pull の後で bringup が `/workspace/launch` が無いと言って起動しない | `compose.yaml` が変わった。`docker compose up -d` でコンテナを作り直す ([development.md](development.md)) |
