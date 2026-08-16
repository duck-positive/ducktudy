---
layout: post
title: "데이터 지향 설계(Data-Oriented Design) 완전 정복: SoA, ECS, 캐시 친화적 프로그래밍으로 10배 성능 끌어내기"
date: 2026-08-16
categories: [cs, computer-science]
tags: [data-oriented-design, dod, ecs, soa, aos, cache, simd, performance, game-engine, unity-dots]
---

## 데이터 지향 설계란 무엇인가

데이터 지향 설계(Data-Oriented Design, DOD)는 "프로그램은 데이터를 변환하는 것이다"라는 전제에서 출발하는 설계 철학이다. 객체 지향 설계(OOP)가 현실 세계의 개념을 객체로 모델링하는 데 집중하는 반면, DOD는 **CPU가 어떻게 데이터에 접근하는가**를 설계의 중심에 놓는다.

2014년 CppCon에서 Naughty Dog의 수석 엔지니어 Mike Acton이 발표한 "Data-Oriented Design and C++"은 이 철학을 게임 업계 전반에 전파했다. 그의 핵심 주장은 세 가지다:

1. **하드웨어는 데이터를 처리하는 기계다.** 소프트웨어는 하드웨어 위에서 실행된다.
2. **성능의 적은 추상화가 아니라 캐시 미스다.** L1 캐시 히트는 4사이클, DRAM 접근은 200사이클 이상이다.
3. **문제는 데이터가 먼저다.** 코드는 데이터를 따른다.

---

## 왜 필요한가 — OOP의 숨겨진 성능 비용

### 현대 CPU의 캐시 계층

현대 CPU는 실제 연산 속도와 메모리 대역폭 사이의 극심한 격차를 캐시 계층으로 메운다:

| 캐시 수준 | 크기 | 레이턴시 |
|----------|------|---------|
| L1 캐시 | 32~64 KB | ~4 사이클 (1ns) |
| L2 캐시 | 256 KB ~ 1 MB | ~12 사이클 (3ns) |
| L3 캐시 | 4~64 MB | ~40 사이클 (10ns) |
| DRAM | GB 단위 | ~200+ 사이클 (60ns) |

CPU는 메모리에서 데이터를 읽을 때 항상 **캐시 라인(64 바이트)** 단위로 읽는다. 필요한 데이터가 캐시 라인 내에 인접해 있으면 한 번의 DRAM 접근으로 여러 데이터를 미리 적재할 수 있다. 반대로 데이터가 메모리에 흩어져 있으면 데이터 하나마다 캐시 미스가 발생한다.

### AoS(Array of Structures)의 문제

전형적인 OOP 스타일의 게임 엔티티:

```cpp
// AoS: 객체 배열 — 각 엔티티가 모든 필드를 포함
struct Entity {
    float  pos_x, pos_y, pos_z;   // 위치  (12 바이트)
    float  vel_x, vel_y, vel_z;   // 속도  (12 바이트)
    float  health;                 // 체력   (4 바이트)
    bool   is_visible;             // 가시성 (1 바이트)
    char   name[64];               // 이름  (64 바이트)
    Model *model;                  // 모델 포인터 (8 바이트)
    // ... 기타 필드 (총 ~120 바이트)
};

Entity entities[10000];

// 물리 업데이트: 위치와 속도만 필요
void update_physics(float dt) {
    for (int i = 0; i < 10000; i++) {
        entities[i].pos_x += entities[i].vel_x * dt;
        entities[i].pos_y += entities[i].vel_y * dt;
        entities[i].pos_z += entities[i].vel_z * dt;
    }
}
```

`update_physics`는 위치(12B)와 속도(12B)만 사용하지만, `Entity`가 120바이트이므로 캐시 라인 하나(64B)에 Entity 하나도 다 들어가지 않는다. 10,000개를 순회하면 약 **18,750번의 캐시 미스**가 발생한다.

---

## 핵심 개념: SoA(Structure of Arrays)

DOD의 핵심 리팩토링은 AoS를 SoA로 전환하는 것이다.

```
AoS:  [pos, vel, health, name, model] [pos, vel, health, name, model] ...
SoA:  [pos pos pos pos ...]  [vel vel vel vel ...]  [health health ...]
```

`update_physics`가 접근하는 데이터만 연속적으로 메모리에 배치되어, 캐시 효율이 극적으로 향상된다.

