---
layout: post
title: "모의 담금질과 유전 알고리즘: NP-난해 문제를 확률로 정복하는 메타휴리스틱"
date: 2026-08-11
categories: [cs, computer-science]
tags: [optimization, simulated-annealing, genetic-algorithm, metaheuristic, tsp, combinatorial-optimization]
---

## 개념 설명

많은 실세계 최적화 문제는 **NP-난해(NP-Hard)**입니다. 외판원 문제(TSP), 배낭 문제(Knapsack), 작업 스케줄링, 레이아웃 최적화 등은 정확한 해를 다항 시간에 구하는 알고리즘이 알려져 있지 않습니다. 이런 문제를 실용적으로 해결하기 위해 탄생한 것이 **메타휴리스틱(Metaheuristic)** 알고리즘입니다.

이 글에서는 두 가지 대표적인 메타휴리스틱 기법인 **모의 담금질(Simulated Annealing, SA)**과 **유전 알고리즘(Genetic Algorithm, GA)**을 심층적으로 다룹니다.

---

### 모의 담금질 (Simulated Annealing)

1983년 Kirkpatrick, Gelatt, Vecchi가 발표한 SA는 야금학의 **담금질(Annealing)** 과정에서 영감을 얻었습니다. 금속을 고온에서 가열한 뒤 서서히 냉각하면 원자들이 최소 에너지 상태(결정 구조)로 배열됩니다. 이때 충분히 천천히 냉각하면 전역 최소(global minimum)에 도달하지만, 너무 빨리 냉각하면 국소 최소(local minimum)에 갇혀 버립니다.

SA의 핵심 아이디어:
1. 현재 해(solution)에서 이웃 해(neighbor)를 생성한다.
2. 이웃 해가 현재 해보다 좋으면 **반드시 이동**한다.
3. 이웃 해가 나빠도 **확률적으로 이동**한다: P(accept) = exp(-Δ/T)
4. 온도 T를 냉각 스케줄(cooling schedule)에 따라 서서히 낮춘다.

온도가 높을 때는 나쁜 해도 자주 받아들여 해 공간(search space)을 광범위하게 탐색하고, 온도가 낮아질수록 이미 찾은 좋은 영역 주변에서만 탐색합니다. 이 메커니즘이 국소 최솟값 함정(local minimum trap)을 탈출하게 해줍니다.

**수락 확률 함수 (Metropolis-Hastings)**:
```
P(accept) = exp(-ΔE / T)
```
- ΔE = new_cost - current_cost (양수이면 해가 나빠진 것)
- T = 현재 온도
- ΔE = 0이면 P = 1.0 (같으면 항상 수락)
- ΔE > 0이면 P < 1.0 (나빠도 일정 확률로 수락)

---

### 유전 알고리즘 (Genetic Algorithm)

1975년 John Holland가 제안한 GA는 **자연 선택과 유전학**의 원리를 모방합니다. 여러 해의 집단(population)을 유지하며 세대(generation)를 거듭하면서 더 좋은 해를 진화시킵니다.

GA의 기본 연산자:
- **선택(Selection)**: 적합도(fitness)가 높은 개체를 부모로 선택 (룰렛 휠, 토너먼트 선택 등)
- **교차(Crossover)**: 두 부모의 유전자를 결합해 자식 생성 (단일점, 다점, 균일 교차)
- **변이(Mutation)**: 낮은 확률로 유전자 일부를 무작위 변경 (다양성 유지)
- **대체(Replacement)**: 자식 세대가 부모 세대를 대체

GA vs SA의 핵심 차이:
| | SA | GA |
|---|---|---|
| 해의 수 | 하나 (단일 개체) | 여러 개 (집단) |
| 탐색 방식 | 이웃 탐색 (sequential) | 교차/변이 (parallel) |
| 다양성 | 온도로 제어 | 집단 크기와 변이율로 제어 |
| 조기 수렴 | 국소 최솟값 탈출 가능 | 조기 수렴(premature convergence) 위험 |

---

## 왜 필요한가

### 실세계 최적화 문제의 규모

현실적인 최적화 문제는 해 공간이 천문학적입니다:
- **물류 최적화**: 100개 도시 TSP의 가능한 경로 수 ≈ 100!/2 ≈ 4.7 × 10^157
- **칩 레이아웃(VLSI)**: 수백만 개 트랜지스터의 배치 문제
- **신경망 하이퍼파라미터 튜닝**: 학습률, 배치 크기, 레이어 구조의 조합

완전 탐색(Brute Force)이나 정확한 알고리즘으로는 수십억 년이 걸리는 문제를 메타휴리스틱으로 수 분 내에 준최적해(near-optimal solution)를 찾을 수 있습니다.

