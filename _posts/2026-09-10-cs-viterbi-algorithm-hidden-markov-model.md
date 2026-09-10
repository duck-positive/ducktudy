---
layout: post
title: "비터비 알고리즘과 은닉 마르코프 모델 완전 정복: 시퀀스 속 숨겨진 패턴을 찾는 동적 프로그래밍"
date: 2026-09-10
categories: [cs, computer-science]
tags: [viterbi, hmm, hidden-markov-model, dynamic-programming, nlp, speech-recognition, sequence-analysis]
---

## 개념 설명

**은닉 마르코프 모델(Hidden Markov Model, HMM)**은 시스템이 마르코프 과정을 따르지만, 그 내부 상태가 직접 관측되지 않고 오직 간접적인 **관측값(observation)**을 통해서만 파악되는 확률 모델이다. 

쉽게 말하면 이렇다. 방 안에 있는 사람이 날씨(맑음/흐림/비)를 직접 볼 수 없지만, 밖에서 돌아온 친구가 가져온 우산 여부, 습도 느낌, 친구의 기분 등 간접적인 신호로 날씨를 추측하는 상황이다. 여기서 날씨는 **은닉 상태(hidden state)**, 우산 여부 등이 **관측 값(observation)**이 된다.

HMM은 세 가지 핵심 파라미터로 정의된다:

- **초기 상태 확률 π**: 각 상태에서 시작할 확률 벡터
- **전이 확률 행렬 A**: 한 상태에서 다른 상태로 이동할 확률 (`A[i][j]` = 상태 i에서 j로 전이할 확률)
- **방출 확률 행렬 B**: 특정 상태에서 관측값이 나타날 확률 (`B[i][k]` = 상태 i에서 관측값 k가 나타날 확률)

**비터비 알고리즘(Viterbi Algorithm)**은 이 HMM에서 주어진 관측 시퀀스에 대해 가장 가능성 높은 **은닉 상태 시퀀스**를 찾는 동적 프로그래밍 알고리즘이다. 1967년 Andrew Viterbi가 오류 정정 코드를 위해 개발했으며, 이후 음성 인식, NLP, 생물정보학 등 수많은 분야에서 핵심 알고리즘으로 자리 잡았다.

---

## 왜 필요한가

### HMM이 해결하는 세 가지 문제

HMM에는 세 가지 고전적 문제가 있다:

1. **평가 문제 (Evaluation)**: 주어진 관측 시퀀스 O가 모델 λ에서 생성될 확률 P(O|λ)는? → Forward 알고리즘
2. **디코딩 문제 (Decoding)**: 관측 시퀀스 O를 가장 잘 설명하는 은닉 상태 시퀀스는? → **비터비 알고리즘**
3. **학습 문제 (Learning)**: 관측 시퀀스가 주어졌을 때 모델 파라미터 λ=(A, B, π)를 어떻게 추정하는가? → Baum-Welch 알고리즘

비터비 알고리즘은 두 번째 문제(디코딩)를 효율적으로 해결한다.

### 실제 응용 사례

- **음성 인식**: 음향 신호(관측)에서 음소 시퀀스(은닉 상태)를 복원
- **품사 태깅(POS Tagging)**: 단어 시퀀스(관측)에서 품사 시퀀스(은닉 상태)를 추론
- **생물정보학**: DNA/RNA 서열에서 유전자 영역 탐지
- **금융**: 시장 상태(강세장/약세장) 추론
- **통신**: 노이즈가 있는 채널에서 원래 신호 복원

---

## 비터비 알고리즘 동작 원리

비터비 알고리즘은 **두 개의 DP 테이블**을 사용한다:

- `δ[t][i]`: 시점 t에서 상태 i로 끝나는 가장 높은 확률 경로의 확률
- `ψ[t][i]`: 시점 t에서 상태 i로 전이했을 때의 이전 상태 (역추적용)

**초기화**:
```
δ[1][i] = π[i] * B[i][O₁]
ψ[1][i] = 0
```

**재귀 (t = 2, ..., T)**:
```
δ[t][j] = max_i [ δ[t-1][i] * A[i][j] ] * B[j][Ot]
ψ[t][j] = argmax_i [ δ[t-1][i] * A[i][j] ]
```

**역추적**: `δ[T][i]`가 최대인 상태부터 `ψ` 테이블을 따라 역방향으로 최적 경로 복원

