---
layout: post
title: "CDN 아키텍처 완전 정복: Anycast·BGP·계층형 캐시가 전 세계 레이턴시를 줄이는 법"
date: 2026-08-20
categories: [cs, computer-science]
tags: [CDN, anycast, BGP, edge-caching, content-delivery-network, DNS, TLS, HTTP, origin-shield, cache-invalidation, networking]
---

유튜브에서 4K 영상이 버퍼링 없이 재생되고, 지구 반대편의 웹사이트가 수백 밀리초 만에 로드된다. 이것이 가능한 이유는 무엇일까? CDN(Content Delivery Network)이다. CDN은 전 세계에 분산된 수천 개의 서버(PoP, Point of Presence)를 통해 콘텐츠를 사용자 가까이에서 전달한다. Cloudflare, Akamai, Fastly, Amazon CloudFront는 단순한 파일 캐시가 아니라, 정교한 네트워킹 기술과 분산 시스템이 결합된 복잡한 인프라다. 이 글에서는 CDN의 내부 동작을 Anycast IP, BGP 라우팅, 계층형 캐시 구조, TLS 종료, 캐시 무효화까지 낱낱이 해부한다.

## CDN이 왜 필요한가

### 레이턴시의 물리적 한계

빛의 속도는 진공에서 초당 약 30만 km, 광섬유에서는 굴절률 때문에 약 20만 km로 줄어든다. 서울에서 뉴욕까지는 약 11,000 km이므로, 순수 전파 지연만 55ms다. 여기에 라우터 홉, 큐잉 지연을 더하면 편도 150-200ms, 왕복은 300-400ms가 된다.

HTTP 요청 하나에 TCP 핸드셰이크(1.5 RTT) + TLS 핸드셰이크(1-2 RTT) + 데이터 전송이 필요하다. 서버가 멀수록 페이지 로드 시간은 수초에 달할 수 있다.

### CDN의 핵심 가치

- **레이턴시 감소**: 사용자에게 가장 가까운 PoP에서 콘텐츠 제공 (예: 서울 사용자 → 서울 PoP, RTT 1-5ms)
- **오리진 부하 분산**: 전체 트래픽의 90% 이상을 캐시에서 처리, 오리진 서버 부하 최소화
- **DDoS 방어**: 수백 Tbps의 분산된 네트워크 용량으로 공격 트래픽 흡수
- **가용성**: 오리진이 다운돼도 캐시에서 서빙 가능
- **TCP/TLS 최적화**: 사용자와 가까운 곳에서 연결 종료, 오리진까지는 최적화된 장거리 연결 유지

## Anycast: 하나의 IP, 수백 개의 서버

### Anycast의 원리

유니캐스트(Unicast)는 하나의 IP가 하나의 호스트를 가리킨다. **Anycast**는 하나의 IP를 여러 서버가 공유하며, 라우터가 자동으로 "가장 가까운" 서버로 트래픽을 보낸다.

```
사용자 (서울)
    ↓
인터넷 라우터들
    ↓ BGP가 최단 경로 선택
서울 PoP: 1.1.1.1 (Cloudflare DNS 예시)
도쿄 PoP: 1.1.1.1  ← 서울 사용자는 여기 못 감
LA  PoP: 1.1.1.1  ← 서울 사용자는 여기 못 감
```

라우터는 동일한 1.1.1.1을 향한 트래픽을 BGP 경로 선택 알고리즘에 따라 가장 가까운 PoP로 보낸다. 사용자는 IP 주소 하나만 알면 되고, 어디로 연결되는지는 신경 쓸 필요 없다.

### Anycast vs GeoDNS

**GeoDNS**: DNS 질의자의 IP를 기반으로 지역별로 다른 IP를 반환한다. 구현이 간단하지만 DNS TTL만큼의 전환 지연이 발생하고, 사용자 ISP의 리졸버 위치로 판단하기 때문에 부정확할 수 있다.

**Anycast**: BGP 라우팅 레벨에서 처리하므로 패킷 단위로 최적 경로를 선택한다. 장애 복구가 BGP 컨버전스 시간(수초~수십초) 내에 자동으로 이루어진다.

대형 CDN(Cloudflare, Fastly)은 Anycast를 주로 사용하고, 소규모 CDN은 GeoDNS와 혼합 사용한다.

## BGP와 PoP 라우팅

