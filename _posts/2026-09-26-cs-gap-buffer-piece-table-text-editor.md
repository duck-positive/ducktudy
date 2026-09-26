---
layout: post
title: "Gap Buffer와 Piece Table 완전 정복: VS Code·Vim이 선택한 텍스트 에디터 자료구조의 비밀"
date: 2026-09-26
categories: [cs, computer-science]
tags: [gap-buffer, piece-table, text-editor, data-structure, vscode, vim, emacs]
---

텍스트 에디터는 가장 흔하게 쓰이는 소프트웨어지만, 그 내부에는 생각보다 정교한 자료구조가 숨어 있다. 단순히 문자열을 배열로 저장하면 안 되는 이유는 무엇일까? Emacs와 Vim이 선택한 **Gap Buffer**와 VS Code가 선택한 **Piece Table**은 이 질문에 서로 다른 방식으로 답한다.

## 1. 왜 단순 문자열로는 부족한가

가장 나이브한 구현은 텍스트를 하나의 연속된 문자 배열로 저장하는 것이다. 그러나 이 방식에는 치명적인 문제가 있다.

- **삽입/삭제의 비용**: 배열 중간에 문자를 삽입하면 뒤의 모든 문자를 한 칸씩 밀어야 한다. 10만 자 문서에서 첫 번째 위치에 문자를 삽입하면 10만 번의 이동이 필요하다. 시간 복잡도 **O(N)**.
- **실제 편집 패턴**: 사용자는 커서 근처를 반복해서 편집하는 경향이 있다(지역성). 문서 전체에 무작위 편집이 균등하게 분포하지 않는다.

텍스트 에디터에서 요구되는 연산과 그 빈도를 생각해보면 다음과 같다.

| 연산 | 빈도 |
|------|------|
| 현재 커서 위치에 삽입 | 매우 빈번 |
| 임의 위치 조회(N번째 문자) | 빈번 |
| 임의 위치 삭제 | 빈번 |
| 블록 삽입/삭제 | 중간 |

이 요구사항을 효율적으로 충족하는 두 가지 주요 자료구조가 Gap Buffer와 Piece Table이다.

## 2. Gap Buffer — 커서 근처에 빈 공간을 두다

Gap Buffer는 Emacs의 핵심 자료구조로 채택된 고전적인 아이디어다. 기본 발상은 단순하다: **편집이 일어날 위치(커서) 주변에 미리 빈 공간(gap)을 두어**, 삽입 연산을 O(1) 분할 상환으로 만들자.

### 구조

```
[ H e l l o _ _ _ _ ,   W o r l d ]
          ↑         ↑
        gap_start  gap_end  (커서 위치)
```

- `gap_start`: gap이 시작하는 인덱스
- `gap_end`: gap이 끝나는 인덱스
- Gap 내부는 garbage 값이지만, `gap_end - gap_start`가 남은 용량이다.
- 논리적 문자 N번째는: `index < gap_start ? buffer[index] : buffer[index + gap_size]`

### 커서 이동

커서를 이동하면 gap도 함께 이동해야 한다. 커서가 왼쪽으로 이동할 때 gap 앞의 문자를 gap 뒤로 복사하고, 오른쪽으로 이동할 때는 반대로 한다.

```
# 커서를 'l' 위치로 이동 후:
[ H e _ _ _ _ l l o ,   W o r l d ]
      ↑         ↑
   gap_start  gap_end
```

gap 이동의 비용은 커서 이동 거리에 비례하므로 O(K)이다(K = 이동 거리). 이것이 Gap Buffer의 약점이다. 파일 상단과 하단을 번갈아 편집하면 gap이 계속 이동하여 성능이 저하된다.

## 3. Piece Table — 절대 원본을 수정하지 않는다

Piece Table은 Microsoft Word에서 사용된 자료구조로, VS Code가 2018년 채택하여 유명해졌다. 핵심 아이디어는 **원본 파일을 절대 수정하지 않고**, 모든 편집은 별도의 추가 전용(append-only) 버퍼에 기록한다는 것이다.

### 구조

두 개의 고정 버퍼와 조각 목록으로 구성된다.

```
원본 버퍼 (original):  "Hello World"
추가 버퍼 (add):       " beautiful"

조각 목록 (pieces):
  Piece 1: { buffer: original, start: 0, length: 5 }  → "Hello"
  Piece 2: { buffer: add,      start: 0, length: 10 } → " beautiful"
  Piece 3: { buffer: original, start: 5, length: 6 }  → " World"

논리적 텍스트: "Hello beautiful World"
```

삽입은 단순히 새 텍스트를 추가 버퍼 끝에 덧붙이고, 해당 위치의 조각을 2~3개로 분할한다. 삭제는 조각을 분할하거나 잘라낸다. **원본 데이터는 절대 변경되지 않는다**.

### Undo/Redo

