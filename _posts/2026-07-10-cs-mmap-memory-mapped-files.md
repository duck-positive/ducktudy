---
layout: post
title: "메모리 맵 파일(mmap) 완전 정복: 파일을 메모리처럼 다루는 운영체제의 비밀"
date: 2026-07-10
categories: [cs, computer-science]
tags: [mmap, memory-mapped-files, operating-system, linux, virtual-memory, ipc, zero-copy, posix]
---

## 메모리 맵 파일(mmap)이란?

**메모리 맵 파일(Memory-Mapped File)**은 파일의 내용을 프로세스의 **가상 주소 공간(virtual address space)**에 직접 매핑하는 운영체제 기법입니다. 이렇게 매핑하면 파일을 `read()`/`write()` 시스템 콜 없이 **메모리 배열처럼** 접근할 수 있습니다.

Linux/macOS/POSIX 시스템에서는 `mmap(2)` 시스템 콜이, Windows에서는 `CreateFileMapping`/`MapViewOfFile` API가 이 기능을 제공합니다.

```
전통적인 파일 I/O:
  프로세스 → read() → 커널 버퍼 → 디스크

mmap 방식:
  프로세스 메모리 주소 → 페이지 폴트 → 디스크 (최초 접근 시만)
```

핵심은 **지연 로딩(Demand Paging)**입니다. `mmap()`을 호출해도 파일 내용이 즉시 물리 메모리에 올라오지 않습니다. 실제로 해당 주소를 읽거나 쓸 때 **페이지 폴트(page fault)**가 발생하고, 그때서야 커널이 디스크에서 해당 페이지를 물리 메모리로 가져옵니다.

---

## 왜 mmap이 필요한가?

### 1. 대용량 파일의 효율적 처리

`read()`로 10GB 파일을 처리하면 반드시 커널 버퍼 → 사용자 버퍼로 복사가 일어납니다. `mmap`은 이 복사 단계를 없애고 파일을 직접 주소 공간에 올립니다. 파일 전체를 메모리에 올리지 않고 필요한 부분(페이지 단위)만 실제 물리 메모리를 사용합니다.

### 2. 프로세스 간 통신 (IPC)

`MAP_SHARED` 플래그와 함께 사용하면 여러 프로세스가 같은 물리 메모리 페이지를 공유할 수 있습니다. 별도의 파이프나 소켓 없이 메모리를 공유하는 가장 빠른 IPC 방법입니다.

### 3. 실행 파일 로딩

Linux ELF 실행 파일, Windows PE, macOS Mach-O 모두 `mmap`으로 로드됩니다. 커널의 프로그램 로더(`execve`)는 코드 섹션과 데이터 섹션을 각각 mmap합니다. 덕분에 같은 실행 파일을 여러 프로세스가 실행할 때 코드 페이지를 **물리 메모리에서 공유**합니다.

### 4. 데이터베이스와 검색 엔진

SQLite, RocksDB, LMDB, Elasticsearch 등 많은 데이터베이스가 mmap을 활용합니다. LMDB(Lightning Memory-Mapped Database)는 전체 데이터베이스를 mmap으로 열어 커널의 페이지 캐시를 데이터베이스 캐시로 직접 활용합니다.

---

## mmap 시스템 콜 분석

### POSIX mmap() 시그니처

```c
#include <sys/mman.h>

void *mmap(
    void  *addr,    // 매핑할 가상 주소 (보통 NULL → 커널이 자동 선택)
    size_t length,  // 매핑할 바이트 수
    int    prot,    // 보호 플래그: PROT_READ | PROT_WRITE | PROT_EXEC | PROT_NONE
    int    flags,   // MAP_SHARED | MAP_PRIVATE | MAP_ANONYMOUS | ...
    int    fd,      // 파일 디스크립터 (-1 for anonymous)
    off_t  offset   // 파일 오프셋 (페이지 크기의 배수여야 함)
);
```

**주요 플래그:**

| 플래그 | 설명 |
|--------|------|
| `MAP_SHARED` | 쓰기가 파일에 반영되고 다른 프로세스에도 공유됨 |
| `MAP_PRIVATE` | Copy-on-Write: 쓰기 시 개인 복사본 생성, 원본 파일 불변 |
| `MAP_ANONYMOUS` | 파일 없이 메모리만 할당 (fd = -1) |
| `MAP_FIXED` | 반드시 addr 위치에 매핑 (위험할 수 있음) |
| `MAP_POPULATE` | 미리 모든 페이지를 물리 메모리에 올림 (prefault) |

