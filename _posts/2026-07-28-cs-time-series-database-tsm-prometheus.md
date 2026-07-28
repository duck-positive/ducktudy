---
layout: post
title: "시계열 데이터베이스(TSDB) 내부 구조 완전 정복: TSM Tree, 델타 인코딩, Prometheus TSDB 청크 설계"
date: 2026-07-28
categories: [cs, computer-science]
tags: [tsdb, time-series, influxdb, prometheus, tsm-tree, compression, storage-engine, monitoring]
---

모니터링 시스템, IoT 플랫폼, 금융 데이터 파이프라인의 심장에는 시계열 데이터베이스(TSDB)가 있습니다. InfluxDB, Prometheus, TimescaleDB, Apache Druid — 이 시스템들은 왜 일반 관계형 데이터베이스를 사용하지 않을까요? 그 답은 시계열 데이터의 본질적인 특성에 있습니다. 이 글에서는 TSDB가 데이터를 저장하고, 압축하고, 쿼리하는 내부 원리를 분해합니다.

## 왜 시계열 데이터는 특별한가

시계열 데이터는 세 가지 독특한 특성을 가집니다.

**1. 쓰기 집중적이고 순서가 있다**: CPU 사용률, 네트워크 패킷 수, 온도 센서 값은 항상 현재 시각에 추가(append)됩니다. 무작위 위치에 삽입하는 일이 거의 없습니다.

**2. 최근 데이터에 대한 읽기가 집중된다**: 대부분의 쿼리는 "최근 1시간의 평균 CPU"처럼 최신 시간 범위를 대상으로 합니다. 오래된 데이터는 점점 덜 조회됩니다.

**3. 시간과 값 사이에 강한 패턴이 있다**: 타임스탬프는 단조 증가하며, 값은 천천히 변하는 경우가 많습니다(온도, CPU 사용률 등). 이 패턴을 활용하면 극도로 높은 압축률을 달성할 수 있습니다.

일반 B-Tree 기반 RDB는 이 특성에 최적화되어 있지 않습니다. 새 행이 인덱스 어디에 삽입될지 예측할 수 없어 쓰기 성능이 낮고, 수십억 개의 타임스탬프 저장에 많은 공간을 낭비합니다.

## InfluxDB TSM (Time-Structured Merge Tree)

InfluxDB는 LSM 트리를 시계열에 특화한 **TSM(Time-Structured Merge Tree)** 스토리지 엔진을 사용합니다. LSM을 이미 알고 있다면 TSM은 LSM의 변형이라고 이해하면 됩니다.

### TSM 스토리지 레이어 구성

```
쓰기 경로:
  클라이언트 → WAL (Write-Ahead Log)
                ↓
             In-Memory Cache (Series별 버퍼)
                ↓ (주기적 flush)
             TSM 파일들 (불변, 시간 범위별 구성)
                ↓ (compaction)
             더 큰 TSM 파일
```

**WAL**: 내구성을 위해 쓰기는 먼저 순차 파일인 WAL에 기록됩니다.

**In-Memory Cache**: 최근 데이터를 메모리에 유지해 빠른 읽기를 지원합니다.

**TSM 파일**: LSM의 SSTable처럼 불변(immutable)입니다. 각 TSM 파일은 다음 섹션으로 구성됩니다.
- Header (magic number + version)
- 압축된 블록들 (Series Key별로 묶인 데이터 포인트)
- Index (각 블록의 오프셋과 시간 범위)
- Footer

### TSI (Time Series Index)

시계열 데이터는 `measurement + tag set + field`로 식별됩니다. 예를 들어 `cpu,host=server1,region=us-east usage=73.2 1627000000000000000`이라는 포인트에서 `cpu,host=server1,region=us-east`가 Series Key입니다.

TSI는 이 Series Key들을 효율적으로 인덱싱하여 `WHERE region='us-east'` 같은 태그 필터를 빠르게 처리합니다. 내부적으로는 역색인(Inverted Index) 구조를 사용합니다.

## 시계열 압축 알고리즘

TSDB의 진짜 마법은 압축에 있습니다. Prometheus는 일반적으로 데이터 포인트당 약 **1.37바이트**만 사용합니다. 타임스탬프(8바이트) + 값(8바이트) = 16바이트짜리 데이터를 1.37바이트로! 어떻게 가능할까요?

### 1. 타임스탬프 델타-of-델타 인코딩

Facebook의 Gorilla TSDB 논문에서 제안된 기법입니다.

```
원본 타임스탬프: [1000, 1015, 1030, 1045, 1060]
델타(1차 차분): [1000, 15, 15, 15, 15]
델타-of-델타(2차 차분): [1000, 15, 0, 0, 0]
```

