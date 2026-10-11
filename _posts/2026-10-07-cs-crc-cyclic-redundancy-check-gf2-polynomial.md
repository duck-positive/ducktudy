---
layout: post
title: "CRC(순환 중복 검사) 완전 정복: GF(2) 다항식 산술로 데이터 무결성을 검증하는 법"
date: 2026-10-07
categories: [cs, computer-science]
tags: [crc, error-detection, gf2, polynomial, networking, storage, checksum]
---

## CRC란 무엇인가?

CRC(Cyclic Redundancy Check, 순환 중복 검사)는 데이터 전송이나 저장 과정에서 발생하는 오류를 감지하기 위한 오류 검출 코드(Error Detection Code)입니다. 이더넷 프레임, ZIP 파일, PNG 이미지, USB 프로토콜, NVMe 드라이브 등 우리가 매일 사용하는 시스템의 수십 곳에서 데이터 무결성을 지키는 핵심 메커니즘입니다.

CRC의 핵심 아이디어는 간단합니다. **송신 측**이 데이터 비트열을 특정 다항식으로 나누어 나머지 값(체크섬)을 데이터에 붙여 전송하고, **수신 측**이 받은 데이터 전체를 같은 다항식으로 나누어 나머지가 0인지 확인합니다. 비트 오류가 없다면 나머지는 항상 0이고, 오류가 있다면 나머지는 0이 아닌 값이 됩니다.

## 왜 CRC가 필요한가?

### 단순 합산 체크섬의 한계

가장 단순한 오류 검출 방법은 모든 바이트를 더하는 것입니다. 하지만 이 방식은 치명적인 약점이 있습니다.

```python
def simple_checksum(data: bytes) -> int:
    return sum(data) % 256

# 두 바이트가 맞바뀌어도 체크섬은 동일
data1 = bytes([0x12, 0x34])  # checksum = 0x46
data2 = bytes([0x34, 0x12])  # checksum = 0x46 (같음!)

print(simple_checksum(data1) == simple_checksum(data2))  # True
```

바이트의 순서가 바뀌거나 두 바이트에서 반대 방향으로 오류가 생기면 단순 합산은 이를 감지하지 못합니다. CRC는 다항식의 위치적 특성을 이용해 이러한 오류 패턴까지 감지합니다.

### CRC의 강점

- **버스트 오류 감지**: 연속된 비트 오류에 특히 강합니다. CRC-32는 최대 32비트 버스트 오류를 100% 감지합니다.
- **고성능**: 하드웨어에서는 LFSR(Linear Feedback Shift Register) 회로로, 소프트웨어에서는 룩업 테이블로 극도로 빠르게 계산됩니다.
- **수학적 보장**: GF(2) 위의 다항식 산술이 제공하는 수학적 특성 덕분에 오류 감지 능력이 엄밀하게 증명됩니다.

## GF(2) 다항식 산술의 핵심 원리

GF(2)는 0과 1만을 원소로 갖는 갈루아 체(Galois Field)입니다. 이 체에서 덧셈은 XOR이고, 곱셈은 AND입니다. 올림(carry)이 없어 모든 연산이 비트 연산으로 직접 구현됩니다.

| 연산 | GF(2) 규칙 |
|------|-----------|
| 0+0  | 0         |
| 0+1  | 1         |
| 1+1  | 0 (XOR)   |
| 뺄셈 | 덧셈과 동일 (XOR) |

데이터 비트열 `1011`은 다항식 `x³ + x + 1`로 표현됩니다. CRC 계산은 이 데이터 다항식 M(x)를 생성 다항식 G(x)로 나눈 나머지 R(x)를 구하는 것입니다.

```
전송 메시지 T(x) = M(x) * x^r + R(x)
  여기서 r = G(x)의 차수
  R(x) = (M(x) * x^r) mod G(x)
```

수신 측에서 T(x)를 G(x)로 나누면 R(x) - R(x) = 0이 됩니다(XOR이므로 뺄셈 = 덧셈).

### CRC-16 직접 구현

```python
def crc16_simple(data: bytes, poly: int = 0x1021, init: int = 0xFFFF) -> int:
    """
    CRC-16/CCITT 구현 (비트 단위 직접 계산).
    poly=0x1021 은 x^16 + x^12 + x^5 + 1 을 나타냄.
    """
    crc = init
    for byte in data:
        crc ^= byte << 8          # 바이트를 레지스터 상위에 XOR
        for _ in range(8):
            if crc & 0x8000:      # MSB가 1이면
                crc = (crc << 1) ^ poly
            else:
                crc <<= 1
            crc &= 0xFFFF         # 16비트로 마스킹
    return crc

# 검증
data = b"Hello, CRC!"
checksum = crc16_simple(data)
print(f"CRC-16: 0x{checksum:04X}")

# 수신 측 검증: 데이터 + 체크섬을 같은 함수에 통과시키면 0이어야 함
received = data + checksum.to_bytes(2, 'big')
print(f"수신 검증 결과: 0x{crc16_simple(received, init=0xFFFF):04X}")
```

## 룩업 테이블 최적화: CRC-32 고성능 구현

비트 단위 계산은 정확하지만 느립니다. 실제 시스템에서는 1바이트를 한 번에 처리하는 룩업 테이블 기법을 사용해 성능을 8배 향상시킵니다.

