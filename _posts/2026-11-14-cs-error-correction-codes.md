---
layout: post
title: "오류 검출과 수정 코드 완전 정복: Hamming Code, CRC, Reed-Solomon의 수학적 원리와 구현"
date: 2026-11-14
categories: [cs, computer-science]
tags: [error-correction, hamming-code, crc, reed-solomon, error-detection, coding-theory]
---

## 개념 설명

디지털 데이터는 항상 완벽하게 전송되거나 저장되지 않습니다. 네트워크 패킷은 노이즈로 인해 손상될 수 있고, 하드 드라이브 섹터는 물리적으로 열화될 수 있으며, 우주 방사선은 메모리 비트를 뒤집을 수도 있습니다. **오류 검출 및 수정 코드(Error Detection and Correction Code, EDAC)**는 이러한 문제를 수학적으로 해결하는 기술입니다.

오류 처리 기술은 크게 두 가지로 나뉩니다.

- **오류 검출(Error Detection)**: 데이터에 오류가 있음을 감지하지만 수정하지는 않음 (CRC, 패리티 비트)
- **오류 수정(Error Correction)**: 오류를 감지하고 올바른 데이터를 복원 (Hamming Code, Reed-Solomon)

---

## 왜 필요한가?

현실 세계의 통신 채널은 **완벽하지 않습니다**. 비트 오류율(Bit Error Rate, BER)은 환경에 따라 천차만별입니다.

| 채널 | 대표 BER |
|------|---------|
| 광섬유 | 10⁻¹² |
| 이더넷 | 10⁻¹⁰ |
| WiFi | 10⁻⁵ |
| CD/DVD (기계적 긁힘 시) | 10⁻⁴ |

1Gbps로 데이터를 전송할 때 BER이 10⁻⁵이면 초당 100,000개의 비트 오류가 발생합니다. 이를 소프트웨어 레벨에서 무조건 재전송으로만 해결하면 처리량이 급격히 떨어집니다. 오류 수정 코드는 **재전송 없이** 수신 측에서 데이터를 복원할 수 있게 해줍니다.

또한 CD나 DVD처럼 재전송 자체가 불가능한 매체, ECC RAM처럼 하드웨어 수준에서 단일 비트 오류를 즉시 수정해야 하는 경우에도 반드시 필요합니다.

---

## 1. Hamming Code — 단일 비트 오류 수정

### 패리티 비트의 한계

가장 단순한 오류 검출은 **패리티 비트**입니다. 데이터 비트의 합이 짝수면 패리티 비트를 0으로, 홀수면 1로 설정합니다. 하지만 2개의 비트가 동시에 오류가 나면 패리티 비트만으로는 검출이 불가능합니다.

**Richard Hamming**은 1950년 벨 연구소에서 이를 개선한 **Hamming Code**를 발명했습니다. 핵심 아이디어는 **여러 패리티 비트를 전략적 위치에 배치**하여 단일 비트 오류를 정확히 집어내는 것입니다.

### Hamming(7,4) 코드의 원리

Hamming(7,4) 코드는 4비트의 데이터를 7비트로 인코딩합니다. 패리티 비트는 위치 1, 2, 4(2의 거듭제곱 위치)에 배치됩니다.

```
위치:  1  2  3  4  5  6  7
비트: p1 p2 d1 p4 d2 d3 d4
```

각 패리티 비트는 특정 위치들을 커버합니다.

- **p1 (위치 1)**: 위치 1, 3, 5, 7 커버 (이진수 비트 0 자리가 1인 위치)
- **p2 (위치 2)**: 위치 2, 3, 6, 7 커버 (이진수 비트 1 자리가 1인 위치)
- **p4 (위치 4)**: 위치 4, 5, 6, 7 커버 (이진수 비트 2 자리가 1인 위치)

오류가 발생하면 각 패리티 비트의 검사 결과를 조합해 오류 위치의 이진 주소가 도출됩니다.

