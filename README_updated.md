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

variables:
  APP_DIR: "."
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

variables:
  APP_DIR: "."
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

variables:
  APP_DIR: "."
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

variables:
  APP_DIR: "."
  SONAR_PROJECT_KEY: my-lib
```

---

## Snapshot / Release の切り替えについて

タグ名の末尾が `-snapshot` の場合、Nexus publish 先は Snapshot 用 URL に切り替わります。

タグ名の末尾が `-snapshot` ではない場合、Nexus publish 先は Release 用 URL に切り替わります。

| Git タグ | 対象 | publish / push 先 | version / tag |
|---|---|---|---|
| `1.0.0-snapshot` | Java Nexus | `${MAVEN_PUBLISH_SNAPSHOT_URL}` | `1.0.0-SNAPSHOT` |
| `1.0.0-snapshot` | React Nexus | `${NPM_PUBLISH_SNAPSHOT_URL}` | `1.0.0-snapshot` |
| `1.0.0-snapshot` | ECR | `${ECR_REGISTRY}/${ECR_REPOSITORY}` | `1.0.0-snapshot` |
| `1.0.0` | Java Nexus | `${MAVEN_PUBLISH_RELEASE_URL}` | `1.0.0` |
| `1.0.0` | React Nexus | `${NPM_PUBLISH_RELEASE_URL}` | `1.0.0` |
| `1.0.0` | ECR | `${ECR_REGISTRY}/${ECR_REPOSITORY}` | `1.0.0` |

Java の Maven publish では、Snapshot バージョンは Maven 標準に合わせて `1.0.0-SNAPSHOT` に変換されます。

React の npm publish では、`1.0.0-snapshot` のまま publish されます。  
また、npm の dist-tag は以下のように設定されます。

| Git タグ | npm version | npm dist-tag |
|---|---|---|
| `1.0.0-snapshot` | `1.0.0-snapshot` | `snapshot` |
| `1.0.0` | `1.0.0` | `latest` |

---

## 機能説明

### `templates/common-pipeline.yml`

Docker / DinD / ECR Push などの共通定義を行うテンプレートです。

| ジョブ名 | 内容 |
|---|---|
| `.docker-base` | Docker / DinD 共通設定 |
| `.docker-build-base` | docker-build ステージ共通定義（Nexus ログイン含む） |
| `.ecr-push-base` | ecr-push ステージ共通定義（ECR ログイン含む） |

### `templates/java-pipeline.yml`

Java / Gradle 向け build / test / coverage / sonar / Nexus publish 定義を行うテンプレートです。

| ジョブ名 | 内容 |
|---|---|
| `java-build` | Gradle build を実行 |
| `java-test` | Gradle test + JaCoCo カバレッジを生成 |
| `java-sonar` | SonarQube 解析を実行（`allow_failure: true`） |
| `java-docker-build` | Docker image を build |
| `java-ecr-push` | Docker image を build し ECR へ Push（タグ時のみ） |
| `java-nexus-publish` | Maven JAR を Nexus へ Publish（タグ時のみ） |

#### `java-build`

Gradle build を実行します。

```bash
gradle -p "$APP_DIR" build -x test --stacktrace
```

#### `java-test`

Gradle test を実行し、JaCoCo カバレッジレポートを生成します。

```bash
gradle -p "$APP_DIR" test jacocoTestReport
```

#### `java-sonar`

SonarQube 解析を実行します。失敗してもパイプラインは継続します。

```bash
gradle -p "$APP_DIR" sonar
```

### `templates/react-pipeline.yml`

React / pnpm 向け build / test / coverage / sonar / Nexus publish 定義を行うテンプレートです。

| ジョブ名 | 内容 |
|---|---|
| `react-build` | pnpm build を実行 |
| `react-test` | pnpm test:ci を実行（Vitest） |
| `react-sonar` | SonarQube 解析を実行（`allow_failure: true`） |
| `react-docker-build` | Docker image を build |
| `react-ecr-push` | Docker image を build し ECR へ Push（タグ時のみ） |
| `react-nexus-publish` | npm パッケージを Nexus へ Publish（タグ時のみ） |

#### `react-build`

React build を実行します。

```bash
pnpm --dir "$APP_DIR" run build
```

#### `react-test`

Vitest を利用してテストを実行し、カバレッジレポートを生成します。

```bash
pnpm --dir "$APP_DIR" run test:ci
```

`package.json` に以下のスクリプトを定義してください。

```json
{
  "scripts": {
    "test:ci": "vitest run --coverage"
  }
}
```

#### `react-sonar`

SonarQube 解析を実行します。失敗してもパイプラインは継続します。  
`APP_DIR` に `sonar-project.properties` を配置してください。

---

## ECR Push について

ECR Push はタグ作成時のみ実行されます。

```bash
git tag v1.0.0
git push origin v1.0.0
```

Push されるイメージは以下の形式です。

```bash
${ECR_REGISTRY}/${ECR_REPOSITORY}:${CI_COMMIT_TAG}
```

`ECR_REGISTRY` は `AWS_ACCOUNT_ID` と `AWS_REGION` から自動導出されます。

タグが `1.0.0-snapshot` の場合、ECR タグも `1.0.0-snapshot` のまま Push されます。

---

## Nexus へライブラリを Publish したい

タグを作成すると `java-nexus-publish` または `react-nexus-publish` が実行されます。

タグ名の末尾が `-snapshot` の場合は Snapshot 用 Repository に publish されます。  
タグ名の末尾が `-snapshot` ではない場合は Release 用 Repository に publish されます。

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

Java の publish バージョンは以下のように扱われます。

| Git タグ | publish 先 | Maven version |
|---|---|---|
| `1.0.0-snapshot` | Snapshot Repository | `1.0.0-SNAPSHOT` |
| `1.0.0` | Release Repository | `1.0.0` |

### React

React は `package.json` の `name` が npm パッケージ名として利用されます。

publish バージョンは Git タグをそのまま使用します。

| Git タグ | publish 先 | npm version | npm dist-tag |
|---|---|---|---|
| `1.0.0-snapshot` | Snapshot Repository | `1.0.0-snapshot` | `snapshot` |
| `1.0.0` | Release Repository | `1.0.0` | `latest` |

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
| `MAVEN_REPO_URL` | `https://bs72-repo.devops.aslead.cloud/repository/maven-central` | 依存解決に使う Maven リポジトリ |
| `MAVEN_PUBLISH_RELEASE_URL` | `${BS72_MAVEN_RELEASE_URL}` | Maven アーティファクトの Release publish 先 |
| `MAVEN_PUBLISH_SNAPSHOT_URL` | `${BS72_MAVEN_SNAPSHOT_URL}` | Maven アーティファクトの Snapshot publish 先 |
| `NPM_REGISTRY_URL` | `https://bs72-repo.devops.aslead.cloud/repository/npm-central` | パッケージインストールに使う npm レジストリ |
| `NPM_PUBLISH_RELEASE_URL` | `${BS72_NPM_HOSTED_RELEASE_URL}` | npm パッケージの Release publish 先 |
| `NPM_PUBLISH_SNAPSHOT_URL` | `${BS72_NPM_HOSTED_SNAPSHOT_URL}` | npm パッケージの Snapshot publish 先 |
| `SONAR_HOST_URL` | `https://bs72-sonar.devops.aslead.cloud` | SonarQube サーバー URL |
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

通常は `CI_COMMIT_TAG` の末尾が `-snapshot` かどうかで、Snapshot / Release の publish 先が自動的に切り替わります。

そのため、通常はプロジェクト側で `MAVEN_PUBLISH_URL` や `NPM_PUBLISH_URL` を直接指定する必要はありません。
