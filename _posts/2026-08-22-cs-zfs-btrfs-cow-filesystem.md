---
layout: post
title: "ZFS와 Btrfs: Copy-on-Write 파일시스템의 내부 구조와 스냅샷 원리"
date: 2026-08-22
categories: [cs, computer-science]
tags: [zfs, btrfs, cow, copy-on-write, filesystem, snapshot, storage, linux, raid]
---

전통적인 ext4나 NTFS 같은 파일시스템은 저널링(journaling)으로 충돌 일관성을 보장합니다. 하지만 **ZFS**와 **Btrfs**는 전혀 다른 접근법인 **Copy-on-Write(CoW)** — 더 정확히는 **Redirect-on-Write(RoW)** — 방식을 채택하여 데이터 무결성, 스냅샷, RAID 기능을 파일시스템 레벨에서 통합 제공합니다. 이 아티클에서는 두 파일시스템의 내부 구조와 스냅샷 작동 원리를 깊이 있게 살펴봅니다.

---

## Copy-on-Write의 핵심 아이디어

전통 파일시스템의 쓰기 흐름:
```
1. 기존 블록 위치를 찾는다
2. 기존 블록을 덮어쓴다 (in-place update)
3. (저널링의 경우) 로그를 기록한다
```

CoW 파일시스템의 쓰기 흐름:
```
1. 새 데이터를 새로운 블록에 기록한다
2. 메타데이터(포인터)를 새 블록 주소로 업데이트한다
3. 이전 블록은 참조 카운트가 0이 되면 해제한다
```

이를 "Redirect-on-Write"라고도 부릅니다. 기존 데이터를 절대 덮어쓰지 않기 때문에:
- **원자적 업데이트**: 쓰기 도중 크래시가 발생해도 이전 데이터가 보존됨
- **스냅샷이 무료에 가까움**: 기존 블록 참조만 복사하면 됨
- **체크섬 검증**: 읽기 시 데이터 블록의 체크섬을 메타데이터와 비교 가능

---

## ZFS 내부 구조

ZFS는 Sun Microsystems에서 2001년에 개발한 128비트 파일시스템으로, 저장소 관리(RAID), 볼륨 관리, 파일시스템을 단일 스택으로 통합합니다.

### ZFS 계층 구조

```
┌────────────────────────────────────────────────┐
│  DSL (Dataset and Snapshot Layer)              │  ← 스냅샷, 클론, 데이터셋
├────────────────────────────────────────────────┤
│  DMU (Data Management Unit)                    │  ← 트랜잭션 그룹, CoW 오브젝트
├────────────────────────────────────────────────┤
│  SPA (Storage Pool Allocator)                  │  ← 블록 할당, VDEV 관리
├────────────────────────────────────────────────┤
│  VDEV (Virtual Device)                         │  ← 물리 디스크 추상화
│  [mirror] [raidz] [raidz2] [disk] [SSD]       │
└────────────────────────────────────────────────┘
```

### Uberblock과 트랜잭션 그룹

ZFS의 모든 쓰기는 **트랜잭션 그룹(Transaction Group, TXG)** 단위로 처리됩니다. 기본적으로 5초마다 현재 TXG가 커밋됩니다.

```
Uberblock (ZFS의 수퍼블록):
┌────────────────────────────────────────┐
│  magic: 0x00BAB10C                     │
│  version: 5000                         │
│  txg: 12345  ← 현재 커밋된 TXG 번호    │
│  timestamp: 1724284800                 │
│  rootbp: [MOS 루트 블록 포인터]         │
└────────────────────────────────────────┘
```

ZFS는 256개의 Uberblock 슬롯을 번갈아 가며 사용합니다(링 버퍼). 가장 높은 유효 TXG를 가진 Uberblock이 현재 루트가 됩니다. 크래시 후에도 이전 Uberblock으로 롤백 가능합니다.

### dnode와 블록 포인터

ZFS의 모든 데이터와 메타데이터는 **dnode**로 표현됩니다. 각 dnode는 **블록 포인터(block pointer)** 를 통해 실제 데이터를 가리킵니다:

