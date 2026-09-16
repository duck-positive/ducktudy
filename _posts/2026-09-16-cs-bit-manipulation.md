---
layout: post
title: "비트 조작 완전 정복: 산술 연산을 비트로 대체하는 고속 프로그래밍 기법"
date: 2026-09-16
categories: [cs, computer-science]
tags: [bit-manipulation, algorithm, optimization, low-level, bitwise]
---

## 개념 설명

**비트 조작(Bit Manipulation)**은 개별 비트를 직접 읽고, 쓰고, 뒤집는 연산 기법입니다. CPU는 비트 연산을 단일 클럭 사이클에 처리하므로, 적절히 활용하면 분기(if/else)와 곱셈·나눗셈을 없애고 수십 배의 성능 향상을 끌어낼 수 있습니다.

### 기본 비트 연산자

| 연산자 | 기호 | 설명 | 예시 |
|--------|------|------|------|
| AND | `&` | 두 비트 모두 1이면 1 | `0b1010 & 0b1100 = 0b1000` |
| OR | `\|` | 둘 중 하나라도 1이면 1 | `0b1010 \| 0b1100 = 0b1110` |
| XOR | `^` | 두 비트가 다르면 1 | `0b1010 ^ 0b1100 = 0b0110` |
| NOT | `~` | 비트 반전 | `~0b1010 = 0b...10101` |
| 왼쪽 시프트 | `<<` | 비트를 왼쪽으로 이동 (×2) | `0b0001 << 3 = 0b1000` |
| 오른쪽 시프트 | `>>` | 비트를 오른쪽으로 이동 (÷2) | `0b1000 >> 2 = 0b0010` |

### 핵심 비트 조작 공식

```
// k번째 비트 읽기 (0-indexed, LSB부터)
bit = (x >> k) & 1

// k번째 비트 세우기 (Set)
x |= (1 << k)

// k번째 비트 지우기 (Clear)
x &= ~(1 << k)

// k번째 비트 토글 (Toggle)
x ^= (1 << k)

// 최하위 세트 비트(LSB) 추출
lsb = x & (-x)      // -x는 2의 보수: 최하위 1비트만 남음

// 최하위 세트 비트 제거
x &= (x - 1)        // x-1은 최하위 1비트와 그 아래를 뒤집음

// 모든 비트가 0이면 true
is_zero = (x == 0)

// x가 2의 거듭제곱인지 확인
is_power_of_two = (x > 0) && ((x & (x - 1)) == 0)
```

---

## 왜 필요한가

**1. 성능**: 현대 CPU에서 비트 연산은 나눗셈(20~100 사이클)보다 훨씬 빠릅니다(1~2 사이클). 내부 루프에서 차이가 극명하게 나타납니다.

**2. 공간 효율**: 64개의 bool 값을 64바이트가 아닌 8바이트(64비트 정수 하나)에 저장합니다. 비트마스크(Bitmask)는 집합 연산을 AND/OR/XOR 한 번으로 처리합니다.

**3. SIMD와 시너지**: 현대 CPU의 SIMD 명령어는 여러 비트를 병렬 처리합니다. 비트 조작을 이해하면 벡터 연산 최적화의 기반이 됩니다.

**4. 시스템 프로그래밍 필수**: 네트워크 프로토콜의 헤더 파싱, 하드웨어 레지스터 제어, 압축 알고리즘, 암호화에서 비트 조작은 불가결합니다.

**5. 경쟁 프로그래밍**: 부분집합 열거, 비트 DP, 상태 압축 등에서 비트마스크 없이는 풀 수 없는 문제가 다수입니다.

---

## 실제 구현 예제

### 예제 1: 핵심 비트 조작 기법 모음 (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

// ─── 기본 기법 ───────────────────────────────────────────────

// 짝수/홀수 판별 (나머지 연산 대신)
bool isOdd(int n) { return n & 1; }

// 절댓값 (분기 없이)
int absNoIf(int n) {
    int mask = n >> 31; // 음수면 -1(0xFF...FF), 양수면 0
    return (n + mask) ^ mask;
}

