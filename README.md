# GitLab CI/CD 共通パイプラインテンプレート

※ 本ファイルは画像解析結果を元にした整理版です。

## タグルール

### Release

```text
release-1.0.0
release-v1.0.0
```

### Snapshot

```text
snapshot-1.2.3-build1
snapshot-v1.2.3-build1
snapshot-v1.2.3-systest-build3
```

## pre:pipeline

タグ解析を行い build.env を生成します。

後続ジョブは dotenv artifact 経由で利用します。

## build.env

|変数|説明|
|---|---|
|TAG_TYPE|release/snapshot|
|VERSION|タグから生成|
|PUBLISH_VERSION|publish用バージョン|
|MAVEN_PUBLISH_URL|publish先|
|NPM_PUBLISH_URL|publish先|
|NPM_TAG|latest/snapshot|

## Dockerfile自動生成

DOCKERFILE_PATH 未指定時はテンプレート側で Dockerfile を生成します。
