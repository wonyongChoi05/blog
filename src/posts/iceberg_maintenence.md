---
title: Apache Iceberg Maintenance
description: Apache Iceberg의 유지보수에 대해 알아봅니다.
permalink: posts/{{ title | slug }}/index.html
date: "2025-04-12"
updated: "2024-04-12"
tags: [Iceberg, Iceberg Maintenance, Table format, Datalake]
---

# Iceberg Maintenance

---

## Expire Snapshots

Iceberg 테이블에 커밋(commit)을 할 때마다 테이블의 새로운 스냅샷이 생성됩니다. 스냅샷은 time travel에 활용되거나, 테이블이 정상적일 때로 롤백할 수 있는 기능을 제공합니다.

스냅샷은 expireSnapshots 옵션이 만료될 때까지 사라지지 않습니다. 그렇기 때문에 더 이상 필요하지 않은 데이터 파일을 삭제하고, 테이블 메타데이터 크기를 작게 유지하려면 정기적으로 스냅샷을 삭제하는 것이 좋습니다.

아래 API는 1일보다 오래된 스냅샷을 만료시키는 Spark Job입니다.

```java
Table table = ...
SparkActions
    .get()
    .expireSnapshots(table)
    .expireOlderThan(tsToExpire)
    .execute();
```

오래된 스냅샷이 만료되면 메타데이터에서 제거되기 때문에 더 이상 time travel 쿼리에 사용할 수 없습니다.

> 데이터 파일은 time travel이나 롤백에 사용될 수 있는 스냅샷에서 더 이상 참조되지 않을 때까지 삭제되지 않습니다. 스냅샷이 정기적으로 만료되면 사용되지 않는 데이터 파일이 삭제되기 때문에 별도로 삭제하지 않아도 됩니다.

## 오래된 메타데이터 파일 제거

Iceberg는 JSON 파일을 사용하여 테이블 메타데이터를 추적합니다. 테이블이 변경될 때마다 새로운 메타데이터 파일이 생성되어 원자성을 제공합니다.

이전 메타데이터 파일은 기록용으로 보관합니다. 스트리밍 작업으로 작성된 테이블처럼 커밋이 자주 발생하는 테이블은 메타데이터 파일을 정기적으로 정리해야 합니다.

메타데이터 파일을 자동으로 정리하려면 `write.metadta.delete-after-commit.enabled=true`을 설정합니다. 이렇게 하면 일부 메타데이터 파일은 유지되고 새 메타데이터 파일이 생성될 때마다 가장 오래된 메타데이터 파일이 삭제됩니다.

- write.metadata.delete-after-commit.enabled: 각 테이블 커밋 후에 마지막 메타데이터를 삭제할지 여부
- write.metadata.previous-versions-max: 보관할 이전 메타데이터 파일의 수

`write.metadta.delete-after-commit.enabled=true` 옵션은 메타데이터 로그에 추적되는 메타데이터 파일만 삭제하며 고아(orphan) 메타데이터 파일은 삭제하지 않습니다.

예를 들어 `write.metadata.previous-versions-max=10`으로 설정하면 100번의 커밋 후 추적되는 메타데이터 파일 10개와 orphan 메타데이터 파일 90개가 생성됩니다.

## 고아(Oprphan) 파일 삭제

orphan file은 테이블의 메타데이터에 의해 추적되지 않는 파일을 의미합니다. 즉, 테이블의 데이터 디렉터리 안에 존재하지만, Iceberg의 스냅샷이나 매니페스트 파일에서 참조되지 않는 파일들입니다.

Orphan file는 보통 다음과 같은 이유로 발생합니다.

1. 작업 중간에 실패하거나 취소된 경우: 데이터를 쓰는 도중 실패했지만 정리(cleanup)이 안 된 경우
2. 테이블 외부에서 수동으로 파일을 추가한 경우: Iceberg는 메타데이터 기반이므로, 직접 파일을 넣으면 Iceberg는 이를 인식하지 못함
3. Write 작업 중 여러 클러스터/워크플로우에서 충돌이 있었던 경우
4. 오래된 스냅샷은 삭제되었지만, 데이터 파일은 삭제되지 않은 경우

