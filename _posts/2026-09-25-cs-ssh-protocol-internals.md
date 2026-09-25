---
layout: post
title: "SSH 프로토콜 내부 구조 완전 정복: 안전한 원격 접속의 암호화와 인증 메커니즘"
date: 2026-09-25
categories: [cs, computer-science]
tags: [ssh, cryptography, network, protocol, security, authentication, key-exchange]
---

SSH(Secure Shell)는 개발자가 매일 사용하는 원격 접속 도구이지만, 그 내부에서 어떤 일이 일어나는지 깊이 이해하는 사람은 많지 않습니다. `ssh user@server` 한 줄이 실행되는 순간, 수십 가지의 암호학적 연산과 프로토콜 협상이 벌어집니다. 이 글에서는 SSH-2 프로토콜의 3계층 구조부터 키 교환, 인증, 채널 멀티플렉싱까지 내부 동작을 완전히 해부합니다.

## SSH가 왜 필요한가

1990년대 초 인터넷은 telnet, rsh, rlogin 같은 원격 접속 프로토콜을 사용했습니다. 이 프로토콜들은 패킷을 평문으로 전송해 네트워크를 감청하면 패스워드와 명령어가 그대로 노출되었습니다. 1994년 핀란드 헬싱키 대학의 Tatu Ylönen이 SSH 1.0을 개발해 이 문제를 해결했습니다.

현재 사용되는 SSH-2는 RFC 4251~4256으로 표준화된 프로토콜로, 세 가지 핵심 보안 목표를 제공합니다:

- **기밀성(Confidentiality)**: AES-CTR, ChaCha20-Poly1305 등 대칭키 암호화
- **무결성(Integrity)**: HMAC-SHA2, Poly1305 기반 메시지 인증
- **인증(Authentication)**: 서버와 클라이언트 모두 상호 인증

## SSH-2 프로토콜의 3계층 구조

SSH-2는 명확히 분리된 세 레이어로 구성됩니다:

```
┌─────────────────────────────────────────────┐
│   Connection Layer (RFC 4254)               │
│   채널 멀티플렉싱 - session, X11, TCP 포워딩 │
├─────────────────────────────────────────────┤
│   Authentication Layer (RFC 4252)           │
│   사용자 인증 - publickey, password, GSSAPI │
├─────────────────────────────────────────────┤
│   Transport Layer (RFC 4253)                │
│   키 교환, 서버 인증, 암호화, 압축          │
├─────────────────────────────────────────────┤
│   TCP/IP (Port 22)                          │
└─────────────────────────────────────────────┘
```

### Transport Layer: 키 교환과 암호화 협상

연결이 시작되면 클라이언트와 서버는 먼저 평문으로 버전 문자열을 교환합니다:

```
SSH-2.0-OpenSSH_9.7
SSH-2.0-OpenSSH_9.3
```

이후 `SSH_MSG_KEXINIT` 패킷을 교환하여 지원하는 알고리즘 목록을 협상합니다:

```
kex_algorithms:               curve25519-sha256, ecdh-sha2-nistp256, ...
server_host_key_algorithms:   ssh-ed25519, ecdsa-sha2-nistp256, rsa-sha2-512, ...
encryption_algorithms:        chacha20-poly1305@openssh.com, aes128-ctr, ...
mac_algorithms:               hmac-sha2-256, hmac-sha2-512, ...
compression_algorithms:       none, zlib@openssh.com
```

협상이 완료되면 Diffie-Hellman 또는 ECDH 키 교환을 수행합니다. 현대적인 구현은 `curve25519-sha256`을 사용합니다:

```
# ECDH over Curve25519 키 교환 흐름
Client → SSH_MSG_KEX_ECDH_INIT  (Q_C: 클라이언트 공개키)
Server → SSH_MSG_KEX_ECDH_REPLY (K_S: 서버 호스트 공개키, Q_S: 서버 공개키, sig: 서명)
```

서버는 `K_S`(서버의 영구 호스트 키)로 교환 해시 `H`에 서명합니다. 클라이언트는 `~/.ssh/known_hosts`에 저장된 서버 공개키와 비교해 MITM(중간자 공격)을 방어합니다. 이 과정에서 세션 키 `K`가 도출되어 이후 모든 통신을 암호화합니다.

## 인증 메커니즘 심층 분석

Transport Layer에서 보안 채널이 열리면 Authentication Layer가 사용자를 인증합니다.

### 공개키 인증 (가장 권장되는 방법)

```
1. Client → SSH_MSG_USERAUTH_REQUEST (method: publickey, algorithm: ssh-ed25519, key: PK)
2. Server  → SSH_MSG_USERAUTH_PK_OK  (알고리즘과 키 허용 확인)
3. Client  → SSH_MSG_USERAUTH_REQUEST (+ sig: session_id || userauth_request에 개인키로 서명)
4. Server  → SSH_MSG_USERAUTH_SUCCESS
```

서버는 `~/.ssh/authorized_keys`에서 클라이언트 공개키를 조회하고, 클라이언트가 보낸 서명을 공개키로 검증합니다. 개인키는 절대 네트워크를 통해 전송되지 않습니다.

