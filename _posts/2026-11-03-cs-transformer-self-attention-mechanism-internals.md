---
layout: post
title: "Transformer 셀프 어텐션 메커니즘 내부 구조 심층 분석"
date: 2026-11-03
categories: [cs, computer-science]
tags: [transformer, attention, deep-learning, nlp, neural-network, self-attention, multi-head-attention]
---

현대 NLP(자연어 처리)와 AI 분야를 완전히 뒤바꾼 Transformer 아키텍처의 핵심은 **셀프 어텐션(Self-Attention)** 메커니즘이다. 2017년 Google Brain 팀이 발표한 "Attention Is All You Need" 논문은 RNN과 CNN 없이 순수 어텐션만으로 시퀀스 모델링을 가능하게 했다. 이 포스트에서는 셀프 어텐션의 수학적 원리부터 멀티헤드 어텐션 구현까지 내부를 완전히 해부한다.

## 왜 셀프 어텐션이 필요한가

전통적인 RNN(LSTM, GRU)은 시퀀스를 순차적으로 처리하기 때문에 다음과 같은 문제가 있었다.

- **장기 의존성(Long-range Dependency) 소실**: 멀리 떨어진 토큰 간의 관계를 학습하기 어렵다
- **병렬 처리 불가**: 순차 처리 특성상 GPU의 병렬 연산을 활용하지 못한다
- **경로 길이(Path Length)**: 두 위치 사이의 신호가 경유해야 할 연산 수가 시퀀스 길이에 비례한다

셀프 어텐션은 시퀀스의 모든 위치 쌍을 **직접 연결**하여 O(1) 경로 길이를 달성한다. 이는 임의 위치 간 관계를 단일 연산으로 포착할 수 있음을 의미한다.

## Query, Key, Value: 어텐션의 기본 구조

셀프 어텐션의 핵심은 입력 벡터로부터 세 개의 행렬을 생성하는 것이다.

```
입력 X ∈ R^(n×d_model)
Q = X·W_Q,  W_Q ∈ R^(d_model × d_k)
K = X·W_K,  W_K ∈ R^(d_model × d_k)
V = X·W_V,  W_V ∈ R^(d_model × d_v)
```

**Scaled Dot-Product Attention** 공식:

```
Attention(Q, K, V) = softmax(QK^T / √d_k) · V
```

`√d_k`로 스케일링하는 이유는 내적 값이 d_k에 비례하여 커지기 때문이다. 값이 커지면 softmax의 그래디언트가 극도로 작아져 학습이 불안정해진다.

## 코드로 구현하는 셀프 어텐션

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
import math

class ScaledDotProductAttention(nn.Module):
    def __init__(self, d_k: int, dropout: float = 0.1):
        super().__init__()
        self.scale = math.sqrt(d_k)
        self.dropout = nn.Dropout(dropout)

    def forward(self, Q, K, V, mask=None):
        # Q: (batch, heads, seq_len, d_k)
        # K: (batch, heads, seq_len, d_k)
        # V: (batch, heads, seq_len, d_v)

        # 어텐션 스코어 계산: QK^T / sqrt(d_k)
        scores = torch.matmul(Q, K.transpose(-2, -1)) / self.scale
        # scores: (batch, heads, seq_len, seq_len)

        # 마스킹 (패딩 또는 인과적 마스크)
        if mask is not None:
            scores = scores.masked_fill(mask == 0, float('-inf'))

        # Softmax로 어텐션 가중치 산출
        attn_weights = F.softmax(scores, dim=-1)
        attn_weights = self.dropout(attn_weights)

        # 가중 합산
        output = torch.matmul(attn_weights, V)
        return output, attn_weights


class MultiHeadAttention(nn.Module):
    def __init__(self, d_model: int, num_heads: int, dropout: float = 0.1):
        super().__init__()
        assert d_model % num_heads == 0

        self.d_model = d_model
        self.num_heads = num_heads
        self.d_k = d_model // num_heads

        # Q, K, V, 출력 선형 변환
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)

        self.attention = ScaledDotProductAttention(self.d_k, dropout)

    def split_heads(self, x: torch.Tensor) -> torch.Tensor:
        batch_size, seq_len, d_model = x.size()
        # (batch, seq_len, d_model) -> (batch, heads, seq_len, d_k)
        x = x.view(batch_size, seq_len, self.num_heads, self.d_k)
        return x.transpose(1, 2)

    def forward(self, query, key, value, mask=None):
        batch_size = query.size(0)

        # 선형 변환 후 헤드 분리
        Q = self.split_heads(self.W_q(query))
        K = self.split_heads(self.W_k(key))
        V = self.split_heads(self.W_v(value))

        # 어텐션 계산
        attn_output, attn_weights = self.attention(Q, K, V, mask)

        # 헤드 병합 후 출력 선형 변환
        # (batch, heads, seq_len, d_k) -> (batch, seq_len, d_model)
        attn_output = attn_output.transpose(1, 2).contiguous()
        attn_output = attn_output.view(batch_size, -1, self.d_model)

        return self.W_o(attn_output), attn_weights


# 사용 예시
d_model = 512
num_heads = 8
batch_size, seq_len = 2, 16

mha = MultiHeadAttention(d_model=d_model, num_heads=num_heads)
x = torch.randn(batch_size, seq_len, d_model)

