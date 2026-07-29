---
layout: post
title: "WebRTC 내부 구조 — ICE, STUN, TURN, SDP 프로토콜 해부"
date: 2026-07-29
categories: [cs, computer-science]
tags: [webrtc, ice, stun, turn, sdp, p2p, nat-traversal, real-time-communication, networking]
---

## 개념 설명

WebRTC(Web Real-Time Communication)는 브라우저와 모바일 앱이 플러그인 없이 실시간 음성·영상·데이터를 주고받을 수 있게 하는 오픈 표준이다. Google Meet, Discord, Zoom의 브라우저 버전 등 수많은 실시간 통신 서비스의 기반이 되지만, 내부 프로토콜 스택은 여러 RFC에 걸쳐 복잡하게 정의되어 있다.

WebRTC가 어려운 이유는 **NAT(Network Address Translation)** 때문이다. 현실의 대부분 기기는 사설 IP 주소를 가지며, 인터넷 라우터가 공인 IP로 변환한다. 두 피어가 모두 NAT 뒤에 있으면 직접 연결이 불가능한 것처럼 보이지만, ICE 프레임워크가 이 문제를 해결한다.

### WebRTC 스택 개요

```
Application (JS/Native)
       │
WebRTC API (RTCPeerConnection)
       │
┌──────┴───────────────────────────────┐
│  Signaling (별도 구현 필요)            │
│  SDP Offer/Answer 교환               │
└──────────────────────────────────────┘
       │
┌──────┴───────────────────────────────┐
│  ICE Framework                        │
│  STUN / TURN 서버 활용               │
│  네트워크 후보 수집 및 연결 검사       │
└──────────────────────────────────────┘
       │
┌──────┴──────────┬─────────────────────┐
│  DTLS-SRTP      │  SCTP over DTLS     │
│  (음성/영상)     │  (데이터 채널)       │
└─────────────────┴─────────────────────┘
```

## 왜 이런 복잡한 구조가 필요한가?

### NAT 타입과 통과 가능성

NAT에는 여러 종류가 있으며, 타입에 따라 통과 전략이 달라진다:

| NAT 타입 | 특징 | STUN으로 통과 |
|---------|------|-------------|
| Full Cone NAT | 내부에서 외부로 패킷 보내면 어느 외부 호스트든 접근 가능 | O |
| Address Restricted Cone | 이전에 통신한 IP에서만 인바운드 허용 | O (조건부) |
| Port Restricted Cone | 이전에 통신한 IP:Port에서만 허용 | O (조건부) |
| Symmetric NAT | 목적지마다 다른 외부 포트 할당 | X (TURN 필수) |

기업 네트워크와 모바일 데이터 환경에서 Symmetric NAT가 많다. 따라서 TURN 서버 없이는 약 10~15%의 연결이 실패한다.

## 핵심 프로토콜 상세

### SDP (Session Description Protocol)

SDP는 미디어 세션의 매개변수를 텍스트로 기술하는 프로토콜(RFC 4566)이다. WebRTC는 Offer/Answer 모델로 SDP를 교환한다.

```
v=0                                    # 프로토콜 버전
o=- 7614219274584779017 2 IN IP4 127.0.0.1  # 발신자 정보
s=-                                    # 세션 이름 (사용 안 함)
t=0 0                                  # 세션 유효 시간
a=group:BUNDLE 0 1                     # 미디어 번들링
a=msid-semantic: WMS                   # 미디어 스트림 식별

m=audio 9 UDP/TLS/RTP/SAVPF 111       # 오디오 미디어, 포트 9(ICE가 실제 포트 결정)
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:F7gI                       # ICE 인증 사용자 이름
a=ice-pwd:x9cml/YouftOsvIXnH7bcNV//   # ICE 인증 비밀번호
a=fingerprint:sha-256 D2:FA:0E:...    # DTLS 인증서 지문
a=setup:actpass                        # DTLS 역할 (active/passive 모두 가능)
a=mid:0                                # 미디어 식별자
a=sendrecv                             # 양방향
a=rtpmap:111 opus/48000/2             # Opus 코덱, 48kHz, 스테레오
a=fmtp:111 minptime=10;useinbandfec=1 # Opus 파라미터

m=video 9 UDP/TLS/RTP/SAVPF 96 97
a=rtpmap:96 VP8/90000
a=rtpmap:97 H264/90000
a=fmtp:97 profile-level-id=42e01f    # H.264 프로파일
```

