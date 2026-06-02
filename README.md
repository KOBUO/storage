# pipeline-template

LPJ向けの GitLab CI パイプラインテンプレートです。

Java / React プロジェクト向けに、ビルド・テスト・カバレッジ取得・SonarQube 解析・Docker Build・ECR Push・Nexus Publish などの CI/CD 処理を共通化することを目的としています。

各プロジェクトでは、本リポジトリのテンプレートを `include` して利用します。

主に以下を共通化します。

- Gradle build / test / coverage（JaCoCo）
- React build / test / coverage（Vitest）
- SonarQube 解析
- Docker build
- ECR Push
- Nexus Maven / npm Publish
- Git タグ末尾の `-snapshot` による Snapshot / Release publish 先の自動切り替え

---

## ファイル構成

| ファイル | 内容 |
|---|---|
| `templates/common-pipeline.yml` | Docker / DinD / Docker build / ECR Push などの共通定義 |
| `templates/java-pipeline.yml` | Java / Gradle 向け build / test / sonar / Nexus publish / ECR push 定義 |
| `templates/react-pipeline.yml` | React / pnpm 向け build / test / sonar / Nexus publish / ECR push 定義 |

---

## 利用イメージ

### アプリ開発（ECR Push）

ビルド・テスト・SonarQube 解析・Docker Build・ECR Push を行う場合のイメージです。

タグ作成時に ECR へイメージが Push されます。

#### Java

```yaml
include:
  - project: 'lpj/common/template/pipeline-template'
    ref: main
    file: '/templates/common-pipeline.yml'
  - project: 'lpj/common/template/pipeline-template'
    ref: main
    file: '/templates/java-pipeline.yml'

stages:
  - build
  - test
  - sonar
  - docker-build
  - ecr-push

java-docker-build:
  extends: .java-docker-build-base

java-ecr-push:
  extends: .java-ecr-push-base

variables:
  ECR_REPOSITORY: my-app
  SONAR_PROJECT_KEY: my-app
```

#### React

```yaml
include:
  - project: 'lpj/common/template/pipeline-template'
    ref: main
    file: '/templates/common-pipeline.yml'
  - project: 'lpj/common/template/pipeline-template'
    ref: main
    file: '/templates/react-pipeline.yml'

stages:
  - build
  - test
  - sonar
  - docker-build
  - ecr-push

react-docker-build:
  extends: .react-docker-build-base

react-ecr-push:
  extends: .react-ecr-push-base

variables:
  ECR_REPOSITORY: my-app
  SONAR_PROJECT_KEY: my-app
```

---

### 共通部品（Nexus Publish）

ビルド・テスト・SonarQube 解析・Nexus Publish を行う場合のイメージです。

Docker build は不要なため、`docker-build` / `ecr-push` ステージは含めません。

タグ作成時に Nexus へ成果物が Publish されます。

#### Java（Maven JAR）

```yaml
include:
  - project: 'lpj/common/template/pipeline-template'
    ref: main
    file: '/templates/common-pipeline.yml'
  - project: 'lpj/common/template/pipeline-template'
    ref: main
    file: '/templates/java-pipeline.yml'

stages:
  - build
  - test
  - sonar
  - nexus-push

java-nexus-publish:
  extends: .java-nexus-publish-base

variables:
  SONAR_PROJECT_KEY: my-lib
```

#### React（npm パッケージ）

```yaml
include:
  - project: 'lpj/common/template/pipeline-template'
    ref: main
    file: '/templates/common-pipeline.yml'
  - project: 'lpj/common/template/pipeline-template'
    ref: main
    file: '/templates/react-pipeline.yml'

stages:
  - build
  - test
  - sonar
  - nexus-push

react-nexus-publish:
  extends: .react-nexus-publish-base

variables:
  SONAR_PROJECT_KEY: my-lib
```

---

## タグルール

本テンプレートでは、タグ名により Release / Snapshot を判定します。

タグ実行時、`pre:pipeline` ジョブでタグ名を解析し、後続ジョブで利用する環境変数を `build.env` に出力します。

### Release タグ

Release 用のタグ形式は以下です。

```text
release-1.0.0
release-v1.0.0
```

Release タグの場合、以下の値が生成されます。

| 変数 | 値 |
|---|---|
| `TAG_TYPE` | `release` |
| `VERSION` | `release-` を除去した値 |
| `PUBLISH_VERSION` | `VERSION` と同じ値 |
| `MAVEN_PUBLISH_URL` | `MAVEN_PUBLISH_RELEASE_URL` |
| `NPM_PUBLISH_URL` | `NPM_PUBLISH_RELEASE_URL` |
| `NPM_TAG` | `latest` |

