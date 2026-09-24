---
layout: post
title: "Intel SGX와 ARM TrustZone 완전 정복: 하드웨어 기반 신뢰 실행 환경(TEE)의 내부 구조와 보안 원리"
date: 2026-09-24
categories: [cs, computer-science]
tags: [tee, intel-sgx, arm-trustzone, enclave, security, hardware-security, confidential-computing]
---

클라우드 서버, IoT 기기, 스마트폰에서 민감한 데이터를 처리할 때 가장 큰 위협은 **특권 소프트웨어의 침해**다. 루트 권한을 가진 OS, 하이퍼바이저, 심지어 물리 메모리에 접근할 수 있는 공격자도 막아야 한다. 이 문제를 하드웨어 수준에서 해결하는 기술이 **신뢰 실행 환경(Trusted Execution Environment, TEE)**이다. 이 아티클에서는 Intel SGX와 ARM TrustZone의 구조와 동작 원리를 깊이 파고들고, 실제 보안 적용 방법을 코드와 함께 살펴본다.

---

## 1. TEE란 무엇인가?

**신뢰 실행 환경(TEE)**은 메인 프로세서 위에서 동작하는 격리된 실행 환경으로, 기밀성(confidentiality)과 무결성(integrity)을 하드웨어가 직접 보장한다.

TEE의 핵심 보안 목표:
- **기밀성**: TEE 내부 데이터를 외부(OS, 하이퍼바이저, DMA 공격)에서 읽을 수 없다.
- **무결성**: TEE 내부 코드와 데이터가 변조되지 않았음을 보장한다.
- **원격 증명(Remote Attestation)**: 제3자가 TEE의 진정성을 암호학적으로 검증할 수 있다.

### TEE vs 일반 보안 기법 비교

| 기법 | 보호 수준 | 위협 모델 |
|---|---|---|
| OS 프로세스 격리 | 다른 사용자 프로세스 | OS 손상 시 무너짐 |
| 가상화(VM) | 다른 VM | 하이퍼바이저 손상 시 무너짐 |
| HSM(하드웨어 보안 모듈) | 완전한 외부 장치 | 고가, 인터페이스 제한 |
| **TEE(SGX/TrustZone)** | OS·하이퍼바이저까지 | 물리 공격(일부) 제외 |

---

## 2. Intel SGX: 엔클레이브의 수학

### 엔클레이브(Enclave)란?

SGX는 **Enclave Page Cache(EPC)**라는 암호화된 메모리 영역을 CPU가 직접 관리한다. 엔클레이브 코드는 Ring 3(사용자 공간)에서 실행되지만, OS조차 EPC에 접근할 수 없다. CPU 패키지 밖으로 나가는 모든 메모리 버스 트래픽은 **MEE(Memory Encryption Engine)**이 AES-128 CTR 모드로 자동 암호화한다.

### SGX 핵심 명령어

```
ECREATE  : 새 엔클레이브 생성, SECS(SGX Enclave Control Structure) 초기화
EADD     : 4KB 페이지를 엔클레이브에 추가
EEXTEND  : 페이지 내용의 해시를 MRENCLAVE에 누적
EINIT    : 엔클레이브 초기화 완료, 서명 검증
EENTER   : 일반 코드 → 엔클레이브 진입 (EEXIT의 반대)
EEXIT    : 엔클레이브 → 일반 코드 복귀
EREPORT  : 증명 보고서(REPORT) 생성 (로컬 증명)
EGETKEY  : 봉인(sealing)키 획득
```

### SGX 측정값(Measurement)

엔클레이브의 신뢰성은 두 가지 해시로 측정된다:

- **MRENCLAVE**: 엔클레이브 로딩 과정 전체의 SHA-256. 코드 페이지 추가 순서까지 포함한다. "이 엔클레이브가 정확히 어떤 코드인가"를 나타낸다.
- **MRSIGNER**: 엔클레이브 서명자의 공개키 해시. 버전 업데이트 시 같은 서명자가 새 버전에 접근할 수 있도록 한다.

### SGX 원격 증명(Remote Attestation)

```
[애플리케이션]         [Intel Attestation Service]      [검증자]
     |                           |                          |
     |-- EREPORT 생성 ---------->|                          |
     |   (MRENCLAVE, 논스 포함) |                          |
     |                           |-- REPORT 검증            |
     |<-- QUOTE 서명 -----------|   (EPID/DCAP 서명)       |
     |   (Intel ECDSA 서명)     |                          |
     |-- QUOTE 전송 ---------------------------------------->|
     |                           |                          |
     |                           |          <-- 검증 요청 --|
     |                           |-- 공개키·정책 반환 ----->|
     |                           |                          |-- 신뢰 수립
```

### SGX 코드 예시 (C, Intel SGX SDK 스타일)

