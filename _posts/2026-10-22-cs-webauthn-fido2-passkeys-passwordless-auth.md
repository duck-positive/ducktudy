---
layout: post
title: "WebAuthn/FIDO2/Passkeys 완전 정복: 패스워드 없는 인증 혁명의 기술적 내부 구조"
date: 2026-10-22
categories: [cs, computer-science]
tags: [WebAuthn, FIDO2, Passkeys, 인증, 보안, 공개키암호화, 패스워드리스, CTAP]
---

"비밀번호를 잊어버리셨나요?" 이 문구는 현대 인터넷의 가장 흔한 고충을 상징합니다. 비밀번호는 피싱, 크리덴셜 스터핑, 데이터 유출 등 수많은 공격의 표적입니다. WebAuthn(Web Authentication)과 FIDO2는 이 문제를 근본적으로 해결하기 위해 설계된 개방형 표준입니다. Apple, Google, Microsoft가 함께 지원하는 Passkeys는 이 기술의 소비자 친화적 구현체입니다. 이 글에서는 이 기술의 암호학적 기반부터 실제 구현까지 깊이 탐구합니다.

## 패스워드 인증의 근본적 문제

전통적인 패스워드 인증은 **공유 비밀(shared secret)** 모델에 기반합니다. 사용자가 서버에 비밀번호를 등록하면, 서버는 이를 해시하여 저장합니다. 로그인 시 사용자가 비밀번호를 입력하면 서버의 저장된 값과 비교합니다.

이 모델의 문제점:
- **피싱 취약**: 가짜 사이트에서 비밀번호를 입력해도 피싱 성공
- **서버 침해**: 해시 데이터베이스가 유출되면 오프라인 크래킹 가능
- **재사용**: 사람들은 여러 사이트에서 같은 비밀번호를 재사용
- **비밀번호 관리**: 수십 개의 강한 비밀번호를 기억하기 불가능

WebAuthn은 **비대칭 키 암호화(공개키 암호화)**를 사용하여 이 모든 문제를 한 번에 해결합니다.

## WebAuthn/FIDO2 아키텍처 이해

### 구성 요소

FIDO2는 두 가지 표준의 결합입니다:

1. **WebAuthn (W3C 표준)**: 브라우저와 웹 애플리케이션 간의 JavaScript API. 릴라잉 파티(서버)가 등록 및 인증을 요청하는 방법을 정의합니다.

2. **CTAP2 (FIDO Alliance 표준)**: Client to Authenticator Protocol. 플랫폼(운영체제/브라우저)과 인증 장치(하드웨어 키, 스마트폰 등) 간의 통신 프로토콜입니다.

```
┌──────────────────────────────────────────────────────────────┐
│  릴라잉 파티 (RP) - 웹 서버                                    │
│  (서비스 도메인: example.com)                                  │
└──────────────────┬───────────────────────────────────────────┘
                   │ HTTPS + WebAuthn API
┌──────────────────▼───────────────────────────────────────────┐
│  클라이언트 (브라우저/OS)                                       │
│  - WebAuthn API 구현                                          │
│  - 사용자 동의 UI 제공                                         │
└──────────────────┬───────────────────────────────────────────┘
                   │ CTAP2 (USB, NFC, BLE, 내부 플랫폼)
┌──────────────────▼───────────────────────────────────────────┐
│  인증자 (Authenticator)                                        │
│  - 플랫폼 내장형: Face ID, 지문 센서, Windows Hello            │
│  - 로밍형: YubiKey, Titan Key 등 하드웨어 보안 키              │
└──────────────────────────────────────────────────────────────┘
```

### 핵심 암호학: 공개키/개인키 쌍

WebAuthn의 암호학적 토대는 **비대칭 키 암호화**입니다. 각 등록마다 인증자는 새로운 키 쌍을 생성합니다:

- **개인키(Private Key)**: 인증자 내부에만 저장. 절대 외부로 나가지 않음.
- **공개키(Public Key)**: 릴라잉 파티(서버)에 등록. 공개되어도 안전.

인증 시 인증자는 개인키로 챌린지에 서명하고, 서버는 저장된 공개키로 서명을 검증합니다. 피싱 사이트는 서버의 공개키에 대응하는 개인키가 없으므로 유효한 서명을 만들 수 없습니다.

## 등록(Registration) 플로우 상세 분석