### STUN (Session Traversal Utilities for NAT, RFC 5389)

STUN은 클라이언트가 자신의 공인 IP:Port를 발견하도록 돕는 경량 프로토콜이다.

```
클라이언트           STUN 서버
(사설: 192.168.1.5:54321)    (공인: 8.8.8.8:3478)
     │                              │
     │── STUN Binding Request ─────>│
     │   (UDP, 20바이트 헤더)        │
     │                              │
     │<─ STUN Binding Response ─────│
     │   XOR-MAPPED-ADDRESS:        │
     │   203.0.113.1:54321          │
     │   (공인 IP:Port 획득)         │
```

STUN 메시지는 20바이트 헤더로 구성된다:
- **Message Type** (2바이트): Binding Request (0x0001), Binding Response (0x0101)
- **Message Length** (2바이트)
- **Magic Cookie** (4바이트): 고정값 0x2112A442
- **Transaction ID** (12바이트): 요청-응답 매칭용

### TURN (Traversal Using Relays around NAT, RFC 5766)

Symmetric NAT에서 STUN만으로 통과 불가능한 경우 TURN이 모든 미디어를 중계한다.

```
클라이언트 A          TURN 서버           클라이언트 B
     │         (공인: 203.0.113.10)           │
     │                  │                     │
     │── ALLOCATE ─────>│                     │
     │<─ ALLOCATE Resp ─│                     │
     │   relay: 203.0.113.10:60001            │
     │                  │                     │
     │── SEND ─────────>│── Data ────────────>│
     │<─ Data ──────────│<── SEND ────────────│
```

TURN은 STUN의 상위 집합으로, Allocate/Refresh/CreatePermission/Send/Data 등의 메서드를 추가한다.

### ICE (Interactive Connectivity Establishment, RFC 8445)

ICE는 STUN과 TURN을 활용해 두 피어 간 최적의 네트워크 경로를 찾는 프레임워크다.

**ICE 후보 수집 단계**:
```
1. Host Candidate:    192.168.1.5:54321    (로컬 인터페이스)
2. srflx Candidate:   203.0.113.1:54321   (STUN으로 발견한 공인 주소)
3. relay Candidate:   203.0.113.10:60001  (TURN 서버 중계 주소)
```

**ICE 연결 검사**:
양쪽에서 수집한 후보들을 조합해 우선순위 순서로 연결 가능 여부를 테스트한다. 성공한 후보 쌍 중 가장 우선순위 높은 것을 선택한다.

## 실제 구현 예제

### 예제 1: WebRTC 발신자 측 (Offerer)

```javascript
// WebRTC 피어 연결 수립 — 발신자(Offerer) 측
const configuration = {
    iceServers: [
        {
            urls: [
                'stun:stun1.l.google.com:19302',
                'stun:stun2.l.google.com:19302'
            ]
        },
        {
            urls: 'turn:turn.myserver.com:3478',
            username: 'webrtc-user',
            credential: 'secure-password'
        },
        {
            urls: 'turns:turn.myserver.com:5349',  // TLS over TURN
            username: 'webrtc-user',
            credential: 'secure-password'
        }
    ],
    iceTransportPolicy: 'all',    // 'relay'로 변경하면 TURN만 사용 (디버깅용)
    bundlePolicy: 'max-bundle',   // 모든 미디어를 하나의 ICE 쌍으로 묶음
    rtcpMuxPolicy: 'require'      // RTP/RTCP를 같은 포트에서 다중화
};

const pc = new RTCPeerConnection(configuration);

// ICE 후보 수집 이벤트
pc.onicecandidate = ({ candidate }) => {
    if (candidate) {
        // 시그널링 채널을 통해 상대방에게 전송
        signalingChannel.send(JSON.stringify({
            type: 'ice-candidate',
            candidate: candidate.toJSON()
        }));

        // 후보 타입 확인
        const parts = candidate.candidate.split(' ');
        const type = parts[7]; // host, srflx, relay
        console.log(`ICE candidate [${type}]: ${candidate.candidate}`);
    } else {
        // null은 후보 수집 완료를 의미
        console.log('ICE candidate gathering complete');
    }
};

// ICE 연결 상태 모니터링
pc.oniceconnectionstatechange = () => {
    const state = pc.iceConnectionState;
    console.log('ICE connection state:', state);

    if (state === 'failed') {
        // ICE 재시작 시도
        pc.restartIce();
    }

    if (state === 'disconnected') {
        // 일시적 네트워크 중단 — 잠시 기다렸다가 재시작 고려
        setTimeout(() => {
            if (pc.iceConnectionState === 'disconnected') {
                pc.restartIce();
            }
        }, 5000);
    }
};

// 연결 상태 모니터링 (iceConnectionState보다 포괄적)
pc.onconnectionstatechange = () => {
    console.log('Connection state:', pc.connectionState);
    // new → connecting → connected → disconnected/failed/closed
};

// 미디어 스트림 추가
async function startCall() {
    const stream = await navigator.mediaDevices.getUserMedia({
        video: { width: 1280, height: 720, frameRate: 30 },
        audio: {
            echoCancellation: true,
            noiseSuppression: true,
            autoGainControl: true
        }
    });

    document.getElementById('localVideo').srcObject = stream;

    // 트랙을 RTCPeerConnection에 추가
    stream.getTracks().forEach(track => {
        pc.addTrack(track, stream);
    });

    // Offer SDP 생성
    const offer = await pc.createOffer({
        offerToReceiveAudio: true,
        offerToReceiveVideo: true,
        voiceActivityDetection: true
    });

    // 로컬 기술 설정 (이 순간 ICE 후보 수집 시작)
    await pc.setLocalDescription(offer);

    // 시그널링 서버에 Offer 전송
    signalingChannel.send(JSON.stringify({
        type: 'offer',
        sdp: pc.localDescription.sdp
    }));
}

startCall();
```

