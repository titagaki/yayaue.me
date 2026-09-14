# 本番構成と運用記録

記録日: 2026-09-14（JST）。本番で実行したコマンドの出力と作業者の表示確認に基づく。
以下のIP、コンテナ名、ネットワーク名はこの時点の値であり、再構築時には確認し直す。

## 配置と管理範囲

| 対象 | 本番の設定 |
|---|---|
| VPS | さくらのVPS、ホスト名 `ik1-423-43594` |
| SSHユーザー | `debian`（sudoを使用） |
| 共通構成 | `/opt/yayaue.me/compose.yaml` |
| 本番の環境変数 | `/opt/yayaue.me/.env`、root所有・モード0600 |
| 0ypソース | `/opt/peercast-0yp` |
| 0yp設定 | `/opt/peercast-0yp/peercast-0yp.toml` |
| Caddy設定 | `/opt/yayaue.me/docker/caddy/Caddyfile` |
| サイトトップ | `/opt/yayaue.me/public/` |
| Composeプロジェクト | `yayaueme` |
| Composeバージョン | v5.1.0 |
| 本番Python | `/usr/bin/python3.11` |

当初の `/srv/yayaue.me`・`/srv/peercast-0yp` という配置案から、既存0ypの `/opt` 配置を維持する方針にした。
本番は **yayaue.meのComposeだけ** でCaddyと0ypを起動する。0yp側のComposeは本番では実行しない。
0yp側のDockerfileとソース、TOMLは引き続き使用する。`peercast-mi` はまだ追加していない。

0ypリポジトリの共通Composeはappのみ、開発用overrideはCaddyとMariaDBを追加、
本番用overrideはapp単体のホストDB接続設定という責務に整理した。
本番の0ypチェックアウトがこの変更を取得したか、今回の作業記録では未確認。

## 配信と証明書

| 入口 | 処理 |
|---|---|
| 80/TCP `/yp/index.txt` | `peercast-0yp:80` へ転送、HTTPのまま200を返す |
| 80/TCP その他 | 同じホスト・URIのHTTPSへ301転送 |
| 443/TCP・UDP `/yp` と `/yp/*` | `peercast-0yp:80` へ転送。パスを削らない |
| 443/TCP・UDP その他 | `public/` の静的ファイル |
| 7144/TCP | 0ypの7144へ直接公開、PCP用 |

0yp内部のHTTP 80とPCP 7144はTOMLで設定する。HTTPポートをホストへ直接公開しない。
稼働コンテナは `yayaueme-caddy-1` と `yayaueme-peercast-0yp-1`。固定の `container_name` は指定していない。

Caddyが証明書を自動管理する。旧Caddyが使用していた次のnamed volumeを引き継いだ。

| コンテナ内 | 実際のvolume名 | 用途 |
|---|---|---|
| `/data` | `peercast-0yp_caddy_data` | 証明書・秘密鍵など |
| `/config` | `peercast-0yp_caddy_config` | Caddyが自動保存する設定など |

`.env` の非秘密設定は次のとおり。

```dotenv
SITE_DOMAIN=yayaue.me
PEERCAST_0YP_DIR=/opt/peercast-0yp
CADDY_DATA_VOLUME=peercast-0yp_caddy_data
CADDY_CONFIG_VOLUME=peercast-0yp_caddy_config
```

旧プロジェクトが作成したvolumeのため、起動時に「expected yayaueme」「Use external: true」という警告が出る。
現状は `name` 指定で再利用しており、`external: true` には変更していない。実際に起動・HTTPS配信できることを確認済み。
通常の停止に `down -v` を使わない。旧・新Caddyを同じvolumeで同時起動しない。

最初の検証時、既定値の `yayaue-me-caddy-data` と `yayaue-me-caddy-config` も作成された。
その後 `.env` を修正して旧volumeへ切り替えた。未使用volumeの削除は今回実施していない。

## DBとファイアウォール

MariaDBはホストの既存サービスを継続利用する。コンテナ化、パスワード変更、スキーマ変更は行っていない。
0ypの `internal/config/config.go` は `.env` を直接読まず、`os.Getenv` でDB環境変数を読む。
新Composeは `/opt/yayaue.me/.env` のDB値を `environment` で渡す。
本番では `/opt/peercast-0yp/.env` は読み込まれない。

DBのユーザー名・パスワード・DB名・ポートは旧設定から転記した。パスワードはドキュメントに記載しない。
DB接続先はComposeで `DB_HOST=host.docker.internal` に指定している。

| 確認項目 | 結果 |
|---|---|
| ホストMariaDB待受 | `0.0.0.0:3306` |
| 接続元アカウント | `app@%` が存在（ほかにlocalhost・127.0.0.1用も存在） |
| 新ネットワーク | `yayaueme_default`、`172.20.0.0/16`、gateway `172.20.0.1` |
| `host.docker.internal` | `host-gateway` 経由で `172.17.0.1` に解決 |
| 旧DB許可 | UFWで `172.19.0.0/16` から3306を許可 |

