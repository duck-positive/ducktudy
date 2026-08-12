---
layout: post
title: "LZ4와 Zstandard 완전 정복: 실시간 압축의 두 거인이 속도와 압축률을 조율하는 방법"
date: 2026-08-12
categories: [cs, computer-science]
tags: [lz4, zstandard, zstd, compression, lz77, algorithm, performance]
---

데이터 압축의 세계에는 언제나 두 가지 상충되는 목표가 있다. **더 많이 압축할 것이냐, 아니면 더 빠르게 처리할 것이냐.** 이 오래된 딜레마를 각자의 방식으로 해결한 두 알고리즘이 현대 인프라의 곳곳에 침투해 있다. LZ4는 압축 속도 500 MB/s 이상을 목표로 설계된 초고속 압축기이고, Zstandard(Zstd)는 Facebook이 개발해 속도와 압축률을 동시에 잡으려 한 범용 압축기다. 이 둘은 경쟁자이기도 하지만, 용도가 다른 보완재이기도 하다.

## 개념 설명: LZ77 계열의 DNA

LZ4와 Zstd는 모두 LZ77 알고리즘을 계승한다. LZ77의 핵심 아이디어는 단순하다: **이미 처리한 데이터 안에서 현재 위치와 일치하는 패턴을 찾아, 그 위치와 길이로 대체한다.**

```
입력:  "abcdeabcde"
출력:  "abcde" (5, 5)   → 5칸 앞에서 5글자를 복사하라
```

이 원리 위에서 LZ4와 Zstd는 서로 다른 최적화 전략을 취한다.

### LZ4의 핵심 설계 철학: "속도는 기능이다"

LZ4는 2011년 Yann Collet이 설계했다. 목표는 단 하나, **디코딩 속도를 RAM 대역폭에 근접시키는 것**이었다.

**포맷 구조**: LZ4의 스트림은 일련의 **시퀀스(sequence)**로 구성된다. 각 시퀀스는 다음으로 이루어진다:
1. **Token (1 byte)**: 상위 4비트 = literal 길이, 하위 4비트 = match 길이
2. **Extra literal length** (optional): token 상위가 15면 추가 바이트
3. **Literals**: 그대로 복사되는 원문 바이트들
4. **Offset (2 bytes, little-endian)**: 매치 위치까지의 거리
5. **Extra match length** (optional): token 하위가 15면 추가 바이트

핵심은 오프셋이 **2바이트(최대 65535)로 고정**된다는 것이다. 이로 인해 검색 윈도우가 64KB로 제한되지만, 디코더가 Branch 없이 단순한 메모리 복사만으로 동작할 수 있다.

### Zstandard의 핵심 설계 철학: "엔트로피를 최대한 쥐어짜라"

Zstd는 2016년 Facebook이 공개했다. LZ77 매칭 단계 이후 **ANS(Asymmetric Numeral Systems) 기반 엔트로피 코딩**을 추가해, LZ4가 그냥 내보내는 토큰과 오프셋까지도 더 작게 인코딩한다.

**3단계 파이프라인**:
1. **LZ 매칭**: 슬라이딩 윈도우(최대 128MB!)에서 패턴을 찾아 (literal, offset, match_length) 트리플을 생성
2. **FSE(Finite State Entropy) 인코딩**: offset, match_length, literal_length를 각각 ANS로 압축
3. **Huffman 코딩**: literal 바이트 자체를 허프만 트리로 압축

이 추가 단계들이 Zstd를 느리게 만드는 것처럼 보이지만, ANS는 수학적으로 산술 코딩(Arithmetic Coding)에 가까운 압축률을 **테이블 룩업만으로** 달성한다. 결과적으로 Zstd는 gzip보다 훨씬 빠르면서 비슷하거나 더 나은 압축률을 보인다.

### 성능 비교 (일반적인 벤치마크 기준)

