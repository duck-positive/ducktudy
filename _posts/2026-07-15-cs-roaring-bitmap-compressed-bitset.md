---
layout: post
title: "Roaring Bitmap 완전 정복: 수십억 개의 정수를 압축하는 하이브리드 비트셋 자료구조"
date: 2026-07-15
categories: [cs, computer-science]
tags: [roaring-bitmap, bitmap, data-structure, compression, set-operations, algorithm]
---

## 개념 설명

Roaring Bitmap은 32비트 정수 집합을 저장하는 압축 비트맵(Compressed Bitmap) 자료구조로, Apache Lucene, Apache Spark, Elasticsearch, Apache Druid, InfluxDB 등 수많은 고성능 시스템에서 핵심 인덱스 자료구조로 채택되고 있다. 기존의 비트맵 방식(비트 배열)은 희소(sparse)한 집합에서 메모리를 낭비하고, WAH(Word-Aligned Hybrid)나 EWAH 같은 RLE 기반 압축은 고밀도(dense) 집합에서 성능이 떨어지는 문제를 안고 있었다. Roaring Bitmap은 이 두 세계의 장점을 결합해, 희소·고밀도·런 기반 집합 모두에서 균형 잡힌 성능을 제공한다.

### 핵심 아이디어: 16비트 분할

32비트 정수를 상위 16비트(키)와 하위 16비트(값)로 분리한다. 동일한 상위 16비트를 공유하는 값들은 하나의 **컨테이너(Container)**에 묶인다. 컨테이너는 내부 데이터 분포에 따라 세 가지 타입 중 하나를 자동으로 선택한다:

| 컨테이너 타입 | 적용 조건 | 내부 표현 |
|---|---|---|
| **Array Container** | 원소 수 ≤ 4,096 | 정렬된 16비트 정수 배열 |
| **Bitmap Container** | 원소 수 > 4,096 | 고정 크기 65,536비트 비트맵 (8KB) |
| **Run Container** | 연속 구간이 많을 때 | (시작값, 길이) 쌍의 배열 |

임계값인 4,096은 수학적으로 최적화된 값이다. Array Container 원소 1개당 2바이트, Bitmap Container는 8,192바이트 고정이므로, 4,096 × 2 = 8,192바이트가 교차점이 된다.

## 왜 필요한가

전통적인 비트맵 방식의 문제는 명확하다. 예를 들어 사용자 ID 집합 `{1, 1000000, 2000000, 3000000}`을 비트맵으로 표현하려면 3,000,001비트(약 375KB)의 배열이 필요하지만, 실제 사용 비트는 4개뿐이다. 반대로 `{0, 1, 2, ..., 65535}` 같은 연속 집합은 RLE로 `(0, 65535)` 단 1쌍으로 표현 가능하지만, 무작위 분포에서는 RLE가 오히려 팽창한다.

Roaring Bitmap은 이런 양극단을 동적으로 감지해 최적 표현을 선택한다. 또한 교집합(AND), 합집합(OR), 차집합(AND NOT), XOR 등의 집합 연산을 컨테이너 단위로 병렬화하고, SIMD 명령어를 통해 벡터 수준의 최적화까지 지원한다.

실제 적용 사례를 보면:
- **Apache Druid**: 쿼리 필터링 시 수십억 개의 행 ID를 Roaring Bitmap으로 관리해 교집합 연산으로 빠른 필터링 수행
- **Elasticsearch**: 역색인의 Posting List를 Roaring Bitmap으로 인코딩
- **Apache Spark**: `Dataset.filter()` 등의 비트 조인 최적화에 활용

## 실제 구현 예제

### 예제 1: Java에서 Roaring Bitmap 기본 사용

```java
import org.roaringbitmap.RoaringBitmap;

public class RoaringBitmapDemo {
    public static void main(String[] args) {
        // 집합 A: 활성 사용자 ID
        RoaringBitmap activeUsers = new RoaringBitmap();
        activeUsers.add(1, 100, 500, 1000, 50000, 100000);

        // 집합 B: 프리미엄 사용자 ID
        RoaringBitmap premiumUsers = new RoaringBitmap();
        premiumUsers.add(100, 500, 200000, 300000);

        // 교집합: 활성이면서 프리미엄인 사용자
        RoaringBitmap activeAndPremium = RoaringBitmap.and(activeUsers, premiumUsers);
        System.out.println("활성 + 프리미엄: " + activeAndPremium);
        // 결과: {100, 500}

        // 합집합
        RoaringBitmap allSpecial = RoaringBitmap.or(activeUsers, premiumUsers);
        System.out.println("전체 특수 사용자 수: " + allSpecial.getCardinality());

        // 범위 추가: 연속 ID 배치 처리 (Run Container로 최적화됨)
        RoaringBitmap bulkRange = new RoaringBitmap();
        bulkRange.add(0L, 65536L); // 0 ~ 65535 전부 추가
        bulkRange.runOptimize(); // Run Container로 변환

        // 직렬화 — 네트워크 전송이나 디스크 저장에 활용
        byte[] serialized = new byte[activeUsers.serializedSizeInBytes()];
        activeUsers.serialize(new java.io.DataOutputStream(
            new java.io.ByteArrayOutputStream()));
        System.out.println("직렬화 크기: " + activeUsers.serializedSizeInBytes() + " bytes");
    }
}
```

