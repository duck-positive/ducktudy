---
layout: post
title: "외부 정렬(External Sorting) 완전 정복: 메모리보다 큰 데이터를 k-way 병합으로 정렬하는 법"
date: 2026-07-05
categories: [cs, computer-science]
tags: [external-sorting, k-way-merge, merge-sort, disk-io, buffer, polyphase-merge, replacement-selection, database]
---

## 개요

1TB 로그 파일을 정렬해야 하는데 서버 메모리는 32GB뿐이다. 이 경우 QuickSort나 MergeSort 같은 **내부 정렬(Internal Sorting)**은 전혀 사용할 수 없다. 데이터가 메모리에 다 올라오지 않기 때문이다.

**외부 정렬(External Sorting)**은 데이터를 **메모리 크기의 청크(chunk)**로 나누어 정렬하고, 결과물을 디스크에 쌓아 둔 뒤 **다단계 병합(multi-pass merge)**으로 최종 정렬 파일을 만드는 알고리즘이다. 관계형 데이터베이스(PostgreSQL의 `ORDER BY`, MySQL의 filesort)와 빅데이터 프레임워크(Hadoop MapReduce, Spark)가 모두 이 원리를 사용한다.

---

## 왜 필요한가

### 내부 정렬의 한계

| 알고리즘 | 시간 복잡도 | 공간 복잡도 | 제약 |
|---|---|---|---|
| QuickSort | O(n log n) avg | O(log n) | 전체 데이터가 메모리에 있어야 함 |
| MergeSort | O(n log n) | O(n) | 전체 데이터가 메모리에 있어야 함 |
| **External Sort** | **O(n log n)** | **O(M)** | **M = 메모리 크기** |

### I/O 비용이 지배하는 세계

디스크(HDD)의 순차 읽기 속도는 메모리 접근보다 약 1만 배 느리다. 외부 정렬에서는 **I/O 횟수(디스크 접근 횟수)**를 최소화하는 것이 핵심 목표다. 알고리즘 복잡도 분석도 CPU 연산이 아닌 **I/O 복잡도**를 기준으로 한다.

```
I/O 복잡도 = O((n/B) log_{M/B}(n/B))
  n: 총 데이터 크기
  B: 디스크 블록(버퍼) 크기
  M: 사용 가능한 메모리 크기
```

---

## 핵심 알고리즘: 2단계 외부 병합 정렬

### Phase 1: Run 생성 (Sorting Phase)

전체 데이터를 메모리 크기 M 단위로 읽어 내부 정렬하고 디스크에 **정렬된 런(sorted run)**으로 저장한다.

```
입력 파일 크기: 1 GB
메모리 크기: 64 MB
→ 런 개수: ceil(1024 / 64) = 16개 런
각 런 크기: 64 MB (마지막 런은 작을 수 있음)
```

### Phase 2: k-way 병합 (Merge Phase)

16개의 정렬된 런을 동시에 병합한다. 메모리를 k+1개의 버퍼로 분할한다:
- k개의 **입력 버퍼**(각 런에서 읽어오는 페이지)
- 1개의 **출력 버퍼**(병합 결과를 디스크에 쓰는 페이지)

```
메모리 64MB, 런 16개, 페이지 크기 4MB
→ 입력 버퍼: 16 × 4MB = 64MB... 메모리 부족
→ 현실적으로: k ≤ floor(M/B) - 1
```

k를 크게 하면 패스 수가 줄어들고, k를 작게 하면 버퍼당 크기가 커져 I/O 효율이 올라간다.

---

## 구현 예제 1: Python으로 구현하는 외부 정렬

