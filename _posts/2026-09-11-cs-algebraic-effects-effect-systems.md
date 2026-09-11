---
layout: post
title: "Algebraic Effects 완전 정복: 부수 효과를 타입 시스템으로 제어하는 새로운 패러다임"
date: 2026-09-11
categories: [cs, computer-science]
tags: [algebraic-effects, effect-systems, functional-programming, koka, ocaml, type-theory, concurrency]
---

함수형 프로그래밍에서 오랫동안 가장 어렵게 다뤄진 문제 중 하나가 바로 **부수 효과(Side Effect)**입니다. IO, 상태 변경, 예외, 비동기 처리 등은 모두 부수 효과이며, 순수 함수형 언어에서는 이를 모나드(Monad)로 처리해 왔습니다. 하지만 모나드는 조합이 어렵고 코드가 복잡해지는 단점이 있습니다. **Algebraic Effects(대수적 효과)**는 이 문제를 근본적으로 다른 방식으로 해결하는 새로운 프로그래밍 패러다임입니다. OCaml 5가 주류 언어 최초로 이를 채택하고, Koka 언어가 이를 핵심 설계 원칙으로 삼으면서 주목받고 있습니다.

## Algebraic Effects란 무엇인가

Algebraic Effects는 **부수 효과를 일급 시민(First-Class Citizen)으로 다루는 메커니즘**입니다. 프로그램이 수행하는 효과(예외, IO, 상태 변경, 난수 생성, 비동기 연산 등)를 **타입 수준에서 명시적으로 선언**하고, 이 효과의 처리 방법을 **호출 지점에서 분리**할 수 있게 합니다.

### 핵심 개념: Effect, Operation, Handler

```
전통적인 예외 처리:
  throw new Exception()  ←→  catch(Exception e) { ... }
  [효과 발생]               [효과 처리, 단 방향: 돌아올 수 없음]

Algebraic Effects:
  perform Op()          ←→  handle { Op() → resume(...) }
  [효과 발생]               [효과 처리, 양방향: resume으로 복귀 가능!]
```

핵심적인 차이는 **재개 가능성(Resumability)**입니다. 예외를 던지면 해당 스택 프레임으로 돌아올 수 없지만, Algebraic Effects에서는 효과를 처리한 핸들러가 `resume`을 통해 효과를 발생시킨 지점으로 값을 돌려주고 실행을 계속할 수 있습니다.

이것이 **한정적 계속(Delimited Continuation)**과 연결되는 지점입니다. 효과 발생 시점부터 핸들러 경계까지의 계속(continuation)을 캡처하고, 핸들러가 이 계속을 선택적으로 실행합니다.

### 왜 이것이 강력한가: 예외 vs Effects

```
예외(Exception):                    Algebraic Effects:
                                   
  foo()                              foo()
    │                                  │
    ↓                                  ↓
  throw Exception      ─────────→  perform ReadLine
    │ (스택 해제, 복귀 불가)          │
    │                                  │ (계속이 캡처됨)
    ↓                                  ↓
  catch { ... }                    handle {
  [여기서만 처리 가능]                ReadLine → {
                                       let input = getInput()
                                       resume(input)  // ← 핵심!
                                     }
                                   }
```

## 왜 Algebraic Effects가 필요한가

### 1. 모나드의 조합 문제

Haskell에서 여러 부수 효과를 조합하려면 모나드 트랜스포머(Monad Transformer)가 필요합니다. 상태(State), IO, 예외(Exception)를 동시에 사용하면:

```haskell
-- Haskell: 3가지 효과를 조합하면 타입이 복잡해짐
type App a = ExceptT AppError (StateT AppState IO) a

-- 각 효과를 사용할 때마다 lift가 필요
processOrder :: App ()
processOrder = do
  order <- lift (lift getNextOrder)  -- IO: lift 두 번!
  state <- lift get                  -- StateT: lift 한 번
  when (isInvalid order state) $
    throwError InvalidOrder          -- ExceptT: lift 불필요
```

Algebraic Effects에서는 이런 `lift` 지옥이 없습니다.

### 2. 표현력과 추상화

