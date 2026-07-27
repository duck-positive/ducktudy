---
layout: post
title: "OAuth 2.0와 JWT 완전 정복: 현대 웹 인증·인가 표준의 모든 것"
date: 2026-07-27
categories: [cs, computer-science]
tags: [oauth2, jwt, 인증, 인가, 보안, 웹표준, oidc, pkce]
---

카카오 로그인, 구글 로그인, GitHub 소셜 로그인 — 이 모든 것의 기반이 되는 표준이 OAuth 2.0이다. 그리고 현대 API 서버에서 인증 토큰으로 가장 많이 쓰이는 형식이 JWT(JSON Web Token)다. 둘은 함께 쓰이는 경우가 많아 혼동되기 쉽지만 본질적으로 다른 개념이다. 이 글에서는 두 표준의 내부 동작 원리부터 올바른 구현 방법까지 깊이 있게 탐구한다.

## 개념: OAuth 2.0란 무엇인가

### 인증(Authentication)과 인가(Authorization)의 차이

먼저 핵심 용어를 정리해야 한다:

- **인증(Authentication)**: "당신이 누구인가?" — 신원 확인 (로그인)
- **인가(Authorization)**: "당신에게 무엇이 허용되는가?" — 권한 확인 (접근 제어)

**OAuth 2.0은 인가(Authorization) 프레임워크다**. "사용자를 대신해서 특정 리소스에 접근할 권한을 제3자 애플리케이션에 위임한다"는 것이 핵심이다.

### OAuth 2.0이 해결하는 문제

소셜 로그인이 없던 시절, 사용자가 서드파티 앱에 구글 캘린더 접근을 허용하려면 구글 계정 비밀번호를 앱에 직접 입력해야 했다. 이 방식은:

- 앱이 해킹되면 구글 비밀번호가 노출
- 앱이 비밀번호로 계정 전체에 접근 가능 (필요 이상의 권한)
- 권한 취소가 불가능 (비밀번호 변경 외에)

OAuth 2.0은 비밀번호를 노출하지 않고, 범위가 제한된 토큰으로 특정 리소스에만 접근을 위임한다.

### OAuth 2.0 주요 역할

```
                    리소스 소유자(Resource Owner)
                         │ 사용자
                         │
              ┌──────────▼──────────┐
              │  클라이언트(Client)   │← 여기가 우리가 만드는 앱
              │  (서드파티 앱)        │
              └──────┬──────────────┘
                     │                      
              ┌──────▼──────────────┐      ┌────────────────────┐
              │  권한 서버           │      │  리소스 서버         │
              │  (Authorization     │      │  (Resource Server) │
              │   Server)           │      │  보호된 API         │
              │  예: accounts.      │      │  예: gmail.google   │
              │      google.com     │      │      apis.com       │
              └─────────────────────┘      └────────────────────┘
```

### 4가지 Grant Type

OAuth 2.0은 사용 시나리오에 따라 4가지 인가 방식을 정의한다:

| Grant Type | 사용 시나리오 | 특징 |
|------------|---------------|------|
| Authorization Code | 웹앱, 모바일앱 | 가장 안전, PKCE와 함께 사용 |
| Implicit | (Deprecated) | 토큰이 URL에 노출되어 위험 |
| Client Credentials | 서버 간 M2M | 사용자 없이 앱 자체 인증 |
| Resource Owner Password | 레거시 | 비밀번호를 앱에 직접 전달 (비권장) |

## 왜 필요한가: Authorization Code Flow 상세 분석

현재 가장 권장되는 방식인 **Authorization Code + PKCE** 흐름을 단계별로 살펴본다.

