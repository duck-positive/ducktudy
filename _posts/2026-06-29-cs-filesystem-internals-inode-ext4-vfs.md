---
layout: post
title: "파일시스템 내부 구조 완전 정복: inode, ext4 디스크 레이아웃, VFS, 저널링까지"
date: 2026-06-29
categories: [cs, computer-science]
tags: [filesystem, inode, ext4, vfs, journaling, linux, storage, os, kernel]
---

## 개념 설명: 파일시스템이란 무엇인가

파일시스템은 저장 장치(HDD, SSD, NVMe)에 데이터를 어떻게 조직화하고 검색할지 정의하는 구조다. "파일"이라는 추상화를 제공하여 애플리케이션이 원시 블록(raw block)을 직접 다루지 않고도 데이터를 저장하고 불러올 수 있게 한다.

Linux에서 파일시스템은 다음과 같은 계층 구조로 동작한다:

```
사용자 공간: open(), read(), write(), stat()
          ↓  (시스템콜)
VFS (Virtual File System): 공통 추상화 계층
          ↓
구체적 파일시스템 드라이버: ext4 / btrfs / xfs / tmpfs / nfs
          ↓
블록 디바이스 레이어: /dev/sda / /dev/nvme0n1
          ↓
하드웨어: HDD, SSD, NVMe
```

**VFS(Virtual File System)**는 Linux 커널 내의 인터페이스 계층으로, 모든 파일시스템 구현체가 공통 구조체(`inode`, `dentry`, `file`, `super_block`)를 구현하도록 강제한다. 덕분에 사용자 프로그램은 로컬 ext4든 원격 NFS든 동일한 POSIX 시스템콜로 접근할 수 있다.

---

## inode: 파일의 진짜 정체

### inode가 담고 있는 것

**inode(Index Node)**는 파일의 **내용이 아닌 메타데이터**를 저장하는 고정 크기 구조체다. ext4에서 기본 inode 크기는 256바이트다.

놀랍게도 **파일 이름은 inode에 없다**. 파일 이름은 **디렉토리 엔트리(dentry)**에 저장되며, dentry가 inode 번호를 가리킨다. 따라서 서로 다른 이름(하드링크)이 동일한 inode를 가리킬 수 있다.

inode에 저장되는 정보:
- **파일 유형**: 일반 파일, 디렉토리, 심볼릭 링크, FIFO, 소켓, 블록/문자 디바이스
- **권한**: 소유자(uid), 그룹(gid), 권한 비트(rwxrwxrwx + setuid/setgid/sticky)
- **파일 크기**: 바이트 단위
- **타임스탬프**: atime(최근 접근), mtime(내용 수정), ctime(inode 변경), crtime(파일 생성, ext4 전용)
- **링크 카운트**: 이 inode를 가리키는 하드링크 수. 0이 되면 파일이 삭제된다
- **데이터 블록 포인터**: 실제 파일 내용이 저장된 디스크 블록 위치

### ext4의 extent 트리

전통적인 ext2/ext3는 블록 포인터를 직접(direct 12개), 간접(indirect), 이중 간접(double indirect), 삼중 간접(triple indirect) 방식으로 저장했다. 파일 하나가 수백만 개의 블록 포인터를 가질 수 있어 단편화와 성능 문제가 심각했다.

ext4는 이를 **extent(연속 블록 범위)** 기반으로 교체했다. 하나의 extent는 `{논리 블록 번호, 물리 시작 블록 번호, 연속 블록 수}` 세 값으로 표현된다. 연속 할당된 파일이라면 수천 개의 블록 포인터 대신 **단 하나의 extent**로 표현된다.

inode에는 최대 4개의 extent를 직접 저장(inline extents)하고, 그 이상은 **extent 트리(B+ 트리 유사 구조)**로 확장한다. 이 구조 덕분에 큰 파일의 랜덤 접근 성능이 크게 향상됐다.

---

## ext4 디스크 레이아웃

ext4 파티션은 고정 크기의 **블록 그룹(Block Group)**으로 분할된다. 기본 블록 크기는 4096바이트이며, 각 블록 그룹은 약 128MiB를 담당한다.

각 블록 그룹의 구성:
1. **슈퍼블록(Superblock)**: 파일시스템 전체 메타데이터 — 총 블록 수, inode 수, 블록 크기, 마운트 카운트, UUID. 그룹 0에 주 슈퍼블록이 위치하며, 일부 그룹에 백업 복사본이 존재한다
2. **그룹 디스크립터(Group Descriptor)**: 해당 그룹의 블록 비트맵/inode 비트맵/inode 테이블의 위치와 가용 블록/inode 수
3. **블록 비트맵**: 블록 사용 여부를 비트로 표시 (1 = 사용 중, 0 = 빈 블록)
4. **inode 비트맵**: inode 사용 여부를 비트로 표시
5. **inode 테이블**: 고정 크기(256바이트) inode 구조체의 배열
6. **데이터 블록**: 실제 파일/디렉토리 데이터