### 그래디언트가 없는 최적화

딥러닝처럼 기울기(gradient)를 계산할 수 없거나, 목적 함수가 불연속적이거나 비미분가능한 경우(예: 이산 조합 문제, 블랙박스 함수)에는 경사 하강법이 적용되지 않습니다. SA와 GA는 이런 상황에서 강력한 대안입니다.

---

## 실제 구현 예제

### 예제 1: 모의 담금질로 TSP(외판원 문제) 풀기

```python
import math
import random
from typing import Callable

def simulated_annealing_tsp(
    cities: list[tuple[float, float]],
    initial_temp: float = 10000.0,
    cooling_rate: float = 0.9995,
    min_temp: float = 1e-8,
    max_iter_per_temp: int = 100,
    seed: int = 42,
) -> tuple[list[int], float]:
    """
    모의 담금질로 TSP의 근사 최적해를 구합니다.
    
    Args:
        cities: [(x, y), ...] 형태의 도시 좌표 리스트
        initial_temp: 초기 온도
        cooling_rate: 냉각 비율 (0 < r < 1, 클수록 천천히 냉각)
        min_temp: 종료 온도
        max_iter_per_temp: 온도당 최대 반복 횟수
        seed: 난수 시드
    
    Returns:
        (best_tour, best_cost): 최적 경로와 비용
    
    시간 복잡도: O(iterations × n) where n = 도시 수
    """
    random.seed(seed)
    n = len(cities)

    def tour_distance(tour: list[int]) -> float:
        total = 0.0
        for i in range(n):
            a, b = tour[i], tour[(i + 1) % n]
            dx = cities[a][0] - cities[b][0]
            dy = cities[a][1] - cities[b][1]
            total += math.sqrt(dx * dx + dy * dy)
        return total

    def two_opt_swap(tour: list[int], i: int, j: int) -> list[int]:
        """i+1부터 j까지 구간을 역전시키는 2-opt 이웃 생성"""
        new_tour = tour[:i + 1] + tour[i + 1:j + 1][::-1] + tour[j + 1:]
        return new_tour

    # 초기 해: 0, 1, 2, ..., n-1 순서의 탐욕 초기화
    current_tour = list(range(n))
    random.shuffle(current_tour)
    current_cost = tour_distance(current_tour)

    best_tour = current_tour[:]
    best_cost = current_cost

    temperature = initial_temp
    iteration = 0

    while temperature > min_temp:
        for _ in range(max_iter_per_temp):
            # 2-opt 이웃 생성: 임의의 두 위치 i < j를 선택해 구간 역전
            i = random.randint(0, n - 2)
            j = random.randint(i + 1, n - 1)
            neighbor = two_opt_swap(current_tour, i, j)
            neighbor_cost = tour_distance(neighbor)

            delta = neighbor_cost - current_cost

            # Metropolis 수락 기준: 좋아지면 무조건, 나빠지면 확률적으로 수락
            if delta < 0 or random.random() < math.exp(-delta / temperature):
                current_tour = neighbor
                current_cost = neighbor_cost

                if current_cost < best_cost:
                    best_tour = current_tour[:]
                    best_cost = current_cost

        # 냉각
        temperature *= cooling_rate
        iteration += 1

    return best_tour, best_cost


# ===================== 테스트 =====================
# 10개 도시 좌표 (임의 설정)
cities = [
    (0.0, 0.0), (1.0, 5.0), (3.0, 2.0), (5.0, 8.0), (7.0, 1.0),
    (2.0, 9.0), (8.0, 5.0), (4.0, 4.0), (6.0, 3.0), (9.0, 7.0),
]

tour, cost = simulated_annealing_tsp(cities, seed=42)
print(f"SA 최적 경로: {tour}")
print(f"SA 경로 비용: {cost:.4f}")

# 무작위 경로와 비교
random.seed(42)
random_tour = list(range(len(cities)))
random.shuffle(random_tour)

def tour_distance(tour):
    n = len(tour)
    total = 0.0
    for i in range(n):
        a, b = tour[i], tour[(i+1) % n]
        dx = cities[a][0] - cities[b][0]
        dy = cities[a][1] - cities[b][1]
        total += math.sqrt(dx*dx + dy*dy)
    return total

random_cost = tour_distance(random_tour)
print(f"무작위 경로 비용: {random_cost:.4f}")
print(f"개선율: {(random_cost - cost) / random_cost * 100:.1f}%")
```

### 예제 2: 유전 알고리즘으로 연속 최적화 문제 풀기

