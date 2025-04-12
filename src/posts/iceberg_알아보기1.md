---
title: Apache Iceberg 살펴보기
description: Apache Iceberg가 무엇이고, 대표적으로 제공하는 기능에 대해 알아봅니다.
permalink: posts/{{ title | slug }}/index.html
date: "2024-10-05"
updated: "2024-10-05"
tags: [Iceberg, Table format, Datalake]
---

# Apache Iceberg란?

---

Apache Iceberg(아파치 아이스버그)는 매우 큰 분석 데이터를 위한 오픈(open) 테이블 포맷입니다. SQL 테이블처럼 작동할 수 있으며 Spark, Trino, PrestoDB, Flink, Hive, Impala를 포함한 컴퓨팅 엔진에서 활용 가능합니다.

## Iceberg가 제공하는 기능

Iceberg는 Schema Evolution을 지원하며, 실수로 데이터를 삭제하거나 취소하지 않습니다. 또한 Hiddon Partition을 지원하기 때문에 사용자는 파티션을 알 필요가 없습니다.

- 스키마 진화는 추가, 삭제, 변경, 컬럼명 변경 등을 지원하며 부작용이 없습니다.
- Hiddon Partitioning은 유저가 실수로 파티션을 기입하지 않음으로 인해 쿼리가 매우 느려지는 것을 방지합니다.
- Partition layout evolution은 데이터 볼륨이나 쿼리 패턴이 변경될 때 테이블 레아이웃을 업데이트 할 수 있습니다.
- Time Travle을 통해 정확히 동일한 테이블 스냅샷을 사용하는 재현 가능하거나 사용자가 변경 사항을 쉽게 조사할 수 있습니다.
- 버전 롤백을 통해 사용자는 테이블을 정상적인 상태로 롤백하여 문제를 빠르게 해결할 수 있습니다.

## 신뢰성 및 성능

Iceberg는 거대한 테이블을 위해 만들어졌습니다. Iceberg는 단일 테이블에 수십 페타바이트(PB)를 저장하고 쿼리할 수 있습니다.

- 테이블을 읽거나 파일을 찾는것을 쿼리 엔진에 위임하지 않기 때문에 쿼리 플랜이 빠릅니다.
- 테이블 메타데이터를 사용하여 파티션 및 컬럼 수준 통계로 데이터 파일을 정리할 수 있습니다.
- 테이블 변경은 원자적이며 부분적이나 커밋되지 않은 변경 사항을 볼 수 없습니다.
- 쓰기가 충돌하는 경우 낙관적 락(Optimistic Lock)으로 안전하게 업데이트합니다.

# Evolution

---

Iceberg는 Schema Evolution을 지원합니다. 또한 데이터 크기가 변경될 때 파티션 레이아웃을 변경할 수도 있습니다. Iceberg는 기존 Hive 테이블을 변경할 때 처럼 테이블 데이터를 다시 쓰거나 새 테이블로 마이그레이션하는 것과 같이 큰 비용이 필요하지 않습니다.

예를 들어 Hive 테이블 파티셔닝은 변경할 수 없으므로 Daily 파티션 레이아웃에서 Hourly 파티션 레이아웃으로 변경하려면 새 테이블이 필요합니다. 그리고 기존 테이블에 대한 쿼리들은 Daily 파티션 레이아웃에 맞춰져있기에 새 테이블에 대한 쿼리를 다시 작성해야 합니다. 어떤 경우에는 컬럼 이름을 변경하는 것과 같은 간단한 변경조차도 지원되지 않거나 데이터 정합성에 문제가 발생할 수도 있습니다.

## Schema Evolution

Iceberg 테이블은 다음과 같은 스키마 진화를 지원합니다.

- 추가: 테이블이나 중첩된(nested) 구조에 새 컬럼을 추가합니다.
- 삭제: 테이블이나 중첩된(nested) 구조체에서 기존 컬럼을 제거합니다.
- 이름 변경: 중첩된(nested) 구조체의 기존 컬럼 또는 필드 이름을 변경합니다.
- 변경: 컬럼, 구조체 필드, 맵 키, 맵 값 또는 리스트 유형을 변경합니다.
- 재정렬: 중첩된(nested) 구조체의 열 또는 필드 순서를 변경합니다.