### Snapshot タグ

Snapshot 用のタグ形式は以下です。

```text
snapshot-1.2.3-build1
snapshot-v1.2.3-build1
snapshot-1.2.3-systest-build3
snapshot-v1.2.3-systest-build3
```

Snapshot タグの場合、以下の値が生成されます。

| 変数 | 値 |
|---|---|
| `TAG_TYPE` | `snapshot` |
| `VERSION` | `snapshot-` を除去した値 |
| `PUBLISH_VERSION` | `${VERSION}-snapshot` |
| `MAVEN_PUBLISH_URL` | `MAVEN_PUBLISH_SNAPSHOT_URL` |
| `NPM_PUBLISH_URL` | `NPM_PUBLISH_SNAPSHOT_URL` |
| `NPM_TAG` | `snapshot` |

### タグ例

| Git タグ | TAG_TYPE | VERSION | PUBLISH_VERSION | npm dist-tag |
|---|---|---|---|---|
| `release-1.0.0` | `release` | `1.0.0` | `1.0.0` | `latest` |
| `release-v1.0.0` | `release` | `v1.0.0` | `v1.0.0` | `latest` |
| `snapshot-1.2.3-build1` | `snapshot` | `1.2.3-build1` | `1.2.3-build1-snapshot` | `snapshot` |
| `snapshot-v1.2.3-build1` | `snapshot` | `v1.2.3-build1` | `v1.2.3-build1-snapshot` | `snapshot` |
| `snapshot-1.2.3-systest-build3` | `snapshot` | `1.2.3-systest-build3` | `1.2.3-systest-build3-snapshot` | `snapshot` |
| `snapshot-v1.2.3-systest-build3` | `snapshot` | `v1.2.3-systest-build3` | `v1.2.3-systest-build3-snapshot` | `snapshot` |

### 不正なタグ

上記形式以外のタグは `pre:pipeline` でエラーになります。

```text
1.0.0
v1.0.0
1.0.0-snapshot
release_1.0.0
snapshot_1.0.0
```

---

## common-pipeline.yml

`common-pipeline.yml` は Java / React 共通の CI/CD 処理を定義します。

### 共通変数

| 変数名 | デフォルト値 | 説明 |
|---|---|---|
| `APP_DIR` | `.` | アプリケーション配置ディレクトリ |
| `DOCKER_REGISTRY` | `${BS72_DOCKER_PROXY_REGISTRY}` | Docker Proxy Registry |
| `NEXUS_USER` | `${BS72_NEXUS_USER}` | Nexus ユーザー |
| `NEXUS_PASSWORD` | `${BS72_NEXUS_PASSWORD}` | Nexus パスワード |
| `MAVEN_REPO_URL` | `${BS72_MAVEN_CENTRAL_URL}` | Maven 依存取得用リポジトリ |
| `MAVEN_PUBLISH_RELEASE_URL` | `${BS72_MAVEN_RELEASE_URL}` | Maven Release Publish 先 |
| `MAVEN_PUBLISH_SNAPSHOT_URL` | `${BS72_MAVEN_SNAPSHOT_URL}` | Maven Snapshot Publish 先 |
| `NPM_REGISTRY_URL` | `${BS72_NPM_CENTRAL_URL}` | npm 依存取得用リポジトリ |
| `NPM_PUBLISH_RELEASE_URL` | `${BS72_NPM_HOSTED_RELEASE_URL}` | npm Release Publish 先 |
| `NPM_PUBLISH_SNAPSHOT_URL` | `${BS72_NPM_HOSTED_SNAPSHOT_URL}` | npm Snapshot Publish 先 |
| `DOCKERFILE_PATH` | `""` | 独自 Dockerfile を利用する場合のパス |

### pre:pipeline

`pre:pipeline` は、タグ名を解析し、後続ジョブで利用する環境変数を生成します。

生成した値は `build.env` に出力され、dotenv artifact として後続ジョブへ引き継がれます。

```yaml
artifacts:
  reports:
    dotenv: build.env
  expire_in: 6 months
```

生成される主な変数は以下です。

| 変数 | 説明 |
|---|---|
| `TAG_TYPE` | `release` または `snapshot` |
| `VERSION` | Docker tag / ECR tag に利用する値 |
| `PUBLISH_VERSION` | Nexus Publish に利用する値 |
| `MAVEN_PUBLISH_URL` | Maven Publish 先 URL |
| `NPM_PUBLISH_URL` | npm Publish 先 URL |
| `NPM_TAG` | npm dist-tag |

### .docker-base

Docker / DinD を利用するための共通設定です。

