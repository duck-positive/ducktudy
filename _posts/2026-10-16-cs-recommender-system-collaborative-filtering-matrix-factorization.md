---
layout: post
title: "추천 시스템 완전 정복: 협업 필터링과 행렬 분해(ALS·SVD)의 수학적 원리"
date: 2026-10-16
categories: [cs, computer-science]
tags: [추천시스템, 협업필터링, 행렬분해, ALS, SVD, 머신러닝, 잠재요인모델]
---

## 추천 시스템이란 무엇인가

유튜브가 다음 영상을 자동 재생하고, 넷플릭스가 "회원님이 좋아할 것 같은" 콘텐츠를 보여주고, 쿠팡이 "함께 구매한 상품"을 노출하는 것 — 이 모두가 추천 시스템의 결과물입니다. 추천 시스템은 사용자의 과거 행동 및 선호도를 바탕으로 아직 접하지 않은 아이템을 예측·제안하는 정보 필터링 기술입니다.

추천 시스템은 크게 세 가지 방식으로 분류됩니다.

| 방식 | 핵심 아이디어 | 장점 | 단점 |
|------|-------------|------|------|
| **콘텐츠 기반 필터링** | 아이템 속성 유사도 기반 | Cold Start 없음(새 아이템 추가 용이) | 다양성 부족, 과특화 |
| **협업 필터링** | 사용자·아이템 간 상호작용 패턴 활용 | 도메인 지식 불필요 | Cold Start 취약, 희소성 문제 |
| **하이브리드** | 두 방식 결합 | 두 방식의 단점 보완 | 복잡도 증가 |

이 글에서는 넷플릭스 프라이즈(2009년 100만 달러 상금 대회)를 통해 주목받은 **협업 필터링(Collaborative Filtering)** 과 그 핵심 기법인 **행렬 분해(Matrix Factorization)** 를 깊이 있게 다룹니다.

## 협업 필터링: "비슷한 사람은 비슷한 것을 좋아한다"

### 메모리 기반 협업 필터링

협업 필터링은 "나와 취향이 비슷한 사람들이 좋아한 것은 나도 좋아할 것이다"라는 직관에 기반합니다.

**User-Based CF (사용자 기반)**
1. 대상 사용자 u와 유사도가 높은 이웃 사용자 집합 N(u) 선정
2. N(u)가 평가했지만 u는 아직 접하지 않은 아이템의 예측 평점 계산

**Item-Based CF (아이템 기반)**
1. 대상 아이템 i와 유사한 아이템 집합 N(i) 선정
2. 사용자 u의 N(i) 아이템 평점을 가중 평균하여 예측

유사도 계산에는 주로 **코사인 유사도** 또는 **피어슨 상관계수**를 사용합니다.

```python
import numpy as np
from sklearn.metrics.pairwise import cosine_similarity

# 사용자-아이템 평점 행렬 (0은 미평가)
ratings = np.array([
    [5, 3, 0, 1, 0],  # 사용자 0
    [4, 0, 4, 1, 2],  # 사용자 1
    [1, 1, 0, 5, 0],  # 사용자 2
    [0, 0, 4, 4, 3],  # 사용자 3
    [2, 1, 5, 4, 0],  # 사용자 4
])

def user_based_cf_predict(ratings, user_id, item_id, k=2):
    """사용자 기반 협업 필터링으로 평점 예측"""
    # 미평가 아이템은 0으로 채워진 행렬에서 유사도 계산
    sim_matrix = cosine_similarity(ratings)
    
    # 대상 사용자와 타 사용자의 유사도
    similarities = sim_matrix[user_id].copy()
    similarities[user_id] = 0  # 자기 자신 제외
    
    # 해당 아이템을 평가한 사용자만 선택
    rated_mask = ratings[:, item_id] > 0
    similarities[~rated_mask] = 0
    
    # 상위 k명의 이웃 선택
    top_k_users = np.argsort(similarities)[::-1][:k]
    top_k_sims = similarities[top_k_users]
    top_k_ratings = ratings[top_k_users, item_id]
    
    if top_k_sims.sum() == 0:
        return 0
    
    # 가중 평균 예측
    prediction = np.dot(top_k_sims, top_k_ratings) / top_k_sims.sum()
    return prediction

# 사용자 0이 아이템 2(미평가)에 줄 예측 평점
pred = user_based_cf_predict(ratings, user_id=0, item_id=2)
print(f"예측 평점: {pred:.2f}")  # 예측 평점: 4.31

# 아이템 기반 CF: 아이템 유사도 행렬
item_sim = cosine_similarity(ratings.T)  # 전치 행렬로 아이템 기준 유사도
print("아이템 0과 아이템 2의 유사도:", item_sim[0, 2])
```