```
blkptr_t (128바이트):
┌─────────────────────────────────────────────────┐
│ DVA[0]: (VDEV 0의 오프셋)                        │  ← 미러링/RAIDZ 시 복본
│ DVA[1]: (VDEV 1의 오프셋)                        │
│ DVA[2]: (VDEV 2의 오프셋)                        │
│ prop: (크기, 압축, 체크섬 타입, 레벨 등)           │
│ cksum: SHA-256/fletcher4 체크섬                   │  ← 데이터 무결성
│ birth_txg: 12340                                  │  ← 스냅샷 효율화용
└─────────────────────────────────────────────────┘
```

`birth_txg`는 스냅샷에서 변경된 블록만 찾는 데 사용됩니다 (`zfs send -i`의 원리).

### ZFS 스냅샷 작동 원리

```
스냅샷 생성 전:
FileSystem A → dnode_tree → [B1][B2][B3]

스냅샷 생성 (zfs snapshot pool/data@snap1):
FileSystem A → dnode_tree_v2 → [B1][B2][B3]  (참조 증가)
Snapshot snap1 → dnode_tree   → [B1][B2][B3]

파일 수정 후 (file.txt 변경):
FileSystem A → dnode_tree_v2 → [B1_new][B2][B3]  ← B1이 새 위치에 기록됨
Snapshot snap1 → dnode_tree  → [B1    ][B2][B3]  ← B1 참조 유지됨

공간 사용: B1_new 크기만큼만 추가 사용
```

스냅샷은 변경된 블록에 대해서만 공간을 차지합니다.

---

## Btrfs 내부 구조

Btrfs(B-tree filesystem)는 Oracle이 2007년에 개발을 시작한 Linux 기본 CoW 파일시스템입니다. 핵심은 모든 데이터와 메타데이터를 **CoW B-tree** 로 관리하는 것입니다.

### Btrfs B-tree 구조

Btrfs는 여러 종류의 B-tree를 사용합니다:

```
┌────────────────────────────────────────────────────┐
│  Superblock                                        │
│   └─→ Root tree (B-tree of B-trees)               │
│         ├─→ Extent tree (블록 할당 관리)             │
│         ├─→ Chunk tree (논리→물리 주소 매핑)          │
│         ├─→ Dev tree (장치 관리)                     │
│         ├─→ FS tree (파일/디렉토리 메타데이터)        │
│         │     ├─→ inode items                      │
│         │     ├─→ dir items                        │
│         │     └─→ extent data (파일 데이터 포인터)    │
│         └─→ Checksum tree (데이터 체크섬)             │
└────────────────────────────────────────────────────┘
```

### Btrfs 서브볼륨과 스냅샷

서브볼륨은 독립적인 FS 트리를 가진 네임스페이스입니다. 스냅샷은 서브볼륨의 FS 트리 루트 노드에 대한 **공유 참조**입니다:

```
서브볼륨 mydata:
Root → [leaf1: file_a, file_b] → [leaf2: file_c]

스냅샷 snap1 (btrfs subvolume snapshot mydata snap1):
mydata Root → [leaf1][leaf2]   (수정 가능)
snap1  Root → [leaf1][leaf2]   (같은 B-tree 노드 공유)

file_a 수정 (CoW):
mydata: Root_new → [leaf1_new: file_a'][leaf2]  ← leaf1이 새 위치에 복사
snap1:  Root     → [leaf1: file_a_old][leaf2]   ← 원본 leaf1 유지
```

---

## 코드 예제 1: ZFS 명령어로 스냅샷 원리 체험 (bash)