| 項目 | 値 |
|---|---|
| image | `docker:29-cli` |
| service | `docker:29-dind` |
| `DOCKER_DRIVER` | `overlay2` |
| `DOCKER_BUILDKIT` | `1` |

### .docker-build-base

Docker image を build する共通処理です。

`DOCKERFILE_PATH` が指定されている場合は、その Dockerfile を利用します。

`DOCKERFILE_PATH` が未指定の場合は、対象テンプレートに応じた Dockerfile を自動生成します。

### .ecr-push-base

ECR に Docker image を Push する共通処理です。

タグ実行時のみ実行されます。

Push 先は以下です。

```text
${ECR_REGISTRY}/${ECR_REPOSITORY}:${VERSION}
```

---

## Dockerfile 自動生成

`DOCKERFILE_PATH` が未指定の場合、テンプレート側で Dockerfile を自動生成します。

### Java 用 Dockerfile

```dockerfile
FROM amazoncorretto:25-alpine
COPY build/libs/*.jar app.jar
ENTRYPOINT ["java","-jar","app.jar"]
```

### React 用 Dockerfile

```dockerfile
FROM nginx:1.30.1-alpine
COPY dist/ /usr/share/nginx/html/
COPY nginx.conf /etc/nginx/nginx.conf
```

### 独自 Dockerfile を利用する場合

```yaml
variables:
  DOCKERFILE_PATH: "docker/Dockerfile"
```

---

## java-pipeline.yml

`java-pipeline.yml` は Java / Gradle プロジェクト向けのテンプレートです。

### .java-base

Java 用の共通ベースジョブです。

| 項目 | 値 |
|---|---|
| image | `${DOCKER_REGISTRY}/gradle:jdk25-corretto` |
| tags | `lpj-runner` |
| `GRADLE_USER_HOME` | `${CI_PROJECT_DIR}/.gradle` |
| `GRADLE_OPTS` | `-Dorg.gradle.daemon=false -Dorg.gradle.parallel=true` |

キャッシュ対象は Gradle Wrapper のみです。

```yaml
cache:
  paths:
    - .gradle/wrapper/
```

### java-build

Java プロジェクトをビルドします。

```bash
gradle -p "$APP_DIR" build -x test --stacktrace --refresh-dependencies
```

成果物は以下です。

```text
build/libs
build/classes
```

保存期間は `1 hour` です。

### java-test

Java のテストと JaCoCo レポート出力を行います。

```bash
gradle -p "$APP_DIR" test jacocoTestReport
```

タグ実行時はスキップされます。

出力対象は以下です。

```text
build/test-results/test/*.xml
build/reports/jacoco/test/jacocoTestReport.xml
```

保存期間は `1 week` です。

### java-sonar

SonarQube 解析を行います。

```bash
gradle -p "$APP_DIR" sonar
```

利用する主な変数は以下です。

| 変数 | 説明 |
|---|---|
| `SONAR_HOST_URL` | SonarQube URL |
| `SONAR_TOKEN` | SonarQube Token |
| `SONAR_PROJECT_KEY` | SonarQube Project Key |

`allow_failure: true` のため、SonarQube 解析に失敗してもパイプライン全体は失敗扱いになりません。

タグ実行時はスキップされます。

### java-nexus-publish

Java の成果物を Maven 形式で Nexus に Publish します。

タグ実行時のみ実行されます。

CI 内で `ci-publish.gradle` を動的生成し、`PUBLISH_VERSION` を全 `MavenPublication` に適用します。

```bash
gradle -p "$APP_DIR" publish -I ci-publish.gradle
```

Publish 先は `pre:pipeline` で生成された `MAVEN_PUBLISH_URL` を利用します。

認証には以下を利用します。

| 変数 | 説明 |
|---|---|
| `NEXUS_USER` | Nexus ユーザー |
| `NEXUS_PASSWORD` | Nexus パスワード |

利用側プロジェクトでは `publishing.publications` を定義してください。

```kotlin
publishing {
    publications {
        create<MavenPublication>("mavenJar") {
            from(components["java"])

            versionMapping {
                usage("java-api") {
                    fromResolutionOf("runtimeClasspath")
                }
                usage("java-runtime") {
                    fromResolutionResult()
                }
            }
        }
    }
}
```

`repositories` の設定は CI 側で注入されるため、利用側プロジェクトで定義する必要はありません。

### java-docker-build

Java アプリケーションの Docker image を build します。

Dockerfile の指定がない場合は、Java 用テンプレート Dockerfile が自動生成されます。

### java-ecr-push

Java アプリケーションの Docker image を ECR に Push します。