```
사용자         브라우저               서버 (RP)
  │                │                    │
  │  "등록 버튼"   │                    │
  │────────────────►                    │
  │                │   등록 요청        │
  │                │───────────────────►│
  │                │   challenge +      │
  │                │   rp.id +          │
  │                │   userInfo         │
  │                │◄───────────────────│
  │                │                    │
  │  생체인식/PIN   │                    │
  │◄───────────────│                    │
  │  인증 완료     │                    │
  │────────────────►                    │
  │                │ (인증자 내부)       │
  │                │ 1. 키쌍 생성       │
  │                │ 2. 개인키 저장     │
  │                │ 3. attestation 생성│
  │                │                    │
  │                │  credential +      │
  │                │  publicKey +       │
  │                │  attestation       │
  │                │───────────────────►│
  │                │                    │ 검증 후
  │                │                    │ 공개키 저장
```

### 등록 시 주요 데이터

**서버 → 클라이언트 (PublicKeyCredentialCreationOptions)**:
```json
{
  "challenge": "base64url_encoded_random_32_bytes",
  "rp": {
    "id": "example.com",
    "name": "Example Service"
  },
  "user": {
    "id": "base64url_user_id",
    "name": "user@example.com",
    "displayName": "Alice"
  },
  "pubKeyCredParams": [
    { "alg": -7, "type": "public-key" },   // ECDSA P-256
    { "alg": -257, "type": "public-key" }  // RSA-PKCS1v15
  ],
  "authenticatorSelection": {
    "userVerification": "required",        // 생체/PIN 필수
    "residentKey": "required"             // 패스키: 인증자에 저장
  },
  "timeout": 60000,
  "attestation": "direct"
}
```

**클라이언트 → 서버 (AuthenticatorAttestationResponse)**:
```json
{
  "id": "credential_id_base64url",
  "rawId": "...",
  "type": "public-key",
  "response": {
    "clientDataJSON": "base64url({type, challenge, origin})",
    "attestationObject": "base64url(CBOR({fmt, attStmt, authData}))"
  }
}
```

`attestationObject`의 `authData`에는 공개키, credential ID, AAGUID(장치 모델 식별자), 서명 횟수(counter) 등이 포함됩니다.

## 인증(Authentication) 플로우 상세 분석

### 서버 검증 단계

서버는 다음 단계를 수행합니다:

1. **origin 검증**: `clientDataJSON.origin`이 `https://example.com`과 일치하는지 확인. 이것이 피싱 방지의 핵심입니다.
2. **RP ID 해시 검증**: `authData`의 앞 32바이트가 `SHA-256("example.com")`과 일치하는지 확인.
3. **사용자 인증 플래그**: `authData.flags.UV` (User Verified) 비트가 설정되어 있는지 확인.
4. **챌린지 검증**: `clientDataJSON.challenge`가 서버가 발급한 챌린지와 일치하는지 확인. 재전송 공격 방지.
5. **서명 카운터**: 새 카운터 값이 저장된 값보다 큰지 확인. 클론 탐지.
6. **서명 검증**: 저장된 공개키로 서명 검증.

## 실제 구현: Node.js + @simplewebauthn/server