// 두 수 스왑 (임시 변수 없이)
void swapNoTemp(int& a, int& b) {
    a ^= b;
    b ^= a;
    a ^= b;
}

// 두 수 중 최솟값 (분기 없이)
int minNoBranch(int a, int b) {
    return b + ((a - b) & ((a - b) >> 31));
}

// ─── 비트 카운팅 ─────────────────────────────────────────────

// Brian Kernighan's 알고리즘: 1의 개수 세기
int countBits(int n) {
    int count = 0;
    while (n) {
        n &= (n - 1); // 최하위 1비트 제거
        count++;
    }
    return count;
}

// GCC 내장 함수 (하드웨어 POPCNT 명령어 활용)
int countBitsBuiltin(int n) { return __builtin_popcount(n); }

// ─── 2의 거듭제곱 관련 ────────────────────────────────────────

// N 이상의 가장 작은 2의 거듭제곱
uint32_t nextPowerOfTwo(uint32_t n) {
    if (n == 0) return 1;
    n--;
    n |= n >> 1;
    n |= n >> 2;
    n |= n >> 4;
    n |= n >> 8;
    n |= n >> 16;
    return n + 1;
}

// log2(n) 계산 (n이 2의 거듭제곱일 때 정확)
int log2Floor(int n) { return 31 - __builtin_clz(n); } // CLZ: 선행 0의 수

// ─── 부분집합 열거 ────────────────────────────────────────────

// 비트마스크 S의 모든 부분집합 열거 (O(2^|S|))
void enumerateSubsets(int S) {
    for (int sub = S; sub > 0; sub = (sub - 1) & S) {
        // sub은 S의 부분집합
        cout << bitset<8>(sub) << "\n";
    }
    // 공집합도 포함 (위에서 sub=0일 때 루프 종료)
    cout << bitset<8>(0) << "\n";
}

// ─── 응용: 비트DP로 최소 방문 순열 비용 (TSP 축소판) ────────

// dp[mask][v] = mask의 도시들을 방문하고 v에서 끝나는 최소 비용
int tspDP(int n, vector<vector<int>>& cost) {
    int FULL = (1 << n) - 1;
    vector<vector<int>> dp(1 << n, vector<int>(n, INT_MAX));
    dp[1][0] = 0; // 도시 0에서 시작

    for (int mask = 1; mask <= FULL; mask++) {
        for (int u = 0; u < n; u++) {
            if (!(mask & (1 << u))) continue;
            if (dp[mask][u] == INT_MAX) continue;

            for (int v = 0; v < n; v++) {
                if (mask & (1 << v)) continue; // 이미 방문
                int newMask = mask | (1 << v);
                dp[newMask][v] = min(dp[newMask][v],
                                     dp[mask][u] + cost[u][v]);
            }
        }
    }

    int ans = INT_MAX;
    for (int u = 1; u < n; u++) {
        if (dp[FULL][u] != INT_MAX)
            ans = min(ans, dp[FULL][u] + cost[u][0]);
    }
    return ans;
}

int main() {
    // 기본 기법 테스트
    cout << "7은 홀수? " << (isOdd(7) ? "예" : "아니오") << "\n";
    cout << "8은 2의 제곱? " << ((__builtin_popcount(8) == 1) ? "예" : "아니오") << "\n";
    cout << "13의 비트 1 개수: " << countBitsBuiltin(13) << "\n"; // 13=1101b → 3
    cout << "15 이상 최소 2의 제곱: " << nextPowerOfTwo(15) << "\n"; // 16

    cout << "\n5(0b0101)의 모든 부분집합:\n";
    enumerateSubsets(5);

    return 0;
}
```

---

### 예제 2: 실용적인 비트 조작 응용 (Python)

```python
# ─── 집합 연산을 비트마스크로 ────────────────────────────────

