---
layout: post
title: "트랜스포머 어텐션 메커니즘 심층 분석: Self-Attention부터 Flash Attention까지"
date: 2026-08-13
categories: [cs, computer-science]
tags: [transformer, attention, self-attention, multi-head-attention, flash-attention, llm, deep-learning]
---

2017년 Vaswani 등이 발표한 논문 "Attention Is All You Need"는 자연어 처리와 AI 전반을 혁신했다. 트랜스포머 아키텍처의 핵심인 어텐션 메커니즘은 현재 ChatGPT, Claude, Gemini 같은 대형 언어 모델(LLM)의 기반이 된다. 이 글에서는 어텐션의 수학적 원리부터 실제 구현, 그리고 GPU 메모리를 혁신적으로 최적화한 Flash Attention까지 심층적으로 분석한다.

## 어텐션 이전의 세계: RNN의 한계

### 순환 신경망(RNN)의 병목

트랜스포머 이전, 시퀀스 처리의 표준은 RNN/LSTM이었다. 하지만 근본적인 한계가 있었다:

```
RNN 정보 흐름:
h1 → h2 → h3 → ... → hn → 출력

문제 1: 장거리 의존성 (Long-range Dependency)
  "The cat that was sitting on the mat [sat] is..."
  "cat"과 "sat"의 관계를 n 스텝 거리를 통해 전파 → 경사 소실

문제 2: 순차 처리 (Sequential Processing)
  h_t = f(h_{t-1}, x_t)
  각 스텝이 이전 스텝에 의존 → GPU 병렬화 불가능

문제 3: 고정 크기 컨텍스트 벡터
  인코더-디코더 구조에서 모든 정보를 하나의 벡터에 압축
  긴 문장에서 병목(bottleneck) 발생
```

어텐션 메커니즘은 이 문제를 근본적으로 해결한다. 모든 위치 쌍의 관계를 **직접** 계산하는 것이다.

## 스케일드 닷-프로덕트 어텐션

### 핵심 수식

트랜스포머 어텐션의 핵심 수식은 간결하지만 강력하다:

$$\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V$$

여기서 Q(Query), K(Key), V(Value)는 입력 시퀀스를 선형 투영한 행렬이다.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
import math

def scaled_dot_product_attention(Q, K, V, mask=None):
    """
    스케일드 닷-프로덕트 어텐션 구현
    
    Args:
        Q: [batch, seq_q, d_k] — 쿼리 행렬
        K: [batch, seq_k, d_k] — 키 행렬  
        V: [batch, seq_k, d_v] — 값 행렬
        mask: 어텐션 마스크 (패딩 또는 인과적 마스킹)
    
    Returns:
        output: [batch, seq_q, d_v]
        attn_weights: [batch, seq_q, seq_k]
    """
    d_k = Q.shape[-1]
    
    # 1. 어텐션 스코어 계산: Q·K^T / √d_k
    # [batch, seq_q, d_k] × [batch, d_k, seq_k] = [batch, seq_q, seq_k]
    scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(d_k)
    
    # 2. 마스킹 (선택적)
    if mask is not None:
        scores = scores.masked_fill(mask == 0, float('-inf'))
    
    # 3. Softmax로 어텐션 가중치 계산
    attn_weights = F.softmax(scores, dim=-1)
    
    # 4. 값 벡터의 가중 합산
    # [batch, seq_q, seq_k] × [batch, seq_k, d_v] = [batch, seq_q, d_v]
    output = torch.matmul(attn_weights, V)
    
    return output, attn_weights


# √d_k 스케일링이 왜 필요한가?
def demonstrate_scaling_effect():
    """
    스케일링 없이는 소프트맥스 포화 발생
    """
    d_k = 64
    seq_len = 10
    
    # 랜덤 Q, K
    Q = torch.randn(1, seq_len, d_k)
    K = torch.randn(1, seq_len, d_k)
    
    # 스케일링 없는 점수
    scores_unscaled = torch.matmul(Q, K.transpose(-2, -1))
    # 스케일링된 점수
    scores_scaled = scores_unscaled / math.sqrt(d_k)
    
    print(f"d_k = {d_k}")
    print(f"비스케일 점수 표준편차: {scores_unscaled.std():.4f}")
    print(f"스케일된 점수 표준편차: {scores_scaled.std():.4f}")
    
    # 소프트맥스 출력 분포 비교
    softmax_unscaled = F.softmax(scores_unscaled, dim=-1)
    softmax_scaled = F.softmax(scores_scaled, dim=-1)
    
    print(f"\n소프트맥스 엔트로피 (높을수록 분산된 어텐션):")
    entropy_unscaled = -(softmax_unscaled * torch.log(softmax_unscaled + 1e-9)).sum(-1).mean()
    entropy_scaled = -(softmax_scaled * torch.log(softmax_scaled + 1e-9)).sum(-1).mean()
    print(f"  비스케일: {entropy_unscaled:.4f} (포화됨 → 경사 소실)")
    print(f"  스케일:   {entropy_scaled:.4f} (건강한 분포)")