```python
def make_crc32_table(poly: int = 0xEDB88320) -> list[int]:
    """
    CRC-32 룩업 테이블 생성 (256 엔트리).
    0xEDB88320은 0x04C11DB7의 비트 역전(반사된) 형태.
    이더넷, ZIP, PNG 등에서 사용하는 CRC-32/ISO-HDLC.
    """
    table = []
    for i in range(256):
        crc = i
        for _ in range(8):
            if crc & 1:
                crc = (crc >> 1) ^ poly
            else:
                crc >>= 1
        table.append(crc)
    return table

CRC32_TABLE = make_crc32_table()

def crc32(data: bytes, init: int = 0xFFFFFFFF) -> int:
    crc = init
    for byte in data:
        table_idx = (crc ^ byte) & 0xFF
        crc = (crc >> 8) ^ CRC32_TABLE[table_idx]
    return crc ^ 0xFFFFFFFF  # 최종 XOR

import binascii
data = b"123456789"
our_result = crc32(data)
lib_result = binascii.crc32(data) & 0xFFFFFFFF
print(f"우리 구현: 0x{our_result:08X}")
print(f"표준 라이브러리: 0x{lib_result:08X}")
assert our_result == lib_result, "CRC-32 구현 오류!"
print("검증 성공!")
```

### 주요 CRC 표준 비교

| 표준 | 다항식 (HEX) | 차수 | 사용처 |
|------|------------|------|--------|
| CRC-8/SMBUS | 0x07 | 8 | SMBus, I2C |
| CRC-16/CCITT | 0x1021 | 16 | USB, XMODEM |
| CRC-32/ISO-HDLC | 0x04C11DB7 | 32 | 이더넷, ZIP, PNG |
| CRC-32C (Castagnoli) | 0x1EDC6F41 | 32 | iSCSI, SCTP, NVMe |
| CRC-64/ECMA-182 | 0x42F0E1EBA9EA3693 | 64 | DLT 자기 테이프 |

CRC-32C(Castagnoli)는 Intel SSE4.2의 `crc32` 명령어와 ARM의 `CRC32CW` 명령어로 하드웨어 가속이 지원되어 소프트웨어 CRC-32보다 수십 배 빠릅니다.

## 하드웨어 구현: LFSR 회로

CRC의 진정한 아름다움은 하드웨어 구현에 있습니다. GF(2) 나눗셈은 LFSR(Linear Feedback Shift Register) 회로로 정확히 대응됩니다.

```
CRC-4 (다항식: x^4 + x + 1 = 0b10011)

       입력 비트 스트림
           │
    ┌──────▼──────────────────────┐
    │      XOR                   │
    │  ┌───▼──┐  ┌────┐  ┌────┐  ┌────┐
    │  │  r3  ├─►│ r2 ├─►│ r1 ├─►│ r0 │──► XOR (피드백)
    │  └──────┘  └────┘  └──┬─┘  └────┘
    │                        │ (x 항 피드백)
    └────────────────────────┘

각 클럭 사이클마다 1비트 처리.
모든 데이터 비트 처리 후 r3..r0 = CRC 값.
```

이 회로는 매 클럭 사이클에 1비트를 처리하며, 추가 하드웨어 없이도 파이프라인으로 확장 가능합니다.

## CRC의 오류 감지 능력과 한계

CRC-32는 다음을 **100% 감지**합니다.
- 1비트 오류
- 2비트 오류 (데이터가 32비트 이상)
- 홀수 개수의 비트 오류 (다항식이 (x+1)을 인수로 가지므로)
- 최대 32비트 길이의 버스트 오류
- 대부분의 32비트 이상 버스트 오류

**한계**: CRC는 오류 **감지**만 하고 **정정**은 하지 못합니다. 또한 의도적인 데이터 변조에는 취약합니다(공격자가 올바른 CRC를 재계산할 수 있으므로). 보안이 필요한 경우 HMAC-SHA256 같은 암호학적 MAC을 사용해야 합니다.

## 주의사항과 실무 팁

**1. 초기값(init)과 최종 XOR을 주의하라**
같은 다항식을 사용해도 init 값과 final XOR 값에 따라 CRC 결과가 달라집니다. 시스템 간 상호 운용성을 위해 표준에서 정의한 파라미터를 정확히 따라야 합니다.

**2. 비트 반사(reflection)를 이해하라**
많은 CRC 구현이 반사된(reflected) 형태를 사용합니다. 이더넷 CRC-32는 비트를 MSB first가 아닌 LSB first로 처리합니다.

**3. 하드웨어 가속을 활용하라**
Python에서는 `zlib.crc32()`, Go에서는 `hash/crc32.New()`, Rust에서는 `crc` crate를 사용하면 내부적으로 SIMD나 하드웨어 명령어를 활용합니다.

**4. CRC≠암호화 해시**
CRC는 무결성 검사용이며 보안 목적으로 사용하면 안 됩니다. 데이터 변조를 막으려면 SHA-256 또는 BLAKE3를 사용하세요.

## 참고 자료
- [Wikipedia - Cyclic Redundancy Check](https://en.wikipedia.org/wiki/Cyclic_redundancy_check)
- [Philip Koopman's CRC Research](https://www.cs.cmu.edu/~koopman/crc/)
- [Chorba: A novel CRC32 implementation (arXiv:2412.16398)](https://arxiv.org/abs/2412.16398)
- [Ross Williams - A Painless Guide to CRC Error Detection Algorithms](http://www.ross.net/crc/download/crc_v3.txt)