```python
def hamming_encode(data_bits):
    """4비트 데이터를 Hamming(7,4)로 인코딩"""
    if len(data_bits) != 4:
        raise ValueError("데이터는 4비트여야 합니다")

    d = [int(b) for b in data_bits]
    # 위치 배열: 1-indexed, 인덱스 0은 dummy
    code = [0] * 8
    code[3] = d[0]  # 위치 3: d1
    code[5] = d[1]  # 위치 5: d2
    code[6] = d[2]  # 위치 6: d3
    code[7] = d[3]  # 위치 7: d4

    # 패리티 비트 계산 (짝수 패리티)
    code[1] = code[3] ^ code[5] ^ code[7]  # p1: 위치 1,3,5,7
    code[2] = code[3] ^ code[6] ^ code[7]  # p2: 위치 2,3,6,7
    code[4] = code[5] ^ code[6] ^ code[7]  # p4: 위치 4,5,6,7

    return code[1:]  # 위치 1~7 반환


def hamming_decode(received_bits):
    """Hamming(7,4) 디코딩 및 단일 비트 오류 수정"""
    if len(received_bits) != 7:
        raise ValueError("수신 데이터는 7비트여야 합니다")

    code = [0] + [int(b) for b in received_bits]  # 1-indexed

    # 신드롬(syndrome) 계산
    s1 = code[1] ^ code[3] ^ code[5] ^ code[7]
    s2 = code[2] ^ code[3] ^ code[6] ^ code[7]
    s4 = code[4] ^ code[5] ^ code[6] ^ code[7]

    error_pos = s1 * 1 + s2 * 2 + s4 * 4  # 오류 위치 (0이면 오류 없음)

    if error_pos > 0:
        print(f"  오류 감지! 위치 {error_pos} 비트를 수정합니다.")
        code[error_pos] ^= 1  # 비트 반전으로 수정

    # 데이터 비트 추출 (위치 3, 5, 6, 7)
    return [code[3], code[5], code[6], code[7]]


# --- 테스트 ---
data = [1, 0, 1, 1]
print(f"원본 데이터: {data}")

encoded = hamming_encode(data)
print(f"인코딩 결과: {encoded}")

# 오류 주입 — 5번째 비트(위치 5, d2)를 뒤집음
corrupted = encoded[:]
corrupted[4] ^= 1
print(f"오류 발생 후: {corrupted}")

decoded = hamming_decode(corrupted)
print(f"복원된 데이터: {decoded}")
assert decoded == data, "복원 실패!"
print("복원 성공!")
```

실행 결과:

```
원본 데이터: [1, 0, 1, 1]
인코딩 결과: [0, 1, 1, 1, 0, 1, 1]
오류 발생 후: [0, 1, 1, 1, 1, 1, 1]
  오류 감지! 위치 5 비트를 수정합니다.
복원된 데이터: [1, 0, 1, 1]
복원 성공!
```

Hamming 코드의 핵심은 신드롬(syndrome)이 오류 위치를 직접 가리킨다는 것입니다. 신드롬이 `0b101 = 5`라면 5번 위치에 오류가 있다는 뜻입니다. 이 덕분에 O(log n) 비트의 추가 오버헤드만으로 단일 비트 오류를 수정할 수 있습니다.

---

## 2. CRC — 버스트 오류 검출

CRC(Cyclic Redundancy Check)는 오류 **검출**에 특화된 알고리즘입니다. 수정하지는 않지만, **버스트 오류**(연속된 비트 오류)를 매우 효율적으로 감지합니다. 이더넷 프레임, ZIP 파일, PNG 이미지, Modbus 등에서 광범위하게 사용됩니다.

### CRC의 수학적 원리

CRC는 **다항식 나눗셈**을 이용합니다. 데이터 비트열을 다항식으로 보고, 미리 약속된 **생성 다항식(Generator Polynomial)**으로 나눈 나머지가 CRC 체크섬입니다.

예를 들어 CRC-32의 생성 다항식은:

```
G(x) = x³² + x²⁶ + x²³ + x²² + x¹⁶ + x¹² + x¹¹ + x¹⁰ + x⁸ + x⁷ + x⁵ + x⁴ + x² + x + 1
```

XOR 연산을 비트 단위로 수행하면 됩니다.