### 코드 예제 1: Python paramiko로 공개키 인증 SSH 연결

```python
import paramiko
import io

def ssh_connect_with_key(hostname: str, username: str, key_path: str) -> paramiko.SSHClient:
    client = paramiko.SSHClient()
    # 호스트 키 자동 추가 (프로덕션에서는 known_hosts 검증 필요)
    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    
    private_key = paramiko.Ed25519Key.from_private_key_file(key_path)
    client.connect(
        hostname=hostname,
        username=username,
        pkey=private_key,
        look_for_keys=False,
        allow_agent=False
    )
    return client

def execute_remote_command(client: paramiko.SSHClient, command: str) -> tuple[str, str]:
    stdin, stdout, stderr = client.exec_command(command)
    return stdout.read().decode(), stderr.read().decode()

# 채널을 통한 명령 실행 - SSH Connection Layer의 session 채널 활용
def ssh_port_forward(client: paramiko.SSHClient, remote_host: str, remote_port: int, local_port: int):
    """direct-tcpip 채널: 로컬 포트를 원격 호스트로 포워딩"""
    transport = client.get_transport()
    channel = transport.open_channel(
        "direct-tcpip",
        (remote_host, remote_port),
        ("127.0.0.1", local_port)
    )
    return channel

if __name__ == "__main__":
    client = ssh_connect_with_key("example.com", "ubuntu", "~/.ssh/id_ed25519")
    stdout, stderr = execute_remote_command(client, "uname -r && df -h /")
    print(stdout)
    client.close()
```

## Connection Layer: 채널 멀티플렉싱

SSH의 가장 강력한 기능 중 하나는 단일 TCP 연결 위에 여러 논리적 채널을 동시에 운용하는 채널 멀티플렉싱입니다.

```
단일 TCP 연결 (Port 22)
├── Channel 0: session        (interactive shell or command)
├── Channel 1: direct-tcpip   (local port forwarding: -L)
├── Channel 2: forwarded-tcpip (remote port forwarding: -R)
└── Channel 3: x11            (X11 forwarding)
```

각 채널은 독립적인 흐름 제어 윈도우를 가집니다. `SSH_MSG_CHANNEL_WINDOW_ADJUST` 패킷으로 수신 버퍼 크기를 동적으로 조절합니다.

### 채널 개방 프로세스

```
Client → SSH_MSG_CHANNEL_OPEN      (channel_type: "session", sender_channel: 0, 
                                    initial_window: 2MB, max_packet: 32KB)
Server → SSH_MSG_CHANNEL_OPEN_CONFIRMATION (recipient_channel: 0, sender_channel: 1,
                                             initial_window: 2MB, max_packet: 32KB)
Client → SSH_MSG_CHANNEL_REQUEST   (channel: 1, request_type: "exec", command: "ls -la")
Server → SSH_MSG_CHANNEL_SUCCESS
Server → SSH_MSG_CHANNEL_DATA      (channel: 0, data: "total 48\ndrwxr-x...")
Server → SSH_MSG_CHANNEL_EOF
Server → SSH_MSG_CHANNEL_CLOSE
Client → SSH_MSG_CHANNEL_CLOSE
```

### 코드 예제 2: Go net/ssh로 간단한 SSH 서버 구현