class BitSet:
    """비트마스크 기반 집합. 원소는 0부터 시작하는 정수 인덱스."""
    
    def __init__(self, max_size: int = 64):
        self._bits = 0
        self._size = max_size
    
    def add(self, x: int) -> None:
        self._bits |= (1 << x)
    
    def remove(self, x: int) -> None:
        self._bits &= ~(1 << x)
    
    def contains(self, x: int) -> bool:
        return bool(self._bits & (1 << x))
    
    def union(self, other: 'BitSet') -> 'BitSet':
        result = BitSet(self._size)
        result._bits = self._bits | other._bits
        return result
    
    def intersection(self, other: 'BitSet') -> 'BitSet':
        result = BitSet(self._size)
        result._bits = self._bits & other._bits
        return result
    
    def difference(self, other: 'BitSet') -> 'BitSet':
        result = BitSet(self._size)
        result._bits = self._bits & ~other._bits
        return result
    
    def size(self) -> int:
        return bin(self._bits).count('1')
    
    def __repr__(self) -> str:
        return f"BitSet({{{', '.join(str(i) for i in range(self._size) if self.contains(i))}}})"


# ─── 비트 조작 유틸리티 ──────────────────────────────────────

def gray_code(n: int) -> int:
    """그레이 코드: 인접한 수가 1비트만 다름 (로터리 인코더 등에 활용)"""
    return n ^ (n >> 1)

def reverse_gray_code(g: int) -> int:
    """그레이 코드에서 원래 수 복원"""
    n = g
    n ^= (n >> 16)
    n ^= (n >> 8)
    n ^= (n >> 4)
    n ^= (n >> 2)
    n ^= (n >> 1)
    return n

def parity(x: int) -> int:
    """1비트의 개수가 홀수면 1, 짝수면 0 (에러 검출에 활용)"""
    x ^= x >> 32
    x ^= x >> 16
    x ^= x >> 8
    x ^= x >> 4
    x ^= x >> 2
    x ^= x >> 1
    return x & 1

def bit_reverse(x: int, bits: int = 32) -> int:
    """비트 순서 반전 (FFT 나비 연산 등에 활용)"""
    result = 0
    for _ in range(bits):
        result = (result << 1) | (x & 1)
        x >>= 1
    return result