### 예제 2: Python으로 Roaring Bitmap 동작 원리 시뮬레이션

실제 라이브러리 대신 Roaring Bitmap의 컨테이너 선택 로직을 직접 구현해보면 내부 동작을 이해하는 데 도움이 된다.

```python
from sortedcontainers import SortedList

ARRAY_TO_BITMAP_THRESHOLD = 4096  # 원소 수 기준 임계값
CHUNK_BITS = 65536  # 2^16

class ArrayContainer:
    """희소 집합: 정렬된 16비트 정수 배열"""
    def __init__(self):
        self.values = SortedList()

    def add(self, val: int):
        if val not in self.values:
            self.values.add(val)

    def should_convert_to_bitmap(self) -> bool:
        return len(self.values) > ARRAY_TO_BITMAP_THRESHOLD

    def cardinality(self) -> int:
        return len(self.values)

    def __contains__(self, val):
        return val in self.values

    def __repr__(self):
        return f"ArrayContainer(size={len(self.values)})"


class BitmapContainer:
    """고밀도 집합: 65536비트 고정 비트맵"""
    def __init__(self):
        self.bits = bytearray(CHUNK_BITS // 8)  # 8,192 bytes = 8KB
        self._cardinality = 0

    def add(self, val: int):
        byte_idx, bit_idx = divmod(val, 8)
        if not (self.bits[byte_idx] & (1 << bit_idx)):
            self.bits[byte_idx] |= (1 << bit_idx)
            self._cardinality += 1

    def cardinality(self) -> int:
        return self._cardinality

    def __contains__(self, val):
        byte_idx, bit_idx = divmod(val, 8)
        return bool(self.bits[byte_idx] & (1 << bit_idx))

    def and_with(self, other: 'BitmapContainer') -> 'BitmapContainer':
        result = BitmapContainer()
        for i in range(len(self.bits)):
            result.bits[i] = self.bits[i] & other.bits[i]
        result._cardinality = bin(int.from_bytes(result.bits, 'little')).count('1')
        return result

    def __repr__(self):
        return f"BitmapContainer(cardinality={self._cardinality})"


class RoaringBitmap:
    """간단한 Roaring Bitmap 구현"""
    def __init__(self):
        # key: 상위 16비트, value: ArrayContainer or BitmapContainer
        self.containers: dict = {}

    def add(self, value: int):
        high = value >> 16       # 상위 16비트 → 청크 키
        low  = value & 0xFFFF    # 하위 16비트 → 컨테이너 내 위치

        if high not in self.containers:
            self.containers[high] = ArrayContainer()

        container = self.containers[high]
        if isinstance(container, ArrayContainer):
            container.add(low)
            # 임계값 초과 시 BitmapContainer로 전환
            if container.should_convert_to_bitmap():
                bitmap = BitmapContainer()
                for v in container.values:
                    bitmap.add(v)
                self.containers[high] = bitmap
        else:
            container.add(low)

    def __contains__(self, value: int) -> bool:
        high, low = value >> 16, value & 0xFFFF
        if high not in self.containers:
            return False
        return low in self.containers[high]

    def cardinality(self) -> int:
        return sum(c.cardinality() for c in self.containers.values())

    def __repr__(self):
        info = [(k, type(v).__name__, v.cardinality())
                for k, v in sorted(self.containers.items())]
        return f"RoaringBitmap(chunks={info})"


# 사용 예시
rb = RoaringBitmap()

# 희소 데이터: ArrayContainer 유지
for uid in [1, 100, 500, 1000, 5000]:
    rb.add(uid)

print(rb)  # ArrayContainer 사용 중
print(1000 in rb)   # True
print(999 in rb)    # False

# 고밀도 데이터: 임계값 초과 시 BitmapContainer로 전환
for uid in range(0, 5000):  # 5000개 추가 → BitmapContainer로 자동 전환
    rb.add(uid)

print(rb)  # BitmapContainer로 전환됨
print(f"전체 원소 수: {rb.cardinality()}")
```

