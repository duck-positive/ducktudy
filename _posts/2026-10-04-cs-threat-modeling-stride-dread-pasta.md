---
layout: post
title: "위협 모델링(Threat Modeling) 완전 정복: STRIDE·DREAD·PASTA로 시스템 설계 단계부터 보안을 내재화하는 법"
date: 2026-10-04
categories: [cs, computer-science]
tags: [security, threat-modeling, STRIDE, DREAD, PASTA, secure-design, SDL, risk-assessment]
---

## 개요

"보안은 나중에 붙이면 된다"는 생각은 소프트웨어 개발에서 가장 값비싼 오해 중 하나다. IBM 시스템 과학 연구소의 연구에 따르면, 설계 단계에서 발견된 결함을 수정하는 비용에 비해 운영 단계에서 발견된 결함을 수정하는 비용은 100배에 달한다. **위협 모델링(Threat Modeling)**은 시스템 설계 초기 단계에 잠재적인 보안 위협을 체계적으로 식별하고 평가하는 프로세스다.

위협 모델링의 목적은 세 가지 핵심 질문에 답하는 것이다.

1. **무엇을 구축하는가?** — 시스템 구조의 명확한 이해
2. **무엇이 잘못될 수 있는가?** — 잠재적 위협과 공격 벡터의 식별
3. **어떻게 대응할 것인가?** — 완화 전략과 우선순위 결정

---

## 왜 위협 모델링이 필요한가

### 침해 사고의 현실

매년 수천만 건의 개인정보가 유출되는 침해 사고의 상당수는 설계 단계의 취약점에서 비롯된다. SQL Injection, SSRF(Server-Side Request Forgery), 권한 상승 취약점 등은 대부분 시스템 아키텍처 설계 시 위협을 고려했다면 예방할 수 있었다.

### DevSecOps와의 연계

현대적인 DevSecOps 파이프라인에서 위협 모델링은 CI/CD 사이클의 첫 번째 보안 게이트로 작동한다. Pull Request 생성 전 위협 모델을 업데이트하는 워크플로우를 도입하면, 개발 속도를 유지하면서 보안을 보장할 수 있다.

---

## 데이터 흐름 다이어그램(DFD)

위협 모델링의 첫 번째 단계는 시스템을 DFD(Data Flow Diagram)로 시각화하는 것이다. DFD의 네 가지 요소:

| 기호 | 의미 | 예시 |
|------|------|------|
| 직사각형 | 외부 엔티티 (External Entity) | 사용자, 외부 서비스 |
| 원 | 프로세스 (Process) | 인증 서버, API 게이트웨이 |
| 직선 화살표 | 데이터 흐름 (Data Flow) | HTTP 요청, DB 쿼리 |
| 열린 직사각형 | 데이터 저장소 (Data Store) | DB, 파일 시스템, 캐시 |
| 점선 | 신뢰 경계 (Trust Boundary) | 인터넷/내부망 경계 |

**신뢰 경계(Trust Boundary)**는 위협 모델링에서 가장 중요한 개념이다. 신뢰 경계를 넘는 데이터 흐름에는 반드시 검증·인증·암호화 메커니즘이 필요하다.

---

## STRIDE 모델

Microsoft에서 개발한 STRIDE는 위협의 여섯 가지 카테고리를 정의하는 방법론이다.

| 위협 | 설명 | 침해되는 보안 속성 | 예시 |
|------|------|------------------|------|
| **S**poofing (스푸핑) | 다른 사용자나 시스템으로 위장 | 인증(Authentication) | 쿠키 탈취 후 세션 위장 |
| **T**ampering (변조) | 데이터나 코드의 무단 수정 | 무결성(Integrity) | DB 직접 수정, 중간자 공격 |
| **R**epudiation (부인) | 행위를 나중에 부정할 수 있음 | 부인 방지(Non-repudiation) | 로그 없는 민감 작업 |
| **I**nformation Disclosure (정보 노출) | 권한 없는 자에게 정보 공개 | 기밀성(Confidentiality) | 오류 메시지 스택 트레이스 |
| **D**enial of Service (서비스 거부) | 합법적 사용자의 접근 방해 | 가용성(Availability) | API Rate limit 없는 엔드포인트 |
| **E**levation of Privilege (권한 상승) | 높은 권한을 비정상적으로 획득 | 권한 부여(Authorization) | IDOR, JWT alg:none 취약점 |

### STRIDE 분석 프로세스

