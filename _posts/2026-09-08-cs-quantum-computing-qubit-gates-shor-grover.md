---
layout: post
title: "양자 컴퓨팅 완전 정복: 큐비트·양자 게이트부터 Shor·Grover 알고리즘까지"
date: 2026-09-08
categories: [cs, computer-science]
tags: [quantum-computing, qubit, quantum-gates, shor-algorithm, grover-algorithm, superposition, entanglement, qiskit]
---

## 개념 설명

양자 컴퓨팅(Quantum Computing)은 양자역학의 원리 — **중첩(superposition)**, **얽힘(entanglement)**, **간섭(interference)** — 을 계산에 활용하는 패러다임이다. 고전 컴퓨터의 비트(bit)가 0 또는 1 중 하나의 상태만 가질 수 있는 것과 달리, 양자 컴퓨터의 **큐비트(qubit)**는 두 상태의 선형 중첩 상태에 놓일 수 있다.

### 큐비트와 중첩

수학적으로 큐비트의 상태는 다음과 같이 표현된다.

```
|ψ⟩ = α|0⟩ + β|1⟩
```

여기서 α, β는 **복소수 진폭(amplitude)**이며, |α|² + |β|² = 1을 만족한다. 측정 시 |0⟩이 나올 확률은 |α|², |1⟩이 나올 확률은 |β|²이다. 측정 전까지 큐비트는 두 상태가 동시에 존재하는 중첩 상태를 유지한다.

**Bloch 구(Bloch Sphere)**는 단일 큐비트의 상태를 3차원 단위 구의 한 점으로 시각화한 것이다.

```
|ψ⟩ = cos(θ/2)|0⟩ + e^(iφ)sin(θ/2)|1⟩
```

### 양자 얽힘 (Entanglement)

두 큐비트가 얽힌 상태의 대표 예시는 **벨 상태(Bell State)**다.

```
|Φ⁺⟩ = (1/√2)(|00⟩ + |11⟩)
```

이 상태에서는 한 큐비트를 측정하는 순간 다른 큐비트의 상태가 즉각적으로 결정된다 — 아무리 멀리 떨어져 있어도. 이 성질은 양자 통신과 양자 오류 수정의 핵심이다.

### 주요 양자 게이트

고전 컴퓨터의 논리 게이트처럼, 양자 컴퓨터는 **유니터리 행렬**로 표현되는 양자 게이트로 큐비트를 변환한다.

| 게이트 | 행렬 | 역할 |
|--------|------|------|
| Pauli-X | [[0,1],[1,0]] | 고전 NOT에 해당 |
| Hadamard (H) | (1/√2)[[1,1],[1,-1]] | 중첩 상태 생성 |
| CNOT | 제어 큐비트에 따라 타겟 반전 | 얽힘 생성 |
| Toffoli | 3-큐비트 제어 NOT | 범용 양자 게이트 |

**Hadamard 게이트**는 양자 알고리즘에서 가장 자주 쓰인다. |0⟩에 H를 적용하면 균등 중첩이 만들어진다.

```
H|0⟩ = (1/√2)(|0⟩ + |1⟩)  →  두 상태가 50/50 확률
H|1⟩ = (1/√2)(|0⟩ - |1⟩)
```

n개의 큐비트에 H를 적용하면 2ⁿ개의 상태 모두를 동시에 나타내는 중첩이 생성된다 — 이것이 양자 병렬성(quantum parallelism)의 출발점이다.

---

## 왜 필요한가

### 고전 컴퓨터의 한계

특정 문제 유형에서 고전 알고리즘은 지수적 시간 복잡도를 피할 수 없다.

- **소인수 분해**: RSA 암호화의 기반. n-비트 정수를 고전 알고리즘으로 분해하면 최선의 알고리즘(GNFS)도 O(exp(n^(1/3))) 시간이 필요하다.
- **비정렬 탐색**: N개의 원소에서 특정 원소를 찾으려면 최악 O(N) 비교가 필요하다.
- **양자 시뮬레이션**: N개 입자의 양자 시스템을 고전 컴퓨터로 시뮬레이션하면 상태 공간이 2ⁿ으로 폭발한다.