Piece Table의 강력한 장점 중 하나는 Undo/Redo 구현이 자연스럽다는 것이다. 각 편집 연산 전후의 조각 목록 상태를 스냅샷으로 저장하면 된다. 추가 버퍼는 append-only이므로 이전 상태로 완벽하게 돌아갈 수 있다.

## 4. 실제 구현 예제

### 예제 1: Python으로 Gap Buffer 구현

```python
class GapBuffer:
    """텍스트 에디터용 Gap Buffer 구현"""

    INITIAL_GAP_SIZE = 16

    def __init__(self, initial_text: str = ""):
        gap_size = self.INITIAL_GAP_SIZE
        self._buf = list(initial_text) + [None] * gap_size
        self._gap_start = len(initial_text)
        self._gap_end = len(self._buf)

    @property
    def _gap_size(self) -> int:
        return self._gap_end - self._gap_start

    @property
    def length(self) -> int:
        return len(self._buf) - self._gap_size

    def _move_gap_to(self, pos: int):
        """커서를 pos 위치로 이동 (gap도 함께 이동)"""
        if pos == self._gap_start:
            return
        if pos < self._gap_start:
            # gap을 왼쪽으로 이동: gap 앞 문자들을 gap 뒤로 복사
            count = self._gap_start - pos
            self._buf[self._gap_end - count:self._gap_end] = \
                self._buf[pos:self._gap_start]
            self._gap_start = pos
            self._gap_end -= count
        else:
            # gap을 오른쪽으로 이동: gap 뒤 문자들을 gap 앞으로 복사
            count = pos - self._gap_start
            self._buf[self._gap_start:self._gap_start + count] = \
                self._buf[self._gap_end:self._gap_end + count]
            self._gap_start += count
            self._gap_end += count

    def _grow_gap(self):
        """gap이 꽉 찼을 때 크기를 2배로 늘린다"""
        new_gap_size = max(self.INITIAL_GAP_SIZE, self.length // 2)
        left = self._buf[:self._gap_start]
        right = self._buf[self._gap_end:]
        self._buf = left + [None] * new_gap_size + right
        self._gap_end = self._gap_start + new_gap_size

    def insert(self, pos: int, text: str):
        """pos 위치에 text 삽입"""
        for ch in text:
            if self._gap_size == 0:
                self._grow_gap()
            self._move_gap_to(pos)
            self._buf[self._gap_start] = ch
            self._gap_start += 1
            pos += 1

    def delete(self, pos: int, count: int = 1):
        """pos 위치부터 count개 문자 삭제"""
        self._move_gap_to(pos)
        self._gap_end = min(self._gap_end + count, len(self._buf))

    def get_text(self) -> str:
        """현재 텍스트 반환"""
        return ''.join(
            ch for ch in (self._buf[:self._gap_start] + self._buf[self._gap_end:])
            if ch is not None
        )

    def char_at(self, pos: int) -> str:
        """pos 위치 문자 반환 (O(1))"""
        if pos < self._gap_start:
            return self._buf[pos]
        return self._buf[pos + self._gap_size]


# 사용 예시
buf = GapBuffer("Hello World")
print(f"초기: '{buf.get_text()}'")

buf.insert(5, " beautiful")
print(f"삽입 후: '{buf.get_text()}'")

buf.delete(0, 6)
print(f"삭제 후: '{buf.get_text()}'")

buf.insert(0, "Goodbye ")
print(f"최종: '{buf.get_text()}'")
# → "Goodbye beautiful World"
```

### 예제 2: Python으로 Piece Table 구현