```typescript
// server.ts - SimpleWebAuthn 라이브러리를 사용한 WebAuthn 서버
import { generateRegistrationOptions, verifyRegistrationResponse,
         generateAuthenticationOptions, verifyAuthenticationResponse } from '@simplewebauthn/server';
import type { RegistrationResponseJSON, AuthenticationResponseJSON } from '@simplewebauthn/types';

const RP_ID = 'example.com';
const RP_NAME = 'Example Service';
const ORIGIN = 'https://example.com';

// 인메모리 저장소 (프로덕션에서는 DB 사용)
const userCredentials = new Map<string, any[]>();
const challengeStore = new Map<string, string>();

// 1. 등록 옵션 생성
async function getRegistrationOptions(userId: string, username: string) {
  const options = await generateRegistrationOptions({
    rpName: RP_NAME,
    rpID: RP_ID,
    userID: Buffer.from(userId),
    userName: username,
    // 기존 크리덴셜 제외 (같은 장치 중복 등록 방지)
    excludeCredentials: (userCredentials.get(userId) || []).map(cred => ({
      id: cred.credentialID,
      type: 'public-key',
    })),
    authenticatorSelection: {
      // Passkey: 인증자에 상주 크리덴셜 요구
      residentKey: 'required',
      userVerification: 'required',
    },
    // ES256 (ECDSA P-256) 우선, RSA PKCS#1 v1.5 폴백
    supportedAlgorithmIDs: [-7, -257],
  });

  // 챌린지 저장 (세션/캐시에 저장, 검증 후 반드시 삭제)
  challengeStore.set(userId, options.challenge);

  return options;
}

// 2. 등록 응답 검증
async function verifyRegistration(userId: string, response: RegistrationResponseJSON) {
  const expectedChallenge = challengeStore.get(userId);
  if (!expectedChallenge) throw new Error('No challenge found');

  const verification = await verifyRegistrationResponse({
    response,
    expectedChallenge,
    expectedOrigin: ORIGIN,
    expectedRPID: RP_ID,
    requireUserVerification: true,
  });

  if (!verification.verified || !verification.registrationInfo) {
    throw new Error('Registration verification failed');
  }

  const { credentialPublicKey, credentialID, counter, credentialDeviceType } =
    verification.registrationInfo;

  // 공개키와 크리덴셜 정보를 DB에 저장
  const credentials = userCredentials.get(userId) || [];
  credentials.push({
    credentialID,
    credentialPublicKey,  // Buffer (COSE 형식 공개키)
    counter,
    credentialDeviceType, // 'singleDevice' | 'multiDevice' (Passkey)
    transports: response.response.transports,
  });
  userCredentials.set(userId, credentials);

  // 챌린지 삭제 (재사용 방지)
  challengeStore.delete(userId);

  return { verified: true, credentialDeviceType };
}

// 3. 인증 옵션 생성
async function getAuthenticationOptions(userId?: string) {
  const allowCredentials = userId
    ? (userCredentials.get(userId) || []).map(cred => ({
        id: cred.credentialID,
        type: 'public-key' as const,
        transports: cred.transports,
      }))
    : []; // 패스키: allowCredentials 없으면 인증자가 자체 검색

  const options = await generateAuthenticationOptions({
    rpID: RP_ID,
    allowCredentials,
    userVerification: 'required',
  });

  if (userId) challengeStore.set(userId, options.challenge);
  else challengeStore.set('anon_' + options.challenge, options.challenge);

  return options;
}

// 4. 인증 응답 검증
async function verifyAuthentication(
  userId: string,
  response: AuthenticationResponseJSON
) {
  const credentials = userCredentials.get(userId) || [];
  const credentialIDBuffer = Buffer.from(response.id, 'base64url');

  // 크리덴셜 ID로 저장된 공개키 찾기
  const storedCredential = credentials.find(
    cred => cred.credentialID.equals(credentialIDBuffer)
  );
  if (!storedCredential) throw new Error('Credential not found');

  const expectedChallenge = challengeStore.get(userId);
  if (!expectedChallenge) throw new Error('No challenge found');

  const verification = await verifyAuthenticationResponse({
    response,
    expectedChallenge,
    expectedOrigin: ORIGIN,
    expectedRPID: RP_ID,
    authenticator: storedCredential,
    requireUserVerification: true,
  });

  if (!verification.verified) throw new Error('Authentication failed');

  // 서명 카운터 업데이트 (클론 탐지)
  // authenticationInfo.newCounter > storedCredential.counter 이어야 함
  storedCredential.counter = verification.authenticationInfo.newCounter;

  challengeStore.delete(userId);

  return { verified: true };
}
```

### 브라우저 JavaScript 구현

```javascript
// client.js - 브라우저에서 WebAuthn API 사용

// 등록
async function registerPasskey() {
  // 서버에서 옵션 가져오기
  const response = await fetch('/webauthn/register/options', { method: 'POST' });
  const options = await response.json();

  // challenge, user.id 등을 ArrayBuffer로 디코딩
  options.challenge = base64urlToArrayBuffer(options.challenge);
  options.user.id = base64urlToArrayBuffer(options.user.id);
  if (options.excludeCredentials) {
    options.excludeCredentials = options.excludeCredentials.map(cred => ({
      ...cred,
      id: base64urlToArrayBuffer(cred.id),
    }));
  }

  // 인증자에게 키 생성 요청
  // 이 시점에 브라우저가 생체인식 UI 표시
  const credential = await navigator.credentials.create({ publicKey: options });

  // 응답을 서버로 전송
  const credentialJSON = {
    id: credential.id,
    rawId: arrayBufferToBase64url(credential.rawId),
    type: credential.type,
    response: {
      clientDataJSON: arrayBufferToBase64url(credential.response.clientDataJSON),
      attestationObject: arrayBufferToBase64url(credential.response.attestationObject),
      transports: credential.response.getTransports?.() ?? [],
    },
  };

  const verifyResponse = await fetch('/webauthn/register/verify', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(credentialJSON),
  });

  return verifyResponse.json();
}

// 인증
async function authenticateWithPasskey() {
  const optResponse = await fetch('/webauthn/auth/options', { method: 'POST' });
  const options = await optResponse.json();

  options.challenge = base64urlToArrayBuffer(options.challenge);
  if (options.allowCredentials) {
    options.allowCredentials = options.allowCredentials.map(cred => ({
      ...cred,
      id: base64urlToArrayBuffer(cred.id),
    }));
  }

  // 저장된 패스키 목록을 표시하고 사용자가 선택
  const assertion = await navigator.credentials.get({ publicKey: options });

  const assertionJSON = {
    id: assertion.id,
    rawId: arrayBufferToBase64url(assertion.rawId),
    type: assertion.type,
    response: {
      clientDataJSON: arrayBufferToBase64url(assertion.response.clientDataJSON),
      authenticatorData: arrayBufferToBase64url(assertion.response.authenticatorData),
      signature: arrayBufferToBase64url(assertion.response.signature),
      userHandle: assertion.response.userHandle
        ? arrayBufferToBase64url(assertion.response.userHandle) : null,
    },
  };

  const verifyResponse = await fetch('/webauthn/auth/verify', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(assertionJSON),
  });

  return verifyResponse.json();
}

// 헬퍼 함수
function base64urlToArrayBuffer(base64url) {
  const base64 = base64url.replace(/-/g, '+').replace(/_/g, '/');
  const binary = atob(base64);
  const buffer = new ArrayBuffer(binary.length);
  const bytes = new Uint8Array(buffer);
  for (let i = 0; i < binary.length; i++) bytes[i] = binary.charCodeAt(i);
  return buffer;
}

function arrayBufferToBase64url(buffer) {
  const bytes = new Uint8Array(buffer);
  let binary = '';
  for (const byte of bytes) binary += String.fromCharCode(byte);
  return btoa(binary).replace(/\+/g, '-').replace(/\//g, '_').replace(/=/g, '');
}
```

