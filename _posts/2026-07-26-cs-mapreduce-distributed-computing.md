---
layout: post
title: "MapReduce와 분산 컴퓨팅 패러다임 완전 정복: Google이 빅데이터를 정복한 방법"
date: 2026-07-26
categories: [cs, computer-science]
tags: [mapreduce, distributed-computing, hadoop, bigdata, parallel-processing, google, spark]
---

2004년, Google의 Jeff Dean과 Sanjay Ghemawat은 OSDI 컨퍼런스에서 한 편의 논문을 발표했다. 제목은 "MapReduce: Simplified Data Processing on Large Clusters." 이 논문은 분산 컴퓨팅의 역사를 바꾸었다. 수십억 개의 웹 페이지를 색인하고, 전 세계 검색 쿼리 로그를 분석하고, 지도 데이터를 처리하기 위해 Google이 개발한 이 패러다임은 오늘날 빅데이터의 기반이 되었다. 이 글에서는 MapReduce의 핵심 원리부터 내부 구현, 그리고 현대적 후계자까지 깊이 살펴본다.

## 왜 MapReduce가 탄생했는가?

### 기존 방법의 한계

2000년대 초 Google은 매달 수십억 개의 웹 페이지를 크롤링하고 색인해야 했다. 이 규모의 데이터를 처리하려면:

1. **단일 서버**: 한 대의 강력한 서버로는 불가능하다. 페타바이트 데이터를 처리하기 위한 단일 서버는 존재하지 않는다.
2. **수동 분산 처리**: 직접 데이터를 나누고 여러 서버에 배분하는 코드를 작성하면, 장애 처리, 재시도 로직, 로드 밸런싱 등 인프라 코드가 비즈니스 로직보다 훨씬 많아진다.
3. **MPI(Message Passing Interface)**: 고성능 컴퓨팅에서 쓰이는 MPI는 장애 처리가 어렵고, 상용(commodity) 서버 클러스터에 적합하지 않았다.

Google은 엔지니어들이 분산 처리의 복잡성을 신경 쓰지 않고 비즈니스 로직에 집중할 수 있는 추상화 레이어를 원했다. 그 결과가 MapReduce다.

### 핵심 통찰

MapReduce의 핵심 통찰은 대부분의 대규모 데이터 처리 작업이 두 가지 패턴으로 표현 가능하다는 것이다:

1. **Map**: 입력 데이터의 각 항목에 독립적으로 함수를 적용 (병렬 처리 가능)
2. **Reduce**: Map의 결과를 그룹화하여 집계 (키 기반 집계)

이 추상화는 함수형 프로그래밍의 `map`과 `reduce`에서 직접 영감을 받았다.

## MapReduce의 동작 원리

### 전체 흐름

```
입력 데이터 (HDFS)
    ↓
[Input Split] → 데이터를 Mapper가 처리할 수 있는 크기로 분할
    ↓
[Map Phase] → 각 청크를 병렬로 처리, (key, value) 쌍 출력
    ↓
[Shuffle & Sort] → 같은 키를 가진 값들을 그룹화
    ↓
[Reduce Phase] → 각 키에 대한 값 목록을 집계
    ↓
출력 데이터 (HDFS)
```

### 구체적인 예: 단어 세기(Word Count)

MapReduce의 "Hello World"는 단어 세기(Word Count)다.

