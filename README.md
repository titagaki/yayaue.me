# yayaue.me VPS 構成

現在の本番配置（`/opt`）と移行・運用記録は [docs/production.md](docs/production.md) を参照してください。以下の `/srv` は新規配置の例です。

VPS 共通の Caddy・0yp・peercast-mi をこの Compose で管理します。

```text
/srv/yayaue.me/          compose.yaml, .env, docker/caddy/, public/
/srv/peercast-mi/        peercast-mi ソース
/srv/peercast-0yp/       Dockerfile のあるアプリソース、peercast-0yp.toml
```

| 接続 | 処理 |
|---|---|
| 80/TCP `/yp/index.txt` | `peercast-0yp:80` へ転送（HTTP 対応プレイヤー用） |
| 80/TCP その他 | 同じホスト・URI の HTTPS へ恒久転送 |
| 443/TCP・UDP `/yp`、`/yp/*` | `peercast-0yp:80` へ転送。パスは削らない |
| 443/TCP・UDP `/mi`、`/mi/*` | `peercast-mi:8080` へパスを保持して転送 |
| 7154/TCP | peercast-mi PCP |
| 1935/TCP | peercast-mi RTMP（ストリームキー認証） |
| 443/TCP・UDP その他 | `public/` の静的ファイル |
| 7144/TCP | app の 7144 へ直接公開（PCP） |
| app → ホスト DB | `host.docker.internal:${DB_PORT}` |

Caddyfile と静的ページは peercast-0yp の既存ファイルを移植しています。
元リポジトリ側のコピーは単独開発用です。アプリの HTTP ポートや DB はホストへ公開しません。
コンテナ間は Compose のサービス名で接続し、固定コンテナ名・固定サブネットは使いません。

