---
layout: post
title: "DNS 내부 동작 원리 완전 정복: 도메인이 IP가 되는 여정과 DNSSEC"
date: 2026-07-27
categories: [cs, computer-science]
tags: [dns, dnssec, 네트워크, 분산시스템, 보안, 인터넷프로토콜]
---

브라우저에 `https://www.example.com`을 입력하면 수 밀리초 안에 응답이 돌아온다. 이 짧은 시간 동안 수십 개의 서버가 협력하여 `www.example.com`을 IP 주소로 바꾸는 놀라운 분산 시스템, DNS(Domain Name System)가 작동한다. DNS는 인터넷에서 가장 크고 오래되었으며 성능이 중요한 분산 데이터베이스다.

## 개념: DNS의 계층 구조

### DNS는 왜 분산되어야 하는가

초창기 인터넷은 `/etc/hosts` 파일 하나로 모든 호스트명을 관리했다. Stanford Research Institute(SRI)에서 유지하던 이 파일은 수십 개의 호스트만 존재할 때는 충분했지만, 네트워크가 성장하면서 세 가지 문제에 직면했다:

- **확장성**: 수십억 개의 도메인을 단일 서버 하나가 처리할 수 없다
- **일관성**: 전 세계에서 동시에 갱신되는 레코드를 단일 파일로 동기화할 수 없다
- **가용성**: 단일 장애점(SPOF)은 인터넷 전체를 마비시킨다

DNS는 이 문제를 **위임(delegation)** 기반의 계층 구조로 해결한다.

### DNS 계층 구조

```
                        . (루트)
                       /|\
                      / | \
                    com net org  ← Top-Level Domain (TLD)
                   /
               example
              /
            www  ← 최종 도메인
```

각 계층은 서로 다른 조직이 관리한다:

- **루트 네임서버**: ICANN이 위임한 13개 조직이 운영 (A~M). 실제로는 Anycast로 600개 이상의 물리 서버
- **TLD 네임서버**: `.com`은 Verisign, `.kr`은 KRNIC이 관리
- **권한 네임서버(Authoritative NS)**: 도메인 소유자(또는 DNS 호스팅 업체)가 관리. `example.com`의 실제 IP를 알고 있는 서버
- **재귀 리졸버(Recursive Resolver)**: ISP나 8.8.8.8(Google), 1.1.1.1(Cloudflare)이 운영. 클라이언트를 대신해 계층을 탐색

### DNS 메시지 구조

DNS는 UDP 포트 53을 기본으로 사용한다 (응답이 512바이트 초과 시 TCP, DNSSEC에서는 주로 TCP 사용).

```
DNS 메시지 형식:
┌────────────────────────────────┐
│ Header (12 바이트)              │
│  ID: 16비트 트랜잭션 ID         │
│  QR: 0=쿼리, 1=응답             │
│  Opcode: 쿼리 유형              │
│  AA: 권한 있는 답변 여부         │
│  TC: 메시지 절삭 여부            │
│  RD: 재귀 원함(Recursion Desired)│
│  RA: 재귀 가능(Recursion Available)│
│  RCODE: 응답 코드 (0=OK, 3=NXDOMAIN)│
├────────────────────────────────┤
│ Question Section               │
│  QNAME: 질의 도메인             │
│  QTYPE: A, AAAA, MX, CNAME 등  │
│  QCLASS: IN(인터넷)             │
├────────────────────────────────┤
│ Answer Section                 │
│ Authority Section              │
│ Additional Section             │
└────────────────────────────────┘
```

## 왜 필요한가: 재귀 쿼리의 전체 흐름

`www.example.com`을 처음 조회할 때 캐시가 없다면 다음과 같은 과정이 일어난다.

```
클라이언트                 재귀 리졸버              루트 NS          .com TLD NS     example.com NS
    │                         │                       │                  │                │
    │─ www.example.com? ──────▶│                       │                  │                │
    │                         │─ www.example.com? ───▶│                  │                │
    │                         │◀─ .com NS: a.gtld-servers.net ──────────│                │
    │                         │─ www.example.com? ─────────────────────▶│                │
    │                         │◀─ example.com NS: ns1.example.com ───────────────────────│
    │                         │─ www.example.com? ──────────────────────────────────────▶│
    │                         │◀─ A: 93.184.216.34 ──────────────────────────────────────│
    │◀─ 93.184.216.34 ────────│
```

