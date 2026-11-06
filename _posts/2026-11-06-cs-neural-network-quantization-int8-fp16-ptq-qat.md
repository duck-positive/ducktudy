---
layout: post
title: "신경망 양자화(Neural Network Quantization) 심화: INT8/FP16, PTQ, QAT 완전 정복"
date: 2026-11-06
categories: [cs, computer-science]
tags: [quantization, neural-network, int8, fp16, ptq, qat, model-compression, deep-learning, inference]
---

대형 언어 모델(LLM)의 파라미터 수가 수백억 개를 넘어서면서, 이를 실제 서비스에 배포하는 것이 하나의 공학적 과제가 되었다. GPT-3의 파라미터를 FP32로 저장하면 약 700GB의 메모리가 필요하다. 양자화(Quantization)는 이 문제를 정면으로 해결하는 핵심 기법이다.

## 양자화란 무엇인가

신경망 양자화는 모델의 가중치(weight)와 활성화(activation)를 낮은 비트 수의 수로 표현하는 기법이다. 일반적으로 FP32(32비트 부동소수점)로 훈련된 모델을 INT8(8비트 정수), FP16(16비트 부동소수점), 또는 INT4로 변환한다.

```
FP32: [-3.4e38, 3.4e38] 범위, 23비트 가수부 → 정밀도 높음, 메모리 많음
FP16: [-6.5e4, 6.5e4] 범위, 10비트 가수부 → 중간
INT8: [-128, 127] 범위, 정수 → 빠른 연산, 메모리 적음
INT4: [-8, 7] 범위 → 극단적 압축
```

**선형 양자화(Affine Quantization)** 의 핵심 공식은 다음과 같다.

```
Q(x) = round(x / scale + zero_point)
x_dequant = scale × (Q(x) - zero_point)
```

- **scale**: 실수 범위를 정수 범위로 매핑하는 스케일 팩터
- **zero_point**: 실수 0.0을 나타내는 정수 값 (대칭 양자화에서는 0)
- **quantization error**: `x - x_dequant` = round-off 오차

### 양자화의 두 차원

1. **Per-tensor vs Per-channel**
   - Per-tensor: 레이어 전체에 단일 scale/zero_point → 간단하지만 채널별 분포가 다를 때 오차가 큼
   - Per-channel: 각 출력 채널마다 개별 scale/zero_point → 정확도 높음

2. **Weight-only vs Weight+Activation**
   - Weight-only: 가중치만 양자화, 추론 시 FP16/FP32로 복원 → 메모리 절약, 속도 개선 제한적
   - W8A8 (8-bit weight, 8-bit activation): 완전한 INT8 연산으로 하드웨어 가속 최대 활용

## 왜 양자화가 필요한가

### 메모리 절약

| 형식 | 비트 수 | 70B 파라미터 모델 크기 |
|------|---------|----------------------|
| FP32 | 32비트  | ~280GB               |
| FP16 | 16비트  | ~140GB               |
| INT8 | 8비트   | ~70GB                |
| INT4 | 4비트   | ~35GB                |

Llama 3 70B 모델을 FP32로 서빙하려면 A100 GPU(80GB VRAM) 4장이 필요하지만, INT4 양자화로는 단 1장으로 가능하다.

### 추론 속도 향상

현대 GPU와 NPU에는 INT8/INT4 행렬 곱셈을 위한 전용 하드웨어(NVIDIA Tensor Core의 INT8 모드, Apple Neural Engine 등)가 탑재되어 있다. FP32 대비 2~4배의 처리량(throughput) 향상을 기대할 수 있다.

### 엣지 디바이스 배포

스마트폰, 임베디드 시스템에서 FP32 모델을 실행하는 것은 사실상 불가능하다. 양자화는 모바일 AI 추론의 필수 전제 조건이다.

## 실제 구현 예제

### 예제 1: PyTorch Dynamic Quantization (PTQ)

PTQ(Post-Training Quantization)는 훈련이 완료된 모델에 별도의 재훈련 없이 양자화를 적용하는 방식이다. 소량의 캘리브레이션 데이터로 scale/zero_point를 결정한다.