```c
/* enclave.edl - Enclave Definition Language */
enclave {
    trusted {
        /* 엔클레이브 내부에서 실행되는 함수 */
        public void ecall_seal_secret(
            [in, size=secret_len] const uint8_t *secret,
            size_t secret_len,
            [out, size=sealed_len] uint8_t *sealed_data,
            size_t sealed_len
        );

        public int ecall_unseal_secret(
            [in, size=sealed_len] const uint8_t *sealed_data,
            size_t sealed_len,
            [out, size=plain_len] uint8_t *plain_data,
            size_t plain_len
        );
    };
    untrusted {
        /* 엔클레이브에서 호출할 수 있는 외부 함수 */
        void ocall_print([in, string] const char *str);
    };
};

/* enclave.c - 엔클레이브 내부 코드 */
#include "sgx_tseal.h"
#include "sgx_tkey_exchange.h"

void ecall_seal_secret(const uint8_t *secret, size_t secret_len,
                       uint8_t *sealed_data, size_t sealed_len) {
    /* EGETKEY로 현재 플랫폼 고유 봉인 키 유도 */
    /* 봉인된 데이터는 동일 엔클레이브(MRENCLAVE)에서만 해제 가능 */
    sgx_seal_data(
        0, NULL,                    /* AAD 없음 */
        (uint32_t)secret_len, secret,
        (uint32_t)sealed_len, (sgx_sealed_data_t*)sealed_data
    );
}

int ecall_unseal_secret(const uint8_t *sealed_data, size_t sealed_len,
                        uint8_t *plain_data, size_t plain_len) {
    uint32_t decrypted_len = (uint32_t)plain_len;
    sgx_status_t ret = sgx_unseal_data(
        (const sgx_sealed_data_t*)sealed_data,
        NULL, NULL,         /* AAD 버퍼 */
        plain_data, &decrypted_len
    );
    return (ret == SGX_SUCCESS) ? 0 : -1;
}
```

---

## 3. ARM TrustZone: 두 세계의 분리

### Secure World vs Normal World

ARM TrustZone은 프로세서를 두 세계(world)로 나눈다:

```
┌─────────────────────────────────────────┐
│           ARM 프로세서                  │
│                                         │
│  ┌──────────────┐  ┌──────────────────┐ │
│  │  Secure      │  │  Normal          │ │
│  │  World       │  │  World           │ │
│  │  (신뢰 영역) │  │  (일반 영역)     │ │
│  │              │  │                  │ │
│  │  TEE OS      │  │  Rich OS (Linux, │ │
│  │  (OP-TEE 등) │  │  Android 등)     │ │
│  │              │  │                  │ │
│  │  Trusted     │  │  일반 앱         │ │
│  │  Application │  │                  │ │
│  └──────┬───────┘  └────────┬─────────┘ │
│         │                   │           │
│  ┌──────▼───────────────────▼─────────┐ │
│  │  Monitor Mode (EL3)                │ │
│  │  Secure Monitor (세계 전환 관리)   │ │
│  └────────────────────────────────────┘ │
└─────────────────────────────────────────┘
```

- **NS 비트(Non-Secure bit)**: 시스템 버스에 달린 1비트로, 현재 트랜잭션이 Secure World인지 Normal World인지를 나타낸다. 하드웨어 IP들은 이 비트를 보고 접근을 허용/거부한다.
- **SMC 명령어(Secure Monitor Call)**: Normal World에서 Secure World로 진입하는 유일한 방법. 소프트웨어 인터럽트처럼 동작한다.

### OP-TEE: 오픈소스 TEE 구현

**OP-TEE(Open Portable Trusted Execution Environment)**는 ARM TrustZone 위에서 동작하는 오픈소스 TEE OS다.

