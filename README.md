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
- Git タグ未尾の `-snapshot` による Snapshot / Release publish 先の自動切り替え

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


### Java プロジェクト

```yaml
include:
  - project: '<テンプレートプロジェクト>'
    ref: main
    file: '/templates/common-pipeline.yml'
  - project: '<テンプレートプロジェクト>'
    ref: main
    file: '/templates/java-pipeline.yml'
```

### React プロジェクト

```yaml
include:
  - project: '<テンプレートプロジェクト>'
    ref: main
    file: '/templates/common-pipeline.yml'
  - project: '<テンプレートプロジェクト>'
    ref: main
    file: '/templates/react-pipeline.yml'
```

---

## Snapshot / Release の切り替えについて

本テンプレートでは、Git タグ名により Release / Snapshot を判定します。

タグ作成時に `pre:pipeline` ジョブでタグ名を解析し、後続ジョブで利用する環境変数を `build.env` に出力します。

### Release タグ

Release は以下の形式です。

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

Snapshot は以下の形式です。

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

| Git Tag | TAG_TYPE | VERSION | PUBLISH_VERSION | npm dist-tag |
|---|---|---|---|---|
| `release-1.0.0` | `release` | `1.0.0` | `1.0.0` | `latest` |
| `release-v1.0.0` | `release` | `v1.0.0` | `v1.0.0` | `latest` |
| `snapshot-1.2.3-build1` | `snapshot` | `1.2.3-build1` | `1.2.3-build1-snapshot` | `snapshot` |
| `snapshot-v1.2.3-build1` | `snapshot` | `v1.2.3-build1` | `v1.2.3-build1-snapshot` | `snapshot` |
| `snapshot-1.2.3-systest-build3` | `snapshot` | `1.2.3-systest-build3` | `1.2.3-systest-build3-snapshot` | `snapshot` |
| `snapshot-v1.2.3-systest-build3` | `snapshot` | `v1.2.3-systest-build3` | `v1.2.3-systest-build3-snapshot` | `snapshot` |

### 不正なタグ

以下のようなタグは `pre:pipeline` でエラーになります。

```text
1.0.0
v1.0.0
1.0.0-snapshot
release_1.0.0
snapshot_1.0.0
```

---

## common-pipeline.yml

Java / React 共通の CI/CD 処理を定義します。

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

タグ名を解析し、後続ジョブで利用する環境変数を `build.env` に出力します。

`build.env` は dotenv artifact として後続ジョブへ引き継がれます。

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

Java / Gradle プロジェクト向けのテンプレートです。

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

タグ実行時はスキップされます。

```bash
gradle -p "$APP_DIR" test jacocoTestReport
```

出力対象は以下です。

```text
build/test-results/test/*.xml
build/reports/jacoco/test/jacocoTestReport.xml
```

保存期間は `1 week` です。

### java-sonar

SonarQube 解析を行います。

タグ実行時はスキップされます。

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

利用側プロジェクトでは `publications` を定義してください。

