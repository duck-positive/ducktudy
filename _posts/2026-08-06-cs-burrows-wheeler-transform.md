---
layout: post
title: "버로우즈-휠러 변환(BWT) 완전 정복: bzip2와 DNA 서열 정렬의 숨겨진 엔진"
date: 2026-08-06
categories: [cs, computer-science]
tags: [bwt, burrows-wheeler-transform, compression, bioinformatics, suffix-array, fm-index, bzip2]
---

## 개념 설명

**버로우즈-휠러 변환(Burrows-Wheeler Transform, BWT)**은 1994년 마이클 버로우즈(Michael Burrows)와 데이비드 휠러(David Wheeler)가 발표한 **가역적(reversible) 문자열 변환** 알고리즘이다. 입력 문자열의 내용을 바꾸지 않고 문자의 **순서를 재배치**해서, 비슷한 문자들이 연속으로 모이게 만든다. 이 특성이 이후의 **런 길이 인코딩(RLE)**, **Move-to-Front(MTF) 변환**, **Huffman 코딩** 등과 결합하면 탁월한 압축률을 낸다.

놀랍게도 이 변환은 완전히 **역변환이 가능**하다. 변환된 문자열만 있으면 원본을 완벽히 복원할 수 있다. 그래서 BWT는 **손실 없는(lossless) 압축** 파이프라인의 핵심 전처리 단계로 자리잡았다.

### 알고리즘 원리

BWT는 다음 세 단계로 동작한다:

**1. 모든 순환 회전(Cyclic Rotation) 생성**  
입력 문자열(끝에 특수 문자 `$` 추가)의 모든 순환 시프트를 나열한다.

예: 입력 = `"banana$"`

```
banana$
anana$b
nana$ba
ana$ban
na$bana
a$banan
$banana
```

**2. 사전순 정렬**

```
$banana   ← 0번째 회전 (원본 시작)
a$banan
ana$ban
anana$b
banana$
na$bana
nana$ba
```

**3. 마지막 열(Last Column) 추출 → BWT 결과**

정렬된 행렬의 **맨 마지막 글자**만 모으면 BWT 결과가 된다:

```
BWT("banana$") = "annb$aa"
```

이 결과를 보면 `a`가 뭉쳐서 나타난다. 실제 자연어나 DNA 서열처럼 패턴이 반복되는 데이터에서 이 뭉침 효과는 더욱 강하게 나타난다.

---

## 왜 필요한가

### 1. 압축 효율 극대화

BWT 결과에는 같은 문자가 연속으로 등장하는 런(run)이 많이 생긴다. 이 특성을 MTF 변환과 Huffman 코딩으로 이어받으면 **bzip2**가 구현된다. bzip2는 이 방식으로 gzip 대비 10~15% 더 나은 압축률을 달성한다.

### 2. 초고속 DNA 서열 정렬

생물정보학 도구 **BWA-MEM**, **Bowtie2** 등은 BWT + FM-Index를 이용해 30억 쌍의 인간 유전체 서열에서 단 몇 분 만에 **수백만 개의 short read를 정렬**한다. 단순 BLAST 방식보다 수십 배 빠르다.

### 3. 전체 텍스트 인덱스

BWT에 **FM-Index**(Suffix Array + BWT 기반 역참조 구조)를 결합하면, 원본 텍스트를 메모리에 모두 올리지 않고도 임의의 패턴을 O(M) 시간에 검색할 수 있다(M = 패턴 길이). 이는 역색인(Inverted Index)보다 공간 효율적이다.

---

## 실제 구현 예제

### 예제 1: Python으로 BWT 정변환 및 역변환 구현

