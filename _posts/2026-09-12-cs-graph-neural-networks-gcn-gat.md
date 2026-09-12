---
layout: post
title: "Graph Neural Networks 완전 정복: GCN·GAT·메시지 패싱으로 그래프 데이터를 학습하는 법"
date: 2026-09-12
categories: [cs, computer-science]
tags: [gnn, graph-neural-network, gcn, gat, message-passing, deep-learning, machine-learning, pytorch]
---

## 개요

소셜 네트워크의 사용자 관계, 분자 구조의 원자 결합, 지식 그래프의 개체 연결, 도로 네트워크의 교차로 — 세상의 수많은 데이터는 **그래프(Graph)** 형태를 띱니다. 기존 CNN은 격자(grid) 구조, RNN은 시퀀스(sequence) 구조에 특화되어 있어 비정형 그래프에 직접 적용하기 어렵습니다.

**Graph Neural Network(GNN)**은 그래프 위에서 동작하는 신경망으로, 노드·엣지·전역 특성을 동시에 학습합니다. 2018년 이후 GNN은 약물 발견(AlphaFold 2의 구조 예측), 추천 시스템(Pinterest PinSage), 반도체 칩 배치 최적화(Google Chip Design) 등 다양한 최전선 응용에서 혁신을 이끌고 있습니다. 이 글에서는 GNN의 핵심 메커니즘인 메시지 패싱부터, 대표적 구조인 GCN·GAT까지 수식과 코드로 깊이 있게 탐구합니다.

---

## 그래프의 기본 표현

그래프 G = (V, E)에서:
- **V**: 노드(vertex) 집합, |V| = N
- **E**: 엣지(edge) 집합, 각 엣지 (u, v) ∈ E
- **인접 행렬 A**: A[i][j] = 1이면 노드 i와 j가 연결
- **노드 특성 행렬 X**: X ∈ ℝ^(N×F), 각 노드의 F차원 특성 벡터

GNN의 목표는 입력 그래프와 특성으로부터:
1. **노드 분류**: 각 노드의 레이블 예측
2. **엣지 예측**: 두 노드 간 엣지 존재 여부 예측
3. **그래프 분류**: 그래프 전체의 속성 예측

---

## 메시지 패싱 신경망(MPNN)의 통합 프레임워크

대부분의 GNN은 **Message Passing Neural Network(MPNN)** 프레임워크로 통합됩니다. 각 레이어는 다음 세 단계를 반복합니다.

```
레이어 l에서 노드 v의 업데이트:

1. Message (메시지 생성):
   m_{u→v}^(l) = φ(h_v^(l), h_u^(l), e_{uv})
   - h_v^(l): 현재 레이어 노드 v의 은닉 상태
   - e_{uv}: 엣지 (u,v)의 특성
   - φ: 메시지 함수

2. Aggregate (집계):
   M_v^(l) = ⊕_{u ∈ N(v)} m_{u→v}^(l)
   - N(v): v의 이웃 노드 집합
   - ⊕: 교환법칙·결합법칙이 성립하는 집계 연산 (Sum, Mean, Max)

3. Update (업데이트):
   h_v^(l+1) = γ(h_v^(l), M_v^(l))
   - γ: 업데이트 함수 (MLP, GRU 등)
```

이 프레임워크의 핵심은 **순열 불변성(Permutation Invariance)**입니다. 이웃 노드의 순서가 바뀌어도 결과가 동일해야 하므로, 집계 연산 ⊕는 반드시 교환법칙·결합법칙이 성립해야 합니다.

---

## Graph Convolutional Network (GCN)

**GCN**은 2017년 Kipf & Welling이 제안한 가장 기본적인 GNN 구조입니다. 스펙트럼 그래프 이론에서 출발하여 간소화된 층별 전파 규칙을 유도합니다.

### 수식 유도

라플라시안 기반 그래프 합성곱을 1차 근사하면:

