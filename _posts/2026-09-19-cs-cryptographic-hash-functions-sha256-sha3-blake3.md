---
layout: post
title: "암호화 해시 함수 완전 정복: SHA-256, SHA-3, BLAKE3의 내부 설계와 보안 원리"
date: 2026-09-19
categories: [cs, computer-science]
tags: [hash, sha256, sha3, blake3, cryptography, security, keccak]
---

## 개요

암호화 해시 함수(Cryptographic Hash Function)는 임의 길이의 입력을 받아 고정 길이의 출력(다이제스트)을 생성하는 단방향 함수입니다. TLS 인증서 검증, 패스워드 저장, 블록체인 트랜잭션, 파일 무결성 검사 등 현대 보안 인프라 전반에 걸쳐 사용됩니다.

좋은 암호화 해시 함수는 세 가지 보안 속성을 만족해야 합니다.

- **역상 저항성(Pre-image resistance)**: 다이제스트 `h`가 주어질 때 `H(x) = h`를 만족하는 `x`를 찾기 어려워야 합니다.
- **제2역상 저항성(Second pre-image resistance)**: `x`와 `H(x)`가 주어질 때 `H(y) = H(x)`인 `y ≠ x`를 찾기 어려워야 합니다.
- **충돌 저항성(Collision resistance)**: `H(x) = H(y)`인 임의의 쌍 `x ≠ y`를 찾기 어려워야 합니다.

이 글에서는 현재 가장 널리 사용되는 세 가지 해시 함수 — SHA-256, SHA-3(Keccak), BLAKE3 — 의 내부 구조를 깊이 살펴봅니다.

---

## SHA-256: Merkle-Damgård 구조의 완성형

### 왜 필요한가?

MD5와 SHA-1의 충돌 공격이 실용화되면서 더 강력한 해시 함수가 필요해졌습니다. NIST는 2001년 SHA-256을 포함한 SHA-2 패밀리를 발표했습니다. SHA-256은 현재 TLS 인증서, Git 커밋 ID, Bitcoin 블록 해시 등에 광범위하게 사용됩니다.

### 내부 구조: Merkle-Damgård + Davies-Meyer

SHA-256은 **Merkle-Damgård 구성**을 기반으로 합니다. 메시지를 512비트(64바이트) 블록으로 나누고, 각 블록을 압축 함수로 처리하여 해시 상태를 업데이트합니다.

```
초기 상태(IV) → [블록 1 압축] → [블록 2 압축] → ... → [최종 상태] → 다이제스트
```

각 512비트 블록은 **64라운드의 Davies-Meyer 압축 함수**를 거칩니다. 8개의 32비트 레지스터(a, b, c, d, e, f, g, h)가 유지되며, 라운드 상수(k[i])와 메시지 스케줄(W[i])이 사용됩니다.

```python
# SHA-256 압축 함수의 핵심 (Python 의사코드)
import struct

# 8개의 초기 해시 값 (소수의 제곱근의 소수점 이하 32비트)
H = [
    0x6a09e667, 0xbb67ae85, 0x3c6ef372, 0xa54ff53a,
    0x510e527f, 0x9b05688c, 0x1f83d9ab, 0x5be0cd19
]

# 64개의 라운드 상수 (소수의 세제곱근의 소수점 이하 32비트)
K = [
    0x428a2f98, 0x71374491, 0xb5c0fbcf, 0xe9b5dba5,
    # ... 총 64개
]

def rotr(x, n):
    return ((x >> n) | (x << (32 - n))) & 0xFFFFFFFF

def compress(block_512bit, state):
    # 1. 메시지 스케줄 W[0..63] 생성
    W = list(struct.unpack('>16I', block_512bit))
    for i in range(16, 64):
        s0 = rotr(W[i-15], 7) ^ rotr(W[i-15], 18) ^ (W[i-15] >> 3)
        s1 = rotr(W[i-2], 17) ^ rotr(W[i-2], 19) ^ (W[i-2] >> 10)
        W.append((W[i-16] + s0 + W[i-7] + s1) & 0xFFFFFFFF)

    a, b, c, d, e, f, g, h = state

    # 2. 64라운드 처리
    for i in range(64):
        S1    = rotr(e, 6) ^ rotr(e, 11) ^ rotr(e, 25)
        ch    = (e & f) ^ (~e & g)
        temp1 = (h + S1 + ch + K[i] + W[i]) & 0xFFFFFFFF
        S0    = rotr(a, 2) ^ rotr(a, 13) ^ rotr(a, 22)
        maj   = (a & b) ^ (a & c) ^ (b & c)
        temp2 = (S0 + maj) & 0xFFFFFFFF

        h, g, f, e, d, c, b, a = g, f, e, (d + temp1) & 0xFFFFFFFF, c, b, a, (temp1 + temp2) & 0xFFFFFFFF

    # 3. 현재 상태에 덧셈 (Davies-Meyer 피드백)
    return [(s + r) & 0xFFFFFFFF for s, r in zip(state, [a,b,c,d,e,f,g,h])]
```

