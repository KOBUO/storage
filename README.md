# GitLab CI/CD 共通パイプラインテンプレート

Java / React プロジェクトで利用する GitLab CI/CD の共通テンプレートです。

各プロジェクトの `.gitlab-ci.yml` から本テンプレートを `include` することで、以下の処理を共通化できます。

- ビルド
- テスト
- カバレッジレポート出力
- SonarQube 解析
- Docker イメージビルド
- ECR Push
- Nexus Publish

---

## 1. テンプレート構成

```text
templates/
├── common-pipeline.yml
├── java-pipeline.yml
└── react-pipeline.yml
```

| ファイル | 内容 |
|---|---|
| `templates/common-pipeline.yml` | Java / React 共通処理。タグ解析、Docker / DinD、Docker Build、ECR Push 共通処理を定義 |
| `templates/java-pipeline.yml` | Java / Gradle 用の build、test、sonar、Nexus Publish、Docker Build、ECR Push を定義 |
| `templates/react-pipeline.yml` | React / pnpm 用の build、test、sonar、Nexus Publish、Docker Build、ECR Push を定義 |

---

## 2. 利用方法

各プロジェクトの `.gitlab-ci.yml` で、必要なテンプレートを include します。

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

## 3. タグルール

本テンプレートでは、タグ名により Release / Snapshot を判定します。

タグ実行時、`pre:pipeline` ジョブでタグ名を解析し、後続ジョブで利用する環境変数を `build.env` に出力します。

### 3.1 Release タグ

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
| `PUBLISH_VERSION` | `VERSION` と同じ |
| `MAVEN_PUBLISH_URL` | `MAVEN_PUBLISH_RELEASE_URL` |
| `NPM_PUBLISH_URL` | `NPM_PUBLISH_RELEASE_URL` |
| `NPM_TAG` | `latest` |

### 3.2 Snapshot タグ

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

### 3.3 タグ例

| Gitタグ | TAG_TYPE | VERSION | PUBLISH_VERSION | npm dist-tag |
|---|---|---|---|---|
| `release-1.0.0` | `release` | `1.0.0` | `1.0.0` | `latest` |
| `release-v1.0.0` | `release` | `v1.0.0` | `v1.0.0` | `latest` |
| `snapshot-1.2.3-build1` | `snapshot` | `1.2.3-build1` | `1.2.3-build1-snapshot` | `snapshot` |
| `snapshot-v1.2.3-build1` | `snapshot` | `v1.2.3-build1` | `v1.2.3-build1-snapshot` | `snapshot` |
| `snapshot-1.2.3-systest-build3` | `snapshot` | `1.2.3-systest-build3` | `1.2.3-systest-build3-snapshot` | `snapshot` |

### 3.4 不正なタグ

上記形式以外のタグは `pre:pipeline` でエラーになります。

```text
1.0.0
v1.0.0
1.0.0-snapshot
release_1.0.0
snapshot_1.0.0
```

---

## 4. common-pipeline.yml

`common-pipeline.yml` は Java / React 共通の CI/CD 処理を定義します。

### 4.1 共通変数

| 変数 | デフォルト値 | 説明 |
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

### 4.2 pre:pipeline

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
| `MAVEN_PUBLISH_URL` | Maven Publish 先URL |
| `NPM_PUBLISH_URL` | npm Publish 先URL |
| `NPM_TAG` | npm dist-tag |

### 4.3 .docker-base

Docker / DinD を利用するための共通設定です。

| 項目 | 値 |
|---|---|
| image | `docker:29-cli` |
| service | `docker:29-dind` |
| DOCKER_DRIVER | `overlay2` |
| DOCKER_BUILDKIT | `1` |

### 4.4 .docker-build-base

Docker image を build する共通処理です。

`DOCKERFILE_PATH` が指定されている場合は、その Dockerfile を利用します。

`DOCKERFILE_PATH` が未指定の場合は、プロジェクト種別に応じてテンプレート Dockerfile を自動生成します。

### 4.5 .ecr-push-base

ECR に Docker image を Push する共通処理です。

タグ実行時のみ実行されます。

Push 先イメージは以下です。

```text
${ECR_REGISTRY}/${ECR_REPOSITORY}:${VERSION}
```

---

## 5. Dockerfile 自動生成

`DOCKERFILE_PATH` が未指定の場合、テンプレート側で Dockerfile を自動生成します。

### 5.1 Java 用 Dockerfile

```dockerfile
FROM amazoncorretto:25-alpine
COPY build/libs/*.jar app.jar
ENTRYPOINT ["java","-jar","app.jar"]
```

### 5.2 React 用 Dockerfile

```dockerfile
FROM nginx:1.30.1-alpine
COPY dist/
COPY nginx.conf
```

### 5.3 独自 Dockerfile を利用する場合