파일을 생성하면: ① inode 비트맵에서 빈 inode 할당 → ② 블록 비트맵에서 빈 블록 할당 → ③ inode에 블록 포인터 기록 → ④ 디렉토리 엔트리에 이름↔inode 번호 매핑 기록.

---

## 왜 저널링이 필요한가

파일 수정은 여러 개의 개별 디스크 쓰기를 수반한다. 예를 들어 파일에 데이터를 추가하면:
1. 새 데이터 블록 쓰기
2. inode의 파일 크기, mtime, extent 포인터 업데이트
3. 블록 비트맵 업데이트

이 중간에 전원이 꺼지면 **파일시스템이 불일치 상태**가 된다. inode는 블록을 가리키지만 비트맵에는 해당 블록이 빈 것으로 남아 다른 파일에 재할당될 수 있다. 이런 불일치는 데이터 손상의 원인이 된다.

전통적으로 이 문제는 부팅 시 `fsck`로 전체 파일시스템을 검사해 복구했지만, 수백 GB 디스크에서 fsck는 수 분에서 수십 분이 걸렸다.

**저널링(Journaling)**은 실제 변경 전에 변경 의도를 **트랜잭션 로그(저널)**에 먼저 기록한다. 크래시 후 재부팅 시 저널만 재실행(replay)하면 되므로 복구가 수 초 이내로 끝난다.

### ext4의 저널링 모드 (jbd2)

ext4는 `jbd2(Journaling Block Device 2)` 커널 모듈을 통해 저널링을 구현한다. 저널 자체는 파일시스템 내부의 숨겨진 파일(inode 8)이다.

| 마운트 옵션 | 저널링 대상 | 성능 | 안전성 |
|-----------|-----------|------|--------|
| `data=writeback` | 메타데이터만 | 가장 빠름 | 크래시 후 파일 내용이 오래된 데이터로 덮어쓰일 수 있음 |
| `data=ordered` | 메타데이터 저널 + 데이터 선기록 순서 보장 | 기본값, 균형 | 새 메타데이터가 가리키는 데이터는 항상 최신 |
| `data=journal` | 메타데이터 + 데이터 모두 저널 | 가장 느림 | 가장 안전 |

기본값은 `data=ordered`이며, 대부분의 워크로드에서 적합하다. 데이터베이스처럼 자체 WAL(Write-Ahead Log)이 있는 애플리케이션은 `data=writeback`으로 성능을 높이기도 한다.

---

## 실제 구현 예제

### 예제 1: C로 inode 정보 직접 읽기 (stat 시스템콜)

```c
#include <stdio.h>
#include <sys/stat.h>
#include <time.h>
#include <pwd.h>
#include <grp.h>

void print_inode_info(const char *path) {
    struct stat st;
    if (lstat(path, &st) == -1) { /* lstat: 심볼릭 링크 자체를 조회 */
        perror("lstat");
        return;
    }

    printf("=== inode 정보: %s ===\n", path);
    printf("inode 번호    : %lu\n", (unsigned long)st.st_ino);

    printf("파일 유형     : ");
    switch (st.st_mode & S_IFMT) {
        case S_IFREG:  printf("일반 파일\n");       break;
        case S_IFDIR:  printf("디렉토리\n");         break;
        case S_IFLNK:  printf("심볼릭 링크\n");      break;
        case S_IFIFO:  printf("FIFO 파이프\n");      break;
        case S_IFSOCK: printf("소켓\n");             break;
        case S_IFBLK:  printf("블록 디바이스\n");    break;
        case S_IFCHR:  printf("문자 디바이스\n");    break;
        default:       printf("기타\n");             break;
    }

    printf("권한 비트     : %04o\n", st.st_mode & 07777);
    printf("하드링크 수   : %lu\n", (unsigned long)st.st_nlink);

    struct passwd *pw = getpwuid(st.st_uid);
    struct group  *gr = getgrgid(st.st_gid);
    printf("소유자        : %s (uid=%d)\n", pw ? pw->pw_name : "?", st.st_uid);
    printf("그룹          : %s (gid=%d)\n", gr ? gr->gr_name : "?", st.st_gid);
    printf("논리 크기     : %lld 바이트\n", (long long)st.st_size);
    printf("실제 블록 수  : %lld × 512 바이트\n", (long long)st.st_blocks);
    /* 스파스 파일이면 st_size >> st_blocks * 512 */

    char buf[64];
    struct tm *tm_info;
    tm_info = localtime(&st.st_atime);
    strftime(buf, sizeof(buf), "%Y-%m-%d %H:%M:%S", tm_info);
    printf("atime (접근)  : %s\n", buf);

    tm_info = localtime(&st.st_mtime);
    strftime(buf, sizeof(buf), "%Y-%m-%d %H:%M:%S", tm_info);
    printf("mtime (내용)  : %s\n", buf);

    tm_info = localtime(&st.st_ctime);
    strftime(buf, sizeof(buf), "%Y-%m-%d %H:%M:%S", tm_info);
    printf("ctime (inode) : %s\n", buf);
}

int main(int argc, char *argv[]) {
    const char *target = (argc > 1) ? argv[1] : "/etc/hostname";
    print_inode_info(target);

    /* 하드링크 확인: 서로 다른 이름이 같은 inode 번호를 가리키면 하드링크 */
    printf("\n--- 하드링크 탐지 예시 ---\n");
    struct stat a, b;
    stat("/bin/sh", &a);
    stat("/bin/dash", &b);  /* 많은 데비안 계열에서 /bin/sh → /bin/dash */
    printf("/bin/sh   inode: %lu\n", (unsigned long)a.st_ino);
    printf("/bin/dash inode: %lu\n", (unsigned long)b.st_ino);
    printf("동일 파일? %s\n",
           (a.st_ino == b.st_ino && a.st_dev == b.st_dev) ? "예(하드링크)" : "아니오");
    return 0;
}
```