demonstrate_scaling_effect()
```

### Q, K, V의 직관적 이해

어텐션은 **정보 검색(information retrieval)** 시스템으로 이해할 수 있다:

```
데이터베이스 검색 비유:
  Query (Q): "나는 무엇을 찾고 있는가?" — 검색어
  Key   (K): "나는 무슨 정보를 가지고 있는가?" — 색인
  Value (V): "실제 반환할 정보" — 내용

예시: "The bank can guarantee deposits will also cover future tuition costs"
  "bank"의 Query가 "강둑"인지 "금융기관"인지 문맥에 따라 결정
  → "deposits", "guarantee", "future"의 Key와 높은 어텐션 스코어
  → 금융 의미의 Value 정보 집중
```

## 멀티-헤드 어텐션 (Multi-Head Attention)

단일 어텐션은 하나의 관점에서만 관계를 학습한다. 멀티-헤드는 여러 관점을 동시에 학습한다.

```python
class MultiHeadAttention(nn.Module):
    """
    멀티-헤드 어텐션 완전 구현
    """
    def __init__(self, d_model: int, num_heads: int, dropout: float = 0.1):
        super().__init__()
        assert d_model % num_heads == 0
        
        self.d_model = d_model
        self.num_heads = num_heads
        self.d_k = d_model // num_heads  # 각 헤드의 차원
        
        # Q, K, V를 위한 선형 투영 레이어 (통합)
        self.W_q = nn.Linear(d_model, d_model, bias=False)
        self.W_k = nn.Linear(d_model, d_model, bias=False)
        self.W_v = nn.Linear(d_model, d_model, bias=False)
        
        # 출력 투영
        self.W_o = nn.Linear(d_model, d_model, bias=False)
        self.dropout = nn.Dropout(dropout)
    
    def split_heads(self, x: torch.Tensor) -> torch.Tensor:
        """[batch, seq, d_model] → [batch, num_heads, seq, d_k]"""
        batch, seq, d_model = x.shape
        x = x.view(batch, seq, self.num_heads, self.d_k)
        return x.transpose(1, 2)  # [batch, heads, seq, d_k]
    
    def forward(self, Q, K, V, mask=None):
        batch_size = Q.shape[0]
        
        # 1. 선형 투영 후 헤드 분리
        Q = self.split_heads(self.W_q(Q))  # [batch, heads, seq_q, d_k]
        K = self.split_heads(self.W_k(K))  # [batch, heads, seq_k, d_k]
        V = self.split_heads(self.W_v(V))  # [batch, heads, seq_k, d_k]
        
        # 2. 각 헤드에 병렬로 어텐션 계산
        d_k = Q.shape[-1]
        scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(d_k)
        
        if mask is not None:
            scores = scores.masked_fill(mask == 0, float('-inf'))
        
        attn_weights = self.dropout(F.softmax(scores, dim=-1))
        output = torch.matmul(attn_weights, V)  # [batch, heads, seq_q, d_k]
        
        # 3. 헤드 결합 후 출력 투영
        output = output.transpose(1, 2)  # [batch, seq_q, heads, d_k]
        output = output.contiguous().view(batch_size, -1, self.d_model)
        
        return self.W_o(output), attn_weights


# 사용 예시
def demo_multihead_attention():
    d_model, num_heads, seq_len = 512, 8, 128
    batch_size = 4
    
    mha = MultiHeadAttention(d_model=d_model, num_heads=num_heads)
    
    # 자기 어텐션(Self-Attention): Q=K=V
    x = torch.randn(batch_size, seq_len, d_model)
    output, attn = mha(x, x, x)
    
    print(f"입력 형태:  {x.shape}")
    print(f"출력 형태:  {output.shape}")
    print(f"어텐션 형태: {attn.shape}  (batch, heads, seq, seq)")
    print(f"  각 헤드가 다른 관계 패턴을 학습:")
    print(f"  헤드 1 → 문법적 관계 (주어-동사)")
    print(f"  헤드 2 → 공참조 관계 (대명사-명사)")
    print(f"  헤드 3 → 인접 관계 (bigram)")
    print(f"  ... (8개 헤드가 서로 다른 관점 학습)")