| 알고리즘 | 압축 속도 | 압축 해제 속도 | 압축률 (vs 원본) |
|----------|-----------|---------------|-----------------|
| LZ4 | ~700 MB/s | ~4,000 MB/s | ~50% |
| Zstd Level 1 | ~430 MB/s | ~1,700 MB/s | ~40% |
| Zstd Level 3 | ~260 MB/s | ~1,600 MB/s | ~35% |
| gzip Level 6 | ~50 MB/s | ~350 MB/s | ~32% |
| bzip2 | ~15 MB/s | ~40 MB/s | ~25% |

## 왜 이 두 알고리즘이 중요한가

### LZ4가 빛나는 곳: I/O 병목이 있는 실시간 시스템

LZ4 해제 속도는 NVMe SSD의 순차 읽기 속도(~3,500 MB/s)보다도 빠르다. 즉, 디스크에서 LZ4 압축 데이터를 읽으면서 동시에 압축 해제해도 CPU가 I/O를 전혀 기다리지 않는다. 이것이 Kafka, ClickHouse, LevelDB 등이 LZ4를 기본 압축 알고리즘으로 채택한 이유다.

### Zstd가 빛나는 곳: 저장 비용과 처리 속도를 둘 다 중요시할 때

Zstd는 압축 레벨 1~22를 제공한다. 레벨 1은 LZ4에 가까운 속도, 레벨 19 이상은 lzma에 가까운 압축률을 낸다. 이 조절 가능성 덕분에 Facebook의 아카이브 시스템, Linux 커널 패키지 포맷(zst), macOS Ventura의 앱 번들 등 다양한 레이어에서 채택됐다.

### 딕셔너리 압축: 작은 데이터의 구원

일반 압축 알고리즘은 데이터가 충분히 클 때 효과적이다. 1KB 미만의 소형 JSON 응답을 압축하면 압축률이 오히려 음수가 될 수도 있다. Zstd의 **딕셔너리 모드**는 이 문제를 해결한다. 학습 단계에서 대표적인 샘플 데이터로 딕셔너리를 훈련하면, 그 딕셔너리를 공유하는 양쪽에서 소형 데이터도 50% 이상 압축할 수 있다.

## 실제 구현 예제

### 예제 1: Python에서 lz4와 zstd 사용하기