Iceberg 스키마 진화는 메타데이터 변경이므로 업데이트를 수행하기 위해 데이터 파일을 다시 작성할 필요가 없습니다. 또한 스키마 진화가 독립적이고 부작용이 없다고 보장합니다.

1. 추가된 컬럼은 다른 컬럼의 기존 값을 읽지 않습니다.
2. 컬럼이나 필드를 삭제해도 다른 컬럼의 값은 변경되지 않습니다.
3. 컬럼이나 필드를 업데이트해도 다른 컬럼의 값은 변경되지 않습니다.
4. 구조체의 컬럼이나 필드의 순서를 변경해도 컬럼이나 필드 이름과 같은 값은 변경되지 않습니다.

Iceberg는 고유 ID를 사용하여 테이블의 각 컬럼을 추적합니다. 만약 컬럼을 추가하면 새로운 ID가 할당되므로, 실수로 기존 데이터가 사용될 일은 없습니다.

- 이름으로 컬럼을 추적하는 방식은 이름이 재사용되면 실수로 컬럼의 삭제를 취소할 수 있으며, 이는 #1을 위반합니다.
- 위치별로 컬럼을 추적하는 방식은 사용된 이름을 변경하지 않고는 컬럼을 삭제할 수 없으므로 #2를 위반합니다.

## Partition Evolution

쿼리가 파티션 값을 직접 참조하지 않기 때문에 Iceberg 테이블 파티셔닝은 기존 테이블에서 업데이트될 수 있습니다.

파티션 레이아웃을 변경할 때 새 데이터는 새 레이아웃에서 새 파티션을 사용하여 작성되지만, 이전 파티션 레이아웃으로 작성된 데이터는 변경되지 않습니다.

각 파티션 버전의 메타데이터는 별도로 보관됩니다. 이 때문에 쿼리를 작성하기 시작하면 파티션 플랜이 제공됩니다. 여기서 각 파티션 레이아웃은 해당 특정 파티션 레이아웃에 대해 파생된 필터를 사용하여 파일을 별도로 플래닝합니다.

![partition-spec-evolution.png](/images/partition-spec-evolution)

> 2008년 데이터는 월별로 분할되어 있습니다. 2009년부터 테이블이 업데이트되어 일별로 분할됩니다. 즉, 두 개의 파티션 레이아웃이 공존할 수 있습니다.

Iceberg는 히든 파티셔닝을 사용하므로 특정 파티션 레이아웃에 대한 쿼리를 작성하지 않아도 빠르게 쿼리할 수 있습니다. 대신 필요한 데이터를 직접 선택하는 쿼리를 작성하면 Iceberg가 일치하는 데이터가 없는 파일을 자동으로 제외하고 쿼리합니다.

파티션 진화는 메타데이터를 변경하는 작업이기 때문에 파일을 즉시 다시 쓰지 않습니다.

Iceberg는 updateSpec이라는 파티션 레이아웃을 변경하는 API를 제공합니다. 예를 들어, 다음 코드는 id 컬럼 값을 8개의 버킷으로 나누고 기존 파티션 필드(category)를 제거합니다.

```java
Table sampleTable = ...;
sampleTable.updateSpec()
    .addField(bucket("id", 8))
    .removeField("category")
    .commit();
```

Spark은 ALTER SQL문을 통해 파티션 스펙을 수정할 수 있습니다.

## Sort order evolution

파티션 스펙과 유사하게, Iceberg 정렬 순서도 기존 테이블에서 업데이트할 수 있습니다. 정렬 순서를 진화시키면 이전 순서로 작성된 이전 데이터는 변경되지 않습니다.

하지만, 엔진은 항상 최신 정렬 순서로 데이터를 작성하거나 정렬 작업 비용이 엄청나게 비쌀 때 정렬되지 않은 상태로 작성될 수 있습니다.

