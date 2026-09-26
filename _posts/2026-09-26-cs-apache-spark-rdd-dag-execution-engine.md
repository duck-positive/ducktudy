---
layout: post
title: "Apache Spark RDD와 DAG 실행 엔진 완전 정복: 분산 데이터 처리 내부 구조"
date: 2026-09-26
categories: [cs, computer-science]
tags: [apache-spark, rdd, dag, distributed-computing, big-data, scala, pyspark]
---

대용량 데이터를 분산 처리하는 시스템에서 Apache Spark는 사실상 표준이 된 프레임워크다. 하지만 "Spark가 빠르다"는 말은 누구나 알아도, 내부적으로 어떻게 동작하는지는 잘 모르는 경우가 많다. 이 글에서는 Spark의 핵심 추상화인 RDD부터 DAG 기반 실행 엔진까지 내부 구조를 깊이 파헤친다.

## 1. RDD란 무엇인가 — Resilient Distributed Dataset

RDD(Resilient Distributed Dataset)는 Spark의 가장 기본적인 데이터 추상화다. 이름 자체에 핵심이 담겨 있다.

- **Resilient(복원력 있는)**: 장애가 발생해도 데이터 계보(lineage)를 통해 재계산 가능하다.
- **Distributed(분산된)**: 클러스터의 여러 노드에 파티션 단위로 나뉘어 저장된다.
- **Dataset(데이터셋)**: 레코드의 컬렉션으로, 타입화된 값을 갖는다.

RDD의 두 가지 핵심 특성은 **불변성(Immutability)**과 **지연 평가(Lazy Evaluation)**다. 한 번 생성된 RDD는 변경되지 않고, 변환(Transformation)을 거쳐 새로운 RDD가 파생된다. 그리고 액션(Action)이 호출되기 전까지는 실제로 아무 연산도 실행되지 않는다.

### Transformation vs Action

RDD의 연산은 크게 두 가지로 나뉜다.

**Transformation**: 새로운 RDD를 반환하는 연산. 지연 평가된다.
- `map`, `filter`, `flatMap`, `groupByKey`, `reduceByKey`, `join`, `union` 등

**Action**: 실제 계산을 트리거하고 결과를 반환하거나 외부로 내보내는 연산.
- `count`, `collect`, `reduce`, `saveAsTextFile`, `foreach` 등

### 파티션과 데이터 지역성

RDD는 파티션(Partition)으로 나뉘고, 각 파티션은 클러스터의 서로 다른 노드에서 병렬로 처리된다. Spark는 가능하면 데이터가 있는 노드에서 해당 파티션을 처리하는 **데이터 지역성(Data Locality)** 원칙을 따른다. 네트워크 전송을 최소화하여 성능을 높이는 것이다.

## 2. 왜 RDD와 DAG가 필요한가

기존 MapReduce는 각 단계마다 HDFS에 중간 결과를 저장하므로 디스크 I/O가 매우 많다. 반면 Spark의 RDD는 메모리에 중간 결과를 유지하고, DAG 스케줄러가 연산 단계를 지능적으로 파이프라인화하여 불필요한 디스크 쓰기를 제거한다.

**장애 복구**도 중요하다. RDD는 데이터 자체를 복제하지 않고 **Lineage(계보)** 정보를 유지한다. 파티션이 손실되면 Lineage를 따라 해당 파티션만 재계산한다. 이는 전체 데이터 복제보다 훨씬 효율적이다.

## 3. DAG 스케줄러 — Directed Acyclic Graph Scheduler

사용자가 액션을 호출하면 Spark는 내부적으로 **DAG(Directed Acyclic Graph)**를 생성하고 이를 스케줄링한다.

### Job → Stage → Task 분해

Spark의 실행 단위 계층 구조는 다음과 같다.

```
Job
 └── Stage (셔플 경계로 구분)
      └── Task (파티션 단위, 병렬 실행)
```

1. **Job**: 액션 하나가 하나의 Job을 만든다.
2. **Stage**: 셔플(Shuffle) 경계에서 Stage가 나뉜다. 셔플은 네트워크를 통해 데이터를 재분배하는 연산으로, `groupByKey`, `reduceByKey`, `join` 등이 해당한다.
3. **Task**: 각 파티션에 대해 하나의 Task가 생성되어 Executor에서 실행된다.

### Narrow vs Wide Transformation

Transformation의 종류에 따라 Stage 분리 여부가 결정된다.

**Narrow Transformation**: 각 입력 파티션이 정확히 하나의 출력 파티션에 기여한다. 셔플 없이 파이프라인 처리 가능하다.
- 예: `map`, `filter`, `union`, `flatMap`

**Wide Transformation**: 여러 입력 파티션의 데이터가 하나의 출력 파티션으로 흘러갈 수 있다. 셔플이 필요하며 Stage 경계를 생성한다.
- 예: `groupByKey`, `reduceByKey`, `join`, `distinct`, `repartition`

DAG 스케줄러는 Narrow Transformation들을 하나의 Stage 안에 묶어 파이프라인으로 처리하고, Wide Transformation이 나타날 때만 Stage를 분리한다.

## 4. 실제 구현 예제

### 예제 1: PySpark로 RDD 기본 연산과 DAG 이해

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("RDD DAG Demo") \
    .master("local[*]") \
    .getOrCreate()

sc = spark.sparkContext

# 1. RDD 생성 (데이터를 4개 파티션으로 분산)
numbers = sc.parallelize(range(1, 101), numSlices=4)