### 예제 2: 수신자 측 및 완전한 시그널링 처리

```javascript
// 수신자(Answerer) 측 + 시그널링 메시지 핸들러
const pc = new RTCPeerConnection(configuration);

// 원격 트랙 수신
pc.ontrack = (event) => {
    console.log('Remote track received:', event.track.kind);
    if (event.streams && event.streams[0]) {
        document.getElementById('remoteVideo').srcObject = event.streams[0];
    }
};

// ICE 후보 수집 및 전송
pc.onicecandidate = ({ candidate }) => {
    if (candidate) {
        signalingChannel.send(JSON.stringify({
            type: 'ice-candidate',
            candidate: candidate.toJSON()
        }));
    }
};

// 시그널링 메시지 처리
signalingChannel.onmessage = async (msg) => {
    const data = JSON.parse(msg.data);

    switch (data.type) {
        case 'offer':
            // 원격 Offer 설정
            await pc.setRemoteDescription(new RTCSessionDescription({
                type: 'offer',
                sdp: data.sdp
            }));

            // 로컬 미디어 추가
            const stream = await navigator.mediaDevices.getUserMedia({
                video: true,
                audio: true
            });
            stream.getTracks().forEach(track => pc.addTrack(track, stream));

            // Answer 생성 및 설정
            const answer = await pc.createAnswer();
            await pc.setLocalDescription(answer);

            // Answer 전송
            signalingChannel.send(JSON.stringify({
                type: 'answer',
                sdp: answer.sdp
            }));
            break;

        case 'answer':
            // 원격 Answer 설정
            await pc.setRemoteDescription(new RTCSessionDescription({
                type: 'answer',
                sdp: data.sdp
            }));
            break;

        case 'ice-candidate':
            // Trickle ICE: Offer/Answer 교환 중에도 후보를 점진적으로 추가
            if (data.candidate) {
                try {
                    await pc.addIceCandidate(new RTCIceCandidate(data.candidate));
                } catch (e) {
                    // setRemoteDescription 완료 전 후보가 도착하면 큐에 저장 필요
                    console.error('Failed to add ICE candidate:', e);
                }
            }
            break;
    }
};

// 연결 통계 모니터링 (디버깅에 유용)
async function getConnectionStats() {
    const stats = await pc.getStats();
    stats.forEach(report => {
        if (report.type === 'candidate-pair' && report.state === 'succeeded') {
            console.log('Selected candidate pair:',
                `local: ${report.localCandidateId}`,
                `remote: ${report.remoteCandidateId}`,
                `RTT: ${report.currentRoundTripTime * 1000}ms`
            );
        }
        if (report.type === 'inbound-rtp' && report.kind === 'video') {
            console.log('Video stats:',
                `frameWidth: ${report.frameWidth}`,
                `frameHeight: ${report.frameHeight}`,
                `framesPerSecond: ${report.framesPerSecond}`,
                `packetsLost: ${report.packetsLost}`
            );
        }
    });
}

setInterval(getConnectionStats, 5000);
```