利用側プロジェクトで `DOCKERFILE_PATH` を指定します。

```yaml
variables:
  DOCKERFILE_PATH: "docker/Dockerfile"
```

---

## 6. java-pipeline.yml

`java-pipeline.yml` は Java / Gradle プロジェクト向けのテンプレートです。

### 6.1 .java-base

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

### 6.2 java-build

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

### 6.3 java-test

Java のテストと JaCoCo レポート出力を行います。

```bash
gradle -p "$APP_DIR" test jacocoTestReport
```

タグ実行時はスキップされます。

```yaml
rules:
  - if: '$CI_COMMIT_TAG'
    when: never
```

出力対象は以下です。

```text
build/test-results/test/*.xml
build/reports/jacoco/test/jacocoTestReport.xml
```

保存期間は `1 week` です。

### 6.4 java-sonar

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

### 6.5 java-nexus-publish

Java の成果物を Maven 形式で Nexus に Publish します。

タグ実行時のみ実行されます。

`ci-publish.gradle` を CI 内で動的生成し、`PUBLISH_VERSION` を全 `MavenPublication` に適用します。

```bash
gradle -p "$APP_DIR" publish -I ci-publish.gradle
```

Publish 先は `pre:pipeline` で生成された `MAVEN_PUBLISH_URL` を利用します。

認証には以下を利用します。

| 変数 | 説明 |
|---|---|
| `NEXUS_USER` | Nexus ユーザー |
| `NEXUS_PASSWORD` | Nexus パスワード |

### 6.6 java-docker-build

Java アプリケーションの Docker image を build します。

内部的には `.java-docker-build-base` を利用します。

Dockerfile の指定がない場合は、Java 用テンプレート Dockerfile が自動生成されます。

### 6.7 java-ecr-push

Java アプリケーションの Docker image を ECR に Push します。

内部的には `.java-ecr-push-base` を利用します。

タグ実行時のみ実行されます。

---

## 7. react-pipeline.yml

`react-pipeline.yml` は React / pnpm プロジェクト向けのテンプレートです。

### 7.1 .react-base

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

### 7.2 npm 認証

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

### 7.3 react-build

React プロジェクトをビルドします。

```bash
pnpm --dir "$APP_DIR" run build
```

成果物は以下です。

```text
dist/
```

保存期間は `1 hour` です。

### 7.4 react-test

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

### 7.5 react-sonar

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

### 7.6 react-nexus-publish

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

### 7.7 react-docker-build

React アプリケーションの Docker image を build します。

内部的には `.react-docker-build-base` を利用します。

Dockerfile の指定がない場合は、React 用テンプレート Dockerfile が自動生成されます。

### 7.8 react-ecr-push

React アプリケーションの Docker image を ECR に Push します。

内部的には `.react-ecr-push-base` を利用します。

タグ実行時のみ実行されます。

---

## 8. ECR Push

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

## 9. Nexus Publish

Nexus Publish はタグ実行時のみ行われます。

Release / Snapshot はタグ名により自動判定されます。

### 9.1 Java Maven Publish

利用側プロジェクトでは `publishing` の `publications` を定義してください。

Repository は CI 側で生成する `ci-publish.gradle` により設定されます。

```kotlin
publishing {
    publications {
        create<MavenPublication>("mavenJar") {
            from(components["java"])
        }
    }
}
```

`repositories` の設定はテンプレート側で注入されるため、利用側プロジェクトで定義しなくても Publish できます。

### 9.2 React npm Publish

`package.json` の `name` が npm パッケージ名として利用されます。

タグ実行時に、CI 側で `package.json` の version を `PUBLISH_VERSION` に更新して Publish します。

Release の場合は `latest`、Snapshot の場合は `snapshot` の dist-tag が付与されます。

---

## 10. 必要な CI/CD 変数

GitLab の CI/CD Variables に必要な変数を設定してください。

### 10.1 共通

| 変数 | 説明 |
|---|---|
| `BS72_DOCKER_PROXY_REGISTRY` | Docker Proxy Registry |
| `BS72_NEXUS_USER` | Nexus ユーザー |
| `BS72_NEXUS_PASSWORD` | Nexus パスワード |
| `SONAR_HOST_URL` | SonarQube URL |
| `SONAR_TOKEN` | SonarQube Token |
| `SONAR_PROJECT_KEY` | SonarQube Project Key |

### 10.2 Maven / Java

| 変数 | 説明 |
|---|---|
| `BS72_MAVEN_CENTRAL_URL` | Maven 依存取得用 Nexus Proxy |
| `BS72_MAVEN_RELEASE_URL` | Maven Release Publish 先 |
| `BS72_MAVEN_SNAPSHOT_URL` | Maven Snapshot Publish 先 |

### 10.3 npm / React

