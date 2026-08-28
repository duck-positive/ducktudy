---
layout: post
title: "SSD 내부 구조와 Flash Translation Layer 완전 정복: NAND Flash 동작 원리부터 쓰기 증폭까지"
date: 2026-08-28
categories: [cs, computer-science]
tags: [ssd, nand-flash, ftl, flash-translation-layer, write-amplification, wear-leveling, garbage-collection, storage]
---

현대 컴퓨팅 환경에서 SSD(Solid State Drive)는 HDD를 대부분 대체했다. 하지만 SSD가 단순히 "빠른 저장 장치"라는 인식은 내부 동작을 완전히 오해한 것이다. NAND Flash 메모리는 일반적인 RAM이나 HDD와 근본적으로 다른 물리적 특성을 가지며, 이 특성을 소프트웨어 계층인 **Flash Translation Layer(FTL)**가 추상화한다. FTL을 이해하면 SSD 성능 저하, 수명 문제, 그리고 왜 SSD가 랜덤 쓰기에 취약한지를 정확하게 파악할 수 있다.

## 개념 설명: NAND Flash의 물리적 특성

### NAND Flash의 계층 구조

NAND Flash는 **Cell → Page → Block → Plane → Die → Package** 계층으로 구성된다.

- **Cell**: 전하를 저장하는 최소 단위. SLC(1비트), MLC(2비트), TLC(3비트), QLC(4비트) 종류가 있다.
- **Page**: 읽기/쓰기의 최소 단위. 보통 4KB~16KB 크기.
- **Block**: 지우기(Erase)의 최소 단위. 보통 256KB~4MB 크기 (128~512개 페이지).
- **Plane**: 여러 Block의 집합. 병렬 작업 단위.

이 계층 구조에서 핵심적인 비대칭이 발생한다.

### NAND Flash의 세 가지 근본적 제약

**1. Write-Before-Erase (쓰기 전 지우기 필요)**

NAND Flash는 0으로 채워진 페이지에만 쓸 수 있다. 이미 데이터가 있는 페이지를 덮어쓰려면 반드시 전체 블록을 먼저 지워야 한다. 즉, 단 1바이트를 수정해도 256KB~4MB짜리 블록 전체를 지우고 다시 써야 한다.

**2. 읽기·쓰기·지우기의 성능 비대칭**

| 연산 | 단위 | 일반적인 시간 |
|------|------|--------------|
| 읽기 (Read) | Page | ~25μs |
| 쓰기 (Program) | Page | ~200μs |
| 지우기 (Erase) | Block | ~1.5ms |

지우기는 쓰기보다 약 7배 느리다.

**3. 유한한 쓰기 수명 (P/E Cycle)**

NAND Flash 셀은 반복 프로그램/이레이즈(P/E Cycle)로 인해 마모된다. TLC는 약 300~1000 사이클, SLC는 약 10만 사이클을 지원한다. 특정 블록에만 쓰기가 집중되면 조기에 불량 블록이 발생한다.

## 왜 FTL이 필요한가?

운영체제는 SSD를 일반적인 블록 디바이스로 인식한다. OS가 "LBA(Logical Block Address) 100번에 쓰기"를 요청할 때, SSD 내부에서는 다음 문제가 발생한다.

- 해당 물리 주소에 이미 데이터가 있으면 블록 전체를 지워야 함
- 지우기가 잦은 블록은 마모됨
- OS는 이런 내부 동작을 알지 못함

**FTL(Flash Translation Layer)**은 논리 주소(LBA)를 물리 주소(PBA, Physical Block Address)로 변환하는 소프트웨어 계층이다. HDD의 디스크 펌웨어가 섹터를 관리하듯, FTL은 NAND Flash의 복잡성을 운영체제로부터 숨긴다.

FTL의 핵심 역할:
1. **논리→물리 주소 매핑 (L2P Mapping)**
2. **웨어 레벨링 (Wear Leveling)**: 쓰기를 모든 블록에 균등하게 분산
3. **가비지 컬렉션 (Garbage Collection)**: 무효화된 페이지가 있는 블록을 재활용
4. **배드 블록 관리 (Bad Block Management)**: 불량 블록 추적 및 우회

## 실제 구현 예제

### 예제 1: 페이지 매핑 테이블 시뮬레이터 (Python)