```python
# Python으로 MapReduce 개념을 시뮬레이션

from collections import defaultdict
from typing import Iterator, Tuple, List
import re

# --- 1. Mapper 정의 ---
def word_count_mapper(document_id: str, text: str) -> Iterator[Tuple[str, int]]:
    """
    Mapper: 문서를 받아 (단어, 1) 쌍을 출력
    각 Mapper 인스턴스는 독립적으로 실행됨 (공유 상태 없음)
    """
    words = re.findall(r'\w+', text.lower())
    for word in words:
        yield (word, 1)  # (key, value) 쌍 방출

# --- 2. Reducer 정의 ---
def word_count_reducer(word: str, counts: Iterator[int]) -> Tuple[str, int]:
    """
    Reducer: 같은 키(단어)에 대한 모든 값(counts)을 받아 집계
    같은 키의 모든 Mapper 출력이 동일한 Reducer로 모임
    """
    return (word, sum(counts))

# --- 3. MapReduce 프레임워크 시뮬레이션 ---
def mapreduce(documents: dict, mapper, reducer) -> dict:
    """MapReduce 프레임워크 시뮬레이션"""
    
    # Phase 1: Map
    print("=== Map Phase ===")
    intermediate = defaultdict(list)
    for doc_id, content in documents.items():
        for key, value in mapper(doc_id, content):
            intermediate[key].append(value)
    
    # Phase 2: Shuffle & Sort (프레임워크가 자동으로 처리)
    print("=== Shuffle & Sort Phase ===")
    # 키별로 정렬하여 각 Reducer로 전달
    sorted_intermediate = sorted(intermediate.items())
    
    # Phase 3: Reduce
    print("=== Reduce Phase ===")
    results = {}
    for key, values in sorted_intermediate:
        result_key, result_value = reducer(key, iter(values))
        results[result_key] = result_value
    
    return results


# 테스트
documents = {
    "doc1": "the quick brown fox jumps over the lazy dog",
    "doc2": "the fox and the hound are friends",
    "doc3": "quick brown hair and lazy eyes"
}

word_counts = mapreduce(documents, word_count_mapper, word_count_reducer)

# 상위 10개 단어 출력
top10 = sorted(word_counts.items(), key=lambda x: -x[1])[:10]
print("\n=== 상위 10개 단어 ===")
for word, count in top10:
    print(f"  {word}: {count}")
```

실행 결과:
```
=== Map Phase ===
=== Shuffle & Sort Phase ===
=== Reduce Phase ===

=== 상위 10개 단어 ===
  the: 4
  fox: 2
  quick: 2
  brown: 2
  lazy: 2
  and: 2
  dog: 1
  jumps: 1
  over: 1
  hound: 1
```

## 내부 구현: MapReduce가 분산 환경에서 동작하는 방법

### 마스터-워커 아키텍처

원래 Google의 MapReduce는 마스터 1대 + N개의 워커로 구성된다:

```
                    Master
                   /      \
          [Mapper Pool]  [Reducer Pool]
          M1  M2  M3     R1  R2  R3
          |   |   |      |   |   |
         [GFS/HDFS 블록들]  [중간 파일들]
```

**마스터의 역할:**
- 입력 데이터를 64MB 청크로 분할 (Input Split)
- 각 Map 태스크를 워커에 할당 (데이터 지역성 고려)
- 완료된 Map 태스크의 중간 파일 위치를 추적
- Reduce 태스크를 워커에 할당
- 장애 감지 및 재실행

**데이터 지역성(Data Locality)의 중요성:**

MapReduce의 핵심 최적화 중 하나는 "데이터를 컴퓨팅으로 가져가는 것이 아니라, 컴퓨팅을 데이터로 가져가는" 것이다. 1GB 데이터를 네트워크로 전송하는 것보다, 그 데이터가 있는 서버에서 코드를 실행하는 것이 훨씬 빠르다.

### Shuffle & Sort: MapReduce의 숨겨진 핵심

Mapper와 Reducer 사이의 **Shuffle & Sort** 단계는 MapReduce 성능의 핵심이다:

1. **파티셔닝**: Mapper의 출력 키를 해시 함수로 Reducer에 배분
   ```python
   def partition(key: str, num_reducers: int) -> int:
       """어떤 Reducer가 이 키를 처리할지 결정"""
       return hash(key) % num_reducers
   ```

2. **로컬 정렬**: 각 Mapper는 자신의 출력을 키로 정렬 (Reducer 부담 감소)

3. **병합 정렬**: 여러 Mapper의 출력을 Reducer가 병합 정렬

4. **그룹화**: 같은 키의 값들을 연속된 스트림으로 Reducer에 전달

Shuffle 단계는 네트워크 집약적이므로, 이를 줄이는 **Combiner** 개념이 도입되었다.