---

## 실제 구현 예제

### 예제 1: C로 구현하는 mmap 기반 파일 읽기/쓰기

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/mman.h>
#include <sys/stat.h>

/* mmap으로 파일 읽기 */
void read_with_mmap(const char *filename) {
    int fd = open(filename, O_RDONLY);
    if (fd == -1) { perror("open"); return; }

    struct stat sb;
    if (fstat(fd, &sb) == -1) { perror("fstat"); close(fd); return; }

    /* 파일 전체를 읽기 전용으로 매핑 */
    char *data = mmap(NULL, sb.st_size, PROT_READ, MAP_PRIVATE, fd, 0);
    if (data == MAP_FAILED) { perror("mmap"); close(fd); return; }

    close(fd); // mmap 후 fd는 닫아도 됨

    /* 일반 메모리처럼 접근 */
    printf("First 100 bytes:\n%.100s\n", data);

    /* madvise로 커널에 접근 패턴 힌트 제공 */
    madvise(data, sb.st_size, MADV_SEQUENTIAL); // 순차 접근 최적화

    /* 사용 후 반드시 해제 */
    munmap(data, sb.st_size);
}

/* mmap으로 파일 수정 */
void write_with_mmap(const char *filename) {
    /* 파일을 읽기/쓰기로 열기 */
    int fd = open(filename, O_RDWR | O_CREAT, 0644);
    if (fd == -1) { perror("open"); return; }

    /* 파일 크기를 4096바이트로 설정 */
    size_t size = 4096;
    if (ftruncate(fd, size) == -1) { perror("ftruncate"); close(fd); return; }

    /* MAP_SHARED: 쓰기가 파일에 직접 반영됨 */
    char *data = mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (data == MAP_FAILED) { perror("mmap"); close(fd); return; }

    close(fd);

    /* 메모리 쓰듯이 파일에 쓰기 */
    strcpy(data, "Hello from mmap!");
    memset(data + 100, 0xAB, 200); // 임의 패턴 기록

    /* msync: 변경사항을 디스크에 명시적으로 플러시 */
    /* MS_SYNC: 완료될 때까지 블록, MS_ASYNC: 비동기 */
    if (msync(data, size, MS_SYNC) == -1) {
        perror("msync");
    }

    munmap(data, size);
    printf("File written via mmap successfully.\n");
}

/* 익명 mmap: 파일 없이 메모리 할당 (malloc 대체) */
void *anonymous_mmap(size_t size) {
    void *ptr = mmap(NULL, size,
                     PROT_READ | PROT_WRITE,
                     MAP_PRIVATE | MAP_ANONYMOUS,
                     -1, 0);
    if (ptr == MAP_FAILED) return NULL;
    return ptr;
}

int main() {
    /* 테스트 파일 생성 */
    system("echo 'Hello World! This is a test file for mmap.' > /tmp/test_mmap.txt");

    printf("=== 파일 읽기 ===\n");
    read_with_mmap("/tmp/test_mmap.txt");

    printf("\n=== 파일 쓰기 ===\n");
    write_with_mmap("/tmp/mmap_output.bin");

    /* 익명 mmap으로 큰 메모리 할당 */
    size_t big_size = 1024 * 1024 * 100; // 100MB
    void *big_mem = anonymous_mmap(big_size);
    if (big_mem) {
        printf("\n100MB 익명 mmap 성공 (지연 할당, 실제 물리 메모리는 접근 시 할당)\n");
        munmap(big_mem, big_size);
    }

    return 0;
}
```

### 예제 2: Python에서 mmap으로 프로세스 간 통신(IPC) 구현

```python
import mmap
import os
import struct
import time
import multiprocessing

# 공유 메모리를 통한 프로세스 간 통신 예시
# 구조: [4바이트 플래그][4바이트 카운터][나머지 데이터 영역]

SHARED_FILE = "/tmp/ipc_mmap.bin"
MMAP_SIZE = 4096
FLAG_READY = 1
FLAG_DONE = 2


