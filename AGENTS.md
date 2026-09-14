# このリポジトリで作業するエージェントへ

## 目的と参照先

このリポジトリは yayaue.me VPS 共通のCompose、Caddy、静的トップページ、Ansibleを管理する。
変更前に関連する設定と [本番構成・運用記録](docs/production.md) を読む。
一般的な導入手順は [README.md](README.md)、Playbookの使い方は [ansible/README.md](ansible/README.md) を参照する。
本番記録は記載日時点の観測であり、ローカル設定や過去の出力だけから現在の本番状態を断定しない。
ユーザーの明示的な指示を優先し、以下はその指示がない場合の既定方針とする。

## 構成を維持するための前提

- 本番配置は `/opt/yayaue.me` と `/opt/peercast-0yp`。READMEの `/srv` は新規配置の例。
- 本番のCaddyと0ypはこのリポジトリのComposeで起動する。0yp側のComposeは単独開発・app単体用。
- HTTP `/yp/index.txt` の直接配信、その他のHTTPのHTTPS転送、HTTPSの `/yp`・`/yp/*` のプロキシ、その他の静的配信を維持する。プロキシで `/yp` を削らない。
- 0yp内部のHTTPは80、PCPは7144。待受ポートは0ypのTOMLに由来する。PCP 7144/TCPは直接公開し、HTTPはCaddy経由とする。
- 本番DBはホストMariaDB。認証情報は本番の `yayaue.me/.env` からCompose経由で渡す。0ypは環境変数を読み、旧0ypの `.env` は新構成では読み込まない。
- 証明書は旧Caddyのnamed volumeを引き継いでいる。実際のvolume名は `.env` で指定する。警告を消す目的だけでvolumeを新規作成・削除しない。
- 固定コンテナ名・固定サブネットは未使用。ネットワーク再作成時はホストDB向けUFW許可との整合を確認する。観測したIPを普遍的な既定値として埋め込まない。
- `peercast-mi` は `/mi`・`/mi/*` をサイト専用8080へパスを保持して転送する。PCPは7154を直接公開。RTMP 1945は直接公開し、発行済みストリームキーで認証する。ユーザー指定によりRTMPS・SSHトンネルは使用しない。
- miの設定は `docker/peercast-mi/config.toml`、永続データは配置先の `data/peercast-mi`（UID/GID 10001）。配信キー・broadcast_idを上書き・削除しない。

## Ansibleと運用上の注意

- `prepare.yml` は手元のファイルを配置し、`deploy.yml` は配置・検証・ビルド・起動・Caddy再読み込みを行う。
- アプリソースのGit更新、0ypのTOML・フロントエンドビルド、OS・Docker導入、DB、UFWは現在のPlaybook管理外。範囲を変更したら運用ドキュメントも更新する。
- `public/` のファイルを追加・削除する際は `ansible/prepare.yml` のコピー対象と本番の削除方法も確認する。コピーだけでは古いファイルは消えない。
- Ansibleは手元のファイルを直接コピーする。本番GitのHEADだけでは配信内容を特定できない。
- `--check` は現在、ファイル変更の予測と読み取りによる前提確認まで。Compose/Caddyの実検証やアプリの健全性確認済みとは報告しない。
- 変更のない通常更新ではdownを挟まない。`down -v`、volume削除、DB復元を通常のデプロイ・復旧に含めない。
- ローカルの編集・検証と、本番への適用を区別する。本番操作が依頼されている場合はその範囲で進め、単なるファイル編集依頼から本番デプロイを推測しない。
- ユーザーにコマンドを案内する際は「手元のWSL」「VPS」の実行場所を明記し、確認結果を受けて次の手順へ進む。

## 秘密情報と検証

`.env`、inventoryの実ファイル、SSH秘密鍵、DBダンプ、証明書バックアップをGitやログに含めない。
認証情報を必要としない検証にはダミー値を使う。実際のCompose設定確認は原則 `config -q` を使い、JSON出力が必要なら必要なフィールドだけ抽出して表示する。

Compose変更時のローカル構成検証例（起動はしない）:

```bash
SITE_DOMAIN=example.test DB_USER=validation DB_PASSWORD=validation-only DB_NAME=validation PEERCAST_X_CLIENT_ID=validation PEERCAST_X_CLIENT_SECRET=validation \
  docker compose --env-file .env.example config -q
```

Ansible変更時は依存環境を準備したうえで、接続先を含まないサンプルinventoryで構文を検証する:

```bash
export ANSIBLE_COLLECTIONS_PATH="$PWD/ansible/collections"
ansible/.venv/bin/ansible-playbook -i ansible/inventory.example.yml ansible/prepare.yml --syntax-check
ansible/.venv/bin/ansible-playbook -i ansible/inventory.example.yml ansible/deploy.yml --syntax-check
```

Caddy変更時は設定検証を実施し、既存ルーティングが維持されていることを確認する。
文書だけの変更ではリンク・コードブロック・`git diff --check` を確認し、本番操作やアプリテストを追加しない。
検証していない項目は明記する。環境変更や移行で新たに確認した事実は、秘密情報を除いて `docs/production.md` に日付とともに残す。