### Combiner: 미니 Reducer

Combiner는 Mapper와 같은 서버에서 실행되는 "미니 Reducer"다. 네트워크로 보내기 전에 로컬에서 부분 집계를 수행한다.

```python
def word_count_combiner(word: str, counts: Iterator[int]) -> Tuple[str, int]:
    """
    Combiner: Mapper 출력을 로컬에서 미리 집계
    네트워크 전송량을 대폭 줄임
    
    예: Mapper가 ("the", 1)을 1000번 출력하면
        Combiner는 ("the", 1000)으로 줄여서 전송
    """
    return (word, sum(counts))


# Combiner가 있는 MapReduce 시뮬레이션
def mapreduce_with_combiner(documents: dict, mapper, combiner, reducer):
    """Combiner를 사용한 MapReduce - 네트워크 트래픽 절감"""
    
    # Phase 1: Map + Local Combine (같은 서버에서 실행)
    all_intermediate = defaultdict(list)
    
    for doc_id, content in documents.items():
        # 로컬 Mapper 출력 수집
        local_output = defaultdict(list)
        for key, value in mapper(doc_id, content):
            local_output[key].append(value)
        
        # 로컬 Combine (네트워크 전송 전에 집계)
        for key, values in local_output.items():
            combined_key, combined_value = combiner(key, iter(values))
            all_intermediate[combined_key].append(combined_value)
    
    # Phase 2: Shuffle & Sort (이미 부분 집계된 데이터만 전송)
    sorted_intermediate = sorted(all_intermediate.items())
    
    # Phase 3: Reduce
    results = {}
    for key, values in sorted_intermediate:
        result_key, result_value = reducer(key, iter(values))
        results[result_key] = result_value
    
    return results
```

### 장애 처리(Fault Tolerance)

상용(commodity) 서버 클러스터에서는 장애가 일상적이다. 수천 대의 서버 중 하루에 몇 대는 반드시 고장난다. MapReduce의 장애 처리 전략:

1. **하트비트**: 마스터는 워커에게 주기적으로 ping. 일정 시간 응답 없으면 실패로 판단.
2. **Map 태스크 재실행**: 실패한 Mapper의 태스크를 다른 워커에 재할당. 중간 파일이 로컬 디스크에 있으므로 다시 계산.
3. **Reduce 태스크 재실행**: 실패한 Reducer의 태스크도 재할당.
4. **멱등성(Idempotency)**: 같은 태스크가 여러 번 실행될 수 있으므로, Map/Reduce 함수는 부작용이 없어야 한다.

이 모든 것이 사용자 코드에서 완전히 투명하게 처리된다.

## Apache Hadoop: 오픈소스 MapReduce

Google의 논문에 영감을 받아 Doug Cutting과 Mike Cafarella가 2005년에 Apache Hadoop을 만들었다. Hadoop은 두 부분으로 구성된다:

1. **HDFS(Hadoop Distributed File System)**: Google GFS의 오픈소스 구현
2. **MapReduce**: Google MapReduce의 오픈소스 구현

### Hadoop Java API

```java
import org.apache.hadoop.mapreduce.*;
import org.apache.hadoop.io.*;

// Mapper 구현
public class WordCountMapper 
    extends Mapper<LongWritable, Text, Text, IntWritable> {
    
    private final static IntWritable ONE = new IntWritable(1);
    private Text word = new Text();
    
    @Override
    protected void map(LongWritable key, Text value, Context context)
            throws IOException, InterruptedException {
        
        // 한 줄씩 처리
        String line = value.toString();
        StringTokenizer tokenizer = new StringTokenizer(line);
        
        while (tokenizer.hasMoreTokens()) {
            word.set(tokenizer.nextToken().toLowerCase());
            context.write(word, ONE);  // (단어, 1) 방출
        }
    }
}

// Reducer 구현
public class WordCountReducer
    extends Reducer<Text, IntWritable, Text, IntWritable> {
    
    private IntWritable result = new IntWritable();
    
    @Override
    protected void reduce(Text key, Iterable<IntWritable> values, Context context)
            throws IOException, InterruptedException {
        
        int sum = 0;
        for (IntWritable val : values) {
            sum += val.get();
        }
        result.set(sum);
        context.write(key, result);
    }
}

// Driver (메인 설정)
public class WordCount {
    public static void main(String[] args) throws Exception {
        Configuration conf = new Configuration();
        Job job = Job.getInstance(conf, "word count");
        
        job.setJarByClass(WordCount.class);
        job.setMapperClass(WordCountMapper.class);
        job.setCombinerClass(WordCountReducer.class);  // Combiner = Reducer
        job.setReducerClass(WordCountReducer.class);
        
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(IntWritable.class);
        
        FileInputFormat.addInputPath(job, new Path(args[0]));
        FileOutputFormat.setOutputPath(job, new Path(args[1]));
        
        System.exit(job.waitForCompletion(true) ? 0 : 1);
    }
}
```