```
H^(l+1) = σ( D̃^(-1/2) Ã D̃^(-1/2) H^(l) W^(l) )

여기서:
- Ã = A + I  (자기 루프 추가: 자신의 특성도 포함)
- D̃: Ã의 차수 행렬 (D̃[i][i] = Σ_j Ã[i][j])
- W^(l): 학습 가능한 가중치 행렬
- σ: ReLU 등 활성화 함수

정규화 의미:
  D̃^(-1/2) Ã D̃^(-1/2)의 [i][j] 원소 = Ã[i][j] / sqrt(d̃_i * d̃_j)
  → 차수가 높은 노드의 메시지가 과도하게 강해지는 것을 방지
```

### NumPy 기반 GCN 구현 (학습용)

```python
import numpy as np

def relu(x):
    return np.maximum(0, x)

def softmax(x):
    exp_x = np.exp(x - x.max(axis=1, keepdims=True))
    return exp_x / exp_x.sum(axis=1, keepdims=True)

class GCNLayer:
    def __init__(self, in_features, out_features):
        # Xavier 초기화
        scale = np.sqrt(2.0 / (in_features + out_features))
        self.W = np.random.randn(in_features, out_features) * scale

    def forward(self, X, A_hat):
        """
        X: (N, in_features) 노드 특성
        A_hat: (N, N) 정규화된 인접 행렬 D̃^(-1/2) Ã D̃^(-1/2)
        """
        return relu(A_hat @ X @ self.W)

def normalize_adjacency(A):
    """GCN 정규화: D̃^(-1/2) Ã D̃^(-1/2)"""
    N = A.shape[0]
    A_tilde = A + np.eye(N)  # 자기 루프 추가
    D_tilde = np.diag(A_tilde.sum(axis=1))
    D_inv_sqrt = np.diag(1.0 / np.sqrt(np.diag(D_tilde)))
    return D_inv_sqrt @ A_tilde @ D_inv_sqrt

# 간단한 그래프: 5개 노드
N = 5
# 인접 행렬 (무방향)
A = np.array([
    [0, 1, 1, 0, 0],
    [1, 0, 1, 1, 0],
    [1, 1, 0, 0, 1],
    [0, 1, 0, 0, 1],
    [0, 0, 1, 1, 0],
], dtype=float)

# 노드 특성: 각 노드 = 4차원 벡터
X = np.random.randn(N, 4)

A_hat = normalize_adjacency(A)

# 2층 GCN
layer1 = GCNLayer(4, 8)
layer2 = GCNLayer(8, 3)  # 3개 클래스 분류

H1 = layer1.forward(X, A_hat)
H2 = layer2.forward(H1, A_hat)
predictions = softmax(H2)

print("노드별 클래스 확률:")
for i, pred in enumerate(predictions):
    print(f"  노드 {i}: {[f'{p:.3f}' for p in pred]}")
```

### PyTorch Geometric을 사용한 실전 GCN

```python
import torch
import torch.nn.functional as F
from torch_geometric.nn import GCNConv
from torch_geometric.datasets import Planetoid

# Cora 데이터셋 로드 (2708 노드, 10556 엣지, 7개 클래스)
dataset = Planetoid(root='/tmp/Cora', name='Cora')
data = dataset[0]

class GCN(torch.nn.Module):
    def __init__(self, in_channels, hidden_channels, out_channels):
        super().__init__()
        self.conv1 = GCNConv(in_channels, hidden_channels)
        self.conv2 = GCNConv(hidden_channels, out_channels)

    def forward(self, x, edge_index):
        # 1층: ReLU + Dropout
        x = self.conv1(x, edge_index)
        x = F.relu(x)
        x = F.dropout(x, p=0.5, training=self.training)
        # 2층: 로짓 출력
        x = self.conv2(x, edge_index)
        return F.log_softmax(x, dim=1)

model = GCN(
    in_channels=dataset.num_features,    # 1433
    hidden_channels=64,
    out_channels=dataset.num_classes,    # 7
)

optimizer = torch.optim.Adam(model.parameters(), lr=0.01, weight_decay=5e-4)

def train():
    model.train()
    optimizer.zero_grad()
    out = model(data.x, data.edge_index)
    loss = F.nll_loss(out[data.train_mask], data.y[data.train_mask])
    loss.backward()
    optimizer.step()
    return loss.item()

def test():
    model.eval()
    out = model(data.x, data.edge_index)
    pred = out.argmax(dim=1)
    accs = []
    for mask in [data.train_mask, data.val_mask, data.test_mask]:
        correct = pred[mask] == data.y[mask]
        accs.append(int(correct.sum()) / int(mask.sum()))
    return accs

for epoch in range(200):
    loss = train()
    if epoch % 50 == 0:
        train_acc, val_acc, test_acc = test()
        print(f'Epoch {epoch:3d}, Loss: {loss:.4f}, '
              f'Train: {train_acc:.4f}, Val: {val_acc:.4f}, Test: {test_acc:.4f}')
# 일반적으로 Test Acc ~0.81 달성
```

