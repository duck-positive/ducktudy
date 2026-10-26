---
layout: post
title: "서버리스 FaaS 콜드 스타트 심화 분석과 최적화 전략"
date: 2026-10-26
categories: [cs, computer-science]
tags: [serverless, faas, cold-start, aws-lambda, optimization, cloud-computing]
---

## 서버리스와 FaaS란 무엇인가

서버리스(Serverless) 컴퓨팅은 개발자가 서버 인프라를 직접 관리하지 않고 함수 단위로 코드를 실행할 수 있는 클라우드 실행 모델입니다. Function-as-a-Service(FaaS)는 서버리스의 핵심 구현 형태로, AWS Lambda, Google Cloud Functions, Azure Functions 같은 플랫폼이 대표적입니다.

FaaS의 핵심 가치는 **이벤트 기반 실행**과 **자동 스케일링**입니다. 요청이 없을 때는 리소스를 소비하지 않고, 트래픽이 급증하면 수천 개의 인스턴스를 자동으로 생성합니다. 사용한 만큼만 비용을 지불하는 과금 모델은 비용 효율성을 극대화합니다.

그러나 이 모델에는 치명적인 단점이 하나 존재합니다. 바로 **콜드 스타트(Cold Start)** 문제입니다.

## 콜드 스타트는 왜 발생하는가

함수 인스턴스의 생명주기를 이해하면 콜드 스타트의 원인이 명확해집니다.

### 실행 컨텍스트 생명주기

FaaS 플랫폼은 함수 인스턴스를 **Warm**, **Cold**, **Frozen** 세 가지 상태로 관리합니다.

1. **Cold 상태**: 인스턴스가 전혀 없는 상태. 첫 요청이나 오랜 유휴 후 새로운 컨테이너를 생성해야 합니다.
2. **Warm 상태**: 이전 요청을 처리한 인스턴스가 재사용 대기 중인 상태. 초기화 오버헤드 없이 즉시 실행됩니다.
3. **Frozen 상태**: Warm 인스턴스가 메모리에 일시 정지된 상태. 재사용될 때 즉시 활성화됩니다.

콜드 스타트는 Cold 상태에서 첫 번째 요청을 처리할 때 발생합니다. 플랫폼이 새 컨테이너를 부팅하고, 런타임을 초기화하고, 사용자 코드를 로드하는 전 과정이 필요하기 때문입니다.

### 콜드 스타트 비용의 구성 요소

AWS Lambda 기준으로 콜드 스타트는 다음 단계로 구성됩니다:

```
[Cold Start Timeline]

1. 컨테이너 생성 및 부팅           (100ms ~ 500ms)
   └─ microVM(Firecracker) 시작
   └─ 컨테이너 이미지 레이어 마운트

2. 런타임 초기화                    (50ms ~ 300ms)
   └─ JVM, Node.js, Python 인터프리터 시작
   └─ 표준 라이브러리 로딩

3. 함수 패키지 다운로드/추출        (100ms ~ 1000ms)
   └─ 배포 패키지 또는 컨테이너 이미지 로드

4. 핸들러 초기화                    (50ms ~ 수초)
   └─ 글로벌 초기화 코드 실행
   └─ DB 커넥션, SDK 초기화

5. 핸들러 실행 (실제 비즈니스 로직)
```

단계 4까지가 콜드 스타트 오버헤드이며, 이 총합이 수백 밀리초에서 수 초에 이를 수 있습니다.

## 실제 구현 예제

### 예제 1: 콜드/웜 스타트 측정 코드 (Node.js)

