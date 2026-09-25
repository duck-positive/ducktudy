---
layout: post
title: "FlatBuffers와 Cap'n Proto 완전 정복: 역직렬화 없는 제로 카피 직렬화 형식"
date: 2026-09-25
categories: [cs, computer-science]
tags: [flatbuffers, capnproto, serialization, zero-copy, performance, protobuf, binary-format]
---

게임 서버에서 엔티티 상태를 매 프레임 직렬화할 때, 고빈도 거래 시스템에서 주문 데이터를 마이크로초 단위로 처리할 때, Protocol Buffers의 파싱 비용이 병목이 된다면 어떻게 할까요? FlatBuffers와 Cap'n Proto는 이 문제를 근본적으로 해결합니다. **역직렬화 단계 자체를 없애버리는** 제로 카피(Zero-Copy) 직렬화 형식입니다. 이 글에서는 두 형식의 내부 메모리 레이아웃부터 구현, 성능 특성, 적합한 사용 사례까지 완전히 다룹니다.

## 기존 직렬화의 비용

Protocol Buffers(protobuf)나 JSON 같은 전통적 직렬화 형식은 두 단계를 거칩니다:

```
전통적 직렬화/역직렬화:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Writer:  [객체] → [인코딩] → [바이트 스트림]
                    ↑ CPU 비용: varint 인코딩, 태그 쓰기
Reader:  [바이트 스트림] → [디코딩] → [새 객체 생성]
                             ↑ CPU 비용: varint 디코딩, 메모리 할당, 복사

제로 카피 직렬화 (FlatBuffers / Cap'n Proto):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Writer:  [객체] → [메모리 레이아웃 직접 구성] → [바이트 스트림]
Reader:  [바이트 스트림] → 포인터만 설정 → [직접 필드 접근]
                             ↑ 역직렬화 단계 없음! 메모리 할당 없음!
```

벤치마크에서 FlatBuffers의 읽기 속도는 protobuf 대비 10~40배 빠릅니다. 단, 쓰기 속도는 protobuf와 비슷하거나 느립니다.

## FlatBuffers: Google이 게임을 위해 만든 형식

FlatBuffers는 Google이 Android 게임 성능 최적화를 위해 개발했습니다. 핵심 아이디어는 **wire format이 곧 in-memory format**이라는 것입니다.

### FlatBuffers 메모리 레이아웃

```
버퍼 (낮은 주소 → 높은 주소):
┌────────────────────────────────────────────────────────────┐
│ 데이터 섹션 (뒤에서 앞으로 구성)     루트 오프셋(4 bytes)│
│  [문자열 "Alice"]  [Vector 데이터]         ↓              │
│                              [vtable][Monster 테이블]      │
└────────────────────────────────────────────────────────────┘
                                       ↑
                              루트 테이블 포인터

vtable 구조:
┌──────────────┬────────────────────────────────────────────┐
│ vtable_size  │ object_size │ field0_offset │ field1_offset │
│   (2 bytes)  │  (2 bytes)  │   (2 bytes)   │   (2 bytes)  │
└──────────────┴────────────────────────────────────────────┘
```

각 테이블은 `vtable`(virtual table)을 통해 필드에 접근합니다. 필드가 없으면 오프셋이 0으로 설정되고, 기본값이 반환됩니다. 이로써 스키마 진화(schema evolution)를 지원합니다.

### FlatBuffers 스키마 정의

```fbs
// monster.fbs
namespace Game;

enum Color: byte { Red = 0, Green, Blue }

struct Vec3 {
    x: float;
    y: float;
    z: float;
}

table Weapon {
    name:   string;
    damage: short;
}

table Monster {
    pos:        Vec3;       // struct: 인라인 저장 (복사 없음)
    mana:       short = 150;
    hp:         short = 100;
    name:       string;
    inventory:  [ubyte];    // 바이트 벡터
    color:      Color = Blue;
    weapons:    [Weapon];   // 오프셋 벡터
    equipped:   Weapon;
}

root_type Monster;
```

### 코드 예제 1: FlatBuffers C++ 빌더와 제로 카피 읽기