```python
class FTLSimulator:
    """간단한 페이지 수준 FTL 시뮬레이터"""

    def __init__(self, num_blocks: int = 8, pages_per_block: int = 4):
        self.num_blocks = num_blocks
        self.pages_per_block = pages_per_block
        self.total_pages = num_blocks * pages_per_block

        # 물리 페이지 상태: None=비어있음, 0=무효, 데이터=유효
        self.physical_pages = [None] * self.total_pages

        # L2P 매핑 테이블: 논리 페이지 → 물리 페이지
        self.l2p: dict[int, int] = {}

        # 다음에 쓸 물리 페이지 포인터 (로그 구조)
        self.write_pointer = 0

        # 블록별 유효 페이지 수 추적
        self.valid_count = [pages_per_block] * num_blocks
        for b in range(num_blocks):
            self.valid_count[b] = 0

        # 쓰기 횟수 추적 (P/E Cycle)
        self.pe_cycles = [0] * num_blocks

        self.write_count = 0   # 호스트 쓰기
        self.flash_write_count = 0  # 실제 Flash 쓰기

    def _block_of(self, ppa: int) -> int:
        return ppa // self.pages_per_block

    def _pages_in_block(self, block: int):
        start = block * self.pages_per_block
        return range(start, start + self.pages_per_block)

    def write(self, lpa: int, data: str):
        """논리 페이지 주소(lpa)에 데이터 쓰기"""
        self.write_count += 1

        # 기존 매핑이 있으면 이전 물리 페이지를 무효화
        if lpa in self.l2p:
            old_ppa = self.l2p[lpa]
            old_block = self._block_of(old_ppa)
            self.physical_pages[old_ppa] = 0  # 무효 표시
            self.valid_count[old_block] -= 1

        # 쓸 공간이 없으면 GC 실행
        if self.write_pointer >= self.total_pages:
            self._garbage_collect()

        ppa = self.write_pointer
        self.physical_pages[ppa] = data
        self.l2p[lpa] = ppa
        block = self._block_of(ppa)
        self.valid_count[block] += 1
        self.write_pointer += 1
        self.flash_write_count += 1

        print(f"  WRITE lpa={lpa} → ppa={ppa} (block={block})")

    def read(self, lpa: int) -> str:
        """논리 페이지 주소 읽기"""
        if lpa not in self.l2p:
            raise KeyError(f"lpa {lpa} not mapped")
        ppa = self.l2p[lpa]
        return self.physical_pages[ppa]

    def _garbage_collect(self):
        """가장 유효 페이지가 적은 블록을 선택해 GC 실행"""
        # 희생 블록 선택: 유효 페이지가 가장 적은 블록
        victim_block = min(
            range(self.num_blocks),
            key=lambda b: self.valid_count[b]
        )
        print(f"\n  [GC] victim block={victim_block}, "
              f"valid={self.valid_count[victim_block]}")

        # 유효 페이지를 임시 저장 후 재배치
        valid_pages = []
        for ppa in self._pages_in_block(victim_block):
            if self.physical_pages[ppa] not in (None, 0):
                # 어떤 lpa가 이 ppa를 가리키는지 역탐색
                for lpa, mapped_ppa in list(self.l2p.items()):
                    if mapped_ppa == ppa:
                        valid_pages.append((lpa, self.physical_pages[ppa]))

        # 블록 지우기 (Erase)
        for ppa in self._pages_in_block(victim_block):
            self.physical_pages[ppa] = None
        self.valid_count[victim_block] = 0
        self.pe_cycles[victim_block] += 1
        print(f"  [GC] block {victim_block} erased "
              f"(P/E cycle: {self.pe_cycles[victim_block]})")

        # write_pointer를 victim block 시작으로 리셋
        self.write_pointer = victim_block * self.pages_per_block

        # 유효 데이터 재기록 (쓰기 증폭 발생!)
        for lpa, data in valid_pages:
            ppa = self.write_pointer
            self.physical_pages[ppa] = data
            self.l2p[lpa] = ppa
            self.valid_count[self._block_of(ppa)] += 1
            self.write_pointer += 1
            self.flash_write_count += 1
            print(f"  [GC] copy lpa={lpa} → ppa={ppa}")

    def write_amplification_factor(self) -> float:
        """쓰기 증폭 계수 (WAF)"""
        if self.write_count == 0:
            return 1.0
        return self.flash_write_count / self.write_count

    def status(self):
        print(f"\n--- FTL Status ---")
        print(f"Host writes:  {self.write_count}")
        print(f"Flash writes: {self.flash_write_count}")
        print(f"WAF:          {self.write_amplification_factor():.2f}")
        print(f"L2P mapping:  {self.l2p}")
        print(f"Valid counts: {self.valid_count}")
        print(f"P/E cycles:   {self.pe_cycles}")


# 시뮬레이션: 8 블록, 블록당 4 페이지
ftl = FTLSimulator(num_blocks=4, pages_per_block=4)

# 초기 쓰기
for i in range(12):
    ftl.write(lpa=i % 8, data=f"data_v{i}")

ftl.status()
```

### 예제 2: 쓰기 증폭(WAF) 측정 및 웨어 레벨링 효과 분석 (Python)