# 2. Narrow Transformation 체인 — 실제 실행 안 됨 (Lazy)
evens = numbers.filter(lambda x: x % 2 == 0)   # filter: narrow
doubled = evens.map(lambda x: x * 2)            # map: narrow
# 위 두 연산은 같은 Stage에 묶인다

# 3. Wide Transformation — 새로운 Stage 생성
pairs = doubled.map(lambda x: (x % 10, x))      # narrow
grouped = pairs.groupByKey()                      # wide: Stage 경계

# 4. Action 호출 → DAG 생성 → 실제 실행 트리거
result = grouped.mapValues(list).collect()

for key, values in sorted(result):
    print(f"Key {key}: {sorted(values)[:3]}...")  # 처음 3개만 출력

# Lineage 확인
print("\n=== RDD Lineage ===")
print(grouped.toDebugString().decode('utf-8'))

spark.stop()
```

실행 결과에서 `toDebugString()`을 보면 RDD의 계보가 트리 형태로 출력된다. 들여쓰기 단계가 늘어날수록 Stage 경계를 뜻한다.

### 예제 2: Scala로 단어 빈도 계산과 Stage 최적화

`groupByKey` 대신 `reduceByKey`를 사용하면 셔플 전에 로컬에서 미리 집계(Map-side reduce)하여 네트워크 전송량을 대폭 줄일 수 있다.

```scala
import org.apache.spark.{SparkConf, SparkContext}

object WordCountOptimized {
  def main(args: Array[String]): Unit = {
    val conf = new SparkConf().setAppName("WordCount").setMaster("local[*]")
    val sc = new SparkContext(conf)

    val text = sc.parallelize(Seq(
      "spark rdd dag is powerful",
      "spark uses dag for optimization",
      "rdd lineage enables fault tolerance",
      "dag scheduler creates stages from rdd"
    ), numSlices = 2)

    // ❌ 비효율적: groupByKey는 모든 값을 셔플한 후 집계
    // val badCount = text.flatMap(_.split(" "))
    //   .map((_, 1))
    //   .groupByKey()
    //   .mapValues(_.sum)

    // ✅ 효율적: reduceByKey는 각 파티션에서 먼저 합산 후 셔플
    val wordCount = text
      .flatMap(_.split(" "))    // Narrow: Stage 1
      .map((_, 1))              // Narrow: Stage 1
      .reduceByKey(_ + _)       // Wide: Stage 1 → 2 (셔플)
      .sortBy(_._2, ascending = false)  // Wide: Stage 2 → 3

    wordCount.collect().foreach { case (word, count) =>
      println(f"$word%-20s $count%d")
    }

    // 결과를 캐시 — 이후 액션에서 재계산 없이 재사용
    val cached = wordCount.cache()
    println(s"\n총 고유 단어 수: ${cached.count()}")
    println(s"가장 빈번한 단어: ${cached.first()._1}")

    sc.stop()
  }
}
```

`reduceByKey`는 각 파티션에서 먼저 로컬 집계를 수행한 후 셔플하므로, `groupByKey` + `mapValues(_.sum)` 조합보다 네트워크 트래픽이 훨씬 적다. 이는 Spark 성능 최적화의 가장 기본적인 원칙이다.

## 5. 주의사항 및 고급 팁

### 셔플 최소화가 핵심

셔플은 네트워크 I/O와 디스크 I/O를 유발하는 가장 비싼 연산이다. 다음을 항상 고려하라.

- `groupByKey` 대신 `reduceByKey`나 `aggregateByKey` 사용
- `join` 전에 작은 RDD를 `broadcast`로 배포하여 Broadcast Join 활용
- `repartition`(셔플 발생)보다 `coalesce`(셔플 없음, 파티션 수 줄이기만 가능) 선호

### 캐싱 전략

RDD를 여러 액션에서 재사용할 때는 `cache()`(메모리) 또는 `persist(StorageLevel.MEMORY_AND_DISK)`(메모리+디스크)를 호출하라. 단, 더 이상 필요 없는 RDD는 `unpersist()`로 메모리를 해제해야 한다.

### DataFrame/Dataset API 우선 사용

RDD는 Spark의 저수준 API다. 실제 프로덕션 코드에서는 Spark SQL의 Catalyst 옵티마이저가 자동으로 최적화해주는 **DataFrame/Dataset API**를 사용하는 것이 대부분의 경우 더 낫다. Catalyst는 쿼리 플랜을 재작성하고, Tungsten 엔진은 코드를 JVM 바이트코드로 직접 생성하여 성능을 더 끌어올린다.

### 파티션 수 조정

파티션 수가 너무 적으면 병렬성이 낮고, 너무 많으면 태스크 관리 오버헤드가 크다. 일반적으로 클러스터 코어 수의 2~4배를 권장하며, 셔플 파티션 수는 `spark.sql.shuffle.partitions`로 조정한다(기본값 200).

## 참고 자료

- [RDD Programming Guide - Apache Spark 공식 문서](https://spark.apache.org/docs/latest/rdd-programming-guide.html)
- [Understanding your Spark Application Through Visualization - Databricks](https://www.databricks.com/blog/2015/06/22/understanding-your-spark-application-through-visualization.html)
- [DAGScheduler.scala - Apache Spark GitHub](https://github.com/apache/spark/blob/master/core/src/main/scala/org/apache/spark/scheduler/DAGScheduler.scala)
- [Apache Spark RDD API ScalaDoc](https://spark.apache.org/docs/latest/api/scala/org/apache/spark/rdd/index.html)