```cpp
#include "flatbuffers/flatbuffers.h"
#include "monster_generated.h"  // flatc --cpp monster.fbs 생성

// === 쓰기 ===
std::vector<uint8_t> build_monster() {
    flatbuffers::FlatBufferBuilder builder(1024);  // 초기 용량 힌트

    // 문자열은 미리 생성해야 함 (후방에서 전방으로 구성)
    auto name     = builder.CreateString("Orc Warrior");
    auto sword_nm = builder.CreateString("Vorpal Sword");
    auto axe_nm   = builder.CreateString("Battle Axe");

    // 무기 생성
    auto sword = Game::CreateWeapon(builder, sword_nm, 3);
    auto axe   = Game::CreateWeapon(builder, axe_nm,  5);

    // 벡터 (FlatBuffers는 역순 구성 후 정방향 접근)
    std::vector<flatbuffers::Offset<Game::Weapon>> weapons_vec = {sword, axe};
    auto weapons = builder.CreateVector(weapons_vec);

    // 인벤토리 바이트 벡터
    std::vector<uint8_t> inventory = {0, 1, 2, 3, 4};
    auto inv = builder.CreateVector(inventory);

    // Monster 테이블 구성
    Game::Vec3 pos(1.0f, 2.0f, 3.0f);  // struct: 스택에 생성, 복사됨
    auto monster = Game::CreateMonster(
        builder,
        &pos,       // Vec3 struct
        300,        // mana
        500,        // hp
        name,
        inv,
        Game::Color_Red,
        weapons
    );

    builder.Finish(monster);
    uint8_t* buf  = builder.GetBufferPointer();
    size_t   size = builder.GetSize();
    return std::vector<uint8_t>(buf, buf + size);
}

// === 읽기 (역직렬화 없음) ===
void read_monster(const uint8_t* buf, size_t size) {
    // 검증 (선택사항, O(n) 비용)
    flatbuffers::Verifier verifier(buf, size);
    assert(Game::VerifyMonsterBuffer(verifier));

    // 루트 오프셋 역참조만으로 접근 준비 완료 - 메모리 할당 없음!
    const Game::Monster* monster = Game::GetMonster(buf);

    printf("Name: %s\n", monster->name()->c_str());
    printf("HP: %d, Mana: %d\n", monster->hp(), monster->mana());
    printf("Position: (%.1f, %.1f, %.1f)\n",
           monster->pos()->x(), monster->pos()->y(), monster->pos()->z());

    // 벡터 접근: 역직렬화 없이 버퍼 내 포인터 반환
    if (monster->weapons()) {
        for (const Game::Weapon* w : *monster->weapons()) {
            printf("Weapon: %s (damage: %d)\n", w->name()->c_str(), w->damage());
        }
    }

    // 인-플레이스 변경 (동일 버퍼 수정): FlatBuffers만의 기능
    auto* mutable_monster = const_cast<Game::Monster*>(monster);
    // MutateHP는 버퍼를 직접 수정 (재직렬화 불필요)
    // mutable_monster->mutate_hp(200);
}

int main() {
    auto buf = build_monster();
    read_monster(buf.data(), buf.size());
    return 0;
}
```

### FlatBuffers Python 접근 (flatbuffers 패키지)

```python
import flatbuffers
from Game import Monster, Weapon, Vec3, Color

def build_and_read_monster() -> bytes:
    builder = flatbuffers.Builder(256)
    
    # 문자열 생성 (역방향)
    name = builder.CreateString("Python Orc")
    
    # Weapon 생성
    sword_name = builder.CreateString("Swift Blade")
    Weapon.WeaponStart(builder)
    Weapon.WeaponAddName(builder, sword_name)
    Weapon.WeaponAddDamage(builder, 25)
    sword = Weapon.WeaponEnd(builder)
    
    # Vector<Weapon>
    Monster.MonsterStartWeaponsVector(builder, 1)
    builder.PrependUOffsetTRelative(sword)
    weapons = builder.EndVector(1)
    
    # Monster 조립
    Monster.MonsterStart(builder)
    Monster.MonsterAddName(builder, name)
    Monster.MonsterAddHp(builder, 300)
    Monster.MonsterAddMana(builder, 150)
    Monster.MonsterAddColor(builder, Color.Color.Red)
    Monster.MonsterAddWeapons(builder, weapons)
    monster = Monster.MonsterEnd(builder)
    
    builder.Finish(monster)
    buf = bytes(builder.Output())
    
    # 읽기: 역직렬화 없음
    m = Monster.Monster.GetRootAsMonster(buf, 0)
    print(f"Name: {m.Name().decode()}, HP: {m.Hp()}")
    print(f"Weapon count: {m.WeaponsLength()}")
    return buf
```

## Cap'n Proto: "직렬화 없는 직렬화"