```python
import torch
import torch.nn as nn
from torch.quantization import quantize_dynamic
import os

class TextTransformerBlock(nn.Module):
    """간단한 Transformer 블록 (양자화 대상)"""
    def __init__(self, d_model=256, n_heads=4, d_ff=512):
        super().__init__()
        self.attention = nn.MultiheadAttention(d_model, n_heads, batch_first=True)
        self.norm1 = nn.LayerNorm(d_model)
        self.ff = nn.Sequential(
            nn.Linear(d_model, d_ff),
            nn.GELU(),
            nn.Linear(d_ff, d_model),
        )
        self.norm2 = nn.LayerNorm(d_model)

    def forward(self, x, attn_mask=None):
        attn_out, _ = self.attention(x, x, x, attn_mask=attn_mask)
        x = self.norm1(x + attn_out)
        ff_out = self.ff(x)
        return self.norm2(x + ff_out)

class SimpleTransformer(nn.Module):
    def __init__(self, vocab_size=30000, d_model=256, n_layers=4):
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, d_model)
        self.layers = nn.ModuleList([
            TextTransformerBlock(d_model) for _ in range(n_layers)
        ])
        self.head = nn.Linear(d_model, vocab_size)

    def forward(self, tokens):
        x = self.embedding(tokens)
        for layer in self.layers:
            x = layer(x)
        return self.head(x)

# FP32 모델 생성
model_fp32 = SimpleTransformer()
model_fp32.eval()

# Dynamic Quantization 적용: Linear 레이어의 가중치를 INT8로 변환
# activation은 FP32로 유지하고, 추론 시 on-the-fly로 처리
model_int8 = quantize_dynamic(
    model_fp32,
    qconfig_spec={nn.Linear, nn.MultiheadAttention},
    dtype=torch.qint8
)

# 크기 비교
def get_model_size_mb(model):
    torch.save(model.state_dict(), "/tmp/_model_tmp.pt")
    size = os.path.getsize("/tmp/_model_tmp.pt") / (1024 * 1024)
    os.remove("/tmp/_model_tmp.pt")
    return size

fp32_size = get_model_size_mb(model_fp32)
int8_size  = get_model_size_mb(model_int8)

print(f"FP32 모델 크기: {fp32_size:.1f} MB")
print(f"INT8 모델 크기: {int8_size:.1f} MB")
print(f"압축률: {fp32_size / int8_size:.2f}x")

# 정확도 비교 (더미 입력)
dummy_input = torch.randint(0, 30000, (1, 64))  # (batch, seq_len)

with torch.no_grad():
    out_fp32 = model_fp32(dummy_input)
    out_int8  = model_int8(dummy_input)

# 최대 로짓 차이
diff = (out_fp32 - out_int8).abs().max().item()
print(f"최대 로짓 차이: {diff:.4f}")
```

### 예제 2: QAT (Quantization-Aware Training)

QAT는 훈련 과정에 가짜 양자화(Fake Quantization) 노드를 삽입하여, 모델이 양자화 오차를 직접 학습하게 한다. PTQ보다 정확도 손실이 적지만 재훈련 비용이 든다.