---

## 실제 구현 예제

### 예제 1: 날씨 추론 — Python 구현

```python
import numpy as np

def viterbi(obs, states, start_p, trans_p, emit_p):
    """
    obs: 관측 시퀀스 (인덱스 배열)
    states: 은닉 상태 목록
    start_p: 초기 확률 {state: prob}
    trans_p: 전이 확률 {from: {to: prob}}
    emit_p: 방출 확률 {state: {obs: prob}}
    """
    T = len(obs)
    N = len(states)

    # DP 테이블 초기화
    delta = np.zeros((T, N))
    psi = np.zeros((T, N), dtype=int)

    # 초기화
    for i, s in enumerate(states):
        delta[0][i] = start_p[s] * emit_p[s][obs[0]]
        psi[0][i] = 0

    # 재귀
    for t in range(1, T):
        for j, s_j in enumerate(states):
            prob_list = [
                delta[t-1][i] * trans_p[states[i]][s_j]
                for i in range(N)
            ]
            best_prev = np.argmax(prob_list)
            delta[t][j] = prob_list[best_prev] * emit_p[s_j][obs[t]]
            psi[t][j] = best_prev

    # 역추적
    path = [0] * T
    path[T-1] = np.argmax(delta[T-1])
    for t in range(T-2, -1, -1):
        path[t] = psi[t+1][path[t+1]]

    return [states[p] for p in path], delta[T-1][path[T-1]]


# 날씨 예제: 상태=[맑음, 비], 관측=[우산없음, 우산있음]
states = ['맑음', '비']
observations = ['우산없음', '우산있음', '우산있음', '우산없음']

start_prob = {'맑음': 0.6, '비': 0.4}

trans_prob = {
    '맑음': {'맑음': 0.7, '비': 0.3},
    '비':   {'맑음': 0.4, '비': 0.6},
}

emit_prob = {
    '맑음': {'우산없음': 0.8, '우산있음': 0.2},
    '비':   {'우산없음': 0.1, '우산있음': 0.9},
}

path, prob = viterbi(observations, states, start_prob, trans_prob, emit_prob)
print(f"가장 가능성 높은 날씨 시퀀스: {path}")
print(f"경로 확률: {prob:.6f}")
# 출력: 가장 가능성 높은 날씨 시퀀스: ['맑음', '비', '비', '맑음']
```

### 예제 2: 품사 태깅(POS Tagging) — 로그 공간 수치 안정화 구현

실제 문제에서는 확률값이 0에 가까워지는 **언더플로우(underflow)** 문제가 발생한다. 로그 공간으로 변환하면 곱셈이 덧셈으로 바뀌어 수치적으로 안정적이다.

```python
import numpy as np
from typing import List, Dict, Tuple

def viterbi_log(
    obs_seq: List[str],
    states: List[str],
    log_start: Dict[str, float],
    log_trans: Dict[str, Dict[str, float]],
    log_emit: Dict[str, Dict[str, float]],
    unk_log_prob: float = -20.0
) -> Tuple[List[str], float]:
    """로그 공간에서 동작하는 비터비 (수치 안정화)"""
    T = len(obs_seq)
    N = len(states)
    state_idx = {s: i for i, s in enumerate(states)}

    # 초기화
    log_delta = np.full((T, N), -np.inf)
    psi = np.zeros((T, N), dtype=int)

    for i, s in enumerate(states):
        obs = obs_seq[0]
        log_e = log_emit[s].get(obs, unk_log_prob)
        log_delta[0][i] = log_start.get(s, -np.inf) + log_e

    # 재귀
    for t in range(1, T):
        obs = obs_seq[t]
        for j, s_j in enumerate(states):
            log_e = log_emit[s_j].get(obs, unk_log_prob)
            candidates = np.array([
                log_delta[t-1][i] + log_trans[states[i]].get(s_j, -np.inf)
                for i in range(N)
            ])
            best_prev = np.argmax(candidates)
            log_delta[t][j] = candidates[best_prev] + log_e
            psi[t][j] = best_prev

    # 역추적
    path = [0] * T
    path[T-1] = int(np.argmax(log_delta[T-1]))
    for t in range(T-2, -1, -1):
        path[t] = psi[t+1][path[t+1]]

    best_log_prob = log_delta[T-1][path[T-1]]
    return [states[p] for p in path], best_log_prob


# 간단한 품사 태깅 예시 (실제에서는 코퍼스로 학습)
import math
log = math.log

states = ['명사', '동사', '형용사']
observations = ['사과', '먹다', '맛있다']

log_start = {'명사': log(0.6), '동사': log(0.2), '형용사': log(0.2)}

log_trans = {
    '명사':  {'명사': log(0.2), '동사': log(0.6), '형용사': log(0.2)},
    '동사':  {'명사': log(0.1), '동사': log(0.3), '형용사': log(0.6)},
    '형용사':{'명사': log(0.5), '동사': log(0.3), '형용사': log(0.2)},
}

log_emit = {
    '명사':  {'사과': log(0.7), '먹다': log(0.05), '맛있다': log(0.05)},
    '동사':  {'사과': log(0.05), '먹다': log(0.8),  '맛있다': log(0.05)},
    '형용사':{'사과': log(0.05), '먹다': log(0.05), '맛있다': log(0.8)},
}

path, log_prob = viterbi_log(observations, states, log_start, log_trans, log_emit)
print(f"태깅 결과: {list(zip(observations, path))}")
print(f"로그 확률: {log_prob:.4f}")
# 출력: 태깅 결과: [('사과', '명사'), ('먹다', '동사'), ('맛있다', '형용사')]
```