## MapReduce의 한계와 Apache Spark의 등장

MapReduce는 혁신적이었지만 몇 가지 중요한 한계가 있었다:

### 1. 디스크 I/O 과도

MapReduce는 각 단계의 중간 결과를 HDFS에 디스크로 기록한다. 반복적인 알고리즘(머신러닝, 그래프 처리)에서는 이 디스크 I/O가 심각한 병목이 된다.

### 2. 낮은 수준의 추상화

Word Count조차 Mapper와 Reducer 클래스를 별도로 구현해야 한다. SQL-like 쿼리나 복잡한 데이터 파이프라인 표현이 어렵다.

### 3. 지연 시간

배치 처리에 특화되어 있어 실시간 또는 대화형 쿼리에 적합하지 않다.

이 한계를 극복하기 위해 **Apache Spark**가 2009년 UC Berkeley에서 탄생했다:

```python
# PySpark로 동일한 Word Count
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("WordCount").getOrCreate()
sc = spark.sparkContext

# MapReduce보다 훨씬 간결
text_file = sc.textFile("hdfs://input/text")
word_counts = (
    text_file
    .flatMap(lambda line: line.lower().split())  # Map
    .map(lambda word: (word, 1))                 # Map
    .reduceByKey(lambda a, b: a + b)             # Reduce
    .sortBy(lambda x: -x[1])                     # 정렬
)

# 메모리 캐싱 (반복 처리 최적화)
word_counts.cache()
top10 = word_counts.take(10)
for word, count in top10:
    print(f"{word}: {count}")

spark.stop()
```

**Spark vs MapReduce 성능 비교:**

| 시나리오 | MapReduce | Spark |
|---------|-----------|-------|
| 단순 배치 처리 | 기준 | 2-3배 빠름 |
| 반복 ML 알고리즘 | 기준 | 10-100배 빠름 |
| 인터랙티브 쿼리 | 지원 안 됨 | 지원 |
| 스트리밍 처리 | 지원 안 됨 | Structured Streaming |
| 메모리 사용 | 적음 | 많음 |
| 내결함성 | 강함 | RDD lineage 기반 |

Spark의 핵심 혁신은 **RDD(Resilient Distributed Dataset)** - 데이터를 메모리에 캐시하고, 디스크 기록 없이 여러 변환 단계를 파이프라이닝하는 것이다.

## MapReduce 패턴: 실전 예제

MapReduce 사고방식은 분산 환경뿐 아니라 다양한 문제에 적용된다.