양자 컴퓨터는 이런 문제들에서 지수적 또는 이차 가속(quadratic speedup)을 제공한다.

---

## 실제 구현 예제

### 예제 1: Grover 알고리즘 — O(√N) 비정렬 탐색

Grover 알고리즘은 N개의 원소에서 특정 원소를 **O(√N)** 번의 오라클 호출로 찾는다. 고전 최선의 O(N)에 비해 이차 가속이다.

핵심 아이디어는 **진폭 증폭(amplitude amplification)**이다:
1. 전체 상태를 균등 중첩으로 초기화
2. 오라클이 정답 상태의 위상을 반전
3. 평균에 대한 반사(inversion about mean)로 정답 진폭을 증폭
4. 약 π√N/4번 반복 후 측정

```python
# Qiskit을 이용한 Grover 알고리즘 구현 (검색 대상: |11⟩ = 3)
from qiskit import QuantumCircuit, QuantumRegister, ClassicalRegister
from qiskit_aer import AerSimulator
from qiskit.visualization import plot_histogram

def grover_oracle_11(qc, qubits):
    """오라클: |11⟩ 상태에 -1 위상 적용 (CZ 게이트 활용)"""
    qc.cz(qubits[0], qubits[1])

def grover_diffuser(qc, qubits):
    """평균에 대한 반사 (Diffuser)"""
    qc.h(qubits)
    qc.x(qubits)
    qc.h(qubits[1])
    qc.cx(qubits[0], qubits[1])
    qc.h(qubits[1])
    qc.x(qubits)
    qc.h(qubits)

# 2-큐비트 회로 구성 (N=4, 정답: |11⟩)
q = QuantumRegister(2, 'q')
c = ClassicalRegister(2, 'c')
qc = QuantumCircuit(q, c)

# Step 1: 균등 중첩 초기화
qc.h(q)
qc.barrier()

# Step 2: Grover 반복 (N=4일 때 1회가 최적)
grover_oracle_11(qc, q)
qc.barrier()
grover_diffuser(qc, q)
qc.barrier()

# Step 3: 측정
qc.measure(q, c)

# 시뮬레이션
simulator = AerSimulator()
job = simulator.run(qc, shots=1024)
result = job.result()
counts = result.get_counts()
print("측정 결과:", counts)
# 출력 예: {'11': 1024} — 100% 확률로 |11⟩ 검출
```

### 예제 2: Shor 알고리즘 — 소인수 분해의 지수 가속

Shor 알고리즘은 n-비트 정수 N을 **O(n³)** 시간에 분해한다. 핵심은 소인수 분해를 **주기 탐색(period finding)** 문제로 환원하고, 이를 양자 푸리에 변환(QFT)으로 효율적으로 해결하는 것이다.