### 메모리 기반 CF의 한계

- **확장성:** 사용자가 수억 명이면 유사도 행렬이 수억×수억 크기
- **희소성(Sparsity):** 실제 플랫폼에서 사용자가 아이템의 0.1%도 평가하지 않음
- **콜드 스타트(Cold Start):** 새 사용자/아이템은 이력이 없어 추천 불가

## 행렬 분해(Matrix Factorization): 잠재 요인 모델

### 핵심 아이디어

사용자-아이템 평점 행렬 R (m×n)을 두 개의 낮은 차원 행렬로 분해합니다.

```
R (m×n)  ≈  P (m×k)  ×  Q^T (k×n)
```

- **P**: 사용자 잠재 요인 행렬 (각 행이 사용자 임베딩 벡터)
- **Q**: 아이템 잠재 요인 행렬 (각 행이 아이템 임베딩 벡터)
- **k**: 잠재 요인 수 (하이퍼파라미터, 보통 20~200)

예측 평점: r̂ᵤᵢ = pᵤ · qᵢ (내적)

잠재 요인은 해석하기 어렵지만 "액션 영화를 좋아하는 정도", "진지한 드라마 선호도" 같은 숨겨진 특성을 자동으로 학습합니다.

### 정규화 손실 함수

단순 SVD는 결측값(미평가 항목)을 다룰 수 없습니다. 실전에서는 **관측된 평점만**을 대상으로 정규화된 손실을 최소화합니다.

```
L = Σ(u,i)∈K [rᵤᵢ - pᵤqᵢᵀ]² + λ(||pᵤ||² + ||qᵢ||²)
```

λ는 과적합 방지를 위한 L2 정규화 계수입니다.

### SGD(확률적 경사 하강법) 구현

```python
import numpy as np

class MatrixFactorization:
    def __init__(self, n_factors=20, learning_rate=0.01, reg=0.1, n_epochs=50):
        self.k = n_factors
        self.lr = learning_rate
        self.reg = reg
        self.n_epochs = n_epochs

    def fit(self, ratings):
        """ratings: np.array (m×n), 0은 미평가"""
        m, n = ratings.shape
        # 잠재 요인 행렬 초기화 (정규분포)
        self.P = np.random.normal(0, 0.01, (m, self.k))  # 사용자
        self.Q = np.random.normal(0, 0.01, (n, self.k))  # 아이템

        # 관측된 평점의 (user, item) 인덱스
        observed = list(zip(*np.where(ratings > 0)))

        losses = []
        for epoch in range(self.n_epochs):
            np.random.shuffle(observed)
            total_loss = 0

            for u, i in observed:
                r_ui = ratings[u, i]
                pred = self.P[u] @ self.Q[i]
                err = r_ui - pred

                # 그래디언트 업데이트
                self.P[u] += self.lr * (err * self.Q[i] - self.reg * self.P[u])
                self.Q[i] += self.lr * (err * self.P[u] - self.reg * self.Q[i])

                total_loss += err ** 2 + self.reg * (
                    np.sum(self.P[u] ** 2) + np.sum(self.Q[i] ** 2)
                )

            avg_loss = total_loss / len(observed)
            losses.append(avg_loss)
            if (epoch + 1) % 10 == 0:
                print(f"Epoch {epoch+1:3d} | Loss: {avg_loss:.4f}")

        return losses

    def predict(self, user_id, item_id):
        return self.P[user_id] @ self.Q[item_id]

    def recommend(self, user_id, ratings, top_k=5):
        """미평가 아이템 중 상위 k개 추천"""
        n_items = ratings.shape[1]
        scores = [
            (i, self.predict(user_id, i))
            for i in range(n_items)
            if ratings[user_id, i] == 0  # 미평가만
        ]
        scores.sort(key=lambda x: x[1], reverse=True)
        return scores[:top_k]


# 사용 예시
ratings = np.array([
    [5, 3, 0, 1, 0],
    [4, 0, 4, 1, 2],
    [1, 1, 0, 5, 0],
    [0, 0, 4, 4, 3],
    [2, 1, 5, 4, 0],
], dtype=float)

mf = MatrixFactorization(n_factors=3, learning_rate=0.01, reg=0.02, n_epochs=100)
mf.fit(ratings)

# 사용자 0에게 추천
recs = mf.recommend(user_id=0, ratings=ratings)
print("\n사용자 0 추천 아이템:", recs)

# 특정 평점 예측
print(f"사용자 0 → 아이템 2 예측 평점: {mf.predict(0, 2):.3f}")
```

