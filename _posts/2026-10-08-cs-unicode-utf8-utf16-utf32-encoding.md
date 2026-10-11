---
layout: post
title: "Unicode와 UTF-8/16/32 인코딩 완전 정복: 전 세계 텍스트를 표현하는 표준의 내부 구조"
date: 2026-10-08
categories: [cs, computer-science]
tags: [unicode, utf-8, utf-16, utf-32, encoding, charset, BOM, surrogate-pair]
---

소프트웨어가 전 세계 사용자를 만나는 순간, 텍스트 인코딩 문제는 피할 수 없는 현실이 됩니다. "한글이 깨진다", "이모지가 두 글자로 보인다", "BOM이 뭔가요?" — 이 모든 혼란의 뿌리는 하나입니다. Unicode와 그 인코딩 방식(UTF-8, UTF-16, UTF-32)을 정확히 이해하지 못하는 것. 이 글에서는 유니코드 표준의 내부 구조부터 세 가지 UTF 인코딩의 알고리즘을 단계별로 분석합니다.

---

## 개념 설명: Unicode란 무엇인가

### 코드 포인트(Code Point)

**Unicode**는 전 세계 모든 문자에 고유한 번호를 부여하는 **문자 집합(Character Set) 표준**입니다. 각 문자에 할당된 번호를 **코드 포인트(Code Point)**라고 하며, `U+XXXX` 형식으로 표기합니다.

- `U+0041` → 'A' (라틴 대문자 A)
- `U+AC00` → '가' (한글 음절의 첫 번째)
- `U+1F600` → '😀' (Grinning Face 이모지)

코드 포인트의 범위는 `U+0000`부터 `U+10FFFF`까지, 총 **1,114,112개**의 슬롯이 있습니다.

### 평면(Plane)과 BMP

Unicode는 **17개의 평면(Plane)**으로 나뉩니다. 각 평면은 65,536개(0x10000개)의 코드 포인트를 담습니다.

| 평면 | 이름 | 범위 | 주요 내용 |
|------|------|------|-----------|
| 0 | BMP (Basic Multilingual Plane) | U+0000~U+FFFF | 대부분의 현대 언어, 기본 기호 |
| 1 | SMP (Supplementary Multilingual Plane) | U+10000~U+1FFFF | 이모지, 고대 문자 |
| 2 | SIP (Supplementary Ideographic Plane) | U+20000~U+2FFFF | 확장 한자 |
| 3~13 | (미할당) | | |
| 14 | SSP (Supplementary Special-purpose Plane) | U+E0000~U+EFFFF | 태그 |
| 15~16 | PUA (Private Use Area) | U+F0000~U+10FFFF | 사용자 정의 |

BMP 내의 `U+D800`~`U+DFFF` 범위는 **서러게이트 쌍(Surrogate Pair)**을 위해 예약된 2,048개의 코드 포인트로, 실제 문자에 사용되지 않습니다.

---

## 왜 필요한가: ASCII와 Code Page의 한계

초기 컴퓨터는 7비트 **ASCII**(128개 문자)로 영어를 표현했습니다. 각국은 나머지 128개 슬롯(128~255)을 자국 문자로 채운 **코드 페이지(Code Page)**를 만들었습니다. EUC-KR, Shift-JIS, CP1252… 이 코드 페이지들은 국가 간 데이터 교환에서 충돌했습니다.

- 한국인이 EUC-KR로 작성한 파일을 일본 시스템이 Shift-JIS로 읽으면: 문자 깨짐
- 같은 바이트 시퀀스가 다른 의미를 가짐: 보안 취약점

Unicode는 이 문제를 해결하기 위해 단일 코드 포인트 공간을 정의했습니다. 그러나 코드 포인트 자체는 추상적 번호이므로, 이를 실제 바이트로 변환하는 **인코딩(Encoding)**이 필요합니다. 이것이 UTF-8, UTF-16, UTF-32의 역할입니다.

---

## 실제 구현 예제 1: UTF-8 인코딩 알고리즘

UTF-8은 가변 길이 인코딩으로, 코드 포인트 범위에 따라 1~4바이트를 사용합니다.