```python
import torch
import torch.nn as nn
from torch.ao.quantization import (
    get_default_qat_qconfig_mapping,
    prepare_qat,
    convert_fx,
    QConfigMapping
)
from torch.ao.quantization.quantize_fx import prepare_qat_fx

class ConvBNReLUBlock(nn.Module):
    """모바일 환경의 경량 블록: Conv-BN-ReLU 융합 최적화"""
    def __init__(self, in_channels, out_channels, stride=1):
        super().__init__()
        # Conv + BN + ReLU: QAT에서 자동으로 융합됨
        self.conv = nn.Conv2d(in_channels, out_channels, 3,
                               stride=stride, padding=1, bias=False)
        self.bn   = nn.BatchNorm2d(out_channels)
        self.relu = nn.ReLU(inplace=True)

    def forward(self, x):
        return self.relu(self.bn(self.conv(x)))

class MobileStyleNet(nn.Module):
    def __init__(self, num_classes=100):
        super().__init__()
        self.features = nn.Sequential(
            ConvBNReLUBlock(3,  32, stride=2),   # 224→112
            ConvBNReLUBlock(32, 64, stride=2),   # 112→56
            ConvBNReLUBlock(64, 128, stride=2),  # 56→28
            ConvBNReLUBlock(128, 256, stride=2), # 28→14
        )
        self.pool = nn.AdaptiveAvgPool2d(1)
        self.classifier = nn.Linear(256, num_classes)

    def forward(self, x):
        x = self.features(x)
        x = self.pool(x).flatten(1)
        return self.classifier(x)

# --- QAT 워크플로우 ---

device = "cpu"  # GPU 환경에서는 "cuda"
model = MobileStyleNet(num_classes=100).to(device)

# 1. QConfig 설정: fbgemm (x86), qnnpack (ARM/모바일)
#    'x86'은 INT8 Tensor Core를 활용하는 최신 config
qconfig_mapping = get_default_qat_qconfig_mapping("x86")

# 2. FX Graph 모드로 준비 (Conv-BN 자동 융합 포함)
example_input = torch.randn(1, 3, 224, 224).to(device)
model_prepared = prepare_qat_fx(
    model.train(),
    qconfig_mapping,
    example_inputs=(example_input,)
)

print("QAT 준비 완료. Fake Quantization 노드 삽입됨")
print(f"준비된 모델 타입: {type(model_prepared).__name__}")

# 3. QAT 훈련 루프
optimizer = torch.optim.AdamW(model_prepared.parameters(), lr=1e-4)
criterion = nn.CrossEntropyLoss()

model_prepared.train()
for step in range(10):  # 실제로는 수 에폭
    batch_input  = torch.randn(16, 3, 224, 224).to(device)
    batch_labels = torch.randint(0, 100, (16,)).to(device)

    optimizer.zero_grad()
    logits = model_prepared(batch_input)
    loss   = criterion(logits, batch_labels)
    loss.backward()
    optimizer.step()

    if step % 5 == 0:
        print(f"Step {step}: loss = {loss.item():.4f}")

# 4. 모델을 INT8로 변환 (추론 모드)
model_prepared.eval()
model_int8_fx = convert_fx(model_prepared)

print("\nQAT 완료! INT8 추론 모델 준비됨")

# 5. 추론 속도 비교
import time

model_fp32 = MobileStyleNet(num_classes=100).eval()
test_input = torch.randn(32, 3, 224, 224)

N_RUNS = 20

# FP32 추론 시간
start = time.perf_counter()
with torch.no_grad():
    for _ in range(N_RUNS):
        _ = model_fp32(test_input)
fp32_time = (time.perf_counter() - start) / N_RUNS * 1000

# INT8 추론 시간
start = time.perf_counter()
with torch.no_grad():
    for _ in range(N_RUNS):
        _ = model_int8_fx(test_input)
int8_time = (time.perf_counter() - start) / N_RUNS * 1000

print(f"FP32 추론: {fp32_time:.1f}ms/batch")
print(f"INT8 추론: {int8_time:.1f}ms/batch")
print(f"속도 향상: {fp32_time/int8_time:.2f}x")
```

## 고급 양자화 기법

### GPTQ — 대규모 모델을 위한 PTQ

가중치의 각 행(row)을 순차적으로 양자화하면서 발생하는 오차를 나머지 가중치에 보상(Hessian 기반 보정)하는 기법이다. LLM의 INT4 양자화에서 사실상 표준으로 자리 잡았다.

```python
# GPTQ의 핵심 아이디어 (의사 코드)
def gptq_quantize_layer(W, X_calib, bits=4):
    """
    W: 가중치 행렬 [out, in]
    X_calib: 캘리브레이션 입력 [N, in]
    bits: 양자화 비트 수
    """
    # 1. Hessian 계산: H = 2 * X^T X
    H = 2 * X_calib.T @ X_calib
    H += 0.01 * torch.eye(H.shape[0])  # 정규화

    # 2. OBQ(Optimal Brain Quantization) 방식으로
    #    열(column)을 순서대로 양자화
    W_quantized = W.clone()
    H_inv = torch.linalg.inv(H)

    for i in range(W.shape[1]):
        # i번째 열 양자화
        w_q = quantize_column(W_quantized[:, i], bits)
        # 양자화 오차를 나머지 열에 보정
        error = W_quantized[:, i] - w_q
        W_quantized[:, i+1:] -= (error.unsqueeze(1) *
                                  H_inv[i, i+1:].unsqueeze(0) /
                                  H_inv[i, i])
        W_quantized[:, i] = w_q

    return W_quantized
```