Algebraic Effects를 사용하면 예외, 상태, 비동기, 제너레이터, 로그, 난수 등 **모든 부수 효과를 통일된 방식으로 표현**할 수 있습니다.

### 3. 테스트 용이성

효과의 구체적 구현을 핸들러에서 제공하므로, 테스트 시 실제 IO 대신 목(Mock) 핸들러를 손쉽게 삽입할 수 있습니다.

## Koka로 배우는 Algebraic Effects

Koka는 Algebraic Effects를 핵심 언어 기능으로 지원하는 함수형 언어입니다. Microsoft Research에서 개발했으며, 효과 타입이 함수 시그니처에 자동으로 추론됩니다.

```koka
// Koka: 효과 선언
effect emit<a>
  ctl emit(value: a): ()   // 값을 방출하는 효과 연산

// 효과를 사용하는 함수: 타입에 <emit<int>> 가 자동 추론됨
fun count-to(n: int): <emit<int>> ()
  for(1, n) fn(i)
    emit(i)   // emit 효과 발생

// 핸들러 1: 방출된 값들을 리스트로 수집
fun collect(action: () -> <emit<a>|e> ()): <|e> list<a>
  var result := []
  handle action
    ctl emit(v)
      result := Cons(v, result)
      resume(())   // 원래 위치로 재개
  result.reverse

// 핸들러 2: 방출된 값들을 출력
fun print-emit(action: () -> <emit<int>|e> ()): <io|e> ()
  handle action
    ctl emit(v)
      println(v.show)
      resume(())   // 원래 위치로 재개

// 사용 예시
fun main()
  // 같은 count-to 함수를 다른 핸들러로 처리
  val nums = collect { count-to(5) }
  println(nums.show)           // [1,2,3,4,5]

  print-emit { count-to(3) }   // 1 \n 2 \n 3

  // 핸들러 조합: 짝수만 수집
  val evens = collect {
    handle { count-to(10) }
      ctl emit(v)
        if v % 2 == 0 then
          emit(v)    // 짝수는 상위 핸들러로 전달
        resume(())   // 홀수는 그냥 재개 (흡수)
  }
  println(evens.show) // [2,4,6,8,10]
```

**타입이 자동으로 전파됩니다**: `count-to`가 `emit<int>` 효과를 사용하므로, 이 함수를 호출하는 모든 곳에서도 이 효과가 타입에 포함됩니다. 핸들러로 감싸면 효과가 타입에서 제거됩니다. 컴파일러가 이 모든 것을 자동으로 추론합니다.

## OCaml 5의 Effect Handlers

OCaml 5.0에서 Algebraic Effects가 주류 언어 최초로 공식 도입되었습니다. OCaml 5는 현재 동적 타입 효과(untyped effects)를 지원하며, 이를 기반으로 강력한 동시성 라이브러리들이 구축되고 있습니다.

```ocaml
(* OCaml 5: 효과 선언 *)
type _ Effect.t +=
  | Read : string Effect.t          (* 사용자 입력 읽기 효과 *)
  | Write : string -> unit Effect.t (* 출력 효과 *)
  | GetState : int Effect.t         (* 상태 읽기 *)
  | SetState : int -> unit Effect.t (* 상태 쓰기 *)

(* 효과를 사용하는 함수 *)
let process_input () =
  let input = Effect.perform Read in
  let len = String.length input in
  let () = Effect.perform (SetState len) in
  Effect.perform (Write (Printf.sprintf "입력 길이: %d" len))

(* 실제 IO 핸들러 *)
let run_with_io action =
  match_with action ()
    { retc = (fun x -> x)  (* 정상 반환 *)
    ; exnc = raise           (* 예외 전파 *)
    ; effc = fun (type a) (eff : a Effect.t) ->
        match eff with
        | Read ->
          Some (fun (k : (a, _) Effect.Deep.continuation) ->
            let input = input_line stdin in
            Effect.Deep.continue k input)  (* resume! *)
        | Write msg ->
          Some (fun k ->
            print_endline msg;
            Effect.Deep.continue k ())
        | GetState ->
          Some (fun k ->
            Effect.Deep.continue k !global_state)
        | SetState v ->
          Some (fun k ->
            global_state := v;
            Effect.Deep.continue k ())
        | _ -> None  (* 처리 못하면 상위로 전파 *)
    }

(* 테스트용 목(Mock) 핸들러 *)
let run_with_mock ?(inputs=["hello"]) action =
  let inputs = ref inputs in
  let outputs = ref [] in
  let state = ref 0 in
  match_with action ()
    { retc = (fun _ -> !outputs)
    ; exnc = raise
    ; effc = fun (type a) (eff : a Effect.t) ->
        match eff with
        | Read ->
          Some (fun k ->
            match !inputs with
            | [] -> Effect.Deep.continue k ""
            | h :: t ->
              inputs := t;
              Effect.Deep.continue k h)
        | Write msg ->
          Some (fun k ->
            outputs := msg :: !outputs;
            Effect.Deep.continue k ())
        | GetState ->
          Some (fun k -> Effect.Deep.continue k !state)
        | SetState v ->
          Some (fun k ->
            state := v;
            Effect.Deep.continue k ())
        | _ -> None
    }

(* 사용: 동일한 비즈니스 로직, 다른 핸들러 *)
let () =
  (* 실제 실행 *)
  run_with_io process_input;
  
  (* 테스트 실행 - 실제 IO 없음! *)
  let mock_outputs = run_with_mock process_input in
  List.iter print_endline (List.rev mock_outputs)
```