```
사용자          클라이언트(앱)        권한 서버        리소스 서버
  │                  │                   │               │
  │─ "구글로 로그인" ─▶│                   │               │
  │                  │─ 1. 인가 요청 ─────▶│               │
  │                  │    (code_challenge) │               │
  │◀─────────────────│                   │               │
  │ 구글 로그인 페이지 │                   │               │
  │─ 2. 로그인 완료 ──────────────────────▶│               │
  │                  │                   │               │
  │◀────────────────────────────────────│               │
  │ 3. 리다이렉트 + Authorization Code    │               │
  │─ 4. Code 전달 ───▶│                   │               │
  │                  │─ 5. Code + code_verifier ─────────▶│
  │                  │◀─ 6. Access Token + Refresh Token ─│
  │                  │─ 7. API 요청 (Access Token) ───────────────▶│
  │                  │◀─ 8. 보호된 리소스 ──────────────────────────│
```

### PKCE (Proof Key for Code Exchange)

모바일 앱은 Client Secret을 안전하게 보관하기 어렵다 (앱 리버스 엔지니어링으로 추출 가능). PKCE는 이 문제를 해결한다:

```
1. 앱이 랜덤 code_verifier 생성 (43~128자)
   code_verifier = "dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk"

2. code_challenge = BASE64URL(SHA256(code_verifier))
   code_challenge = "E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM"

3. 인가 요청에 code_challenge 포함
   → 권한 서버가 저장

4. 토큰 요청 시 code_verifier 전송
   → 권한 서버가 SHA256(code_verifier) == code_challenge 검증

공격자가 Authorization Code를 탈취해도
code_verifier 없이는 토큰 교환 불가!
```

## 실제 구현 예제

### 예제 1: JWT 완전 구현 (Python)

```python
import base64
import hashlib
import hmac
import json
import time
import secrets
from typing import Any

def base64url_encode(data: bytes) -> str:
    """URL-safe Base64 인코딩 (패딩 제거)"""
    return base64.urlsafe_b64encode(data).rstrip(b'=').decode()

def base64url_decode(data: str) -> bytes:
    """URL-safe Base64 디코딩 (패딩 복원)"""
    padding = 4 - len(data) % 4
    if padding != 4:
        data += '=' * padding
    return base64.urlsafe_b64decode(data)

class JWT:
    """JWT 수동 구현 (교육 목적 — 실제로는 python-jose, PyJWT 사용)"""
    
    @staticmethod
    def encode(payload: dict, secret: str, algorithm: str = 'HS256') -> str:
        """JWT 생성"""
        # 1. Header
        header = {
            'alg': algorithm,
            'typ': 'JWT'
        }
        header_encoded = base64url_encode(json.dumps(header, separators=(',', ':')).encode())
        
        # 2. Payload
        payload_encoded = base64url_encode(json.dumps(payload, separators=(',', ':')).encode())
        
        # 3. Signature (HMAC-SHA256)
        signing_input = f"{header_encoded}.{payload_encoded}"
        signature = hmac.new(
            secret.encode(),
            signing_input.encode(),
            hashlib.sha256
        ).digest()
        signature_encoded = base64url_encode(signature)
        
        return f"{signing_input}.{signature_encoded}"
    
    @staticmethod
    def decode(token: str, secret: str, verify_exp: bool = True) -> dict:
        """JWT 검증 및 디코딩"""
        parts = token.split('.')
        if len(parts) != 3:
            raise ValueError("유효하지 않은 JWT 형식")
        
        header_encoded, payload_encoded, signature_encoded = parts
        
        # 1. 서명 검증 (Timing-safe 비교 필수!)
        signing_input = f"{header_encoded}.{payload_encoded}"
        expected_sig = hmac.new(
            secret.encode(),
            signing_input.encode(),
            hashlib.sha256
        ).digest()
        expected_sig_encoded = base64url_encode(expected_sig)
        
        # hmac.compare_digest: 타이밍 공격 방지
        if not hmac.compare_digest(signature_encoded, expected_sig_encoded):
            raise ValueError("서명 검증 실패: 토큰이 변조되었습니다")
        
        # 2. Payload 디코딩
        payload = json.loads(base64url_decode(payload_encoded))
        
        # 3. 만료 시간 검증
        if verify_exp and 'exp' in payload:
            if time.time() > payload['exp']:
                raise ValueError(f"토큰이 만료되었습니다 (만료: {payload['exp']})")
        
        # 4. Not Before 검증
        if 'nbf' in payload and time.time() < payload['nbf']:
            raise ValueError("토큰이 아직 활성화되지 않았습니다")
        
        return payload

# 실제 사용 예시
SECRET_KEY = secrets.token_hex(32)  # 실제로는 환경 변수에서 로드

# Access Token 생성 (짧은 만료)
access_payload = {
    'sub': 'user-123',          # Subject: 사용자 ID
    'iss': 'auth.example.com',  # Issuer: 발급자
    'aud': 'api.example.com',   # Audience: 수신자
    'iat': int(time.time()),     # Issued At
    'exp': int(time.time()) + 900,  # Expiry: 15분
    'scope': 'read:profile write:posts',
    'jti': secrets.token_hex(16)    # JWT ID: 재사용 방지용 고유 ID
}

token = JWT.encode(access_payload, SECRET_KEY)
print(f"Access Token:\n{token}\n")

# JWT 구조 확인
parts = token.split('.')
print(f"Header:    {json.loads(base64url_decode(parts[0]))}")
print(f"Payload:   {json.loads(base64url_decode(parts[1]))}")
print(f"Signature: {parts[2][:20]}...")

# 검증
try:
    decoded = JWT.decode(token, SECRET_KEY)
    print(f"\n검증 성공: sub={decoded['sub']}, scope={decoded['scope']}")
except ValueError as e:
    print(f"검증 실패: {e}")

# 변조 시도
tampered = token[:-5] + "AAAAA"
try:
    JWT.decode(tampered, SECRET_KEY)
except ValueError as e:
    print(f"\n변조 감지: {e}")
```