| 変数 | 説明 |
|---|---|
| `BS72_NPM_CENTRAL_URL` | npm 依存取得用 Nexus Proxy |
| `BS72_NPM_HOSTED_RELEASE_URL` | npm Release Publish 先 |
| `BS72_NPM_HOSTED_SNAPSHOT_URL` | npm Snapshot Publish 先 |

### 10.4 AWS / ECR

| 変数 | 説明 |
|---|---|
| `AWS_DEFAULT_REGION` | AWS リージョン |
| `AWS_ROLE_ARN` | AssumeRole 用 Role ARN |
| `ECR_REGISTRY` | ECR Registry |
| `ECR_REGISTRY_ID` | ECR Registry ID |
| `ECR_REPOSITORY` | ECR Repository |

### 10.5 プロジェクトごとの任意変数

| 変数 | 説明 |
|---|---|
| `APP_DIR` | アプリ配置ディレクトリ。省略時は `.` |
| `DOCKERFILE_PATH` | 独自 Dockerfile パス。省略時はテンプレート Dockerfile を自動生成 |

---

## 11. 利用例

### 11.1 Java アプリケーション

```yaml
include:
  - project: '<テンプレートプロジェクト>'
    ref: main
    file: '/templates/common-pipeline.yml'
  - project: '<テンプレートプロジェクト>'
    ref: main
    file: '/templates/java-pipeline.yml'

variables:
  APP_DIR: "."
  ECR_REPOSITORY: "sample-java-app"
  SONAR_PROJECT_KEY: "sample-java-app"
```

### 11.2 Java 共通部品

```yaml
include:
  - project: '<テンプレートプロジェクト>'
    ref: main
    file: '/templates/common-pipeline.yml'
  - project: '<テンプレートプロジェクト>'
    ref: main
    file: '/templates/java-pipeline.yml'

variables:
  APP_DIR: "."
  SONAR_PROJECT_KEY: "sample-java-library"
```

### 11.3 React アプリケーション

```yaml
include:
  - project: '<テンプレートプロジェクト>'
    ref: main
    file: '/templates/common-pipeline.yml'
  - project: '<テンプレートプロジェクト>'
    ref: main
    file: '/templates/react-pipeline.yml'

variables:
  APP_DIR: "."
  ECR_REPOSITORY: "sample-react-app"
  SONAR_PROJECT_KEY: "sample-react-app"
```

### 11.4 React 共通部品

```yaml
include:
  - project: '<テンプレートプロジェクト>'
    ref: main
    file: '/templates/common-pipeline.yml'
  - project: '<テンプレートプロジェクト>'
    ref: main
    file: '/templates/react-pipeline.yml'

variables:
  APP_DIR: "."
  SONAR_PROJECT_KEY: "sample-react-library"
```

---

## 12. よくある質問

### 12.1 カスタム Dockerfile を使いたい

`DOCKERFILE_PATH` を指定してください。

```yaml
variables:
  DOCKERFILE_PATH: "docker/Dockerfile"
```

### 12.2 サブディレクトリのプロジェクトを対象にしたい

`APP_DIR` を指定してください。

```yaml
variables:
  APP_DIR: "apps/sample-java"
```

### 12.3 Nexus Publish 先を直接指定したい

Release / Snapshot の Publish 先は、以下の変数で指定できます。

```yaml
variables:
  MAVEN_PUBLISH_RELEASE_URL: "https://example.com/repository/maven-releases/"
  MAVEN_PUBLISH_SNAPSHOT_URL: "https://example.com/repository/maven-snapshots/"
  NPM_PUBLISH_RELEASE_URL: "https://example.com/repository/npm-releases/"
  NPM_PUBLISH_SNAPSHOT_URL: "https://example.com/repository/npm-snapshots/"
```

### 12.4 タグなしの通常コミットでは何が実行されるか

通常コミットでは、主に build / test / sonar が実行されます。

Nexus Publish / ECR Push はタグ実行時のみ実行されます。

### 12.5 タグ実行時に test / sonar は実行されるか

Java / React ともに、タグ実行時は test / sonar はスキップされます。

タグ実行時は、ビルド済み成果物を利用して Nexus Publish / Docker Build / ECR Push を行います。

### 12.6 Release / Snapshot はどこで切り替わるか

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

---

## 13. 注意事項

- タグ形式が不正な場合、`pre:pipeline` でエラーになります。
- `release-*` は Release として扱います。
- `snapshot-*` は Snapshot として扱います。
- Java の Nexus Publish では、CI 側で `ci-publish.gradle` を生成します。
- React の Nexus Publish では、CI 側で `package.json` の version を変更します。
- Dockerfile 未指定時は、テンプレート Dockerfile が自動生成されます。
- 独自の Dockerfile を利用する場合は `DOCKERFILE_PATH` を指定してください。