```python
import random
import math

class GeneticAlgorithm:
    """
    실수 인코딩 유전 알고리즘.
    다변수 연속 최적화 문제에 적용합니다.
    
    목적: minimize f(x) over x in [lower, upper]^dimension
    """

    def __init__(
        self,
        fitness_fn,
        dimension: int,
        lower: float,
        upper: float,
        pop_size: int = 50,
        max_generations: int = 500,
        crossover_rate: float = 0.8,
        mutation_rate: float = 0.1,
        tournament_k: int = 3,
        seed: int = 42,
    ):
        self.fitness_fn = fitness_fn      # 최소화할 목적 함수
        self.dim = dimension
        self.lower = lower
        self.upper = upper
        self.pop_size = pop_size
        self.max_gen = max_generations
        self.cx_rate = crossover_rate
        self.mut_rate = mutation_rate
        self.k = tournament_k
        random.seed(seed)

    def _random_individual(self) -> list[float]:
        return [random.uniform(self.lower, self.upper) for _ in range(self.dim)]

    def _tournament_selection(self, population, fitnesses) -> list[float]:
        """토너먼트 선택: k개 개체 중 가장 적합도가 낮은 것 선택 (최소화 문제)"""
        candidates = random.sample(range(len(population)), self.k)
        winner = min(candidates, key=lambda i: fitnesses[i])
        return population[winner][:]

    def _arithmetic_crossover(
        self, parent1: list[float], parent2: list[float]
    ) -> tuple[list[float], list[float]]:
        """산술 교차: α를 무작위로 선택해 볼록 결합"""
        alpha = random.random()
        child1 = [alpha * p1 + (1 - alpha) * p2 for p1, p2 in zip(parent1, parent2)]
        child2 = [(1 - alpha) * p1 + alpha * p2 for p1, p2 in zip(parent1, parent2)]
        return child1, child2

    def _gaussian_mutation(self, individual: list[float]) -> list[float]:
        """가우시안 변이: 각 유전자에 독립적으로 가우시안 노이즈 추가"""
        sigma = (self.upper - self.lower) * 0.1  # 탐색 범위의 10% 표준편차
        mutated = []
        for gene in individual:
            if random.random() < self.mut_rate:
                new_gene = gene + random.gauss(0, sigma)
                # 경계 클리핑
                new_gene = max(self.lower, min(self.upper, new_gene))
                mutated.append(new_gene)
            else:
                mutated.append(gene)
        return mutated

    def evolve(self) -> tuple[list[float], float, list[float]]:
        """
        GA를 실행합니다.
        
        Returns:
            (best_individual, best_fitness, history):
              best_individual: 최적해 벡터
              best_fitness: 최적 목적 함수 값
              history: 세대별 최적 적합도 기록
        """
        # 초기 집단 생성
        population = [self._random_individual() for _ in range(self.pop_size)]
        fitnesses = [self.fitness_fn(ind) for ind in population]

        best_idx = min(range(self.pop_size), key=lambda i: fitnesses[i])
        best_individual = population[best_idx][:]
        best_fitness = fitnesses[best_idx]
        history = [best_fitness]

        for gen in range(self.max_gen):
            new_population = []

            # 엘리트주의(Elitism): 상위 1개는 그대로 유지
            elite_idx = min(range(self.pop_size), key=lambda i: fitnesses[i])
            new_population.append(population[elite_idx][:])

            # 자식 개체 생성
            while len(new_population) < self.pop_size:
                parent1 = self._tournament_selection(population, fitnesses)
                parent2 = self._tournament_selection(population, fitnesses)

                if random.random() < self.cx_rate:
                    child1, child2 = self._arithmetic_crossover(parent1, parent2)
                else:
                    child1, child2 = parent1[:], parent2[:]

                child1 = self._gaussian_mutation(child1)
                child2 = self._gaussian_mutation(child2)
                new_population.extend([child1, child2])

            population = new_population[:self.pop_size]
            fitnesses = [self.fitness_fn(ind) for ind in population]

            # 현 세대 최적 갱신
            gen_best_idx = min(range(self.pop_size), key=lambda i: fitnesses[i])
            if fitnesses[gen_best_idx] < best_fitness:
                best_fitness = fitnesses[gen_best_idx]
                best_individual = population[gen_best_idx][:]

            history.append(best_fitness)

        return best_individual, best_fitness, history


# ===================== 테스트: Rastrigin 함수 최적화 =====================
# Rastrigin 함수는 수많은 국소 최솟값을 가져 최적화 벤치마크로 자주 사용됩니다.
# f(x) = A*n + Σ [x_i² - A*cos(2π*x_i)]  (전역 최솟값: f(0,...,0) = 0)

def rastrigin(x: list[float], A: float = 10.0) -> float:
    n = len(x)
    return A * n + sum(xi**2 - A * math.cos(2 * math.pi * xi) for xi in x)

ga = GeneticAlgorithm(
    fitness_fn=rastrigin,
    dimension=5,           # 5차원
    lower=-5.12,
    upper=5.12,
    pop_size=100,
    max_generations=300,
    seed=42,
)

best_x, best_f, history = ga.evolve()

print("GA 최적화 결과 (Rastrigin 5D)")
print(f"  최적해 x: [{', '.join(f'{v:.4f}' for v in best_x)}]")
print(f"  최적값 f(x): {best_f:.6f}  (이론적 전역 최솟값: 0.0)")
print(f"  세대 1 최적: {history[0]:.4f}")
print(f"  세대 50 최적: {history[50]:.4f}")
print(f"  세대 300 최적: {history[-1]:.4f}")
```

