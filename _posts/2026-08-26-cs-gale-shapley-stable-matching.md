---
layout: post
title: "Gale-Shapley 안정 매칭 알고리즘 심층 분석"
date: 2026-08-26
categories: [cs, computer-science]
tags: [algorithm, matching, graph, gale-shapley, stable-matching, bipartite]
---

## 안정 매칭 문제란 무엇인가

안정 매칭(Stable Matching) 문제는 두 집합 간의 1:1 대응 관계를 구성할 때, 어느 쌍도 현재 파트너보다 서로를 더 선호하는 "불안정한" 쌍이 존재하지 않도록 매칭하는 문제입니다. 1962년 수학자 David Gale과 Lloyd Shapley가 처음 공식적으로 제안했으며, Shapley는 이 연구로 2012년 Alvin Roth와 함께 노벨 경제학상을 수상했습니다.

가장 고전적인 형태는 **안정 결혼 문제(Stable Marriage Problem)**입니다. n명의 남성과 n명의 여성이 있고, 각자 상대 집합의 모든 구성원에 대한 선호도 순위를 가지고 있을 때, 모든 참가자가 매칭된 완전 매칭(perfect matching) 중에서 안정적인 것을 찾는 것이 목표입니다.

### 불안정 쌍의 정의

매칭 M에서 남성 m과 여성 w가 서로 현재 파트너가 아닌데, m은 w를 자신의 현재 파트너보다 더 선호하고, 동시에 w도 m을 자신의 현재 파트너보다 더 선호하는 경우 (m, w)를 **불안정 쌍(blocking pair)**이라 합니다. 안정 매칭은 이러한 불안정 쌍이 하나도 없는 매칭을 의미합니다.

### 왜 안정 매칭이 필요한가

단순히 가능한 매칭을 찾는 것은 쉽습니다. 문제는 참가자들이 자신의 선호도와 다른 매칭에 배정될 경우, 합리적 행위자는 그 매칭을 깨고 더 나은 파트너와 새로운 쌍을 이루려 할 것이라는 데 있습니다. 안정 매칭은 이러한 "탈주 인센티브"를 제거하여 시스템 전체의 지속 가능성을 보장합니다.

실제 응용 사례:
- **NRMP(National Resident Matching Program)**: 미국에서 매년 4만여 명의 의대 졸업생과 병원 인턴십 자리를 매칭하는 데 사용
- **대학 입학 시스템**: 한국의 수시·정시 배치, 지역 연구를 통해 안정 매칭 원리가 활용
- **구직 플랫폼**: 구직자와 채용 기업 간의 최적 매칭
- **신장 기증 체인**: 비적합 신장 기증자-수여자 쌍을 교환 매칭

---

## Gale-Shapley 알고리즘의 동작 원리

알고리즘은 **지연 수락(Deferred Acceptance)** 전략을 사용합니다. 제안을 받는 쪽은 최선의 제안을 잠정적으로 보류하고, 더 좋은 제안이 오면 기존 제안을 버립니다.

**입력**: 남성 선호도 목록 `men_pref[m]`, 여성 선호도 목록 `women_pref[w]`

**알고리즘 단계**:
1. 모든 남성을 "자유(free)" 상태로 초기화
2. 자유 남성 m이 선호도 순서대로 아직 제안하지 않은 여성 w에게 제안
3. w가 현재 파트너가 없으면 → m과 임시 매칭
4. w에게 기존 파트너 m'가 있으면:
   - w가 m을 m'보다 선호하면 → m과 새로 매칭, m'는 자유
   - 그렇지 않으면 → m의 제안 거절, m은 계속 자유
5. 자유 남성이 없을 때까지 반복

**시간 복잡도**: O(n²) — 각 남성은 최대 n번 제안, 여성 선호도 비교는 사전 전처리 후 O(1)

---

## 구현 예제

### 예제 1: Python 구현

