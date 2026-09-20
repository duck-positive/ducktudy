---
layout: post
title: "REST API 설계 원칙 심화: Richardson Maturity Model, HATEOAS, 그리고 실전 Best Practices"
date: 2026-09-20
categories: [cs, computer-science]
tags: [rest, api, http, hateoas, richardson-maturity-model, openapi, web, architecture]
---

## 개요

REST(Representational State Transfer)는 2000년 Roy Fielding이 자신의 박사 논문에서 정의한 분산 하이퍼미디어 시스템을 위한 아키텍처 스타일입니다. "RESTful API"라는 용어는 오늘날 거의 모든 웹 서비스에서 남발되지만, 진정한 REST 원칙을 완전히 만족하는 API는 생각보다 드뭅니다. 이 글에서는 REST의 6가지 제약 조건, Leonard Richardson이 제시한 성숙도 모델, 그리고 실무에서 바로 적용할 수 있는 설계 원칙을 심층적으로 살펴봅니다.

---

## REST의 6가지 아키텍처 제약 조건

### 1. Client-Server (클라이언트-서버 분리)

UI 관심사와 데이터 저장 관심사를 분리합니다. 이를 통해 클라이언트와 서버가 독립적으로 발전할 수 있습니다. 서버는 UI를 신경 쓰지 않고, 클라이언트는 데이터 저장 방식을 알 필요가 없습니다.

### 2. Stateless (무상태성)

각 요청은 서버가 이해하는 데 필요한 모든 정보를 포함해야 합니다. 서버는 세션 상태를 저장하지 않습니다. 이는 확장성(scalability)의 핵심 기반입니다 — 어떤 서버도 요청을 처리할 수 있기 때문에 로드 밸런싱이 단순해집니다.

### 3. Cacheable (캐시 가능성)

응답은 암묵적 또는 명시적으로 캐시 가능 여부를 표시해야 합니다. 올바른 캐시 전략은 클라이언트-서버 상호작용을 줄이고 성능을 향상시킵니다. HTTP의 `Cache-Control`, `ETag`, `Last-Modified` 헤더가 이를 지원합니다.

### 4. Uniform Interface (균일한 인터페이스)

REST의 중심 특징으로, 4가지 하위 제약 조건으로 구성됩니다:
- **Resource Identification**: URI로 리소스를 식별
- **Resource Manipulation through Representations**: 표현(JSON, XML 등)을 통해 리소스를 조작
- **Self-descriptive Messages**: 메시지가 자신을 처리하는 방법을 포함
- **HATEOAS**: 하이퍼미디어를 통한 애플리케이션 상태 엔진

### 5. Layered System (계층화 시스템)

클라이언트는 자신이 최종 서버와 직접 통신하는지, 중간 서버(프록시, 게이트웨이)와 통신하는지 알 수 없습니다. 이를 통해 보안 레이어, 캐시 레이어, 로드 밸런서를 투명하게 삽입할 수 있습니다.

### 6. Code on Demand (선택적)

서버가 클라이언트에 실행 가능한 코드(JavaScript 등)를 전송하여 기능을 확장할 수 있습니다. 유일한 선택적 제약 조건입니다.

---

## Richardson Maturity Model: REST 성숙도 4단계

Leonard Richardson은 REST API의 성숙도를 4단계(Level 0~3)로 분류했습니다. 이 모델은 "얼마나 RESTful한가"를 객관적으로 평가하는 기준을 제시합니다.

### Level 0: 하나의 URI, 하나의 HTTP 메서드

HTTP를 단순한 터널링 메커니즘으로 사용합니다. SOAP 웹 서비스의 전형적인 패턴입니다.

```xml
<!-- 요청: 모든 작업이 POST로 단일 엔드포인트에 -->
POST /api
Content-Type: application/xml

<appointmentRequest>
  <date>2026-09-20</date>
  <slot>14:00</slot>
  <doctor>Kim</doctor>
</appointmentRequest>

<!-- 응답 -->
<appointment>
  <slot doctor="Kim" start="14:00" end="15:00"/>
  <patient>Park</patient>
</appointment>
```

### Level 1: 리소스(Resource) 도입

단일 엔드포인트 대신 각 리소스가 고유한 URI를 갖습니다. 하지만 여전히 모든 작업에 단일 HTTP 메서드(POST)를 사용합니다.

```
POST /doctors/Kim
POST /doctors/Kim/appointments
POST /appointments/1234
```

### Level 2: HTTP 동사(Verbs) 활용

HTTP 메서드(GET, POST, PUT, DELETE, PATCH)를 의미에 맞게 사용하고, 적절한 HTTP 상태 코드를 반환합니다. 오늘날 대부분의 "RESTful" API가 이 수준에 해당합니다.

```http
# 예약 목록 조회
GET /doctors/Kim/appointments?date=2026-09-20
HTTP/1.1 200 OK

# 새 예약 생성
POST /doctors/Kim/appointments
HTTP/1.1 201 Created
Location: /appointments/5678

# 예약 수정
PATCH /appointments/5678
HTTP/1.1 200 OK

# 예약 취소
DELETE /appointments/5678
HTTP/1.1 204 No Content
```