```javascript
// AWS Lambda Handler - Cold Start 측정
const startTime = Date.now();  // 모듈 로드 시점 기록 (글로벌)

// 무거운 초기화: DB 클라이언트, SDK 등
let dbClient = null;
let isInitialized = false;
const initStartTime = Date.now();

function initializeExpensiveResources() {
  if (isInitialized) return;
  
  // 실제 초기화 (예: DynamoDB 클라이언트)
  const { DynamoDBClient } = require('@aws-sdk/client-dynamodb');
  dbClient = new DynamoDBClient({ region: 'ap-northeast-2' });
  
  isInitialized = true;
  console.log(`Init time: ${Date.now() - initStartTime}ms`);
}

// Lazy initialization: 첫 호출 시에만 초기화
exports.handler = async (event) => {
  const invokeTime = Date.now();
  const isColdStart = !isInitialized;
  
  initializeExpensiveResources();
  
  const initDuration = invokeTime - startTime;
  
  // X-Ray나 CloudWatch로 측정값 기록
  console.log(JSON.stringify({
    coldStart: isColdStart,
    initDurationMs: initDuration,
    requestId: event.requestContext?.requestId
  }));
  
  // 비즈니스 로직 실행
  const result = await processRequest(event);
  
  return {
    statusCode: 200,
    body: JSON.stringify({
      result,
      metadata: { coldStart: isColdStart }
    })
  };
};

async function processRequest(event) {
  // 실제 처리 로직
  return { message: 'processed', timestamp: new Date().toISOString() };
}
```

핵심은 `dbClient` 초기화를 **글로벌 스코프**에서 **핸들러 최초 호출 시점**으로 미루는 것입니다. 이를 **지연 초기화(Lazy Initialization)**라고 합니다.

### 예제 2: Python Lambda 최적화 기법 모음

```python
import json
import os
import time
import logging
from functools import lru_cache
from typing import Optional

# 모듈 수준 캐시 - Warm 재사용 시 유지됨
_db_connection: Optional[object] = None
_config_cache: Optional[dict] = None
COLD_START_TIME = time.time()

logger = logging.getLogger()
logger.setLevel(logging.INFO)

# 1. 환경 변수 캐싱: os.environ 호출 최소화
@lru_cache(maxsize=None)
def get_config():
    """환경 변수를 한 번만 읽고 캐싱"""
    return {
        'db_host': os.environ.get('DB_HOST', 'localhost'),
        'db_port': int(os.environ.get('DB_PORT', '5432')),
        'cache_ttl': int(os.environ.get('CACHE_TTL', '300')),
    }

def get_db_connection():
    """2. Connection Reuse: 전역 커넥션 재사용"""
    global _db_connection
    if _db_connection is None:
        config = get_config()
        # 실제로는 psycopg2 또는 SQLAlchemy 등 사용
        logger.info(f"Creating new DB connection to {config['db_host']}")
        _db_connection = create_connection(config)
    return _db_connection

def create_connection(config: dict):
    # DB 연결 시뮬레이션
    time.sleep(0.01)  # 실제 연결 대기 시간
    return {'host': config['db_host'], 'connected': True}

def lambda_handler(event, context):
    """3. 핸들러 최적화 - 불필요한 임포트 제거, 코드 경로 단축"""
    is_cold_start = (time.time() - COLD_START_TIME) < 2.0
    
    try:
        # 4. 조기 반환(Early Return): 불필요한 초기화 건너뜀
        if event.get('source') == 'aws.events':
            # 웜업 이벤트 - 초기화만 수행하고 반환
            get_db_connection()
            return {'statusCode': 200, 'body': 'warmed up'}
        
        conn = get_db_connection()
        result = process_business_logic(event, conn)
        
        logger.info(json.dumps({
            'cold_start': is_cold_start,
            'remaining_ms': context.get_remaining_time_in_millis()
        }))
        
        return {
            'statusCode': 200,
            'body': json.dumps(result)
        }
    
    except Exception as e:
        logger.error(f"Error: {str(e)}", exc_info=True)
        # 5. 오류 시 커넥션 재설정
        global _db_connection
        _db_connection = None
        raise

def process_business_logic(event: dict, conn: dict) -> dict:
    # 실제 비즈니스 로직
    return {
        'status': 'success',
        'connection': conn['host']
    }
```

## 콜드 스타트 최소화 전략

### 1. Provisioned Concurrency (프로비저닝된 동시성)

AWS Lambda의 프로비저닝된 동시성은 미리 워밍업된 인스턴스를 지정 개수만큼 상시 유지합니다. 콜드 스타트를 완전히 제거할 수 있지만 비용이 발생합니다.