def writer_process(filename: str):
    """생산자: mmap에 데이터를 쓰는 프로세스"""
    fd = os.open(filename, os.O_RDWR)
    mm = mmap.mmap(fd, MMAP_SIZE, mmap.MAP_SHARED, mmap.PROT_READ | mmap.PROT_WRITE)
    os.close(fd)

    for i in range(5):
        # 데이터 준비
        data = f"Message #{i} from writer at time {time.time():.2f}".encode()

        # 카운터와 데이터를 먼저 쓰고 (메모리 순서 보장을 위해)
        mm.seek(4)  # 플래그 다음부터
        mm.write(struct.pack('I', len(data)))  # 4바이트 길이
        mm.write(data.ljust(256))              # 데이터

        # 마지막으로 준비 플래그 설정 (원자적 쓰기 의미)
        mm.seek(0)
        mm.write(struct.pack('I', FLAG_READY))
        mm.flush()  # msync 호출

        print(f"Writer: sent message #{i}")
        time.sleep(0.5)

    # 종료 플래그
    mm.seek(0)
    mm.write(struct.pack('I', FLAG_DONE))
    mm.flush()
    mm.close()


def reader_process(filename: str):
    """소비자: mmap에서 데이터를 읽는 프로세스"""
    fd = os.open(filename, os.O_RDWR)
    mm = mmap.mmap(fd, MMAP_SIZE, mmap.MAP_SHARED, mmap.PROT_READ | mmap.PROT_WRITE)
    os.close(fd)

    last_read_flag = 0

    while True:
        mm.seek(0)
        (flag,) = struct.unpack('I', mm.read(4))

        if flag == FLAG_DONE:
            print("Reader: writer is done, exiting.")
            break

        if flag == FLAG_READY and flag != last_read_flag:
            (length,) = struct.unpack('I', mm.read(4))
            data = mm.read(length).decode().strip()
            print(f"Reader: received '{data}'")
            last_read_flag = flag

            # 읽었음을 표시 (플래그 초기화)
            mm.seek(0)
            mm.write(struct.pack('I', 0))
            mm.flush()

        time.sleep(0.05)

    mm.close()


def demo_mmap_ipc():
    # 공유 파일 초기화
    with open(SHARED_FILE, 'wb') as f:
        f.write(b'\x00' * MMAP_SIZE)

    # 두 프로세스를 동시에 실행
    reader = multiprocessing.Process(target=reader_process, args=(SHARED_FILE,))
    writer = multiprocessing.Process(target=writer_process, args=(SHARED_FILE,))

    reader.start()
    time.sleep(0.1)  # reader 먼저 준비
    writer.start()

    writer.join()
    reader.join()

    os.unlink(SHARED_FILE)
    print("IPC via mmap completed.")


# --- mmap으로 대용량 파일 효율적으로 처리 ---
def count_lines_with_mmap(filepath: str) -> int:
    """read() 대신 mmap으로 대용량 파일 줄 수 계산"""
    with open(filepath, 'rb') as f:
        # 읽기 전용, 복사본(MAP_PRIVATE)으로 매핑
        mm = mmap.mmap(f.fileno(), 0, access=mmap.ACCESS_READ)
        count = 0
        while mm.readline():
            count += 1
        mm.close()
    return count


# --- mmap 활용 고성능 바이너리 파일 파싱 ---
def parse_binary_file(filepath: str):
    """구조화된 바이너리 파일을 mmap으로 파싱"""
    RECORD_SIZE = 16  # 각 레코드: 4바이트 ID + 8바이트 timestamp + 4바이트 value

    with open(filepath, 'r+b') as f:
        mm = mmap.mmap(f.fileno(), 0)
        file_size = mm.size()
        num_records = file_size // RECORD_SIZE

        print(f"총 레코드 수: {num_records}")

        for i in range(min(num_records, 5)):
            offset = i * RECORD_SIZE
            mm.seek(offset)
            record_id, timestamp, value = struct.unpack('IQI', mm.read(RECORD_SIZE))
            print(f"Record {i}: id={record_id}, ts={timestamp}, val={value}")

        mm.close()


if __name__ == "__main__":
    demo_mmap_ipc()
```

---

## 내부 동작 원리: 가상 메모리와의 연관

### 페이지 테이블과 페이지 폴트

`mmap()` 호출 시 커널은 다음 작업만 수행합니다:

1. 프로세스의 **VMA(Virtual Memory Area)** 자료구조에 매핑 정보 기록
2. 가상 주소 범위 예약 (물리 메모리는 아직 할당 안 함)
3. 반환값으로 가상 주소 반환

실제 데이터 로딩은 **Demand Paging**으로 이루어집니다:

```
프로세스가 매핑된 주소에 접근
        ↓
MMU가 페이지 테이블 엔트리 확인 → 없음 → 페이지 폴트 발생
        ↓
커널 페이지 폴트 핸들러 호출
        ↓