```python
import heapq
import os
import tempfile

def external_sort(input_file: str, output_file: str,
                  memory_lines: int = 10000, k: int = 8):
    """
    input_file:   정렬할 텍스트 파일 (한 줄 = 한 레코드)
    output_file:  정렬 결과 파일
    memory_lines: 메모리에 한 번에 올릴 최대 줄 수 (메모리 크기 시뮬레이션)
    k:            k-way 병합 팬인(fan-in)
    """
    # Phase 1: 정렬된 런 파일 생성
    run_files = _create_sorted_runs(input_file, memory_lines)
    print(f"Phase 1 완료: {len(run_files)}개 런 생성")

    # Phase 2: k-way 병합 (런이 1개 남을 때까지 반복)
    pass_num = 0
    while len(run_files) > 1:
        next_runs = []
        for i in range(0, len(run_files), k):
            batch = run_files[i:i+k]
            merged = _merge_k_runs(batch, pass_num, i // k)
            next_runs.append(merged)
            for f in batch:
                os.remove(f)  # 중간 파일 삭제
        run_files = next_runs
        pass_num += 1
        print(f"Pass {pass_num} 완료: {len(run_files)}개 런 남음")

    os.rename(run_files[0], output_file)
    print(f"정렬 완료: {output_file}")


def _create_sorted_runs(input_file: str, memory_lines: int) -> list[str]:
    run_files = []
    with open(input_file, 'r') as f:
        while True:
            lines = []
            for _ in range(memory_lines):
                line = f.readline()
                if not line:
                    break
                lines.append(line.rstrip('\n'))
            if not lines:
                break

            lines.sort()   # 내부 정렬 (TimSort)

            tmp = tempfile.NamedTemporaryFile(
                mode='w', suffix='.run', delete=False)
            tmp.write('\n'.join(lines) + '\n')
            tmp.close()
            run_files.append(tmp.name)

    return run_files


def _merge_k_runs(run_files: list[str], pass_num: int, idx: int) -> str:
    """MinHeap을 사용한 k-way 병합"""
    handles = [open(f, 'r') for f in run_files]
    heap = []

    # 각 런에서 첫 번째 줄을 읽어 힙에 삽입
    for i, fh in enumerate(handles):
        line = fh.readline().rstrip('\n')
        if line:
            heapq.heappush(heap, (line, i))

    tmp = tempfile.NamedTemporaryFile(
        mode='w', suffix='.merged', delete=False)

    while heap:
        val, i = heapq.heappop(heap)
        tmp.write(val + '\n')
        next_line = handles[i].readline().rstrip('\n')
        if next_line:
            heapq.heappush(heap, (next_line, i))

    tmp.close()
    for fh in handles:
        fh.close()

    return tmp.name


# 테스트
if __name__ == "__main__":
    # 테스트 파일 생성
    import random, string
    with open("/tmp/input.txt", "w") as f:
        for _ in range(50000):
            word = ''.join(random.choices(string.ascii_lowercase, k=8))
            f.write(word + '\n')

    external_sort("/tmp/input.txt", "/tmp/output.txt",
                  memory_lines=5000, k=4)

    # 정렬 검증
    with open("/tmp/output.txt") as f:
        lines = f.read().splitlines()
    assert lines == sorted(lines), "정렬 실패!"
    print(f"검증 완료: {len(lines)}개 줄 정렬됨")
```

---

## 구현 예제 2: Java로 구현하는 정수 외부 정렬 (버퍼 I/O 최적화)

```java
import java.io.*;
import java.util.*;

public class ExternalSort {
    static final int MEMORY_INTS = 1_000_000;  // 메모리 4MB (int 100만개)
    static final int K = 8;                    // k-way 병합

    public static void sort(String inputPath, String outputPath) throws IOException {
        List<String> runFiles = createRuns(inputPath);
        System.out.printf("Phase 1: %d개 런 생성%n", runFiles.size());

        while (runFiles.size() > 1) {
            List<String> next = new ArrayList<>();
            for (int i = 0; i < runFiles.size(); i += K) {
                List<String> batch = runFiles.subList(i, Math.min(i + K, runFiles.size()));
                String merged = mergeRuns(batch);
                next.add(merged);
                batch.forEach(File::new); // GC에게 맡기기 (실제로는 delete 호출)
            }
            runFiles = next;
        }
        new File(runFiles.get(0)).renameTo(new File(outputPath));
    }

    static List<String> createRuns(String path) throws IOException {
        List<String> runs = new ArrayList<>();
        try (DataInputStream in = new DataInputStream(
                new BufferedInputStream(new FileInputStream(path), 1 << 20))) {
            int[] buf = new int[MEMORY_INTS];
            int count;
            while ((count = readBlock(in, buf)) > 0) {
                Arrays.sort(buf, 0, count);
                String tmp = File.createTempFile("run", ".tmp").getAbsolutePath();
                writeBlock(tmp, buf, count);
                runs.add(tmp);
            }
        }
        return runs;
    }

    static String mergeRuns(List<String> files) throws IOException {
        // PriorityQueue: (값, 파일 인덱스)
        PriorityQueue<long[]> pq = new PriorityQueue<>(
            Comparator.comparingLong(a -> a[0]));
        DataInputStream[] streams = new DataInputStream[files.size()];

        for (int i = 0; i < files.size(); i++) {
            streams[i] = new DataInputStream(
                new BufferedInputStream(new FileInputStream(files.get(i))));
            if (streams[i].available() > 0) {
                pq.offer(new long[]{streams[i].readInt(), i});
            }
        }

        String tmp = File.createTempFile("merged", ".tmp").getAbsolutePath();
        try (DataOutputStream out = new DataOutputStream(
                new BufferedOutputStream(new FileOutputStream(tmp), 1 << 20))) {
            while (!pq.isEmpty()) {
                long[] top = pq.poll();
                out.writeInt((int) top[0]);
                int idx = (int) top[1];
                if (streams[idx].available() > 0) {
                    pq.offer(new long[]{streams[idx].readInt(), idx});
                }
            }
        }
        for (DataInputStream s : streams) s.close();
        return tmp;
    }

    static int readBlock(DataInputStream in, int[] buf) throws IOException {
        int i = 0;
        while (i < buf.length && in.available() > 0) {
            buf[i++] = in.readInt();
        }
        return i;
    }

    static void writeBlock(String path, int[] buf, int count) throws IOException {
        try (DataOutputStream out = new DataOutputStream(
                new BufferedOutputStream(new FileOutputStream(path)))) {
            for (int i = 0; i < count; i++) out.writeInt(buf[i]);
        }
    }
}
```