수집 간격이 일정하면 2차 차분의 대부분이 0이 됩니다. 이를 가변 길이 비트 인코딩으로 저장하면 극도로 높은 압축률을 달성합니다.

| 델타-of-델타 값 | 비트 표현 |
|---|---|
| 0 | `0` (1비트) |
| -63 ~ 64 | `10` + 7비트 |
| -255 ~ 256 | `110` + 9비트 |
| 나머지 | `1110` + 12비트 |

### 2. 부동소수점 XOR 압축 (Gorilla 알고리즘)

값의 경우, 연속된 두 float64 값을 XOR하면 값이 천천히 변할 때 결과에 많은 0비트가 나타납니다.

```
v1 = 73.2  = 0x4052333333333333 (IEEE 754)
v2 = 73.5  = 0x4052600000000000
v1 XOR v2  = 0x0000133333333333  (앞쪽에 0이 많음)
```

XOR 결과에서 앞쪽 제로 비트 수(leading zeros)와 뒤쪽 제로 비트 수(trailing zeros)를 기록하고, 의미 있는 비트만 저장합니다.

## 실제 구현 예제

### 예제 1: Python으로 시계열 청크와 델타 인코딩 구현

```python
import struct
import time
from typing import List, Tuple

class TSDBChunk:
    """단순화된 시계열 청크 구현 (델타-of-델타 타임스탬프 인코딩)"""
    
    def __init__(self):
        self.points: List[Tuple[int, float]] = []  # (timestamp_ms, value)
        self._compressed_timestamps: List[int] = []  # 델타-of-델타
        self._prev_ts = 0
        self._prev_delta = 0
    
    def append(self, ts_ms: int, value: float):
        self.points.append((ts_ms, value))
        
        delta = ts_ms - self._prev_ts
        dod = delta - self._prev_delta  # 델타-of-델타
        
        self._compressed_timestamps.append(dod)
        self._prev_ts = ts_ms
        self._prev_delta = delta
    
    def encode_varint_dod(self) -> bytes:
        """델타-of-델타를 가변 길이 정수로 인코딩"""
        result = bytearray()
        for dod in self._compressed_timestamps:
            if dod == 0:
                result.extend(b'\x00')  # 1바이트
            elif -63 <= dod <= 64:
                # 2비트 헤더 + 7비트 값
                encoded = (0b10 << 7) | (dod & 0x7F)
                result.extend(struct.pack('>H', encoded))
            else:
                # 4비트 헤더 + 부호 있는 32비트
                result.extend(b'\xe0')
                result.extend(struct.pack('>i', dod))
        return bytes(result)
    
    def compress_xor_float(self) -> List[int]:
        """연속된 float 값에 XOR 압축 적용 (비트 표현)"""
        if not self.points:
            return []
        
        result = []
        prev_bits = 0
        for _, value in self.points:
            bits = struct.unpack('Q', struct.pack('d', value))[0]
            xor = bits ^ prev_bits
            result.append(xor)
            prev_bits = bits
        return result
    
    def compression_ratio(self) -> float:
        """압축 전후 크기 비율"""
        raw_size = len(self.points) * 16  # 8B timestamp + 8B float
        
        ts_encoded = self.encode_varint_dod()
        xor_vals = self.compress_xor_float()
        
        # XOR 값 중 0의 비율 (0 = 값이 변하지 않음)
        zero_xor = sum(1 for x in xor_vals if x == 0)
        estimated_compressed_size = len(ts_encoded) + len(xor_vals) * 2  # 근사치
        
        return raw_size / max(estimated_compressed_size, 1)


def simulate_metrics():
    chunk = TSDBChunk()
    base_ts = int(time.time() * 1000)
    
    # 15초 간격 CPU 사용률 시뮬레이션 (천천히 변동)
    cpu_usage = 45.0
    for i in range(100):
        ts = base_ts + i * 15000  # 15초 간격
        cpu_usage += (hash(i) % 5 - 2) * 0.1  # ±0.2% 변동
        chunk.append(ts, round(cpu_usage, 2))
    
    ts_encoded = chunk.encode_varint_dod()
    xor_vals = chunk.compress_xor_float()
    
    raw_bytes = len(chunk.points) * 16
    zero_xor_count = sum(1 for x in xor_vals if x == 0)
    
    print(f"데이터 포인트 수: {len(chunk.points)}")
    print(f"원본 크기: {raw_bytes} bytes")
    print(f"타임스탬프 인코딩 크기: {len(ts_encoded)} bytes")
    print(f"XOR 제로 값 비율: {zero_xor_count}/{len(xor_vals)} = {zero_xor_count/len(xor_vals)*100:.1f}%")
    print(f"예상 압축률: {chunk.compression_ratio():.1f}x")
    
    # 첫 5개 델타-of-델타 확인
    print("\n첫 5개 델타-of-델타 값:")
    for i, dod in enumerate(chunk._compressed_timestamps[:5]):
        print(f"  [{i}] = {dod}")


if __name__ == "__main__":
    simulate_metrics()
```