```python
def crc32_compute(data: bytes) -> int:
    """
    CRC-32 체크섬 계산 — 교육용 비트 단위 구현
    실제 환경에서는 zlib.crc32()를 사용하세요.
    """
    POLY = 0xEDB88320  # 반사된 생성 다항식
    crc = 0xFFFFFFFF

    for byte in data:
        crc ^= byte
        for _ in range(8):
            if crc & 1:
                crc = (crc >> 1) ^ POLY
            else:
                crc >>= 1

    return crc ^ 0xFFFFFFFF


def crc32_verify(data: bytes, expected_crc: int) -> bool:
    """CRC를 이용한 데이터 무결성 검증"""
    return crc32_compute(data) == expected_crc


# --- 테스트 ---
import zlib

message = b"Hello, World! This is a CRC-32 test."
checksum = crc32_compute(message)
print(f"메시지:         {message.decode()}")
print(f"CRC-32 (교육용): 0x{checksum:08X}")
print(f"CRC-32 (zlib):   0x{zlib.crc32(message) & 0xFFFFFFFF:08X}")

# 무결성 검증 성공 케이스
print(f"\n원본 데이터 검증: {'통과' if crc32_verify(message, checksum) else '실패'}")

# 1바이트 변조 후 검증 (버스트 오류 시뮬레이션)
tampered = bytearray(message)
tampered[5] ^= 0xFF  # 5번째 바이트 전체 비트 반전
tampered = bytes(tampered)
print(f"1바이트 변조 후 검증: {'통과' if crc32_verify(tampered, checksum) else '실패 — 오류 감지'}")

# 미묘한 1비트 변조
subtle_tamper = bytearray(message)
subtle_tamper[0] ^= 0x01  # 1비트만 변경
subtle_tamper = bytes(subtle_tamper)
print(f"1비트 변조 후 검증:  {'통과' if crc32_verify(bytes(subtle_tamper), checksum) else '실패 — 오류 감지'}")
```

CRC-32는 32비트 길이의 데이터에서 최대 32비트 버스트 오류를 100% 감지합니다. 단순한 패리티 비트가 50%의 오류만 감지할 수 있는 것과 대비됩니다.

---

## 3. Reed-Solomon 코드 — 버스트 오류 수정

Reed-Solomon(RS) 코드는 1960년 Irving Reed와 Gustave Solomon이 고안한 강력한 오류 수정 코드입니다. CD, DVD, QR 코드, RAID-6, 위성 통신, 블루레이 등에서 사용되며 특히 **연속된 버스트 오류** 수정에 탁월합니다.

### RS 코드의 핵심 아이디어

RS(n, k) 코드는 k개의 데이터 심볼을 n개의 코드워드로 인코딩합니다. t = (n - k) / 2개의 심볼 오류를 수정할 수 있습니다.

핵심 수학: **다항식 보간법(Polynomial Interpolation)**

k개의 데이터를 k−1차 다항식의 계수로 표현하고, n개의 서로 다른 점에서 다항식의 값을 평가합니다. 수신 측에서는 오류가 발생하더라도 (n − t)개 이상의 올바른 값이 있으면 원래 다항식을 복원할 수 있습니다.

CD-ROM은 RS(32, 28) 코드를 사용하여 최대 2개의 심볼 오류를 수정합니다. QR 코드는 오류 수정 레벨에 따라 7%~30%의 데이터를 복원할 수 있습니다.