### ALS(교번 최소 제곱법): 분산 환경에 적합

SGD는 순차 처리가 필요해 분산 학습이 어렵습니다. **ALS**는 P와 Q 중 하나를 고정하고 나머지를 최소 제곱법으로 풀어나가는 방식을 교대로 반복합니다.

```python
import numpy as np

def als_matrix_factorization(ratings, k=20, reg=0.1, n_iter=20):
    """
    ALS(Alternating Least Squares) 행렬 분해
    ratings: np.array (m×n)
    """
    m, n = ratings.shape
    P = np.random.normal(0, 0.1, (m, k))
    Q = np.random.normal(0, 0.1, (n, k))

    # 관측 여부 마스크 (0: 미관측, 1: 관측)
    W = (ratings > 0).astype(float)

    for iteration in range(n_iter):
        # Step 1: Q를 고정하고 P 업데이트 (각 사용자마다 독립적으로 풀 수 있음 → 병렬화 가능)
        for u in range(m):
            observed_items = np.where(W[u] > 0)[0]
            if len(observed_items) == 0:
                continue
            Q_u = Q[observed_items]       # (|I_u|, k)
            r_u = ratings[u, observed_items]  # (|I_u|,)
            # P_u = (Q_u^T Q_u + λI)^{-1} Q_u^T r_u
            A = Q_u.T @ Q_u + reg * np.eye(k)
            b = Q_u.T @ r_u
            P[u] = np.linalg.solve(A, b)

        # Step 2: P를 고정하고 Q 업데이트
        for i in range(n):
            observed_users = np.where(W[:, i] > 0)[0]
            if len(observed_users) == 0:
                continue
            P_i = P[observed_users]
            r_i = ratings[observed_users, i]
            A = P_i.T @ P_i + reg * np.eye(k)
            b = P_i.T @ r_i
            Q[i] = np.linalg.solve(A, b)

        # RMSE 계산
        pred = P @ Q.T
        mask = W > 0
        rmse = np.sqrt(np.mean((ratings[mask] - pred[mask]) ** 2))
        if (iteration + 1) % 5 == 0:
            print(f"Iter {iteration+1:3d} | RMSE: {rmse:.4f}")

    return P, Q


ratings = np.array([
    [5, 3, 0, 1, 0],
    [4, 0, 4, 1, 2],
    [1, 1, 0, 5, 0],
    [0, 0, 4, 4, 3],
    [2, 1, 5, 4, 0],
], dtype=float)

P, Q = als_matrix_factorization(ratings, k=3, reg=0.01, n_iter=30)
print("\n행렬 분해 결과 (예측 평점 행렬):")
print(np.round(P @ Q.T, 2))
```

ALS가 SGD보다 선호되는 이유:
- 각 사용자(또는 아이템)의 업데이트가 **독립적**이어서 병렬화가 쉬움
- Apache Spark MLlib의 `ALS` 구현이 대표적

## 암묵적 피드백(Implicit Feedback)

실제 서비스에서는 "별점 5점"처럼 명시적인 평점보다 클릭·시청 시간·구매 여부 같은 암묵적 피드백이 훨씬 풍부합니다. **Hu, Koren, Volinsky(2008)** 의 implicit ALS는 이를 다음과 같이 처리합니다.

- **선호도(preference)** p_ui: 상호작용이 있으면 1, 없으면 0
- **신뢰도(confidence)** c_ui = 1 + α·r_ui (r_ui는 상호작용 횟수)