### SmoothQuant — Activation Outlier 문제 해결

Transformer 모델의 activation에는 특정 채널에서 극단적으로 큰 값(Outlier)이 나타난다. SmoothQuant는 이 문제를 수학적 등가 변환으로 해결한다.

```
Y = (X·diag(s)^-1) · (diag(s)·W) = X' · W'
```

X(activation)의 outlier를 s로 나누고, W(weight)에 s를 곱한다. 이렇게 하면 activation 범위가 줄어 INT8 양자화가 쉬워진다.

## 주의사항과 실전 팁

### 1. 하드웨어 호환성 확인 필수

INT8 연산은 모든 하드웨어에서 빠르지 않다. NVIDIA Volta 이전 GPU, 일부 모바일 SoC는 INT8 행렬 곱을 지원하지 않아 오히려 느려질 수 있다.

```python
# CUDA INT8 지원 확인
import torch
if torch.cuda.is_available():
    device = torch.cuda.get_device_properties(0)
    # Tensor Core + INT8: Volta (SM 7.0+) 필요
    print(f"Compute Capability: {device.major}.{device.minor}")
    if device.major >= 7:
        print("INT8 Tensor Core 가속 사용 가능")
    else:
        print("INT8 소프트웨어 에뮬레이션만 가능 (느림)")
```

### 2. 민감한 레이어는 양자화 제외

마지막 Linear 레이어(LM Head), 첫 번째 임베딩 레이어, LayerNorm 등은 양자화에 민감하다. 이런 레이어는 FP16/FP32로 유지하는 것이 좋다.

```python
from torch.quantization import QConfig
import torch.quantization

# 특정 레이어 양자화 비활성화
def set_quantization_config(model):
    for name, module in model.named_modules():
        if isinstance(module, nn.LayerNorm):
            # LayerNorm은 양자화 제외
            module.qconfig = None
        elif 'head' in name or 'embedding' in name:
            # 입출력 레이어 양자화 제외
            module.qconfig = None
```

### 3. 캘리브레이션 데이터 품질

PTQ의 캘리브레이션 데이터는 실제 추론 데이터의 분포를 대표해야 한다. 일반적으로 512~1024개 샘플로 충분하지만, 도메인 특화 모델은 도메인 데이터를 사용해야 한다.

### 4. INT4는 까다롭다

INT4는 메모리 효율이 뛰어나지만, 표현 범위가 [-8, 7]에 불과하다. Group Quantization (128개 가중치마다 개별 scale 적용)으로 정확도를 보완하는 것이 일반적이다.

## 양자화 기법 비교표

| 기법 | 재훈련 필요 | 정확도 손실 | 속도 | 적합한 상황 |
|------|------------|------------|------|------------|
| Dynamic PTQ | 불필요 | 낮음~중간 | 빠름 | 간단한 배포 |
| Static PTQ | 불필요(캘리브레이션만) | 중간 | 가장 빠름 | 대부분의 상황 |
| QAT | 필요 | 가장 낮음 | 가장 빠름 | 정확도가 중요한 경우 |
| GPTQ | 불필요(보정만) | 낮음 | INT4로 최대 압축 | 대형 언어 모델 |

신경망 양자화는 더 이상 선택이 아닌 필수다. 엣지 AI, 실시간 추론, 비용 효율적인 클라우드 서빙 모두 양자화를 전제로 설계된다. PTQ로 시작하여 정확도가 부족하면 QAT로, 더 큰 모델에는 GPTQ나 AWQ 같은 최신 기법을 적용하는 것이 현실적인 접근이다.

## 참고 자료
- [TorchAO: PyTorch Native Quantization Library](https://github.com/pytorch/ao)
- [PubGrub Rust Implementation](https://github.com/pubgrub-rs/pubgrub)