### 예제 2: OAuth 2.0 Authorization Code + PKCE 서버 구현 (Go)

```go
package main

import (
    "crypto/rand"
    "crypto/sha256"
    "encoding/base64"
    "encoding/json"
    "fmt"
    "net/http"
    "sync"
    "time"
)

// PKCE 코드 검증자/챌린지 생성
type PKCEParams struct {
    CodeVerifier  string
    CodeChallenge string
}

func generatePKCE() (*PKCEParams, error) {
    // code_verifier: 43-128자의 랜덤 문자열
    b := make([]byte, 32)
    if _, err := rand.Read(b); err != nil {
        return nil, err
    }
    verifier := base64.RawURLEncoding.EncodeToString(b)

    // code_challenge = BASE64URL(SHA256(code_verifier))
    h := sha256.Sum256([]byte(verifier))
    challenge := base64.RawURLEncoding.EncodeToString(h[:])

    return &PKCEParams{
        CodeVerifier:  verifier,
        CodeChallenge: challenge,
    }, nil
}

// Authorization Code 저장소 (실제로는 Redis 등 사용)
type AuthCodeStore struct {
    mu    sync.Mutex
    codes map[string]*AuthCode
}

type AuthCode struct {
    Code          string
    ClientID      string
    UserID        string
    Scope         string
    CodeChallenge string
    ExpiresAt     time.Time
}

func NewAuthCodeStore() *AuthCodeStore {
    return &AuthCodeStore{codes: make(map[string]*AuthCode)}
}

func (s *AuthCodeStore) Store(code *AuthCode) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.codes[code.Code] = code
}

func (s *AuthCodeStore) Consume(code string) (*AuthCode, bool) {
    s.mu.Lock()
    defer s.mu.Unlock()
    ac, ok := s.codes[code]
    if !ok || time.Now().After(ac.ExpiresAt) {
        delete(s.codes, code)
        return nil, false
    }
    delete(s.codes, code) // 1회용: 사용 후 즉시 삭제
    return ac, true
}

// Token Response
type TokenResponse struct {
    AccessToken  string `json:"access_token"`
    TokenType    string `json:"token_type"`
    ExpiresIn    int    `json:"expires_in"`
    RefreshToken string `json:"refresh_token,omitempty"`
    Scope        string `json:"scope"`
}

// 토큰 엔드포인트 핸들러
func tokenHandler(store *AuthCodeStore) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        if r.Method != http.MethodPost {
            http.Error(w, "Method Not Allowed", http.StatusMethodNotAllowed)
            return
        }

        if err := r.ParseForm(); err != nil {
            http.Error(w, "Bad Request", http.StatusBadRequest)
            return
        }

        grantType := r.FormValue("grant_type")
        if grantType != "authorization_code" {
            http.Error(w, `{"error":"unsupported_grant_type"}`, http.StatusBadRequest)
            return
        }

        code := r.FormValue("code")
        codeVerifier := r.FormValue("code_verifier")

        // Authorization Code 조회 및 소비
        authCode, ok := store.Consume(code)
        if !ok {
            w.WriteHeader(http.StatusBadRequest)
            json.NewEncoder(w).Encode(map[string]string{
                "error": "invalid_grant",
                "error_description": "Authorization code is invalid or expired",
            })
            return
        }

        // PKCE 검증
        h := sha256.Sum256([]byte(codeVerifier))
        challenge := base64.RawURLEncoding.EncodeToString(h[:])
        if challenge != authCode.CodeChallenge {
            w.WriteHeader(http.StatusBadRequest)
            json.NewEncoder(w).Encode(map[string]string{
                "error": "invalid_grant",
                "error_description": "PKCE verification failed",
            })
            return
        }

        // 액세스 토큰 발급 (실제로는 JWT 서명 포함)
        b := make([]byte, 32)
        rand.Read(b)
        accessToken := base64.RawURLEncoding.EncodeToString(b)
        rand.Read(b)
        refreshToken := base64.RawURLEncoding.EncodeToString(b)

        w.Header().Set("Content-Type", "application/json")
        w.Header().Set("Cache-Control", "no-store") // 토큰 캐싱 금지
        json.NewEncoder(w).Encode(TokenResponse{
            AccessToken:  accessToken,
            TokenType:    "Bearer",
            ExpiresIn:    900, // 15분
            RefreshToken: refreshToken,
            Scope:        authCode.Scope,
        })

        fmt.Printf("토큰 발급: user=%s, scope=%s\n", authCode.UserID, authCode.Scope)
    }
}

func main() {
    store := NewAuthCodeStore()

    // 테스트용 Authorization Code 미리 저장
    store.Store(&AuthCode{
        Code:          "test_auth_code_12345",
        ClientID:      "my-app",
        UserID:        "user-456",
        Scope:         "read:profile",
        CodeChallenge: "E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM",
        ExpiresAt:     time.Now().Add(10 * time.Minute),
    })

    http.HandleFunc("/token", tokenHandler(store))

    fmt.Println("OAuth 서버 시작: http://localhost:8080")
    http.ListenAndServe(":8080", nil)
}
```