# 셀프 어텐션: query, key, value 모두 동일한 입력
output, weights = mha(x, x, x)
print(f"Output shape: {output.shape}")   # (2, 16, 512)
print(f"Attention weights: {weights.shape}")  # (2, 8, 16, 16)
```

## 멀티헤드 어텐션의 핵심 통찰

단일 어텐션 헤드는 각 위치에서 하나의 관점만으로 다른 위치를 바라본다. **멀티헤드 어텐션(Multi-Head Attention)**은 h개의 서로 다른 어텐션을 병렬로 실행하여 다양한 표현 공간에서의 관계를 동시에 포착한다.

```
MultiHead(Q, K, V) = Concat(head_1, ..., head_h) · W_O
head_i = Attention(QW_i^Q, KW_i^K, VW_i^V)
```

각 헤드는 독립적인 W_Q, W_K, W_V 파라미터를 가지므로, 서로 다른 관계를 학습한다. 예를 들어 한 헤드는 구문적 관계를, 다른 헤드는 의미적 관계를 포착할 수 있다.

## 인과적 마스킹 (Causal Masking)

GPT 계열의 **디코더(Decoder)** 모델에서는 현재 위치 이후의 토큰을 참조할 수 없도록 **마스크**를 적용한다. 이를 **인과적 어텐션(Causal Attention)** 또는 **마스크드 어텐션(Masked Attention)**이라 한다.

```python
def create_causal_mask(seq_len: int, device: torch.device) -> torch.Tensor:
    """미래 토큰을 보지 못하도록 하는 하삼각 마스크 생성"""
    # 1 = 볼 수 있음, 0 = 마스킹
    mask = torch.tril(torch.ones(seq_len, seq_len, device=device))
    return mask.unsqueeze(0).unsqueeze(0)  # (1, 1, seq_len, seq_len)


def create_padding_mask(src: torch.Tensor, pad_token_id: int) -> torch.Tensor:
    """패딩 토큰을 어텐션에서 제외하는 마스크 생성"""
    # src: (batch, seq_len)
    mask = (src != pad_token_id)  # (batch, seq_len)
    return mask.unsqueeze(1).unsqueeze(2)  # (batch, 1, 1, seq_len)


# 인과적 마스크 확인
seq_len = 5
causal_mask = create_causal_mask(seq_len, device=torch.device('cpu'))
print("Causal Mask:")
print(causal_mask.squeeze())
# tensor([[1., 0., 0., 0., 0.],
#         [1., 1., 0., 0., 0.],
#         [1., 1., 1., 0., 0.],
#         [1., 1., 1., 1., 0.],
#         [1., 1., 1., 1., 1.]])
```

## 포지셔널 인코딩 (Positional Encoding)

어텐션 연산 자체는 **순서 불변(Permutation-Invariant)**이다. 토큰의 위치 정보를 주입하기 위해 포지셔널 인코딩을 입력에 더한다.

원래 논문에서는 사인/코사인 함수를 사용한다:

```
PE(pos, 2i)   = sin(pos / 10000^(2i/d_model))
PE(pos, 2i+1) = cos(pos / 10000^(2i/d_model))
```

이 수식의 특성:
- 같은 오프셋을 가진 두 위치의 PE는 선형 변환으로 표현 가능 → 상대적 위치 관계 학습 가능
- 훈련 중 본 것보다 긴 시퀀스에도 적용 가능 (외삽 가능)

## 어텐션 복잡도와 한계

셀프 어텐션의 시간/공간 복잡도는 시퀀스 길이 n에 대해 **O(n²)**이다. 이는 긴 시퀀스를 처리할 때 병목이 된다.

| 접근 방식 | 복잡도 | 특징 |
|---|---|---|
| Full Self-Attention | O(n²·d) | 표준 Transformer |
| Sparse Attention | O(n·√n·d) | Longformer, BigBird |
| Linear Attention | O(n·d) | Performer, Linear Transformer |
| Flash Attention | O(n²·d) | IO-aware, GPU 메모리 효율화 |

FlashAttention은 알고리즘적 복잡도는 동일하지만 GPU HBM과 SRAM 간의 데이터 이동을 최소화하여 실제로 수 배 빠른 속도를 달성한다.

## 주의사항 및 실무 팁

**1. 수치 안정성**: softmax에 `-inf`를 사용할 때 모든 위치가 마스킹되면 NaN이 발생할 수 있다. 마스크 이후 최소 하나의 위치가 살아있는지 확인하라.

**2. 그래디언트 소실**: 매우 긴 시퀀스에서 어텐션 가중치가 균일하게 분산되면 학습 신호가 희석된다. Layer Normalization과 Residual Connection이 이를 완화한다.

**3. KV 캐싱**: 추론 시 이미 처리한 토큰의 K, V를 캐싱하면 각 토큰 생성 비용을 O(n·d)로 줄일 수 있다. 프로덕션 LLM 서빙에서 필수 최적화다.

**4. 헤드 수와 d_k 균형**: d_model=512, num_heads=8이면 d_k=64. d_k가 너무 작으면 표현력이 떨어지고, 너무 크면 각 헤드가 유사한 것을 학습하는 경향이 있다.

셀프 어텐션은 단순한 행렬 연산의 조합이지만, 그 안에 인류가 만든 가장 강력한 표현 학습 메커니즘이 담겨 있다. GPT, BERT, T5, LLaMA 등 모든 현대 LLM의 근간이 여기서 출발한다.

## 참고 자료
- [Attention Is All You Need (Vaswani et al., 2017)](https://arxiv.org/abs/1706.03762)
- [The Illustrated Transformer – Jay Alammar](https://jalammar.github.io/illustrated-transformer/)
- [An Introduction to Transformers (Turner, 2023)](https://arxiv.org/abs/2304.10557)
- [FlashAttention: Fast and Memory-Efficient Exact Attention](https://arxiv.org/abs/2205.14135)