def nth_bit_permutation(mask: int) -> int:
    """
    Gosper's Hack: 같은 팝카운트의 다음 순열 비트마스크.
    예: 0b0111 → 0b1011 → 0b1101 → 0b1110
    경쟁 프로그래밍에서 크기 k인 부분집합 전체 열거에 활용.
    """
    c = mask & -mask          # 최하위 1비트
    r = mask + c              # 연속된 1비트 블록을 다음 위치로 올림
    return (((r ^ mask) >> 2) // c) | r

# ─── 상태 압축 DP 예시: 부분집합 합 ─────────────────────────

def subset_sum_exists(nums: list, target: int) -> bool:
    """
    비트마스크 DP로 부분집합 합 존재 여부 확인.
    dp는 정수 하나로 표현: i번째 비트가 1이면 합 i가 가능.
    """
    dp = 1  # 비트 0이 세트: 합 0은 항상 가능
    for num in nums:
        dp |= dp << num  # dp의 모든 도달 가능 합에 num을 더한 값도 도달 가능
    return bool(dp & (1 << target))

# ─── 테스트 ─────────────────────────────────────────────────

# 비트셋 테스트
a = BitSet()
b = BitSet()
for x in [1, 3, 5, 7]: a.add(x)
for x in [3, 5, 9]: b.add(x)
print(f"A: {a}")
print(f"B: {b}")
print(f"A ∪ B: {a.union(b)}")
print(f"A ∩ B: {a.intersection(b)}")
print(f"A - B: {a.difference(b)}")

# 그레이 코드
print("\n그레이 코드 (0~7):")
for i in range(8):
    g = gray_code(i)
    print(f"  {i:03b} → {g:03b}")

# 부분집합 합
nums = [3, 1, 4, 1, 5]
print(f"\n{nums}에서 합 9 가능? {subset_sum_exists(nums, 9)}")  # True
print(f"{nums}에서 합 10 가능? {subset_sum_exists(nums, 10)}") # True (3+1+1+5)

# Gosper's Hack으로 크기 2인 부분집합 열거
print("\n4개 원소에서 크기 2인 부분집합:")
mask = 0b0011  # 초기: {0, 1}
while mask < (1 << 4):
    selected = [i for i in range(4) if mask & (1 << i)]
    print(f"  {mask:04b} → {selected}")
    mask = nth_bit_permutation(mask)
```

---

## 주의사항과 팁

### 1. 부호 있는 정수의 오른쪽 시프트

C/C++에서 **부호 있는 정수의 오른쪽 시프트**는 구현 의존적입니다. 음수에서 산술 시프트(부호 비트 복사)가 일반적이지만, 이식성을 위해 `(unsigned)n >> k`를 사용하거나 명확히 캐스팅하세요. Java는 `>>` (산술), `>>>` (논리) 를 명확히 구분합니다.

### 2. 정수 오버플로우 주의

`1 << 31`은 C/C++에서 32비트 int 오버플로우가 발생합니다. 64비트 값이 필요하면 `1LL << 31` 또는 `1ULL << 31`을 사용하세요.

### 3. XOR 스왑의 위험성

`a ^= b; b ^= a; a ^= b;`는 a와 b가 **같은 메모리 위치**를 가리킬 때(자기 자신과 XOR) 0이 됩니다. `swap(a, a)` 같은 상황에서 문제가 생기므로 현대 컴파일러 최적화가 있는 경우 일반 스왑이 더 안전합니다.

### 4. 내장 함수 활용

| 함수 | GCC | Java | Python |
|------|-----|------|--------|
| 팝카운트 | `__builtin_popcount(x)` | `Integer.bitCount(x)` | `bin(x).count('1')` |
| 선행 0 수 | `__builtin_clz(x)` | `Integer.numberOfLeadingZeros(x)` | `x.bit_length()` |
| 후행 0 수 | `__builtin_ctz(x)` | `Integer.numberOfTrailingZeros(x)` | `(x & -x).bit_length() - 1` |

### 5. 비트 조작이 항상 빠르지는 않다

컴파일러의 최적화 능력이 발전하여 **명확한 코드**가 종종 비트 조작 코드와 동일한 어셈블리를 생성합니다. 가독성 vs 성능의 트레이드오프를 고려하세요. 프로파일링을 먼저 하고 최적화하세요.

### 6. 비트마스크 DP의 복잡도

상태 압축 DP에서 마스크를 사용하면 O(2^N × N)의 복잡도를 가집니다. N=20이면 약 2000만 연산으로 실용적이지만, N=30이면 10억으로 한계에 달합니다. N ≤ 25 이하가 일반적인 적용 범위입니다.

### 7. 자주 쓰이는 패턴 모음

```cpp
// 모든 비트가 1인 k비트 마스크
int mask_k = (1 << k) - 1;

// i번째부터 j번째까지의 비트만 추출 (0-indexed)
int bits_i_to_j = (x >> i) & ((1 << (j - i + 1)) - 1);

// n을 k의 배수로 올림 (k가 2의 거듭제곱일 때)
int ceil_to_k = (n + k - 1) & ~(k - 1);

// n을 k의 배수로 내림
int floor_to_k = n & ~(k - 1);

// 최하위 n개 비트 세트
int low_n = (1 << n) - 1;

// 부호 없는 정수의 나머지 (k가 2의 거듭제곱)
unsigned mod_k = x & (k - 1);

// 두 수의 평균 (오버플로우 없이)
int avg = (a & b) + ((a ^ b) >> 1);
```

비트 조작은 단순해 보이지만 그 위에 복잡한 알고리즘 최적화가 쌓입니다. 기본 공식을 외우는 것보다 **왜 작동하는지**를 이해하면, 새로운 상황에서도 적절한 비트 연산을 도출할 수 있습니다.

## 참고 자료
- [Bit Manipulation - Algorithms for Competitive Programming](https://cp-algorithms.com/algebra/bit-manipulation.html)
- [Bit Manipulation - Wikipedia](https://en.wikipedia.org/wiki/Bit_manipulation)
- [Bits manipulation tactics - GeeksforGeeks](https://www.geeksforgeeks.org/dsa/bits-manipulation-important-tactics/)
- [Bit Manipulation Hacks - Brilliant](https://brilliant.org/wiki/bit-manipulation-hacks/)