컴파일 및 실행:
```bash
gcc -o inode_info inode_info.c && ./inode_info /etc/hosts
```

### 예제 2: Python으로 파일시스템 상태 분석 및 inode 고갈 경고

```python
import os
import stat
from collections import Counter
from pathlib import Path

def analyze_filesystem(mount_point: str = "/") -> dict:
    """statvfs로 파일시스템 전체 통계 조회."""
    vfs = os.statvfs(mount_point)

    block_size   = vfs.f_frsize    # 실제 블록 크기 (바이트)
    total_blocks = vfs.f_blocks
    free_blocks  = vfs.f_bfree
    total_inodes = vfs.f_files
    free_inodes  = vfs.f_ffree

    total_gb = (total_blocks * block_size) / 1024**3
    free_gb  = (free_blocks  * block_size) / 1024**3
    used_gb  = total_gb - free_gb
    used_pct = (used_gb / total_gb * 100) if total_gb > 0 else 0

    inode_used = total_inodes - free_inodes
    inode_pct  = (inode_used / total_inodes * 100) if total_inodes > 0 else 0

    print(f"=== 파일시스템 통계: {mount_point} ===")
    print(f"블록 크기     : {block_size:,} 바이트")
    print(f"전체 용량     : {total_gb:.2f} GB")
    print(f"사용 중       : {used_gb:.2f} GB  ({used_pct:.1f}%)")
    print(f"남은 용량     : {free_gb:.2f} GB")
    print(f"inode 전체    : {total_inodes:,}")
    print(f"inode 사용 중 : {inode_used:,}  ({inode_pct:.1f}%)")

    if inode_pct > 90:
        print(f"\n  ⚠  경고: inode 사용률 {inode_pct:.1f}%! "
              "용량이 남아도 파일 생성이 불가능해질 수 있습니다.")
    else:
        print(f"\n  ✓  inode 상태 정상")

    return {"total_gb": total_gb, "used_pct": used_pct, "inode_pct": inode_pct}


def file_size_distribution(directory: str, max_files: int = 2000) -> None:
    """디렉토리 내 파일 크기 분포를 샘플링해 출력한다."""
    buckets_def = [
        (0,          4_096,      "0–4 KB     "),
        (4_096,      65_536,     "4–64 KB    "),
        (65_536,     1_048_576,  "64 KB–1 MB "),
        (1_048_576,  float('inf'), "1 MB+      "),
    ]
    counts = Counter()
    total_wasted = 0  # 블록 낭비 (내부 단편화)
    sampled = 0

    for root, dirs, files in os.walk(directory):
        dirs[:] = [d for d in dirs
                   if not os.path.ismount(os.path.join(root, d))]
        for fname in files:
            fpath = os.path.join(root, fname)
            try:
                st = os.lstat(fpath)
                if not stat.S_ISREG(st.st_mode):
                    continue
                fsize = st.st_size
                alloc = st.st_blocks * 512  # 실제 할당된 바이트
                # 스파스 파일이 아닌 경우 내부 단편화 계산
                if alloc >= fsize:
                    total_wasted += alloc - fsize
                for lo, hi, label in buckets_def:
                    if lo <= fsize < hi:
                        counts[label] += 1
                        break
            except (PermissionError, OSError):
                continue
            sampled += 1
            if sampled >= max_files:
                break
        if sampled >= max_files:
            break

    if sampled == 0:
        print("파일을 찾지 못했습니다.")
        return

    print(f"\n=== 파일 크기 분포 ({directory}, 샘플 {sampled}개) ===")
    max_count = max(counts.values(), default=1)
    for _, _, label in buckets_def:
        n = counts[label]
        bar_len = int(n / max_count * 35)
        bar = "█" * bar_len
        print(f"  {label}: {bar} {n}")

    wasted_kb = total_wasted / 1024
    print(f"\n  블록 내부 단편화(추정): {wasted_kb:,.1f} KB "
          f"(샘플 {sampled}개 기준)")


def sparse_file_demo(path: str = "/tmp/sparse_demo.bin") -> None:
    """스파스 파일 생성 및 논리 크기 vs 실제 할당 크기 비교."""
    # 1 GiB 논리 크기의 스파스 파일 생성
    with open(path, "wb") as f:
        f.seek(1024 * 1024 * 1024 - 1)  # 1 GiB - 1 로 이동
        f.write(b'\x00')                  # 마지막 1 바이트만 기록

    st = os.stat(path)
    logical_gb  = st.st_size / 1024**3
    physical_kb = st.st_blocks * 512 / 1024

    print(f"\n=== 스파스 파일 데모: {path} ===")
    print(f"논리 크기 (ls -lh): {logical_gb:.2f} GB")
    print(f"실제 할당 (du -sh) : {physical_kb:.1f} KB")
    print("→ 대부분의 디스크 공간을 낭비하지 않음!")

    os.remove(path)


# 실행
analyze_filesystem("/")
file_size_distribution("/etc", max_files=500)
sparse_file_demo()
```