### AS (Autonomous System)

인터넷은 AS들의 집합이다. 각 AS는 독립적으로 관리되는 IP 주소 블록과 라우터 집합이며, 고유한 ASN(Autonomous System Number)을 가진다. Cloudflare는 AS13335, Google은 AS15169다.

CDN은 자신의 AS에서 동일한 IP prefix(예: 1.1.1.0/24)를 전 세계 모든 PoP에서 BGP로 광고한다. 각 BGP 피어(ISP, IXP)는 이 광고를 받아 자신의 라우팅 테이블에 추가한다.

### BGP 경로 선택 알고리즘

여러 AS를 통해 동일한 prefix로 가는 경로가 여럿 있을 때, BGP는 다음 순서로 최적 경로를 선택한다:

```
1. Weight (Cisco 로컬 속성, 높을수록 우선)
2. Local Preference (AS 내부, 높을수록 우선)
3. 로컬 오리지네이션 (자신이 광고하는 경로 우선)
4. AS Path Length (짧을수록 우선) ← CDN이 주로 활용
5. Origin Type (IGP > EGP > Incomplete)
6. MED (Multi-Exit Discriminator, 낮을수록 우선)
7. eBGP > iBGP
8. IGP metric (낮을수록 우선)
```

CDN은 **AS Path Prepending**으로 특정 경로를 의도적으로 길어 보이게 만들어 트래픽을 원하는 PoP로 유도한다. 장애가 발생한 PoP에서는 BGP 광고를 철회(withdraw)해 자동으로 트래픽이 다른 PoP로 전환된다.

## 계층형 캐시 구조

### 3계층 캐시 아키텍처

```
[사용자]
   ↕ 1-10 ms
[L1 엣지 캐시 (PoP)]
  - 수백 GB SSD, 고속 NVMe
  - 핫 콘텐츠만 저장
  - TTL 짧음
  - Hit Rate 목표: 80-90%
   ↕ 10-50 ms
[L2 Origin Shield (중간 계층)]
  - 수 TB HDD/SSD
  - L1 미스 시 여기서 처리
  - 오리진 요청 대신 수신
  - Hit Rate 목표: 95-99%
   ↕ 50-200 ms
[오리진 서버]
  - 전체 콘텐츠
  - L2 미스 시에만 요청
  - 오리진 요청 = 1-5%
```

### Cache Key와 Vary 헤더

CDN은 요청 URL을 기반으로 캐시 키를 생성한다. 그러나 쿼리 파라미터, 쿠키, Accept-Encoding, User-Agent 등에 따라 다른 응답이 필요할 수 있다.

```
# 기본 캐시 키: https://example.com/image.jpg

# Vary: Accept-Encoding 이 있으면:
# CDN은 각 인코딩별로 별도 캐시 항목 유지:
# https://example.com/image.jpg + Accept-Encoding: gzip
# https://example.com/image.jpg + Accept-Encoding: br
# https://example.com/image.jpg + (no encoding)
```

`Vary` 헤더를 남발하면 캐시 hit rate가 급락한다. `Vary: Cookie`는 거의 모든 요청을 캐시 미스로 만든다.

## 실제 구현 예제

### 예제 1: Nginx 기반 엣지 캐시 서버 설정

```nginx
# /etc/nginx/nginx.conf - CDN 엣지 서버 역할을 하는 Nginx 설정

proxy_cache_path /data/nginx/cache
    levels=1:2
    keys_zone=cdn_cache:100m
    max_size=100g
    inactive=60m
    use_temp_path=off;

upstream origin_servers {
    server origin-shield-1.internal:443 weight=5;
    server origin-shield-2.internal:443 weight=5 backup;
    server origin.example.com:443 backup;
    keepalive 32;
}

server {
    listen 443 ssl http2;
    server_name cdn.example.com;

    ssl_certificate /etc/ssl/cdn.example.com.crt;
    ssl_certificate_key /etc/ssl/cdn.example.com.key;
    ssl_protocols TLSv1.3 TLSv1.2;
    ssl_ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    location / {
        proxy_pass https://origin_servers;
        proxy_cache cdn_cache;
        proxy_cache_key "$scheme$host$request_uri";

        proxy_cache_valid 200 301 302 1d;
        proxy_cache_valid 404 1m;
        proxy_cache_valid any 1m;

        proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
        proxy_cache_background_update on;  # stale-while-revalidate
        proxy_cache_lock on;               # request collapse
        proxy_cache_lock_timeout 5s;

        if ($request_method = PURGE) {
            proxy_cache_purge cdn_cache "$scheme$host$request_uri";
        }

        add_header X-Cache-Status $upstream_cache_status;
        add_header X-Served-By $hostname;
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    }

    # 정적 에셋: 버전 해시 포함 URL → 장기 캐시
    location ~* \.(js|css|woff2|png|jpg|webp)$ {
        proxy_pass https://origin_servers;
        proxy_cache cdn_cache;
        proxy_cache_valid 200 365d;
        proxy_cache_key "$scheme$host$request_uri";
        add_header Cache-Control "public, max-age=31536000, immutable";
    }
}
```