---

## Graph Attention Network (GAT)

GCN은 이웃 노드를 차수 기반으로 동등하게 정규화합니다. 하지만 실제로 이웃 노드마다 중요도가 다를 수 있습니다. **GAT**는 Transformer의 어텐션 메커니즘을 그래프에 적용하여 이웃의 중요도를 **학습**합니다.

### GAT 어텐션 계산

```
노드 v와 이웃 u 사이의 어텐션 계수:

1. Linear projection:
   z_v = W · h_v  (h_v: 현재 노드 특성)

2. 어텐션 점수 계산 (concat + LeakyReLU):
   e_{uv} = LeakyReLU( a^T [z_v || z_u] )
   - a: 학습 가능한 벡터 (2F' 차원)
   - ||: 연결(concatenation)

3. Softmax 정규화 (이웃 집합 N(v)에 대해):
   α_{uv} = softmax_u(e_{uv}) = exp(e_{uv}) / Σ_{k∈N(v)} exp(e_{kv})

4. 가중 합산:
   h_v' = σ( Σ_{u∈N(v)} α_{uv} · z_u )

5. Multi-head attention (K개 head):
   h_v' = ||_{k=1}^{K} σ( Σ_{u∈N(v)} α_{uv}^k · W^k h_u )
```

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class GATLayer(nn.Module):
    def __init__(self, in_features, out_features, n_heads=4, dropout=0.6):
        super().__init__()
        self.n_heads = n_heads
        self.out_features = out_features
        
        # 선형 변환 (헤드별)
        self.W = nn.Linear(in_features, out_features * n_heads, bias=False)
        # 어텐션 파라미터 a = [a_left || a_right]
        self.a_left = nn.Parameter(torch.FloatTensor(n_heads, out_features))
        self.a_right = nn.Parameter(torch.FloatTensor(n_heads, out_features))
        
        self.leaky_relu = nn.LeakyReLU(0.2)
        self.dropout = nn.Dropout(dropout)
        
        nn.init.xavier_uniform_(self.W.weight)
        nn.init.xavier_uniform_(self.a_left.unsqueeze(0))
        nn.init.xavier_uniform_(self.a_right.unsqueeze(0))

    def forward(self, h, edge_index):
        """
        h: (N, in_features)
        edge_index: (2, E) - [source; target]
        """
        N = h.size(0)
        
        # (N, n_heads, out_features)
        Wh = self.W(h).view(N, self.n_heads, self.out_features)
        
        src, dst = edge_index[0], edge_index[1]
        
        # 어텐션 점수: e_{src→dst}
        # (E, n_heads)
        e = (Wh[src] * self.a_left).sum(dim=-1) + \
            (Wh[dst] * self.a_right).sum(dim=-1)
        e = self.leaky_relu(e)
        
        # Softmax (각 destination 노드 기준)
        # Sparse softmax 대신 scatter_softmax 사용 (torch_geometric)
        from torch_geometric.utils import softmax as geo_softmax
        alpha = geo_softmax(e, dst, num_nodes=N)  # (E, n_heads)
        alpha = self.dropout(alpha)
        
        # 가중 합산: (N, n_heads, out_features)
        out = torch.zeros(N, self.n_heads, self.out_features, device=h.device)
        out.scatter_add_(
            0,
            dst.view(-1, 1, 1).expand(-1, self.n_heads, self.out_features),
            alpha.unsqueeze(-1) * Wh[src]
        )
        
        # Multi-head concat → (N, n_heads * out_features)
        out = out.view(N, -1)
        return F.elu(out)