## JWT 클레임 심화

### 등록된 클레임 (RFC 7519)

```json
{
  "iss": "https://auth.example.com",     // Issuer: 토큰 발급자
  "sub": "user-123",                     // Subject: 토큰 주체 (보통 사용자 ID)
  "aud": ["api.example.com"],            // Audience: 토큰 수신자 (배열 가능)
  "exp": 1753615200,                     // Expiration Time: 만료 시각 (Unix timestamp)
  "nbf": 1753611600,                     // Not Before: 유효 시작 시각
  "iat": 1753611600,                     // Issued At: 발급 시각
  "jti": "a1b2c3d4e5f6"                 // JWT ID: 고유 식별자 (재사용 방지)
}
```

### 비공개 클레임

`iss`, `sub` 등 표준 클레임 외에 애플리케이션 전용 클레임을 추가할 수 있다:

```json
{
  "sub": "user-123",
  "exp": 1753615200,
  "roles": ["admin", "editor"],
  "tenant_id": "org-789",
  "plan": "enterprise"
}
```

**주의**: JWT는 서버 측에서 검증만 하고 저장하지 않는다. 누구나 Base64로 디코딩해서 내용을 볼 수 있으므로 **민감한 정보(비밀번호, 신용카드 번호 등)를 Payload에 넣어선 안 된다**.

