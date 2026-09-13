---
layout: post
title: "Git 내부 구조 완전 정복: Objects, Refs, Packfile이 버전 관리를 만드는 방법"
date: 2026-09-13
categories: [cs, computer-science]
tags: [git, version-control, object-store, packfile, merkle-tree, content-addressable]
---

Git은 단순한 버전 관리 도구처럼 보이지만, 그 내부는 정교하게 설계된 **콘텐츠 주소 기반 파일 시스템(content-addressable file system)**이다. `git commit`, `git push`, `git merge`가 내부적으로 어떤 자료구조를 조작하는지 이해하면, 충돌 해결부터 히스토리 재작성까지 모든 Git 작업이 명확해진다. 이 글에서는 `.git/` 디렉토리 안에서 일어나는 모든 일을 낱낱이 해부한다.

## Git의 핵심: 콘텐츠 주소 기반 오브젝트 저장소

Git의 가장 근본적인 특징은 **모든 것이 SHA-1(또는 SHA-256) 해시로 참조된다**는 점이다. 파일명이나 경로로 데이터를 찾는 것이 아니라, 데이터의 내용으로부터 계산된 40자리 16진수 해시로 접근한다. 이 방식은 두 가지 강력한 보장을 제공한다:

1. **무결성(Integrity)**: 오브젝트의 내용이 조금이라도 바뀌면 해시가 달라지므로 변조를 즉시 탐지할 수 있다.
2. **중복 제거(Deduplication)**: 동일한 내용의 파일은 어떤 커밋에서든, 어떤 경로에 있든 단 하나의 오브젝트로 저장된다.

`.git/objects/` 디렉토리가 이 오브젝트 저장소다. SHA-1의 앞 두 자리가 서브디렉토리, 나머지 38자리가 파일명이 된다.

```
.git/objects/
├── 8a/
│   └── b686eafeb1f44702738c8b0f24f2567c36da6d  (blob)
├── 4b/
│   └── 825dc642cb6eb9a060e54bf8d69288fbee4904  (tree)
├── a1/
│   └── b2c3d4e5f6...                          (commit)
└── pack/
    ├── pack-abc123.pack
    └── pack-abc123.idx
```

## 4가지 오브젝트 타입

### 1. Blob: 파일의 스냅샷

Blob(Binary Large Object)은 단순히 파일의 내용이다. 파일명도, 권한도, 타임스탬프도 없다. 오직 내용만 저장된다.

형식: `blob {내용 바이트 수}\0{내용}`

이 헤더를 포함한 전체 내용을 SHA-1로 해시하고, zlib로 압축해 저장한다.

### 2. Tree: 디렉토리의 스냅샷

Tree는 디렉토리 구조를 표현한다. 각 항목은 모드(파일 타입·권한), 오브젝트 타입, SHA-1, 이름으로 구성된다.

```
100644 blob 8ab686ea...  README.md
100644 blob 3f4e8b2c...  main.go
040000 tree 9d4c7b1a...  src/
```

### 3. Commit: 스냅샷의 메타데이터

Commit은 프로젝트의 특정 시점 스냅샷에 메타데이터를 추가한다. 반드시 루트 tree 오브젝트를 참조하며, 부모 커밋(들)도 참조한다. 이 참조 구조가 커밋 히스토리를 **방향 비순환 그래프(DAG)**로 만든다.

### 4. Tag: 이름 붙인 참조

Annotated Tag는 커밋(또는 다른 오브젝트)에 이름, 설명, 서명을 추가하는 오브젝트다. Lightweight Tag는 단순히 특정 커밋 SHA-1을 가리키는 ref 파일이다.