```python
def gale_shapley(men_pref, women_pref):
    """
    men_pref[i]: i번 남성의 선호도 순서 (여성 인덱스 리스트)
    women_pref[j]: j번 여성의 선호도 순서 (남성 인덱스 리스트)
    반환값: man_partner[i] = i번 남성의 매칭 여성 인덱스
    """
    n = len(men_pref)

    # O(1) 비교를 위해 여성 선호도를 랭킹 행렬로 변환
    women_rank = [[0] * n for _ in range(n)]
    for w in range(n):
        for rank, m in enumerate(women_pref[w]):
            women_rank[w][m] = rank  # 낮을수록 더 선호

    next_proposal = [0] * n      # 각 남성의 다음 제안 대상 인덱스
    woman_partner = [-1] * n     # 각 여성의 현재 파트너 (-1: 미매칭)
    man_partner = [-1] * n       # 각 남성의 현재 파트너

    free_men = list(range(n))

    while free_men:
        m = free_men.pop(0)
        w = men_pref[m][next_proposal[m]]
        next_proposal[m] += 1

        if woman_partner[w] == -1:
            # 여성이 자유 상태 → 임시 매칭
            woman_partner[w] = m
            man_partner[m] = w
        else:
            current = woman_partner[w]
            if women_rank[w][m] < women_rank[w][current]:
                # 새 제안자가 더 선호됨 → 교체
                woman_partner[w] = m
                man_partner[m] = w
                man_partner[current] = -1
                free_men.append(current)
            else:
                # 기존 파트너가 더 선호됨 → 제안 거절
                free_men.append(m)

    return man_partner


# === 사용 예시 ===
# 남성 0,1,2의 여성 선호 순서
men_pref = [
    [0, 1, 2],   # 남성 0: 여성 0 > 1 > 2
    [1, 0, 2],   # 남성 1: 여성 1 > 0 > 2
    [0, 2, 1],   # 남성 2: 여성 0 > 2 > 1
]
# 여성 0,1,2의 남성 선호 순서
women_pref = [
    [1, 0, 2],   # 여성 0: 남성 1 > 0 > 2
    [0, 1, 2],   # 여성 1: 남성 0 > 1 > 2
    [0, 1, 2],   # 여성 2: 남성 0 > 1 > 2
]

result = gale_shapley(men_pref, women_pref)
for m, w in enumerate(result):
    print(f"남성 {m} ↔ 여성 {w}")
# 출력:
# 남성 0 ↔ 여성 1
# 남성 1 ↔ 여성 0
# 남성 2 ↔ 여성 2
```

### 예제 2: Java 구현 및 안정성 검증

```java
import java.util.*;

public class GaleShapley {

    public static int[] match(int[][] menPref, int[][] womenPref) {
        int n = menPref.length;

        // 여성 선호도 랭킹 행렬 사전 구축
        int[][] womenRank = new int[n][n];
        for (int w = 0; w < n; w++) {
            for (int rank = 0; rank < n; rank++) {
                womenRank[w][womenPref[w][rank]] = rank;
            }
        }

        int[] nextProposal = new int[n];
        int[] womanPartner = new int[n];
        int[] manPartner = new int[n];
        Arrays.fill(womanPartner, -1);
        Arrays.fill(manPartner, -1);

        Deque<Integer> freeMen = new ArrayDeque<>();
        for (int i = 0; i < n; i++) freeMen.add(i);

        while (!freeMen.isEmpty()) {
            int m = freeMen.poll();
            int w = menPref[m][nextProposal[m]++];

            if (womanPartner[w] == -1) {
                womanPartner[w] = m;
                manPartner[m] = w;
            } else {
                int current = womanPartner[w];
                if (womenRank[w][m] < womenRank[w][current]) {
                    womanPartner[w] = m;
                    manPartner[m] = w;
                    manPartner[current] = -1;
                    freeMen.add(current);
                } else {
                    freeMen.add(m);
                }
            }
        }
        return manPartner;
    }

    /** 매칭이 안정적인지 검증 */
    public static boolean isStable(int[] manPartner, int[][] menPref, int[][] womenPref) {
        int n = menPref.length;
        int[][] menRank = new int[n][n];
        int[][] womenRank = new int[n][n];

        for (int i = 0; i < n; i++) {
            for (int r = 0; r < n; r++) {
                menRank[i][menPref[i][r]] = r;
                womenRank[i][womenPref[i][r]] = r;
            }
        }

        int[] womanPartner = new int[n];
        for (int m = 0; m < n; m++) {
            womanPartner[manPartner[m]] = m;
        }

        // 모든 쌍에 대해 불안정 쌍 검사
        for (int m = 0; m < n; m++) {
            for (int w = 0; w < n; w++) {
                if (manPartner[m] == w) continue;
                // m이 w를 현재 파트너보다 선호하는가?
                if (menRank[m][w] < menRank[m][manPartner[m]]) {
                    // w가 m을 현재 파트너보다 선호하는가?
                    if (womenRank[w][m] < womenRank[w][womanPartner[w]]) {
                        System.out.printf("불안정 쌍 발견: 남성 %d, 여성 %d%n", m, w);
                        return false;
                    }
                }
            }
        }
        return true;
    }

    public static void main(String[] args) {
        int[][] menPref   = {{0,1,2},{1,0,2},{0,2,1}};
        int[][] womenPref = {{1,0,2},{0,1,2},{0,1,2}};

        int[] matching = match(menPref, womenPref);
        System.out.println("매칭 결과: " + Arrays.toString(matching));
        System.out.println("안정성: " + isStable(matching, menPref, womenPref));
    }
}
```

