---
layout: post
title: "Mo's Algorithm 완전 정복: 오프라인 범위 쿼리를 √N 블록으로 정복하는 법"
date: 2026-08-08
categories: [cs, computer-science]
tags: [algorithm, sqrt-decomposition, offline-query, range-query, competitive-programming]
---

## 개념 설명: Mo's Algorithm이란?

Mo's Algorithm은 인도의 경쟁 프로그래머 Mo Tao가 고안한 **오프라인 범위 쿼리(Offline Range Query) 최적화 기법**이다. 핵심 아이디어는 간단하다. 배열에 대한 여러 개의 범위 쿼리 `(l, r)`가 주어졌을 때, 쿼리를 입력 순서대로 처리하면 매번 구간을 재계산해야 하지만, **쿼리를 영리한 순서로 재배열하면** 전체 포인터 이동 횟수를 크게 줄일 수 있다.

### 핵심 원리: 블록 정렬

배열의 크기를 N, 쿼리 수를 Q라 하면, 블록 크기 B = √N 으로 배열을 나눈다. 이후 쿼리를 다음 기준으로 정렬한다:

1. **1차 정렬 기준**: 왼쪽 끝 인덱스 `l`가 속한 블록 번호 오름차순
2. **2차 정렬 기준**: 같은 블록 내에서는 홀수 블록이면 `r` 오름차순, 짝수 블록이면 `r` 내림차순 (힐버트 커브 최적화)

이 정렬 이후, 두 개의 포인터 `[curL, curR]`를 유지하며 현재 포인터를 다음 쿼리 범위로 확장하거나 축소한다. 각 이동마다 원소를 하나씩 추가(`add`)하거나 제거(`remove`)하는 연산이 실행된다.

### 시간복잡도 분석

- **오른쪽 포인터 r 이동**: 같은 블록 내 쿼리들은 r이 단조증가(혹은 단조감소)하므로 총 O(N) 이동
- **왼쪽 포인터 l 이동**: 블록이 바뀔 때마다 최대 O(√N) 이동 × Q개의 쿼리 = O(Q√N)
- **전체 시간복잡도**: **O((N + Q) × √N)**

단순하게 매 쿼리마다 O(N) 재계산하는 것이 O(NQ)인 점을 감안하면, Q ≈ N일 때 O(N√N)으로 크게 개선된다. N = 10^5이면 단순 방법은 10^10, Mo's는 약 3.2 × 10^7이다.

---

## 왜 필요한가?

범위 쿼리 문제는 대표적으로 **구간 합**, **구간 내 서로 다른 원소의 개수**, **구간 최빈값** 등을 묻는다. 업데이트가 없는 정적 배열에서 쿼리만 다수 주어지는 상황이라면, 세그먼트 트리나 펜윅 트리가 각 쿼리를 O(log N)에 처리하지만, **추가/제거 연산을 빠르게 정의할 수 없는 문제** — 예를 들어 구간 내 서로 다른 원소 수를 세는 문제 — 에서는 이런 자료구조가 적용되지 않는다.

Mo's Algorithm은 추가/제거가 O(1) 혹은 O(log N)에 가능한 상황에서 오프라인으로 쿼리를 효율적으로 처리할 수 있게 해준다. 쿼리가 미리 알려져 있고(오프라인), 배열이 업데이트되지 않는다는 제약이 있지만, 해당 조건 내에서는 매우 강력한 도구다.

---

## 실제 구현 예제

### 예제 1: 구간 내 서로 다른 원소 수 세기 (C++)

가장 대표적인 Mo's Algorithm 응용 문제다. 배열 A에 대한 Q개의 쿼리 `(l, r)`에서 A[l..r] 내 서로 다른 값의 개수를 구한다.

```cpp
#include <bits/stdc++.h>
using namespace std;

const int MAXN = 100005;
int arr[MAXN], cnt[MAXN];
int n, q;
int cur_ans = 0;

// 원소 추가: cnt가 0->1이 되면 서로 다른 원소 수 증가
void add(int idx) {
    if (cnt[arr[idx]]++ == 0) cur_ans++;
}

// 원소 제거: cnt가 1->0이 되면 서로 다른 원소 수 감소
void rem(int idx) {
    if (--cnt[arr[idx]] == 0) cur_ans--;
}

struct Query {
    int l, r, idx;
};

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(NULL);

    cin >> n >> q;
    for (int i = 0; i < n; i++) cin >> arr[i];

    int block = max(1, (int)sqrt(n));
    vector<Query> queries(q);
    for (int i = 0; i < q; i++) {
        cin >> queries[i].l >> queries[i].r;
        queries[i].l--; queries[i].r--;  // 0-indexed
        queries[i].idx = i;
    }

    // Mo's 정렬: 블록 번호 오름차순, 같은 블록 내 힐버트 커브 최적화
    sort(queries.begin(), queries.end(), [&](const Query& a, const Query& b) {
        int ba = a.l / block, bb = b.l / block;
        if (ba != bb) return ba < bb;
        return (ba & 1) ? (a.r > b.r) : (a.r < b.r);
    });

    vector<int> ans(q);
    int curL = 0, curR = -1;

    for (auto& qr : queries) {
        // 포인터 확장/축소
        while (curR < qr.r) add(++curR);
        while (curL > qr.l) add(--curL);
        while (curR > qr.r) rem(curR--);
        while (curL < qr.l) rem(curL++);
        ans[qr.idx] = cur_ans;
    }

    for (int i = 0; i < q; i++) cout << ans[i] << "\n";
    return 0;
}
```