Cap'n Proto는 Protocol Buffers의 창시자 Kenton Varda가 개발했습니다. FlatBuffers보다 더 극단적인 접근을 취합니다: **쓰기 단계도 직렬화가 없습니다**. 메모리에서 구성한 메시지 구조가 그대로 wire format입니다.

### Cap'n Proto 메모리 레이아웃

Cap'n Proto는 8바이트 정렬을 기반으로 두 섹션으로 나뉩니다:

```
세그먼트 구조 (Segment 0):
┌─────────────────┬─────────────────────────────────────────────────────┐
│   Segment Table  │                    Segment Data                      │
│  [0, segment0_sz]│  [struct data][list][struct data][string data]...   │
└─────────────────┴─────────────────────────────────────────────────────┘

Struct 포인터 (8 bytes):
Bits [1:0]   = 00  (struct 타입)
Bits [31:2]  = 오프셋 (현재 위치에서 데이터 섹션까지의 단어 수)
Bits [47:32] = 데이터 섹션 크기 (단어 단위, 1 단어 = 8 bytes)
Bits [63:48] = 포인터 섹션 크기 (포인터 수)

List 포인터 (8 bytes):
Bits [1:0]   = 01  (list 타입)
Bits [31:2]  = 오프셋
Bits [34:32] = 원소 크기 (0=void, 1=1bit, 2=1byte, 3=2byte, 4=4byte, 5=8byte, 6=ptr, 7=composite)
Bits [63:35] = 원소 수
```

### Cap'n Proto 스키마와 코드 예제 2

```capnp
# addressbook.capnp
@0x934efea7f017fff0;

struct Person {
    name    @0 :Text;
    id      @1 :UInt32;
    email   @2 :Text;

    struct PhoneNumber {
        number @0 :Text;
        type   @1 :Type;
        enum Type { mobile @0; home @1; work @2; }
    }

    phones  @3 :List(PhoneNumber);

    union {
        employment @4 :Void;
        employer   @5 :Text;
        school     @6 :Text;
        selfEmployed @7 :Void;
    }
}

struct AddressBook {
    people @0 :List(Person);
}
```

```python
# capnp Python 바인딩 사용 (pycapnp)
import capnp
import addressbook_capnp  # 자동 생성

def write_address_book() -> bytes:
    address_book = addressbook_capnp.AddressBook.new_message()
    people = address_book.init('people', 2)
    
    # Alice 설정
    alice = people[0]
    alice.name    = "Alice"
    alice.id      = 123
    alice.email   = "alice@example.com"
    alice.employer = "ACME Corp"  # union 필드 선택
    
    phones = alice.init('phones', 2)
    phones[0].number = "+1-555-1234"
    phones[0].type   = 'mobile'
    phones[1].number = "+1-555-5678"
    phones[1].type   = 'work'
    
    # Bob 설정
    bob = people[1]
    bob.name = "Bob"
    bob.id   = 456
    bob.email = "bob@example.com"
    bob.selfEmployed = None  # void union 필드
    
    # 직렬화: 메모리 구조를 그대로 바이트로 내보냄
    return address_book.to_bytes()

def read_address_book(data: bytes):
    # 역직렬화 없음: 바이트 스트림에 직접 포인터 설정
    with addressbook_capnp.AddressBook.from_bytes(data) as address_book:
        for person in address_book.people:
            print(f"\nName: {person.name}")
            print(f"Email: {person.email}")
            
            # union 접근
            which = person.employment.which()
            if which == 'employer':
                print(f"Works at: {person.employment.employer}")
            elif which == 'selfEmployed':
                print("Self-employed")
            
            for phone in person.phones:
                print(f"Phone ({phone.type}): {phone.number}")

# Cap'n Proto RPC: 프로미스 파이프라이닝
# 네트워크 왕복 없이 원격 반환값을 다음 호출의 인수로 파이프라인 구성 가능
```

## FlatBuffers vs Cap'n Proto vs Protocol Buffers 비교