---

## 최적화 기법들

### 1. Replacement Selection으로 런 크기 2배 늘리기

기본 방식은 메모리 M만큼의 런을 생성한다. **Replacement Selection(대체 선택)**을 쓰면 평균 런 크기를 **2M**으로 늘릴 수 있어 패스 수가 절반으로 줄어든다.

아이디어:
1. 메모리 M개를 힙(MinHeap)으로 채운다.
2. 힙에서 최솟값 `x`를 출력하고 디스크에 쓴다.
3. 다음 입력 값 `y`를 읽는다.
   - `y >= x`이면 현재 런에 계속 포함 (힙에 삽입)
   - `y < x`이면 현재 런 종료 불가 → 별도 구역에 격리 (다음 런으로)
4. 힙이 비면 새 런 시작

난수 데이터에서 평균 런 크기는 `2M`이 되는 것이 이론적으로 증명되어 있다 (Knuth, TAOCP Vol.3).

### 2. 더블 버퍼링 (Double Buffering)

I/O와 CPU 연산을 겹쳐서 수행한다. 버퍼 A를 CPU가 처리하는 동안 버퍼 B에 I/O를 수행한다:

```
     CPU 연산:  [==A==][==B==][==A==][==B==]
     디스크 I/O:      [==B==][==A==][==B==]
```

디스크 대기 시간을 거의 숨길 수 있어 처리량이 크게 향상된다.

### 3. Polyphase Merge (다단 병합)

일반 k-way 병합은 각 패스에서 균등하게 런을 처리한다. **Polyphase Merge**는 피보나치 수열 비율로 런을 분배하여 최적의 패스 수를 달성한다. 테이프 드라이브 시대의 최적화지만 테이프 장치 수가 제한될 때 여전히 유효하다.

### 4. SSD를 위한 외부 정렬

SSD는 임의 접근 비용이 HDD보다 훨씬 낮다. 따라서:
- 작은 k로 여러 패스를 도는 것보다 **큰 k로 한 패스**가 더 유리할 수 있다.
- Write Amplification을 줄이기 위해 중간 런 파일을 최소화한다.
- `O_DIRECT` 플래그로 OS 페이지 캐시를 우회해 직접 I/O를 수행한다.

---

## 데이터베이스에서의 외부 정렬

### PostgreSQL의 `work_mem`과 external sort

```sql
-- 현재 설정 확인
SHOW work_mem;  -- 기본값: 4MB

-- 더 많은 메모리를 허용하면 런 수 감소 → 패스 수 감소 → 빠른 정렬
SET work_mem = '256MB';
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM large_table ORDER BY created_at;
```

`EXPLAIN` 출력에서 `Sort Method: external merge Disk: XXXKB`가 보이면 메모리 부족으로 외부 정렬이 발생한 것이다. `work_mem`을 늘리면 `Sort Method: quicksort Memory`로 바뀐다.

