# 🎥 WebRTC 32-CH Live Source Diagnostic Console

> 웹 브라우저 하나로 로컬 카메라 영상을 **32개 채널에 실시간 스트리밍**하는 WebRTC 데모 프로젝트입니다.  
> 서버 없이 순수 브라우저 API만으로 구현된 **P2P 스트리밍 아키텍처** 학습용 레퍼런스 코드입니다.

![demo](./1775722701201_image.png)

---

## 📌 목차

1. [프로젝트 개요](#1-프로젝트-개요)
2. [적용 기술](#2-적용-기술)
3. [전체 아키텍처 구조](#3-전체-아키텍처-구조)
4. [핵심 알고리즘 상세 분석](#4-핵심-알고리즘-상세-분석)
   - 4-1. [소스 영상 획득](#4-1-소스-영상-획득--getusermedia)
   - 4-2. [WebRTC P2P 스트리밍 연결](#4-2-webrtc-p2p-스트리밍-연결--rtcpeerconnection)
   - 4-3. [화질 동적 제어 엔진](#4-3-화질-동적-제어-엔진--rtpencodingparameters)
   - 4-4. [스포트라이트 전환](#4-4-스포트라이트-전환--채널-포커스)
   - 4-5. [버퍼 드리프트 보정](#4-5-버퍼-드리프트-보정--실시간-동기화)
5. [실행 방법](#5-실행-방법)
6. [주요 기능](#6-주요-기능)
7. [확장 개발 활용 분야](#7-확장-개발-활용-분야)
8. [기술 한계 및 개선 방향](#8-기술-한계-및-개선-방향)

---

## 1. 프로젝트 개요

이 프로그램은 **WebRTC**의 `RTCPeerConnection` API를 활용하여, 단일 카메라 소스를 **32개의 독립적인 가상 P2P 채널**로 복제·스트리밍합니다.

### 무엇이 특별한가?

일반적인 WebRTC 데모는 1:1 연결을 보여줍니다. 이 프로젝트는 다음을 실증합니다.

- **동일 소스 → N개 채널 분기**: 하나의 `MediaStream`을 32개 `RTCPeerConnection`에 동시에 공급
- **채널별 독립 화질 제어**: 그리드 모드(저화질·저프레임)와 스포트라이트 모드(고화질·고프레임)를 채널 단위로 실시간 전환
- **버퍼 드리프트 자동 보정**: 재생 속도와 타임스탬프를 주기적으로 감시하여 영상 지연 누적을 방지

### 동작 범위

| 구분 | 내용 |
|---|---|
| 채널 수 | 최대 32채널 동시 스트리밍 |
| 화질 모드 | 480p / 720p / 1080p 선택 가능 |
| 연결 방식 | 로컬 루프백 P2P (STUN/TURN 서버 불필요) |
| 실행 환경 | 모던 브라우저 (Chrome 권장) + Python 웹서버 |

---

## 2. 적용 기술

### WebRTC (Web Real-Time Communication)

이 프로젝트의 핵심 기술입니다. WebRTC는 브라우저가 **플러그인 없이** 실시간 음성·영상·데이터를 P2P로 전송할 수 있는 웹 표준입니다.

```
일반 스트리밍(RTMP/HLS):   [카메라] → 서버 → CDN → 시청자
이 프로그램(WebRTC P2P):  [카메라] → RTCPeerConnection(1~32) → 각 채널 video 태그
```

> **왜 WebRTC인가?**  
> RTMP는 서버가 필요하고, HLS는 지연이 5~30초입니다. WebRTC는 브라우저만으로 500ms 미만의 초실시간 스트리밍이 가능합니다. 서버 인프라 없이 P2P 스트리밍 원리를 학습하기에 가장 적합한 기술입니다.

### 사용된 브라우저 API

| API | 역할 |
|---|---|
| `MediaDevices.getUserMedia()` | 카메라/마이크 스트림 획득 |
| `RTCPeerConnection` | P2P 연결 생성 및 관리 |
| `RTCRtpSender.setParameters()` | 인코딩 비트레이트/프레임레이트 실시간 조정 |
| `HTMLMediaElement` | video 태그 재생 제어 |

---

## 3. 전체 아키텍처 구조

```
┌─────────────────────────────────────────────────────────────────┐
│                        브라우저 (단일 탭)                         │
│                                                                  │
│  ┌──────────────────┐                                            │
│  │  카메라 (물리 장치) │                                           │
│  └────────┬─────────┘                                           │
│           │ getUserMedia()                                        │
│           ▼                                                      │
│  ┌──────────────────┐    미러링 표시                              │
│  │   localStream    │ ──────────────→ [사이드 패널 localVideo]    │
│  └────────┬─────────┘                                           │
│           │ addTrack() × 32                                       │
│           ▼                                                      │
│  ┌────────────────────────────────────────────┐                 │
│  │           Channel Array [0 ~ 31]            │                 │
│  │                                             │                 │
│  │  ch[0]: pc1(송신) ──loopback──  pc2(수신)   │                 │
│  │  ch[1]: pc1(송신) ──loopback──  pc2(수신)   │                 │
│  │  ...                                        │                 │
│  │  ch[31]: pc1(송신) ──loopback── pc2(수신)   │                 │
│  └────────────────────────────────────────────┘                 │
│           │ ontrack → srcObject                                   │
│           ▼                                                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  8×4 그리드 모니터 (CH1 ~ CH32)                           │   │
│  │  [CH1][CH2][CH3]...[CH8]                                  │   │
│  │  [CH9]...[CH16]                                           │   │
│  │  [CH17]...[CH24]                                          │   │
│  │  [CH25]...[CH32]                                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│           │ 클릭 시                                                │
│           ▼                                                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  스포트라이트 오버레이 (전체 화면 고화질 단일 채널 표시)    │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  드리프트 보정 타이머 (500ms 주기) ──────────── 재생 속도 조절     │
└─────────────────────────────────────────────────────────────────┘
```

### 핵심 데이터 구조

```javascript
// 채널 객체 (32개 배열)
channels = [
  {
    id: 1,
    pc1: RTCPeerConnection,  // 송신 측 (localStream 트랙 보유)
    pc2: RTCPeerConnection,  // 수신 측 (화면에 표시)
    video: HTMLVideoElement, // 그리드의 <video> 태그
    statusLabel: HTMLElement,// ON/OFF 상태 표시
    stream: MediaStream      // pc2가 수신한 스트림
  },
  // ... × 32
]
```

---

## 4. 핵심 알고리즘 상세 분석

### 4-1. 소스 영상 획득 — `getUserMedia`

```javascript
localStream = await navigator.mediaDevices.getUserMedia({ 
    video: { width: 1920, height: 1080, frameRate: 30 }, 
    audio: false 
});
document.getElementById('localVideo').srcObject = localStream;
```

**동작 원리**

1. 브라우저가 OS에 카메라 접근 권한을 요청합니다.
2. 허용되면 최고 해상도(1080p 30fps)의 `MediaStream` 객체를 반환합니다.
3. 이 스트림은 `localStream` 변수에 저장되어 **32개 채널 모두의 원본 소스**로 공유됩니다.
4. 사이드 패널의 미리보기 `<video>` 태그에 즉시 표시됩니다. (`transform: scaleX(-1)`로 좌우 반전 — 자연스러운 셀카 화면)

> 💡 **중요**: 카메라 스트림은 하나만 열립니다. 32개 채널이 각각 카메라를 여는 것이 아닙니다. 하나의 `MediaStream`에서 트랙을 꺼내어 32개 RTCPeerConnection에 **공유**합니다.

---

### 4-2. WebRTC P2P 스트리밍 연결 — `RTCPeerConnection`

이 프로그램의 가장 핵심적인 부분입니다. 실제 네트워크 없이 **같은 브라우저 탭 안에서** `pc1`(송신)과 `pc2`(수신)를 직접 연결하는 **루프백(Loopback) P2P** 구조입니다.

```javascript
async function startChannel(chId) {
    const ch = channels[chId - 1];

    // 1단계: 양쪽 PeerConnection 생성
    ch.pc1 = new RTCPeerConnection(); // 송신 측
    ch.pc2 = new RTCPeerConnection(); // 수신 측

    // 2단계: ICE 후보 교환 (루프백이므로 직접 교환)
    ch.pc1.onicecandidate = e => e.candidate && ch.pc2.addIceCandidate(e.candidate);
    ch.pc2.onicecandidate = e => e.candidate && ch.pc1.addIceCandidate(e.candidate);

    // 3단계: 수신 측이 트랙을 받으면 video 태그에 연결
    ch.pc2.ontrack = e => {
        ch.stream = e.streams[0];
        ch.video.srcObject = ch.stream; // 그리드 화면에 표시
    };

    // 4단계: 원본 스트림의 트랙을 송신 측에 추가
    localStream.getTracks().forEach(track => ch.pc1.addTrack(track, localStream));

    // 5단계: SDP 협상 (Offer/Answer 교환)
    const offer = await ch.pc1.createOffer();
    await ch.pc1.setLocalDescription(offer);   // 송신 측 SDP 설정
    await ch.pc2.setRemoteDescription(offer);  // 수신 측에 Offer 전달
    const answer = await ch.pc2.createAnswer();
    await ch.pc2.setLocalDescription(answer);  // 수신 측 SDP 설정
    await ch.pc1.setRemoteDescription(answer); // 송신 측에 Answer 전달

    // 6단계: 화질 엔진 초기화 (그리드 모드)
    await setEngine(ch, false);
}
```

**연결 시퀀스 다이어그램**

```
pc1 (송신)                              pc2 (수신)
    │                                       │
    │─── createOffer() ─────────────────────│
    │─── setLocalDescription(offer) ────────│
    │──────────────── offer ───────────────►│
    │                         setRemoteDescription(offer)
    │                         createAnswer()
    │                         setLocalDescription(answer)
    │◄─────────────── answer ──────────────│
    │─── setRemoteDescription(answer) ──────│
    │                                       │
    │═══ ICE 후보 직접 교환 (루프백) ═══════│
    │                                       │
    │════════ 미디어 스트림 전송 시작 ═══════│
    │ (실제로는 브라우저 내부 메모리 버스)   │
    │                              ontrack()│
    │                    video.srcObject = stream
```

> **실제 네트워크 vs 이 프로그램**  
> 실제 원격 WebRTC에서는 STUN/TURN 서버가 ICE 후보를 중계하고, 시그널링 서버(WebSocket 등)가 SDP를 교환합니다. 이 프로그램은 같은 탭 안에서 JavaScript 변수로 직접 교환하므로 서버가 전혀 필요 없습니다.

---

### 4-3. 화질 동적 제어 엔진 — `RTPEncodingParameters`

WebRTC의 강력한 기능 중 하나인 **연결 중에 인코딩 파라미터를 실시간 변경**하는 기능을 활용합니다.

```javascript
const CONFIG = {
    SD:  { h: 480,  gridKbps: 150,  gridFps: 7,  spotKbps: 3000,  spotFps: 30, scale: 2.5 },
    HD:  { h: 720,  gridKbps: 250,  gridFps: 10, spotKbps: 6000,  spotFps: 30, scale: 3   },
    FHD: { h: 1080, gridKbps: 400,  gridFps: 10, spotKbps: 10000, spotFps: 30, scale: 4   }
};

async function setEngine(ch, isSpot) {
    const sender = ch.pc1.getSenders().find(s => s.track?.kind === 'video');
    const c = CONFIG[qualityPreset.value];
    const params = sender.getParameters();

    params.encodings = [{
        maxBitrate:            (isSpot ? c.spotKbps : c.gridKbps) * 1000,
        maxFramerate:          isSpot ? c.spotFps : c.gridFps,
        scaleResolutionDownBy: isSpot ? 1.0 : c.scale,  // 1.0 = 원본, 3.0 = 1/3 해상도
        priority:              isSpot ? 'high' : 'low'
    }];
    params.degradationPreference = isSpot ? 'maintain-resolution' : 'balanced';

    await sender.setParameters(params);
}
```

**그리드 vs 스포트라이트 파라미터 비교 (HD 기준)**

| 파라미터 | 그리드 모드 (32채널) | 스포트라이트 모드 (1채널) |
|---|---|---|
| 최대 비트레이트 | 250 Kbps | 6,000 Kbps |
| 최대 프레임레이트 | 10 fps | 30 fps |
| 해상도 축소 배율 | 1/3 (scaleBy=3) | 원본 (scaleBy=1) |
| 우선순위 | low | high |
| 품질 저하 방식 | balanced | maintain-resolution |

> 💡 **왜 이게 중요한가?**  
> 32채널이 모두 원본 품질로 동작하면 브라우저가 감당할 수 없습니다. 그리드에서는 작은 셀 크기에 맞게 **해상도와 비트레이트를 최소화**하고, 사용자가 특정 채널을 클릭했을 때만 **고품질로 즉시 전환**합니다. 이것이 실제 CCTV NVR 소프트웨어가 동작하는 원리와 동일합니다.

---

### 4-4. 스포트라이트 전환 — 채널 포커스

```javascript
async function openSpotlight(id) {
    const ch = channels[id - 1];
    isSpotlight = true;

    // 1. 선택되지 않은 모든 채널의 비디오 트랙 비활성화 (CPU/메모리 절약)
    channels.forEach(c => {
        if (c.id !== id && c.stream) {
            c.stream.getVideoTracks().forEach(t => t.enabled = false);
        }
    });

    // 2. 그리드 숨기고 전체화면 오버레이 표시
    monitorGrid.classList.add('hidden');
    spotlightOverlay.style.display = 'flex';

    // 3. 선택 채널을 고화질 모드로 전환
    const sender = ch.pc1.getSenders().find(s => s.track?.kind === 'video');
    sender.track.enabled = false;          // 순간 비활성화
    await setEngine(ch, true);             // 파라미터 변경 (고화질)
    setTimeout(() => sender.track.enabled = true, 50); // 50ms 후 재활성화

    // 4. 스포트라이트 video에 해당 채널 스트림 연결
    spotlightVideo.srcObject = ch.stream;
}
```

**트랙 비활성화 패턴의 의미**

```
[스포트라이트 진입 시]
  - 다른 31개 채널: track.enabled = false → 디코딩 중단 → CPU 절약
  - 선택 채널:      setParameters() → 고화질 인코딩으로 전환
  
[스포트라이트 해제 시]
  - 모든 채널:      setEngine(ch, false) → 그리드 저화질로 복구
  - 모든 채널:      track.enabled = true → 디코딩 재개
```

---

### 4-5. 버퍼 드리프트 보정 — 실시간 동기화

WebRTC 루프백은 시간이 지나면 수신 측 버퍼가 쌓여 재생이 점점 뒤처지는 **드리프트(Drift)** 현상이 발생합니다. 이를 자동으로 보정합니다.

```javascript
setInterval(() => {
    const syncVideo = (v) => {
        if (v.readyState >= 2 && v.buffered.length > 0) {
            const drift = v.buffered.end(0) - v.currentTime;
            // buffered.end(0): 이미 받아둔 데이터의 끝 시점
            // currentTime:     현재 재생 중인 시점
            // drift:           버퍼에 쌓인 지연량

            if (drift > 1.0) {
                // 1초 이상 밀렸으면 즉시 점프
                v.currentTime = v.buffered.end(0) - 0.1;
            } else if (drift > 0.15) {
                // 0.15~1초 사이면 재생 속도를 8% 올려서 따라잡기
                v.playbackRate = 1.08;
            } else {
                // 정상 범위면 정속 유지
                v.playbackRate = 1.0;
            }
        }
    };

    if (isSpotlight) syncVideo(spotlightVideo);
    else channels.forEach(ch => { if(ch.pc1) syncVideo(ch.video); });

}, 500); // 500ms마다 체크
```

**드리프트 보정 전략**

```
drift > 1.0초  → 강제 점프 (currentTime 재설정)  ━━▶ 즉각 복구
drift > 0.15초 → 재생 속도 1.08배 가속           ━━▶ 서서히 따라잡기
drift ≤ 0.15초 → 정상 속도 1.0배 유지            ━━▶ 안정 상태
```

---

## 5. 실행 방법

> 서버 설치 없이 **Python 하나**만 있으면 즉시 실행할 수 있습니다.

### 사전 요구사항

- Python 3.x 설치 ([python.org](https://www.python.org/downloads/))
- Chrome 또는 Edge 브라우저 (최신 버전 권장)
- 웹캠(카메라) 연결 상태

### Step 1 — 파일 다운로드

```bash
# 방법 A: git clone
git clone https://github.com/your-username/webrtc-32ch-console.git
cd webrtc-32ch-console

# 방법 B: ZIP 다운로드 후 압축 해제
# GitHub 저장소 우측 상단 [Code] → [Download ZIP] 클릭 후 압축 해제
```

### Step 2 — 웹 서버 실행

`webrtc.html` 파일이 있는 폴더에서 명령 프롬프트(CMD)를 열고 아래 명령어를 실행합니다.

**Windows (CMD / PowerShell)**
```bash
cd C:\Users\사용자명\다운로드\webrtc-32ch-console
python -m http.server 9000
```

**Mac / Linux (Terminal)**
```bash
cd ~/Downloads/webrtc-32ch-console
python3 -m http.server 9000
```

서버가 정상 실행되면 다음과 같이 표시됩니다.
```
Serving HTTP on 0.0.0.0 port 9000 (http://0.0.0.0:9000/) ...
```

### Step 3 — 브라우저에서 접속

Chrome 또는 Edge 주소창에 입력합니다.

```
http://localhost:9000/webrtc.html
```

> ⚠️ **주의**: `file://` 로 직접 열면 카메라 권한이 차단됩니다. 반드시 `http://localhost`로 접속하세요.

### Step 4 — 조작 순서

```
① 우측 상단 화질 선택: 480p / 720p / 1080p 중 선택 (기본값: 720p)
       ↓
② 좌측 [SYSTEM ON] 버튼 클릭 → 브라우저 카메라 권한 허용
       ↓
③ 우측 상단 [전체 채널 연결] 클릭 → 32채널 스트리밍 시작 (약 2~3초 소요)
       ↓
④ 그리드에서 원하는 채널 클릭 → 스포트라이트 고화질 전체화면 전환
       ↓
⑤ ESC 키 또는 [닫기] 버튼 → 그리드 복귀
```

### 채널 개별 조작

- 각 채널 우측 상단의 `ON` / `OFF` 텍스트를 클릭하면 해당 채널만 연결/해제할 수 있습니다.
- `SOURCE RAW VIEW` 버튼 클릭 시 카메라 원본 영상이 별도 팝업 창으로 표시됩니다.

---

## 6. 주요 기능

| 기능 | 설명 |
|---|---|
| **32채널 동시 스트리밍** | 단일 카메라 소스를 8×4 그리드로 32분할 표시 |
| **화질 프리셋** | 480p / 720p / 1080p 전체 채널 일괄 전환 |
| **스포트라이트 모드** | 채널 클릭 시 고화질(최대 10,000 Kbps, 30fps) 전체화면 전환 |
| **개별 채널 ON/OFF** | 각 채널 독립적 연결·해제 |
| **전체 채널 일괄 연결/해제** | 버튼 하나로 전체 채널 제어 |
| **SOURCE RAW VIEW** | 원본 카메라 영상 팝업 표시 |
| **드리프트 자동 보정** | 500ms 주기로 버퍼 지연 감지 및 재생 속도 자동 조정 |
| **실시간 로그** | 좌측 패널에 시스템 이벤트 타임스탬프 출력 |

---

## 7. 확장 개발 활용 분야

이 프로젝트의 구조는 여러 실무 분야로 확장할 수 있는 강력한 기반입니다.

### 🏢 CCTV / 영상 관제 시스템

```
현재: 로컬 카메라 1개 → 32채널 (루프백 P2P)
확장: 각 채널에 실제 IP 카메라(RTSP) 연결 추가
      → RTSP를 WebRTC로 변환하는 미디어 서버(MediaMTX 등) 연동
      → 실제 32채널 IP 카메라 관제 NVR 소프트웨어
```

현재 구현된 **그리드 표시, 스포트라이트 전환, 채널별 ON/OFF** 로직은 상용 NVR 소프트웨어의 핵심 UI 패턴과 동일합니다.

### 🎓 온라인 교육 / 멀티뷰 강의 시스템

```
확장: 학생별로 WebRTC 연결을 1개 채널로 매핑
      → 강사가 32명 학생의 화면을 동시에 모니터링
      → 특정 학생 클릭 시 스포트라이트로 확대 발표 모드 전환
```

화상 수업에서 교사가 학생 전체를 한눈에 보다가 특정 학생을 확대하는 기능을 WebRTC만으로 구현 가능합니다.

### 🏭 산업 현장 / 품질 모니터링

```
확장: 생산 라인 각 공정의 카메라를 채널별로 연결
      → 불량 감지 AI 모델과 연동하여 이상 채널 자동 스포트라이트 전환
      → 실시간 품질 관제 대시보드
```

### 🎮 라이브 스트리밍 / 멀티소스 방송

```
확장: 여러 출연자의 WebRTC 스트림을 수신
      → 스포트라이트 채널 = 현재 방송 중인 화면
      → 감독이 전환 버튼을 누르면 방송 소스 전환
      → 간이 라이브 스위처(비디오 믹서) 구현
```

RTMP로 서버에 송출하기 전 소스 선택 단계를 브라우저로 구현할 수 있습니다.

### 🚗 자율주행 / 드론 원격 모니터링

```
확장: 차량/드론의 여러 방향 카메라를 각각 채널로 연결
      → 전방, 후방, 좌측, 우측, 하부 카메라를 동시 모니터링
      → 이상 상황 감지 시 해당 방향 스포트라이트 자동 전환
```

WebRTC의 극초저지연(< 500ms) 특성이 원격 제어 모니터링에 필수적입니다.

### 💡 기술 확장 로드맵

```
현재 (V12.7)
    │
    ├─ 실제 RTSP 카메라 연동 (MediaMTX 미디어 서버 추가)
    │
    ├─ WebSocket 시그널링 서버 추가 → 실제 원격 P2P 연결
    │
    ├─ SFU 서버 연동 (mediasoup) → 100채널 이상 확장
    │
    └─ AI 이상 감지 + 자동 스포트라이트 전환
```

---

## 8. 기술 한계 및 개선 방향

| 한계 | 원인 | 개선 방향 |
|---|---|---|
| 로컬 루프백만 지원 | STUN/TURN 서버 미구현 | WebSocket 시그널링 + TURN 서버 추가 |
| 단일 소스 (카메라 1대) | 설계 단순화 | 채널별 독립 소스 URL 입력 기능 추가 |
| 32채널 초과 시 성능 저하 | 브라우저 RTCPeerConnection 한계 | SFU 서버 도입으로 N:N 구조 전환 |
| 오디오 미지원 | 설계 단순화 | getUserMedia에 audio: true 추가 |

---

## 📄 라이선스

MIT License — 자유롭게 사용, 수정, 배포 가능합니다.

---

> 💬 **이 프로젝트에 대한 질문이나 개선 제안은 Issues 탭을 이용해 주세요.**