Iceberg는 replaceSortOrder이라는 정렬 순서를 업데이트하는 API를 제공합니다. 예를 들어 다음 코드를 사용하여 id 컬럼을 오름차순으로 정렬하고 null을 마지막으로 보내고, category 컬럼은 내림차순, null이 먼저 있는 새 정렬 전략을 만들 수 있습니다.

```java
Table sampleTable = ...;
sampleTable.replaceSortOrder()
   .asc("id", NullOrder.NULLS_LAST)
   .dec("category", NullOrder.NULL_FIRST)
   .commit();
```

Spark은 ALTER SQL문을 통해 파티션 스펙을 수정할 수 있습니다.

# Partitioning

파티셔닝은 쿼리를 작성할 때 비슷한 행(row)를 grouping하여 쿼리를 더 빠르게 만드는 방법입니다.

예를 들어, logs테이블의 로그 항목에 대한 쿼리에는 일반적으로 다음과 같이 오전 10시부터 오전 12시 사이의 로그에 대한 쿼리와 같이 시간 범위가 포함됩니다.

```sql
SELECT level, message FROM logs
WHERE event_time BETWEEN '2018-12-01 10:00:00' AND '2018-12-01 12:00:00';
```

logs 테이블을 날짜별로 파티셔닝되도록 구성하면 event_time 로그 이벤트가 동일한 이벤트 날짜의 파일로 그룹핑됩니다. Iceberg는 해당 날짜를 추적하고 이를 사용하여 해당 날짜가 아닌 데이터는 건너뜁니다.

Iceberg는 타임스탬프를 연도, 월, 일, 시간 단위로 분할할 수 있습니다. 또한 level이 로그 예제와 같이 카테고리컬한 컬럼을 사용하여 행을 함께 저장하여 쿼리 속도를 높일수도 있습니다.

## Iceberg의 Hidden Partitioning이 Hive 파티셔닝과 다른 점이 무엇일까?

Hive와 같은 다른 테이블 형식은 파티셔닝을 지원하지만 Iceberg는 히든 파티셔닝을 지원합니다.

- Iceberg는 테이블의 로우에 대한 파티션 값을 생성하는 지루하고 오류가 발생하기 쉬운 작업을 처리합니다.
- Iceberg는 불필요한 파티션을 자동으로 읽는 것을 피합니다. 소비자는 테이블이 어떻게 파티션되어 있는지 알 필요가 없고 쿼리에 파티션 필터를 추가할 필요가 없습니다.
- Iceberg Partition Layout은 필요에 따라 진화할 수 있습니다.

## Partitioning of Hive

차이점을 설명하기 위해 Hive가 logs 테이블을 처리하는 방법을 알아보겠습니다. Hive에서 파티션은 명시적이고 컬럼으로 나타나므로 logs 테이블에는 event_date라는 컬럼이 있습니다. 쓰기 시 insert 쿼리는 event_date 컬럼에 대한 데이터를 제공해야 합니다.

```sql
INSERT INTO logs PARTITION (event_date)
  SELECT level, message, event_time, format_time(event_time, 'YYYY-MM-dd')
  FROM unstructured_log_source;
```

마찬가지로, 효율적으로 테이블을 검색하기 위해서는 조회 쿼리에 파티션 필터가 존재해야합니다.

```sql
SELECT level, count(1) as count FROM logs
WHERE event_time BETWEEN '2018-12-01 10:00:00' AND '2018-12-01 12:00:00'
  AND event_date = '2018-12-01';
```

파티션 필터를 걸지 않고 쿼리를 실행하면 Hive는 테이블의 모든 파일을 검색하게 됩니다. 이로 인해 여러 가지 문제가 발생합니다.

- Hive는 파티션 값을 검증할 수 없습니다. 올바른 쿼리를 작성하는 것은 사용자의 몫입니다.
  - 잘못된 형식을 사용하면 쿼리 실패가 아닌 자동으로 잘못된 결과가 생성됩니다.
  - 잘못된 컬럼을 파티션 컬럼으로 지정하면 실패가 아닌 잘못된 결과가 생성됩니다.