## 전체 연결 수립 시퀀스

```
피어 A              시그널링 서버           피어 B
  │                      │                    │
  │── getUserMedia() ────│                    │
  │── createOffer() ─────│                    │
  │── setLocalDesc() ────│                    │
  │                      │                    │
  │─── Offer SDP ───────>│──── Offer SDP ────>│
  │                      │                    │── setRemoteDesc()
  │                      │                    │── getUserMedia()
  │                      │                    │── createAnswer()
  │                      │                    │── setLocalDesc()
  │                      │                    │
  │<── Answer SDP ────────│<─── Answer SDP ───│
  │── setRemoteDesc() ───│                    │
  │                      │                    │
  │      [ICE 후보 수집 및 교환 병렬 진행]     │
  │── ICE candidate ────>│── ICE candidate ──>│
  │<─ ICE candidate ──────│<─ ICE candidate ──│
  │                      │                    │
  │      [ICE 연결 검사 (STUN Binding)]        │
  │─────────── STUN Binding Request ──────────>│
  │<────────── STUN Binding Response ──────────│
  │                      │                    │
  │      [DTLS 핸드셰이크 — 미디어 암호화 키 교환]│
  │                      │                    │
  │      [SRTP/SCTP 미디어 전송 시작]          │
  │<══════════════ 실시간 미디어/데이터 ════════>│
```

## 주의사항 및 팁

### TURN 서버는 선택이 아닌 필수
Symmetric NAT 비율은 환경에 따라 다르지만 프로덕션에서 약 10~20%에 달한다. TURN 없이 배포하면 이 비율의 사용자가 연결에 실패한다. 오픈소스 **Coturn** 서버를 직접 운영하거나, Twilio, Agora, Cloudflare Calls 같은 관리형 서비스를 사용한다.

### Trickle ICE로 연결 시간 단축
Offer/Answer 교환 완료 후 ICE 후보를 전송하는 vanilla ICE 대신, Trickle ICE를 사용하면 후보를 SDP 교환과 병렬로 전송해 연결 수립 시간을 500ms~2초 단축할 수 있다. 단, `setRemoteDescription()` 완료 전에 도착한 후보는 큐에 저장했다가 추가해야 한다.

### mDNS와 프라이버시
현대 브라우저(Chrome 75+, Firefox 68+)는 Host Candidate의 로컬 IP 노출을 방지하기 위해 `xxxxx.local` 형식의 mDNS 주소를 사용한다. 같은 로컬 네트워크에서는 정상 동작하지만, 크로스 네트워크 연결에서는 STUN/TURN 후보에 의존하게 된다.

### 연결 실패 시 ICE 재시작
```javascript
// ICE 재시작: 새로운 ICE 자격증명과 함께 새 Offer 생성
pc.oniceconnectionstatechange = async () => {
    if (pc.iceConnectionState === 'failed') {
        const offer = await pc.createOffer({ iceRestart: true });
        await pc.setLocalDescription(offer);
        signalingChannel.send(JSON.stringify({
            type: 'offer',
            sdp: offer.sdp
        }));
    }
};
```

### 네트워크 변경 처리
모바일 환경에서 Wi-Fi에서 LTE로 전환되면 ICE 후보가 무효화된다. `RTCPeerConnection` v2.0부터는 ICE 재시작으로 이를 처리하며, WebRTC Perfect Negotiation 패턴을 사용하면 발신자/수신자 역할 충돌 없이 재협상을 안전하게 처리할 수 있다.

WebRTC는 복잡하지만 그 복잡성 뒤에는 NAT 통과라는 근본 문제를 해결하기 위한 엔지니어링의 집약이 담겨 있다. ICE/STUN/TURN/SDP 각 레이어가 왜 존재하는지 이해하면 디버깅과 최적화가 훨씬 수월해진다.

## 참고 자료
- [Introduction to WebRTC Protocols — MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Protocols)
- [WebRTC Best Practices: SDP/ICE, STUN/TURN — Wowza](https://www.wowza.com/blog/webrtc-best-practices-what-you-need-to-know-about-sdp-ice-whip-whep-and-stun-turn)
- [RFC 8445: Interactive Connectivity Establishment (ICE)](https://datatracker.ietf.org/doc/html/rfc8445)
- [RFC 5389: Session Traversal Utilities for NAT (STUN)](https://datatracker.ietf.org/doc/html/rfc5389)