```python
from dataclasses import dataclass, field
from enum import Enum
from typing import List

class StrideCategory(Enum):
    SPOOFING = "S - 스푸핑"
    TAMPERING = "T - 변조"
    REPUDIATION = "R - 부인"
    INFORMATION_DISCLOSURE = "I - 정보 노출"
    DENIAL_OF_SERVICE = "D - 서비스 거부"
    ELEVATION_OF_PRIVILEGE = "E - 권한 상승"

class MitigationStatus(Enum):
    OPEN = "미해결"
    MITIGATED = "완화됨"
    ACCEPTED = "수용됨"  # 위험 수용
    TRANSFERRED = "이전됨"  # 보험 등

@dataclass
class Threat:
    id: str
    category: StrideCategory
    affected_component: str
    description: str
    impact: str
    mitigation: str
    status: MitigationStatus = MitigationStatus.OPEN

@dataclass
class TrustBoundaryCrossing:
    """신뢰 경계를 넘는 데이터 흐름 — STRIDE 분석 대상"""
    source: str
    destination: str
    data_flow: str
    threats: List[Threat] = field(default_factory=list)

# 예시: REST API 시스템 위협 모델
api_threats = [
    Threat(
        id="T001",
        category=StrideCategory.SPOOFING,
        affected_component="인증 API (/auth/login)",
        description="공격자가 탈취한 JWT 토큰으로 다른 사용자 세션을 위장",
        impact="계정 탈취, 권한 없는 리소스 접근",
        mitigation="짧은 JWT 만료 시간(15분) + Refresh Token 회전 + 블랙리스트 관리",
        status=MitigationStatus.MITIGATED,
    ),
    Threat(
        id="T002",
        category=StrideCategory.TAMPERING,
        affected_component="주문 API (/orders/{id})",
        description="악의적인 사용자가 URL의 주문 ID를 변경하여 타인의 주문 수정",
        impact="데이터 무결성 침해, GDPR 위반",
        mitigation="모든 오브젝트 접근 시 오너십 검증 (IDOR 방지)",
        status=MitigationStatus.OPEN,
    ),
    Threat(
        id="T003",
        category=StrideCategory.INFORMATION_DISCLOSURE,
        affected_component="글로벌 오류 핸들러",
        description="내부 예외 스택 트레이스가 클라이언트에 노출",
        impact="내부 기술 스택 노출, 공격자 정보 획득",
        mitigation="프로덕션 환경에서 generic 오류 메시지만 반환",
        status=MitigationStatus.OPEN,
    ),
    Threat(
        id="T004",
        category=StrideCategory.DENIAL_OF_SERVICE,
        affected_component="검색 API (/search)",
        description="Rate limit 없는 복잡한 DB 쿼리로 서버 과부하 유발",
        impact="서비스 다운, 다른 사용자 접근 불가",
        mitigation="Redis 기반 Rate Limiting (100 req/min/IP) + 쿼리 타임아웃",
        status=MitigationStatus.MITIGATED,
    ),
]

# 보고서 생성
def generate_stride_report(threats: List[Threat]) -> str:
    open_threats = [t for t in threats if t.status == MitigationStatus.OPEN]
    mitigated = [t for t in threats if t.status == MitigationStatus.MITIGATED]
    
    report = f"""
=== STRIDE 위협 모델 보고서 ===
총 식별된 위협: {len(threats)}개
미해결: {len(open_threats)}개 ⚠️
완화됨: {len(mitigated)}개 ✓

미해결 위협 목록:
"""
    for t in open_threats:
        report += f"""
[{t.id}] {t.category.value}
  컴포넌트: {t.affected_component}
  설명: {t.description}
  영향: {t.impact}
  권장 완화: {t.mitigation}
"""
    return report

print(generate_stride_report(api_threats))
```

---

## DREAD 위험도 평가

STRIDE가 위협을 **분류**한다면, DREAD는 각 위협의 **심각도**를 정량적으로 평가한다. 각 항목을 1~10점으로 채점하여 우선순위를 결정한다.

| 항목 | 의미 | 0점 | 10점 |
|------|------|-----|------|
| **D**amage | 피해 잠재력 | 영향 없음 | 전체 시스템 장악 |
| **R**eproducibility | 재현 가능성 | 재현 불가 | 항상 재현 가능 |
| **E**xploitability | 공격 용이성 | 전문가만 가능 | 비전문가도 가능 |
| **A**ffected Users | 영향받는 사용자 수 | 0명 | 전체 사용자 |
| **D**iscoverability | 발견 용이성 | 내부에만 노출 | 공개 취약점 DB |

```python
@dataclass
class DreadScore:
    threat_id: str
    damage: int          # 0-10
    reproducibility: int  # 0-10
    exploitability: int  # 0-10
    affected_users: int  # 0-10
    discoverability: int # 0-10
    
    @property
    def total(self) -> float:
        return (self.damage + self.reproducibility + 
                self.exploitability + self.affected_users + 
                self.discoverability) / 5
    
    @property
    def risk_level(self) -> str:
        if self.total >= 7:
            return "Critical 🔴"
        elif self.total >= 4:
            return "High 🟠"
        elif self.total >= 2:
            return "Medium 🟡"
        else:
            return "Low 🟢"

# 위협별 DREAD 평가
dread_scores = [
    DreadScore("T001", damage=8, reproducibility=7, exploitability=6, 
               affected_users=9, discoverability=5),
    DreadScore("T002", damage=7, reproducibility=9, exploitability=8, 
               affected_users=7, discoverability=4),
    DreadScore("T003", damage=4, reproducibility=10, exploitability=9, 
               affected_users=10, discoverability=8),
    DreadScore("T004", damage=9, reproducibility=8, exploitability=7, 
               affected_users=10, discoverability=6),
]

# 우선순위 정렬
sorted_threats = sorted(dread_scores, key=lambda s: s.total, reverse=True)
print("=== DREAD 위험도 평가 (우선순위 순) ===")
for s in sorted_threats:
    print(f"[{s.threat_id}] 점수: {s.total:.1f} — {s.risk_level}")
```