```go
package main

import (
    "crypto/rand"
    "crypto/rsa"
    "fmt"
    "io"
    "log"
    "net"
    "os/exec"

    "golang.org/x/crypto/ssh"
)

func generateHostKey() ssh.Signer {
    privateKey, err := rsa.GenerateKey(rand.Reader, 4096)
    if err != nil {
        log.Fatal(err)
    }
    signer, err := ssh.NewSignerFromKey(privateKey)
    if err != nil {
        log.Fatal(err)
    }
    return signer
}

func handleSession(ch ssh.Channel, requests <-chan *ssh.Request) {
    defer ch.Close()
    for req := range requests {
        switch req.Type {
        case "exec":
            // SSH_MSG_CHANNEL_REQUEST: exec
            // req.Payload 형식: uint32 길이 + 명령어 문자열
            if len(req.Payload) < 4 {
                req.Reply(false, nil)
                continue
            }
            cmdLen := int(req.Payload[0])<<24 | int(req.Payload[1])<<16 |
                int(req.Payload[2])<<8 | int(req.Payload[3])
            command := string(req.Payload[4 : 4+cmdLen])
            
            req.Reply(true, nil)
            cmd := exec.Command("/bin/sh", "-c", command)
            cmd.Stdout = ch
            cmd.Stderr = ch.Stderr()
            cmd.Run()
            
            // exit-status 요청 전송
            exitStatus := make([]byte, 4)
            ch.SendRequest("exit-status", false, exitStatus)
            return
        case "pty-req":
            req.Reply(true, nil)
        case "shell":
            req.Reply(true, nil)
            // 간단한 에코 셸
            io.Copy(ch, ch)
            return
        default:
            if req.WantReply {
                req.Reply(false, nil)
            }
        }
    }
}

func main() {
    config := &ssh.ServerConfig{
        // 패스워드 인증 (데모용 - 프로덕션에서는 공개키 인증 사용)
        PasswordCallback: func(conn ssh.ConnMetadata, password []byte) (*ssh.Permissions, error) {
            if conn.User() == "demo" && string(password) == "password" {
                return &ssh.Permissions{}, nil
            }
            return nil, fmt.Errorf("invalid credentials")
        },
        // 공개키 인증
        PublicKeyCallback: func(conn ssh.ConnMetadata, key ssh.PublicKey) (*ssh.Permissions, error) {
            // authorized_keys 조회 로직 (실제 구현 필요)
            return &ssh.Permissions{
                Extensions: map[string]string{
                    "pubkey-fp": ssh.FingerprintSHA256(key),
                },
            }, nil
        },
    }
    config.AddHostKey(generateHostKey())

    listener, err := net.Listen("tcp", "0.0.0.0:2222")
    if err != nil {
        log.Fatal(err)
    }
    log.Println("SSH server listening on :2222")

    for {
        tcpConn, err := listener.Accept()
        if err != nil {
            continue
        }
        go func(c net.Conn) {
            // Transport Layer: 키 교환 및 암호화 협상
            sshConn, chans, reqs, err := ssh.NewServerConn(c, config)
            if err != nil {
                return
            }
            log.Printf("새 연결: %s (%s)", sshConn.RemoteAddr(), sshConn.ClientVersion())
            
            go ssh.DiscardRequests(reqs)
            
            // Connection Layer: 채널 처리
            for newChan := range chans {
                if newChan.ChannelType() != "session" {
                    newChan.Reject(ssh.UnknownChannelType, "unsupported channel type")
                    continue
                }
                ch, requests, err := newChan.Accept()
                if err != nil {
                    return
                }
                go handleSession(ch, requests)
            }
        }(tcpConn)
    }
}
```

## 패킷 구조와 암호화 상세

SSH-2의 각 패킷은 다음 구조를 가집니다:

```
┌──────────────┬──────────────┬──────────────┬──────────────┬──────────────┐
│ packet_length│ padding_len  │   payload    │   padding    │     MAC      │
│   (4 bytes)  │   (1 byte)   │  (variable)  │ (8-255 bytes)│  (variable)  │
└──────────────┴──────────────┴──────────────┴──────────────┴──────────────┘
```

`chacha20-poly1305@openssh.com` 암호 스위트를 사용하면:
- `packet_length`는 별도의 ChaCha20 키로 암호화 (길이 기반 공격 방어)
- `payload + padding`은 주 ChaCha20 키로 암호화
- `Poly1305`로 전체 암호화 패킷의 무결성 보장 (MAC 포함)

## SSH 에이전트와 키 관리

`ssh-agent`는 복호화된 개인키를 메모리에 캐시해 매번 패스프레이즈 입력 없이 인증을 수행합니다. `SSH_AUTH_SOCK` 환경 변수가 가리키는 Unix 도메인 소켓으로 통신합니다:

```bash
# 에이전트에 키 등록
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

# 에이전트 연동 포워딩 (-A 플래그)
# 원격 서버에서 로컬 에이전트를 통해 추가 인증 가능
ssh -A bastion.example.com

# 키 핑거프린트 확인 (SHA256 해시)
ssh-keygen -l -f ~/.ssh/id_ed25519.pub
# 256 SHA256:abc123...xyz (ed25519)
```

## 주의사항과 보안 팁

**키 알고리즘 선택:**
- 신규 생성은 `Ed25519` 사용 (타원 곡선 EdDSA, 빠르고 안전)
- RSA 사용 시 최소 4096비트, 가능하면 `rsa-sha2-512` 서명 알고리즘 지정

**`known_hosts` 검증:**
- `StrictHostKeyChecking yes`로 MITM 방어 강화
- `ssh-keyscan`으로 공개키를 사전 수집 후 배포

**서버 설정 강화 (`/etc/ssh/sshd_config`):**
```
# 패스워드 인증 비활성화
PasswordAuthentication no
# 루트 로그인 금지
PermitRootLogin no
# 허용 알고리즘 제한 (약한 알고리즘 제거)
KexAlgorithms curve25519-sha256,ecdh-sha2-nistp521
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
```

**키 재교환(Re-keying):** SSH는 1GB 또는 1시간 경과 시 자동으로 키를 재교환합니다. 오래된 세션의 세션 키가 노출되더라도 과거 세션을 복호화할 수 없는 **완전 순방향 비밀성(Perfect Forward Secrecy)**을 보장합니다.

## 참고 자료
- [RFC 4253: The Secure Shell (SSH) Transport Layer Protocol](https://datatracker.ietf.org/doc/html/rfc4253)
- [RFC 4252: The Secure Shell (SSH) Authentication Protocol](https://datatracker.ietf.org/doc/html/rfc4252)
- [RFC 4254: The Secure Shell (SSH) Connection Protocol](https://datatracker.ietf.org/doc/html/rfc4254)
- [OpenSSH 공식 문서](https://www.openssh.com/manual.html)