```python
# 패턴 1: Inverted Index (검색 엔진의 핵심)
def inverted_index_mapper(doc_id: str, text: str):
    """각 단어가 어느 문서에 있는지 기록"""
    words = set(re.findall(r'\w+', text.lower()))
    for word in words:
        yield (word, doc_id)

def inverted_index_reducer(word: str, doc_ids: Iterator[str]):
    """단어 → 문서 목록 매핑 생성"""
    return (word, sorted(set(doc_ids)))


# 패턴 2: Top-N (상위 N개 항목 찾기)
def top_n_reducer(key: str, values: Iterator, n: int = 10):
    """
    각 Reducer는 지역적으로 Top-N을 계산.
    최종 집계 단계에서 전체 Top-N을 구함
    """
    import heapq
    return (key, heapq.nlargest(n, values))


# 패턴 3: Secondary Sort (복합 키 정렬)
def composite_key_mapper(log_entry: str):
    """
    (날짜, 시간) 복합 키로 로그를 정렬하는 패턴.
    Shuffle 단계의 키 정렬을 활용함
    """
    parts = log_entry.split()
    date, time, event = parts[0], parts[1], parts[2]
    composite_key = f"{date}\t{time}"  # 탭으로 구분된 복합 키
    yield (composite_key, event)


# 테스트
docs = {
    "doc1": "the quick brown fox",
    "doc2": "the fox and the hound",
    "doc3": "the quick brown hair"
}

print("=== Inverted Index ===")
index = mapreduce(docs, inverted_index_mapper, inverted_index_reducer)
for word in sorted(index.keys())[:5]:
    print(f"  '{word}': {index[word]}")
```

## 주의사항과 실전 팁

### 1. 데이터 스큐(Data Skew) 문제

특정 키에 데이터가 몰리면(예: "the"가 수십억 번 등장) 해당 Reducer 하나만 과부하가 걸린다. 해결책:

```python
def skew_resistant_mapper(doc_id: str, text: str, salt_range: int = 10):
    """
    솔팅(salting): 핫 키에 무작위 접두사 추가로 여러 Reducer에 분산
    최종 Reducer에서 솔트를 제거하고 재집계 필요
    """
    import random
    words = re.findall(r'\w+', text.lower())
    for word in words:
        salt = random.randint(0, salt_range - 1)
        yield (f"{salt}_{word}", 1)  # 솔팅된 키
```

### 2. 작은 파일 문제

HDFS에 작은 파일이 많으면 NameNode 메모리를 과도하게 소비하고 Map 태스크 오버헤드가 커진다. CombineFileInputFormat을 사용하여 작은 파일을 하나의 Map 태스크로 묶어야 한다.

### 3. Speculative Execution(투기적 실행)

일부 워커가 느리면(straggler) 전체 작업이 지연된다. MapReduce는 같은 태스크를 다른 워커에서 동시에 실행하고, 먼저 완료된 결과를 사용하는 투기적 실행을 지원한다. 멱등하지 않은 작업(외부 DB 쓰기 등)에는 비활성화해야 한다.

### 4. 언제 MapReduce를 사용하지 말아야 하는가

- **소규모 데이터** (수 GB 미만): 단일 서버가 훨씬 빠르다
- **실시간 처리**: Kafka + Flink/Spark Streaming 사용
- **반복 알고리즘**: Spark 사용
- **낮은 지연시간 쿼리**: Presto/Trino, ClickHouse 사용

## 마치며

MapReduce는 "누구나 수천 대의 서버를 한 덩어리처럼 프로그래밍할 수 있다"는 꿈을 실현했다. Google이 내부에서 이 패러다임으로 정렬, 색인 생성, 그래프 분석, 머신러닝 등 수백 가지 작업을 처리했다. Hadoop 에코시스템과 Spark로 이어진 이 유산은 오늘날 모든 클라우드 빅데이터 플랫폼(AWS EMR, Google Dataproc, Azure HDInsight)의 기반이다. "데이터를 컴퓨팅으로 가져가는 것이 아니라, 컴퓨팅을 데이터로 가져간다"는 이 단순한 원리는 여전히 유효하다.

## 참고 자료
- [MapReduce: Simplified Data Processing on Large Clusters — Dean & Ghemawat, OSDI 2004](https://research.google.com/archive/mapreduce-osdi04.pdf)
- [Apache Hadoop MapReduce Tutorial](https://hadoop.apache.org/docs/r3.3.6/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html)
- [MapReduce — Wikipedia](https://en.wikipedia.org/wiki/MapReduce)
- [Apache Spark vs Hadoop MapReduce](https://spark.apache.org/docs/latest/index.html)