### 예제 2: Cache-Control 헤더 설계와 ETag 검증

```python
# CDN 오리진 서버에서의 올바른 캐시 헤더 설계 (Python/FastAPI)
from fastapi import FastAPI, Request, Response
import hashlib
import json

app = FastAPI()

CACHE_STRATEGIES = {
    "immutable_asset": {
        "max-age": 31536000,
        "s-maxage": 31536000,
        "immutable": True,
        "public": True,
    },
    "api_response": {
        "max-age": 0,
        "s-maxage": 60,
        "stale-while-revalidate": 30,
        "stale-if-error": 86400,
        "public": True,
    },
    "user_specific": {
        "no-store": True,
        "private": True,
    },
    "html_page": {
        "max-age": 0,
        "s-maxage": 300,
        "stale-while-revalidate": 60,
        "public": True,
    }
}

def build_cache_control(strategy_name: str) -> str:
    strategy = CACHE_STRATEGIES[strategy_name]
    parts = []
    if strategy.get("public"):  parts.append("public")
    if strategy.get("private"): parts.append("private")
    if strategy.get("no-store"): parts.append("no-store")
    if "max-age" in strategy:   parts.append(f"max-age={strategy['max-age']}")
    if "s-maxage" in strategy:  parts.append(f"s-maxage={strategy['s-maxage']}")
    if strategy.get("immutable"): parts.append("immutable")
    if "stale-while-revalidate" in strategy:
        parts.append(f"stale-while-revalidate={strategy['stale-while-revalidate']}")
    if "stale-if-error" in strategy:
        parts.append(f"stale-if-error={strategy['stale-if-error']}")
    return ", ".join(parts)

def generate_etag(content: bytes) -> str:
    return f'"{hashlib.sha256(content).hexdigest()[:16]}"'

@app.get("/api/products")
async def get_products(request: Request):
    data = {"products": [{"id": 1, "name": "상품 A", "price": 10000}]}
    content = json.dumps(data, ensure_ascii=False).encode()
    etag = generate_etag(content)
    cache_control = build_cache_control("api_response")

    if request.headers.get("If-None-Match") == etag:
        return Response(status_code=304,
                        headers={"ETag": etag, "Cache-Control": cache_control})

    return Response(
        content=content,
        media_type="application/json",
        headers={
            "ETag": etag,
            "Cache-Control": cache_control,
            "Last-Modified": "Thu, 01 Jan 2026 00:00:00 GMT",
            # Surrogate-Key: CDN 태그 기반 일괄 무효화
            "Surrogate-Key": "products product-list category-electronics",
        }
    )

@app.get("/static/{filename}")
async def serve_static(filename: str):
    content = b"/* minified JS with content hash in URL */"
    return Response(
        content=content,
        media_type="application/javascript",
        headers={
            "ETag": generate_etag(content),
            "Cache-Control": build_cache_control("immutable_asset"),
            "Vary": "Accept-Encoding",
        }
    )
```

## TLS 종료와 QUIC 최적화

### TLS 1.3 0-RTT와 CDN

CDN은 사용자 가까이에서 TLS를 종료한다. 오리진까지는 CDN 내부 최적화된 터널을 사용한다.

**TLS 1.3 0-RTT (Early Data)**: 이전에 연결한 서버에 재연결 시, 첫 요청 데이터를 ClientHello와 함께 보낼 수 있다. RTT를 1회 절약. 단, 재전송 공격(Replay Attack) 취약점이 있어 GET 요청만 허용하는 것이 원칙이다.

### HTTP/3와 QUIC

