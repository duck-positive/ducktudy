---
layout: post
title: "Language Server Protocol(LSP) 내부 구조 완전 정복: JSON-RPC 통신부터 직접 구현까지"
date: 2026-10-06
categories: [cs, computer-science]
tags: [lsp, language-server-protocol, json-rpc, ide, compiler, clangd, rust-analyzer, developer-tools]
---

## Language Server Protocol이란

VS Code에서 Go 파일을 열면 자동완성이 뜨고, Rust 코드에 오류가 표시된다. PyCharm에서는 Python 함수 정의로 바로 이동할 수 있다. 이 모든 기능은 어떻게 동작할까?

예전에는 각 편집기(VS Code, Vim, Emacs, Sublime Text…)마다 각 언어(Python, Go, Rust, Java…)의 지원을 따로 구현해야 했다. **N개의 편집기 × M개의 언어 = N×M개의 플러그인**이 필요한 구조였다. 유지보수 악몽이다.

**Language Server Protocol(LSP)**은 Microsoft가 2016년 VS Code를 위해 고안한 오픈 프로토콜로, 이 문제를 **N+M** 문제로 바꾼다. 언어별 분석 서버(Language Server)는 LSP를 통해 편집기와 통신하고, 편집기는 언어가 무엇이든 동일한 프로토콜로 Language Server에 질의한다.

- **편집기**: LSP 클라이언트 역할. "커서 위치의 타입이 뭐야?", "이 심볼이 정의된 곳은?" 같은 요청을 보낸다.
- **Language Server**: LSP 서버 역할. 코드를 분석해 응답한다. 대표적으로 `clangd`(C/C++), `rust-analyzer`(Rust), `pyright`(Python), `gopls`(Go), `tsserver`(TypeScript)가 있다.

현재 LSP 3.18 명세가 표준이며, 수백 개의 편집기와 언어 서버가 이 프로토콜을 구현하고 있다.

---

## 왜 LSP가 필요한가

### N×M 문제

```
편집기 없이 LSP 이전:

         Vim    VS Code   Emacs   JetBrains
Python    ✗       ✗        ✗         ✗
Go        ✗       ✗        ✗         ✗
Rust      ✗       ✗        ✗         ✗
Java      ✗       ✗        ✗         ✗

→ 4 × 4 = 16개 플러그인 필요 (각 편집기마다 언어별 독자 구현)
```

```
LSP 도입 후:

Language Servers: pyright, gopls, rust-analyzer, eclipse.jdt.ls
LSP Clients:      Vim, VS Code, Emacs, JetBrains

→ 4 + 4 = 8개 구현으로 동일한 기능 제공
```

### 언어 분석의 복잡성

현대 언어(특히 Rust, TypeScript, C++)의 타입 검사, 자동완성, 리팩터링은 매우 복잡한 분석을 필요로 한다. 이를 각 편집기 플러그인 안에서 구현하면 중복이 심하고 품질도 들쭉날쭉해진다. LSP는 이 분석 로직을 **언어 서버 프로세스**로 분리해 단일 고품질 구현이 모든 편집기에서 사용될 수 있게 한다.

---

## 프로토콜 구조: JSON-RPC over stdio/socket

LSP는 **JSON-RPC 2.0** 위에서 동작한다. 편집기와 언어 서버는 보통 stdio(표준 입출력) 또는 TCP 소켓으로 연결된다.

### 메시지 포맷

모든 메시지는 **HTTP-like 헤더 + JSON 본문** 구조를 가진다:

```
Content-Length: 97\r\n
\r\n
{"jsonrpc":"2.0","id":1,"method":"textDocument/completion","params":{...}}
```

세 가지 메시지 타입이 있다:

```json
// 1. Request (id가 있음 — 응답을 기대)
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "textDocument/hover",
  "params": {
    "textDocument": { "uri": "file:///src/main.go" },
    "position": { "line": 10, "character": 5 }
  }
}

// 2. Response (id가 있음 — Request에 대한 응답)
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "contents": {
      "kind": "markdown",
      "value": "```go\nfunc fmt.Println(a ...any) (n int, err error)\n```"
    }
  }
}

// 3. Notification (id 없음 — 응답 불필요)
{
  "jsonrpc": "2.0",
  "method": "textDocument/publishDiagnostics",
  "params": {
    "uri": "file:///src/main.go",
    "diagnostics": [
      {
        "range": { "start": {"line":5,"character":2}, "end": {"line":5,"character":10} },
        "severity": 1,
        "message": "undefined: fmtt"
      }
    ]
  }
}
```

### 생명주기

LSP 세션의 생명주기는 엄격히 정의되어 있다:

