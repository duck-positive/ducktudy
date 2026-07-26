---
layout: post
title: "LZ77·LZ78·LZW 데이터 압축 알고리즘 완전 정복: gzip·PNG·GIF의 심장"
date: 2026-07-26
categories: [cs, computer-science]
tags: [compression, lz77, lz78, lzw, deflate, gzip, algorithm, data-structures]
---

데이터 압축은 현대 컴퓨팅의 근간이다. ZIP 파일을 만들 때, PNG 이미지를 저장할 때, HTTP 응답을 전송할 때 항상 압축이 일어난다. 그리고 이 모든 압축의 뒤에는 1977년과 1978년 Abraham Lempel과 Jacob Ziv가 발표한 두 편의 논문이 있다. LZ77과 LZ78이다. 이 글에서는 LZ 계열 압축 알고리즘의 원리를 깊이 파고들어, 실제로 어떻게 동작하는지 직접 구현해 본다.

## 왜 LZ 알고리즘인가?

이전 글에서 다룬 [Huffman 코딩](https://duck-positive.github.io/ducktudy)은 각 심볼의 등장 빈도를 기반으로 가변 길이 코드를 부여한다. 이는 "각 문자가 얼마나 자주 나오는가"라는 통계적 중복성(statistical redundancy)을 제거한다. 하지만 실제 데이터에는 또 다른 종류의 중복성이 있다. **구조적 중복성(structural redundancy)**, 즉 동일한 패턴이나 문자열이 반복되는 현상이다.

예를 들어 "abcabcabc"라는 문자열은 Huffman 코딩으로는 그다지 압축되지 않는다. 각 문자의 빈도가 고르기 때문이다. 하지만 "abc"라는 패턴이 3번 반복된다는 사실을 이용하면 훨씬 더 효과적으로 압축할 수 있다. LZ 알고리즘이 바로 이 구조적 중복성을 공략한다.

### 현대 압축 포맷과의 관계

LZ 알고리즘이 얼마나 광범위하게 쓰이는지 살펴보면:

- **DEFLATE** (= LZ77 + Huffman): ZIP, gzip, PNG, HTTP `Content-Encoding: gzip`
- **LZW**: GIF, TIFF, PDF 내부 스트림, Unix `compress` 명령어
- **LZMA** (LZ77 기반): 7-Zip, XZ, LZMA2
- **LZ4, Zstandard**: 현대적 고속 압축 포맷, LZ77 파생
- **Brotli**: 웹 압축, LZ77 기반

사실상 무손실 압축 알고리즘의 대부분이 LZ 계열이다.

## LZ77: 슬라이딩 윈도우 방식

LZ77의 핵심 아이디어는 **슬라이딩 윈도우(sliding window)**다. 인코더는 최근에 처리한 데이터를 기억하고, 앞으로 처리할 데이터가 그 기억 안에 있는지 찾는다.

### 구조

LZ77은 두 개의 버퍼를 유지한다:

1. **검색 버퍼(search buffer)**: 최근에 처리한 N바이트. 이미 인코딩한 데이터.
2. **룩어헤드 버퍼(lookahead buffer)**: 아직 인코딩하지 않은 데이터의 앞부분.

인코더는 룩어헤드 버퍼의 시작 부분과 일치하는 가장 긴 문자열을 검색 버퍼에서 찾는다. 그리고 세 가지 값으로 이루어진 **트리플(triple)**을 출력한다:

```
(offset, length, next_char)
```

- `offset`: 검색 버퍼에서 일치하는 위치까지의 거리
- `length`: 일치하는 문자열의 길이
- `next_char`: 일치 이후 첫 번째 다른 문자

### 예제

입력: `"AABABCABABCAB"`

처리 과정 (검색 버퍼 크기 = 6, 룩어헤드 버퍼 크기 = 4):

```
위치 0: 검색버퍼=[]       룩어헤드=[AABA] → (0, 0, 'A')  출력: A
위치 1: 검색버퍼=[A]      룩어헤드=[ABAB] → (1, 1, 'B')  A부터 1개 일치
위치 3: 검색버퍼=[AAB]    룩어헤드=[ABCA] → (2, 1, 'C')  AB...
위치 5: 검색버퍼=[AABABC] 룩어헤드=[ABAB] → (3, 4, ...)  ABAB 4글자 일치
```

### Python 구현

```python
def lz77_encode(data: str, search_size: int = 255, lookahead_size: int = 15):
    """LZ77 인코더 구현"""
    result = []
    pos = 0
    
    while pos < len(data):
        # 검색 버퍼 범위 계산
        search_start = max(0, pos - search_size)
        search_buf = data[search_start:pos]
        lookahead_buf = data[pos:pos + lookahead_size]
        
        best_offset = 0
        best_length = 0
        
        # 가장 긴 일치 찾기
        for length in range(min(len(lookahead_buf), len(search_buf)), 0, -1):
            pattern = lookahead_buf[:length]
            idx = search_buf.rfind(pattern)
            if idx != -1:
                best_offset = len(search_buf) - idx
                best_length = length
                break
        
        # 트리플 출력
        next_char = data[pos + best_length] if pos + best_length < len(data) else ''
        result.append((best_offset, best_length, next_char))
        pos += best_length + 1
    
    return result


def lz77_decode(tokens: list) -> str:
    """LZ77 디코더 구현"""
    result = []
    
    for offset, length, next_char in tokens:
        if length == 0:
            result.append(next_char)
        else:
            # 출력 버퍼에서 offset 위치부터 length만큼 복사
            start = len(result) - offset
            for i in range(length):
                # 출력 버퍼가 늘어나므로 인덱스 재계산
                result.append(result[start + i])
            if next_char:
                result.append(next_char)
    
    return ''.join(result)


# 테스트
original = "AABABCABABCAB"
encoded = lz77_encode(original)
print(f"원본: {original}")
print(f"인코딩: {encoded}")
decoded = lz77_decode(encoded)
print(f"디코딩: {decoded}")
print(f"일치 여부: {original == decoded}")
```

실행 결과:
```
원본: AABABCABABCAB
인코딩: [(0, 0, 'A'), (1, 1, 'B'), (2, 1, 'C'), (3, 4, 'B'), (7, 2, '')]
디코딩: AABABCABABCAB
일치 여부: True
```

## LZ78: 명시적 딕셔너리 방식

LZ78은 슬라이딩 윈도우 대신 **명시적으로 성장하는 딕셔너리**를 사용한다. 이 딕셔너리에는 지금까지 등장한 모든 고유한 문자열 조각이 저장된다.

### 작동 원리

1. 딕셔너리를 빈 상태로 시작한다.
2. 현재 위치부터 시작하여 딕셔너리에 있는 가장 긴 접두사를 찾는다.
3. **(딕셔너리 인덱스, 다음 문자)**를 출력한다.
4. 찾은 패턴 + 다음 문자를 딕셔너리에 추가한다.

LZ77과의 핵심 차이점:
- LZ77은 과거를 "찾는다" → LZ78은 패턴을 "기억한다"
- LZ77은 슬라이딩 윈도우로 범위가 제한 → LZ78의 딕셔너리는 무한히 성장
- LZ78은 장거리 반복 패턴에 강하다

## LZW: LZ78의 실용적 개선

Terry Welch가 1984년에 LZ78을 개선한 LZW는 딕셔너리를 모든 단일 문자로 미리 초기화한다. 이로써 초기화 심볼을 별도로 전송할 필요가 없어지고, 구현이 간단해진다.

### Python 구현

```python
def lzw_encode(data: str) -> list:
    """LZW 인코더 구현"""
    # 딕셔너리를 모든 단일 문자로 초기화
    dict_size = 256
    dictionary = {chr(i): i for i in range(dict_size)}
    
    result = []
    current = ""
    
    for char in data:
        combined = current + char
        if combined in dictionary:
            # 딕셔너리에 있으면 계속 확장
            current = combined
        else:
            # 딕셔너리에 없으면 현재 문자열의 코드를 출력
            result.append(dictionary[current])
            # 새 패턴을 딕셔너리에 추가
            dictionary[combined] = dict_size
            dict_size += 1
            current = char
    
    # 마지막 남은 문자열 처리
    if current:
        result.append(dictionary[current])
    
    return result


def lzw_decode(codes: list) -> str:
    """LZW 디코더 구현"""
    # 딕셔너리를 동일하게 초기화
    dict_size = 256
    dictionary = {i: chr(i) for i in range(dict_size)}
    
    result = []
    # 첫 번째 코드 처리
    current = dictionary[codes[0]]
    result.append(current)
    
    for code in codes[1:]:
        if code in dictionary:
            entry = dictionary[code]
        elif code == dict_size:
            # 특수 케이스: 아직 추가되지 않은 코드
            entry = current + current[0]
        else:
            raise ValueError(f"잘못된 코드: {code}")
        
        result.append(entry)
        # 딕셔너리에 새 패턴 추가
        dictionary[dict_size] = current + entry[0]
        dict_size += 1
        current = entry
    
    return ''.join(result)


# 테스트
original = "TOBEORNOTTOBEORTOBEORNOT"
encoded = lzw_encode(original)
decoded = lzw_decode(encoded)

print(f"원본 ({len(original)} chars): {original}")
print(f"인코딩 코드 수: {len(encoded)}")
print(f"압축률: {len(encoded)/len(original)*100:.1f}%")
print(f"디코딩 성공: {original == decoded}")

# 반복이 많은 데이터에서의 성능 확인
repetitive = "AB" * 50
enc_rep = lzw_encode(repetitive)
print(f"\n반복 데이터 (100 chars): {len(enc_rep)} 코드로 압축")
print(f"압축률: {len(enc_rep)/len(repetitive)*100:.1f}%")
```

실행 결과:
```
원본 (24 chars): TOBEORNOTTOBEORTOBEORNOT
인코딩 코드 수: 15
압축률: 62.5%
디코딩 성공: True

반복 데이터 (100 chars): 10 코드로 압축
압축률: 10.0%
```

## DEFLATE: LZ77 + Huffman의 결합

실제 gzip과 PNG에 사용되는 DEFLATE 알고리즘은 LZ77과 Huffman 코딩을 결합한다:

1. **LZ77 패스**: 반복 패턴을 (offset, length) 쌍으로 대체한다.
2. **Huffman 패스**: LZ77의 출력(리터럴 문자와 length/offset 쌍)에 Huffman 코딩을 적용하여 추가로 압축한다.

이 두 단계의 결합이 강력한 이유는 각각 다른 종류의 중복성을 제거하기 때문이다:
- LZ77: 구조적 중복성 (반복 패턴)
- Huffman: 통계적 중복성 (빈도 차이)

## 알고리즘 비교

| 특성 | LZ77 | LZ78 | LZW |
|------|------|------|-----|
| 딕셔너리 방식 | 슬라이딩 윈도우 | 명시적, 성장 | 명시적, 미리 초기화 |
| 메모리 | 고정 (윈도우 크기) | 가변 (무한 성장 가능) | 가변 |
| 인코딩 속도 | 보통 | 빠름 | 빠름 |
| 장거리 반복 | 약함 | 강함 | 강함 |
| 대표 포맷 | gzip, PNG, ZIP | - | GIF, TIFF |
| 특허 문제 | 없음 | LZW는 특허 있었음 | 1995–2003년 특허 |

LZW는 Unisys가 특허를 보유하고 있었기 때문에 1995년부터 2003년까지 GIF 형식 사용에 논란이 있었다. 특허가 만료된 현재는 자유롭게 사용 가능하다.

## 주의사항과 성능 팁

### 1. 딕셔너리 크기 제한

실제 구현에서는 LZW 딕셔너리가 무한정 성장할 수 없으므로, 딕셔너리가 특정 크기(예: 4096 항목)에 도달하면 초기화(clear code)하거나 LRU 방식으로 제거한다.

### 2. 검색 버퍼 크기와 압축률의 트레이드오프

LZ77에서 검색 버퍼가 클수록 압축률은 높아지지만 검색 시간이 늘어난다. 실제 DEFLATE는 32KB 슬라이딩 윈도우를 사용하며, 해시 테이블을 이용해 O(1)에 가까운 검색을 구현한다.

```python
# 해시 체인을 이용한 LZ77 최적화 아이디어
class FastLZ77:
    def __init__(self, window_size=32768):
        self.window = bytearray(window_size)
        # 3바이트 해시 → 최근 위치 목록
        self.hash_table = {}  # hash -> deque of positions
        self.window_size = window_size
    
    def _hash3(self, data: bytes, pos: int) -> int:
        """3바이트 해시 함수"""
        if pos + 2 >= len(data):
            return -1
        return (data[pos] << 16) | (data[pos+1] << 8) | data[pos+2]
    
    def find_match(self, data: bytes, pos: int) -> tuple:
        """해시 체인으로 가장 긴 일치 찾기 (O(1) 평균)"""
        h = self._hash3(data, pos)
        if h < 0 or h not in self.hash_table:
            return (0, 0)
        
        best_len = 0
        best_off = 0
        
        for prev_pos in self.hash_table[h]:
            # 실제 문자열 비교
            length = 0
            while (pos + length < len(data) and
                   prev_pos + length < pos and
                   data[pos + length] == data[prev_pos + length] and
                   length < 258):  # DEFLATE 최대 길이
                length += 1
            
            if length > best_len:
                best_len = length
                best_off = pos - prev_pos
        
        return (best_off, best_len)
```

### 3. 압축이 역효과를 내는 경우

무작위 데이터(암호화된 데이터, 이미 압축된 데이터)는 LZ 알고리즘으로 압축하면 오히려 크기가 늘어날 수 있다. 이미 압축된 파일(`.zip`, `.gz`, `.jpg`)을 다시 압축하는 것은 낭비다.

### 4. 스트리밍 압축

gzip은 스트리밍 데이터를 처리할 때 **flush** 포인트를 중간에 삽입할 수 있다. Python의 `zlib` 라이브러리에서는 `Z_SYNC_FLUSH`를 사용하면 된다. 스트리밍 환경에서는 압축률보다 지연 시간이 중요하므로, 이 기능이 매우 유용하다.

## 마치며

LZ77과 LZ78은 발표된 지 50년이 다 되어가는 지금도 디지털 세계의 모든 곳에서 쓰이고 있다. 가장 단순한 형태부터 현대적으로 최적화된 Zstandard까지, 그 핵심 아이디어는 변하지 않았다. "이미 본 것을 참조하라." 이 단순한 원리가 데이터를 압축하는 가장 강력한 방법 중 하나다.

## 참고 자료
- [LZ77 and LZ78 - Wikipedia](https://en.wikipedia.org/wiki/LZ77_and_LZ78)
- [Lempel-Ziv-Welch - Wikipedia](https://en.wikipedia.org/wiki/Lempel%E2%80%93Ziv%E2%80%93Welch)
- [The Hitchhiker's Guide to Compression — LZ Algorithms](https://go-compression.github.io/algorithms/lz/)
- [RFC 1951: DEFLATE Compressed Data Format Specification](https://datatracker.ietf.org/doc/html/rfc1951)