```python
# Shor 알고리즘의 고전 부분 + 주기 찾기 시뮬레이션
# (완전한 양자 구현은 수십 큐비트 필요, 여기서는 핵심 로직 시연)
import math
import random
from fractions import Fraction

def quantum_period_finding_classical_sim(a, N):
    """
    f(x) = a^x mod N 의 주기 r을 찾는다.
    실제로는 양자 위상 추정(QPE) + QFT를 사용하지만
    여기서는 고전 시뮬레이션으로 대체.
    """
    x = 1
    for r in range(1, N):
        x = (x * a) % N
        if x == 1:
            return r
    return None

def shor_factor(N):
    """
    Shor 알고리즘으로 N의 비자명 인수 찾기
    N=15를 예시로 사용
    """
    if N % 2 == 0:
        return 2, N // 2

    for _ in range(50):  # 최대 시도 횟수
        a = random.randint(2, N - 1)
        g = math.gcd(a, N)

        # 운 좋게 인수 발견
        if g > 1:
            return g, N // g

        # 양자 주기 탐색 (시뮬레이션)
        r = quantum_period_finding_classical_sim(a, N)
        if r is None or r % 2 != 0:
            continue

        # r이 짝수일 때 인수 후보 계산
        factor1 = math.gcd(a**(r//2) - 1, N)
        factor2 = math.gcd(a**(r//2) + 1, N)

        for f in [factor1, factor2]:
            if 1 < f < N:
                return f, N // f

    return None

# N=15 분해 시연
N = 15
result = shor_factor(N)
if result:
    p, q = result
    print(f"{N} = {p} × {q}")  # 출력: 15 = 3 × 5 또는 15 = 5 × 3
    print(f"검증: {p} × {q} = {p * q}")

# 양자 푸리에 변환 (QFT) 핵심 구현
from qiskit import QuantumCircuit
import numpy as np

def qft_circuit(n):
    """n-큐비트 QFT 회로"""
    qc = QuantumCircuit(n)
    for i in range(n):
        qc.h(i)
        for j in range(i + 1, n):
            phase = 2 * np.pi / (2 ** (j - i + 1))
            qc.cp(phase, j, i)
    # 비트 역순 정렬
    for i in range(n // 2):
        qc.swap(i, n - i - 1)
    return qc

qft = qft_circuit(4)
print("\nQFT 회로:")
print(qft.draw(output='text'))
```

---

## 주의사항 및 팁

### 1. 노이즈와 오류 (NISQ 시대의 현실)

현재 양자 컴퓨터는 **NISQ(Noisy Intermediate-Scale Quantum)** 단계다. 큐비트 수는 수백~수천 개로 늘었지만, **결어긴(decoherence)**과 게이트 오류(gate error)가 크다. 실용적인 Shor 알고리즘 실행에는 수백만 개의 논리 큐비트(오류 수정 포함)가 필요하다고 추정된다.

### 2. 양자 오류 수정 (QEC)

**표면 코드(Surface Code)**가 현재 가장 유망한 QEC 방식으로, 약 1000개의 물리 큐비트로 1개의 논리 큐비트를 구현한다. 오류 임계값(fault-tolerance threshold)은 약 1%다.

### 3. 개발 환경

| 도구 | 제공사 | 특징 |
|------|--------|------|
| Qiskit | IBM | 오픈소스, 실제 IBM 양자 하드웨어 접근 가능 |
| Cirq | Google | TensorFlow Quantum 연동 |
| PennyLane | Xanadu | ML과 양자 통합 |
| Q# | Microsoft | Azure Quantum 플랫폼 |

### 4. 양자 우위의 실제 범위

양자 컴퓨터가 모든 문제를 빠르게 해결하는 것은 아니다. 명확한 양자 가속이 증명된 문제는 제한적이다.

- **지수 가속**: 소인수 분해 (Shor), 이산 로그 (Shor 변형)
- **이차 가속**: 비정렬 탐색 (Grover), 최적화 일부
- **고전과 동일**: NP-완전 문제 일반 (양자 컴퓨터도 NP를 P로 만들지 못함)

### 5. 학습 경로

양자 컴퓨팅을 시작하려면 먼저 선형대수(행렬 곱, 고유값, 유니터리 행렬)와 복소수를 탄탄히 익혀야 한다. 이후 Nielsen & Chuang의 *"Quantum Computation and Quantum Information"*이 표준 교재다. 실습은 Qiskit의 무료 클라우드 시뮬레이터로 시작할 수 있다.

## 참고 자료
- [IBM Quantum Learning - Grover's Algorithm](https://quantum.cloud.ibm.com/learning/en/modules/computer-science/grovers)
- [IBM Quantum Documentation - Grover's Algorithm Tutorial](https://quantum.cloud.ibm.com/docs/en/tutorials/grovers-algorithm)
- [CWI - Main Quantum Algorithms: Shor and Grover (Ronald de Wolf)](https://homepages.cwi.nl/~rdewolf/warsaw2short.pdf)
- [arXiv - A Gateway to Quantum Computing for Industrial Engineering](https://arxiv.org/abs/2510.20620)