```python
from dataclasses import dataclass, field
from typing import List, Tuple
import copy

@dataclass
class Piece:
    is_original: bool  # True = 원본 버퍼, False = 추가 버퍼
    start: int         # 버퍼 내 시작 인덱스
    length: int        # 조각 길이

class PieceTable:
    """텍스트 에디터용 Piece Table 구현"""

    def __init__(self, original: str = ""):
        self._original = original          # 원본 버퍼 (불변)
        self._added = []                   # 추가 전용 버퍼
        self._pieces: List[Piece] = []
        if original:
            self._pieces.append(Piece(True, 0, len(original)))
        self._history: List[List[Piece]] = []  # undo 스택

    def _get_char(self, piece: Piece, offset: int) -> str:
        buf = self._original if piece.is_original else ''.join(self._added)
        return buf[piece.start + offset]

    def _find_piece(self, pos: int) -> Tuple[int, int]:
        """pos에 해당하는 (조각 인덱스, 조각 내 오프셋) 반환"""
        remaining = pos
        for i, piece in enumerate(self._pieces):
            if remaining <= piece.length:
                return i, remaining
            remaining -= piece.length
        return len(self._pieces), 0

    def _save_state(self):
        self._history.append(copy.deepcopy(self._pieces))

    def insert(self, pos: int, text: str):
        """pos 위치에 text 삽입"""
        if not text:
            return
        self._save_state()

        # 추가 버퍼에 텍스트 기록
        add_start = len(self._added)
        self._added.extend(list(text))
        new_piece = Piece(False, add_start, len(text))

        if pos == self.length:
            self._pieces.append(new_piece)
            return

        idx, offset = self._find_piece(pos)
        piece = self._pieces[idx]

        if offset == 0:
            self._pieces.insert(idx, new_piece)
        else:
            # 기존 조각을 둘로 분할하고 중간에 삽입
            left = Piece(piece.is_original, piece.start, offset)
            right = Piece(piece.is_original, piece.start + offset, piece.length - offset)
            self._pieces[idx:idx+1] = [left, new_piece, right]

    def delete(self, pos: int, count: int):
        """pos 위치부터 count개 문자 삭제"""
        if count <= 0:
            return
        self._save_state()
        end = pos + count

        start_idx, start_off = self._find_piece(pos)
        end_idx, end_off = self._find_piece(end)

        new_pieces = []
        # 시작 조각의 앞부분 보존
        if start_off > 0:
            p = self._pieces[start_idx]
            new_pieces.append(Piece(p.is_original, p.start, start_off))

        # 끝 조각의 뒷부분 보존
        if end_idx < len(self._pieces):
            p = self._pieces[end_idx]
            remaining = p.length - end_off
            if remaining > 0:
                new_pieces.append(Piece(p.is_original, p.start + end_off, remaining))

        # 변경 범위 밖의 조각 유지
        self._pieces = (
            self._pieces[:start_idx] +
            new_pieces +
            self._pieces[end_idx+1:]
        )

    def undo(self) -> bool:
        """마지막 편집 취소"""
        if not self._history:
            return False
        self._pieces = self._history.pop()
        return True

    @property
    def length(self) -> int:
        return sum(p.length for p in self._pieces)

    def get_text(self) -> str:
        """현재 텍스트 반환"""
        result = []
        added_str = ''.join(self._added)
        for piece in self._pieces:
            buf = self._original if piece.is_original else added_str
            result.append(buf[piece.start:piece.start + piece.length])
        return ''.join(result)


# 사용 예시
pt = PieceTable("Hello World")
print(f"초기: '{pt.get_text()}'")

pt.insert(5, " beautiful")
print(f"삽입 후: '{pt.get_text()}'")

pt.delete(0, 6)
print(f"삭제 후: '{pt.get_text()}'")

pt.undo()
print(f"Undo 후: '{pt.get_text()}'")

pt.undo()
print(f"Undo 후: '{pt.get_text()}'")
# 초기 상태로 복원
```

## 5. 두 자료구조 비교와 선택 기준

| 항목 | Gap Buffer | Piece Table |
|------|-----------|-------------|
| 메모리 | 낮음 (버퍼 1개) | 높음 (버퍼 2개 + 조각 목록) |
| 커서 근처 삽입 | O(1) 분할 상환 | O(log N) (균형 트리 구현 시) |
| 커서 이동 | O(K) | O(log N) |
| 임의 위치 접근 | O(1) | O(log N) |
| Undo/Redo | 복잡 (스냅샷 필요) | 자연스러움 |
| 대용량 파일 | 파일 전체를 메모리에 | 원본 파일 참조 가능 |
| 채택 사례 | Emacs, Vim | VS Code, Word |

**Gap Buffer**는 단일 커서 편집이 지역화된 경우에 매우 효율적이다. 구현도 단순하다. Emacs와 Vim처럼 커서 위치 근처에서 집중적으로 편집하는 워크플로에 최적이다.

**Piece Table**은 다중 커서, 넓은 범위 검색-교체, 협업 편집, 대용량 파일 처리에서 강점을 보인다. VS Code가 이를 채택한 이유도 수십만 줄의 파일과 다중 커서 편집을 염두에 뒀기 때문이다.

### 주의사항 및 팁

실제 프로덕션 구현에서는 조각 목록을 단순 배열 대신 **Red-Black Tree** 또는 **AVL Tree**로 관리하여 삽입·삭제·위치 탐색을 O(log N)으로 보장한다. 각 노드에 서브트리의 총 문자 수(offset)를 저장하는 **Order Statistic Tree** 형태로 구현하면 임의 위치 접근도 O(log N)에 가능하다.

현대 텍스트 에디터들은 개행 문자('\n') 위치를 별도로 인덱싱하여 줄 번호 기반 탐색도 O(log N)에 처리한다. VS Code의 `monaco-editor`는 이 두 인덱스를 조합하여 수백만 줄의 파일도 부드럽게 처리한다.

## 참고 자료

- [Piece table - Wikipedia](https://en.wikipedia.org/wiki/Piece_table)
- [Gap Buffer Data Structure - GeeksforGeeks](https://www.geeksforgeeks.org/dsa/gap-buffer-data-structure/)
- [Text Editor: Data Structures - Avery Laird](https://www.averylaird.com/programming/the%20text%20editor/2017/09/30/the-piece-table.html)
- [The Data Structures Behind Text Editors - Medium](https://gauravsarma1992.medium.com/the-data-structures-behind-text-editors-gap-buffers-piece-tables-ropes-and-crdts-8df38a999cce)