---

## 시간 복잡도 분석

- **시간 복잡도**: O(T × N²)  
  - T: 관측 시퀀스 길이, N: 은닉 상태 수  
  - 각 시점 t마다 N개 상태에 대해 N개 이전 상태를 모두 검토

- **공간 복잡도**: O(T × N) — DP 테이블과 역추적 테이블

단순 브루트 포스가 O(N^T)인 것에 비해 DP를 통해 지수 시간을 다항 시간으로 줄인 것이 핵심이다.

---

## 주의사항과 실전 팁

### 1. 언더플로우 문제
확률값을 직접 곱하면 부동소수점 언더플로우가 발생한다. 반드시 **로그 공간**에서 계산하라. 곱셈 → 덧셈, 나눗셈 → 뺄셈으로 변환된다.

### 2. 미등록 단어(Unknown Word) 처리
학습 데이터에 없는 관측값이 나타나면 방출 확률이 0이 되어 모든 경로 확률이 0이 된다. **스무딩(Smoothing)** 기법(Laplace smoothing, Kneser-Ney 등)이나 UNK 토큰 처리가 필수다.

### 3. Beam Search로 속도 향상
상태 공간이 매우 클 때 (N이 크고 T가 길 때) 모든 상태를 탐색하는 것은 비효율적이다. **빔 서치(Beam Search)**를 사용해 매 시점마다 상위 k개 후보만 유지하면 속도를 크게 높일 수 있다. 다만 최적해를 놓칠 수 있다.

### 4. HMM과 CRF의 비교
HMM은 생성 모델(generative model)로 P(O, S)를 모델링하는 반면, **조건부 무작위장(CRF, Conditional Random Field)**는 판별 모델(discriminative model)로 P(S|O)를 직접 모델링한다. NLP에서는 CRF가 HMM보다 더 높은 정확도를 보이는 경우가 많으며, 비터비 알고리즘은 선형 체인 CRF의 디코딩에도 동일하게 적용된다.

### 5. 현대 딥러닝과의 관계
BERT, GPT 등 Transformer 기반 모델이 등장했음에도 비터비 알고리즘은 여전히 중요하다. BiLSTM-CRF 모델에서는 LSTM이 방출 스코어를 계산하고, CRF 레이어에서 비터비를 실행해 최적 태그 시퀀스를 찾는다. **딥러닝 + 비터비**의 조합은 NER, 품사 태깅에서 여전히 강력한 베이스라인이다.

---

## 참고 자료
- [Viterbi Algorithm — Wikipedia](https://en.wikipedia.org/wiki/Viterbi_algorithm)
- [Viterbi Algorithm for HMMs — GeeksforGeeks](https://www.geeksforgeeks.org/artificial-intelligence/viterbi-algorithm-for-hidden-markov-models-hmms/)
- [The Viterbi Algorithm: A Personal History (Andrew Viterbi)](https://arxiv.org/abs/cs/0504020)
- [Hidden Markov Models Tutorial — AudioLabs Erlangen](https://www.audiolabs-erlangen.de/resources/MIR/FMP/C5/C5S3_Viterbi.html)