### 길이 확장 공격(Length Extension Attack)

SHA-256의 중요한 취약점은 **길이 확장 공격**입니다. 메시지 `m`의 다이제스트 `H(m)`을 알고 있다면, `m`의 내용을 모르더라도 `H(m || padding || m')`을 계산할 수 있습니다. 이 때문에 MAC(메시지 인증 코드)에는 SHA-256 직접 사용 대신 **HMAC-SHA256**을 사용해야 합니다.

```python
import hmac, hashlib

# 잘못된 방식: H(secret || message) - 길이 확장 공격에 취약
# 올바른 방식: HMAC
def secure_mac(key: bytes, message: bytes) -> str:
    return hmac.new(key, message, hashlib.sha256).hexdigest()
```

---

## SHA-3: 스펀지 구조의 혁명

### 왜 필요한가?

SHA-2는 현재도 안전하지만, MD5·SHA-1·SHA-2가 모두 Merkle-Damgård 구조를 공유한다는 점이 설계상 단일 실패 지점이 될 수 있습니다. NIST는 2007년부터 2012년까지 SHA-3 공모전을 진행했고, Keccak 알고리즘이 채택되었습니다.

### 스펀지 구조(Sponge Construction)

SHA-3의 핵심은 **스펀지 구조**입니다. 스펀지는 두 단계로 동작합니다.

```
흡수(Absorb) 단계: 입력 메시지를 r비트 블록으로 나누어 내부 상태에 XOR 후 Keccak-f 치환
착출(Squeeze) 단계: 내부 상태에서 출력 비트를 r비트씩 추출
```

내부 상태는 `b = r + c` 비트입니다. SHA3-256은 `r=1088, c=512, b=1600`입니다. 이 1600비트 상태는 **5×5×64 비트의 3차원 배열**로 표현됩니다.

```python
# Keccak-f[1600] 내부 치환의 핵심 단계 (Python 의사코드)
def keccak_f(state_5x5x64):
    """5라운드 함수로 구성된 내부 치환"""
    A = state_5x5x64  # shape: [5][5], 각 원소는 64비트 레인

    for round_idx in range(24):  # 24 라운드
        # θ (세타): 열 패리티 혼합
        C = [A[x][0] ^ A[x][1] ^ A[x][2] ^ A[x][3] ^ A[x][4] for x in range(5)]
        D = [C[(x-1) % 5] ^ rot64(C[(x+1) % 5], 1) for x in range(5)]
        A = [[A[x][y] ^ D[x] for y in range(5)] for x in range(5)]

        # ρ (로): 비트 회전
        # π (파이): 레인 재배치
        # χ (카이): 비선형 치환 (유일한 비선형 단계)
        A_chi = [[A[x][y] ^ ((~A[(x+1)%5][y]) & A[(x+2)%5][y])
                  for y in range(5)] for x in range(5)]

        # ι (이오타): 라운드 상수 XOR
        A_chi[0][0] ^= ROUND_CONSTANTS[round_idx]

    return A_chi
```

### SHA-3 vs SHA-256 비교

| 항목 | SHA-256 | SHA-3-256 |
|------|---------|-----------|
| 구조 | Merkle-Damgård | Sponge |
| 출력 크기 | 256비트 | 256비트 |
| 내부 상태 | 256비트 | 1600비트 |
| 길이 확장 공격 | 취약 | 저항 |
| 성능(SW) | 빠름 | 느림 |
| 성능(HW) | 중간 | 빠름 |

SHA-3는 내부 상태가 커서(1600비트) 길이 확장 공격에 구조적으로 저항합니다.

---

## BLAKE3: 병렬 Merkle 트리 해싱

### 왜 필요한가?

SHA-256과 SHA-3는 순차적 처리를 강요합니다. 현대의 멀티코어 CPU와 SIMD 명령어를 활용하려면 병렬 처리가 가능한 설계가 필요합니다. BLAKE3는 2020년 발표되어 이 문제를 해결합니다.

### Merkle 트리 기반 설계

BLAKE3는 입력을 1KB 청크로 나누고, 각 청크를 독립적으로 해시하여 **Merkle 트리**를 구성합니다.