### Algebraic Effects로 코루틴 구현

Algebraic Effects의 진정한 힘은 **비동기/동시성 추상화를 라이브러리 수준에서 구현**할 수 있다는 점입니다.

```ocaml
(* Algebraic Effects로 협력적 멀티태스킹 구현 *)
type _ Effect.t += Yield : unit Effect.t
                 | Fork : (unit -> unit) -> unit Effect.t

(* 간단한 협력적 스케줄러 *)
let scheduler main_task =
  let queue = Queue.create () in
  
  let rec run task =
    match_with task ()
      { retc = (fun () ->
          (* 태스크 완료, 다음 태스크 실행 *)
          if not (Queue.is_empty queue) then
            run (Queue.pop queue))
      ; exnc = raise
      ; effc = fun (type a) (eff : a Effect.t) ->
          match eff with
          | Yield ->
            Some (fun (k : (unit, _) Effect.Deep.continuation) ->
              (* 현재 태스크를 큐에 넣고 다음 태스크로 *)
              Queue.push (fun () -> Effect.Deep.continue k ()) queue;
              if not (Queue.is_empty queue) then
                run (Queue.pop queue))
          | Fork child_task ->
            Some (fun k ->
              (* 자식 태스크를 큐에 추가 *)
              Queue.push child_task queue;
              Effect.Deep.continue k ())
          | _ -> None
      }
  in
  run main_task

(* 사용 예시: async/await 없이도 협력적 실행 *)
let producer_consumer () =
  let buffer = ref [] in
  
  let producer () =
    for i = 1 to 5 do
      buffer := i :: !buffer;
      Printf.printf "Produced: %d\n%!" i;
      Effect.perform Yield  (* 소비자에게 양보 *)
    done
  in
  
  let consumer () =
    for _ = 1 to 5 do
      Effect.perform Yield;  (* 생산자에게 양보 *)
      match !buffer with
      | h :: t ->
        buffer := t;
        Printf.printf "Consumed: %d\n%!" h
      | [] -> ()
    done
  in
  
  (* Fork로 태스크 생성, 스케줄러가 협력적으로 실행 *)
  Effect.perform (Fork consumer);
  producer ()

let () = scheduler producer_consumer
```

출력:
```
Produced: 1
Consumed: 1
Produced: 2
Consumed: 2
...
```

이 코드에서 async/await, 스레드, 콜백이 전혀 없습니다. Algebraic Effects만으로 협력적 멀티태스킹이 구현되었습니다.

## Effect Systems: 타입 수준에서의 효과 추적

Effect System은 **함수의 효과를 타입에 포함**시키는 타입 시스템 확장입니다. Koka에서는 이것이 자동으로 추론됩니다.