## 집합 연산의 내부 동작

Roaring Bitmap의 집합 연산이 빠른 이유는 컨테이너 타입별 최적화된 알고리즘을 선택하기 때문이다.

| 연산 | Array ∩ Array | Array ∩ Bitmap | Bitmap ∩ Bitmap |
|---|---|---|---|
| 교집합 | Galloping 교차법 | 배열 원소로 비트 조회 | 64비트 AND 병렬 처리 |
| 합집합 | 병합 정렬 | 비트셋에 삽입 | 64비트 OR 병렬 처리 |
| 시간 복잡도 | O(m + n) | O(min(m,n)) | O(N/64) |

**Galloping 알고리즘**은 두 정렬 배열의 교집합을 구할 때 작은 배열의 원소를 큰 배열에서 이진 탐색으로 찾는 방법으로, 두 집합의 크기 차이가 클수록 효과적이다.

## Run Container 최적화

연속 구간(예: `{0,1,2,...,999}`)은 `(시작=0, 길이=999)` 하나로 표현해 공간을 획기적으로 줄인다. `runOptimize()` 메서드를 호출하면 Roaring Bitmap이 자동으로 각 컨테이너에 대해 Run 인코딩 적용 여부를 결정한다.

```python
# Run Container 효과 시뮬레이션
def rle_encode(sorted_values: list) -> list:
    """(start, run_length) 쌍으로 RLE 인코딩"""
    if not sorted_values:
        return []
    runs = []
    start = sorted_values[0]
    length = 0
    for i in range(1, len(sorted_values)):
        if sorted_values[i] == sorted_values[i-1] + 1:
            length += 1
        else:
            runs.append((start, length))
            start = sorted_values[i]
            length = 0
    runs.append((start, length))
    return runs

# 연속 구간: Run Container가 유리
sequential = list(range(0, 10000))
runs = rle_encode(sequential)
print(f"연속 10000개: 배열 {len(sequential)*2}B → RLE {len(runs)*4}B")
# 연속 10000개: 배열 20000B → RLE 4B

# 무작위 분포: Run Container가 불리
import random
random.seed(42)
sparse = sorted(random.sample(range(65536), 100))
runs_sparse = rle_encode(sparse)
print(f"희소 100개: 배열 {len(sparse)*2}B → RLE {len(runs_sparse)*4}B")
# 희소 100개: 배열 200B → RLE ~400B (Run이 오히려 큼)
```

## 주의사항 및 팁

### 1. `runOptimize()` 호출 타이밍
배치로 데이터를 추가한 후 한 번에 `runOptimize()`를 호출하는 것이 효율적이다. 삽입 도중 매번 호출하면 불필요한 컨테이너 변환 오버헤드가 발생한다.

### 2. 직렬화 포맷 호환성
Roaring Bitmap의 **Portable Serialization Format**은 Java, C/C++, Go, Python 등 여러 언어 구현체 간에 호환된다. 크로스 언어 파이프라인(예: Spark → Druid)에서도 재직렬화 없이 비트맵을 재사용할 수 있다.

### 3. 64비트 정수 지원
기본 Roaring Bitmap은 32비트 정수만 지원한다. 64비트 정수를 다루려면 **Roaring64Bitmap** 또는 **RoaringTreemap**을 사용해야 한다. 이는 상위 32비트를 키로 하는 트리맵 위에 Roaring Bitmap 노드를 올리는 구조다.

### 4. 수정이 잦은 집합에서의 주의점
Array ↔ Bitmap 컨테이너 전환이 잦으면 메모리 할당 오버헤드가 발생할 수 있다. 업데이트가 매우 빈번한 경우라면, 배치 단위로 수정 후 한 번에 적용하는 전략이 좋다.

### 5. SIMD 가속
C/C++ 구현체(`CRoaring`)는 SSE4.2, AVX2, AVX-512 명령어를 자동으로 활용해 Bitmap Container의 교집합/합집합 연산을 벡터화한다. JVM 환경에서는 Panama Vector API 또는 JDK 내장 최적화를 통해 유사한 혜택을 얻을 수 있다.

## 참고 자료
- [RoaringBitmap/RoaringBitmap - GitHub](https://github.com/RoaringBitmap/RoaringBitmap)
- [RoaringBitmap/CRoaring - C/C++ 구현체](https://github.com/RoaringBitmap/CRoaring)
- [Roaring Bitmaps: Implementation of an Optimized Software Library (arXiv)](https://arxiv.org/abs/1709.07821)
