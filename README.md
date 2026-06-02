# README.md

## タグルール

### Releaseタグ

```text
release-1.0.0
release-v1.0.0
```

| 変数 | 値 |
|------|------|
| TAG_TYPE | release |
| VERSION | release- を除去した値 |
| PUBLISH_VERSION | VERSION と同じ |
| MAVEN_PUBLISH_URL | MAVEN_PUBLISH_RELEASE_URL |
| NPM_PUBLISH_URL | NPM_PUBLISH_RELEASE_URL |
| NPM_TAG | latest |

### Snapshotタグ

```text
snapshot-1.2.3-build1
snapshot-v1.2.3-build1
snapshot-1.2.3-systest-build3
snapshot-v1.2.3-systest-build3
```

| 変数 | 値 |
|------|------|
| TAG_TYPE | snapshot |
| VERSION | snapshot- を除去した値 |
| PUBLISH_VERSION | ${VERSION}-snapshot |
| MAVEN_PUBLISH_URL | MAVEN_PUBLISH_SNAPSHOT_URL |
| NPM_PUBLISH_URL | NPM_PUBLISH_SNAPSHOT_URL |
| NPM_TAG | snapshot |

### タグ例

| Gitタグ | TAG_TYPE | VERSION | PUBLISH_VERSION | npm dist-tag |
|----------|----------|----------|----------|----------|
| release-1.0.0 | release | 1.0.0 | 1.0.0 | latest |
| release-v1.0.0 | release | v1.0.0 | v1.0.0 | latest |
| snapshot-1.2.3-build1 | snapshot | 1.2.3-build1 | 1.2.3-build1-snapshot | snapshot |
| snapshot-v1.2.3-build1 | snapshot | v1.2.3-build1 | v1.2.3-build1-snapshot | snapshot |
| snapshot-1.2.3-systest-build3 | snapshot | 1.2.3-systest-build3 | 1.2.3-systest-build3-snapshot | snapshot |
| snapshot-v1.2.3-systest-build3 | snapshot | v1.2.3-systest-build3 | v1.2.3-systest-build3-snapshot | snapshot |

### 不正タグ

```text
1.0.0
v1.0.0
1.0.0-snapshot
release_1.0.0
snapshot_1.0.0
```