```kotlin
publishing {
    publications {
        create<MavenPublication>("mavenJar") {
            from(components["java"])
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

React / pnpm プロジェクト向けのテンプレートです。

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

タグ実行時はスキップされます。

```bash
pnpm --dir "$APP_DIR" run test:ci
```

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

タグ実行時はスキップされます。

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

## ECR Push

ECR Push はタグ実行時のみ行われます。

Push 先は以下です。

```text
${ECR_REGISTRY}/${ECR_REPOSITORY}:${VERSION}
```

### Release の例

```bash
git tag release-1.0.0
git push origin release-1.0.0
```

Push されるイメージ例です。

```text
${ECR_REGISTRY}/${ECR_REPOSITORY}:1.0.0
```

### Snapshot の例

```bash
git tag snapshot-1.2.3-build1
git push origin snapshot-1.2.3-build1
```

Push されるイメージ例です。

```text
${ECR_REGISTRY}/${ECR_REPOSITORY}:1.2.3-build1
```

---

## Nexus Publish

Nexus Publish はタグ実行時のみ行われます。

Release / Snapshot はタグ名により自動判定されます。

### Java Maven Publish

利用側プロジェクトでは `publishing.publications` を定義してください。

```kotlin
publishing {
    publications {
        create<MavenPublication>("mavenJar") {
            from(components["java"])
        }
    }
}
```

Repository は CI 側で生成する `ci-publish.gradle` により設定されます。

### React npm Publish

`package.json` の `name` が npm パッケージ名として利用されます。

タグ実行時に、CI 側で `package.json` の version を `PUBLISH_VERSION` に更新して Publish します。

Release の場合は `latest`、Snapshot の場合は `snapshot` の dist-tag が付与されます。

---

## 必要な CI/CD 変数

GitLab の CI/CD Variables に必要な変数を設定してください。

### 共通

| 変数 | 説明 |
|---|---|
| `BS72_DOCKER_PROXY_REGISTRY` | Docker Proxy Registry |
| `BS72_NEXUS_USER` | Nexus ユーザー |
| `BS72_NEXUS_PASSWORD` | Nexus パスワード |
| `SONAR_HOST_URL` | SonarQube URL |
| `SONAR_TOKEN` | SonarQube Token |
| `SONAR_PROJECT_KEY` | SonarQube Project Key |

### Maven / Java

| 変数 | 説明 |
|---|---|
| `BS72_MAVEN_CENTRAL_URL` | Maven 依存取得用 Nexus Proxy |
| `BS72_MAVEN_RELEASE_URL` | Maven Release Publish 先 |
| `BS72_MAVEN_SNAPSHOT_URL` | Maven Snapshot Publish 先 |

### npm / React

| 変数 | 説明 |
|---|---|
| `BS72_NPM_CENTRAL_URL` | npm 依存取得用 Nexus Proxy |
| `BS72_NPM_HOSTED_RELEASE_URL` | npm Release Publish 先 |
| `BS72_NPM_HOSTED_SNAPSHOT_URL` | npm Snapshot Publish 先 |

### AWS / ECR

| 変数 | 説明 |
|---|---|
| `AWS_DEFAULT_REGION` | AWS リージョン |
| `AWS_ROLE_ARN` | AssumeRole 用 Role ARN |
| `ECR_REGISTRY` | ECR Registry |
| `ECR_REGISTRY_ID` | ECR Registry ID |
| `ECR_REPOSITORY` | ECR Repository |

### プロジェクトごとの任意変数

| 変数 | 説明 |
|---|---|
| `APP_DIR` | アプリ配置ディレクトリ。省略時は `.` |
| `DOCKERFILE_PATH` | 独自 Dockerfile パス。省略時はテンプレート Dockerfile を自動生成 |

---

## よくある質問

### カスタム Dockerfile を使いたい

`DOCKERFILE_PATH` を指定してください。

```yaml
variables:
  DOCKERFILE_PATH: "docker/Dockerfile"
```

### サブディレクトリのプロジェクトを対象にしたい

`APP_DIR` を指定してください。

```yaml
variables:
  APP_DIR: "apps/sample-java"
```

### Nexus Publish 先を直接指定したい

Release / Snapshot の Publish 先は、以下の変数で指定できます。

```yaml
variables:
  MAVEN_PUBLISH_RELEASE_URL: "https://example.com/repository/maven-releases/"
  MAVEN_PUBLISH_SNAPSHOT_URL: "https://example.com/repository/maven-snapshots/"
  NPM_PUBLISH_RELEASE_URL: "https://example.com/repository/npm-releases/"
  NPM_PUBLISH_SNAPSHOT_URL: "https://example.com/repository/npm-snapshots/"
```

### タグなしの通常コミットでは何が実行されるか

通常コミットでは、build / test / sonar が実行されます。

Nexus Publish / ECR Push はタグ実行時のみ実行されます。

### タグ実行時に test / sonar は実行されるか

Java / React ともに、タグ実行時は test / sonar はスキップされます。

タグ実行時は、Nexus Publish / Docker Build / ECR Push を行います。

### Release / Snapshot はどこで切り替わるか

`pre:pipeline` で `CI_COMMIT_TAG` を解析し、Release / Snapshot を判定します。

後続ジョブは `pre:pipeline` が出力した以下の変数を利用します。

```text
TAG_TYPE
VERSION
PUBLISH_VERSION
MAVEN_PUBLISH_URL
NPM_PUBLISH_URL
NPM_TAG
```