---

## 실제 구현 예제 1: AoS vs SoA 성능 비교

```cpp
#include <vector>
#include <chrono>
#include <random>
#include <iostream>
#include <immintrin.h>  // SIMD 헤더

constexpr int N = 1'000'000;

// ── AoS 버전 ──────────────────────────────────────
struct EntityAoS {
    float pos_x, pos_y, pos_z;
    float vel_x, vel_y, vel_z;
    float health;
    char  padding[100];  // 실제 게임 엔티티의 기타 필드 시뮬레이션
};

void update_aos(std::vector<EntityAoS>& entities, float dt) {
    for (auto& e : entities) {
        e.pos_x += e.vel_x * dt;
        e.pos_y += e.vel_y * dt;
        e.pos_z += e.vel_z * dt;
    }
}

// ── SoA 버전 ──────────────────────────────────────
struct PhysicsComponents {
    std::vector<float> pos_x, pos_y, pos_z;
    std::vector<float> vel_x, vel_y, vel_z;
    // health, name 등 물리 업데이트에 불필요한 데이터는 별도 배열로 분리
};

void update_soa(PhysicsComponents& pc, float dt) {
    const int n = (int)pc.pos_x.size();
    // 컴파일러가 이 루프를 SIMD 자동 벡터화할 수 있음
    for (int i = 0; i < n; i++) {
        pc.pos_x[i] += pc.vel_x[i] * dt;
        pc.pos_y[i] += pc.vel_y[i] * dt;
        pc.pos_z[i] += pc.vel_z[i] * dt;
    }
}

// ── 명시적 AVX2 SIMD 버전 (8개 float 동시 처리) ────
void update_soa_simd(PhysicsComponents& pc, float dt) {
    const int n = (int)pc.pos_x.size();
    __m256 vdt = _mm256_set1_ps(dt);

    int i = 0;
    // 8개씩 묶어서 처리
    for (; i <= n - 8; i += 8) {
        __m256 px = _mm256_loadu_ps(&pc.pos_x[i]);
        __m256 vx = _mm256_loadu_ps(&pc.vel_x[i]);
        px = _mm256_fmadd_ps(vx, vdt, px);  // FMA: px + vx * dt
        _mm256_storeu_ps(&pc.pos_x[i], px);

        __m256 py = _mm256_loadu_ps(&pc.pos_y[i]);
        __m256 vy = _mm256_loadu_ps(&pc.vel_y[i]);
        py = _mm256_fmadd_ps(vy, vdt, py);
        _mm256_storeu_ps(&pc.pos_y[i], py);

        __m256 pz = _mm256_loadu_ps(&pc.pos_z[i]);
        __m256 vz = _mm256_loadu_ps(&pc.vel_z[i]);
        pz = _mm256_fmadd_ps(vz, vdt, pz);
        _mm256_storeu_ps(&pc.pos_z[i], pz);
    }
    // 나머지 처리
    for (; i < n; i++) {
        pc.pos_x[i] += pc.vel_x[i] * dt;
        pc.pos_y[i] += pc.vel_y[i] * dt;
        pc.pos_z[i] += pc.vel_z[i] * dt;
    }
}

int main() {
    std::mt19937 rng(42);
    std::uniform_real_distribution<float> dist(-10.f, 10.f);

    // AoS 초기화
    std::vector<EntityAoS> aos(N);
    for (auto& e : aos) {
        e.pos_x = dist(rng); e.vel_x = dist(rng);
        e.pos_y = dist(rng); e.vel_y = dist(rng);
        e.pos_z = dist(rng); e.vel_z = dist(rng);
    }

    // SoA 초기화
    PhysicsComponents soa;
    soa.pos_x.resize(N); soa.vel_x.resize(N);
    soa.pos_y.resize(N); soa.vel_y.resize(N);
    soa.pos_z.resize(N); soa.vel_z.resize(N);
    for (int i = 0; i < N; i++) {
        soa.pos_x[i] = dist(rng); soa.vel_x[i] = dist(rng);
        soa.pos_y[i] = dist(rng); soa.vel_y[i] = dist(rng);
        soa.pos_z[i] = dist(rng); soa.vel_z[i] = dist(rng);
    }

    auto bench = [](auto&& fn, const char* label) {
        auto t0 = std::chrono::high_resolution_clock::now();
        for (int iter = 0; iter < 100; iter++) fn();
        auto t1 = std::chrono::high_resolution_clock::now();
        double ms = std::chrono::duration<double, std::milli>(t1 - t0).count();
        std::cout << label << ": " << ms / 100 << " ms/frame\n";
    };

    bench([&]{ update_aos(aos, 0.016f); },              "AoS         ");
    bench([&]{ update_soa(soa, 0.016f); },              "SoA         ");
    bench([&]{ update_soa_simd(soa, 0.016f); },         "SoA + SIMD  ");

    /* 전형적인 결과 (100만 엔티티, Release 빌드):
       AoS         : 8.4 ms/frame
       SoA         : 1.8 ms/frame   (~4.7× 빠름)
       SoA + SIMD  : 0.6 ms/frame   (~14× 빠름)  */
    return 0;
}
```