이 과정에서 재귀 리졸버가 모든 복잡성을 대신 처리한다. 클라이언트는 리졸버에 한 번만 물어보면 된다.

## 실제 구현 예제

### 예제 1: 파이썬으로 DNS 쿼리 직접 구현

```python
import socket
import struct
import random

def build_dns_query(domain: str, qtype: int = 1) -> bytes:
    """DNS 쿼리 패킷을 직접 생성 (qtype=1: A레코드, 28: AAAA)"""
    transaction_id = random.randint(0, 65535)
    
    # Header: ID(2) + Flags(2) + QDCOUNT(2) + ANCOUNT(2) + NSCOUNT(2) + ARCOUNT(2)
    # Flags: 0x0100 = QR=0(쿼리), RD=1(재귀 원함)
    header = struct.pack('>HHHHHH', transaction_id, 0x0100, 1, 0, 0, 0)
    
    # Question: 도메인을 DNS 이름 형식으로 인코딩
    question = b''
    for part in domain.split('.'):
        question += bytes([len(part)]) + part.encode()
    question += b'\x00'  # 루트 레이블 (종료)
    question += struct.pack('>HH', qtype, 1)  # QTYPE, QCLASS=IN
    
    return header + question

def parse_dns_response(data: bytes) -> list:
    """DNS 응답에서 A 레코드 IP 주소 파싱"""
    # Header 파싱
    transaction_id, flags, qdcount, ancount = struct.unpack('>HHHH', data[:8])
    rcode = flags & 0x000F
    
    if rcode == 3:
        raise Exception("NXDOMAIN: 도메인이 존재하지 않습니다")
    if rcode != 0:
        raise Exception(f"DNS 오류: RCODE={rcode}")
    
    # Question section 건너뛰기
    offset = 12
    for _ in range(qdcount):
        while data[offset] != 0:
            if data[offset] & 0xC0 == 0xC0:  # 포인터 압축
                offset += 2
                break
            offset += data[offset] + 1
        else:
            offset += 1
        offset += 4  # QTYPE, QCLASS
    
    # Answer section 파싱
    ips = []
    for _ in range(ancount):
        # 이름 (포인터 압축 처리)
        if data[offset] & 0xC0 == 0xC0:
            offset += 2
        else:
            while data[offset] != 0:
                offset += data[offset] + 1
            offset += 1
        
        rtype, rclass, ttl, rdlength = struct.unpack('>HHIH', data[offset:offset+10])
        offset += 10
        
        if rtype == 1 and rdlength == 4:  # A 레코드
            ip = '.'.join(str(b) for b in data[offset:offset+4])
            ips.append(ip)
        offset += rdlength
    
    return ips

def dns_lookup(domain: str, dns_server: str = '8.8.8.8') -> list:
    """DNS 조회 수행"""
    query = build_dns_query(domain)
    
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    sock.settimeout(3.0)
    
    try:
        sock.sendto(query, (dns_server, 53))
        response, _ = sock.recvfrom(4096)
        return parse_dns_response(response)
    finally:
        sock.close()

# 사용 예시
try:
    ips = dns_lookup('google.com')
    print(f"google.com → {ips}")
    
    ips2 = dns_lookup('github.com', '1.1.1.1')
    print(f"github.com → {ips2}")
except Exception as e:
    print(f"오류: {e}")
```

### 예제 2: Go로 구현한 DNS 캐시 리졸버