```
클라이언트                        서버
    |                               |
    |--- initialize request ------->|  (클라이언트 기능 알림)
    |<-- initialize response -------|  (서버 기능 알림)
    |--- initialized notification ->|  (세션 시작 확인)
    |                               |
    |--- textDocument/didOpen ----->|  (파일 열림)
    |<-- textDocument/publishDiagnostics (에러 알림)
    |                               |
    |--- textDocument/completion --->|  (자동완성 요청)
    |<-- completion response --------|
    |                               |
    |--- shutdown request --------->|  (종료 요청)
    |<-- shutdown response ----------|
    |--- exit notification -------->|  (프로세스 종료)
```

---

## 실제 구현 예제

### 예제 1: Python으로 최소 Language Server 구현

아래는 hover 기능만 지원하는 최소한의 Language Server를 Python으로 구현한 예다. 실제 `rust-analyzer`나 `pyright`의 기반 구조와 동일하다.

```python
#!/usr/bin/env python3
"""최소 Language Server — hover 응답만 구현"""
import json
import sys
import logging

logging.basicConfig(filename="/tmp/lsp_server.log", level=logging.DEBUG)
log = logging.getLogger(__name__)

def send_message(msg: dict) -> None:
    """JSON-RPC 메시지를 stdout으로 전송"""
    body = json.dumps(msg, ensure_ascii=False)
    header = f"Content-Length: {len(body.encode())}\r\n\r\n"
    sys.stdout.buffer.write((header + body).encode())
    sys.stdout.buffer.flush()

def read_message() -> dict | None:
    """stdin에서 JSON-RPC 메시지 읽기"""
    header_line = sys.stdin.buffer.readline().decode()
    if not header_line:
        return None
    content_length = int(header_line.strip().split(": ")[1])
    sys.stdin.buffer.readline()  # 빈 줄 (\r\n) 소비
    body = sys.stdin.buffer.read(content_length).decode()
    return json.loads(body)

def handle_initialize(req_id: int) -> None:
    send_message({
        "jsonrpc": "2.0", "id": req_id,
        "result": {
            "capabilities": {
                "hoverProvider": True,
                "textDocumentSync": 1  # Full sync
            },
            "serverInfo": {"name": "mini-ls", "version": "0.1.0"}
        }
    })

def handle_hover(req_id: int, params: dict) -> None:
    uri = params["textDocument"]["uri"]
    line = params["position"]["line"]
    char = params["position"]["character"]
    send_message({
        "jsonrpc": "2.0", "id": req_id,
        "result": {
            "contents": {
                "kind": "markdown",
                "value": f"**Hover at** `{uri}` line={line}, char={char}"
            }
        }
    })

def main():
    open_docs: dict[str, str] = {}  # uri -> 내용

    while True:
        msg = read_message()
        if msg is None:
            break
        method = msg.get("method", "")
        req_id = msg.get("id")
        params = msg.get("params", {})
        log.debug("← %s", method)

        match method:
            case "initialize":
                handle_initialize(req_id)
            case "initialized":
                pass  # notification, 응답 불필요
            case "textDocument/didOpen":
                uri = params["textDocument"]["uri"]
                open_docs[uri] = params["textDocument"]["text"]
            case "textDocument/hover":
                handle_hover(req_id, params)
            case "shutdown":
                send_message({"jsonrpc": "2.0", "id": req_id, "result": None})
            case "exit":
                sys.exit(0)

if __name__ == "__main__":
    main()
```

이 서버를 Neovim이나 VS Code에 연결하면 실제로 hover 툴팁이 표시된다!

### 예제 2: LSP 클라이언트 — Go로 Language Server 프로세스 구동

편집기 플러그인이 Language Server를 어떻게 시작하고 통신하는지 Go 코드로 살펴본다.