```bash
#!/bin/bash
# ZFS 스냅샷과 클론 실습 (zpool이 있는 환경 필요)

POOL="testpool"
DS="${POOL}/mydata"

# 1. ZFS 데이터셋 생성 (임시 파일로 풀 시뮬레이션)
dd if=/dev/zero of=/tmp/zfs_disk.img bs=1M count=512 2>/dev/null
zpool create $POOL /tmp/zfs_disk.img
zfs create $DS
zfs set compression=lz4 $DS    # LZ4 압축 활성화
zfs set checksum=sha256 $DS     # SHA-256 체크섬

# 2. 초기 데이터 작성
echo "Hello, ZFS!" > /mnt/$DS/file1.txt
echo "Important data" > /mnt/$DS/file2.txt
sync

# 3. 스냅샷 생성 (거의 즉시, 추가 공간 0 사용)
zfs snapshot ${DS}@snap1
echo "스냅샷 생성 후 공간 사용:"
zfs list -o name,used,referenced,written ${DS}@snap1

# 4. 데이터 수정
echo "Modified content" > /mnt/$DS/file1.txt
dd if=/dev/urandom of=/mnt/$DS/bigfile.bin bs=1M count=100 2>/dev/null
sync

# 5. 스냅샷은 이전 상태 보존
echo "현재 file1: $(cat /mnt/$DS/file1.txt)"
echo "스냅샷의 file1: $(cat /mnt/${DS}/.zfs/snapshot/snap1/file1.txt)"

# 6. 차이 확인 (변경된 블록만 추적)
zfs diff ${DS}@snap1 $DS
# 출력 예: M	/mydata/file1.txt   (M=Modified)
#           +	/mydata/bigfile.bin  (+=Added)

# 7. 스냅샷 공간 사용량 (변경된 블록만 계산)
echo "\n공간 사용 현황:"
zfs list -t all -o name,used,referenced

# 8. 특정 파일 복원 (스냅샷에서 롤백)
cp /mnt/${DS}/.zfs/snapshot/snap1/file1.txt /mnt/${DS}/file1.txt
echo "복원된 file1: $(cat /mnt/$DS/file1.txt)"

# 9. 클론 생성 (스냅샷의 쓰기 가능 복사본, 즉시 생성)
zfs clone ${DS}@snap1 ${POOL}/myclone
echo "클론도 동일한 블록을 공유하여 즉시 생성됨"

# 10. 증분 백업 (변경된 블록만 전송)
zfs snapshot ${DS}@snap2
# zfs send -i ${DS}@snap1 ${DS}@snap2 | zfs receive ${POOL}/backup
# snap1→snap2 사이에 변경된 블록만 전송됨 (birth_txg 활용)

echo "\n스냅샷 목록:"
zfs list -t snapshot

# 정리
zpool destroy $POOL
rm -f /tmp/zfs_disk.img
```

---

## 코드 예제 2: Btrfs 서브볼륨과 스냅샷 관리 (Python + subprocess)