디스크에서 해당 페이지(4KB) 읽기
        ↓
페이지 캐시(page cache)에 저장
        ↓
페이지 테이블 엔트리 업데이트
        ↓
프로세스 재개 (투명하게)
```

두 번째 접근부터는 페이지가 이미 페이지 캐시에 있으므로 페이지 폴트 없이 즉시 접근됩니다. 이것이 `mmap`이 `read()`보다 **반복 접근에서 더 빠른** 이유입니다.

### MAP_SHARED의 Copy-on-Write

`MAP_PRIVATE`는 CoW(Copy-on-Write) 의미를 가집니다. 쓰기 시 커널이 해당 페이지의 복사본을 만들어 이 프로세스 전용으로 제공하며, 원본 파일은 변경되지 않습니다. `fork()` 후 자식 프로세스의 메모리도 이 방식으로 동작합니다.

---

## 실전 활용 사례

### LMDB (Lightning Memory-Mapped Database)

LMDB는 전체 데이터베이스를 하나의 mmap으로 열고, 쓰기는 MVCC(Multi-Version Concurrency Control)를 통해 Copy-on-Write 방식으로 처리합니다. 커널의 페이지 캐시가 자연스러운 데이터베이스 캐시 역할을 합니다.

### Python의 mmap 활용

Python의 `pickle`, `shelve`, pandas `read_parquet(memory_map=True)` 등은 내부적으로 mmap을 활용합니다. NumPy의 `np.memmap`도 대용량 배열을 mmap으로 처리합니다.

```python
import numpy as np

# 1GB 배열을 메모리에 올리지 않고 mmap으로 처리
arr = np.memmap('/tmp/large_array.bin', dtype='float64', mode='w+', shape=(100_000_000,))
arr[:1000] = np.arange(1000)
del arr  # flush to disk

# 다시 읽기 (페이지 캐시 활용으로 빠름)
arr_read = np.memmap('/tmp/large_array.bin', dtype='float64', mode='r', shape=(100_000_000,))
print(arr_read[:5])  # [0. 1. 2. 3. 4.]
```

---

## 주의사항과 실전 팁

### 흔한 실수

**1. 파일 크기보다 큰 범위 매핑**
파일이 비어 있거나 length가 파일 크기를 초과하면 `SIGBUS` 시그널이 발생합니다. 반드시 `ftruncate()`로 파일 크기를 먼저 설정하세요.

**2. offset이 페이지 크기의 배수여야 함**
대부분 시스템에서 페이지 크기는 4096바이트입니다. `sysconf(_SC_PAGESIZE)`로 확인하세요.

**3. munmap 없이 종료**
프로세스 종료 시 자동으로 해제되지만, 장수 프로세스에서는 반드시 `munmap()`을 명시적으로 호출해야 주소 공간 누수가 없습니다.

**4. MAP_SHARED에서 msync 누락**
`MAP_SHARED`로 쓴 데이터는 커널이 적절한 시점에 디스크에 반영하지만, 즉시 반영이 필요하면 `msync()`를 명시적으로 호출해야 합니다.

### 성능 최적화 팁

```c
// 순차 접근: 커널에 힌트 제공
madvise(ptr, size, MADV_SEQUENTIAL);

// 랜덤 접근: 미리 읽기(prefetch) 비활성화
madvise(ptr, size, MADV_RANDOM);

// 곧 필요 없는 영역: 물리 메모리 반환 허용
madvise(ptr, size, MADV_DONTNEED);

// 핵심 영역: 스왑 아웃 방지
mlock(ptr, size);
```

### 언제 mmap을 피해야 하는가

- **작은 파일 반복 읽기**: 일반 `read()`가 더 직관적이고 충분히 빠름
- **네트워크 파일시스템(NFS)**: 페이지 폴트가 네트워크 지연을 유발해 예측 불가능한 성능
- **잦은 크기 변경이 필요한 파일**: remapping이 필요해 복잡도 증가

---

## 참고 자료

- [mmap(2) - Linux Manual Page](https://www.man7.org/linux/man-pages/man2/mmap.2.html)
- [Memory Mapping - The Linux Kernel Documentation](https://linux-kernel-labs.github.io/refs/heads/master/labs/memory_mapping.html)
- [Memory-Mapped Files - Wikipedia](https://en.wikipedia.org/wiki/Mmap)
- [Memory Mapped Files - i0exception Medium](https://medium.com/i0exception/memory-mapped-files-5e083e653b1)