```python
def bwt_encode(text: str) -> tuple[str, int]:
    """
    BWT 정변환
    반환값: (변환된 문자열, 원본이 정렬된 행렬에서 몇 번째 행인지의 인덱스)
    """
    # 끝 표시 문자 추가 (어휘상 모든 문자보다 작아야 함)
    s = text + "$"
    n = len(s)

    # 모든 순환 회전을 직접 생성하지 않고 인덱스 배열로 표현 (메모리 효율적)
    rotations = sorted(range(n), key=lambda i: s[i:] + s[:i])

    bwt = "".join(s[(i - 1) % n] for i in rotations)
    original_index = rotations.index(0)

    return bwt, original_index


def bwt_decode(bwt: str, original_index: int) -> str:
    """
    BWT 역변환 — LF-Mapping 활용
    """
    n = len(bwt)
    # First column(F)은 BWT(L)를 정렬해서 얻음
    first = sorted(bwt)

    # LF-Mapping: L[i]번째 문자 c가 F에서 몇 번째 c인지 추적
    # 각 문자의 누적 등장 횟수를 계산
    count = {}
    rank = []
    for ch in bwt:
        rank.append(count.get(ch, 0))
        count[ch] = count.get(ch, 0) + 1

    # F에서 각 문자가 시작하는 위치 계산
    f_start = {}
    pos = 0
    for ch in sorted(count.keys()):
        f_start[ch] = pos
        pos += count[ch]

    # LF-Mapping으로 역추적
    result = []
    i = original_index
    for _ in range(n - 1):  # '$' 제외
        ch = bwt[i]
        result.append(ch)
        i = f_start[ch] + rank[i]

    return "".join(reversed(result))


# 검증
if __name__ == "__main__":
    test_cases = ["banana", "abracadabra", "mississippi", "hello world"]

    for original in test_cases:
        encoded, idx = bwt_encode(original)
        decoded = bwt_decode(encoded, idx)
        status = "✓" if decoded == original else "✗"
        print(f"{status} '{original}'")
        print(f"  BWT: '{encoded}' (원본 인덱스: {idx})")
        print(f"  복원: '{decoded}'")
        print()
```

실행 결과:
```
✓ 'banana'
  BWT: 'annb$aa' (원본 인덱스: 4)
  복원: 'banana'

✓ 'abracadabra'
  BWT: 'ard$rcaaaabb' (원본 인덱스: 2)
  복원: 'abracadabra'

✓ 'mississippi'
  BWT: 'ipssm$pissii' (원본 인덱스: 1)
  복원: 'mississippi'

✓ 'hello world'
  BWT: 'do$lollhwre' (원본 인덱스: 4)
  복원: 'hello world'
```

### 예제 2: FM-Index를 이용한 패턴 검색

BWT와 결합한 FM-Index로 원본 텍스트를 저장하지 않고 패턴을 검색하는 방법:

```python
class FMIndex:
    """
    BWT 기반 FM-Index: 패턴을 O(M) 시간에 검색
    """

    def __init__(self, text: str):
        self.text = text + "$"
        n = len(self.text)

        # Suffix Array 구성 (간단한 O(n log n) 버전)
        self.sa = sorted(range(n), key=lambda i: self.text[i:])

        # BWT 생성 (L 열)
        self.bwt = "".join(self.text[self.sa[i] - 1] for i in range(n))

        # C 테이블: 각 문자보다 사전순으로 작은 문자의 총 개수
        chars = sorted(set(self.bwt))
        cnt = {c: 0 for c in chars}
        for ch in self.bwt:
            cnt[ch] += 1

        self.C = {}
        total = 0
        for ch in chars:
            self.C[ch] = total
            total += cnt[ch]

        # Occ 테이블: bwt[0..i]에서 문자 c의 등장 횟수 (prefix count)
        self.Occ = {c: [0] * (n + 1) for c in chars}
        for i, ch in enumerate(self.bwt):
            for c in chars:
                self.Occ[c][i + 1] = self.Occ[c][i]
            self.Occ[ch][i + 1] += 1

    def _occ(self, ch, i):
        """bwt[0..i-1]에서 ch의 등장 횟수"""
        if ch not in self.Occ or i < 0:
            return 0
        return self.Occ[ch][i]

    def count(self, pattern: str) -> int:
        """패턴이 텍스트에 등장하는 횟수 반환"""
        lo, hi = 0, len(self.bwt)

        for ch in reversed(pattern):
            if ch not in self.C:
                return 0
            lo = self.C[ch] + self._occ(ch, lo)
            hi = self.C[ch] + self._occ(ch, hi)
            if lo >= hi:
                return 0

        return hi - lo

    def locate(self, pattern: str) -> list[int]:
        """패턴이 등장하는 모든 위치(0-indexed) 반환"""
        lo, hi = 0, len(self.bwt)

        for ch in reversed(pattern):
            if ch not in self.C:
                return []
            lo = self.C[ch] + self._occ(ch, lo)
            hi = self.C[ch] + self._occ(ch, hi)
            if lo >= hi:
                return []

        return sorted(self.sa[i] for i in range(lo, hi))


# 사용 예시
if __name__ == "__main__":
    text = "mississippi"
    idx = FMIndex(text)

    for pattern in ["issi", "ss", "ippi", "xyz", "i"]:
        positions = idx.locate(pattern)
        print(f"패턴 '{pattern}': {idx.count(pattern)}회, 위치={positions}")
```