---

## 알고리즘의 중요한 성질

### 제안자 최적성 (Proposer-Optimality)

남성이 제안하는 Gale-Shapley 알고리즘은 **남성 최적(man-optimal)** 안정 매칭을 생성합니다. 즉, 가능한 모든 안정 매칭 중에서 모든 남성이 동시에 가장 선호하는 파트너를 얻는 결과입니다. 반대로 이 매칭은 **여성 최악(woman-pessimal)** 안정 매칭이기도 합니다.

역할을 뒤집어 여성이 제안하면 여성 최적 매칭이 나옵니다.

### 유일성이 아님

안정 매칭은 유일하지 않을 수 있습니다. 동일한 입력에도 남성 제안 방식과 여성 제안 방식이 다른 결과를 줄 수 있으며, 그 사이에 또 다른 안정 매칭들이 존재할 수 있습니다.

### 전략적 진실성 (Strategyproofness)

남성이 제안하는 GS 알고리즘은 **남성에게 전략적으로 진실한(strategy-proof)** 매커니즘입니다. 즉 남성이 선호도를 거짓으로 보고해도 더 나은 결과를 얻을 수 없습니다. 그러나 여성은 전략적 허위 보고로 더 나은 파트너를 얻을 수 있습니다.

---

## 주의사항 및 응용 팁

**불완전 선호도 처리**: 실제로는 모든 상대를 선호도 리스트에 포함하지 않는 경우가 많습니다. 허용 불가 상대는 아예 리스트에서 제외하며, 이 경우 완전 매칭이 아닌 최대 안정 매칭을 구합니다.

**타이(동점) 처리**: 선호도에 동점이 있는 경우 "약한 안정 매칭"과 "강한 안정 매칭"으로 개념이 나뉘며, 문제가 NP-hard로 복잡해집니다.

**대규모 시스템**: 실제 NRMP에서는 수만 명의 참가자를 처리합니다. O(n²) 복잡도는 n=40,000일 때 약 1.6×10⁹ 연산이므로 최적화된 자료구조와 병렬 처리가 필요합니다.

**일반화 버전**:
- **College Admissions Problem**: 수용 인원이 1 이상인 경우 (병원은 여러 인턴 채용)
- **Roommate Problem**: 한 집합 내에서의 매칭 (안정 매칭이 존재하지 않을 수 있음)

---

## 참고 자료

- [Stable Matching Problem - Wikipedia](https://en.wikipedia.org/wiki/Stable_matching_problem)
- [Gale-Shapley Algorithm Explained - Built In](https://builtin.com/articles/gale-shapley-algorithm)
- [Cornell Networks Course: Gale-Shapley Theorems](https://blogs.cornell.edu/info2040/2021/09/22/gale-shapley-algorithm-and-some-related-theorems-to-stable-marriage-problem/)
- [Nobel Prize 2012 - Alvin Roth & Lloyd Shapley](https://www.nobelprize.org/prizes/economic-sciences/2012/press-release/)
