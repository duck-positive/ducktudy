---
layout: post
title: "비순서 실행과 토마슬로 알고리즘: 현대 CPU ILP의 핵심"
date: 2026-10-15
categories: [cs, computer-science]
tags: [cpu, out-of-order-execution, tomasulo, ILP, computer-architecture, reservation-station, register-renaming]
---

## 비순서 실행(Out-of-Order Execution)이란?

현대 CPU는 프로그램 코드에 적힌 순서대로 명령어를 실행하지 않는다. **비순서 실행(Out-of-Order Execution, OOoE)**은 CPU가 준비된 명령어를 프로그램 순서와 무관하게 먼저 실행하는 기법으로, 명령어 수준 병렬성(Instruction-Level Parallelism, ILP)을 극대화하는 핵심 메커니즘이다.

이를 가능하게 하는 하드웨어 알고리즘이 바로 **토마슬로 알고리즘(Tomasulo's Algorithm)**이다. 1967년 IBM의 Robert Tomasulo가 IBM System/360 Model 91의 부동소수점 유닛을 위해 개발했으며, 오늘날 모든 고성능 프로세서의 설계 기반이 됐다.

---

## 왜 필요한가: 데이터 해저드 문제

파이프라인 프로세서에서 명령어들이 서로 의존할 때 **데이터 해저드(Data Hazard)**가 발생한다. 해저드의 종류는 세 가지다.

| 해저드 종류 | 설명 | 예시 |
|---|---|---|
| RAW (Read-After-Write) | 진짜 의존성 | `ADD R1, R2, R3` 후 `SUB R4, R1, R5` |
| WAR (Write-After-Read) | 이름 의존성 | `SUB R4, R1, R5` 후 `ADD R1, R2, R3` |
| WAW (Write-After-Write) | 출력 의존성 | `ADD R1, R2, R3` 후 `MUL R1, R4, R5` |

단순한 인순서(in-order) 파이프라인은 RAW 해저드를 만나면 모든 파이프라인이 멈춘다(스톨). WAR와 WAW는 사실 레지스터 이름의 재사용 때문에 생기는 가짜 의존성이다. 토마슬로 알고리즘은 **하드웨어 레지스터 이름 변경(Register Renaming)**으로 WAR·WAW를 제거하고, **예약 스테이션(Reservation Station)**으로 RAW 해저드를 동적으로 처리한다.

---

## 토마슬로 알고리즘의 세 가지 핵심 구조

### 1. 예약 스테이션 (Reservation Station)

각 실행 유닛(ALU, FPU, 로드/스토어 유닛 등) 앞에 위치하는 버퍼다. 명령어가 **발행(Issue)** 되면 예약 스테이션에 들어가며, 피연산자가 아직 준비되지 않으면 대기 상태로 피연산자 태그(어떤 CDB 신호를 기다릴지)를 기록한다.

```
예약 스테이션 엔트리:
+------+-----+----+----+------+------+
| Busy | Op  | Vj | Vk | Qj   | Qk   |
+------+-----+----+----+------+------+
| YES  | ADD | 3  | -  | -    | RS#2 |   ← Vk는 RS#2의 결과를 기다림
+------+-----+----+----+------+------+

Vj, Vk: 실제 피연산자 값 (준비된 경우)
Qj, Qk: 어느 예약 스테이션 결과를 기다리는지 (미준비인 경우)
```

### 2. 공통 데이터 버스 (Common Data Bus, CDB)

실행 유닛이 결과를 내면 CDB를 통해 브로드캐스트한다. 모든 예약 스테이션이 이를 감시하다가, 자신이 기다리던 태그의 결과가 방송되면 즉시 피연산자를 캡처한다. 이것이 **값 포워딩(Value Forwarding)**의 핵심이다.

### 3. 레지스터 결과 상태 테이블 (Register Result Status)

각 아키텍처 레지스터가 어느 예약 스테이션의 결과를 기다리는지 추적한다. 새 명령어를 발행할 때 이 테이블을 참조해 피연산자 값을 직접 읽거나 태그를 예약 스테이션에 기록한다. 이것이 하드웨어 레지스터 이름 변경의 실체다.

---

## 알고리즘의 세 단계: 발행 → 실행 → 기록

```
1. ISSUE (발행):
   - 명령어를 인순서로 예약 스테이션에 배치
   - 피연산자가 레지스터 파일에 있으면 값을 복사
   - 아직 준비 안 됐으면 태그(어느 RS를 기다리는지) 기록
   - 레지스터 결과 상태 테이블 갱신 (이 레지스터는 RS#n이 씀)

2. EXECUTE (실행):
   - 모든 피연산자가 준비(Qj == Qk == 0)되면 실행
   - CDB를 감시해 방송된 결과를 즉시 캡처
   - 로드/스토어는 주소 계산 후 메모리 큐를 통해 처리

3. WRITE RESULT (기록):
   - 실행 완료 후 CDB에 결과 방송
   - 이 태그를 기다리던 모든 예약 스테이션이 값을 캡처
   - 레지스터 파일도 갱신
```

---

## 코드 예제 1: Python으로 시뮬레이션하는 토마슬로 알고리즘

```python
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class ReservationStation:
    name: str
    busy: bool = False
    op: Optional[str] = None
    vj: Optional[int] = None   # 피연산자 j 값
    vk: Optional[int] = None   # 피연산자 k 값
    qj: Optional[str] = None   # 피연산자 j를 기다리는 RS 이름
    qk: Optional[str] = None   # 피연산자 k를 기다리는 RS 이름
    result: Optional[int] = None
    cycles_left: int = 0

    @property
    def ready(self):
        return self.busy and self.qj is None and self.qk is None and self.result is None

class TomasuloSimulator:
    def __init__(self):
        # 예약 스테이션 초기화
        self.rs_add = [ReservationStation(f"ADD{i}") for i in range(3)]
        self.rs_mul = [ReservationStation(f"MUL{i}") for i in range(2)]
        self.all_rs = self.rs_add + self.rs_mul

        # 레지스터 파일 (값)과 레지스터 결과 상태 (어느 RS가 쓸 예정인지)
        self.reg_file = {"R0": 0, "R1": 10, "R2": 20, "R3": 30, "R4": 5}
        self.reg_status = {}  # {레지스터명: RS 이름}

        self.cycle = 0

    def _get_rs_pool(self, op: str):
        return self.rs_add if op in ("ADD", "SUB") else self.rs_mul

    def issue(self, op: str, dest: str, src1: str, src2: str):
        pool = self._get_rs_pool(op)
        rs = next((r for r in pool if not r.busy), None)
        if rs is None:
            print(f"  STALL: 예약 스테이션이 가득 참 ({op} 발행 불가)")
            return False

        rs.busy = True
        rs.op = op
        rs.cycles_left = 2 if op in ("ADD", "SUB") else 4

        # 피연산자 j 처리
        if src1 in self.reg_status:
            rs.qj = self.reg_status[src1]  # 아직 결과 미준비
        else:
            rs.vj = self.reg_file.get(src1, 0)

        # 피연산자 k 처리
        if src2 in self.reg_status:
            rs.qk = self.reg_status[src2]
        else:
            rs.vk = self.reg_file.get(src2, 0)

        # 레지스터 결과 상태 갱신 (이 RS가 dest에 쓸 것임)
        self.reg_status[dest] = rs.name
        rs._dest = dest
        print(f"  ISSUE: {op} {dest}, {src1}, {src2} → {rs.name}")
        return True

    def tick(self):
        self.cycle += 1
        broadcast = {}

        # EXECUTE 단계
        for rs in self.all_rs:
            if rs.ready:
                rs.cycles_left -= 1
                if rs.cycles_left == 0:
                    # 실행 완료
                    if rs.op == "ADD":
                        rs.result = rs.vj + rs.vk
                    elif rs.op == "SUB":
                        rs.result = rs.vj - rs.vk
                    elif rs.op == "MUL":
                        rs.result = rs.vj * rs.vk
                    broadcast[rs.name] = rs.result
                    print(f"  EXEC: {rs.name} 완료, 결과 = {rs.result}")

        # WRITE RESULT (CDB 브로드캐스트)
        for rs_name, value in broadcast.items():
            for rs in self.all_rs:
                if rs.qj == rs_name:
                    rs.vj = value
                    rs.qj = None
                if rs.qk == rs_name:
                    rs.vk = value
                    rs.qk = None
            # 레지스터 파일 갱신
            for dest, waiting_rs in list(self.reg_status.items()):
                if waiting_rs == rs_name:
                    self.reg_file[dest] = value
                    del self.reg_status[dest]
                    print(f"  WRITE: {dest} ← {value}")
            # RS 해제
            for rs in self.all_rs:
                if rs.name == rs_name:
                    rs.busy = False
                    rs.op = None
                    rs.result = None


# 데모 실행
sim = TomasuloSimulator()

print("=== 명령어 발행 ===")
sim.issue("MUL", "R1", "R2", "R3")   # R1 = R2 * R3 (4사이클)
sim.issue("ADD", "R4", "R1", "R2")   # R4 = R1 + R2 (R1에 RAW 의존)
sim.issue("ADD", "R5", "R2", "R3")   # R5 = R2 + R3 (독립적 → 먼저 실행 가능)

print("\n=== 사이클 시뮬레이션 ===")
for _ in range(7):
    print(f"--- 사이클 {sim.cycle + 1} ---")
    sim.tick()

print("\n=== 최종 레지스터 상태 ===")
for reg, val in sim.reg_file.items():
    print(f"  {reg} = {val}")
```

실행 결과를 보면 `R5 = R2 + R3` 연산(R1에 무관)이 MUL보다 먼저 완료된다. `R4 = R1 + R2`는 MUL이 완료되어 CDB가 R1 값을 방송한 뒤에야 실행된다.

---

## 코드 예제 2: 리오더 버퍼(ROB)를 포함한 정확한 예외 처리

현대 CPU는 비순서 실행 결과를 **인순서로 커밋**하기 위해 **리오더 버퍼(Reorder Buffer, ROB)**를 추가한다.

```c
#include <stdio.h>
#include <stdbool.h>
#include <string.h>

#define ROB_SIZE 8
#define RS_SIZE  4

typedef struct {
    int  tag;           // 명령어 ID
    char dest[4];       // 목적 레지스터
    int  value;         // 실행 결과
    bool ready;         // 커밋 가능 여부
    bool valid;         // ROB 엔트리 유효
} ROBEntry;

typedef struct {
    bool busy;
    char op[4];
    int  vj, vk;        // 준비된 피연산자 값
    int  qj, qk;        // 기다리는 ROB 태그 (-1이면 준비됨)
    int  dest_rob;      // 결과를 쓸 ROB 인덱스
} RS;

ROBEntry rob[ROB_SIZE];
RS       rs[RS_SIZE];
int      reg_file[8];
int      reg_rob_tag[8]; // 레지스터별 ROB 태그 (-1이면 레지스터 파일이 최신)
int      rob_head, rob_tail;

void init() {
    memset(rob, 0, sizeof(rob));
    memset(rs,  0, sizeof(rs));
    for (int i = 0; i < 8; i++) {
        reg_file[i] = i * 10;
        reg_rob_tag[i] = -1;
    }
    rob_head = rob_tail = 0;
}

int alloc_rob(const char *dest, int dest_reg) {
    int idx = rob_tail;
    rob[idx].valid = true;
    rob[idx].ready = false;
    rob[idx].tag   = idx;
    snprintf(rob[idx].dest, 4, "%s", dest);
    rob_tail = (rob_tail + 1) % ROB_SIZE;
    reg_rob_tag[dest_reg] = idx;
    return idx;
}

void commit() {
    // 헤드부터 순서대로 커밋 (인순서 커밋)
    while (rob[rob_head].valid && rob[rob_head].ready) {
        ROBEntry *e = &rob[rob_head];
        int reg = e->dest[1] - '0';
        reg_file[reg] = e->value;
        if (reg_rob_tag[reg] == e->tag)
            reg_rob_tag[reg] = -1;
        printf("  COMMIT: %s ← %d (ROB[%d])\n", e->dest, e->value, e->tag);
        e->valid = false;
        rob_head = (rob_head + 1) % ROB_SIZE;
    }
}

int main() {
    init();
    printf("ROB 기반 비순서 실행 시뮬레이터\n");

    // 명령어 1: R1 = R2 + R3  (ADD)
    int rob1 = alloc_rob("R1", 1);
    rs[0].busy = true;
    strncpy(rs[0].op, "ADD", 4);
    rs[0].qj = (reg_rob_tag[2] != -1) ? reg_rob_tag[2] : -1;
    rs[0].vj = (rs[0].qj == -1)       ? reg_file[2]    : 0;
    rs[0].qk = (reg_rob_tag[3] != -1) ? reg_rob_tag[3] : -1;
    rs[0].vk = (rs[0].qk == -1)       ? reg_file[3]    : 0;
    rs[0].dest_rob = rob1;
    printf("ISSUE: R1 = R2 + R3 → ROB[%d]\n", rob1);

    // 명령어 2: R2 = R1 * R4  (MUL, R1은 ROB[0]을 기다림)
    int rob2 = alloc_rob("R2", 2);
    rs[1].busy = true;
    strncpy(rs[1].op, "MUL", 4);
    rs[1].qj = rob1;  // R1의 결과를 ROB[0]에서 기다림
    rs[1].vj = 0;
    rs[1].qk = -1;
    rs[1].vk = reg_file[4];
    rs[1].dest_rob = rob2;
    printf("ISSUE: R2 = R1 * R4 → ROB[%d] (R1 대기 중)\n", rob2);

    // 사이클 1: RS[0] 실행 완료
    printf("\n--- 사이클 1 ---\n");
    if (rs[0].qj == -1 && rs[0].qk == -1) {
        int result = rs[0].vj + rs[0].vk;
        rob[rs[0].dest_rob].value = result;
        rob[rs[0].dest_rob].ready = true;
        // CDB 브로드캐스트
        for (int i = 0; i < RS_SIZE; i++) {
            if (rs[i].busy && rs[i].qj == rs[0].dest_rob) {
                rs[i].vj = result; rs[i].qj = -1;
            }
            if (rs[i].busy && rs[i].qk == rs[0].dest_rob) {
                rs[i].vk = result; rs[i].qk = -1;
            }
        }
        printf("  EXEC+CDB: R1 = %d + %d = %d\n", rs[0].vj, rs[0].vk, result);
        rs[0].busy = false;
    }

    // 사이클 2: RS[1] 실행 완료 (피연산자 이제 준비됨)
    printf("--- 사이클 2 ---\n");
    if (rs[1].qj == -1 && rs[1].qk == -1) {
        int result = rs[1].vj * rs[1].vk;
        rob[rs[1].dest_rob].value = result;
        rob[rs[1].dest_rob].ready = true;
        printf("  EXEC: R2 = %d * %d = %d\n", rs[1].vj, rs[1].vk, result);
        rs[1].busy = false;
    }

    // 인순서 커밋
    printf("--- 커밋 ---\n");
    commit();

    printf("\n최종 레지스터: R1=%d, R2=%d\n", reg_file[1], reg_file[2]);
    return 0;
}
```

ROB 덕분에 명령어는 비순서로 실행되더라도 항상 프로그램 순서대로 커밋된다. 인터럽트나 예외가 발생해도 ROB를 플러시(flush)하면 정확한 예외 처리가 가능하다.

---

## 스코어보드와의 차이점

| 항목 | 스코어보드 | 토마슬로 알고리즘 |
|---|---|---|
| WAR/WAW 해저드 | 발생 가능 | 레지스터 이름 변경으로 제거 |
| 결과 전달 | 레지스터 파일 경유 | CDB 브로드캐스트 (즉시) |
| 스톨 기준 | 구조적·데이터 해저드 모두 | 진짜 RAW만 스톨 |
| 복잡도 | 낮음 | 높음 (CDB, RS, ROB 필요) |

---

## 주의사항과 현대 CPU에서의 응용

**메모리 의존성**: 로드/스토어 명령어는 주소가 계산될 때까지 의존성 판단이 불가능하다. 현대 CPU는 **Load-Store Queue**와 **메모리 의존성 예측**으로 이를 처리한다.

**CDB 병목**: 단일 CDB는 한 사이클에 하나의 결과만 방송한다. Intel, AMD의 현대 CPU는 **다중 CDB**를 사용한다.

**ROB 크기**: ROB가 클수록 더 많은 명령어를 비순서 실행할 수 있지만, 전력 소비와 면적이 증가한다. Intel Sapphire Rapids는 ROB가 512 엔트리에 달한다.

**정확한 예외 처리**: ROB 없이는 비순서 실행 상태에서 예외가 발생했을 때 CPU 상태를 일관되게 복구하기 어렵다. ROB의 인순서 커밋이 이를 해결한다.

---

## 참고 자료
- [Tomasulo's Algorithm — Wikipedia](https://en.wikipedia.org/wiki/Tomasulo%27s_algorithm)
- [Advanced Concepts of ILP — Dynamic Scheduling (UMD)](https://www.cs.umd.edu/~meesh/411/CA-online/chapter/advanced-concepts-of-ilp-dynamic-scheduling)
- [CS 433: Computer System Organization — Chapter 3 (UIUC)](https://courses.grainger.illinois.edu/CS433/fa2020/slides/chapter3-part1.pdf)
