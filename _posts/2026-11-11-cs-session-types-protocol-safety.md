---
layout: post
title: "Session Types: 타입으로 보장하는 프로토콜 안전성과 데드락 자유"
date: 2026-11-11
categories: [cs, computer-science]
tags: [session-types, type-system, concurrency, protocol, rust, deadlock-freedom, pi-calculus]
---

분산 시스템과 동시 프로그래밍에서 가장 골치 아픈 버그 중 하나는 **프로토콜 위반**입니다. 클라이언트가 서버에게 메시지를 보내는 순서가 틀렸거나, 서버가 보내야 할 응답을 빠뜨렸거나, 양쪽이 서로 메시지를 기다리는 데드락에 빠지는 경우가 대표적입니다. 이런 버그는 런타임에 비결정적으로 발생하고 재현이 어려우며, 단위 테스트로도 완전히 잡기 어렵습니다. **Session Types**는 이러한 통신 프로토콜의 올바름을 타입 수준에서 정적으로 보장하는 타입 이론입니다.

## Session Types란?

Session Types는 1994년 Kohei Honda가 π-계산법(π-calculus)을 기반으로 제안한 타입 시스템입니다. 핵심 아이디어는 **채널(channel)에 타입을 부여하여 그 채널을 통해 주고받는 메시지의 순서와 방향을 명세한다**는 것입니다.

이진 세션 타입(Binary Session Types)에서 각 채널은 두 개의 쌍대 타입(dual type)을 가집니다. 예를 들어 클라이언트 쪽 채널의 타입이 `!Int . ?String . End`(정수를 보내고, 문자열을 받고, 종료)라면, 서버 쪽 채널의 타입은 자동으로 `?Int . !String . End`(정수를 받고, 문자열을 보내고, 종료)가 됩니다. 양쪽이 서로 반대되는 행동을 하도록 타입이 보장하므로, 프로토콜 불일치가 불가능해집니다.

### 세션 타입의 주요 구성 요소

| 구성 요소 | 표기 | 의미 |
|-----------|------|------|
| 송신 | `!T . S` | 타입 T 값을 보내고 세션 S를 계속 |
| 수신 | `?T . S` | 타입 T 값을 받고 세션 S를 계속 |
| 내부 선택 | `⊕{l: S}` | 레이블 l을 선택하고 세션 S를 계속 |
| 외부 선택 | `&{l: S}` | 상대방의 선택에 따라 분기 |
| 종료 | `End` | 세션 완료 |
| 재귀 | `μX. S` | 반복적인 프로토콜 표현 |

### 왜 필요한가?

**기존 방식의 문제점**: 일반적인 채널(예: `mpsc::channel<Box<dyn Any>>`)은 무엇을 보낼지, 순서는 어떻게 되는지를 타입이 전혀 보장하지 않습니다. 프로토콜 명세는 주석이나 문서에만 존재하고, 위반 여부는 런타임에만 알 수 있습니다.

**Session Types의 장점**:
1. **데드락 자유 보장**: 쌍대 타입 덕분에 두 프로세스가 서로 기다리는 상황이 타입 체계상 불가능합니다.
2. **통신 오류 방지**: 잘못된 메시지 타입 송신, 잘못된 순서, 누락된 메시지를 컴파일 타임에 감지합니다.
3. **자기 문서화**: 세션 타입 자체가 프로토콜 명세 역할을 합니다.
4. **선형성(Linearity)**: 채널을 "한 번만 사용"하도록 강제하여 재사용으로 인한 프로토콜 혼란을 방지합니다.

## 코드 예제 1: Rust의 session_types 라이브러리로 계산기 프로토콜 구현

```rust
// Cargo.toml에 추가:
// session_types = "0.3"
//
// 간단한 덧셈 계산기 프로토콜:
// 클라이언트: i32 보내기 → i32 보내기 → i32 받기 → 완료
// 서버: i32 받기 → i32 받기 → i32 보내기 → 완료

use session_types::*;

// 프로토콜 타입 정의 (클라이언트 관점)
// !i32 → !i32 → ?i32 → End
type CalculatorClient = Send<i32, Send<i32, Recv<i32, Eps>>>;

// 서버 타입은 클라이언트의 쌍대 (자동으로 결정됨)
// ?i32 → ?i32 → !i32 → End
type CalculatorServer = <CalculatorClient as HasDual>::Dual;

fn server(c: Chan<(), CalculatorServer>) {
    // 첫 번째 피연산자 받기
    let (c, x) = c.recv();
    // 두 번째 피연산자 받기
    let (c, y) = c.recv();
    // 합계 보내기
    let c = c.send(x + y);
    // 세션 완료
    c.close();
}

fn client(c: Chan<(), CalculatorClient>) {
    // 첫 번째 피연산자 보내기
    let c = c.send(10);
    // 두 번째 피연산자 보내기
    let c = c.send(32);
    // 결과 받기
    let (c, result) = c.recv();
    println!("10 + 32 = {}", result);  // 출력: 10 + 32 = 42
    // 세션 완료
    c.close();
}

fn main() {
    // connect()는 두 채널의 쌍대 관계를 타입 수준에서 검증한 후 스레드 생성
    connect(server, client);
}
```

만약 클라이언트가 `send` 대신 `recv`를 먼저 호출하거나, 보내는 타입을 `f64`로 바꾸면 **컴파일 오류**가 발생합니다. 프로토콜 위반이 런타임 버그가 아닌 컴파일 오류로 처리됩니다.

## 코드 예제 2: 선택(Choice)이 있는 프로토콜 — 인증 서버

실제 프로토콜에서는 성공/실패 같은 분기가 필요합니다. Session Types의 선택(Choice) 연산자로 이를 표현합니다:

```rust
use session_types::*;

// 인증 프로토콜 (클라이언트 관점):
//   1. 사용자명 보내기
//   2. 비밀번호 보내기
//   3. 서버로부터 선택 받기:
//      - 성공: 환영 메시지 받기 → 완료
//      - 실패: 오류 메시지 받기 → 완료
type AuthClient = Send<String,          // 사용자명 송신
                  Send<String,          // 비밀번호 송신
                  Recv<                 // 서버 선택 수신
                    Choose<
                      Recv<String, Eps>,  // 성공: 환영 메시지
                      Recv<String, Eps>   // 실패: 오류 메시지
                    >,
                    Eps
                  >>>;

// 서버 타입 (자동으로 쌍대 결정)
type AuthServer = <AuthClient as HasDual>::Dual;
// = Recv<String, Recv<String, Send<Offer<Send<String, Eps>, Send<String, Eps>>, Eps>>>

fn auth_server(c: Chan<(), AuthServer>) {
    let (c, username) = c.recv();
    let (c, password) = c.recv();

    if username == "alice" && password == "secret" {
        // 성공 경로 선택
        let c = c.sel1();
        let c = c.send(format!("환영합니다, {}!", username));
        c.close();
    } else {
        // 실패 경로 선택
        let c = c.sel2();
        let c = c.send("인증 실패: 사용자명 또는 비밀번호가 틀렸습니다.".to_string());
        c.close();
    }
}

fn auth_client(c: Chan<(), AuthClient>) {
    let c = c.send("alice".to_string());
    let c = c.send("secret".to_string());

    // 서버의 선택에 따라 분기 처리 (두 경우 모두 처리해야 컴파일 가능)
    offer!(c, {
        // 성공 경우
        Left(c) => {
            let (c, msg) = c.recv();
            println!("성공: {}", msg);
            c.close();
        },
        // 실패 경우
        Right(c) => {
            let (c, msg) = c.recv();
            println!("실패: {}", msg);
            c.close();
        }
    });
}

fn main() {
    connect(auth_server, auth_client);
}
```

`offer!` 매크로를 통해 클라이언트가 서버의 모든 선택지를 반드시 처리하도록 강제합니다. 하나라도 빠뜨리면 컴파일 오류입니다.

## 선형 타입과의 관계

Session Types의 채널은 **선형(linear)** 자원입니다. Rust의 소유권 시스템이 이를 자연스럽게 지원합니다:

- 채널을 `Clone`할 수 없어 두 스레드가 같은 채널을 동시에 사용하는 것이 불가능
- `send`나 `recv` 호출 시 채널 소유권을 소비(consume)하고 새로운 타입의 채널을 반환
- `close()`를 호출하지 않으면(= `End`에 도달하지 않으면) 컴파일러 경고 발생

이 덕분에 채널 "재사용"이나 "버려짐"으로 인한 프로토콜 오염이 불가능합니다.

## 멀티파티 세션 타입 (Multiparty Session Types)

이진 세션 타입이 두 참여자 간 통신을 모델링한다면, **MPST(Multiparty Session Types)**는 세 명 이상의 참여자 간 통신 프로토콜을 모델링합니다. HTTP/2의 클라이언트-프록시-서버 삼각 통신, 분산 트랜잭션의 다자간 2PC 등이 대표적인 사례입니다.

MPST에서는 **전역 타입(Global Type)**으로 전체 프로토콜을 명세하고, 이를 각 참여자별 **로컬 타입(Local Type)**으로 프로젝션(projection)합니다. `mpstthree` Rust 크레이트가 이를 지원합니다.

## 주의사항과 팁

### 1. 재귀적 세션의 복잡성

반복적인 프로토콜(예: "계속해서 요청을 받아 처리")은 재귀적 세션 타입 `μX. ...`으로 표현해야 하는데, 타입이 복잡해져 가독성이 떨어질 수 있습니다. 타입 별칭을 적극 활용하세요.

### 2. 비동기 환경에서의 제한

현재 대부분의 Session Types 라이브러리는 동기적(synchronous) 통신을 기반으로 합니다. 비동기 Rust(`tokio`, `async-std`)와의 통합은 아직 발전 중이며, `ferrite` 같은 별도 라이브러리가 이를 시도하고 있습니다.

### 3. 선형성 강제로 인한 불편함

채널을 복사할 수 없기 때문에, 같은 채널을 여러 함수에 넘겨야 하는 패턴은 구현이 불편할 수 있습니다. 이때는 프로토콜을 더 작은 단위로 분리하는 것을 고려하세요.

### 4. 직렬화와 네트워크 연동

Session Types는 주로 프로세스 내부 채널을 모델링합니다. 실제 네트워크 소켓과 연동하려면 추가 계층이 필요하며, 이 과정에서 타입 보장이 약화될 수 있습니다. `serde`와 직렬화 타입을 함께 사용하면 도움이 됩니다.

### 5. 점진적 도입 전략

기존 코드베이스에 Session Types를 도입할 때는 가장 버그가 많거나 복잡한 프로토콜부터 시작하세요. 타입 설계에 투자하는 초반 비용이 있지만, 이후 디버깅 비용이 크게 줄어듭니다.

Session Types는 아직 주류 프로그래밍에서 널리 쓰이지는 않지만, 분산 시스템·마이크로서비스 아키텍처·안전이 중요한 프로토콜 구현에서 그 가치가 점점 인정받고 있습니다. Rust처럼 선형 타입을 지원하는 언어가 보급되면서 Session Types의 실용적 적용 사례도 늘어날 전망입니다.

## 참고 자료

- [session-types Rust 라이브러리 (GitHub)](https://github.com/Munksgaard/session-types)
- [crossbeam: Rust 동시성 도구 모음](https://github.com/crossbeam-rs/crossbeam)