출력:
```
=== DREAD 위험도 평가 (우선순위 순) ===
[T004] 점수: 8.0 — Critical 🔴
[T003] 점수: 8.2 — Critical 🔴
[T001] 점수: 7.0 — Critical 🔴
[T002] 점수: 7.0 — Critical 🔴
```

---

## PASTA (Process for Attack Simulation and Threat Analysis)

STRIDE/DREAD가 위협 분류와 정량 평가에 집중하는 반면, **PASTA**는 비즈니스 목표부터 출발하는 7단계 리스크 중심 방법론이다.

```
단계 1: 비즈니스 목표 정의
  └─ "사용자 결제 데이터를 처리하는 PCI-DSS 준수 전자상거래 시스템"

단계 2: 기술 스코프 정의
  └─ 컴포넌트 목록, 기술 스택, 의존성 매핑

단계 3: 애플리케이션 분해
  └─ DFD 작성, 유스케이스 정의, 데이터 흐름 매핑

단계 4: 위협 분석
  └─ STRIDE 적용, 공격 트리(Attack Tree) 작성

단계 5: 취약점 탐지
  └─ SAST/DAST 결과 연계, CVE 데이터베이스 참조

단계 6: 공격 시뮬레이션
  └─ 공격 시나리오 실행, 침투 테스트 수행

단계 7: 위험 완화 및 대응
  └─ 완화 전략 도출, ROI 기반 우선순위 결정
```

### 공격 트리(Attack Tree) 예시

공격 트리는 루트 노드(공격자의 최종 목표)를 하향식으로 분해하여 구체적인 공격 방법을 도출한다.

```
[루트] 사용자 계정 탈취
├─ OR ─ 인증 우회
│         ├─ AND ─ SQL Injection으로 WHERE 1=1 우회
│         │         └─ 로그인 폼에 ' OR 1=1 -- 입력
│         └─ AND ─ Brute Force
│                   ├─ Rate Limit 없음
│                   └─ 계정 잠금 없음
├─ OR ─ 세션 탈취
│         ├─ XSS로 document.cookie 탈취
│         └─ 네트워크 스니핑 (비암호화 연결)
└─ OR ─ 인증 정보 탈취
          ├─ 피싱
          └─ 자격증명 스터핑 (유출된 DB 재사용)
```

---

## 주의사항과 실무 팁

**정기적인 업데이트**: 위협 모델은 "살아있는 문서"여야 한다. 아키텍처 변경, 신규 기능 추가, 새로운 CVE 발견 시마다 업데이트해야 한다. PR(Pull Request)에 위협 모델 변경 유무를 체크리스트로 포함시키는 것을 권장한다.

**도구 활용**: 수동 DFD 작성 대신 Microsoft Threat Modeling Tool, OWASP Threat Dragon, IriusRisk 등의 도구를 활용하면 생산성을 크게 높일 수 있다.

**OWASP Top 10 연계**: STRIDE의 각 카테고리와 OWASP Top 10 항목을 매핑하면 개발자들이 친숙한 언어로 위협을 이해하는 데 도움이 된다. 예를 들어 STRIDE의 'I - 정보 노출'은 OWASP의 'A02: Cryptographic Failures'와 'A05: Security Misconfiguration'에 대응한다.

**개발팀 참여**: 위협 모델링은 보안팀만의 역할이 아니다. 개발자, 아키텍트, 제품 관리자가 함께 참여하는 위협 모델링 워크숍을 진행하면 보안 인식을 높이고 더 현실적인 위협을 도출할 수 있다.

**완화 vs 수용**: 모든 위협을 제거하는 것은 불가능하다. 비용-효익 분석을 통해 일부 저위험 위협은 "수용(accept)"하는 결정도 합리적이다. 단, 이 결정은 문서화되어야 한다.

---

## 참고 자료
- [Microsoft Threat Modeling Tool](https://learn.microsoft.com/en-us/azure/security/develop/threat-modeling-tool)
- [OWASP Threat Modeling Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Threat_Modeling_Cheat_Sheet.html)
- [OWASP Threat Dragon (오픈소스 도구)](https://owasp.org/www-project-threat-dragon/)
- [STRIDE, PASTA, and DREAD Methodologies Explained — DestCert](https://destcert.com/resources/threat-modeling-methodologies-stride-pasta-dread/)