**포인터 확장 순서 주의**: curR 확장 → curL 확장 → curR 축소 → curL 축소 순서를 지켜야 포인터가 겹치는 경우(curL > curR + 1)를 예방할 수 있다.

---

### 예제 2: Mo's Algorithm Python 구현 — 구간 합

Python으로 구현할 때는 속도 한계 때문에 PyPy를 사용하거나 연산을 최소화해야 한다.

```python
import sys
from math import isqrt

def solve():
    input_data = sys.stdin.buffer.read().split()
    ptr = 0
    n = int(input_data[ptr]); ptr += 1
    arr = [int(input_data[ptr + i]) for i in range(n)]; ptr += n
    q = int(input_data[ptr]); ptr += 1

    queries = []
    for i in range(q):
        l = int(input_data[ptr]) - 1; ptr += 1
        r = int(input_data[ptr]) - 1; ptr += 1
        queries.append((l, r, i))

    block = max(1, isqrt(n))

    # Mo's 정렬
    queries.sort(key=lambda x: (x[0] // block, x[1] if (x[0] // block) % 2 == 0 else -x[1]))

    ans = [0] * q
    cur_sum = 0
    cur_l, cur_r = 0, -1

    for l, r, idx in queries:
        while cur_r < r:
            cur_r += 1
            cur_sum += arr[cur_r]
        while cur_l > l:
            cur_l -= 1
            cur_sum += arr[cur_l]
        while cur_r > r:
            cur_sum -= arr[cur_r]
            cur_r -= 1
        while cur_l < l:
            cur_sum -= arr[cur_l]
            cur_l += 1
        ans[idx] = cur_sum

    sys.stdout.write("\n".join(map(str, ans)) + "\n")

solve()
```

---

## 주의사항 및 팁

### 1. 오프라인 전용

Mo's Algorithm은 쿼리가 미리 모두 주어지는 **오프라인 환경**에서만 사용 가능하다. 쿼리 결과가 다음 쿼리에 영향을 주는 경우(온라인 쿼리)나 배열 업데이트가 있는 경우에는 Mo with Updates(업데이트 포함 버전, O(N^(5/3)))를 사용해야 한다.

### 2. add/remove 함수의 O(1) 달성이 핵심

Mo's Algorithm의 전체 이동 횟수는 O((N + Q)√N)이므로, add/remove가 O(1)이면 전체 O((N+Q)√N)이 달성된다. 만약 add/remove가 O(log N)이면 전체 O((N+Q)√N × log N)이 된다. 문제에 따라 add/remove 설계가 알고리즘의 성능을 결정한다.

### 3. 블록 크기 튜닝

블록 크기 B = √N이 이론적 최적이지만, 실제로는 `max(1, (int)sqrt((long long)n * n / q))`로 쿼리 수 Q에 맞게 조절하면 성능이 향상되는 경우가 있다. 특히 Q ≪ N일 때 유용하다.

### 4. 힐버트 커브 정렬로 상수 개선

기본 Mo's 정렬 대신 힐버트 커브(Hilbert curve) 순서로 쿼리를 정렬하면 포인터 이동 거리의 기댓값이 줄어 실제 수행 시간이 1.5~2배 향상된다. 경쟁 프로그래밍 대회에서 TLE가 날 경우 시도할 수 있다.

### 5. 좌표 압축

원소의 범위가 크면 `cnt` 배열 대신 `unordered_map`을 사용하거나, 사전에 **좌표 압축(coordinate compression)**을 적용해 값의 범위를 Q 이내로 줄이면 메모리와 캐시 효율이 개선된다.

---

## 참고 자료

- [OmarBazaraa - Mo's Algorithm C++ 구현](https://github.com/OmarBazaraa/Competitive-Programming/blob/master/src/data_structures/sqrt_decomposition/mo_algorithm.cpp)
- [jyuv - Mos-Algorithm Python 구현 및 인터페이스](https://github.com/jyuv/Mos-Algorithm)
- [joy-mollick - Sqrt Decomposition & Mo's Algorithm 문제 모음](https://github.com/joy-mollick/Sqrt-Decomposition-and-Mo-s-algorithm-problems.)
