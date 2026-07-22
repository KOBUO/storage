# パターン2: 電文受付② ファイルダウンロード（S3中継）

> 独立資料。最優先資料: 方式設計書 4-2-18(p70) / 4-2-2 / 4-2-10、画面系DL 4-3-17。
> テンプレート: `templates/02-filedl.conf.template`

---

## 1. これは何か（目的）

POS 等が配信ファイルをDLするための Nginx。**S3 を裏で中継**し、利用者からは Nginx ドメインだけを見せる。
署名付きURL方式（署名は API 側で発行）で、Nginx はホスト名を S3 に置換して中継＋下り帯域制御を行う。
**カナリアや流量制御(業務)は配信制御側で実施済のため、ここでは行わない**（4-2-2）。

## 2. どういう仕組みか（図）

```
[事前] 配信計画取得API が「署名付きURL」を発行（ホスト名は nginx を指す）
        例: https://<nginxドメイン>/download/<バケット>/<パス>?<署名>

                              ECS タスク（電文受付②）
  POS ──▶ ②ELB ──▶ ┌────────────────────────────────────┐
   (署名付きURL)     │  nginx②                             │
                    │   1) /download/... を受信             │
                    │   2) Host を S3ドメインへ置換          │
                    │   3) ?署名クエリはそのまま保持          │
                    │   4) 下り帯域制御(limit_rate)          │
                    │   5) proxy_pass（VPC Gateway EP経由）  │
                    └───────────────┬────────────────────┘
                                    ▼
                         S3（VPC Gateway エンドポイント経由）
                    https://<S3ドメイン>/<バケット>/<パス>?<署名>

  ※ オフラインDL: 固定URL /download/offline(仮)/固定ファイル名?署名&posVersion=2.0.0
  ※ nginx は署名しない（中継のみ）。S3の4xx/5xxはそのまま返す（4-2-10）。
```

## 3. どういうファイルが必要か

| ファイル | 役割 | 状態 |
|---------|------|------|
| `nginx.conf`(template) | S3中継・ホスト名置換・帯域制御 | 雛形: `templates/02-filedl.conf.template` |
| `entrypoint.sh` | envsubst で conf 生成 → nginx起動 | 要作成（njs不要でシンプル）|
| `Dockerfile`(nginx) | nginx（**njs不要**）| 要作成 |
| ECSタスク定義 | nginx単体（loaderサイドカー不要）| 要作成 |
| VPC Gateway エンドポイント | VPC内→S3到達 | インフラ側設定 |

→ **njs も loader も DB も不要**。パターン1よりシンプル。

## 4. どういう設定が必要か

### Nginx ディレクティブ
| 設定 | 用途 | 方式根拠 |
|------|------|---------|
| `location /download/` → `proxy_pass https://${S3_ENDPOINT}` | S3中継 | 4-2-18 |
| `proxy_set_header Host ${S3_ENDPOINT}` | ホスト名置換 | 4-2-18 |
| `$request_uri` 透過 | ?署名クエリ保持 | 4-2-18 |
| `limit_rate` / `limit_rate_after` | 下り帯域制御 | 4-2-2 |
| `limit_conn` / `limit_conn_zone` | 同時接続制御 | 4-2-7 |
| `proxy_intercept_errors off` | S3エラーを素通し | 4-2-10 |
| `proxy_ssl_server_name on` | SNI（S3向けTLS）| - |

### 環境変数（例）
`SERVER_NAME` / `WORKER_CONNECTIONS` / `FILE_DL_PATH`(=/download/) /
`S3_ENDPOINT`(S3ドメイン/VPC EP) / `LIMIT_RATE` / `LIMIT_RATE_AFTER` / `LIMIT_CONN` / `LIMIT_CONN_STATUS`

## 5. TODO（このパターン固有）

- [ ] **★相違E: 署名はnginxでなくAPI発行**。ホスト名置換の実装（proxy_pass + Host）。4-2-18
- [ ] `?署名`クエリの透過検証（プロキシで署名が壊れないこと）。4-2-18
- [ ] `/download/` からバケット/キーの取り出し（rewrite要否）。4-2-18
- [ ] VPC Gateway EP のルーティング / バケットポリシー / resolver要否。4-3-17
- [ ] 帯域制御値・「帯域制御ヘッダ」仕様。4-2-2
- [ ] `download` コンテキストパスで本パターンの nginx へ振り分ける方法。4-2-2
- [ ] ログJSON（subtype=NGINX）。4-2-13