新ネットワークからのDB接続がエラー2002・タイムアウト110になった。
UFWが旧ネットワークだけを許可していたため、以下を追加したところ `SELECT 1` が成功した。

```bash
# VPSで実施済みの設定（同じ値を無条件に再適用しない）
sudo ufw allow from 172.20.0.0/16 to 172.17.0.1 port 3306 proto tcp
```

旧 `172.19.0.0/16` 用ルールは残している。UFWのデフォルトはincoming/routedをdeny、outgoingをallow。
公開許可は22/TCP、80/TCP、443/TCP・UDP、7144/TCP。DBの外部全体への許可は追加していない。

固定サブネットは使っていないため、**Composeのdown後などにネットワークを再作成すると、UFWルールと割当がずれる可能性がある**。
その場合は次で現在値を確認して、必要な送信元・宛先だけを許可する。通常の更新はdownを挟まずupで行う。

```bash
# VPS
cd /opt/yayaue.me
sudo docker network inspect yayaueme_default \
  --format '{{range .IPAM.Config}}subnet={{.Subnet}} gateway={{.Gateway}}{{end}}'
sudo docker compose run --rm --no-deps --entrypoint sh peercast-0yp \
  -c 'grep host.docker.internal /etc/hosts'
sudo ufw status verbose
```

DBの疎通・認証確認に使用したコマンド。一時コンテナにクライアントをインストールするので、パッケージ取得への通信が必要。
SQLは読み取りのみで、稼働中のアプリコンテナは変更しない。

```bash
# VPS、/opt/yayaue.me
sudo docker compose run --rm --no-deps --entrypoint sh peercast-0yp -c '
  test -n "$DB_PASSWORD" || { echo "DB_PASSWORD is empty"; exit 1; }
  apk add --no-cache mariadb-client >/dev/null &&
  MYSQL_PWD="$DB_PASSWORD" mariadb --protocol=TCP --connect-timeout=5 \
    -h "$DB_HOST" -P "$DB_PORT" -u "$DB_USER" "$DB_NAME" \
    -e "SELECT 1;"
'
```

このクライアントは「insecure passwordless login」という警告を出したが、パスワード非空チェックと接続は成功した。
警告だけでパスワード未設定とは判断しない。確認用クライアントの挙動であり、DB通信のTLS有効性は今回検証していない。

## Ansibleによる通常更新

手元のWSLに `ansible/.venv` と `ansible/collections` を準備済み。
Python 3.12.3、ansible-core 2.19.13、community.docker 5.3.0で検証した。
`ansible/inventory.yml` はGit対象外。SSHユーザーはdebian、`deploy_dir` は `/opt/yayaue.me`。
WSLの公開鍵をVPSのdebianユーザーの `~/.ssh/authorized_keys` に追記し、SSH接続とAnsible pingに成功した。
Windowsの秘密鍵をWSLへコピーする方法は採用していない。

手元でインフラ設定を編集し、0ypを更新する場合は先にVPSのソースを目的の版へ更新する。
PlaybookはGitのpull、TOML変更、フロントエンドの事前ビルド、DB/UFWの管理を行わない。
詳細は [Ansible README](../ansible/README.md) を参照。

```bash
# 手元のWSL（VPSではない）
cd ~/src/yayaue.me
export ANSIBLE_COLLECTIONS_PATH="$PWD/ansible/collections"
ansible/.venv/bin/ansible-playbook \
  -i ansible/inventory.yml ansible/deploy.yml \
  --check --diff --ask-become-pass

# 予行確認の結果を確認してから適用
ansible/.venv/bin/ansible-playbook \
  -i ansible/inventory.yml ansible/deploy.yml \
  --ask-become-pass
```

`BECOME password` はVPSのdebianユーザーのsudoパスワード。
checkモードはファイル差分の予測と前提条件確認まで。ビルド・Compose/Caddy検証・起動・再読み込みは実実行で行う。
`changed=2` は毎回実行するビルドとCaddy再読み込みで発生する。自動ロールバックはない。

本番のyayaue.meは最初にGit cloneしたが、Ansibleは手元のファイルをコピーする。
以降は手元の設定を正とし、VPS側だけを編集したり、GitのHEADだけで配信中の版を判断したりしない。
0ypソースのGit更新はこれとは別に管理する。

TOMLだけを変更した場合は、VPSの `/opt/yayaue.me` で `sudo docker compose restart peercast-0yp` を実行する。
Ansibleの起動待ちはコンテナがrunningになるまでであり、アプリ・DB・PCPの機能確認の代わりにはならない。

## 移行履歴とバックアップ

1. 共通構成を `/opt/yayaue.me` にcloneし、`.env` を作成。
2. Compose構成検証、0ypのビルド、Caddy設定検証を実施。
3. 証明書volume名を修正し、新ネットワークのUFW許可を追加。DBへの `SELECT 1` 成功。
4. 旧設定・ホストDBをバックアップ。
5. 更新前の旧Composeで `down`（`-v`なし）を実行し、旧Caddy・app・ネットワークを削除。
6. 停止したCaddyのvolumeをバックアップ。
7. 新Composeで起動し、HTTP/HTTPSの応答とブラウザ表示を確認。
8. WSLのAnsibleからping、checkモード、実デプロイを順に実行。