- 쿼리는 테이블의 파티션 구성에 밀접한 관계가 있으므로, 쿼리를 중단하지 않고는 파티션 구성을 변경할 수 없습니다.
  - 예를 들면 daily 파티션 테이블을 hourly 파티션 테이블로 변경할 때 기존에 모든 배치(ETL) 쿼리들에 대해 수정이 필요할 것 입니다.

## Iceberg의 Hidden Partitioning

Iceberg는 컬럼 값을 가져와서 선택적으로 변환하여 파티션 값을 생성합니다. Iceberg는 event_time으로 변환하는 역할을 하며 event_date와 관계를 추적합니다.

테이블 파티셔닝은 이러한 관계를 사용하여 구성됩니다. logs 테이블은 date(event_time) 및 level 컬럼을 기준으로 파티셔닝 됩니다.

Iceberg는 사용자가 유지 관리하는 파티션 컬럼을 필요로 하지 않으므로 파티셔닝을 숨길 수 있습니다. 파티션 값은 항상 올바르게 생성되며 가능한 경우 쿼리 속도를 높이는 데 사용됩니다. 생산자와 소비자는 event_date 컬럼을 볼 수도 없습니다.

가장 중요한 것은 쿼리가 더 이상 테이블의 물리적인 레이아웃에 의존하지 않는다는 것입니다. 물리적 및 논리적 구분을 통해 Iceberg 테이블은 데이터 크기가 변경됨에 따라 파티션을 진화시킬 수 있습니다. 잘못 구성된 테이블은 값 비싼 마이그레이션 없이 수정할 수 있습니다.

## Hidden Partitioning 성능 문제

히든 파티셔닝은 파티션 레이아웃을 변경할 때 기존 데이터를 리파티셔닝 하지 않기 때문에 파티션 수정이 Hive에 비해 부담스럽지 않습니다.

하지만 그로 인해 파티션 일자별 성능이 다르게 나올 수 있습니다. 예를 들면,

1. year 파티션에서 order_date로 변경 (2023년 6월 이후부터):

   - 만약 2023년 6월부터 order_date를 기준으로 파티셔닝한다면, 2023년 6월 이전의 데이터는 여전히 year=2023과 같은 year 파티션 구조로 존재합니다.
   - 따라서, 2023년 6월 이전 데이터는 year=2023 파티션으로 읽고, 2023년 6월 이후의 데이터는 order_date 기준으로 분할된 파티션을 사용하게 됩니다.

2. 쿼리 성능에 미치는 영향:

   - 2023년 6월 이전에 대한 쿼리는 year=2023 파티션을 풀스캔해야 하므로, 성능이 다소 떨어질 수 있습니다.

   - 2023년 6월 이후에는 order_date 기준으로 데이터를 쪼개서 파티셔닝했기 때문에 날짜별로 빠른 쿼리 성능을 제공할 것입니다.

## 성능 최적화 방법 (데이터 리파티셔닝):

성능을 최적화하고 싶다면, 이전 파티션 전략으로 저장된 데이터를 새로운 파티션 전략에 맞게 리파티셔닝하는 것이 유리할 수 있습니다.

Iceberg에서는 rewrite(재작성) 또는 repartition을 통해 기존 데이터를 새로운 파티션 기준에 맞게 변경할 수 있습니다.

예를 들어, year=2023 데이터를 order_date 기준으로 리파티셔닝하면, 향후 쿼리 성능이 더 향상될 수 있습니다.

```sql
spark.sql("""
    CREATE TABLE new_table USING iceberg
    PARTITIONED BY (order_date)
    AS SELECT * FROM old_table
""")
```

### Hidden Partitioning 정리

- 사용자는 쿼리에 파티션 컬럼을 직접 추가할 필요가 없습니다.
- 기존 데이터 파일을 이동하지 않고 파티션 레이아웃 변경 가능합니다.(리파티셔닝이 불필요)
- 폴더 구조가 아닌, 메타데이터 기반으로 파티션 관리합니다.
- 다양한 파티션 트랜스폼을 지원합니다. (bucket, truncate 등)
- Iceberg는 파티션 전략을 변경한다고 해서 기존 데이터를 자동으로 리파티셔닝하지 않으며, 그로 인해 기존 데이터에 대해서는 성능이 떨어질 수 있습니다.
- 성능 최적화를 위해서는 데이터를 새로운 파티션 전략에 맞게 rewrite하거나 리파티셔닝하는 방법이 필요합니다.