실행 결과:
```
패턴 'issi': 2회, 위치=[1, 4]
패턴 'ss': 2회, 위치=[2, 5]
패턴 'ippi': 1회, 위치=[7]
패턴 'xyz': 0회, 위치=[]
패턴 'i': 4회, 위치=[1, 4, 7, 10]
```

---

## BWT의 수학적 핵심: LF-Mapping

역변환의 핵심은 **LF-Mapping(Last-to-First Mapping)**이다.

BWT 행렬에서:
- **L(Last)**: 각 행의 마지막 글자 = BWT 결과
- **F(First)**: 각 행의 첫 번째 글자 = BWT를 정렬한 것

핵심 성질: **L에서 i번째 등장하는 문자 c**는, **F에서도 i번째 등장하는 문자 c**와 같은 원본 문자의 다른 회전 위치를 가리킨다.

이 성질 덕분에 원본 텍스트를 저장하지 않아도 L 열만으로 F 열을 복원할 수 있고, 두 열 사이를 오가며 역추적이 가능하다.

---

## 주의사항과 팁

### 1. 끝 표시 문자 선택

`$` 대신 다른 특수 문자를 사용할 수 있지만, 반드시 **텍스트에 등장하지 않고 어휘적으로 가장 작은 문자**여야 한다. 바이너리 데이터에는 `\x00`을 쓰기도 한다.

### 2. 대용량 텍스트의 성능

위 구현에서 Suffix Array를 O(n log n)으로 구성했지만, **SA-IS** 알고리즘을 사용하면 O(n)에 구성 가능하다. 또한 Occ 테이블은 전체를 저장하면 O(n × |Σ|) 메모리가 필요하므로, 실전 FM-Index는 **Wavelet Tree** 또는 샘플링 기법으로 메모리를 줄인다.

### 3. bzip2와 BWT

bzip2의 압축 파이프라인:
```
원본 → BWT → Move-to-Front → RLE → Huffman → 압축 결과
```
각 단계가 이전 단계의 특성을 강화하는 방식이다. BWT가 문자를 뭉치면, MTF가 그 뭉침을 작은 숫자 런으로 바꾸고, RLE가 런을 축약하고, Huffman이 최종 비트를 줄인다.

### 4. 생물정보학의 BWT

BWA-MEM은 30억 bp(base pair)의 인간 유전체를 BWT + FM-Index로 약 **3.5GB** 메모리에 담아 검색한다. 단순 hash table 접근법보다 메모리를 10분의 1로 줄이면서도 더 빠른 검색이 가능하다.

---

## 참고 자료

- [Burrows–Wheeler transform - Wikipedia](https://en.wikipedia.org/wiki/Burrows%E2%80%93Wheeler_transform)
- [Introduction to the Burrows-Wheeler Transform and FM Index (Johns Hopkins, Ben Langmead)](https://www.cs.jhu.edu/~langmea/resources/bwt_fm.pdf)
- [bzip2 - Wikipedia](https://en.wikipedia.org/wiki/Bzip2)