demo_multihead_attention()
```

## 어텐션의 계산 복잡도와 Flash Attention

### 표준 어텐션의 병목

표준 어텐션의 계산 복잡도는 시퀀스 길이 n에 대해 **O(n²)** 이다. n=4096인 경우 어텐션 행렬은 4096×4096 = 16M 원소로, A100 GPU의 HBM(고대역폭 메모리)에 상당한 부담을 준다.

```
표준 어텐션의 메모리 접근 패턴:
┌─────────────┬───────────────────────────────────────┐
│ 단계        │ HBM 읽기/쓰기                         │
├─────────────┼───────────────────────────────────────┤
│ Q·K^T 계산  │ Q, K 읽기 → S 쓰기 (O(n²) 메모리)   │
│ Softmax(S)  │ S 읽기 → P 쓰기 (O(n²) 메모리)       │
│ P·V 계산    │ P, V 읽기 → O 쓰기                   │
└─────────────┴───────────────────────────────────────┘

→ O(n²) 크기의 어텐션 행렬을 HBM에 저장
→ HBM 대역폭이 병목 (연산보다 메모리 접근이 느림)
```

### Flash Attention: IO-Aware 알고리즘

2022년 Dao 등이 제안한 Flash Attention은 **타일링(tiling)** 기법으로 어텐션을 SRAM에서 청크 단위로 계산하여 HBM 접근을 획기적으로 줄인다.

```python
def flash_attention_concept(Q, K, V, block_size=64):
    """
    Flash Attention의 핵심 아이디어: 타일드 소프트맥스
    
    핵심 통찰: 소프트맥스를 온라인으로 (incrementally) 계산 가능
    
    표준: softmax(x) = exp(x) / sum(exp(x))
         → 전체 x를 먼저 알아야 함 → O(n) 메모리
    
    온라인: 수치 안정성을 유지하며 블록별 계산
    """
    N, d = Q.shape[-2], Q.shape[-1]
    
    # 출력과 통계값 초기화
    O = torch.zeros_like(Q)  # 출력 누적
    l = torch.zeros(N)       # 분모 누적 (sum of exp)
    m = torch.full((N,), float('-inf'))  # 최대값 추적 (수치 안정성)
    
    print("Flash Attention 타일링 계산:")
    print(f"  시퀀스 길이: {N}, 블록 크기: {block_size}")
    print(f"  블록 수: {N // block_size}")
    print(f"  각 블록은 SRAM에서 처리 → HBM 접근 최소화")
    
    # KV 블록을 순회
    for j in range(0, N, block_size):
        K_j = K[..., j:j+block_size, :]  # SRAM에 로드
        V_j = V[..., j:j+block_size, :]
        
        # Q 블록을 순회
        for i in range(0, N, block_size):
            Q_i = Q[..., i:i+block_size, :]  # SRAM에 로드
            
            # 블록 어텐션 점수
            S_ij = torch.matmul(Q_i, K_j.transpose(-2, -1)) / math.sqrt(d)
            
            # 온라인 소프트맥스 업데이트
            m_new = torch.maximum(m[i:i+block_size], S_ij.max(-1).values)
            l_new = torch.exp(m[i:i+block_size] - m_new) * l[i:i+block_size] + \
                    torch.exp(S_ij - m_new.unsqueeze(-1)).sum(-1)
            
            # 누적 출력 업데이트 (재스케일링)
            O[..., i:i+block_size, :] = (
                torch.exp(m[i:i+block_size] - m_new).unsqueeze(-1) * O[..., i:i+block_size, :]
                + torch.matmul(torch.exp(S_ij - m_new.unsqueeze(-1)), V_j)
            ) / l_new.unsqueeze(-1)
            
            # 통계 업데이트
            m[i:i+block_size] = m_new
            l[i:i+block_size] = l_new
    
    print("\nFlash Attention vs 표준 어텐션:")
    print(f"  HBM 읽기/쓰기: O(n²) → O(n) 감소")
    print(f"  메모리 사용: O(n²) → O(n) 감소")
    print(f"  계산량: 동일 O(n²d) (연산 수는 같음)")
    print(f"  속도: 2-4배 향상 (메모리 대역폭이 병목이었으므로)")
    
    return O