```
특성별 비교:

                    FlatBuffers    Cap'n Proto    Protocol Buffers
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
직렬화 비용          중간           매우 낮음       중간
역직렬화 비용        없음           없음            중간
메시지 크기          중간 (↑)       중간 (↑)        작음 (varint)
메모리 효율          중간           중간            높음 (packed)
인플레이스 변경      지원           미지원           미지원
스키마 진화          지원(선택 필드) 지원(번호 기반) 지원(번호 기반)
RPC 지원            없음           있음 (pipelining) gRPC 통해 지원
랜덤 접근           지원           지원             미지원 (전체 파싱 필요)
언어 지원           C++,C#,Go,Java,Python,Rust    광범위   광범위
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

성능 벤치마크 (상대적 수치, FlatBuffers 공식 자료 기준):
- 읽기 처리량: FlatBuffers ≈ Cap'n Proto >> protobuf ≈ JSON
- 쓰기 처리량: protobuf ≈ FlatBuffers > Cap'n Proto (배치) ≈ JSON
- 메시지 크기: protobuf < FlatBuffers ≈ Cap'n Proto << JSON
```

## 스키마 진화 (Schema Evolution)

두 형식 모두 필드 번호 기반으로 하위/상위 호환성을 보장합니다:

```fbs
// FlatBuffers: 필드 추가는 끝에만, 제거는 deprecated 처리
table Player {
    id:    uint64;        // @0 암묵적
    name:  string;        // @1
    score: int32;         // @2
    // 구버전 클라이언트는 @3을 무시함
    rank:  int32 = -1;    // @3: 새로 추가된 필드
    // level: int32;      // @X: 제거 금지! 대신 deprecated 태그 사용
}
```

```capnp
# Cap'n Proto: 각 필드에 고유 번호 (@N) 부여
struct Player {
    id    @0 :UInt64;
    name  @1 :Text;
    score @2 :Int32;
    rank  @3 :Int32;        # 새 버전에 추가
    # 이전 @번호는 재사용 불가 (even after removal)
}
```

## 적합한 사용 사례

**FlatBuffers가 적합한 경우:**
- 게임 엔진 (Unity/Unreal): 매 프레임 수천 개의 엔티티 상태 업데이트
- 인게임 에셋 포맷: 모델·텍스처 메타데이터를 복사 없이 직접 접근
- 공유 메모리 IPC: 동일 머신 내 프로세스 간 zero-copy 데이터 공유
- 임베디드 시스템: 메모리 할당을 피해야 하는 RTOS 환경

**Cap'n Proto가 적합한 경우:**
- 고성능 RPC (Promise Pipelining으로 왕복 지연 제거)
- 데이터베이스 내부 포맷 (Sandstorm, KJ 라이브러리 기반)
- 메모리 내 구조를 파일이나 네트워크로 직접 공유

**Protocol Buffers/gRPC를 유지해야 하는 경우:**
- 생태계 호환성이 중요한 경우 (마이크로서비스 API)
- 메시지 크기가 대역폭 비용에 중요한 경우
- 광범위한 언어·도구 지원이 필요한 경우

## 주의사항과 트레이드오프

**쓰기 시 정방향 진행 불가:**
FlatBuffers는 버퍼를 뒤에서 앞으로 구성합니다. 중첩된 객체는 먼저 생성해야 합니다. 직관적이지 않아 초기 학습 비용이 있습니다.

**메시지 크기 증가:**
정렬 패딩과 vtable 오버헤드로 protobuf 대비 메시지 크기가 20~50% 더 클 수 있습니다. 대역폭이 제한된 환경에서는 주의가 필요합니다.

**뮤테이션 제약:**
Cap'n Proto 메시지는 생성 후 필드 크기가 변하는 변경(예: 문자열 길이 변경)을 지원하지 않습니다. FlatBuffers는 고정 크기 필드(int, float)만 인플레이스 변경 가능합니다.

**디버깅 어려움:**
바이너리 형식이라 Wireshark나 텍스트 에디터로 직접 검사가 어렵습니다. `flatc --json`이나 `capnp decode` 도구를 활용하세요:

```bash
# FlatBuffers 바이너리를 JSON으로 디코딩
flatc --json monster.fbs -- monster.bin

# Cap'n Proto 메시지 확인
capnp decode addressbook.capnp AddressBook < message.bin
```

## 참고 자료
- [FlatBuffers 공식 문서 및 벤치마크](https://flatbuffers.dev/benchmarks/)
- [Cap'n Proto: FlatBuffers, SBE 비교](https://capnproto.org/news/2014-06-17-capnproto-flatbuffers-sbe.html)
- [FlatBuffers GitHub 저장소](https://github.com/google/flatbuffers)
- [Protocol Buffers vs FlatBuffers vs Cap'n Proto 심층 비교](https://adhdecode.com/api-architecture/grpc-deep-dive/protobuf-vs-flatbuffers-vs-capn-proto/)