---

## 실제 구현 예제 2: ECS (Entity-Component-System) 아키텍처

ECS는 DOD를 아키텍처 수준으로 체계화한 패턴이다. Unity DOTS, Unreal Mass Entity, Bevy Engine이 채택하고 있다.

```cpp
#include <vector>
#include <unordered_map>
#include <bitset>
#include <cstdint>
#include <cassert>

// ─ 컴포넌트 타입 ID ───────────────────────────────
using EntityID    = uint32_t;
using ComponentID = uint8_t;

static ComponentID next_component_id = 0;
template <typename T>
ComponentID component_id() {
    static ComponentID id = next_component_id++;
    return id;
}

// ─ 컴포넌트 스토어: 각 타입별 연속 배열 ─────────────
template <typename T>
struct ComponentArray {
    std::vector<T>        data;
    std::vector<EntityID> entity_ids;  // 인덱스 → 엔티티 역매핑
    std::unordered_map<EntityID, size_t> entity_to_idx;

    void insert(EntityID eid, T component) {
        entity_to_idx[eid] = data.size();
        data.push_back(std::move(component));
        entity_ids.push_back(eid);
    }

    T* get(EntityID eid) {
        auto it = entity_to_idx.find(eid);
        if (it == entity_to_idx.end()) return nullptr;
        return &data[it->second];
    }

    /* 연속 순회: 캐시 친화적 */
    T* begin() { return data.data(); }
    T* end()   { return data.data() + data.size(); }
    size_t size() const { return data.size(); }
};

// ─ 컴포넌트 정의 ─────────────────────────────────
struct Position  { float x, y, z; };
struct Velocity  { float x, y, z; };
struct Health    { float value;     };
struct Renderable{ uint32_t mesh_id; };

// ─ 월드: 엔티티와 컴포넌트 스토어 관리 ───────────────
struct World {
    EntityID next_id = 1;
    ComponentArray<Position>   positions;
    ComponentArray<Velocity>   velocities;
    ComponentArray<Health>     healths;
    ComponentArray<Renderable> renderables;

    EntityID create_entity() { return next_id++; }

    void add(EntityID e, Position p)    { positions.insert(e, p); }
    void add(EntityID e, Velocity v)    { velocities.insert(e, v); }
    void add(EntityID e, Health h)      { healths.insert(e, h); }
    void add(EntityID e, Renderable r)  { renderables.insert(e, r); }
};

// ─ 시스템: 특정 컴포넌트 집합을 처리하는 순수 함수 ────
namespace Systems {

/* 물리 시스템: Position + Velocity 보유 엔티티만 처리
   데이터가 연속 메모리에 있으므로 캐시 친화적 */
void physics(World& world, float dt) {
    auto& pos = world.positions;
    auto& vel = world.velocities;

    // 두 배열을 동시에 순회 (공통 EntityID 기준 조인)
    for (size_t i = 0; i < vel.size(); i++) {
        EntityID eid = vel.entity_ids[i];
        Position* p = pos.get(eid);
        if (!p) continue;

        p->x += vel.data[i].x * dt;
        p->y += vel.data[i].y * dt;
        p->z += vel.data[i].z * dt;
    }
}

/* 체력 시스템: 체력이 0 이하인 엔티티 처리 */
void health_check(World& world, std::vector<EntityID>& dead) {
    for (size_t i = 0; i < world.healths.size(); i++) {
        if (world.healths.data[i].value <= 0.f)
            dead.push_back(world.healths.entity_ids[i]);
    }
}

} // namespace Systems

// ─ 사용 예시 ──────────────────────────────────────
int main() {
    World world;

    /* 엔티티 생성: 각 컴포넌트는 별도 연속 배열에 저장 */
    for (int i = 0; i < 100000; i++) {
        EntityID e = world.create_entity();
        world.add(e, Position{(float)i, 0.f, 0.f});
        world.add(e, Velocity{1.f, 0.5f, 0.f});
        if (i % 2 == 0) world.add(e, Health{100.f});
        if (i % 3 == 0) world.add(e, Renderable{(uint32_t)(i % 256)});
    }

    /* 게임 루프 */
    for (int frame = 0; frame < 60; frame++) {
        Systems::physics(world, 0.016f);

        std::vector<EntityID> dead;
        Systems::health_check(world, dead);
        // dead 엔티티 처리...
    }
    return 0;
}
```