### Level 3: HATEOAS (하이퍼미디어 컨트롤)

응답에 관련 링크를 포함하여 클라이언트가 다음 가능한 액션을 동적으로 탐색할 수 있게 합니다. 진정한 REST의 완성형입니다.

```json
// GET /appointments/5678 응답
{
  "id": "5678",
  "doctor": "Kim",
  "patient": "Park",
  "slot": {
    "date": "2026-09-20",
    "start": "14:00",
    "end": "15:00"
  },
  "status": "confirmed",
  "_links": {
    "self": {
      "href": "/appointments/5678"
    },
    "cancel": {
      "href": "/appointments/5678",
      "method": "DELETE"
    },
    "reschedule": {
      "href": "/appointments/5678/reschedule",
      "method": "POST"
    },
    "doctor": {
      "href": "/doctors/Kim"
    }
  }
}
```

HATEOAS의 핵심 이점은 **클라이언트-서버 결합도 감소**입니다. 클라이언트는 하드코딩된 URL 대신 서버가 제공하는 링크를 따라가므로, 서버가 URL 구조를 변경해도 클라이언트는 영향을 받지 않습니다.

---

## 실전 코드 예제

### 예제 1: Python (FastAPI)로 Level 2 REST API 구현

```python
from fastapi import FastAPI, HTTPException, status
from fastapi.responses import JSONResponse
from pydantic import BaseModel
from typing import Optional
import uuid

app = FastAPI(title="의료 예약 API", version="1.0.0")

# 인메모리 저장소 (실제로는 DB 사용)
appointments: dict[str, dict] = {}

class AppointmentCreate(BaseModel):
    doctor: str
    date: str
    time_slot: str
    patient: str

class AppointmentPatch(BaseModel):
    date: Optional[str] = None
    time_slot: Optional[str] = None

@app.get("/appointments", status_code=status.HTTP_200_OK)
async def list_appointments(doctor: Optional[str] = None):
    """예약 목록 조회. doctor 쿼리로 필터링 가능."""
    result = list(appointments.values())
    if doctor:
        result = [a for a in result if a["doctor"] == doctor]
    return {"appointments": result, "total": len(result)}

@app.post("/appointments", status_code=status.HTTP_201_CREATED)
async def create_appointment(data: AppointmentCreate):
    """새 예약 생성. 성공 시 201 + Location 헤더 반환."""
    appt_id = str(uuid.uuid4())[:8]
    appointment = {
        "id": appt_id,
        **data.model_dump(),
        "status": "confirmed"
    }
    appointments[appt_id] = appointment
    response = JSONResponse(
        content=appointment,
        status_code=status.HTTP_201_CREATED
    )
    response.headers["Location"] = f"/appointments/{appt_id}"
    return response

@app.get("/appointments/{appt_id}")
async def get_appointment(appt_id: str):
    """특정 예약 조회."""
    if appt_id not in appointments:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Appointment {appt_id} not found"
        )
    return appointments[appt_id]

@app.patch("/appointments/{appt_id}")
async def update_appointment(appt_id: str, data: AppointmentPatch):
    """예약 부분 수정. PUT은 전체 교체, PATCH는 부분 수정."""
    if appt_id not in appointments:
        raise HTTPException(status_code=404, detail="Not found")
    update_data = data.model_dump(exclude_none=True)
    appointments[appt_id].update(update_data)
    return appointments[appt_id]

@app.delete("/appointments/{appt_id}", status_code=status.HTTP_204_NO_CONTENT)
async def cancel_appointment(appt_id: str):
    """예약 취소. 성공 시 204 No Content (바디 없음)."""
    if appt_id not in appointments:
        raise HTTPException(status_code=404, detail="Not found")
    del appointments[appt_id]
```

### 예제 2: HATEOAS 링크를 포함한 응답 직렬화 (Python)

```python
from dataclasses import dataclass, field
from typing import Any
import json

@dataclass
class Link:
    href: str
    method: str = "GET"
    rel: str = ""

@dataclass
class HateoasResponse:
    """HATEOAS 응답 래퍼: 데이터 + _links 포함."""
    data: dict[str, Any]
    links: dict[str, Link] = field(default_factory=dict)

    def to_dict(self) -> dict:
        result = dict(self.data)
        result["_links"] = {
            rel: {"href": link.href, "method": link.method}
            for rel, link in self.links.items()
        }
        return result

def build_appointment_response(appt: dict, base_url: str = "") -> dict:
    """예약 데이터에 HATEOAS 링크를 추가하여 반환."""
    appt_id = appt["id"]
    path = f"{base_url}/appointments/{appt_id}"

    links = {
        "self": Link(href=path),
        "cancel": Link(href=path, method="DELETE"),
        "doctor": Link(href=f"{base_url}/doctors/{appt['doctor']}"),
    }

    # 상태에 따라 가능한 액션 동적 결정
    if appt["status"] == "confirmed":
        links["reschedule"] = Link(href=f"{path}/reschedule", method="POST")

    return HateoasResponse(data=appt, links=links).to_dict()

# 사용 예
sample = {
    "id": "abc123",
    "doctor": "Kim",
    "patient": "Park",
    "date": "2026-09-20",
    "time_slot": "14:00",
    "status": "confirmed"
}
print(json.dumps(build_appointment_response(sample), ensure_ascii=False, indent=2))
```