```python
# Git 오브젝트 저장소를 Python으로 구현
import hashlib
import zlib
import os
import struct
from pathlib import Path

class GitObjectStore:
    """Git의 .git/objects/ 디렉토리를 모방한 오브젝트 저장소"""
    
    def __init__(self, git_dir: str = '/tmp/git-demo/.git'):
        self.objects_dir = Path(git_dir) / 'objects'
        self.objects_dir.mkdir(parents=True, exist_ok=True)
    
    def hash_object(self, content: bytes, obj_type: str = 'blob') -> str:
        """오브젝트를 저장하고 SHA-1 해시를 반환"""
        # Git 오브젝트 형식: "{type} {size}\0{content}"
        header = f'{obj_type} {len(content)}\0'.encode('ascii')
        store = header + content
        
        sha1 = hashlib.sha1(store).hexdigest()
        
        # SHA-1의 앞 2자리: 디렉토리, 나머지 38자리: 파일명
        obj_dir = self.objects_dir / sha1[:2]
        obj_path = obj_dir / sha1[2:]
        
        if not obj_path.exists():
            obj_dir.mkdir(exist_ok=True)
            # zlib 압축 저장
            obj_path.write_bytes(zlib.compress(store))
        
        return sha1
    
    def read_object(self, sha1: str) -> tuple[str, bytes]:
        """SHA-1로 오브젝트 읽기"""
        path = self.objects_dir / sha1[:2] / sha1[2:]
        raw = zlib.decompress(path.read_bytes())
        
        # 헤더와 내용 분리
        null_idx = raw.index(b'\0')
        obj_type = raw[:null_idx].split(b' ')[0].decode()
        content = raw[null_idx + 1:]
        return obj_type, content
    
    def write_tree(self, entries: list[tuple]) -> str:
        """Tree 오브젝트 생성: [(mode, name, sha1_bytes), ...]"""
        content = b''
        # Tree 항목을 이름 순으로 정렬해야 함
        for mode, name, sha1_hex in sorted(entries, key=lambda e: e[1]):
            sha1_bytes = bytes.fromhex(sha1_hex)
            entry = f'{mode} {name}\0'.encode() + sha1_bytes
            content += entry
        return self.hash_object(content, 'tree')
    
    def write_commit(self, tree_sha1: str, message: str,
                     parent: str = None, author: str = 'Dev <dev@example.com>') -> str:
        """Commit 오브젝트 생성"""
        import time
        ts = int(time.time())
        tz = '+0900'
        
        lines = [f'tree {tree_sha1}']
        if parent:
            lines.append(f'parent {parent}')
        lines.append(f'author {author} {ts} {tz}')
        lines.append(f'committer {author} {ts} {tz}')
        lines.append('')
        lines.append(message)
        
        content = '\n'.join(lines).encode()
        return self.hash_object(content, 'commit')


# 사용 예시: 미니 커밋 시뮬레이션
store = GitObjectStore()

# 1. 파일 내용을 Blob으로 저장
main_py = b'def hello():\n    print("Hello, Git!")\n'
main_sha1 = store.hash_object(main_py)
print(f'blob: {main_sha1}')

readme = b'# My Project\nA demo of Git internals.\n'
readme_sha1 = store.hash_object(readme)
print(f'blob: {readme_sha1}')

# 2. 디렉토리 구조를 Tree로 저장
tree_sha1 = store.write_tree([
    ('100644', 'main.py', main_sha1),
    ('100644', 'README.md', readme_sha1),
])
print(f'tree: {tree_sha1}')

# 3. Commit 생성
commit_sha1 = store.write_commit(tree_sha1, 'Initial commit')
print(f'commit: {commit_sha1}')

# 4. 두 번째 커밋 — 파일 수정
updated_main = b'def hello():\n    print("Hello, Git Internals!")\n'
updated_sha1 = store.hash_object(updated_main)

# README.md는 변경 없음 → 같은 blob 재사용(중복 제거)
tree2_sha1 = store.write_tree([
    ('100644', 'main.py', updated_sha1),
    ('100644', 'README.md', readme_sha1),  # 동일 SHA-1 재사용!
])
commit2_sha1 = store.write_commit(tree2_sha1, 'Update greeting', parent=commit_sha1)
print(f'commit2: {commit2_sha1}')

# 5. 오브젝트 읽기 검증
obj_type, content = store.read_object(main_sha1)
print(f'\n타입: {obj_type}')
print(f'내용: {content.decode()}')
```

## Refs: 해시의 사람 이름

40자리 SHA-1을 매번 기억하는 것은 불편하다. **Refs(참조)**는 SHA-1에 사람이 읽을 수 있는 이름을 붙인다.

```
.git/
├── HEAD              → "ref: refs/heads/main"
└── refs/
    ├── heads/
    │   ├── main      → "a1b2c3d4..."  (로컬 브랜치)
    │   └── feature   → "e5f6g7h8..."
    ├── remotes/
    │   └── origin/
    │       └── main  → "i9j0k1l2..."
    └── tags/
        └── v1.0      → "m3n4o5p6..."  (lightweight tag)
```

`HEAD`는 현재 체크아웃된 브랜치나 커밋을 가리키는 특수 ref다. `git checkout feature`는 단순히 `HEAD` 파일의 내용을 `ref: refs/heads/feature`로 바꾸는 파일 수정이다.

**브랜치는 단지 파일 하나다.** `main` 브랜치는 `.git/refs/heads/main`이라는 41바이트짜리 텍스트 파일(SHA-1 + 줄바꿈)에 불과하다. 브랜치를 생성하는 `git branch new-feature`는 이 파일을 하나 더 만드는 것이고, 브랜치 삭제는 파일을 지우는 것이다. 이 때문에 Git 브랜치는 O(1)로 생성·삭제된다.

## Packfile: 오브젝트의 고압 압축

프로젝트가 성장하면 loose 오브젝트(개별 파일) 수가 수만 개를 넘는다. `git gc`나 `git push` 시 Git은 이를 **Packfile**로 압축한다.