```

GAT의 핵심 장점은 **어떤 이웃이 더 중요한지를 학습**한다는 것입니다. 이는 그래프의 구조가 불규칙하거나, 노이즈가 많은 엣지가 존재할 때 GCN보다 우수한 성능을 발휘합니다.

---

## GNN의 한계와 극복 방안

### 1. Over-Smoothing 문제

레이어 수를 늘릴수록 모든 노드의 표현이 수렴하여 구별 불가능해집니다. 이는 반복 평균화가 각 노드를 전체 그래프의 평균으로 밀어넣기 때문입니다.

```python
# Over-smoothing 시각화 (개념 코드)
import numpy as np

A_hat = normalize_adjacency(A)  # 앞서 정의한 함수

X = np.random.randn(5, 4)  # 초기에는 다양한 표현

for layer in range(20):
    X = A_hat @ X  # 반복 평균화
    # 레이어가 깊어질수록 노드들의 표현이 수렴

# 해결책: JK-Net(Jumping Knowledge), DropEdge, PairNorm
```

**해결책**:
- **Jumping Knowledge (JK-Net)**: 각 레이어의 출력을 모두 모아 최종 표현 생성
- **DropEdge**: 학습 중 랜덤으로 엣지 제거 (Dropout과 유사)
- **PairNorm**: 레이어별 표준화로 수렴 방지

### 2. WL 그래프 동형 테스트 한계

GNN의 표현력은 **Weisfeiler-Lehman(WL) 그래프 동형 테스트**와 동일한 수준으로 상한이 있습니다. 즉, WL 테스트가 구분 못하는 그래프 구조를 GNN도 구분하지 못합니다.

이를 극복하기 위한 방법:
- **GIN (Graph Isomorphism Network)**: Sum 집계 + MLP 사용으로 WL과 동등한 최대 표현력 달성
- **Higher-order GNN**: 노드 쌍, 삼중쌍 등 서브그래프 정보 활용
- **고유값·위치 인코딩**: 그래프 라플라시안의 고유벡터를 노드 특성에 추가

### 3. 대규모 그래프 미니배치 처리

전체 그래프가 GPU 메모리에 들어가지 않을 때:
- **GraphSAGE**: 이웃 샘플링으로 미니배치 학습
- **Cluster-GCN**: 그래프를 클러스터로 분할
- **GraphSAINT**: 중요도 기반 서브그래프 샘플링

---

## 주의사항과 실전 팁

1. **입력 특성 정규화**: 노드 특성을 배치 정규화 또는 레이어 정규화로 전처리하면 학습 안정성이 크게 향상됩니다.

2. **엣지 방향성**: 유방향 그래프에서는 메시지 방향을 명시해야 합니다. 역방향 엣지를 추가하거나 양방향 메시지를 별도 처리하세요.

3. **이종 그래프(Heterogeneous Graph)**: 노드와 엣지 타입이 여러 개인 경우 R-GCN(Relational GCN)이나 HAN(Heterogeneous Attention Network)을 고려하세요.

4. **PyTorch Geometric vs DGL**: PyG는 사용 편의성, DGL은 대규모 그래프 처리 성능에서 강점입니다. 연구용은 PyG, 프로덕션 대규모는 DGL을 추천합니다.

---

## 마치며

GNN은 이제 단순한 연구 주제를 넘어 산업 현장의 핵심 기술로 자리잡고 있습니다. DeepMind의 AlphaFold 2에서 단백질 구조를 예측할 때 사용한 Evoformer의 핵심이 그래프 어텐션 메커니즘이었으며, Google의 반도체 칩 배치 최적화도 GNN을 활용해 인간 전문가 수준의 결과를 달성했습니다. 그래프 구조 데이터를 다루는 모든 도메인에서 GNN은 점점 더 불가결한 도구가 되어가고 있습니다.

## 참고 자료
- [PyTorch Geometric 공식 문서](https://pytorch-geometric.readthedocs.io/)
- [Semi-supervised Classification with Graph Convolutional Networks (Kipf & Welling, 2017)](https://arxiv.org/abs/1609.02907)
- [Graph Attention Networks (Veličković et al., 2018)](https://arxiv.org/abs/1710.10903)
- [Stanford CS224W: Machine Learning with Graphs](https://web.stanford.edu/class/cs224w/)