| 코드 포인트 범위 | 바이트 수 | 비트 패턴 |
|----------------|---------|-----------|
| U+0000 ~ U+007F | 1 | `0xxxxxxx` |
| U+0080 ~ U+07FF | 2 | `110xxxxx 10xxxxxx` |
| U+0800 ~ U+FFFF | 3 | `1110xxxx 10xxxxxx 10xxxxxx` |
| U+10000 ~ U+10FFFF | 4 | `11110xxx 10xxxxxx 10xxxxxx 10xxxxxx` |

**'가'(U+AC00)를 UTF-8로 인코딩하는 과정:**

```
U+AC00 = 1010 1100 0000 0000 (이진수)

범위: U+0800 ~ U+FFFF → 3바이트 패턴: 1110xxxx 10xxxxxx 10xxxxxx

AC00의 이진수: 1010 1100 0000 0000 (16비트)
상위 4비트:   1010  → 1110 1010 = 0xEA
중간 6비트:   110000 → 10 110000 = 0xB0
하위 6비트:   000000 → 10 000000 = 0x80

결과: EA B0 80 (3바이트)
```

아래는 이 알고리즘을 Python으로 직접 구현한 코드입니다:

```python
def encode_utf8(codepoint: int) -> bytes:
    """
    Unicode 코드 포인트를 UTF-8 바이트열로 인코딩합니다.
    표준 라이브러리 없이 직접 구현.
    """
    if codepoint < 0 or codepoint > 0x10FFFF:
        raise ValueError(f"Invalid code point: U+{codepoint:04X}")

    if codepoint <= 0x7F:
        # 1바이트: ASCII 범위
        return bytes([codepoint])

    elif codepoint <= 0x7FF:
        # 2바이트
        b1 = 0b11000000 | (codepoint >> 6)
        b2 = 0b10000000 | (codepoint & 0x3F)
        return bytes([b1, b2])

    elif codepoint <= 0xFFFF:
        # 3바이트 (BMP)
        # 서러게이트 범위는 인코딩 불가
        if 0xD800 <= codepoint <= 0xDFFF:
            raise ValueError(f"Surrogate code point: U+{codepoint:04X}")
        b1 = 0b11100000 | (codepoint >> 12)
        b2 = 0b10000000 | ((codepoint >> 6) & 0x3F)
        b3 = 0b10000000 | (codepoint & 0x3F)
        return bytes([b1, b2, b3])

    else:
        # 4바이트 (보조 평면)
        b1 = 0b11110000 | (codepoint >> 18)
        b2 = 0b10000000 | ((codepoint >> 12) & 0x3F)
        b3 = 0b10000000 | ((codepoint >> 6) & 0x3F)
        b4 = 0b10000000 | (codepoint & 0x3F)
        return bytes([b1, b2, b3, b4])


def decode_utf8(data: bytes) -> list[int]:
    """UTF-8 바이트열을 코드 포인트 리스트로 디코딩합니다."""
    result = []
    i = 0
    while i < len(data):
        byte = data[i]
        if byte < 0x80:           # 1바이트
            result.append(byte)
            i += 1
        elif byte < 0xC0:         # 연속 바이트 (단독 등장 시 오류)
            raise ValueError(f"Unexpected continuation byte at {i}")
        elif byte < 0xE0:         # 2바이트
            cp = ((byte & 0x1F) << 6) | (data[i+1] & 0x3F)
            result.append(cp)
            i += 2
        elif byte < 0xF0:         # 3바이트
            cp = ((byte & 0x0F) << 12) | ((data[i+1] & 0x3F) << 6) | (data[i+2] & 0x3F)
            result.append(cp)
            i += 3
        else:                     # 4바이트
            cp = ((byte & 0x07) << 18) | ((data[i+1] & 0x3F) << 12) | \
                 ((data[i+2] & 0x3F) << 6) | (data[i+3] & 0x3F)
            result.append(cp)
            i += 4
    return result


# 테스트
test_chars = [0x0041, 0xAC00, 0x1F600]  # 'A', '가', '😀'
for cp in test_chars:
    encoded = encode_utf8(cp)
    decoded = decode_utf8(encoded)
    print(f"U+{cp:04X} → {encoded.hex().upper()} → U+{decoded[0]:04X}")

# 출력:
# U+0041 → 41 → U+0041
# U+AC00 → EAB080 → U+AC00
# U+1F600 → F09F9880 → U+1F600
```