```go
package main

import (
    "fmt"
    "net"
    "sync"
    "time"
)

// DNS 캐시 엔트리
type CacheEntry struct {
    IPs       []string
    ExpiresAt time.Time
}

// TTL 기반 DNS 캐시
type DNSCache struct {
    mu    sync.RWMutex
    cache map[string]*CacheEntry
}

func NewDNSCache() *DNSCache {
    dc := &DNSCache{
        cache: make(map[string]*CacheEntry),
    }
    // 백그라운드 만료 정리
    go dc.evictExpired()
    return dc
}

func (dc *DNSCache) Get(domain string) ([]string, bool) {
    dc.mu.RLock()
    defer dc.mu.RUnlock()
    
    entry, ok := dc.cache[domain]
    if !ok || time.Now().After(entry.ExpiresAt) {
        return nil, false
    }
    return entry.IPs, true
}

func (dc *DNSCache) Set(domain string, ips []string, ttl time.Duration) {
    dc.mu.Lock()
    defer dc.mu.Unlock()
    
    dc.cache[domain] = &CacheEntry{
        IPs:       ips,
        ExpiresAt: time.Now().Add(ttl),
    }
}

func (dc *DNSCache) evictExpired() {
    ticker := time.NewTicker(30 * time.Second)
    for range ticker.C {
        dc.mu.Lock()
        now := time.Now()
        for key, entry := range dc.cache {
            if now.After(entry.ExpiresAt) {
                delete(dc.cache, key)
            }
        }
        dc.mu.Unlock()
    }
}

// 캐시를 활용한 DNS 리졸버
type CachingResolver struct {
    cache *DNSCache
}

func (r *CachingResolver) Lookup(domain string) ([]string, error) {
    // 1. 캐시 확인
    if ips, ok := r.cache.Get(domain); ok {
        fmt.Printf("[캐시 HIT] %s → %v\n", domain, ips)
        return ips, nil
    }
    
    // 2. 실제 DNS 조회
    fmt.Printf("[캐시 MISS] %s 조회 중...\n", domain)
    addrs, err := net.LookupHost(domain)
    if err != nil {
        return nil, fmt.Errorf("DNS 조회 실패: %w", err)
    }
    
    // 3. TTL 300초로 캐싱 (실제로는 DNS 응답의 TTL 값 사용)
    r.cache.Set(domain, addrs, 300*time.Second)
    fmt.Printf("[캐시 저장] %s → %v (TTL: 300s)\n", domain, addrs)
    
    return addrs, nil
}

func main() {
    resolver := &CachingResolver{
        cache: NewDNSCache(),
    }
    
    domains := []string{"google.com", "github.com", "google.com"} // 세 번째는 캐시 히트
    for _, domain := range domains {
        ips, err := resolver.Lookup(domain)
        if err != nil {
            fmt.Printf("오류: %v\n", err)
            continue
        }
        fmt.Printf("결과: %s → %v\n\n", domain, ips)
    }
}
```

## DNS 레코드 타입 심화

### 주요 레코드 타입

```
# A 레코드: 도메인 → IPv4
example.com.    300    IN    A       93.184.216.34

# AAAA 레코드: 도메인 → IPv6
example.com.    300    IN    AAAA    2606:2800:21f:cb07:6820:80da:af6b:8b2c

# CNAME: 별칭 (Canonical Name)
www.example.com. 300   IN    CNAME   example.com.

# MX: 메일 서버 (우선순위 포함)
example.com.    300    IN    MX      10 mail.example.com.
example.com.    300    IN    MX      20 backup-mail.example.com.

# TXT: 텍스트 레코드 (SPF, DKIM, 도메인 인증 등)
example.com.    300    IN    TXT     "v=spf1 include:_spf.google.com ~all"

# NS: 네임서버
example.com.    86400  IN    NS      ns1.example.com.

# SOA: 존의 권한 시작점 (Start of Authority)
example.com.    86400  IN    SOA     ns1.example.com. admin.example.com. (
                                    2026072701 ; 시리얼
                                    3600       ; 리프레시
                                    900        ; 재시도
                                    604800     ; 만료
                                    300 )      ; 최소 TTL
```

### CNAME 체인 문제

```
api.example.com  → CNAME → lb.example.com
lb.example.com   → CNAME → prod-lb-us.cloudprovider.com
prod-lb-us.cloudprovider.com → A → 1.2.3.4
```

CNAME을 중첩하면 DNS 조회 횟수가 늘어나 지연이 커진다. 루트 도메인(`example.com`)에는 CNAME을 쓸 수 없다 (APEX 도메인 제한). 이를 해결하기 위해 Cloudflare, AWS Route 53은 CNAME Flattening 또는 ALIAS 레코드를 제공한다.

## DNSSEC: DNS 보안 확장

### DNS 캐시 포이즈닝 공격

2008년 Dan Kaminsky가 발견한 취약점: 공격자가 재귀 리졸버의 캐시를 오염시켜 사용자를 악성 서버로 리다이렉트할 수 있다. DNS는 UDP를 사용하고 검증 없이 응답을 신뢰하기 때문이다.