### MySQL의 filesort

MySQL은 `ORDER BY`에서 인덱스를 사용할 수 없을 때 filesort를 수행한다. `sort_buffer_size`가 작으면 임시 파일에 결과를 쓰는 외부 정렬이 발생한다:

```sql
-- filesort 여부 확인
EXPLAIN SELECT * FROM orders ORDER BY amount DESC;
-- Extra: Using filesort

-- 버퍼 크기 증가
SET sort_buffer_size = 67108864;  -- 64MB
```

---

## 성능 분석: 패스 수와 I/O 횟수

| 조건 | 패스 수 | 총 I/O 횟수 |
|---|---|---|
| 1GB 파일, 64MB 메모리, k=2 (binary merge) | log₂(16) = 4 패스 | 8 × (1GB/4MB) = 2048회 |
| 1GB 파일, 64MB 메모리, k=16 (16-way merge) | 1 패스 | 2 × (1GB/4MB) = 512회 |
| 1GB 파일, 64MB 메모리, k=16 + Replacement Selection | 1 패스 | 2 × (512MB/4MB) = 256회 |

k를 크게 할수록 패스 수가 줄지만, 버퍼당 크기가 작아져 I/O 효율(디스크 스트리밍)이 떨어질 수 있다. 최적 k는 `M / B - 1` (여기서 B는 I/O 효율이 최대가 되는 버퍼 크기).

---

## 주의사항 및 팁

### 1. 중간 파일 위치
중간 런 파일을 입력 파일과 같은 디스크에 두면 읽기와 쓰기가 경쟁한다. 가능하면 **별도 디스크(다른 물리 드라이브 또는 NVMe)**에 중간 파일을 두자.

### 2. 압축된 런 파일
중간 런 파일을 LZ4나 Snappy로 압축하면 I/O 양은 줄지만 CPU 비용이 늘어난다. CPU가 병목이 아닌 경우(디스크가 병목인 경우) 유효한 최적화다.

### 3. 병렬화
여러 코어를 활용해 Phase 1의 런 생성을 병렬화할 수 있다. 각 스레드가 독립된 청크를 처리하면 된다. Phase 2의 k-way 병합은 순차 I/O 특성상 병렬화 이점이 적다.

### 4. 레코드 크기 고정 vs 가변
레코드 크기가 고정이면(정수, 고정 길이 문자열) 오프셋 계산이 쉬워 구현이 단순하다. 가변 레코드는 줄 바꿈이나 길이 접두사로 경계를 표시해야 한다.

---

## 실전 활용 사례

- **PostgreSQL/MySQL**: `ORDER BY`, `GROUP BY`, `DISTINCT`, 해시 조인의 빌드 단계
- **Hadoop MapReduce**: Map 결과를 키 기준으로 정렬하는 Shuffle 단계
- **Apache Spark**: `sortWithinPartitions`, `orderBy` 연산의 내부 구현
- **UNIX `sort` 명령**: `-T` 옵션으로 중간 파일 디렉터리 지정, `-u` 옵션으로 중복 제거
- **데이터 웨어하우스 ETL**: 수백 GB 로그 파일을 타임스탬프 기준 정렬 후 파티셔닝

---

## 정리

외부 정렬은 **메모리에 다 올라오지 않는 데이터**를 정렬하기 위한 2단계 알고리즘이다. Phase 1에서 메모리 크기의 정렬된 런을 만들고, Phase 2에서 MinHeap 기반 k-way 병합으로 하나의 정렬 파일을 완성한다. 성능의 핵심은 I/O 횟수 최소화이며, k 값 선택, Replacement Selection, 더블 버퍼링, 중간 파일 위치 최적화로 패스 수를 줄일 수 있다. 데이터베이스의 `ORDER BY`, 빅데이터 프레임워크의 Shuffle 단계가 모두 이 원리 위에 구축되어 있다.

## 참고 자료
- [Wikipedia: External sorting](https://en.wikipedia.org/wiki/External_sorting)
- [Wikipedia: K-way merge algorithm](https://en.wikipedia.org/wiki/K-way_merge_algorithm)
- [AlgoCademy: Handling Large Data Sets - External Sorting Algorithms](https://algocademy.com/blog/handling-large-data-sets-external-sorting-algorithms/)
- [PostgreSQL Documentation: Resource Consumption - work_mem](https://www.postgresql.org/docs/current/runtime-config-resource.html)