# 실제 사용: torch.nn.functional.scaled_dot_product_attention
def modern_efficient_attention():
    """
    PyTorch 2.0+의 최적화된 어텐션
    """
    Q = torch.randn(2, 8, 512, 64, device='cuda' if torch.cuda.is_available() else 'cpu')
    K = torch.randn(2, 8, 512, 64, device=Q.device)
    V = torch.randn(2, 8, 512, 64, device=Q.device)
    
    # PyTorch가 자동으로 Flash Attention 사용 (CUDA 환경)
    with torch.backends.cuda.sdp_kernel(
        enable_flash=True,
        enable_math=False,
        enable_mem_efficient=False
    ):
        out = F.scaled_dot_product_attention(Q, K, V)
    
    print(f"Flash Attention 출력 형태: {out.shape}")
    
    # 인과적 마스크 (언어 모델용 — 미래 토큰 참조 금지)
    out_causal = F.scaled_dot_product_attention(Q, K, V, is_causal=True)
    print(f"인과적 어텐션 출력 형태: {out_causal.shape}")

modern_efficient_attention()
```

## 포지셔널 인코딩과 어텐션의 완성

어텐션은 위치 정보가 없다. 동일한 토큰이 시퀀스의 어느 위치에 있어도 같은 결과를 낸다. 포지셔널 인코딩이 이를 보완한다.

```python
class PositionalEncoding(nn.Module):
    """사인/코사인 포지셔널 인코딩"""
    def __init__(self, d_model: int, max_len: int = 5000):
        super().__init__()
        pe = torch.zeros(max_len, d_model)
        position = torch.arange(0, max_len).unsqueeze(1).float()
        div_term = torch.exp(
            torch.arange(0, d_model, 2).float() * (-math.log(10000.0) / d_model)
        )
        pe[:, 0::2] = torch.sin(position * div_term)  # 짝수 인덱스
        pe[:, 1::2] = torch.cos(position * div_term)  # 홀수 인덱스
        self.register_buffer('pe', pe.unsqueeze(0))  # [1, max_len, d_model]
    
    def forward(self, x):
        return x + self.pe[:, :x.size(1)]
```

## 주의사항 및 실전 팁

### 1. 어텐션 복잡도와 컨텍스트 길이

O(n²) 복잡도는 긴 컨텍스트에서 급격히 증가한다:
- n=2K: 4M 원소 (가볍다)
- n=32K: 1G 원소 (무겁다)
- n=1M: 1T 원소 (불가능)

해결책: Sliding Window Attention(Mistral), Linear Attention(Performer), RoPE(회전 위치 인코딩) + Flash Attention v3.

### 2. 어텐션 헤드 수와 차원 균형

`d_model / num_heads = d_k`가 너무 작으면 각 헤드의 표현력 저하. GPT-3는 d_model=12288, num_heads=96으로 d_k=128을 유지한다.

### 3. KV 캐시 최적화

추론 시 디코딩에서 각 스텝마다 전체 어텐션을 재계산하지 않고 Key/Value를 캐시한다. 컨텍스트 길이가 길수록 캐시 크기가 선형 증가하므로, **그룹화 쿼리 어텐션(GQA)** 으로 KV 헤드 수를 줄여 메모리를 절약한다.

```
Multi-Head Attention:  h개의 Q, h개의 K, h개의 V
Grouped Query Attention: h개의 Q, g개의 K, g개의 V (g < h)
Multi-Query Attention: h개의 Q, 1개의 K, 1개의 V (극단적 절약)
```

트랜스포머 어텐션은 이제 단순한 NLP 기술을 넘어 비전, 오디오, 단백질 구조(AlphaFold2), 코드 생성 등 거의 모든 AI 도메인의 핵심이 되었다. 그 수학적 원리를 이해하는 것은 현대 AI 시스템을 효과적으로 설계하고 최적화하는 데 필수적이다.

## 참고 자료
- [Attention Is All You Need (Vaswani et al., 2017)](https://arxiv.org/abs/1706.03762)
- [FlashAttention: Fast and Memory-Efficient Exact Attention (Dao et al., 2022)](https://arxiv.org/abs/2205.14135)
- [Understanding and Coding Self-Attention in LLMs (Sebastian Raschka)](https://magazine.sebastianraschka.com/p/understanding-and-coding-self-attention)
- [Multi-Head Attention Mechanism — GeeksforGeeks](https://www.geeksforgeeks.org/nlp/multi-head-attention-mechanism/)
