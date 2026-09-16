# peercast-miのログイン・配信記録

2026-09-16にローカル設定・手順を追加。その後ユーザーの実行結果で本番migration・起動・配信履歴保存を確認。無期限保存への変更はアプリ更新と設定再配置が必要。

miの監査ログは既存ホストMariaDBに独立した `peercast_mi` DBを作成して保存する。0ypのDB・ユーザー・時刻設定は流用しない。配信キーとフォーム設定のJSONは従来どおり。

Composeは `host.docker.internal` と専用の `PEERCAST_AUDIT_DB_*` 環境変数を渡す。設定は有効、無期限保存（retention_days=0）。資格情報・スキーマがまだなければ `/config/site-data/audit` にJSONLを退避し、サイトと配信は継続する。容量上限512MiBを超えると新しいログが失われるため、DB準備前のまま放置しない。

## 導入順序

### 1. VPS: DBとユーザーの準備

現在のMariaDBバージョンとバックアップを確認する。ローカル検証はMariaDB 11.4.13。以下はDB管理者が専用の空DBに適用するSQL例。パスワードは別々の実値を設定し、Git・チャット・通常ログには残さない。接続元許可とUFWは現在のComposeネットワークに合わせる。

```sql
CREATE DATABASE peercast_mi CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'mi_runtime'@'%' IDENTIFIED BY 'replace-runtime-password';
GRANT SELECT, INSERT, UPDATE, DELETE ON peercast_mi.* TO 'mi_runtime'@'%';
CREATE USER 'mi_migrate'@'%' IDENTIFIED BY 'replace-migration-password';
GRANT SELECT, INSERT, CREATE, ALTER, INDEX ON peercast_mi.* TO 'mi_migrate'@'%';
```

`%`は例で、DBポートを外部全体へ開ける指示ではない。0ypで使用中のDBアカウントを変更しない。実行用の資格情報をVPSの `/opt/yayaue.me/.env` に追加する。`.env.example` の変数名を使用し、USERはmi_runtime、NAMEはpeercast_mi。ファイルはroot所有0600を維持する。

### 2. VPSでソース更新、手元WSLから設定配置

VPSの `/opt/peercast-mi` を目的のコミットへ更新する。次に手元WSLの `/home/megan/src/yayaue.me` で既存の `ansible/prepare.yml` を実行してComposeとTOMLを配置する。`prepare.yml` は起動しない。DBと資格情報はAnsibleの管理外。

### 3. VPS: イメージ作成と明示migration

```sh
cd /opt/yayaue.me
sudo docker compose config -q
sudo docker compose build peercast-mi
```

root所有0600の `/root/peercast-mi-migrate.env` を作り、`PEERCAST_AUDIT_DB_USER` と `PEERCAST_AUDIT_DB_PASSWORD` だけを移行用の値に設定する。このファイルはリポジトリ外に置く。

```sh
# VPS。後のenv-fileが移行用ユーザーとパスワードだけを上書きする。
sudo docker compose --env-file .env --env-file /root/peercast-mi-migrate.env \
  run --rm --no-deps --entrypoint /app/audit-migrate peercast-mi
```

`audit schema version 1 applied` を確認する。通常起動はmigrationを実行しない。適用済み版のchecksum不一致は手作業で上書きせず、イメージ・ソースとDBの対応を確認する。

### 4. 手元WSL: 通常デプロイ

既存の `ansible/deploy.yml` を実行する。通常更新はdownを挟まない。コンテナには `.env` の実行用ユーザーが渡される。移行用env-fileを通常のupに指定しない。

### 5. ブラウザーとVPS: 記録確認

X管理者でログインし、同じブラウザーで `/mi/site/api/audit/status` を確認する。`enabled=true`、`degraded=false`、`lastSuccess`があり、滞留が減ることを確認する。一般アカウントからは403。

DB管理者は `audit_events` のauth.loginとbroadcasts / broadcast_inputsをSQLで確認する。日時はUTC。実メディアの開始はfirst_media_at、枠の作成はcreated_atで別物。異常終了ではended_atがNULLのままinterruption_detected_atが入る。

## 運用

- DB断ではスプール、メモリー・ディスク上限では欠落を通常ログへ出す。`.bad`の隔離ファイルは管理者が原因を確認して保存・除去する。
- DBと `data/peercast-mi/site-data/audit` をそれぞれバックアップする。キーや既存データを上書きしない。
- 記録を止める場合は手元のTOMLのaudit.enabledをfalseにして通常配置・再起動する。既存DBやスプールを削除する必要はない。
- X管理者は `/mi/admin` の「ログ」で操作ログ・配信履歴・RTMP受信区間を検索できる。初期表示は過去7日、条件をクリアすれば古い記録も対象になる。一般アカウントには公開しない。履歴画面の導入に追加のDBマイグレーションは不要（アプリの更新・再ビルドは必要）。
- 実装仕様はpeercast-miの `docs/spec/audit.md`、検証は `docs/reviews/2026-09-16-audit-implementation.md`。

保持期限変更: 新版のpeercast-miはretention_days=0でDBの自動削除・再送時の期限切れ破棄を行わない。旧版の0は90日なので、TOMLだけでなくアプリも更新する。テーブルの再migrationは不要。