```c
/* Trusted Application (TA) 구현 예시 - OP-TEE 스타일 */
#include <tee_internal_api.h>
#include <tee_internal_api_extensions.h>

#define TA_SECURE_STORAGE_CMD_WRITE    0
#define TA_SECURE_STORAGE_CMD_READ     1

/* 안전한 스토리지에 데이터 쓰기 */
static TEE_Result cmd_write_secure(uint32_t param_types,
                                   TEE_Param params[4]) {
    TEE_ObjectHandle obj = TEE_HANDLE_NULL;
    TEE_Result res;
    const char *obj_id = "secure_key";
    uint32_t obj_id_len = strlen(obj_id);
    void *data = params[0].memref.buffer;
    uint32_t data_len = params[0].memref.size;

    /* TEE 안전 스토리지에 오브젝트 생성 */
    res = TEE_CreatePersistentObject(
        TEE_STORAGE_PRIVATE,    /* 이 TA만 접근 가능 */
        obj_id, obj_id_len,
        TEE_DATA_FLAG_ACCESS_WRITE | TEE_DATA_FLAG_OVERWRITE,
        TEE_HANDLE_NULL,        /* 속성 없음 */
        data, data_len,
        &obj
    );

    if (res == TEE_SUCCESS)
        TEE_CloseObject(obj);

    return res;
}

/* 안전한 스토리지에서 데이터 읽기 */
static TEE_Result cmd_read_secure(uint32_t param_types,
                                  TEE_Param params[4]) {
    TEE_ObjectHandle obj = TEE_HANDLE_NULL;
    TEE_Result res;
    const char *obj_id = "secure_key";
    uint32_t obj_id_len = strlen(obj_id);
    uint32_t read_bytes;

    res = TEE_OpenPersistentObject(
        TEE_STORAGE_PRIVATE,
        obj_id, obj_id_len,
        TEE_DATA_FLAG_ACCESS_READ,
        &obj
    );
    if (res != TEE_SUCCESS) return res;

    res = TEE_ReadObjectData(
        obj,
        params[0].memref.buffer,
        params[0].memref.size,
        &read_bytes
    );
    TEE_CloseObject(obj);
    params[1].value.a = read_bytes;
    return res;
}

/* TA Entry Point: REE(Normal World)에서의 명령 처리 */
TEE_Result TA_InvokeCommandEntryPoint(void *sess_ctx,
                                      uint32_t cmd_id,
                                      uint32_t param_types,
                                      TEE_Param params[4]) {
    switch (cmd_id) {
    case TA_SECURE_STORAGE_CMD_WRITE:
        return cmd_write_secure(param_types, params);
    case TA_SECURE_STORAGE_CMD_READ:
        return cmd_read_secure(param_types, params);
    default:
        return TEE_ERROR_NOT_SUPPORTED;
    }
}
```

---

## 4. SGX vs TrustZone 상세 비교

| 항목 | Intel SGX | ARM TrustZone |
|---|---|---|
| **격리 단위** | 엔클레이브(프로세스 내 영역) | Secure World 전체 |
| **위협 모델** | OS·하이퍼바이저까지 | OS까지 (Secure OS 신뢰) |
| **원격 증명** | 강력 (EPID/DCAP) | 제한적 (TA별 구현 필요) |
| **메모리 암호화** | 자동 (MEE, AES-128) | 옵션 (CryptoCell 등) |
| **EPC 크기** | 수십~수백 MB (SGX2) | 플랫폼별 상이 |
| **사용 플랫폼** | 서버, 데스크톱 Intel CPU | 모바일, IoT, 임베디드 |
| **프로그래밍 모델** | ECALL/OCALL 분리 | CA/TA 분리 (TEE API) |
| **알려진 취약점** | Spectre, LVI, Plundervolt | 캐시 타이밍 공격 |

---

## 5. 주의사항과 팁

### 사이드 채널 공격
TEE는 소프트웨어 기반 공격을 막지만, **사이드 채널**은 여전히 취약하다:
- **캐시 타이밍 공격**: 엔클레이브가 캐시 패턴을 통해 비밀키를 유출할 수 있다. 상수 시간(constant-time) 구현이 필수다.
- **Spectre/LVI**: 투기적 실행으로 엔클레이브 메모리 유출 가능. 마이크로코드 업데이트 및 LLVM 기반 레트폴린(retpoline)으로 완화.
- **Plundervolt**: 전압 언더볼팅으로 EPC 암호화를 우회하는 공격 (SGX2에서 수정).

### 코드 설계 원칙
- 엔클레이브 코드는 **최소화** 한다 (TCB 축소 원칙).
- OCALL은 신뢰하지 않는 환경(OS)으로 나가는 것이므로, 반환 값을 항상 검증한다.
- `sgx_read_rand()`과 같은 SGX 제공 API를 사용해 안전한 난수를 생성한다.

### 확장: Confidential Computing
Azure Confidential Computing, Google Confidential VMs, AWS Nitro Enclaves 등 주요 클라우드는 TEE를 서비스로 제공한다. 딥러닝 추론 모델 보호, 의료 데이터 공동 분석, 암호화폐 키 관리 등에 활용한다.

---

## 참고 자료

- [Intel SGX 개발자 가이드 (Intel)](https://www.intel.com/content/www/us/en/developer/tools/software-guard-extensions/overview.html)
- [ARM TrustZone Technology (ARM Developer)](https://developer.arm.com/ip-products/security-ip/trustzone)
- [Preliminary Study of TEE on Heterogeneous Edge Platforms (ResearchGate)](https://www.researchgate.net/publication/328228513_Preliminary_Study_of_Trusted_Execution_Environments_on_Heterogeneous_Edge_Platforms)
- [OP-TEE 공식 문서](https://optee.readthedocs.io/)