```python
import subprocess
import os
import json
from pathlib import Path

class BtrfsManager:
    """Btrfs 서브볼륨/스냅샷 관리 도우미"""
    
    def __init__(self, mount_point: str):
        self.mount = Path(mount_point)
    
    def _run(self, *args: str, check=True) -> subprocess.CompletedProcess:
        result = subprocess.run(
            ['btrfs', *args],
            capture_output=True, text=True, check=False
        )
        if check and result.returncode != 0:
            raise RuntimeError(f"btrfs 오류: {result.stderr.strip()}")
        return result
    
    def list_subvolumes(self) -> list[dict]:
        """서브볼륨 목록 조회"""
        result = self._run('subvolume', 'list', '-a', '-s', '-o',
                          str(self.mount))
        volumes = []
        for line in result.stdout.strip().split('\n'):
            if not line:
                continue
            parts = line.split()
            if len(parts) >= 9:
                volumes.append({
                    'id': parts[1],
                    'gen': parts[3],
                    'top_level': parts[6],
                    'path': ' '.join(parts[8:]),
                    'is_snapshot': 'snapshot' in ' '.join(parts[8:]).lower()
                })
        return volumes
    
    def create_subvolume(self, name: str) -> Path:
        """새 서브볼륨 생성"""
        path = self.mount / name
        self._run('subvolume', 'create', str(path))
        print(f"서브볼륨 생성: {path}")
        return path
    
    def create_snapshot(self, source: str, dest: str, readonly: bool = True) -> Path:
        """
        CoW 스냅샷 생성 (거의 즉시, 공간 사용 없음)
        readonly=True: 백업용, readonly=False: 클론/테스트용
        """
        src = self.mount / source
        dst = self.mount / dest
        args = ['subvolume', 'snapshot']
        if readonly:
            args.append('-r')
        args += [str(src), str(dst)]
        self._run(*args)
        print(f"스냅샷 생성: {src} → {dst} ({'읽기전용' if readonly else '쓰기가능'})")
        return dst
    
    def get_usage(self, path: str) -> dict:
        """서브볼륨 공간 사용량 조회"""
        p = self.mount / path
        result = self._run('subvolume', 'show', str(p), check=False)
        info = {}
        for line in result.stdout.split('\n'):
            line = line.strip()
            if ':' in line:
                key, _, val = line.partition(':')
                info[key.strip()] = val.strip()
        return info
    
    def find_changed_files(self, old_snap: str, new_snap: str) -> list[str]:
        """두 스냅샷 사이에 변경된 파일 목록 (btrfs send 활용)"""
        old = self.mount / old_snap
        new_s = self.mount / new_snap
        # send --no-data로 변경 파일만 추적
        result = subprocess.run(
            ['btrfs', 'send', '--no-data', '-p', str(old), str(new_s)],
            capture_output=True, check=False
        )
        # 실제 환경에서는 btrfs receive 파이프로 처리
        print(f"증분 변경 추적: {old_snap} → {new_snap}")
        return result.stdout[:200] if result.stdout else []
    
    def delete_snapshot(self, name: str):
        """스냅샷 삭제 (공유 블록은 참조 카운트만 감소)"""
        path = self.mount / name
        self._run('subvolume', 'delete', str(path))
        print(f"스냅샷 삭제: {path}")


def demo_btrfs_cow():
    """CoW 스냅샷 데모 (실제 btrfs 마운트 포인트 필요)"""
    print("=== Btrfs CoW 스냅샷 원리 데모 ===\n")
    
    # 이 코드는 Btrfs 마운트 포인트에서 실행해야 합니다
    # 테스트 환경: truncate -s 2G /tmp/btrfs.img && mkfs.btrfs /tmp/btrfs.img
    # mount -o loop /tmp/btrfs.img /mnt/btrfs
    
    btrfs = BtrfsManager('/mnt/btrfs')
    
    print("1. 서브볼륨 생성")
    data_path = btrfs.create_subvolume('production')
    
    print("\n2. 데이터 생성")
    (data_path / 'config.json').write_text('{"version": 1, "env": "prod"}')
    (data_path / 'data.db').write_bytes(os.urandom(1024 * 1024))  # 1MB
    
    print("\n3. 스냅샷 생성 (즉시, 추가 디스크 공간 거의 없음)")
    snap1 = btrfs.create_snapshot('production', 'snapshots/2026-08-22-v1')
    
    print("\n4. 데이터 수정 (CoW: 수정된 블록만 새 위치에 기록)")
    (data_path / 'config.json').write_text('{"version": 2, "env": "prod", "debug": true}')
    
    print("\n5. 스냅샷은 v1 config.json을 그대로 보존")
    print(f"  현재: {(data_path / 'config.json').read_text()}")
    snap1_config = (data_path.parent / 'snapshots/2026-08-22-v1' / 'config.json')
    if snap1_config.exists():
        print(f"  스냅샷: {snap1_config.read_text()}")
    
    print("\n6. 서브볼륨 목록")
    for vol in btrfs.list_subvolumes():
        print(f"  {vol}")


# CoW 파일시스템의 쓰기 증폭(Write Amplification) 분석
def analyze_cow_overhead():
    print("\n=== CoW 파일시스템 쓰기 증폭 분석 ===")
    
    scenarios = [
        {
            "name": "작은 파일 반복 수정 (4KB 파일)",
            "file_size_kb": 4,
            "block_size_kb": 4,
            "snapshot_count": 5,
            "writes": 100
        },
        {
            "name": "대용량 파일 부분 수정 (1GB 파일의 1% 수정)",
            "file_size_kb": 1024 * 1024,
            "block_size_kb": 4,
            "modify_percent": 0.01,
            "snapshot_count": 10,
            "writes": 10
        }
    ]
    
    for s in scenarios:
        print(f"\n시나리오: {s['name']}")
        blocks = s.get('file_size_kb', 4) // s.get('block_size_kb', 4)
        modified = max(1, int(blocks * s.get('modify_percent', 1.0)))
        snapshot_overhead = modified * s.get('snapshot_count', 0)
        print(f"  수정 블록 수: {modified}")
        print(f"  스냅샷 {s.get('snapshot_count', 0)}개 시 추가 공간: {snapshot_overhead * 4}KB")
        print(f"  Btrfs는 변경된 {modified}개 블록만 CoW, 공유 블록 {blocks - modified}개는 참조만 증가")


if __name__ == '__main__':
    analyze_cow_overhead()
    # demo_btrfs_cow()  # 실제 btrfs 마운트 환경에서 주석 해제
```