```python
import random
import statistics

def simulate_waf(num_lpas: int, num_writes: int,
                 hot_ratio: float = 0.2) -> dict:
    """
    hot_ratio: 전체 LPA 중 반복 쓰기가 집중되는 핫 영역 비율
    핫-콜드 워크로드에서 WAF가 얼마나 증가하는지 측정
    """
    # 물리 저장소 시뮬레이션
    mapping: dict[int, int] = {}
    physical: dict[int, str] = {}
    next_ppa = 0
    block_size = 16  # 블록당 페이지 수
    invalid_per_block: dict[int, int] = {}
    host_writes = 0
    flash_writes = 0

    hot_lpas = set(range(int(num_lpas * hot_ratio)))

    def get_block(ppa): return ppa // block_size

    for write_idx in range(num_writes):
        # 핫-콜드 분포: 80% 확률로 핫 영역에 쓰기
        if random.random() < 0.8:
            lpa = random.choice(list(hot_lpas))
        else:
            lpa = random.randint(0, num_lpas - 1)

        host_writes += 1

        # 기존 페이지 무효화
        if lpa in mapping:
            old_ppa = mapping[lpa]
            old_block = get_block(old_ppa)
            invalid_per_block[old_block] = \
                invalid_per_block.get(old_block, 0) + 1

        # GC 트리거: 주기적으로 실행 (단순화)
        if write_idx % (block_size * 2) == 0 and mapping:
            # 가장 많은 무효 페이지를 가진 블록 GC
            if invalid_per_block:
                victim = max(invalid_per_block, key=invalid_per_block.get)
                invalid_count = invalid_per_block.pop(victim, 0)
                # 유효 페이지 재배치
                valid_in_victim = [
                    (lpa, ppa) for lpa, ppa in mapping.items()
                    if get_block(ppa) == victim
                ]
                for gc_lpa, gc_ppa in valid_in_victim:
                    mapping[gc_lpa] = next_ppa
                    physical[next_ppa] = physical.get(gc_ppa, "")
                    next_ppa += 1
                    flash_writes += 1  # GC로 인한 추가 쓰기

        # 새 페이지에 쓰기
        mapping[lpa] = next_ppa
        physical[next_ppa] = f"data_{write_idx}"
        next_ppa += 1
        flash_writes += 1

    waf = flash_writes / host_writes if host_writes else 1.0
    return {
        "host_writes": host_writes,
        "flash_writes": flash_writes,
        "waf": round(waf, 2),
        "hot_ratio": hot_ratio,
    }

# 다양한 핫 비율로 WAF 측정
print("=== Write Amplification Factor vs Hot Ratio ===")
print(f"{'Hot Ratio':>12} | {'Host Writes':>12} | "
      f"{'Flash Writes':>13} | {'WAF':>6}")
print("-" * 55)

for hr in [0.05, 0.1, 0.2, 0.5, 1.0]:
    result = simulate_waf(num_lpas=1000, num_writes=5000, hot_ratio=hr)
    print(f"{result['hot_ratio']:>12.0%} | "
          f"{result['host_writes']:>12,} | "
          f"{result['flash_writes']:>13,} | "
          f"{result['waf']:>6.2f}")
```

## 주의사항과 팁

### Over-Provisioning (OP)

SSD 제조사는 사용자가 볼 수 있는 용량보다 더 많은 NAND Flash를 탑재한다. 이 예비 공간(OP)은 FTL이 GC를 실행할 여유 공간을 확보하여 WAF를 낮추고 성능을 유지한다. OP가 클수록 WAF는 낮아지고 수명은 늘어난다.

일반 SSD는 7~28%, 엔터프라이즈 SSD는 28~100%의 OP를 가진다.

### SSD 성능이 시간이 지날수록 느려지는 이유

새 SSD는 모든 블록이 비어 있어 지우기 없이 바로 쓸 수 있다. 하지만 시간이 지나면 모든 블록에 데이터가 차고, FTL은 쓸 때마다 GC를 실행해야 한다. 이를 **"Write Cliff"** 현상이라 한다. 특히 SSD 용량의 70~80% 이상 채워지면 성능이 급격히 저하된다.

### TRIM 명령

운영체제가 파일을 삭제해도 SSD는 해당 물리 페이지가 더 이상 필요 없다는 것을 알지 못한다. **TRIM(ATA) / UNMAP(SCSI)** 명령은 OS가 SSD에 "이 논리 블록은 삭제됐으니 GC 시 재활용해도 됨"을 알리는 명령이다. TRIM을 지원하지 않는 시스템(일부 RAID 컨트롤러)에서는 SSD 성능이 빠르게 저하된다.

### NVMe와 ZNS (Zoned Namespaces)

전통적인 FTL은 호스트와 SSD 사이의 불투명한 계층이다. **ZNS(Zoned Namespaces) SSD**는 이 경계를 허물어 호스트가 SSD의 Zone(블록 그룹)을 직접 관리하게 한다. RocksDB 같은 LSM-Tree 기반 데이터베이스는 이미 순차 쓰기 패턴을 가지므로, ZNS를 사용하면 WAF를 1에 가깝게 낮출 수 있다.

## 참고 자료
- [Coding for SSDs – Part 3: Pages, Blocks, and the Flash Translation Layer](https://codecapsule.com/2014/02/12/coding-for-ssds-part-3-pages-blocks-and-the-flash-translation-layer/)
- [NAND Flash Translation Layer (FTL) Explained](https://journal.rouleur.cc/nand-flash-translation-layer/)
- [A Brief History of Data Placement Technologies – Samsung Semiconductor](https://semiconductor.samsung.com/news-events/tech-blog/a-brief-history-of-data-placement-technologies/)
- [Towards Efficient Flash Caches with Emerging NVMe Flexible Data Placement SSDs (arXiv)](https://arxiv.org/pdf/2503.11665)