UTF-8의 설계는 매우 영리합니다. 첫 번째 바이트의 상위 비트 패턴만 보면 총 바이트 수를 알 수 있고, 연속 바이트는 항상 `10xxxxxx` 패턴이므로 스트림 중간에서도 경계를 찾을 수 있습니다. 또한 U+0000~U+007F는 ASCII와 완전히 호환됩니다.

---

## 실제 구현 예제 2: UTF-16과 서러게이트 쌍

UTF-16은 BMP 문자는 2바이트(16비트), 보조 평면 문자는 4바이트(서러게이트 쌍)로 표현합니다.

**서러게이트 쌍 알고리즘 (U+10000 이상):**

```
코드 포인트 C = U+1F600 (= 0x1F600 = 128512)

1. U+10000을 뺀다: C' = 0x1F600 - 0x10000 = 0xF600 = 0b 0000 1111 0110 0000 0000

2. 20비트로 분할 (상위 10비트 + 하위 10비트):
   상위 10비트: 0b00 0011 1101 = 0x03D → 상위 서러게이트 = 0xD800 + 0x03D = 0xD83D
   하위 10비트: 0b10 0000 0000 = 0x200 → 하위 서러게이트 = 0xDC00 + 0x200 = 0xDE00

결과: D83D DE00 (4바이트, Big-Endian)
```

```c
#include <stdio.h>
#include <stdint.h>
#include <stdbool.h>

typedef struct {
    uint16_t units[2];
    int count;  // 1 또는 2
} Utf16Sequence;

/* 코드 포인트를 UTF-16으로 인코딩 */
Utf16Sequence encode_utf16(uint32_t codepoint) {
    Utf16Sequence seq = {0};

    if (codepoint < 0x10000) {
        /* BMP: 단일 16비트 코드 유닛 */
        if (codepoint >= 0xD800 && codepoint <= 0xDFFF) {
            fprintf(stderr, "Invalid: surrogate code point U+%04X\n", codepoint);
            seq.count = 0;
            return seq;
        }
        seq.units[0] = (uint16_t)codepoint;
        seq.count = 1;
    } else if (codepoint <= 0x10FFFF) {
        /* 보조 평면: 서러게이트 쌍 */
        uint32_t c_prime = codepoint - 0x10000;
        uint16_t high = 0xD800 + (uint16_t)(c_prime >> 10);   /* 상위 10비트 */
        uint16_t low  = 0xDC00 + (uint16_t)(c_prime & 0x3FF); /* 하위 10비트 */
        seq.units[0] = high;
        seq.units[1] = low;
        seq.count = 2;
    } else {
        fprintf(stderr, "Invalid code point: U+%04X\n", codepoint);
        seq.count = 0;
    }
    return seq;
}

/* UTF-16 서러게이트 쌍을 코드 포인트로 디코딩 */
uint32_t decode_surrogate_pair(uint16_t high, uint16_t low) {
    /* 검증: high는 0xD800~0xDBFF, low는 0xDC00~0xDFFF */
    if (!((high >= 0xD800 && high <= 0xDBFF) &&
          (low  >= 0xDC00 && low  <= 0xDFFF))) {
        fprintf(stderr, "Invalid surrogate pair: %04X %04X\n", high, low);
        return 0xFFFD; /* 대체 문자 */
    }
    uint32_t c_prime = ((uint32_t)(high - 0xD800) << 10) | (low - 0xDC00);
    return c_prime + 0x10000;
}

int main(void) {
    uint32_t test[] = {0x0041, 0xAC00, 0x1F600}; /* 'A', '가', '😀' */

    for (int i = 0; i < 3; i++) {
        Utf16Sequence seq = encode_utf16(test[i]);
        printf("U+%04X → ", test[i]);
        for (int j = 0; j < seq.count; j++) {
            printf("%04X ", seq.units[j]);
        }

        /* 디코딩 검증 */
        uint32_t decoded;
        if (seq.count == 1) {
            decoded = seq.units[0];
        } else {
            decoded = decode_surrogate_pair(seq.units[0], seq.units[1]);
        }
        printf("→ U+%04X\n", decoded);
    }
    return 0;
}
/* 출력:
   U+0041 → 0041  → U+0041
   U+AC00 → AC00  → U+AC00
   U+1F600 → D83D DE00 → U+1F600
*/
```