タグ実行時のみ実行されます。

Push 先は以下です。

```text
${ECR_REGISTRY}/${ECR_REPOSITORY}:${VERSION}
```

---

## react-pipeline.yml

`react-pipeline.yml` は React / pnpm プロジェクト向けのテンプレートです。

### .react-base

React 用の共通ベースジョブです。

| 項目 | 値 |
|---|---|
| image | `${DOCKER_REGISTRY}/node:lts-alpine3.23` |
| tags | `lpj-runner` |
| `PNPM_HOME` | `${CI_PROJECT_DIR}/.pnpm-store` |

キャッシュ対象は以下です。

```text
.pnpm-store/
```

### npm 認証

ジョブ内で `.npmrc` を動的生成します。

利用する主な変数は以下です。

| 変数 | 説明 |
|---|---|
| `NEXUS_USER` | Nexus ユーザー |
| `NEXUS_PASSWORD` | Nexus パスワード |
| `NPM_REGISTRY_URL` | npm 依存取得用リポジトリ |

設定内容は以下です。

```text
registry=<NPM_REGISTRY_URL>
always-auth=true
```

### react-build

React プロジェクトをビルドします。

```bash
pnpm --dir "$APP_DIR" run build
```

成果物は以下です。

```text
dist/
```

保存期間は `1 hour` です。

### react-test

React のテストとカバレッジレポート出力を行います。

```bash
pnpm --dir "$APP_DIR" run test:ci
```

タグ実行時はスキップされます。

出力対象は以下です。

```text
junit.xml
coverage/cobertura-coverage.xml
```

Coverage 形式は `cobertura` です。

保存期間は `1 week` です。

`package.json` には `test:ci` を定義してください。

```json
{
  "scripts": {
    "test:ci": "vitest run --coverage"
  }
}
```

### react-sonar

SonarQube 解析を行います。

| 項目 | 値 |
|---|---|
| image | `sonarsource/sonar-scanner-cli:12.1.0.3225_8.0.1` |

実行コマンドは以下です。

```bash
sonar-scanner
```

利用する主な変数は以下です。

| 変数 | 説明 |
|---|---|
| `SONAR_HOST_URL` | SonarQube URL |
| `SONAR_TOKEN` | SonarQube Token |
| `SONAR_PROJECT_KEY` | SonarQube Project Key |

`allow_failure: true` のため、SonarQube 解析に失敗してもパイプライン全体は失敗扱いになりません。

タグ実行時はスキップされます。

### react-nexus-publish

React の成果物を npm パッケージとして Nexus に Publish します。

タグ実行時のみ実行されます。

`package.json` の version を `PUBLISH_VERSION` に更新します。

```bash
pnpm version "${PUBLISH_VERSION}" --no-git-tag-version
```

Nexus に Publish します。

```bash
pnpm publish \
  --no-git-checks \
  --registry "${NPM_PUBLISH_URL}/" \
  --tag "${NPM_TAG}"
```

Publish 先は `pre:pipeline` で生成された `NPM_PUBLISH_URL` を利用します。

npm dist-tag は以下です。

| TAG_TYPE | NPM_TAG |
|---|---|
| `release` | `latest` |
| `snapshot` | `snapshot` |

### react-docker-build

React アプリケーションの Docker image を build します。

Dockerfile の指定がない場合は、React 用テンプレート Dockerfile が自動生成されます。

### react-ecr-push

React アプリケーションの Docker image を ECR に Push します。

タグ実行時のみ実行されます。

Push 先は以下です。

```text
${ECR_REGISTRY}/${ECR_REPOSITORY}:${VERSION}
```

---

## ECR Push について

ECR Push はタグ作成時のみ実行されます。

```bash
git tag release-1.0.0
git push origin release-1.0.0
```

Push されるイメージは以下の形式です。

```bash
${ECR_REGISTRY}/${ECR_REPOSITORY}:${VERSION}
```

`ECR_REGISTRY` は `AWS_ACCOUNT_ID` と `AWS_REGION` から自動導出されます。

---

## Nexus へライブラリを Publish したい

タグを作成すると `java-nexus-publish` または `react-nexus-publish` が実行されます。

Release / Snapshot はタグ名により自動判定されます。

### Java

Java は `build.gradle.kts` に `publications {}` ブロックが必要です。

`repositories {}` はパイプライン側で自動注入するため不要です。

```kotlin
publishing {
    publications {
        create<MavenPublication>("mavenJar") {
            from(components["java"])

            versionMapping {
                usage("java-api") {
                    fromResolutionOf("runtimeClasspath")
                }
                usage("java-runtime") {
                    fromResolutionResult()
                }
            }
        }
    }
}
```

