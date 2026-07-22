# パターン1: 電文受付① 通常API（njs カナリアリリース）

> 独立資料。この1枚で「何を・どう作るか」が分かるようにまとめる。
> 最優先資料: 方式設計書 4-2-2 / 4-2-7 / 4-2-8 / 4-2-10 / 4-2-13。PoC: `reference/poc-source/`。
> テンプレート: `templates/01-denbun-api.conf.template`

---

## 1. これは何か（目的）

POS/SC からの通常API電文を受け付ける Nginx。**店舗×バージョンで転送先(backend)を動的に切り替える
カナリアリリース**を行い、あわせて流量制御・タイムアウト・エラー応答を担う。

## 2. どういう仕組みか（図）

```
                              ECS タスク（電文受付①）
  POS/SC ──▶ ①ELB ──▶ ┌──────────────────────────────────────────┐
                       │  nginx①                                   │
                       │   1) リクエスト受信                          │
                       │   2) njs(route.js) が振分けキーを取得         │
                       │        ★方式: BODYの店舗コード (4-2-2)        │
                       │        （PoC: X-Shop-Id ヘッダー）           │
                       │   3) URIから context_path / version 抽出     │
                       │   4) routes.js(=DBキャッシュ) を参照して判定   │
                       │        stores → defaults → DEFAULT の順      │
                       │   5) $upstream を決定 → proxy_pass           │
                       │                                            │
                       │  loader(サイドカー, 起動時ワンショット)         │
                       │   DB(Aurora) → routing.json 出力             │
                       └───────────┬──────────────┬────────────────┘
                                   │              │
                          ④ELB ─▶ backend(default)  backend(shopXXX_vN)
                                   （通常のAPIアプリ / カナリア版）

  ※ DB変更の反映 = ECSタスク再起動（起動時に routing.json → routes.js を作り直す）
```

### データの流れ（起動時 → リクエスト時）

```
[起動時]  Aurora(routing/default_routing)
            │ loader.py が SELECT
            ▼
          /shared/routing.json   ──(entrypoint がラップ)──▶  /etc/nginx/njs/routes.js
                                                              （routingTable + upstreamMap）
[リクエスト時]  route.js が routes.js を参照して $upstream を決定
```

## 3. どういうファイルが必要か

| ファイル | 役割 | 状態 |
|---------|------|------|
| `nginx.conf`(template) | njs読込・流量制御・proxy・404固定JSON | 雛形: `templates/01-denbun-api.conf.template` |
| `njs/route.js` | 振分けロジック本体（`pick_upstream`）| PoC復元: `reference/poc-source/nginx/route.js`。★BODY対応へ改修要 |
| `njs/routes.js` | DBキャッシュ(config)。**起動時に自動生成** | entrypoint が生成 |
| `entrypoint.sh` | routing.json → routes.js 生成 → nginx起動 | PoC: `reference/.../entrypoint.sh` |
| `loader.py` | Aurora → routing.json 出力（サイドカー）| PoC: `reference/.../loader.py` |
| `Dockerfile`(nginx) | nginx + njs モジュール | PoC: 1.27-alpine（商用版は要確定）|
| `Dockerfile`(loader) | python + psycopg2 | PoC: python:3.12-slim |
| ECSタスク定義 | nginx + loader(essential=false, dependsOn SUCCESS) | PoC: `reference/.../task-def/` |
| DBテーブル | `routing` / `default_routing` | ★テーブル定義書と要突合 |

## 4. どういう設定が必要か

### Nginx ディレクティブ
| 設定 | 用途 | 方式根拠 |
|------|------|---------|
| `load_module ngx_http_js_module` / `js_import` / `js_set $upstream` | njs有効化・振分け | PoC 方式B |
| `worker_processes auto` / `worker_connections`(512基本) | 自コンテナ負荷制御 | 4-2-7 |
| `limit_conn` / `limit_conn_zone $server_name` | 同時接続制御（**limit_reqは方式外**）| 4-2-7 |
| `proxy_connect_timeout` / `proxy_read_timeout` | アイドルTO→504 | 4-2-8 |
| `proxy_pass $upstream` | 動的転送 | 4-2-10 |
| 未定義パス `return 404 '{...}'` | 404固定JSON | JIKIPH0-2288 |

### 環境変数（例）
`SERVER_NAME` / `WORKER_CONNECTIONS`(=512) / `LIMIT_CONN` / `LIMIT_CONN_STATUS` /
`PROXY_CONNECT_TIMEOUT` / `PROXY_READ_TIMEOUT` / `DENBUN_PATH` /
（loader用）`DB_HOST` `DB_PORT` `DB_NAME` `DB_USER` `DB_PASSWORD`(→Secrets) `ROUTING_JSON_PATH` /
（entrypoint用）`DEFAULT_UPSTREAM` `SHOP0002_V2_UPSTREAM`…(=upstreamMap、拡張性課題)

## 5. TODO（このパターン固有）

- [ ] **★相違A: 振分けキーを「BODYの店舗コード」へ**（njsでbody読取り設計）。4-2-2
- [ ] **対象テーブル定義**（routing/default_routing 正式スキーマ）。テーブル定義書と突合
- [ ] upstreamMap の拡張性（envハードコード列挙 → DB化/動的生成）
- [ ] 未定義コンテキストパスの404固定JSON（無限ループ回避）。JIKIPH0-2288
- [ ] DB認証を Secrets Manager 化（PoCは平文）
- [ ] アイドルTO値の決定。4-2-8
- [ ] ログJSON（subtype=NGINX）フォーマット。4-2-13（後続定義）
- [ ] nginx/njs 商用バージョン。JIKIPH0-267