```yaml
# AWS SAM 예시
Resources:
  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: index.handler
      Runtime: nodejs20.x
      AutoPublishAlias: live
      ProvisionedConcurrencyConfig:
        ProvisionedConcurrentExecutions: 5  # 항상 5개 워밍업 유지
```

### 2. 배포 패키지 경량화

패키지 크기가 클수록 코드 로딩 시간이 길어집니다.

- **Tree Shaking**: esbuild, webpack으로 사용하지 않는 코드 제거
- **Lazy Import**: 핸들러 글로벌이 아닌 필요한 시점에 `require()`
- **Layer 활용**: 공통 의존성은 Lambda Layer로 분리하여 캐싱 활용
- **Python**: 반드시 필요한 패키지만 포함, `boto3`는 기본 제공이므로 배포 패키지에서 제외

### 3. 런타임 선택 전략

런타임별 콜드 스타트 특성이 다릅니다:

| 런타임 | 콜드 스타트 | 특징 |
|--------|-------------|------|
| Node.js 20.x | 빠름 (~100ms) | V8 스냅샷 덕분에 초기화 빠름 |
| Python 3.12 | 빠름 (~200ms) | 인터프리터 시작 빠름 |
| Java 21 (GraalVM) | 중간 | Native Image로 JVM 부팅 제거 |
| Java 21 (JVM) | 느림 (~1s+) | JVM 부팅 비용 존재 |
| Go | 매우 빠름 | 단일 바이너리, 런타임 없음 |

### 4. SnapStart (Java)

AWS Lambda SnapStart는 초기화 후 상태를 스냅샷으로 저장하고 재사용합니다. Java 함수에서 최대 10배 콜드 스타트 감소 효과를 제공합니다.

## 주의사항 및 팁

**1. 글로벌 상태 오염 주의**  
Warm 재사용 시 이전 호출의 전역 변수가 남아있을 수 있습니다. 민감 데이터나 요청별로 달라야 하는 값은 반드시 핸들러 내부 로컬 변수로 관리하세요.

**2. DB 커넥션 풀 사이즈 제한**  
Lambda 인스턴스가 급격히 스케일아웃될 때 각 인스턴스가 DB 커넥션을 물고 있으면 DB 커넥션 한계를 초과할 수 있습니다. RDS Proxy를 사용하면 커넥션 풀링을 Lambda 외부에서 관리할 수 있습니다.

**3. 콜드 스타트 모니터링**  
AWS CloudWatch Metrics의 `Init Duration` 지표로 실제 콜드 스타트 빈도와 시간을 모니터링하세요. 비즈니스 SLA가 있는 함수는 반드시 추적이 필요합니다.

**4. 웜업 스케줄러의 한계**  
EventBridge로 주기적으로 함수를 호출해 Warm 상태를 유지하는 방법은 비용 효율적이지만, 여러 인스턴스를 동시에 워밍업할 수 없고 정확성이 보장되지 않습니다. Provisioned Concurrency가 더 확실합니다.

**5. SnapStart와 랜덤성**  
SnapStart를 사용할 때 UUID 생성, 난수 시드 등이 스냅샷 시점에 고정될 수 있습니다. `@SnapshotHook` 인터페이스로 스냅샷 복원 시 재초기화 로직을 정의해야 합니다.

서버리스 아키텍처는 적절한 최적화 전략과 함께 사용할 때 비용 효율성과 개발 생산성을 동시에 달성할 수 있는 강력한 패러다임입니다.

## 참고 자료
- [AWS Lambda Runtime Environment](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtime-environment.html)
- [AWS Lambda Provisioned Concurrency](https://docs.aws.amazon.com/lambda/latest/dg/provisioned-concurrency.html)
- [AWS Lambda SnapStart](https://docs.aws.amazon.com/lambda/latest/dg/snapstart.html)
- [Google Cloud Functions Execution Environment](https://cloud.google.com/functions/docs/concepts/execution-environment)