따라서 이러한 고아 파일을 삭제하려면 deleteOrphanFiles job을 사용하면 됩니다.

```java
Table table = ...
SparkActions
    .get()
    .deleteOrphanFiles(table)
    .execute();
```

> 데이터 및 메타데이터 디렉터리에 파일이 많으면 이 작업을 완료하는 데 시간이 오래 걸릴 수 있습니다. 주기적으로 실행하는 것이 좋지만, 자주 실행할 필요는 없습니다.

### 주의할 점

쓰기가 아직 끝나지 않았는데, 그 중간 파일을 orphan 파일로 잘못 판단해서 삭제하면 테이블에 문제가 생길 수 있습니다. 따라서 기본적으로는 3일 이전 파일까지만 삭제합니다.

예를 들어 데이터 파일을 쓰는 도중 커밋이 완료되지 않은 상태에서 remvoe orphan file을 실행한다면 테이블이 깨지거나 데이터가 유실됩니다.

그래서 등장한 개념이 retention interval(removeOrphanFiles().olderThan())인데, 해당 옵션은 지금 시점에서 X시간 X일보다 더 오래된 파일만 orphan으로 간주하고 삭제해라 라는 뜻입니다. 기본값은 3일이며, 3일 이내에 생성된 데이터 파일은 절대 삭제하지 않습니다.

# Optional Maintenance

일부 테이블은 추가적인 메인터넌스가 필요합니다. 예를 들어 스트리밍 쿼리는 스몰 데이터 파일을 많이 생성하며, 이러한 파일은 더 큰 파일로 합쳐야 합니다. 또한, 일부 테이블은 쿼리에 필요한 데이터를 훨씬 더 빠르게 찾을 수 있도록 매니페스트 파일을 다시 작성하면 도움이 됩니다.

## Compact Data File

Iceberg는 각 데이터 파일을 테이블로 추적합니다. 데이터파일이 많을수록 매니페스트 파일에 더 많은 메타데이터가 저장되기 때문에 파일 오픈 비용 때문에 쿼리가 느려질 수 있습니다.

Iceberg는 Spark rewriteDataFiles 옵션을 통해 스몰 파일들을 하나의 큰 데이터파일로 컴팩션 할 수 있습니다. 이를 이를 통해 메타데이터 오버헤드와 런타임 파일 오픈 비용을 줄일 수 있습니다.

```java
Table table = ...
SparkActions
    .get()
    .rewriteDataFiles(table)
    .filter(Expressions.equal("date", "2020-08-18"))
    .option("target-file-size-bytes", Long.toString(500 * 1024 * 1024)) // 500 MB
    .execute();
```

## Rewrite Manifast

Manifast 파일은 어떤 데이터 파일들이 어떤 파티션에 있고, 어떤 통계값을 가지는지 요약해 놓은 메타데이터 인덱스입니다. Manifest list는 여러 개의 manifest를 묶어 관리하는 메타데이터 트리의 상위 구조입니다. 이 구조 덕분에 Iceberg는 쿼리할 때 전체 데이터 파일을 읽지 않고도 필요한 데이터만 조회할 수 있습니다.

Iceberg는 데이터를 쓸 때마다(commit interval) 새로운 manifest를 생성하지만, 일정 조건에 따라 자동으로 압축(compaction)하여 너무 작은 manifest 파일이 많아지지 않도록 관리합니다

이 자동 압축은 쓰기 순서대로 진행되며, 쓰기 패턴이 쿼리 조건과 잘 맞을 때 (예: 시간 파티셔닝 + 시간 조건 쿼리) 쿼리 성능이 매우 좋아집니다

하지만 문제가 생기는 경우는 쓰기 패턴과 쿼리 패턴이 일치하지 않는 경우인데

> 예: 지역 기반으로 데이터를 썼는데, 시간 조건 쿼리가 많다든지

이런 경우 manifest에 저장된 데이터 그룹이 쿼리 필터와 잘 안 맞아서 프루닝 효율이 떨어집니다.

Iceberg는 이런 경우를 위해 manifest 파일을 다시 쓰는 기능을 제공합니다

```scala
Actions.forTable(table)
  .rewriteManifests()
  .clusterBy("date") // 예: 첫 번째 파티션 필드
  .execute()
```