peercast-mi の追加前に [追加手順](ansible/README.md#peercast-mi-の追加) に従ってソース・認証情報・永続ディレクトリを用意します。既存の `.env` に X 認証情報がないまま新しい Compose を適用すると構成検証が失敗します。

## 初回起動

Linux VPS に Docker Engine と Docker Compose を用意し、上記配置に各リポジトリをチェックアウトします。
`SITE_DOMAIN` の DNS を VPS に向け、80/TCP、443/TCP・UDP、7144/TCP を到達可能にしてください。
既存環境からの切り替えは、先に次の「既存環境の移行」を確認してください。

```bash
cd /srv/yayaue.me
cp .env.example .env
chmod 600 .env
# .env を編集（公開ドメイン、既存ホスト DB の接続情報）
```

`.env` の DB パスワードは既存 mysqld の値を設定します。`$` や `#` を含む場合は値をシングルクォートで囲みます。
本番では peercast-0yp 側の `.env` は読みません。アプリには必要な DB 環境変数だけを渡します。
手元の配置では `PEERCAST_0YP_DIR=/home/megan/src/go/peercast-0yp` に変更できます。
既定の build context は `../peercast-0yp`、Dockerfile は `docker/Dockerfile` です。

`../peercast-0yp/peercast-0yp.toml` の `[http] port = 80` と `[pcp] port = 7144` を確認し、
公開 URL をドメインに合わせてください。現在のアプリとフロントエンドは `/yp/` を使用するため、このパスを維持します。
フロントエンドを変更した場合は、アプリの README に従って先にビルドします。

本番 DB の作成・スキーマ適用・バックアップは従来どおりホスト側で行います。この Compose は mysqld を起動・初期化しません。
`host-gateway` でホストに接続するので、mysqld がコンテナから到達できるホスト側アドレスで待ち受け、
DB ユーザーの接続元許可とホストのファイアウォールが新しい Docker ネットワークを許可している必要があります。
localhost のみの待受では接続できません。DB ポートをインターネットへ開放する必要はありません。

```bash
cd /srv/yayaue.me
# 秘密情報を表示しない構成検証
docker compose config -q
docker compose build peercast-0yp peercast-mi
# 証明書取得は行わず、設定を検証
docker compose run --rm --no-deps caddy caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
docker compose up -d
docker compose ps
docker compose logs --tail=100 caddy peercast-0yp
```

初回の自動証明書取得には DNS と外部からの到達性が必要です。
起動後は DB 接続エラーがないこと、HTTP の `/yp/index.txt` がリダイレクトされないこと、
HTTP の `/` が HTTPS へ転送されること、HTTPS の `/` と `/yp/` の表示、PCP の掲載を確認してください。

## 既存環境の移行

停止を伴います。旧 Caddy と旧 app を残したまま新構成を起動すると、80・443・7144 番が競合します。

1. **旧構成を更新する前に**、現在の Compose ファイル・`.env`・TOML とホスト DB をバックアップします。
   現在使用中の Compose プロジェクト名と起動オプションも記録してください。
2. 旧 Caddy の実際のボリューム名を確認します。既存の固定コンテナ名なら次のコマンドです。

   ```bash
   docker inspect peercast-0yp-caddy --format '{{range .Mounts}}{{println .Destination .Name}}{{end}}'
   ```

   `/data`、`/config` の名前を新しい `.env` の `CADDY_DATA_VOLUME`、`CADDY_CONFIG_VOLUME` に指定します。
   新しい名前のままなら新規証明書取得になります。既存ボリュームの再利用時に Compose の所有プロジェクトの警告が出ることがあります。
   ホスト上で変更していた静的ファイルがあれば、`/srv/yayaue.me/public/` へ反映してください。
3. 新構成の `.env` と TOML を準備し、`config -q` とビルドを済ませます。
   固定の `172.19.0.0/16` は廃止するため、旧サブネットに限定した DB 接続元許可やファイアウォール設定を確認します。
   必要なら `docker compose create peercast-0yp` で起動前にネットワークを作成し、
   `docker network ls` / `docker network inspect <実際のネットワーク名>` で割当を確認して、必要な範囲だけ許可してください。
   ネットワーク再作成時にも再確認が必要です。
4. **更新前の旧 Compose** で停止します（元のプロジェクト名・オプションを使用）。

   ```bash
   cd /srv/peercast-0yp
   docker compose -f docker-compose.yml -f docker-compose.prod.yml down
   ```

   `-v` は付けません。Caddy 停止後に証明書ボリュームもバックアップしてください。
   先にファイルを更新してしまった場合は、バックアップした旧構成を元の配置・プロジェクト名で使って停止してください。
   新しい app のみの構成での `down` では、旧 Caddy が残る場合があります。
5. `/srv/yayaue.me` で前述の Caddy 検証と `docker compose up -d` を実行し、配信を確認します。
   旧・新 Caddy から同じ証明書ボリュームへ同時に書き込まないでください。
6. 問題があれば新構成を `docker compose down`（`-v` なし）で停止し、旧ファイルを復元して旧構成を起動します。
   この移行自体は DB スキーマを変更しません。

## 更新・停止・ログ

以下は `/srv/yayaue.me` で実行します。

```bash
# 各リポジトリの変更を取得（ローカル変更がある場合は先に確認）
git pull --ff-only
git -C ../peercast-0yp pull --ff-only
git -C ../peercast-mi pull --ff-only
# PEERCAST_0YP_DIR を変更した場合は git -C のパスも合わせる
docker compose config -q
docker compose build peercast-0yp peercast-mi
docker compose up -d

# Caddy イメージの更新時
docker compose pull caddy
docker compose up -d caddy

# 停止（証明書ボリュームとホスト DB を残す）
docker compose down

# ログ・状態
docker compose logs -f --tail=100 caddy peercast-0yp
docker compose ps
```

`.env` の変更は `up -d`、0yp の TOML の変更は `restart peercast-0yp` で反映します。
`public/` の変更はマウント経由で反映されます。
Caddy の `/data` と `/config` は named volume に永続化されます。通常の停止に `down -v` やボリューム削除を使わないでください。
`.env`、秘密鍵、バックアップは Git 管理対象外です。`docker compose config` の通常出力にはパスワードが含まれるため、共有用の検証には `-q` を使います。

## Caddy 設定の再読み込み

`docker/caddy/Caddyfile` を編集して、次を実行します。ディレクトリをマウントしているため、エディタがファイルを置換しても変更が見えます。

```bash
docker compose exec caddy caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile --adapter caddyfile
```

検証に失敗したら修正してから再読み込みします。`.env` のドメイン変更は再読み込みだけでは反映されないため `docker compose up -d caddy` が必要です。
更新手順の `up -d` だけでは Caddyfile の変更を読み直さない場合があるため、設定変更時は上の手順も実施してください。

## 別アプリの追加

`compose.yaml` にサービスを追加し、Caddyfile にサブドメインまたはパスの `reverse_proxy サービス名:内部HTTPポート` を追加します。
同じ Compose のデフォルトネットワークで名前解決できるため、追加アプリの HTTP ポートをホストへ公開する必要はありません。
パス振り分けを使う場合は既存の静的ファイル用 `handle` より前にルートを追加し、アプリがそのパスに対応することを確認してください。
必要な秘密情報だけを各サービスへ渡し、永続データにはアプリごとの volume を用意します。
`peercast-mi` の設定・導入手順は [Ansible README](ansible/README.md#peercast-mi-の追加) を参照してください。

## Ansible での配置・更新

小さなデプロイ用 Playbook を [ansible/](ansible/README.md) に用意しています。
初回の手動移行が完了した後、設定配置・ビルド・起動・Caddy 再読み込みを自動化できます。
本番で確認した既存0ypの配置先 `/opt/peercast-0yp` を維持する例も記載しています。
