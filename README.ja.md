# zammad-poc

[Zammad](https://zammad.org) ヘルプデスクを Docker Compose でローカルに立ち上げ、
**任意の単一ネットワークインターフェース**(VPN の IP・LAN の IP・localhost)にだけ
公開する評価用構成です。`0.0.0.0` には公開しません。外部公開なし・TLS なし・
OAuth アプリ登録なし。本番導入前のお試し用です。

English version: [README.md](README.md)

## 仕組み

このリポジトリは[公式 `zammad-docker-compose`](https://github.com/zammad/zammad-docker-compose)
の薄いオーバーレイです。upstream は `setup.sh` が clone するだけで、
一切書き換えず・同梱もしません(`git pull` でそのまま追従できます)。

オーバーレイがやることは 3 つだけ:

- **Web UI を単一インターフェースにバインド。** `docker-compose.override.yml` が
  upstream の `0.0.0.0:8080` 公開を `<BIND_IP>:<ポート>` に差し替えます
  (`ports: !override`、[公式推奨](https://docs.zammad.org/en/latest/install/docker-compose.html)のカスタマイズ方法)。
  その IP に到達できる相手(VPN ピア、LAN 内ホスト、`127.0.0.1` なら自分だけ)しか届きません
- **初回起動時の設定をシード。** `ZAMMAD_FQDN` / `ZAMMAD_HTTP_TYPE` を実際に
  ブラウザで開く URL に合わせます(非標準ポート・平文 HTTP でも cookie /
  WebSocket / 生成リンクが正しく動く)
- **同梱バックアップサービスの無効化**(upstream 自身の
  `scenarios/disable-backup-service.yml` を利用)

それ以外 — PostgreSQL / Redis / memcached / Elasticsearch / DB マイグレーションを行う
init コンテナ — はすべて素の upstream です。

## 必要なもの

- Docker + Compose **v2.24.4 以上**(`ports: !override` に必要)
- バインド先の IP アドレス: VPN インターフェース(WireGuard、Tailscale など)、
  LAN の IP、または `127.0.0.1`
- 空きメモリ 4GB 程度(Elasticsearch 込み)、`vm.max_map_count >= 262144`

## クイックスタート

```bash
git clone https://github.com/hideki5123/zammad-poc.git
cd zammad-poc
BIND_IP=192.168.1.20 ZAMMAD_HOST=192.168.1.20 ./setup.sh   # 公開したいIPとブラウザで使うホスト名
docker compose up -d    # 初回は DB マイグレーションで 1〜2 分かかります
```

あとはその IP に到達できるデバイスから `http://<ZAMMAD_HOST>:8081` を開き、
セットアップウィザードを進めるだけです(管理者作成にメール検証は不要、
メールチャネルのステップはスキップ可)。

補足: [Tailscale](https://tailscale.com/) が稼働している環境で変数を省略すると、
`setup.sh` が `BIND_IP` に Tailscale の IPv4、`ZAMMAD_HOST` に MagicDNS 名を
自動設定します。これは単なる自動検出で、スタック自体は Tailscale に依存しません。

## 設定

`setup.sh` が `.env` を生成します(コミットされません)。

| 変数 | 既定値 | 用途 |
|---|---|---|
| `BIND_IP` | Tailscale があればその IPv4、なければ入力 | UI を公開する**唯一の**インターフェース |
| `ZAMMAD_UI_PORT` | `8081` | Web UI のホスト側ポート |
| `ZAMMAD_FQDN` | `ZAMMAD_HOST` + ポート | **初回起動時のみ**シード。ブラウザで打つホスト名と一致させる |
| `ZAMMAD_HTTP_TYPE` | `http` | `http` のまま推奨 — 下記ハマりどころ参照 |
| `POSTGRES_PASS` | ランダム | DB パスワード、**初回起動時のみ**有効 |
| `TZ` | ホストのタイムゾーン | コンテナのタイムゾーン |

upstream の変数(バージョン固定、Elasticsearch のヒープなど)もそのまま使えます:
[`.env.dist`](https://github.com/zammad/zammad-docker-compose/blob/master/.env.dist) 参照。

## 運用コマンド

```bash
docker compose logs -f zammad-init   # 初回起動が進まないときはまずここ
docker compose down                  # 停止(データは named volume に残る)
docker compose down -v               # DB ごと完全リセット
git -C zammad-docker-compose pull && docker compose pull && docker compose up -d   # アップグレード
```

## ハマりどころ

- **平文 HTTP のまま `http_type` を `https` にしない。** セッション cookie が
  Secure 扱いになりブラウザが保存せず、全ログインが
  「CSRF token verification failed」で失敗します。やってしまったら:
  `docker compose exec zammad-railsserver bundle exec rails r "Setting.set('http_type', 'http')"`
  のあと rails サーバーを再起動
- **`ZAMMAD_FQDN` / `ZAMMAD_HTTP_TYPE` は初回起動時のみ有効**(DB シード)。
  あとから変えるときは上記の `Setting.set(...)` で
- **`POSTGRES_PASS` も初回起動時のみ有効** — postgres volume は初期化時の
  認証情報を保持します。変えるなら `docker compose down -v` から
- **FQDN にポートが入った状態でのメール送信は upstream の既知バグ**
  ([zammad#5127](https://github.com/zammad/zammad/issues/5127)):通知送信者が
  `noreply@host:8081` という不正なアドレスになります。SMTP を設定する前に
  *チャネル → メール → 設定 → Notification Sender* をポート無しアドレスに変更
- WebSocket は素で動きます(同梱 nginx が `/ws` と `/cable` を同一オリジンで
  プロキシし、シード済み FQDN が ActionCable のオリジンチェックを通す)

## スコープ

これは**評価用**であり、意図的に本番品質にしていません:平文 HTTP
(信頼できるネットワークにのみバインド)、バックアップ無効、実メール配送なし。
本番では TLS 終端のリバースプロキシを前段に置き、`http_type` を `https` に、
バックアップサービスを再有効化した上で
[公式ドキュメント](https://docs.zammad.org/en/latest/install/docker-compose.html)に従ってください。

## ライセンス

このリポジトリのファイルは [MIT](LICENSE) です。Zammad 本体および
`zammad-docker-compose`(セットアップ時に clone、再配布はしていません)は
[AGPL-3.0](https://github.com/zammad/zammad/blob/develop/LICENSE) です。