```python
import lz4.frame
import zstandard as zstd
import time
import os

def benchmark_compression(data: bytes, label: str):
    """압축/해제 속도와 압축률을 측정"""
    # LZ4 압축
    start = time.perf_counter()
    lz4_compressed = lz4.frame.compress(data)
    lz4_compress_time = time.perf_counter() - start

    start = time.perf_counter()
    lz4_decompressed = lz4.frame.decompress(lz4_compressed)
    lz4_decompress_time = time.perf_counter() - start

    # Zstd 압축 (레벨 3)
    cctx = zstd.ZstdCompressor(level=3)
    dctx = zstd.ZstdDecompressor()

    start = time.perf_counter()
    zstd_compressed = cctx.compress(data)
    zstd_compress_time = time.perf_counter() - start

    start = time.perf_counter()
    zstd_decompressed = dctx.decompress(zstd_compressed)
    zstd_decompress_time = time.perf_counter() - start

    original_size = len(data)
    print(f"\n=== {label} ===")
    print(f"원본 크기: {original_size:,} bytes")
    print(f"LZ4:  압축={len(lz4_compressed):,}B ({len(lz4_compressed)/original_size:.1%}), "
          f"압축 {original_size/lz4_compress_time/1e6:.0f}MB/s, "
          f"해제 {original_size/lz4_decompress_time/1e6:.0f}MB/s")
    print(f"Zstd: 압축={len(zstd_compressed):,}B ({len(zstd_compressed)/original_size:.1%}), "
          f"압축 {original_size/zstd_compress_time/1e6:.0f}MB/s, "
          f"해제 {original_size/zstd_decompress_time/1e6:.0f}MB/s")

    assert lz4_decompressed == data, "LZ4 무결성 오류!"
    assert zstd_decompressed == data, "Zstd 무결성 오류!"


# 반복 패턴이 많은 텍스트 데이터
text_data = (b"Hello, this is a test of compression algorithms. " * 10000)

# 난수 데이터 (압축 불리)
import random
random_data = bytes([random.randint(0, 255) for _ in range(500_000)])

benchmark_compression(text_data, "반복 텍스트 (500KB)")
benchmark_compression(random_data, "난수 바이너리 (500KB)")

# Zstd 딕셔너리 예시: 소형 JSON 압축
import json

# 학습 데이터 생성
samples = [
    json.dumps({"user_id": i, "action": "click", "timestamp": 1720000000 + i, "page": f"/page/{i%10}"}).encode()
    for i in range(1000)
]

# 딕셔너리 훈련
dict_data = zstd.ZstdCompressionDict(
    zstd.train_dictionary(8192, samples)
)

cctx_dict = zstd.ZstdCompressor(level=3, dict_data=dict_data)
dctx_dict = zstd.ZstdDecompressor(dict_data=dict_data)
cctx_plain = zstd.ZstdCompressor(level=3)

test_json = json.dumps({"user_id": 12345, "action": "click", "timestamp": 1720001234, "page": "/page/5"}).encode()
compressed_dict = cctx_dict.compress(test_json)
compressed_plain = cctx_plain.compress(test_json)

print(f"\n=== 딕셔너리 압축 효과 ===")
print(f"원본 JSON: {len(test_json)} bytes")
print(f"딕셔너리 없이: {len(compressed_plain)} bytes ({len(compressed_plain)/len(test_json):.1%})")
print(f"딕셔너리 사용: {len(compressed_dict)} bytes ({len(compressed_dict)/len(test_json):.1%})")
```

### 예제 2: C언어로 LZ4 스트리밍 압축 구현

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "lz4frame.h"

#define CHUNK_SIZE (64 * 1024)   // 64KB 청크 단위 처리

/*
 * LZ4 스트리밍: 파일을 청크 단위로 읽어 압축하고
 * LZ4 프레임 포맷으로 출력하는 예시
 */
int compress_file_lz4(const char* src_path, const char* dst_path) {
    FILE* src = fopen(src_path, "rb");
    FILE* dst = fopen(dst_path, "wb");
    if (!src || !dst) { perror("fopen"); return -1; }

    LZ4F_preferences_t prefs = LZ4F_INIT_PREFERENCES;
    prefs.compressionLevel = 0;    // 0 = 기본 속도 모드
    prefs.frameInfo.blockSizeID = LZ4F_max64KB;
    prefs.frameInfo.contentChecksumFlag = LZ4F_contentChecksumEnabled;

    size_t src_buf_size = CHUNK_SIZE;
    size_t dst_buf_size = LZ4F_compressBound(CHUNK_SIZE, &prefs);
    char* src_buf = malloc(src_buf_size);
    char* dst_buf = malloc(dst_buf_size);

    LZ4F_cctx* ctx;
    LZ4F_createCompressionContext(&ctx, LZ4F_VERSION);

    // 프레임 헤더 쓰기
    size_t header_size = LZ4F_compressBegin(ctx, dst_buf, dst_buf_size, &prefs);
    fwrite(dst_buf, 1, header_size, dst);

    size_t total_in = 0, total_out = header_size;
    size_t n;

    while ((n = fread(src_buf, 1, src_buf_size, src)) > 0) {
        size_t compressed = LZ4F_compressUpdate(
            ctx, dst_buf, dst_buf_size, src_buf, n, NULL
        );
        if (LZ4F_isError(compressed)) {
            fprintf(stderr, "압축 오류: %s\n", LZ4F_getErrorName(compressed));
            break;
        }
        fwrite(dst_buf, 1, compressed, dst);
        total_in += n;
        total_out += compressed;
    }

    // 프레임 종료 (체크섬 포함)
    size_t end_size = LZ4F_compressEnd(ctx, dst_buf, dst_buf_size, NULL);
    fwrite(dst_buf, 1, end_size, dst);
    total_out += end_size;

    printf("압축 완료: %zu → %zu bytes (%.1f%%)\n",
           total_in, total_out, 100.0 * total_out / total_in);

    LZ4F_freeCompressionContext(ctx);
    free(src_buf); free(dst_buf);
    fclose(src); fclose(dst);
    return 0;
}