バックアップはVPSの **`/root/yayaue-migration.Wb70TR/`** に保存済み。

| ファイル | 内容 |
|---|---|
| `old-config.tar.gz` | 旧共通・本番Compose、旧.env、TOML、Caddy設定、public |
| `databases.sql` | `mariadb-dump --all-databases --single-transaction --routines --events` の出力 |
| `caddy-volumes.tar.gz` | 旧Caddyのdata/config volumeの `_data` |

rootのみ読み取り可能な権限で作成した。認証情報・秘密鍵を含むため、Gitへ追加しない。
DBバックアップは旧アプリ停止前の時点のもの。移行自体はDBスキーマを変更していない。
バックアップの復元テスト、別ホストへの保管は今回の作業では未実施。

旧構成へ戻す場合は新Composeを `down`（`-v`なし）で停止し、
バックアップの旧Compose・設定を `/opt/peercast-0yp` に復元して旧構成を起動する。
旧ネットワーク用UFWルールと証明書volumeは残している。
通常の構成切り戻しでDBをダンプ時点へ戻す必要はない。DB復元は移行後のデータを失う別操作なので、自動実行しない。

## 確認結果と残る運用事項

| 項目 | 結果 |
|---|---|
| HTTP `/yp/index.txt` | 200 |
| HTTP `/` | 301、`https://yayaue.me/` へ転送 |
| HTTPS `/` | 200（curlで証明書検証を無効化していない） |
| HTTPS `/yp/` | 200 |
| ブラウザ | 作業者からページ表示に問題なしとの確認 |
| DB | 新構成の環境変数・ネットワークで `SELECT 1` 成功 |
| Ansible check | ok=10、changed=0、failed=0 |
| Ansible実デプロイ | ok=14、changed=2、failed=0。起動タスクはok、再読み込み成功 |

PCPは7144で起動・公開されていることを確認したが、個別の配信再掲載テスト結果は記録していない。
Ansible実デプロイ後のブラウザ再確認についても、独立した結果は未記録。

volumeの所有プロジェクト警告、Python interpreter discoveryの警告は残っている。
UFWのサブネット許可、DB、バックアップはAnsible管理外。
外部volume化やネットワーク許可の自動化は今後の検討事項であり、今回の導入には含めていない。

## peercast-mi 導入後の確認（2026-09-14 追記）

ユーザーが提示した実行結果とブラウザー確認に基づく。VPSの `/opt/peercast-mi` に `5de6feb` を取得し、Ansible実適用は `ok=17 changed=8 unreachable=0 failed=0 skipped=1`。Compose/Caddy検証、ビルド、起動、mi再起動、Caddy再読み込みが成功した。既存の証明書volume所有プロジェクト警告は継続。

ユーザーが `https://yayaue.me/mi/` の表示とXログイン成功、既存 `/yp/` の表示を確認した。miの番組一覧は空との報告。導入時の番組一覧取得先は0ypのみで、空の原因や0ypの掲載件数は独立確認していない。実OBS配信・映像視聴・外部PCP接続は未確認。

その後、ユーザーが一覧取得先をすべて追加するよう指定したため、手元の本番TOMLにローカルと同じSP・p@YPを追加した。0ypを先頭に維持し、配信掲載先は変更しない。この追加のVPS適用と3件の一覧取得結果はまだ未確認。TOML構文・3件の取得先と順序、差分チェックをローカルで検証した。

### 全YP表示・視聴確認と管理パネル修正

ユーザー提示のYP追加後のAnsible結果は `ok=17 changed=4 unreachable=0 failed=0 skipped=1`。TOML配置、ビルド、mi再起動、Caddy再読み込みが成功し、その後ユーザーが一覧表示と視聴成功を確認した。配信入力の実OBSテストは未確認。

管理パネルが使えないとの報告に対し、手元のコード調査で管理UIの接続先が閲覧者PCのloopbackであり、サイトに管理API入口がなかったことを確認した。ユーザーはXの特定アカウント認証を指定し、数値ID `121039382` を提示した。手元の本番設定の `site.admin_x_ids` に登録した。管理者用APIの実装を含むmiソースの更新・再ビルドが必要。この管理パネル修正の本番適用・画面確認はまだ未実施。

### 一覧を0yp掲載分のみに変更（2026-09-14）

未掲載のローカル配信枠がmiの一覧に混ざるとの報告を受け、ユーザー指定により本番のHTTP番組一覧取得先を0ypの `https://yayaue.me/yp/index.txt` のみに変更した。SP・p@YPのPCP接続先は維持し、両者の `channels_url` を削除した。mi側ではYP未掲載のローカルチャンネル追加を廃止する変更を用意した。TOMLの構文と一覧URLが0ypのみであることをローカル検証。本番適用は未実施。