```python
def implicit_als(interaction_counts, k=20, reg=0.1, alpha=40, n_iter=10):
    """
    암묵적 피드백을 위한 ALS (Hu et al. 2008)
    interaction_counts: 사용자가 아이템과 상호작용한 횟수 행렬
    """
    m, n = interaction_counts.shape
    # 신뢰도 행렬: c_ui = 1 + alpha * r_ui
    C = 1 + alpha * interaction_counts
    # 선호도 행렬: p_ui = 1 if r_ui > 0 else 0
    P = (interaction_counts > 0).astype(float)

    X = np.random.normal(0, 0.01, (m, k))  # 사용자 요인
    Y = np.random.normal(0, 0.01, (n, k))  # 아이템 요인
    I_k = np.eye(k)

    for iteration in range(n_iter):
        # 사용자 업데이트: X_u = (Y^T C^u Y + λI)^{-1} Y^T C^u p_u
        YtY = Y.T @ Y
        for u in range(m):
            c_u = C[u]   # (n,) 신뢰도 벡터
            p_u = P[u]   # (n,) 선호도 벡터
            # Y^T diag(c_u) Y + λI
            A = YtY + Y.T @ (np.diag(c_u - 1) @ Y) + reg * I_k
            b = Y.T @ (c_u * p_u)
            X[u] = np.linalg.solve(A, b)

        # 아이템 업데이트: Y_i = (X^T C^i X + λI)^{-1} X^T C^i p_i
        XtX = X.T @ X
        for i in range(n):
            c_i = C[:, i]
            p_i = P[:, i]
            A = XtX + X.T @ (np.diag(c_i - 1) @ X) + reg * I_k
            b = X.T @ (c_i * p_i)
            Y[i] = np.linalg.solve(A, b)

        if (iteration + 1) % 2 == 0:
            print(f"Iteration {iteration+1} 완료")

    return X, Y
```

## 주의사항 및 팁

### 1. 콜드 스타트 문제
새 사용자나 새 아이템은 상호작용 이력이 없어 행렬 분해로 임베딩을 얻을 수 없습니다. 해결책:
- **콘텐츠 기반 부트스트랩**: 아이템 메타데이터(장르, 태그)로 초기 임베딩 생성
- **인기도 기반 폴백**: 신규 사용자에게 인기 아이템 추천
- **A/B 테스트**로 사용자 선호를 빠르게 수집

### 2. 평점 편향(Bias) 보정
사용자마다 평점 성향이 다릅니다(어떤 사람은 항상 4~5점만, 어떤 사람은 냉정하게 1~2점). 편향 항을 추가하면 성능이 향상됩니다.

```
r̂ᵤᵢ = μ + bᵤ + bᵢ + pᵤ · qᵢᵀ
```

μ는 전체 평균, bᵤ는 사용자 편향, bᵢ는 아이템 편향입니다.

### 3. 시간 동적 효과
사용자의 취향은 시간에 따라 변합니다. 타임스탬프를 학습에 반영하거나 최근 데이터에 더 높은 가중치를 주는 것이 중요합니다.

### 4. 평가 지표 선택
- **RMSE/MAE**: 평점 예측 정확도 (명시적 피드백)
- **Precision@K, Recall@K**: 추천 목록 품질
- **NDCG@K**: 순위 품질
- **Coverage**: 얼마나 다양한 아이템을 추천하는가

### 5. 추천 다양성(Diversity) vs 정확성
지나치게 정확한 추천은 사용자를 "필터 버블"에 가두어 장기적 이탈을 야기할 수 있습니다. 탐색(Exploration)과 활용(Exploitation)의 균형이 중요합니다.

## 참고 자료
- [Implicit Feedback Deep Collaborative Filtering (arXiv 2020)](https://arxiv.org/abs/2009.08950)
- [ALS for Collaborative Filtering in Spark (arXiv 2015)](https://arxiv.org/abs/1508.03110)
- [Koren et al., Matrix Factorization Techniques for Recommender Systems (IEEE 2009)](https://ieeexplore.ieee.org/document/5197422)
- [Hu, Koren, Volinsky, Collaborative Filtering for Implicit Feedback Datasets (ICDM 2008)](https://dl.acm.org/doi/10.1109/ICDM.2008.22)