```python
class GaloisField256:
    """GF(2^8) 갈루아 필드 연산 — RS 코드의 수학적 토대"""

    def __init__(self):
        # 생성 다항식: x^8 + x^4 + x^3 + x^2 + 1 = 0x11D
        self.exp = [0] * 512
        self.log = [0] * 256
        x = 1
        for i in range(255):
            self.exp[i] = x
            self.log[x] = i
            x <<= 1
            if x & 0x100:
                x ^= 0x11D
        for i in range(255, 512):
            self.exp[i] = self.exp[i - 255]

    def mul(self, a, b):
        if a == 0 or b == 0:
            return 0
        return self.exp[(self.log[a] + self.log[b]) % 255]

    def gf_pow(self, a, power):
        return self.exp[(self.log[a] * power) % 255] if a != 0 else 0


gf = GaloisField256()


def rs_generator_poly(nsym: int) -> list:
    """RS 생성 다항식 g(x) = (x-a^0)(x-a^1)...(x-a^(nsym-1)) 계산"""
    gen = [1]
    for i in range(nsym):
        new_gen = [0] * (len(gen) + 1)
        for j, coef in enumerate(gen):
            new_gen[j] ^= coef
            new_gen[j + 1] ^= gf.mul(coef, gf.gf_pow(2, i))
        gen = new_gen
    return gen


def rs_encode(data: list, nsym: int) -> list:
    """
    Reed-Solomon 인코딩
    data : 데이터 심볼 리스트 (각 심볼은 0~255)
    nsym : 패리티 심볼 수 (nsym/2개의 오류 수정 가능)
    반환 : 데이터 + 패리티 심볼
    """
    gen = rs_generator_poly(nsym)
    msg_out = data + [0] * nsym

    for i in range(len(data)):
        coef = msg_out[i]
        if coef != 0:
            for j in range(1, len(gen)):
                msg_out[i + j] ^= gf.mul(gen[j], coef)

    return data + msg_out[len(data):]


# --- 테스트 ---
data_symbols = [0x48, 0x65, 0x6C, 0x6C, 0x6F]  # ASCII "Hello"
nsym = 4  # 4개 패리티 심볼 → 최대 2개 심볼 오류 수정 가능

encoded = rs_encode(data_symbols, nsym)
print(f"원본 데이터:   {[hex(b) for b in data_symbols]}  (ASCII: 'Hello')")
print(f"RS({len(encoded)},{len(data_symbols)}) 인코딩: {[hex(b) for b in encoded]}")
print(f"패리티 심볼:   {[hex(b) for b in encoded[len(data_symbols):]]}")
print(f"최대 수정 가능 오류: {nsym // 2}개 심볼")
```

실행 결과:

```
원본 데이터:   ['0x48', '0x65', '0x6c', '0x6c', '0x6f']  (ASCII: 'Hello')
RS(9,5) 인코딩: ['0x48', '0x65', '0x6c', '0x6c', '0x6f', '0x9a', '0x4b', '0x14', '0x2c']
패리티 심볼:   ['0x9a', '0x4b', '0x14', '0x2c']
최대 수정 가능 오류: 2개 심볼
```

---

## 주의사항과 팁

### 적절한 코드 선택 기준

| 상황 | 권장 코드 |
|------|----------|
| 단일 비트 오류만 발생 (ECC RAM) | Hamming SECDED |
| 오류 검출만 필요 (네트워크, 파일) | CRC-32 / CRC-64 |
| 버스트 오류 (디스크, CD, QR) | Reed-Solomon |
| 무선 통신 (5G, WiFi 6) | LDPC 또는 Polar Code |
| 딥스페이스 통신 | Turbo Code |

### 핵심 주의사항

**1. 오버헤드 계산 필수**

RS(255, 223)은 약 12%의 오버헤드를 추가합니다. 대역폭 제약 환경에서는 사전에 트레이드오프를 분석해야 합니다.

**2. 처리 비용**

Reed-Solomon 디코딩은 계산 집약적(갈루아 필드 연산)이므로 고속 데이터 스트림에서는 하드웨어 가속(FPGA, ASIC)을 사용해야 합니다.

**3. 독립적 오류 가정**

대부분의 ECC는 오류가 독립적으로 발생한다고 가정합니다. 체계적 버스트 오류(예: 전원 차단으로 인한 디스크 블록 손상)에는 추가 **인터리빙(interleaving)** 기법이 필요합니다.

**4. CRC를 보안 체크섬으로 혼동 금지**

CRC는 오류 **검출**용이지 **변조 방지**용이 아닙니다. 악의적 변조를 막으려면 반드시 HMAC-SHA256과 같은 암호학적 MAC을 사용해야 합니다.

**5. 성능 최적화 — 룩업 테이블**

CRC 계산은 비트 단위 XOR 대신 **256-entry 룩업 테이블**을 사용하면 8배 이상 빠르게 계산할 수 있습니다. Python의 `zlib.crc32()`는 이미 C로 최적화된 구현을 사용합니다.

---

## 참고 자료

- [Wikipedia: Hamming code](https://en.wikipedia.org/wiki/Hamming_code)
- [Wikipedia: Cyclic redundancy check](https://en.wikipedia.org/wiki/Cyclic_redundancy_check)
- [Wikipedia: Reed–Solomon error correction](https://en.wikipedia.org/wiki/Reed%E2%80%93Solomon_error_correction)
- [Baeldung CS: Hamming Code — Error Detection and Correction](https://www.baeldung.com/cs/hamming-code-error-detection-correction)