실행하면 100개 포인트의 타임스탬프가 대부분 0(= 일정 간격)으로 인코딩되어 극소량의 바이트만 사용하는 것을 확인할 수 있습니다.

### 예제 2: Go로 Prometheus 스타일 TSDB 샤드와 청크 관리 구현

```go
package main

import (
    "fmt"
    "math"
    "sort"
    "time"
)

// Series는 단일 시계열 (metric + label set)
type Series struct {
    Labels map[string]string
    Chunks []*Chunk
}

// Chunk는 고정 시간 범위의 데이터 블록 (Prometheus는 기본 2시간)
type Chunk struct {
    MinTime int64
    MaxTime int64
    Points  []Point
}

type Point struct {
    Timestamp int64   // Unix milliseconds
    Value     float64
}

// Shard는 특정 시간 범위를 담당하는 저장 단위
type Shard struct {
    MinTime int64
    MaxTime int64
    Series  map[string]*Series
}

func NewShard(minTime, maxTime int64) *Shard {
    return &Shard{
        MinTime: minTime,
        MaxTime: maxTime,
        Series:  make(map[string]*Series),
    }
}

func labelsToKey(labels map[string]string) string {
    keys := make([]string, 0, len(labels))
    for k := range labels {
        keys = append(keys, k)
    }
    sort.Strings(keys)
    key := ""
    for _, k := range keys {
        key += k + "=" + labels[k] + ","
    }
    return key
}

func (s *Shard) Append(labels map[string]string, ts int64, val float64) {
    key := labelsToKey(labels)
    series, ok := s.Series[key]
    if !ok {
        series = &Series{Labels: labels, Chunks: []*Chunk{}}
        s.Series[key] = series
    }

    // 현재 청크가 없거나 가득 찼으면 새 청크 생성
    chunkDuration := int64(2 * 60 * 60 * 1000) // 2시간 (ms)
    if len(series.Chunks) == 0 ||
        ts-series.Chunks[len(series.Chunks)-1].MinTime >= chunkDuration {
        series.Chunks = append(series.Chunks, &Chunk{
            MinTime: ts,
            MaxTime: ts,
        })
    }

    chunk := series.Chunks[len(series.Chunks)-1]
    chunk.Points = append(chunk.Points, Point{Timestamp: ts, Value: val})
    if ts > chunk.MaxTime {
        chunk.MaxTime = ts
    }
}

// RangeQuery는 시간 범위 쿼리 실행
func (s *Shard) RangeQuery(
    labelFilter map[string]string,
    startMs, endMs int64,
) []Point {
    results := []Point{}
    for _, series := range s.Series {
        if !matchLabels(series.Labels, labelFilter) {
            continue
        }
        for _, chunk := range series.Chunks {
            // 청크 시간 범위와 겹치지 않으면 스킵 (블록 pruning)
            if chunk.MaxTime < startMs || chunk.MinTime > endMs {
                continue
            }
            for _, p := range chunk.Points {
                if p.Timestamp >= startMs && p.Timestamp <= endMs {
                    results = append(results, p)
                }
            }
        }
    }
    sort.Slice(results, func(i, j int) bool {
        return results[i].Timestamp < results[j].Timestamp
    })
    return results
}

func matchLabels(series, filter map[string]string) bool {
    for k, v := range filter {
        if series[k] != v {
            return false
        }
    }
    return true
}

// Downsample: 범위 내 포인트를 평균으로 다운샘플링
func Downsample(points []Point, windowMs int64) []Point {
    if len(points) == 0 {
        return nil
    }
    result := []Point{}
    windowStart := points[0].Timestamp
    sum, count := 0.0, 0

    for _, p := range points {
        if p.Timestamp >= windowStart+windowMs {
            if count > 0 {
                result = append(result, Point{
                    Timestamp: windowStart + windowMs/2,
                    Value:     math.Round(sum/float64(count)*100) / 100,
                })
            }
            windowStart = p.Timestamp
            sum, count = 0, 0
        }
        sum += p.Value
        count++
    }
    if count > 0 {
        result = append(result, Point{
            Timestamp: windowStart + windowMs/2,
            Value:     math.Round(sum/float64(count)*100) / 100,
        })
    }
    return result
}

func main() {
    now := time.Now().UnixMilli()
    shardDuration := int64(24 * 60 * 60 * 1000) // 24시간
    shard := NewShard(now, now+shardDuration)

    labels := map[string]string{"job": "node_exporter", "instance": "server1:9100"}

    // 5분 간격으로 6시간치 CPU 데이터 삽입
    interval := int64(5 * 60 * 1000) // 5분 (ms)
    for i := 0; i < 72; i++ {
        ts := now + int64(i)*interval
        cpu := 30.0 + float64(i%20)*1.5 + float64(hash(i)%10)*0.3
        shard.Append(labels, ts, cpu)
    }

    // 최근 1시간 쿼리
    startMs := now + 5*int64(60*60*1000)
    endMs := now + 6*int64(60*60*1000)
    points := shard.RangeQuery(labels, startMs, endMs)
    fmt.Printf("최근 1시간 데이터 포인트 수: %d\n", len(points))

    // 15분 단위 다운샘플링
    downsampled := Downsample(points, 15*60*1000)
    fmt.Printf("15분 다운샘플링 후 포인트 수: %d\n", len(downsampled))
    for _, p := range downsampled {
        fmt.Printf("  ts=%d value=%.2f\n", p.Timestamp-now, p.Value)
    }

    // 청크 현황
    key := labelsToKey(labels)
    series := shard.Series[key]
    fmt.Printf("\n총 청크 수: %d\n", len(series.Chunks))
    for i, c := range series.Chunks {
        fmt.Printf("  청크[%d]: %d포인트, 시간범위=%dms\n",
            i, len(c.Points), c.MaxTime-c.MinTime)
    }
}

func hash(n int) int {
    n = n*2654435769 + 0x9e3779b9
    return n & 0x7FFFFFFF
}
```

