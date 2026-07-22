# パターン3: 本部画面用 React（静的配信 + S3中継）

> 独立資料。最優先資料: 方式設計書 4-3-4(p81) / 4-3-6 / 4-3-9(p92) / 4-3-14 / 4-3-16 / 4-3-17 / 4-3-19。
> テンプレート: `templates/03-react.conf.template`

---

## 1. これは何か（目的）

本部画面（React SPA）を配信する Web サーバ Nginx。**S3 をバックに静的コンテンツを配信**し、
ファイルの**UL/DL は S3 へ中継**する。認証は ALB(OIDC/Entra ID) が担うため **nginx は認証しない**。

## 2. どういう仕組みか（図）

```
                        LZDC > サブシステムA
  LAW端末 ──(SSL-VPN/DX)──▶ ①ALB ──▶ ┌────────────────────────────┐
            (ブラウザ)      (OIDC認証)  │  ECS: Webサーバ nginx        │
                                       │   ├ 静的配信: S3をproxy       │──▶ S3(静的コンテンツ)
                                       │   │   未定義URL→index.html     │
                                       │   │   (200, 404にしない)       │
                                       │   └ UL/DL中継: VPC GW EP経由    │──▶ S3(UL/DLファイル)
                                       └───────────┬────────────────┘
                                                   │ (API呼出はALB→別ECS)
                                          ④ELB ─▶ API(Spring) ─▶ Aurora / ElastiCache

  認証(4-3-14): 未認証→ALBがIdP(Entra ID)へリダイレクト→認可コード→IDトークン(JWT)
                →認証セッションID→静的コンテンツ取得（nginxは認証済みリクエストを配信するだけ）
  閉塞(4-3-19): 初回起動＋操作時に API経由でAuroraの閉塞状態を問合せ（nginx直接でない）
```

### UL/DL の仕組み（4-3-16 / 4-3-17）

```
React ──▶ API(Spring): 格納先/取得元URLを発行（ホスト名を nginx に置換して返す）
React ──▶ nginx(そのURL): ──VPC Gateway EP──▶ S3 へUL中継 / S3からDL中継
  ※ UL物理名 = システム時刻+UUID（上書き禁止）、論理名↔物理名はDB管理
  ※ UL と業務データ登録は別トランザクション
```

## 3. どういうファイルが必要か

| ファイル | 役割 | 状態 |
|---------|------|------|
| `nginx.conf`(template) | 静的配信・SPA fallback・UL/DL中継 | 雛形: `templates/03-react.conf.template` |
| `entrypoint.sh` | envsubst で conf 生成 → nginx起動 | 要作成 |
| `Dockerfile`(nginx) | nginx（njs不要）| 要作成 |
| 静的ファイル(index.html等) | S3配置 or コンテナ同梱 | ★配置方式 要確定 |
| ECSタスク定義 | nginx単体 | 要作成 |
| VPC Gateway エンドポイント | UL/DL の S3 到達 | インフラ側 |

→ **njs も loader も DB も不要**。「アプリごとに微修正で済む」共通ベースにする（Confluence）。

## 4. どういう設定が必要か

### Nginx ディレクティブ
| 設定 | 用途 | 方式根拠 |
|------|------|---------|
| `location /` → S3配信（proxy or root）| 静的コンテンツ | 4-3-4 |
| `error_page 403 404 = @spa_fallback` → `index.html` | **SPA fallback（200, 404にしない）** | 4-3-9(1-5) p92 |
| `location /files/` → `proxy_pass https://${S3_ENDPOINT}` | UL/DL中継 | 4-3-16/17 |
| `client_max_body_size` | UL用サイズ上限 | 4-3-16 |
| `limit_rate` / `limit_rate_after` | DL帯域制御 | 4-3-17 |
| `limit_conn` / `limit_conn_zone` | S3同時接続制限 | 4-3-6 |
| （認証設定なし）| ALBが認証 | 4-3-14 |

### 環境変数（例）
`SERVER_NAME` / `WORKER_CONNECTIONS` / `S3_ENDPOINT` / `FILE_RELAY_PATH`(=/files/) /
`MAX_UPLOAD_SIZE` / `LIMIT_RATE` / `LIMIT_RATE_AFTER` / `LIMIT_CONN` /（同梱時）`REACT_ROOT`

## 5. TODO（このパターン固有）

- [ ] **★相違F: SPA fallback**（未定義URL→index.html 200、404にしない）。4-3-9 p92
- [ ] 静的配信方式の確定（S3をproxy_pass or S3同期してroot）。4-3-4
- [ ] UL/DL中継（VPC GW EP + ホスト名置換）。4-3-16/17
- [ ] キャッシュ制御（index.html no-cache / ハッシュ資産 長期）
- [ ] 「アプリごとに微修正で済む」テンプレート粒度。Confluence
- [ ] React で参照すべき資料をアプリチームに確認