---

## 주의사항과 실전 팁

### 1. inode 고갈: 용량이 있어도 파일을 만들 수 없다

디스크 공간이 충분해도 inode가 소진되면 파일 생성이 불가능하다. `df -i`로 inode 사용량을 주기적으로 모니터링하라. 소형 파일이 대규모로 생성되는 환경(메일 서버 Maildir, npm `node_modules`, PHP 세션 파일 등)에서 특히 주의한다. `mkfs.ext4 -i 4096`으로 inode 비율을 높게 설정할 수 있다.

### 2. `mtime` vs `ctime` 혼동 금지

`mtime`은 **파일 내용** 수정 시각, `ctime`은 **inode** 변경 시각(권한 변경, 소유자 변경, 하드링크 생성 포함)이다. `rsync`의 기본 변경 탐지는 `mtime + 크기`를 사용한다. 권한만 바꾼 경우 `rsync`가 동기화를 건너뛸 수 있으므로 주의하라.

### 3. `noatime` 마운트 옵션으로 쓰기 I/O 줄이기

파일을 읽을 때마다 `atime`이 갱신되면 읽기 I/O가 추가 쓰기 I/O를 수반한다. `/etc/fstab`에서 `noatime` 또는 `relatime`(기본값보다 atime 갱신 빈도를 줄임)을 사용하면 성능이 개선된다. SSD 환경에서 더욱 효과적이다.

### 4. 저널 크기 튜닝

ext4 저널 기본 크기는 128 MiB다. I/O 부하가 높은 서버에서 저널이 작으면 자주 fsync가 발생해 throughput이 줄어든다. `mkfs.ext4 -J size=512` (MiB)로 늘리거나, 저널을 고속 SSD 파티션에 별도로 배치(`-J device=/dev/nvme0p1`)할 수 있다.

### 5. 스파스 파일 주의사항

스파스 파일은 `cp`로 복사하면 논리 크기 전체를 실제로 쓴다. 스파스 특성을 유지하려면 `cp --sparse=always`를 사용하라. `rsync -a --sparse`, `dd conv=sparse`도 같은 맥락이다. 가상 머신 이미지, Docker 레이어를 다룰 때 이 점을 간과하면 디스크가 순식간에 가득 찰 수 있다.

### 6. 디렉토리도 파일이다

리눅스에서 디렉토리는 `{파일 이름 → inode 번호}` 매핑의 테이블을 내용으로 갖는 특수 파일이다. 파일이 많은 디렉토리는 그 자체로 수 MB짜리 데이터 블록을 점유한다. 수백만 개 파일을 단일 디렉토리에 두면 `ls`나 `readdir()` 속도가 극도로 느려진다. 날짜/해시 기반 서브디렉토리 분산이 표준 해법이다.

---

## 참고 자료
- [ext4 — Wikipedia](https://en.wikipedia.org/wiki/Ext4)
- [ext4 General Information — The Linux Kernel documentation](https://docs.kernel.org/admin-guide/ext4.html)
- [Ext4 Disk Layout — ext4 wiki](https://ext4.wiki.kernel.org/index.php/Ext4_Disk_Layout)
- [ext4 Data Structures and Algorithms — kernel.org](https://www.kernel.org/doc/html/latest/filesystems/ext4/index.html)
