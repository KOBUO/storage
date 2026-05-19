# gitlab-ci-common

管理業務DX向けの GitLab CI パイプラインテンプレートです。

Java / React プロジェクト向けに、ビルド・テスト・カバレッジ取得・SonarQube 解析・Docker Build・ECR Push などの CI/CD 処理を共通化することを目的としています。

各プロジェクトでは、本リポジトリのテンプレートを `include` して利用します。

主に以下を共通化します。

- Gradle build / test / coverage（JaCoCo）
- React build / test / coverage（Vitest）
- SonarQube 解析
- Docker build
- ECR Push
- Nexus Maven / npm Publish

---

## ファイル構成

| ファイル | 内容 |
|---|---|
| `templates/common-pipeline.yml` | Docker / DinD / ECR Push などの共通定義 |
| `templates/java-pipeline.yml` | Java / Gradle 向け build / test / sonar / ECR push 定義 |
| `templates/react-pipeline.yml` | React / pnpm 向け build / test / sonar / ECR push 定義 |
| `templates/docker/java/Dockerfile` | Java アプリ用 Dockerfile |
| `templates/docker/react/Dockerfile` | React アプリ用 Dockerfile（nginx） |

---

## 利用イメージ

### アプリ開発（ECR Push）

ビルド・テスト・SonarQube 解析・Docker Build・ECR Push を行う場合のイメージです。  
タグ作成時に ECR へイメージが Push されます。

**Java**

```yaml
include:
  - project: 'your-group/gitlab-ci-common'
    ref: main
    file: '/templates/common-pipeline.yml'
  - project: 'your-group/gitlab-ci-common'
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

**React**

```yaml
include:
  - project: 'your-group/gitlab-ci-common'
    ref: main
    file: '/templates/common-pipeline.yml'
  - project: 'your-group/gitlab-ci-common'
    ref: main
    file: '/templates/react-pipeline.yml'

stages:
  - build
  - test
  - sonar
  - docker-build
  - ecr-push

variables:
  APP_DIR: "frontend"
  ECR_REPOSITORY: my-app
  SONAR_PROJECT_KEY: my-app
```

---

### 共通部品（Nexus Publish）

ビルド・テスト・SonarQube 解析・Nexus Publish を行う場合のイメージです。  
Docker build は不要なため、`docker-build` / `ecr-push` ステージは含めません。  
タグ作成時に Nexus へ成果物が Publish されます。

**Java（Maven JAR）**

```yaml
include:
  - project: 'your-group/gitlab-ci-common'
    ref: main
    file: '/templates/common-pipeline.yml'
  - project: 'your-group/gitlab-ci-common'
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

**React（npm パッケージ）**

```yaml
include:
  - project: 'your-group/gitlab-ci-common'
    ref: main
    file: '/templates/common-pipeline.yml'
  - project: 'your-group/gitlab-ci-common'
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

## 機能説明

### `templates/common-pipeline.yml`

Docker / DinD / ECR Push などの共通定義を行うテンプレートです。

| ジョブ名 | 内容 |
|---|---|
| `.docker-base` | Docker / DinD 共通設定 |
| `.docker-build-base` | docker-build ステージ共通定義（Nexus ログイン含む） |
| `.ecr-push-base` | ecr-push ステージ共通定義（ECR ログイン含む） |

### `templates/java-pipeline.yml`

Java / Gradle 向け build / test / coverage 定義を行うテンプレートです。

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

```
gradle -p "$APP_DIR" build -x test --stacktrace
```

#### `java-test`

Gradle test を実行し、JaCoCo カバレッジレポートを生成します。

```
gradle -p "$APP_DIR" test jacocoTestReport
```

#### `java-sonar`

SonarQube 解析を実行します。失敗してもパイプラインは継続します。

```
gradle -p "$APP_DIR" sonar
```

### `templates/react-pipeline.yml`

React / pnpm 向け build / test / coverage 定義を行うテンプレートです。

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

```
pnpm --dir "$APP_DIR" run build
```

#### `react-test`

Vitest を利用してテストを実行し、カバレッジレポートを生成します。

```
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

SonarQube 解析を実行します。失敗してもパイプラインは継続します。`APP_DIR` に `sonar-project.properties` を配置してください。

---

## ECR Push について

ECR Push はタグ作成時のみ実行されます。

```
git tag v1.0.0
git push origin v1.0.0
```

Push されるイメージは以下の形式です。

```
${ECR_REGISTRY}/${ECR_REPOSITORY}:${CI_COMMIT_TAG}
${ECR_REGISTRY}/${ECR_REPOSITORY}:latest
```

`ECR_REGISTRY` は `AWS_ACCESS_KEY_ID` をもとに AWS アカウント ID から自動導出されます。

---

## CI 変数

`.gitlab-ci.yml` の `variables` に設定します。

| 変数名 | 説明 |
|---|---|
| `APP_DIR` | アプリのサブディレクトリ |
| `ECR_REPOSITORY` | ECR リポジトリ名（ECR Push 時のみ） |
| `SONAR_PROJECT_KEY` | SonarQube プロジェクトキー |

---

## よくある質問

### サブディレクトリのプロジェクトを対象にしたい

`APP_DIR` を指定してください。

```yaml
variables:
  APP_DIR: "sample/java"
```

### Nexus へライブラリを Publish したい

タグを作成すると `java-nexus-publish` または `react-nexus-publish` が実行されます。

**Java**: `build.gradle` に `publishing{}` ブロックが必要です。

```kotlin
publishing {
    publications {
        create<MavenPublication>("mavenJava") {
            from(components["java"])
        }
    }
    repositories {
        maven {
            url = uri(System.getenv("BS72_MAVEN_RELEASE_URL"))
            credentials {
                username = System.getenv("BS72_NEXUS_USER")
                password = System.getenv("BS72_NEXUS_PASSWORD")
            }
        }
    }
}
```


### ECR Push を実行したい

タグを作成して Push してください。

```
git tag v1.0.0
git push origin v1.0.0
```