---

## SA vs GA — 언제 무엇을 선택할까

| 기준 | SA 적합 | GA 적합 |
|---|---|---|
| 해의 구조 | 단순 이웃 연산이 잘 정의된 경우 | 해가 벡터·순열·트리 등 구조가 있는 경우 |
| 집단 기반 | 단일 해로 충분한 경우 | 다양한 해 풀(pool)이 탐색에 유리한 경우 |
| 구현 복잡도 | 단순 (핵심 파라미터: 초기 온도, 냉각 속도) | 복잡 (교차·선택·변이 연산자 설계 필요) |
| 수렴 속도 | 빠른 편 (단일 체인) | 느리지만 다양성 유지 능력 우수 |
| 병렬화 | 어렵 (상태 의존적) | 쉬움 (집단 내 개체 독립 평가) |
| 대표 적용처 | VLSI 배치, 일정 계획 | 자동화 설계, 신경 구조 탐색(NAS) |

---

## 주의사항 및 팁

### 1. 냉각 스케줄 설계 (SA)
냉각이 너무 빠르면 국소 최솟값에 갇히고, 너무 느리면 실행 시간이 기하급수적으로 늘어납니다. 실용적인 공식은:
- **기하 냉각(Geometric Cooling)**: T(t) = T₀ × αᵗ  (0.9 < α < 0.999 권장)
- **로그 냉각(Logarithmic Cooling)**: T(t) = T₀ / ln(1 + t) (이론적 보장 있음, 매우 느림)
- 초기 수락률을 80~90%로 시작해, 종료 수락률을 1% 미만으로 맞추도록 T₀와 T_min을 설정하세요.

### 2. 재시작 전략 (SA + GA 공통)
단일 실행으로는 전역 최솟값에 도달하기 어렵습니다. 서로 다른 초기 해와 시드로 여러 번 실행 후 최선을 선택하는 **다중 재시작(Multi-start)** 전략이 효과적입니다.

### 3. 집단 다양성 유지 (GA)
너무 이른 수렴(조기 수렴, Premature Convergence)은 GA의 고질적 문제입니다. 대책:
- **변이율 적응**: 다양성이 낮아지면 변이율을 높임
- **니싱(Niching)**: 같은 영역의 개체끼리 경쟁을 제한
- **섬 모델(Island Model)**: 여러 독립 집단이 가끔씩 개체를 교환

### 4. 제약 조건 처리
실세계 문제에는 제약 조건이 많습니다. 처리 방법:
- **패널티 함수법**: 제약 위반 시 목적 함수에 큰 패널티를 추가 (구현 단순, 파라미터 튜닝 필요)
- **수리 연산자(Repair Operator)**: 이웃 생성 후 제약을 만족하도록 해를 수정
- **해 인코딩 자체로 제약 반영**: TSP의 순열 인코딩처럼 항상 유효한 해를 생성하는 표현 방식 사용

### 5. 하이브리드 접근법
SA와 GA를 조합한 **유전 담금질(Genetic Simulated Annealing)**도 실용적입니다. GA로 좋은 영역을 빠르게 찾은 뒤, SA로 해당 영역을 정밀하게 탐색합니다. Google의 AlphaChip(반도체 배치) 같은 최신 도구도 이런 하이브리드 방식을 채택합니다.

---

## 참고 자료

- [Simulated Annealing — Wikipedia](https://en.wikipedia.org/wiki/Simulated_annealing)
- [Genetic Algorithm — Wikipedia](https://en.wikipedia.org/wiki/Genetic_algorithm)
- [Kirkpatrick, S. et al. (1983). "Optimization by Simulated Annealing" — Science](https://www.science.org/doi/10.1126/science.220.4598.671)
- [Holland, J. (1975). "Adaptation in Natural and Artificial Systems" — MIT Press](https://mitpress.mit.edu/9780262581110/adaptation-in-natural-and-artificial-systems/)