ECS의 핵심 이점은 시스템이 자신이 필요한 컴포넌트 데이터만 연속으로 순회한다는 점이다. `physics` 시스템은 `Renderable`, `Health`, `name` 데이터를 전혀 건드리지 않는다.

---

## 성능 측정: Perf로 캐시 미스 확인

리팩토링 전후의 성능 차이를 정량화하려면 하드웨어 성능 카운터를 사용한다:

```bash
# AoS 버전 캐시 미스 측정
perf stat -e cache-references,cache-misses,instructions,cycles \
    ./game_aos 2>&1 | grep -E "cache|instructions|cycles"

# SoA 버전 비교
perf stat -e cache-references,cache-misses,instructions,cycles \
    ./game_soa 2>&1 | grep -E "cache|instructions|cycles"

# 전형적인 결과:
#  AoS: cache-misses = 12,847,320  (미스율 ~45%)
#  SoA: cache-misses =  1,203,445  (미스율  ~8%)
```

---

## 주의사항과 설계 팁

### DOD는 항상 OOP보다 빠르지 않다

DOD의 이점은 **동종 데이터를 대량으로 처리할 때** 극대화된다. 10만 개 이상의 엔티티를 매 프레임 처리하는 게임, 실시간 시뮬레이션, ML 추론 배치 처리 등이 적합한 도메인이다. 수십 개의 객체를 다루는 비즈니스 로직, UI 계층, 이벤트 핸들러 등에는 OOP가 여전히 유지보수성 면에서 유리하다.

### 도메인 분리

실제 프로젝트에서는 "핫 경로(Hot Path)"와 "콜드 경로(Cold Path)"를 구분한다. 매 프레임 업데이트되는 물리/렌더링 데이터는 SoA+ECS로 관리하고, 게임 설정, 아이템 정보, 대화 데이터 등 변경이 드문 데이터는 일반 OOP로 관리하는 하이브리드 방식이 현실적이다.

### Unity DOTS의 교훈

Unity의 DOTS(Data-Oriented Technology Stack)는 기존 MonoBehaviour 기반 OOP 시스템을 ECS + Job System + Burst Compiler로 전환했다. Burst Compiler는 SoA 데이터를 자동으로 SIMD 벡터화한다. DOTS를 사용하면 동일한 씬에서 기존 대비 **수십 배 많은 오브젝트**를 처리할 수 있다고 공식 문서에서 밝히고 있다. 다만 DOTS의 Archetype 시스템(컴포넌트 집합을 공유하는 엔티티를 같은 청크에 배치)은 컴포넌트 추가/제거 시 엔티티 이동 비용이 발생한다. 자주 변경되는 컴포넌트 집합은 별도 Archetype을 설계해야 한다.

### false sharing 주의

멀티스레드 환경에서 여러 스레드가 같은 캐시 라인의 서로 다른 원소를 동시에 쓰면 **False Sharing** 문제가 발생해 성능이 오히려 저하된다. 64바이트 정렬(alignas(64))과 스레드별 청크 분할로 이를 방지한다.

---

## 참고 자료

- [dbartolini/data-oriented-design — 큐레이티드 DOD 리소스 모음 (발표, 블로그, 비디오)](https://github.com/dbartolini/data-oriented-design)
- [jslee02/awesome-entity-component-system — ECS 라이브러리 및 리소스 큐레이션](https://github.com/jslee02/awesome-entity-component-system)