```go
package lsp

import (
    "bufio"
    "encoding/json"
    "fmt"
    "io"
    "os/exec"
    "strconv"
    "strings"
    "sync"
    "sync/atomic"
)

type Client struct {
    cmd     *exec.Cmd
    writer  io.Writer
    reader  *bufio.Reader
    nextID  atomic.Int64
    pending sync.Map // id -> chan response
}

type Message struct {
    JSONRPC string          `json:"jsonrpc"`
    ID      *int64          `json:"id,omitempty"`
    Method  string          `json:"method,omitempty"`
    Params  json.RawMessage `json:"params,omitempty"`
    Result  json.RawMessage `json:"result,omitempty"`
}

// Language Server 프로세스 시작
func NewClient(command string, args ...string) (*Client, error) {
    cmd := exec.Command(command, args...)
    stdin, _ := cmd.StdinPipe()
    stdout, _ := cmd.StdoutPipe()
    if err := cmd.Start(); err != nil {
        return nil, err
    }
    c := &Client{cmd: cmd, writer: stdin, reader: bufio.NewReader(stdout)}
    go c.readLoop()
    return c, nil
}

// 메시지 전송
func (c *Client) Send(method string, params any) (int64, error) {
    id := c.nextID.Add(1)
    p, _ := json.Marshal(params)
    msg := Message{JSONRPC: "2.0", ID: &id, Method: method, Params: p}
    body, _ := json.Marshal(msg)
    header := fmt.Sprintf("Content-Length: %d\r\n\r\n", len(body))
    _, err := fmt.Fprint(c.writer, header+string(body))
    return id, err
}

// 수신 루프 — 서버 응답을 파싱해 대기 중인 채널에 전달
func (c *Client) readLoop() {
    for {
        line, _ := c.reader.ReadString('\n')
        if !strings.HasPrefix(line, "Content-Length:") {
            continue
        }
        length, _ := strconv.Atoi(strings.TrimSpace(strings.Split(line, ":")[1]))
        c.reader.ReadString('\n') // 빈 줄

        buf := make([]byte, length)
        io.ReadFull(c.reader, buf)

        var msg Message
        json.Unmarshal(buf, &msg)
        if msg.ID != nil {
            if ch, ok := c.pending.Load(*msg.ID); ok {
                ch.(chan Message) <- msg
                c.pending.Delete(*msg.ID)
            }
        }
    }
}

// Request/Response 동기 호출
func (c *Client) Call(method string, params any) (json.RawMessage, error) {
    ch := make(chan Message, 1)
    id, err := c.Send(method, params)
    if err != nil {
        return nil, err
    }
    c.pending.Store(id, ch)
    resp := <-ch
    return resp.Result, nil
}
```

---

## 주요 LSP 기능과 내부 동작

### Incremental Document Sync

`textDocument/didChange`는 파일 전체를 보내는 Full Sync(TextDocumentSyncKind = 1)와 변경된 부분만 보내는 Incremental Sync(= 2)를 지원한다. `rust-analyzer` 같은 고성능 서버는 Incremental Sync를 사용해 타이핑마다 전체 파일을 재분석하지 않는다.

### Semantic Tokens

LSP 3.16부터 추가된 `textDocument/semanticTokens` 요청은 편집기의 단순 정규식 기반 구문 강조를 언어 서버의 의미 분석 기반 강조로 대체한다. 예를 들어 Rust에서 `Copy` trait을 구현하는 타입과 그렇지 않은 타입을 다른 색으로 표시할 수 있다.

### 취소와 재시작

자동완성처럼 빠른 응답이 필요한 기능은 `$/cancelRequest` 알림으로 이전 요청을 취소한다. 언어 서버는 취소된 요청을 즉시 중단해야 한다.

---

## 주의사항 및 팁

### 1. 프로세스 수명 관리

Language Server는 편집기 프로세스의 자식 프로세스로 실행된다. 편집기가 비정상 종료하면 서버도 종료되어야 한다. 실제 구현에서는 stdin EOF 감지, heartbeat 알림(`$/alive`) 등으로 이를 보장한다.

### 2. URI 처리

LSP는 파일 경로를 `file:///path/to/file.go` 형식의 URI로 전달한다. Windows에서는 `file:///C:/path/to/file.go`처럼 드라이브 레터가 포함된다. URI 인코딩/디코딩을 올바르게 처리하지 않으면 Windows에서 파일을 찾지 못하는 버그가 발생한다.

### 3. LSP는 상태 있는(stateful) 프로토콜

서버는 `textDocument/didOpen`, `didChange`, `didClose` 이벤트로 현재 열린 파일들의 상태를 메모리에 유지한다. 클라이언트가 동기화를 깨트리면(예: didChange를 잃어버림) 분석 결과가 오염된다.

### 4. 확장 기능 협상

`initialize` 요청에서 클라이언트와 서버는 각자 지원하는 기능(capabilities)을 교환한다. 서버가 `hoverProvider: true`를 선언해도 클라이언트가 hover를 지원하지 않으면 사용할 수 없다. 상호 협상을 통해 최대 공통 기능 집합이 활성화된다.

---

## 마무리

LSP는 편집기 생태계를 근본적으로 바꾼 프로토콜이다. 단순한 JSON-RPC 레이어 위에 문서 동기화, 진단, 자동완성, 정의 이동, 리팩터링 등 수십 가지 기능이 표준화되어 있다. `rust-analyzer` 하나를 구현하면 VS Code, Vim, Emacs, JetBrains 모두에서 최고 수준의 Rust 지원을 제공할 수 있다. 언어 도구를 개발한다면 LSP 위에 구현하는 것이 사실상 표준이 되었다.

## 참고 자료
- [LSP 3.18 공식 명세 — Microsoft](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.18/specification)
- [LSP 구현 가이드 — LSP 공식 사이트](https://microsoft.github.io/language-server-protocol/implementors/servers/)
- [rust-analyzer 아키텍처 문서](https://github.com/rust-lang/rust-analyzer/blob/master/docs/dev/architecture.md)
- [lsif — Language Server Index Format](https://microsoft.github.io/language-server-protocol/specifications/lsif/0.4.0/specification/)