# Time Travel

---

Iceberg의 Time Travel 기능은 테이블의 특정 시점이나 스냅샷 상태를 조회할 수 있는 강력한 도구로, 데이터 디버깅, 감사, 그리고 과거 데이터 분석에 유용합니다. 이 기능은 테이블의 모든 변경 사항을 스냅샷으로 저장하며, 이러한 스냅샷을 기반으로 데이터를 조회할 수 있습니다.

## Time Travel의 주요 개념

1. 스냅샷(Snapshot)

- Iceberg는 테이블이 수정될 때마다 스냅샷을 생성합니다. 각 스냅샷은 테이블의 특정 시점 상태를 완전하고 일관되게 나타냅니다. 각 스냅샷은 고유 ID를 가지며, 이를 통해 특정 상태를 식별할 수 있습니다.

2. 사용 사례

- 디버깅: 데이터가 잘못변경되었거나 삭제된 경우, 과거 상태를 확인하여 문제를 파악할 수 있습니다.
- 감사 및 규제 준수: 특정 시점의 데이터를 조회하여 규제 요구 사항을 충족할 수 있습니다.
- 재현성: 동일한 데이터 상태를 재현하여 분석 결과를 검증할 수 있습니다.

### Time Travel 쿼리 방법

1. Timestamp 기반 조회

```sql
-- 특정 타임스탬프 기준으로 데이터 조회
SELECT * FROM my_table FOR TIMESTAMP AS OF TIMESTAMP '2025-01-01 12:00:00';
```

2. Snapshot ID 기반 조회

```sql
-- 특정 스냅샷 ID 기준으로 데이터 조회
SELECT * FROM my_table FOR VERSION AS OF 2583872980615177898;
```

3. Named Reference(Brach/Tag)
   브랜치나 태그 이름을 사용하여 특정 시점의 데이터를 조회합니다.

### Time Travel 구현 예시

```python
# 특정 타임스탬프 기준으로 데이터 로드
timestamp = "2025-01-01T12:00:00.000Z"
historical_data = spark.read.format("iceberg") \
    .option("as-of-timestamp", timestamp) \
    .load("spark_catalog.default.customers")
historical_data.show()

#### 또는 아래와 같이 Spark SQL 사용
spark.sql("SELECT * FROM iceberg.default.customers TIMESTAMP AS OF '2025-03-30 12:00:00'").show(truncate=False)
```

### 결과

> SELECT \* FROM iceberg.default.customers TIMESTAMP AS OF '2025-03-30 12:00:00'

```
+---+-------+---+
|id |name   |age|
+---+-------+---+
|1  |Alice  |30 |
|2  |Bob    |25 |
|3  |Charlie|35 |
|1  |Alice  |30 |
|2  |Bob    |25 |
|3  |Charlie|35 |
+---+-------+---+
```

> SELECT \* FROM iceberg.default.customers FOR TIMESTAMP AS OF TIMESTAMP '2025-03-30 15:00:00'

```
+---+-------+---+
|id |name   |age|
+---+-------+---+
|1  |Alice  |30 |
|2  |Bob    |25 |
|3  |Charlie|35 |
|1  |Alice  |30 |
|2  |Bob    |25 |
|3  |Charlie|35 |
|1  |Alice  |30 |
|2  |Bob    |25 |
|3  |Charlie|35 |
+---+-------+---+
```

### 주의사항

- Iceberg는 기본적으로 최근 몇 개의 스냅샷만 유지하며, 오래된 스냅샷은 만료될 수 있으므로 필요한 경우 스냅샷 보존 기간을 설정해야 합니다.
- 타임스탬프 기반 조회는 클럭 스큐 문제로 인해 정확히 동일한 데이터를 보장하지 않을 수 있으므로, 중요한 경우 Snapshot ID를 사용하는 것이 더 안전합니다.