HTTP/3는 UDP 기반 QUIC 프로토콜 위에서 동작한다. TCP의 핵심 문제인 **Head-of-Line(HOL) Blocking** — 패킷 손실 시 스트림 전체가 대기해야 하는 문제 — 을 해결한다.

QUIC의 주요 특성:
- **연결 마이그레이션**: Wi-Fi → 모바일 데이터 전환 시 연결 유지 (Connection ID 기반)
- **0-RTT 연결**: 이전 연결의 세션 파라미터 재사용
- **내장 TLS 1.3**: UDP + QUIC + TLS가 하나의 핸드셰이크로 처리
- **스트림 독립성**: 패킷 손실이 특정 스트림만 지연, 다른 스트림 영향 없음

Anycast와 QUIC의 조합: Anycast PoP가 바뀌어도 QUIC Connection ID를 통해 연결을 이어갈 수 있다 (단, 상태 동기화 필요).

## 캐시 일관성과 무효화 전략

### TTL 기반 만료

가장 간단한 전략. `Cache-Control: max-age=300`이면 300초 후 캐시 만료. 이 사이에 오리진이 변경되어도 사용자는 구 버전을 받는다. **캐시 신선도(Freshness)**와 **오리진 부하** 사이의 트레이드오프.

### Surrogate-Key / Cache Tags 기반 무효화

Fastly, Cloudflare 등 고급 CDN이 지원. 오리진이 응답에 `Surrogate-Key: product-42 category-5` 같은 태그를 붙이면, 콘텐츠 업데이트 시 API 호출 한 번으로 해당 태그의 모든 캐시 항목을 즉시 삭제할 수 있다.

```bash
# Fastly 태그 기반 퍼지 (즉각적)
curl -X POST "https://api.fastly.com/service/{service_id}/purge/product-42" \
     -H "Fastly-Key: $FASTLY_API_KEY"
# → 전 세계 Fastly PoP에서 product-42 태그 항목 즉시 삭제
```

### Stale-While-Revalidate

백그라운드 갱신 패턴. 캐시가 만료되어도 즉시 오리진에서 가져오는 대신, 만료된 캐시를 먼저 반환하고 백그라운드에서 새 버전을 가져온다. 사용자 입장에서 레이턴시 없음, 오리진 부하도 균등하게 분산된다.

## 주의사항과 실무 팁

**캐시 오염(Cache Poisoning)**: 공격자가 악의적인 콘텐츠를 CDN 캐시에 저장시키는 공격. Vary 헤더와 캐시 키를 엄격히 관리하고, 사용자 입력을 캐시 키에 포함하지 않아야 한다.

**Hot Spot 문제**: 특정 콘텐츠에 트래픽이 몰릴 때 캐시 미스가 동시에 쏟아지는 **Thundering Herd 문제**가 발생한다. `proxy_cache_lock on` 설정으로 첫 번째 요청만 오리진에 보내고 나머지는 결과를 기다리게 한다 (Request Collapse).

**Cache-Control의 private vs public**: `private`은 브라우저 캐시는 가능하지만 CDN은 캐시 불가. `public`은 CDN 캐시 가능. 세션 데이터, 사용자별 콘텐츠는 반드시 `private` 또는 `no-store`.

**동적 콘텐츠 캐싱**: 쿠키나 Authorization 헤더가 있으면 대부분 CDN이 기본적으로 캐시하지 않는다. 동적 콘텐츠를 캐시하려면 쿠키를 제거하고 캐시 키를 명시적으로 설정해야 한다.

**CDN 우회 디버깅**: `curl -v -H "Pragma: no-cache" -H "Cache-Control: no-cache" https://cdn.example.com/file.js`로 캐시 우회 요청을 보내 오리진 응답을 직접 확인할 수 있다.

## 참고 자료
- [Cloudflare CDN 레퍼런스 아키텍처](https://developers.cloudflare.com/reference-architecture/architectures/cdn/)
- [Fastly — Surrogate-Key 기반 즉각 퍼지](https://developer.fastly.com/reference/http/http-headers/Surrogate-Key/)
- [RFC 9110 — HTTP Semantics (캐시 관련 섹션)](https://httpwg.org/specs/rfc9110.html)
- [RFC 9000 — QUIC Transport Protocol](https://www.rfc-editor.org/rfc/rfc9000)