**BOM(Byte Order Mark)**: UTF-16은 2바이트 단위이므로 빅 엔디언(Big-Endian)과 리틀 엔디언(Little-Endian) 구분이 필요합니다. 파일 앞에 `U+FEFF`(= `FF FE` in LE, `FE FF` in BE)를 붙여 바이트 순서를 표시합니다. UTF-8에서 BOM(`EF BB BF`)은 불필요하지만, 일부 윈도우 도구가 추가합니다.

---

## UTF-32: 단순하지만 비효율적

UTF-32는 모든 코드 포인트를 **고정 4바이트**로 표현합니다.

```
'A'    → 00 00 00 41
'가'   → 00 00 AC 00
'😀'   → 00 01 F6 00
```

인덱싱이 O(1)으로 단순하지만, ASCII 텍스트에서도 4배의 메모리를 소비합니다. 실제 파일 전송과 저장에는 거의 사용되지 않으며, 내부 처리 API(일부 C++ 런타임, Python 내부 표현)에서만 쓰입니다.

---

## 주의사항 및 팁

### 1. "문자 수"와 "코드 유닛 수"를 혼동하지 마세요

```python
s = "😀"  # U+1F600, 1개의 문자
print(len(s))           # Python 3: 1 (코드 포인트 기준)
print(len(s.encode('utf-8')))   # 4 (바이트)
print(len(s.encode('utf-16-le')))  # 4 (UTF-16 코드 유닛 2개 × 2바이트)
```

JavaScript에서 `"😀".length`는 **2**를 반환합니다. UTF-16 코드 유닛 수를 세기 때문입니다. 진짜 문자 수는 `[..."😀"].length`(=1) 또는 `"😀".codePointAt(0)` 등을 사용해야 합니다.

### 2. 결합 문자(Combining Character)와 정규화

'é'는 두 가지 방식으로 표현할 수 있습니다:
- **NFC**: `U+00E9` (é, 사전 합성)
- **NFD**: `U+0065 U+0301` (e + 결합 악센트, 2개의 코드 포인트)

두 표현은 시각적으로 동일하지만 바이트가 다르므로 단순 문자열 비교가 실패합니다. `unicodedata.normalize('NFC', s)`로 정규화가 필요합니다.

### 3. 인코딩 감지(Encoding Detection)

파일을 읽을 때 인코딩이 명시되지 않으면 `chardet` 라이브러리나 BOM 검사를 통해 추측해야 합니다. **항상 인코딩을 명시적으로 지정**하는 것이 최선입니다:

```python
# 좋은 예: 명시적 인코딩
with open('data.txt', 'r', encoding='utf-8') as f:
    content = f.read()

# 나쁜 예: 플랫폼 기본값에 의존
with open('data.txt', 'r') as f:  # Windows에서는 CP949가 될 수 있음
    content = f.read()
```

### 4. UTF-8의 우위

오늘날 웹과 리눅스 생태계는 **UTF-8이 사실상 표준**입니다. ASCII 호환성, 가변 길이로 인한 공간 효율, 그리고 바이트 순서 문제가 없다는 장점 때문입니다. 새 시스템을 설계한다면 UTF-8을 선택하세요.

---

## 참고 자료
- [RFC 2781: UTF-16, an encoding of ISO 10646](https://rfc-editor.org/rfc/rfc2781)
- [Unicode Standard - Microsoft Learn](https://learn.microsoft.com/ko-kr/globalization/encoding/unicode-standard)
- [Wikipedia: UTF-8](https://en.wikipedia.org/wiki/UTF-8)