int main(int argc, char* argv[]) {
    if (argc != 3) {
        fprintf(stderr, "사용법: %s <입력파일> <출력파일.lz4>\n", argv[0]);
        return 1;
    }
    return compress_file_lz4(argv[1], argv[2]);
}
```

실행 예시:
```bash
gcc -O2 compress.c -o compress -llz4
./compress large_log.txt large_log.txt.lz4
# 압축 완료: 104857600 → 38291445 bytes (36.5%)
```

## 주의사항 및 팁

### 1. LZ4와 Zstd 선택 기준

- **LZ4를 선택**: 압축 해제 속도가 최우선이거나, 실시간 스트리밍(게임 엔진, 네트워크 패킷, DB WAL 압축)에서 레이턴시가 중요할 때
- **Zstd를 선택**: 저장 공간 비용이 중요하거나, 한 번 쓰고 여러 번 읽는 패턴(아카이브, 배포 패키지), 또는 소형 데이터가 많은 API 응답 압축

### 2. 압축 레벨과 CPU 비용

Zstd의 레벨 1~3은 단일 스레드 기준으로도 일반 서버에서 충분히 처리 가능하다. 레벨 19 이상은 압축 시간이 급격히 증가하므로 오프라인 아카이브 전용으로 사용해야 한다. 반면 레벨 `--fast` (음수값)는 LZ4보다 낮은 압축률이지만 동등한 속도를 내려고 할 때 유용하다.

### 3. 스트리밍 프레임 포맷

LZ4와 Zstd 모두 **프레임 포맷**과 **블록 포맷**을 구분한다. 블록 포맷은 헤더나 체크섬이 없어 더 빠르지만, 프레임 포맷은 매직 넘버, 체크섬, 컨텐츠 크기 등을 포함해 안전하다. 실제 파일 저장에는 항상 프레임 포맷을 사용해야 한다.

### 4. 딕셔너리의 공유 문제

Zstd 딕셔너리 압축에서 서버는 압축 시 사용한 딕셔너리의 ID를 스트림 헤더에 포함한다. 클라이언트는 같은 ID의 딕셔너리를 보유해야 해제할 수 있다. 딕셔너리를 업데이트할 때 구버전 호환성을 반드시 고려해야 한다.

### 5. checksum과 보안

LZ4와 Zstd의 체크섬은 데이터 무결성(전송 중 손상 감지)을 위한 것이지, 암호화 해시가 아니다. 악의적인 변조를 감지하려면 별도의 HMAC 또는 서명이 필요하다.

## 정리

LZ4는 "이 이상 빠를 수 없다"는 철학으로 알고리즘을 설계해 하드웨어의 이론적 한계에 근접한 압축 해제 속도를 달성했다. Zstd는 "속도와 압축률은 트레이드오프가 아니다"는 도전을 ANS 엔트로피 코딩과 광대한 탐색 윈도우로 극복했다. 둘 다 현대 인프라의 핵심 레이어에서 활발히 사용되며, 용도에 맞게 선택하는 것이 중요하다. 특히 마이크로서비스 환경에서 반복적인 소형 API 응답을 압축할 때는 Zstd 딕셔너리 모드가 놀라운 효과를 낼 수 있다.

## 참고 자료
- [GitHub: lz4/lz4 — 극한의 압축 속도를 위한 공식 구현](https://github.com/lz4/lz4)
- [GitHub: facebook/zstd — Zstandard 공식 저장소](https://github.com/facebook/zstd)