## Passkeys: 다중 기기 크리덴셜

WebAuthn의 초기 버전은 각 기기에 별도로 등록해야 했습니다. **Passkeys**는 클라우드 동기화를 통해 이 문제를 해결합니다.

- **Apple Passkeys**: iCloud 키체인을 통해 Apple 기기 간 동기화. AES-256 + end-to-end 암호화.
- **Google Password Manager Passkeys**: Google 계정 통해 Android/Chrome 동기화.
- **1Password, Dashlane 등 비밀번호 관리자**: 크로스 플랫폼 동기화 지원.

기술적으로 Passkeys는 `credentialDeviceType: 'multiDevice'`인 크리덴셜입니다. 개인키는 암호화된 상태로 클라우드에 저장되고, 각 기기에서 복호화되어 사용됩니다.

## 주의사항과 보안 고려사항

### RP ID 바인딩: 피싱 방지의 핵심

WebAuthn에서 크리덴셜은 특정 `rpID`(도메인)에 바인딩됩니다. `example.com`에 등록된 크리덴셜은 `evil-example.com`에서 사용할 수 없습니다. 인증자가 서명 시 `rpID`의 SHA-256 해시를 `authData`에 포함하고, 서버가 이를 검증합니다.

이것이 **피싱에 완전히 면역인 이유**입니다. 공격자가 `evil-example.com`을 만들어도, 사용자의 인증자는 올바른 `example.com`의 크리덴셜을 제공하지 않습니다.

### 서명 카운터와 클론 탐지

각 인증 시 서명 카운터(32비트 정수)가 1씩 증가합니다. 서버는 새 카운터 값이 이전 값보다 큰지 검증합니다. 만약 카운터가 감소하거나 같다면, 인증자가 클론되었을 가능성을 의미합니다.

단, 소프트웨어 기반 Passkeys(클라우드 동기화)는 카운터를 0으로 고정하거나 신뢰성 있게 관리하기 어려울 수 있습니다. 이 경우 카운터 검증을 완화하는 것이 권장됩니다.

### Attestation 처리

Attestation은 인증자 제조사가 "이 장치는 진짜 우리 장치"임을 증명하는 메커니즘입니다. 기업 환경에서 특정 브랜드의 하드웨어 키만 허용할 때 유용합니다. 소비자 서비스에서는 보통 `"attestation": "none"`으로 설정하여 프라이버시를 보호합니다.

WebAuthn과 Passkeys는 인터넷 인증의 패러다임을 바꾸고 있습니다. 피싱, 크리덴셜 스터핑, 비밀번호 피로를 모두 해결하는 이 기술은 이미 Google, Apple, Microsoft, GitHub, Dropbox 등 주요 서비스에서 지원하고 있습니다. 개발자로서 이 기술을 이해하고 서비스에 도입하는 것은 이제 선택이 아닌 필수가 되어가고 있습니다.

## 참고 자료
- [W3C Web Authentication Level 3 Specification](https://www.w3.org/TR/webauthn-3/)
- [FIDO Alliance FIDO2 Overview](https://fidoalliance.org/fido2/)
- [passkeys.dev - 개발자 가이드](https://passkeys.dev/docs/intro/what-are-passkeys/)
- [SimpleWebAuthn 라이브러리 문서](https://simplewebauthn.dev/docs/)