---

## ZFS vs Btrfs 비교

| 특징 | ZFS | Btrfs |
|-----|-----|-------|
| 개발 주체 | OpenZFS 커뮤니티 | Linux 커뮤니티 (Meta, Oracle 등) |
| 라이선스 | CDDL (Linux 커널과 비호환) | GPL v2 |
| Linux 통합 | 별도 모듈(ZFS on Linux) | 커널 내장 |
| RAID 지원 | RAIDZ, RAIDZ2, RAIDZ3 | RAID 0, 1, 10 (RAID 5/6 실험적) |
| 압축 | LZ4, gzip, zstd | zlib, lzo, zstd |
| 안정성 | 매우 높음 (FreeBSD에서 오래 검증) | 높음 (RAID 5/6은 주의) |
| 스냅샷 | 매우 강력, 재귀 스냅샷 | 서브볼륨 단위 |
| 체크섬 | 메타데이터 + 데이터 전체 | 메타데이터 + 데이터 전체 |
| 복제 | `zfs send/receive` | `btrfs send/receive` |
| 메모리 사용 | 많음 (ARC 캐시) | 적음 |

---

## 주의사항 및 팁

### CoW의 단점: 단편화
CoW 방식은 쓸 때마다 새 위치에 기록하므로 시간이 지나면 파일이 여러 블록에 분산됩니다. Btrfs는 `btrfs filesystem defragment`로, ZFS는 `zpool scrub` + `zfs trim`으로 주기적으로 최적화해야 합니다.

### 작은 파일의 Write Amplification
4KB 이하의 파일을 자주 수정하면 CoW 오버헤드가 커집니다. 데이터베이스 파일(Postgres의 data dir 등)은 `chattr +C`(Btrfs) 또는 ZFS의 `recordsize=16k` 조정으로 CoW를 비활성화하거나 최적화하는 것이 권장됩니다.

### ZFS ARC 캐시 조정
ZFS의 Adaptive Replacement Cache(ARC)는 기본적으로 RAM의 절반까지 사용합니다. 서버에서 다른 애플리케이션과 메모리를 나눠야 한다면:
```bash
echo "options zfs zfs_arc_max=4294967296" >> /etc/modprobe.d/zfs.conf  # 4GB로 제한
```

### 스냅샷 보관 정책
스냅샷이 쌓일수록 공간을 차지합니다. 자동화된 보관 정책 도구:
- ZFS: `zfs-auto-snapshot`, `sanoid`/`syncoid`
- Btrfs: `snapper`, `btrbk`

---

## 참고 자료

- [ZFS vs Btrfs: Architecture, Features, and Stability — Klara Systems](https://klarasystems.com/articles/zfs-vs-btrfs-architects-features-and-stability/)
- [ZFS Administration, Part IX: Copy-on-Write — Aaron Toponce](https://pthree.org/2012/12/14/zfs-administration-part-ix-copy-on-write/)
- [Btrfs: Modern Copy-on-Write Filesystem — Abhik Sarkar](https://www.abhik.ai/concepts/systems/btrfs-filesystem/)
- [Btrfs | Internals for Interns](https://internals-for-interns.com/posts/btrfs-filesystem/)