```koka
// Koka 효과 타입의 예시

// 순수 함수: 효과 없음
fun add(x: int, y: int): int
  x + y

// IO 효과가 있는 함수
fun greet(name: string): io ()
  println("Hello, " ++ name)

// 실패 가능한 함수 (exn 효과)
fun divide(x: int, y: int): exn int
  if y == 0 then throw("Division by zero")
  else x / y

// 비결정론적 함수 (nondet 효과)
fun coin-flip(): nondet bool
  amb()  // 비결정론적으로 true/false 반환

// 여러 효과 조합
fun complex(name: string): <io, exn> int
  greet(name)  // io 효과
  divide(10, 0)  // exn 효과

// 효과가 자동으로 타입에 전파됨
// complex의 타입은 <io, exn> int (컴파일러가 자동 추론)
```

## 현실 언어에서의 현황

| 언어 | 상태 | 특징 |
|------|------|------|
| **Koka** | 연구용 언어, 안정 릴리스 | 효과 타입 완전 지원, Perceus GC |
| **OCaml 5** | 프로덕션 사용 가능 | 동적 타입 효과, Domain 기반 병렬성 |
| **Multicore OCaml** | OCaml 5에 통합 | 작업 스틸링 스케줄러 |
| **Eff** | 연구용 | 최초의 효과 언어 중 하나 |
| **Effekt** | 연구용 | 정적 타입 효과, Scala 유사 문법 |
| **Unison** | 실험적 | 분산 시스템을 위한 효과 |
| **Scala 3** | 제한적 지원 | CPS 기반 실험적 구현 |
| **Java/Kotlin** | 미지원 | 구조화된 동시성(StructuredConcurrency)으로 일부 대체 |

## 주의사항 및 팁

### 1. 성능: Stack Overflow와 Tail Resume

Algebraic Effects는 내부적으로 계속(continuation)을 캡처하므로, 많은 구현에서 힙 할당이 발생합니다. OCaml 5는 이를 매우 효율적으로 구현했지만, 깊은 재귀적 효과 사용은 성능에 영향을 줄 수 있습니다.

**Tail Resume 최적화**: 핸들러에서 `resume`이 마지막 연산이면(tail position), 스택 할당 없이 최적화될 수 있습니다. 이를 의식하며 핸들러를 작성하면 성능을 개선할 수 있습니다.

### 2. 핸들러 순서가 의미론을 바꾼다

핸들러는 중첩 순서에 따라 의미가 달라집니다. 상태(State)와 예외(Exception) 핸들러를 어떤 순서로 감싸느냐에 따라 예외 발생 시 상태가 롤백될지 여부가 결정됩니다.

```ocaml
(* State outside Exception: 예외 발생 시 상태 보존 *)
run_with_state (run_with_exception action)

(* Exception outside State: 예외 발생 시 상태 롤백 *)
run_with_exception (run_with_state action)
```

이는 모나드 트랜스포머의 순서 문제와 정확히 동일한 이슈입니다. Algebraic Effects도 이 문제를 완전히 해결하지는 않으며, 올바른 핸들러 순서를 프로그래머가 이해해야 합니다.

### 3. 효과의 전파와 누락

핸들러가 특정 효과를 처리하지 않으면 상위로 전파됩니다. 최상위까지 전파되어도 처리되지 않으면 런타임 오류가 발생합니다(정적 타입 시스템이 있는 언어에서는 컴파일 오류). 모든 효과가 반드시 어딘가에서 처리됨을 보장하는 것이 Effect System의 핵심 가치입니다.

### 4. 기존 언어와의 통합

대부분의 주류 언어는 Algebraic Effects를 지원하지 않습니다. 그러나 아이디어는 빌려올 수 있습니다:

- **Java/Kotlin**: `CoroutineScope`, `structured concurrency`가 Effects의 일부 아이디어를 구현
- **Haskell**: `fused-effects`, `polysemy` 등 라이브러리로 모나드 기반 에뮬레이션
- **Rust**: `async/await` + `tower` 미들웨어 패턴이 효과 계층화와 유사

## 참고 자료
- [Koka Language GitHub](https://github.com/koka-lang/koka)
- [OCaml Multicore Effects Tutorial](https://github.com/ocaml-multicore/ocaml-effects-tutorial)
