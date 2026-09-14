# Ansible による配置と更新

本番の実際の設定値・UFW対応・導入結果は [本番構成と運用記録](../docs/production.md) にまとめています。

手元のこのリポジトリから、SSH 経由で VPS へ設定を配置し、Compose のビルド・起動と Caddy の再読み込みを行います。
`prepare.yml` はファイル配置のみ、`deploy.yml` は配置を含む通常のデプロイです。
Docker/Compose・Python 3・sudo・ホスト mysqld が VPS にあることが前提です。
Docker Compose は 2.18.0 以降が必要です。

最初の範囲はデプロイに限定しています。OS、Docker のインストール、mysqld、ファイアウォールは管理しません。
0yp ソースの git 更新、TOML、フロントエンドの事前ビルドも従来どおり管理してください。
実際に VPS にあるソースからビルドするため、デプロイ対象のコミットを事前に確認します。

## 手元の準備

リポジトリルートで実行します。

```bash
python3 -m venv ansible/.venv
ansible/.venv/bin/pip install -r ansible/requirements.txt
ansible/.venv/bin/ansible-galaxy collection install -r ansible/requirements.yml -p ansible/collections
cp ansible/inventory.example.yml ansible/inventory.yml
# ansible_host を SSH 接続先に編集
export ANSIBLE_COLLECTIONS_PATH="$PWD/ansible/collections"
```

SSH のホスト鍵確認は有効のままです。通常の SSH で接続先を確認してください。
`--ask-become-pass` は VPS の sudo パスワード入力用です。パスワード不要の sudo なら省略できます。
秘密情報は inventory に書かず、VPS 上の `.env` に保持します。

## 初回配置と移行

```bash
ansible/.venv/bin/ansible-playbook -i ansible/inventory.yml ansible/prepare.yml --ask-become-pass
```

既定では `/opt/yayaue.me` へ配置します。ここは Ansible が管理する配置先で、Git チェックアウトは不要です。
既存の新 Compose 配置先が別にあるなら、**そのパスを inventory の deploy_dir に指定**してください。
配置先を変更すると Compose プロジェクト名も変わり、別スタックになるため、運用開始後は固定します。
`prepare.yml` は起動・再読み込みをしませんが、稼働中の配置先では静的ページの変更は直ちに反映されます。

VPS で `.env` を新規作成・編集します。既存ファイルがある場合は上書きせず編集してください。

```bash
sudo cp -n /opt/yayaue.me/.env.example /opt/yayaue.me/.env
sudo chmod 600 /opt/yayaue.me/.env
sudoedit /opt/yayaue.me/.env
```

今回確認できた本番配置では次を設定します。DB の値は既存 `/opt/peercast-0yp/.env` から転記します。

```dotenv
PEERCAST_0YP_DIR=/opt/peercast-0yp
CADDY_DATA_VOLUME=peercast-0yp_caddy_data
CADDY_CONFIG_VOLUME=peercast-0yp_caddy_config
```

初回の旧 Caddy・0yp 停止、証明書引き継ぎ、DB 接続許可の確認は [ルート README](../README.md#既存環境の移行) の手順で実施します。
旧コンテナが稼働中なら `deploy.yml` は配置前に停止します。旧環境の自動停止・削除は行いません。
初回切り替えは手動で動作確認まで済ませ、その後の更新に以下を使ってください。

## 通常の更新

手元のインフラ設定と、VPS 上の0ypソースを目的の版に更新した後で実行します。

```bash
export ANSIBLE_COLLECTIONS_PATH="$PWD/ansible/collections"
ansible/.venv/bin/ansible-playbook -i ansible/inventory.yml ansible/deploy.yml --syntax-check
ansible/.venv/bin/ansible-playbook -i ansible/inventory.yml ansible/deploy.yml --check --diff --ask-become-pass
ansible/.venv/bin/ansible-playbook -i ansible/inventory.yml ansible/deploy.yml --ask-become-pass
```

`--check` は前提条件の読み取りとファイル変更の予測のみです。未配置ファイルを検証できないため、Compose/Caddy の検証、ビルド、起動は実実行時に行います。
実実行では Compose 検証 → アプリビルド → Caddy 検証 → 起動 → 再読み込みの順です。
ビルドと再読み込みは毎回実行するので、そのタスクは毎回 changed と表示されます。
Caddy イメージは存在しない場合だけ取得します。イメージ更新は従来の手動手順で明示的に行ってください。
TOML だけを変更した場合は別途 `sudo docker compose restart peercast-0yp` を配置先で実行します。

設定ファイルの既存内容は Ansible copy の日時付きバックアップに残します。
配置したファイルは手元の内容で上書きされるため、変更はこのリポジトリで行ってください。
静的ファイルの管理対象は prepare.yml のリストです。追加・削除時にはリストと配置先も整合させてください。
秘密情報を含み得る Compose タスクの出力は no_log で抑制します。
失敗時は VPS の配置先で README の検証・ログコマンドを実行して原因を確認してください。

コンテナが running になったことまで待ちますが、現状の Compose にアプリの healthcheck はありません。
DB 接続、HTTPS ページ表示、HTTP の `/yp/index.txt`、PCP 掲載はデプロイ後に確認してください。
失敗時の自動ロールバックはありません。復元する版のソースと設定を用意して再実行します。

利用モジュール: [community.docker.docker_compose_v2](https://docs.ansible.com/projects/ansible/latest/collections/community/docker/docker_compose_v2_module.html)