```bash
# Git 내부 구조 탐색 — plumbing 명령어 사용

# 1. blob 오브젝트 직접 생성
$ echo 'Hello, Git Internals!' | git hash-object -w --stdin
7df5a4c9e80d54b2de0618c15f53c4b0a9d31b6e

# 2. 오브젝트 타입 확인
$ git cat-file -t 7df5a4c9e80d54b2de0618c15f53c4b0a9d31b6e
blob

# 3. 오브젝트 내용 확인
$ git cat-file -p 7df5a4c9e80d54b2de0618c15f53c4b0a9d31b6e
Hello, Git Internals!

# 4. 현재 HEAD가 가리키는 tree 확인
$ git cat-file -p HEAD^{tree}
100644 blob 3f4e8b2c... .gitignore
100644 blob 9d4c7b1a... README.md
040000 tree 8ab686ea... src

# 5. commit 오브젝트 전체 내용
$ git cat-file -p HEAD
tree 4b825dc642cb6eb9a060e54bf8d69288fbee4904
parent a1b2c3d4e5f67890abcdef1234567890abcdef12
author Alice <alice@example.com> 1726200000 +0900
committer Alice <alice@example.com> 1726200000 +0900

Add main feature

# 6. Packfile 생성 (gc)
$ git gc
Counting objects: 234, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (198/198), done.
Writing objects: 100% (234/234), done.
Total 234 (delta 87), reused 0 (delta 0)

# 7. Packfile 내용 검증
$ git verify-pack -v .git/objects/pack/pack-abc123.idx | head -5
7df5a4c9... blob   22 31 1234      # SHA1, 타입, 크기, pack 내 오프셋
4b825dc6... tree   86 93 1265
# "delta" 항목은 다른 오브젝트와의 차이만 저장됨
a1b2c3d4... commit 220 185 1358 1 e5f6g7h8  # 마지막 두 필드: delta base
```

Packfile 내부에서 Git은 **델타 압축**을 사용한다. 비슷한 오브젝트끼리 묶어, 하나는 전체를 저장하고 나머지는 차이분(delta)만 저장한다. 파일의 연속적인 버전들은 서로 유사하므로 엄청난 압축률을 얻는다.

## `git merge`와 `git rebase`의 내부 동작

**Merge**: 두 커밋의 공통 조상(merge base)을 찾아 3-way merge를 수행한다. 새로운 merge commit을 만들며, 이 커밋은 두 개의 parent를 갖는다. DAG가 합쳐진다.

**Rebase**: `feature` 브랜치의 커밋들을 하나씩 꺼내어 `main`의 HEAD 위에 **새로운 커밋으로 재생성**한다. SHA-1이 완전히 달라진다. DAG가 선형으로 정리된다. 히스토리를 재작성하므로 이미 push된 브랜치에 사용하면 위험하다.

**Fast-forward**: `main`이 `feature`의 직접 조상인 경우, 새 커밋 없이 `main` ref를 `feature`가 가리키는 커밋으로 이동시키기만 하면 된다. O(1) 작업이다.

## 주의사항과 실전 팁

**1. `git reflog`로 삭제된 커밋 복구**  
ref가 이동하면 이전 커밋은 접근할 수 없게 보이지만, 오브젝트는 `git gc`가 실행되기 전까지 보존된다. `git reflog`는 HEAD의 이동 이력을 보여주므로 실수로 삭제한 커밋을 복구할 수 있다.

**2. `.git/objects/` 수동 검사**  
오브젝트 손상을 의심할 때는 `git fsck`로 전체 오브젝트 그래프 무결성을 검사한다.

**3. Shallow Clone과 Partial Clone**  
`git clone --depth 1`은 최신 커밋만 포함된 packfile을 받아온다. 오래된 커밋 오브젝트 없이도 체크아웃이 가능한 이유는 tree와 blob만 있으면 되기 때문이다.

**4. Commit SHA-1 충돌**  
이론적으로 SHA-1 충돌 공격이 가능하다(SHAttered). Git은 2023년부터 SHA-256 오브젝트 포맷을 점진적으로 도입하고 있다 (`git init --object-format=sha256`).

**5. 브랜치 전략과 오브젝트 수**  
브랜치가 많아도 오브젝트 저장소는 증가하지 않는다. 브랜치는 단지 ref 파일이고, 오브젝트는 커밋 자체에 달려있다.

## 참고 자료
- [Pro Git Book - Git Internals](https://git-scm.com/book/en/v2/Git-Internals-Plumbing-and-Porcelain)
- [Git Object Model](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects)
- [Git Packfile Format](https://git-scm.com/docs/pack-format)
- [SHA-1 Collision Attack and Git](https://git-scm.com/docs/hash-function-transition)
