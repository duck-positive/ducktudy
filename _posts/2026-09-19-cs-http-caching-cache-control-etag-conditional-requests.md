---
layout: post
title: "HTTP 캐싱 완전 정복: Cache-Control, ETag, 조건부 요청과 CDN 전략"
date: 2026-09-19
categories: [cs, computer-science]
tags: [http, caching, cache-control, etag, cdn, rfc9111, web-performance]
---

## 개요

HTTP 캐싱은 웹 성능 최적화의 가장 강력한 도구 중 하나입니다. 올바르게 구성하면 서버 부하를 90% 이상 줄이고, 사용자 체감 로딩 속도를 수십 배 개선할 수 있습니다. 반면 잘못 설정하면 구 버전 파일이 계속 서비스되는 참사가 발생합니다.

HTTP 캐싱은 2022년 발표된 [RFC 9111](https://www.rfc-editor.org/rfc/rfc9111.html)로 정의됩니다. 이 글에서는 캐시의 동작 원리부터 실무 전략까지를 체계적으로 정리합니다.

---

## 캐시 계층 구조

HTTP 요청이 처리되는 경로에는 여러 캐시 레이어가 존재합니다.

```
사용자 브라우저
    ↓ (캐시 미스 시)
Service Worker 캐시
    ↓
브라우저 디스크 캐시
    ↓
프록시 캐시 (기업 방화벽, ISP)
    ↓
CDN Edge 캐시 (CloudFront, Cloudflare 등)
    ↓
원본 서버(Origin)
```

캐시의 핵심 개념은 **신선도(Freshness)**입니다. 캐시된 응답이 신선하면 원본 서버에 요청하지 않고 캐시에서 즉시 반환합니다.

---

## Cache-Control 헤더 심화

`Cache-Control`은 RFC 9111의 핵심 헤더로, 요청과 응답 양방향에서 사용됩니다.

### 응답 측 주요 디렉티브

```
Cache-Control: max-age=3600, s-maxage=86400, must-revalidate, public
```

| 디렉티브 | 의미 |
|---------|------|
| `max-age=N` | N초 동안 신선한 것으로 간주 (브라우저 캐시) |
| `s-maxage=N` | N초 동안 신선 (공유 캐시/CDN에만 적용, max-age 덮어씀) |
| `public` | 응답을 모든 캐시(공유 캐시 포함)에 저장 가능 |
| `private` | 브라우저 캐시에만 저장 가능 (CDN 저장 불가) |
| `no-cache` | 저장은 하되, 사용 전 반드시 서버에 재검증 필요 |
| `no-store` | 어떤 캐시에도 저장하지 않음 (민감 데이터) |
| `must-revalidate` | max-age 초과 시 반드시 재검증 (stale 응답 금지) |
| `stale-while-revalidate=N` | 만료 후 N초 동안은 stale 응답을 반환하며 백그라운드 갱신 |
| `immutable` | 응답 내용이 절대 변경되지 않음 (max-age 기간 내 재검증 없음) |

### 요청 측 주요 디렉티브

```
Cache-Control: no-cache
```

브라우저가 강제 새로고침(Ctrl+F5)을 할 때 전송합니다. 서버에 최신 응답을 강제합니다.

### 실전 시나리오별 설정

```nginx
# Nginx 설정 예시

# 1. HTML 문서: 항상 재검증
location ~* \.html$ {
    add_header Cache-Control "no-cache, must-revalidate";
    add_header Vary "Accept-Encoding";
}

# 2. 버전된 정적 자산 (빌드 해시 포함): 영구 캐시
# 파일명: app.a3f9c2.js, style.7b2d1e.css
location ~* \.(js|css)$ {
    if ($uri ~* "[a-f0-9]{8}\.(js|css)$") {
        add_header Cache-Control "public, max-age=31536000, immutable";
    }
}

# 3. 이미지: 1일 캐시 후 재검증
location ~* \.(jpg|jpeg|png|webp|svg)$ {
    add_header Cache-Control "public, max-age=86400, stale-while-revalidate=3600";
}

# 4. API 응답: 개인 데이터 캐시 금지
location /api/user {
    add_header Cache-Control "private, no-store";
}
```

---

## 조건부 요청: ETag와 Last-Modified

캐시된 응답이 만료된 경우, 브라우저는 **조건부 요청(Conditional Request)**으로 내용이 변경되었는지 서버에 묻습니다. 내용이 동일하면 서버는 `304 Not Modified`를 반환하고 바디 전송을 생략합니다. 이것이 **재검증(Revalidation)**입니다.

### ETag (Entity Tag)

ETag는 리소스의 버전을 식별하는 불투명한 문자열입니다.

```
# 1차 요청 → 서버 응답
GET /api/products HTTP/1.1

HTTP/1.1 200 OK
ETag: "a3f9c2d7e1b8"
Cache-Control: no-cache
Content-Type: application/json

[{"id": 1, "name": "Widget"}, ...]


# 2차 요청 → 브라우저가 If-None-Match 헤더 포함
GET /api/products HTTP/1.1
If-None-Match: "a3f9c2d7e1b8"

HTTP/1.1 304 Not Modified
ETag: "a3f9c2d7e1b8"
# 바디 없음 → 네트워크 비용 절감!
```

### Last-Modified / If-Modified-Since

날짜 기반 재검증입니다. ETag보다 정밀도가 낮지만(1초 단위) 간단합니다.

```
# 1차 응답
HTTP/1.1 200 OK
Last-Modified: Fri, 19 Sep 2026 08:00:00 GMT

# 재검증 요청
GET /image.png HTTP/1.1
If-Modified-Since: Fri, 19 Sep 2026 08:00:00 GMT
```

### ETag 생성 전략

```python
# Python/Flask에서 ETag 구현
import hashlib
from flask import Flask, request, make_response, jsonify

app = Flask(__name__)

@app.route('/api/data')
def get_data():
    data = fetch_data_from_db()  # 실제 데이터 조회
    
    # 방법 1: 내용 기반 ETag (강한 검증자)
    content = jsonify(data).get_data()
    etag = hashlib.sha256(content).hexdigest()[:16]
    
    # 방법 2: 버전 기반 ETag
    # etag = f'"{data["version"]}"'
    
    # If-None-Match 처리
    if request.headers.get('If-None-Match') == f'"{etag}"':
        return '', 304, {'ETag': f'"{etag}"'}
    
    resp = make_response(content)
    resp.headers['ETag'] = f'"{etag}"'
    resp.headers['Cache-Control'] = 'no-cache'
    resp.headers['Content-Type'] = 'application/json'
    return resp
```

### 강한 ETag vs 약한 ETag

```
ETag: "abc123"      → 강한(Strong): 바이트 단위 동일성 보장
ETag: W/"abc123"    → 약한(Weak): 의미적 동일성만 보장 (gzip 재압축 시 유용)
```

---

## Vary 헤더: 다차원 캐시 키

`Vary` 헤더는 캐시 키에 추가 요청 헤더를 포함시킵니다. 같은 URL이라도 `Accept-Encoding`이 다르면 다른 캐시 항목으로 관리합니다.

```python
# Express.js (Node.js) 예시
app.get('/api/data', (req, res) => {
    res.set({
        'Cache-Control': 'public, max-age=3600',
        'Vary': 'Accept-Encoding, Accept-Language',
        // Accept-Encoding: gzip 요청과 identity 요청을 별도 캐시
        // Accept-Language: ko 응답과 en 응답을 별도 캐시
    });
    res.json(data);
});
```

**주의**: `Vary: *`는 캐시를 완전히 비활성화합니다. `Vary: Cookie`도 공유 캐시를 사실상 무력화합니다. CDN에서는 Vary 헤더 처리 방식이 제품마다 다르므로 문서를 확인하세요.

---

## CDN 캐싱 전략

CDN(Content Delivery Network)은 전 세계 Edge PoP(Point of Presence)에서 캐시를 운용합니다.

### Cache-Control + s-maxage 조합

```
# CDN에서 7일 캐시, 브라우저에서 1시간 캐시
Cache-Control: public, max-age=3600, s-maxage=604800
```

### Surrogate-Control (CDN 전용)

일부 CDN은 `Surrogate-Control` 헤더를 지원합니다. CDN이 소비한 후 제거하므로 브라우저에 노출되지 않습니다.

```
Surrogate-Control: max-age=86400  # CDN만 보는 TTL
Cache-Control: no-cache           # 브라우저는 항상 재검증
```

### Cache 무효화 (Purge/Invalidation)

배포 시 CDN 캐시를 즉시 갱신해야 할 경우 Purge API를 사용합니다.

```bash
# Cloudflare Cache Purge API
curl -X POST "https://api.cloudflare.com/client/v4/zones/{zone_id}/purge_cache" \
  -H "Authorization: Bearer ${CF_TOKEN}" \
  -H "Content-Type: application/json" \
  --data '{"files":["https://example.com/api/products"]}'

# 태그 기반 Purge (Surrogate-Key)
curl -X POST "https://api.cloudflare.com/client/v4/zones/{zone_id}/purge_cache" \
  -H "Authorization: Bearer ${CF_TOKEN}" \
  -H "Content-Type: application/json" \
  --data '{"tags":["product-catalog"]}'
```

```python
# 응답 시 태그 설정 (캐시 선택적 무효화)
response.headers['Cache-Tag'] = 'product-catalog,product-1234'
response.headers['Surrogate-Key'] = 'product-catalog'
```

---

## stale-while-revalidate: 성능과 신선도의 절충

`stale-while-revalidate`는 만료된 응답을 즉시 반환하면서 백그라운드에서 갱신을 트리거합니다. "항상 빠른 응답 + 결국 신선한 데이터"를 동시에 달성합니다.

```
Cache-Control: max-age=60, stale-while-revalidate=3600

→ 0~60초: 캐시에서 즉시 반환 (신선)
→ 60~3660초: stale 응답 즉시 반환 + 백그라운드에서 재검증
→ 3660초 초과: 강제 재검증
```

```javascript
// Service Worker에서 stale-while-revalidate 구현
self.addEventListener('fetch', event => {
    event.respondWith(
        caches.open('v1').then(cache => {
            return cache.match(event.request).then(cachedResponse => {
                const fetchPromise = fetch(event.request).then(networkResponse => {
                    cache.put(event.request, networkResponse.clone());
                    return networkResponse;
                });
                // 캐시가 있으면 즉시 반환, 없으면 네트워크 대기
                return cachedResponse || fetchPromise;
            });
        })
    );
});
```

---

## 실전 캐싱 전략 패턴

### 패턴 1: 콘텐츠 해시 기반 정적 자산 영구 캐시

빌드 도구(Webpack, Vite)가 파일 내용의 해시를 파일명에 포함시키면, 내용이 변경될 때마다 새 URL이 생성됩니다. 이를 이용해 정적 자산을 `immutable`로 캐시합니다.

```
# 빌드 결과
dist/
  index.html                    ← Cache-Control: no-cache
  assets/
    app.a3f9c2d7.js             ← Cache-Control: public, max-age=31536000, immutable
    style.7b2d1e4f.css          ← Cache-Control: public, max-age=31536000, immutable
    logo.9f3a2c1b.webp          ← Cache-Control: public, max-age=31536000, immutable
```

`index.html`은 `no-cache`로 항상 재검증하여 최신 자산 URL을 참조하도록 합니다.

### 패턴 2: API 응답 캐싱 + ETag 재검증

```python
# Django REST Framework 예시
from django.views.decorators.cache import cache_control
from django.views.decorators.vary import vary_on_headers

@cache_control(public=True, max_age=60, s_maxage=3600)
@vary_on_headers('Accept-Language')
def product_list(request):
    products = Product.objects.all()
    etag = compute_etag(products)  # 쿼리셋 기반 ETag 계산

    if request.META.get('HTTP_IF_NONE_MATCH') == etag:
        return HttpResponse(status=304)

    response = JsonResponse(serialize(products), safe=False)
    response['ETag'] = etag
    return response
```

---

## 주의사항

1. **민감 데이터**: 사용자 개인정보, 인증 토큰은 반드시 `Cache-Control: private, no-store` 사용
2. **Set-Cookie**: 응답에 `Set-Cookie`가 있으면 공유 캐시에 저장되지 않음 (RFC 9111)
3. **HTTPS**: HTTP에서 민감한 응답이 프록시에 캐시될 수 있음. 항상 HTTPS 사용
4. **CDN과 Vary**: `Vary: User-Agent`는 캐시 효율을 극단적으로 낮춤 — 사용 금지
5. **max-age=0 vs no-cache**: `max-age=0`은 즉시 만료(재검증 후 사용 가능), `no-cache`는 저장은 허용하되 반드시 재검증

---

## 참고 자료

- [RFC 9111 — HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111.html)
- [RFC 7232 — HTTP Conditional Requests](https://datatracker.ietf.org/doc/html/rfc7232)
- [HTTP 캐싱 — MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Caching)
- [HTTP ETag — Wikipedia](https://en.wikipedia.org/wiki/HTTP_ETag)