## 주의사항과 실전 팁

### 1. "none" 알고리즘 공격

일부 초기 JWT 라이브러리는 `alg: none`을 허용했다. 공격자가 헤더를 `{"alg":"none","typ":"JWT"}`로 변조하면 서명 없이 토큰을 위조할 수 있었다. **반드시 허용할 알고리즘 목록을 명시적으로 지정**해야 한다.

```python
# 잘못된 방식 (alg를 토큰에서 신뢰)
jwt.decode(token, secret)  # 위험!

# 올바른 방식 (알고리즘 명시)
jwt.decode(token, secret, algorithms=['HS256'])  # 안전
```

### 2. RS256 vs HS256

| 알고리즘 | 서명 | 검증 | 사용 시나리오 |
|---------|------|------|--------------|
| HS256 | HMAC-SHA256 (대칭키) | 동일한 비밀키 | 단일 서비스 |
| RS256 | RSA-SHA256 (비대칭키) | 공개키로 검증 | 마이크로서비스, OpenID Connect |

마이크로서비스 환경에서 RS256은 권한 서버만 개인키로 서명하고, 각 서비스는 공개키로만 검증하므로 비밀키 배포 위험이 없다.

### 3. 토큰 저장 위치

| 위치 | XSS 위험 | CSRF 위험 | 권장 |
|------|---------|---------|------|
| localStorage | 높음 | 없음 | ✗ |
| sessionStorage | 높음 | 없음 | △ |
| HttpOnly Cookie | 없음 | 있음 | ✓ (CSRF 방어 추가) |
| Memory (변수) | 낮음 | 없음 | ✓ (페이지 새로고침 시 소실) |

### 4. Refresh Token 보안

- Refresh Token은 오래 살고 강력하다. 반드시 `HttpOnly`, `Secure`, `SameSite=Strict` 쿠키에 저장
- Refresh Token Rotation: 리프레시 토큰을 사용할 때마다 새 것을 발급하고 이전 것을 무효화
- 탐지된 재사용 시 해당 사용자의 모든 세션을 만료

### 5. OAuth 2.0 vs OpenID Connect (OIDC)

OAuth 2.0은 **인가** 프레임워크이지 **인증** 프레임워크가 아니다. "구글 로그인"처럼 신원 확인이 필요할 때는 OAuth 2.0 위에 구축된 **OpenID Connect(OIDC)**를 사용해야 한다. OIDC는 `id_token`(사용자 신원 정보를 담은 JWT)을 추가로 발급한다.

```
OAuth 2.0: "사용자를 대신해 구글 캘린더에 접근할 수 있다"
OIDC:      "사용자가 alice@gmail.com임을 보장한다 + OAuth 2.0"
```

## 결론

OAuth 2.0은 비밀번호 없이 제3자 앱에 제한된 리소스 접근 권한을 위임하는 표준이다. JWT는 서버 상태 없이 자가 검증 가능한 정보를 담는 토큰 형식이다. 이 두 가지를 올바르게 이해하고 구현하면 확장 가능하고 안전한 현대 인증 시스템을 구축할 수 있다.

핵심 체크리스트:
- Authorization Code + PKCE를 기본으로 사용한다
- JWT 알고리즘을 명시적으로 지정한다
- 민감한 정보를 Payload에 넣지 않는다
- Refresh Token은 HttpOnly 쿠키로 보관한다
- 인증(신원 확인)에는 OIDC를 사용한다

## 참고 자료
- [RFC 6749 - The OAuth 2.0 Authorization Framework](https://datatracker.ietf.org/doc/html/rfc6749)
- [RFC 7519 - JSON Web Token (JWT)](https://datatracker.ietf.org/doc/html/rfc7519)
- [RFC 7636 - PKCE for OAuth Public Clients](https://datatracker.ietf.org/doc/html/rfc7636)
- [OAuth 2.0 공식 사이트](https://oauth.net/2/)