```
정상 흐름:
클라이언트 → 리졸버 → 권한NS → 진짜 IP 반환

공격 흐름:
클라이언트 → 리졸버 → [권한NS에 질의하는 동시에]
                      ← 공격자가 위조 응답을 대량 전송
                         트랜잭션 ID 랜덤 추측으로 성공 시
리졸버 캐시에 악성 IP가 저장됨 → 클라이언트는 공격자 서버로 연결
```

### DNSSEC 작동 원리

DNSSEC는 공개키 암호화를 사용한 디지털 서명으로 DNS 응답의 무결성을 보장한다.

```
DNSSEC 레코드 체인:
┌──────────────────────────────────────────────┐
│ 루트 신뢰 앵커(Trust Anchor)                  │
│  루트 KSK(Key Signing Key) 공개키              │
│  (브라우저/OS에 하드코딩)                       │
└─────────────────┬────────────────────────────┘
                  │ 서명
                  ▼
┌──────────────────────────────────────────────┐
│ 루트 존                                        │
│  .com NS 레코드에 대한 RRSIG                   │
│  .com의 DS 레코드 (자식 존의 KSK 해시)          │
└─────────────────┬────────────────────────────┘
                  │ 서명 체인
                  ▼
┌──────────────────────────────────────────────┐
│ .com TLD 존                                    │
│  example.com NS 레코드에 대한 RRSIG            │
│  example.com의 DS 레코드                        │
└─────────────────┬────────────────────────────┘
                  │ 서명 체인
                  ▼
┌──────────────────────────────────────────────┐
│ example.com 권한 존                            │
│  A 레코드 + RRSIG (ZSK로 서명)                 │
│  DNSKEY 레코드 (ZSK + KSK 공개키)              │
└──────────────────────────────────────────────┘

검증기(리졸버)는 루트부터 순차적으로 서명을 검증하며
중간에 변조 여부를 확인한다.
```

## 주의사항과 실전 팁

### 1. TTL 전략

| 상황 | 권장 TTL |
|------|----------|
| 자주 변경되는 레코드 | 60~300초 |
| 안정적인 A 레코드 | 3600초 (1시간) |
| MX, NS 레코드 | 86400초 (1일) |
| 마이그레이션 전 | TTL을 먼저 낮춘 후 변경 |

**실수 예방**: DNS 전환 전 최소 TTL의 2배 시간을 기다려야 이전 캐시가 완전히 만료된다.

### 2. Negative Caching과 NXDOMAIN

존재하지 않는 도메인(NXDOMAIN) 응답도 SOA 레코드의 minimum TTL만큼 캐싱된다. 스크립트에서 잘못된 도메인을 대량 쿼리하면 NXDOMAIN 폭풍으로 리졸버에 과부하가 걸릴 수 있다.

### 3. DNS over HTTPS (DoH)와 DNS over TLS (DoT)

전통적인 DNS는 평문 UDP로 전송되어 ISP가 쿼리를 볼 수 있다. DoH(RFC 8484)와 DoT(RFC 7858)는 DNS 쿼리를 암호화하여 프라이버시를 보호한다.

### 4. Split-Horizon DNS

내부 네트워크에서는 내부 IP를, 외부에서는 공인 IP를 반환하는 분리 DNS 구성이다. 기업 환경에서 VPN 없이 내부 서비스에 접근할 때 활용한다.

## 결론

DNS는 수십 억 개의 도메인을 수 밀리초 안에 해석하는 인터넷의 전화번호부다. 계층적 위임, 분산 캐싱, DNSSEC 서명 체인이 결합하여 확장성·성능·보안을 동시에 달성한다. DNS 전파 지연, CNAME 체인, DNSSEC 설정 오류는 장애의 흔한 원인이다. TTL을 이해하고, 캐시를 활용하며, 보안 설정을 갖추는 것이 실전에서 가장 중요하다.

## 참고 자료
- [RFC 1035 - Domain Names: Implementation and Specification](https://datatracker.ietf.org/doc/html/rfc1035)
- [RFC 4033 - DNS Security Introduction and Requirements (DNSSEC)](https://datatracker.ietf.org/doc/html/rfc4033)
- [Cloudflare Learning: What is DNS?](https://www.cloudflare.com/learning/dns/what-is-dns/)
- [IANA Root Zone Database](https://www.iana.org/domains/root/db)