출력:
```json
{
  "id": "abc123",
  "doctor": "Kim",
  "patient": "Park",
  "date": "2026-09-20",
  "time_slot": "14:00",
  "status": "confirmed",
  "_links": {
    "self": {"href": "/appointments/abc123", "method": "GET"},
    "cancel": {"href": "/appointments/abc123", "method": "DELETE"},
    "doctor": {"href": "/doctors/Kim", "method": "GET"},
    "reschedule": {"href": "/appointments/abc123/reschedule", "method": "POST"}
  }
}
```

---

## REST API 설계 Best Practices

### URI 설계 원칙

| 원칙 | 나쁜 예 | 좋은 예 |
|------|---------|---------|
| 명사 사용 | `/getUsers` | `/users` |
| 복수형 | `/user/1` | `/users/1` |
| 소문자 + 하이픈 | `/User_Posts` | `/user-posts` |
| 계층 표현 | `/getUserPosts?id=1` | `/users/1/posts` |
| 동사 금지 (행위는 HTTP 메서드로) | `/users/1/deletePost/5` | `DELETE /users/1/posts/5` |

### HTTP 상태 코드 올바른 사용

```
2xx 성공
  200 OK             - 조회, 수정 성공
  201 Created        - 리소스 생성 성공 (Location 헤더 포함)
  204 No Content     - 삭제 성공 (응답 바디 없음)

4xx 클라이언트 오류
  400 Bad Request    - 잘못된 요청 파라미터
  401 Unauthorized   - 인증 필요
  403 Forbidden      - 권한 없음 (인증은 됐지만 접근 불가)
  404 Not Found      - 리소스 없음
  409 Conflict       - 충돌 (중복 생성 시도 등)
  422 Unprocessable  - 유효성 검사 실패

5xx 서버 오류
  500 Internal       - 예상치 못한 서버 오류
  503 Unavailable    - 서비스 일시 불가 (Retry-After 헤더 포함)
```

### API 버전 관리 전략

```http
# 전략 1: URI 버전 (가장 일반적, 캐시 친화적)
GET /v1/users
GET /v2/users

# 전략 2: 커스텀 헤더
GET /users
API-Version: 2026-09-20

# 전략 3: Accept 헤더 (Content Negotiation, 순수 REST)
GET /users
Accept: application/vnd.myapi.v2+json
```

URI 버전은 명시적이고 캐시하기 쉬워 가장 많이 사용됩니다. GitHub, Stripe 등 주요 API가 이 방식을 채택합니다.

---

## 주의사항과 팁

**1. GET 요청에 사이드 이펙트를 넣지 마세요**
GET은 안전(safe)하고 멱등(idempotent)해야 합니다. 데이터를 변경하는 로직을 GET에 넣으면 캐시, 크롤러, 브라우저 프리페치가 의도치 않은 부작용을 일으킬 수 있습니다.

**2. 에러 응답도 일관된 스키마를 사용하세요**
RFC 7807 "Problem Details for HTTP APIs"를 따르면 클라이언트가 에러를 파싱하기 쉬워집니다:
```json
{
  "type": "https://example.com/errors/validation-failed",
  "title": "Validation Failed",
  "status": 422,
  "detail": "The 'date' field must be in ISO 8601 format.",
  "instance": "/appointments"
}
```

**3. Pagination은 Cursor 기반을 선호하세요**
오프셋 기반(`?page=3&size=20`)은 대용량 데이터에서 성능 문제가 있고, 데이터 삽입/삭제 시 페이지 건너뜀/중복 문제가 발생합니다. Cursor 기반(`?after=cursor_token`)은 이런 문제를 해결합니다.

**4. HATEOAS는 현실적으로 판단하세요**
Level 3은 이론적으로 완벽하지만, 클라이언트 개발팀과 계약 상 합의가 필요합니다. 대부분의 팀은 Level 2 + OpenAPI 문서화로 실용적인 균형을 찾습니다.

**5. 멱등성을 적극 활용하세요**
PUT과 DELETE는 멱등이어야 합니다 — 같은 요청을 여러 번 보내도 결과가 같아야 합니다. POST는 멱등이 아니므로, 재시도 시 중복 생성을 방지하려면 `Idempotency-Key` 헤더를 도입하세요(Stripe, Stripe API 등이 사용하는 패턴).

---

## 참고 자료

- [Richardson Maturity Model - REST API Tutorial](https://restfulapi.net/richardson-maturity-model/)
- [HATEOAS - htmx Essays](https://htmx.org/essays/hateoas/)
- [RESTful API Design: Richardson Maturity Model - GeeksforGeeks](https://www.geeksforgeeks.org/node-js/richardson-maturity-model-restful-api/)
- [RFC 7807 - Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc7807)