```
            Root Hash
           /         \
    Hash(L1)       Hash(L2)
    /      \       /      \
Chunk1  Chunk2 Chunk3  Chunk4
```

각 청크는 서로 독립적이므로 멀티스레드로 병렬 처리가 가능합니다. 이것이 BLAKE3가 단일 코어에서는 SHA-256보다 약간 느리지만, 멀티코어에서 선형적으로 빠른 이유입니다.

### ChaCha20 기반 압축 함수

BLAKE3의 압축 함수는 ChaCha20 스트림 암호를 기반으로 한 **BLAKE2의 G 함수**를 계승합니다.

```rust
// BLAKE3 G 함수 (Rust)
fn g(state: &mut [u32; 16], a: usize, b: usize, c: usize, d: usize, x: u32, y: u32) {
    state[a] = state[a].wrapping_add(state[b]).wrapping_add(x);
    state[d] = (state[d] ^ state[a]).rotate_right(16);
    state[c] = state[c].wrapping_add(state[d]);
    state[b] = (state[b] ^ state[c]).rotate_right(12);
    state[a] = state[a].wrapping_add(state[b]).wrapping_add(y);
    state[d] = (state[d] ^ state[a]).rotate_right(8);
    state[c] = state[c].wrapping_add(state[d]);
    state[b] = (state[b] ^ state[c]).rotate_right(7);
}

// Rust에서 BLAKE3 사용 (실용 예제)
fn blake3_hash_file(path: &str) -> Result<String, std::io::Error> {
    use std::io::Read;
    let mut file = std::fs::File::open(path)?;
    let mut hasher = blake3::Hasher::new();
    let mut buffer = [0u8; 65536]; // 64KB 버퍼
    loop {
        let n = file.read(&mut buffer)?;
        if n == 0 { break; }
        hasher.update(&buffer[..n]);
    }
    Ok(hasher.finalize().to_hex().to_string())
}
```

### 성능 비교 (단일 코어, x86-64)

| 알고리즘 | 처리량(approx) | 비고 |
|----------|---------------|------|
| MD5 | ~450 MB/s | 보안 취약, 사용 금지 |
| SHA-256 | ~250 MB/s | 범용 표준 |
| SHA3-256 | ~150 MB/s | 하드웨어 가속 없는 경우 |
| BLAKE3 | ~1000 MB/s | SIMD 최적화 |

---

## 주의사항과 실전 선택 가이드

### 용도별 권장 알고리즘

1. **일반 무결성 검증 (파일, API)**: SHA-256 (검증도 높고 지원 범위 넓음)
2. **패스워드 해싱**: `bcrypt`, `scrypt`, `Argon2` 사용 — 암호화 해시 함수를 직접 사용하지 말 것
3. **MAC**: HMAC-SHA256 또는 HMAC-SHA3 (길이 확장 공격 방지)
4. **대용량 파일/스트림**: BLAKE3 (병렬 처리로 빠른 속도)
5. **FIPS 인증 필요 환경**: SHA-256 또는 SHA-3 (NIST 승인)

### MD5·SHA-1은 절대 사용하지 말 것

MD5는 2004년, SHA-1은 2017년 실용적 충돌 공격이 입증되었습니다. Git도 SHA-1에서 SHA-256으로 전환을 진행 중입니다. 기존 코드에서 MD5/SHA-1을 발견하면 즉시 교체해야 합니다.

```python
# 잘못된 예 (보안 목적에 절대 사용 금지)
import hashlib
hashlib.md5(data).hexdigest()   # ❌
hashlib.sha1(data).hexdigest()  # ❌

# 올바른 예
hashlib.sha256(data).hexdigest()  # ✅ 일반적 용도
hashlib.sha3_256(data).hexdigest()  # ✅ SHA-3 필요 시
```

### 타이밍 공격 방어

해시 비교 시 `==` 대신 `hmac.compare_digest()`를 사용하십시오. 일반 문자열 비교는 첫 불일치 바이트에서 반환하므로 시간 차이로 정보가 누출됩니다.

```python
import hmac
# 타이밍 공격에 안전한 비교
if hmac.compare_digest(expected_hash, computed_hash):
    print("검증 성공")
```

---

## 참고 자료

- [NIST FIPS 180-4 — SHA-2 표준](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.180-4.pdf)
- [NIST FIPS 202 — SHA-3 표준 (Keccak)](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.202.pdf)
- [BLAKE3 공식 명세서](https://github.com/BLAKE3-team/BLAKE3-specs/blob/master/blake3.pdf)
- [BLAKE 해시 함수 — Wikipedia](https://en.wikipedia.org/wiki/BLAKE_(hash_function))