### React

React は `package.json` の `name` が npm パッケージ名として利用されます。

publish バージョンは Git タグから生成した `PUBLISH_VERSION` を使用します。

---

## CI 変数

### テンプレートグローバル変数

`common-pipeline.yml` で定義されているグローバル変数です。

プロジェクトの `.gitlab-ci.yml` の `variables` で同名の変数を定義することで上書きできます。

| 変数名 | デフォルト値 | 説明 |
|---|---|---|
| `APP_DIR` | `.` | アプリのサブディレクトリ |
| `DOCKER_REGISTRY` | `${BS72_DOCKER_PROXY_REGISTRY}` | Docker イメージの pull 元レジストリ |
| `NEXUS_USER` | `${BS72_NEXUS_USER}` | Nexus 認証ユーザー名 |
| `NEXUS_PASSWORD` | `${BS72_NEXUS_PASSWORD}` | Nexus 認証パスワード |
| `MAVEN_REPO_URL` | `${BS72_MAVEN_CENTRAL_URL}` | 依存解決に使う Maven リポジトリ |
| `MAVEN_PUBLISH_RELEASE_URL` | `${BS72_MAVEN_RELEASE_URL}` | Maven アーティファクトの Release publish 先 |
| `MAVEN_PUBLISH_SNAPSHOT_URL` | `${BS72_MAVEN_SNAPSHOT_URL}` | Maven アーティファクトの Snapshot publish 先 |
| `NPM_REGISTRY_URL` | `${BS72_NPM_CENTRAL_URL}` | パッケージインストールに使う npm レジストリ |
| `NPM_PUBLISH_RELEASE_URL` | `${BS72_NPM_HOSTED_RELEASE_URL}` | npm パッケージの Release publish 先 |
| `NPM_PUBLISH_SNAPSHOT_URL` | `${BS72_NPM_HOSTED_SNAPSHOT_URL}` | npm パッケージの Snapshot publish 先 |
| `DOCKERFILE_PATH` | `""` | カスタム Dockerfile のパス。未指定時はテンプレートの Dockerfile を使用 |

### CI/CD Settings 変数

各ジョブが GitLab CI/CD Settings から直接参照している変数です。

| 変数名 | 説明 |
|---|---|
| `AWS_ACCESS_KEY_ID` | AWS アクセスキー ID |
| `AWS_SECRET_ACCESS_KEY` | AWS シークレットアクセスキー |
| `AWS_ACCOUNT_ID` | AWS アカウント ID |
| `AWS_REGION` | AWS リージョン |
| `SONAR_TOKEN` | SonarQube 認証トークン |
| `BS72_DOCKER_PROXY_REGISTRY` | Nexus Docker Proxy Registry |
| `BS72_NEXUS_USER` | Nexus 認証ユーザー名 |
| `BS72_NEXUS_PASSWORD` | Nexus 認証パスワード |
| `BS72_MAVEN_RELEASE_URL` | Maven Release Repository URL |
| `BS72_MAVEN_SNAPSHOT_URL` | Maven Snapshot Repository URL |
| `BS72_NPM_HOSTED_RELEASE_URL` | npm Release Repository URL |
| `BS72_NPM_HOSTED_SNAPSHOT_URL` | npm Snapshot Repository URL |

### プロジェクト変数

`.gitlab-ci.yml` の `variables` に設定します。

| 変数名 | 説明 |
|---|---|
| `ECR_REPOSITORY` | ECR リポジトリ名。ECR Push 時のみ必要 |
| `SONAR_PROJECT_KEY` | SonarQube プロジェクトキー |
| `DOCKERFILE_PATH` | カスタム Dockerfile のパス。未指定時はテンプレートのデフォルトを使用 |
| `APP_DIR` | アプリのサブディレクトリ。未指定時は `.` |

---

## よくある質問

### カスタム Dockerfile を使いたい

デフォルトでは Java / React それぞれのテンプレート Dockerfile が使用されます。

プロジェクト独自の Dockerfile を使う場合は `DOCKERFILE_PATH` を指定してください。

```yaml
variables:
  DOCKERFILE_PATH: "docker/Dockerfile"
```

### サブディレクトリのプロジェクトを対象にしたい

`APP_DIR` を指定してください。

```yaml
variables:
  APP_DIR: "sample/java"
```

### Nexus Publish 先を直接指定したい

通常はタグ名により Snapshot / Release の publish 先が自動的に切り替わります。

そのため、通常はプロジェクト側で `MAVEN_PUBLISH_URL` や `NPM_PUBLISH_URL` を直接指定する必要はありません。