## Prometheus TSDB의 설계 철학

Prometheus 2.0에서 완전히 재작성된 TSDB는 다음 원칙을 따릅니다.

**블록 기반 구조**: 디스크는 2시간짜리 블록들로 구성됩니다. 각 블록은 독립적이며, 쿼리 시 필요한 블록만 열어 시간 범위 필터링을 블록 단위로 수행합니다(Block Pruning). 오래된 블록 삭제가 단순히 디렉토리 삭제로 구현됩니다.

**헤드 블록(Head Block)**: 최근 2시간의 데이터는 메모리와 WAL에 유지됩니다. 이 헤드 블록은 2시간마다 디스크 블록으로 flush됩니다.

**Compaction**: 여러 작은 블록들이 주기적으로 더 큰 블록으로 병합됩니다. Compaction은 중복 제거와 청크 정렬을 수행해 읽기 성능을 향상시킵니다.

## 주의사항과 팁

**1. Cardinality 폭발 문제**: `user_id`, `request_id` 같은 고유값 레이블을 태그로 사용하면 Series 수가 수백만 개로 폭발합니다. InfluxDB와 Prometheus는 이를 "High Cardinality" 문제라 부르며, TSI 인덱스가 디스크를 가득 채우는 주요 원인입니다.

**2. 원거리(Out-of-Order) 쓰기**: 시간 순서를 어기는 포인트가 들어오면 청크 구조가 복잡해집니다. Prometheus는 기본적으로 과거 5분 이내의 out-of-order 쓰기만 허용합니다.

**3. 보존 정책(Retention)과 다운샘플링**: 오래된 데이터는 낮은 해상도로 다운샘플링하여 저장 공간을 절약하세요. InfluxDB의 Continuous Query나 Prometheus의 Recording Rule이 이를 자동화합니다.

**4. 쓰기 배치 처리**: 포인트를 하나씩 쓰지 말고 배치로 묶어서 보내세요. WAL의 fsync 비용이 지배적이기 때문에 배치 크기가 커질수록 처리량이 급격히 향상됩니다.

## 참고 자료
- [InfluxDB OSS v2 Storage Engine 공식 문서](https://docs.influxdata.com/influxdb/v2/reference/internals/storage-engine/)
- [In-memory indexing and the TSM Tree — InfluxDB v1](https://docs.influxdata.com/influxdb/v1/concepts/storage_engine/)
- [The Database Zoo: Inside Time-Series Engines](https://www.mexc.co/news/106332)
- [Analysis of the Storage Mechanism in InfluxDB — Medium](https://medium.com/dataseries/analysis-of-the-storage-mechanism-in-influxdb-b84d686f3697)
