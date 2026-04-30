# 📹 WebRTC — Real-time P2P Audio/Video/Data

> **WebRTC (Web Real-Time Communication) — ব্রাউজার ও মোবাইলে plugin ছাড়াই peer-to-peer audio, video এবং arbitrary data পাঠানোর open standard।**
> Pathao Care doctor consult, Foodpanda customer-rider video call, bKash agent video KYC, 10 Minute School live class — সবখানে WebRTC বা তার উপর ভিত্তি করে stack।

---

## 📖 সূচিপত্র

- [WebRTC কেন এবং কী?](#-webrtc-কেন-এবং-কী)
- [Architecture Overview](#-architecture-overview)
- [ICE / STUN / TURN](#-ice--stun--turn)
- [Signaling — কী এবং কেন](#-signaling)
- [Components — Peer Connection, SDP, Tracks](#-components)
- [Codecs ও Bandwidth](#-codecs-ও-bandwidth)
- [Beyond P2P — Mesh, SFU, MCU](#-beyond-p2p--mesh-sfu-mcu)
- [Security](#-security)
- [Implementations](#-implementations)
- [Code: Browser Video Call](#-code-browser-video-call)
- [Code: coturn Setup](#-code-coturn-setup)
- [Code: LiveKit Quickstart](#-code-livekit-quickstart)
- [Operational Concerns](#-operational-concerns)
- [WebRTC vs Alternatives](#-webrtc-vs-alternatives)
- [Bangladesh Scenarios](#-bangladesh-scenarios)
- [Pitfalls ও Checklist](#-pitfalls-ও-checklist)

---

## 🎯 WebRTC কেন এবং কী?

**WebRTC** হলো একটি browser API + protocol stack যা ব্রাউজার-থেকে-ব্রাউজার (বা ব্রাউজার-থেকে-server, native app) **low-latency**, **encrypted** real-time media ও data transfer সম্ভব করে।

### Use Cases

- **ভিডিও কলিং**: Pathao Care doctor consult, telemedicine
- **লাইভ স্ট্রিমিং (small audience)**: ১-অনেক broadcast (large audience-এ HLS ভালো)
- **স্ক্রিন শেয়ার**: customer support, remote teaching
- **P2P file transfer**: WeTransfer-এর মতো, কিন্তু server-bypass (data channel)
- **IoT/গেমিং**: low-latency control signal
- **ভিডিও KYC**: bKash agent onboarding, ব্যাংক
- **AR/VR collaboration**: spatial audio + video

### মূল বৈশিষ্ট্য

- **Plugin-free**: Chrome, Firefox, Safari, Edge সবাই native support
- **End-to-end encrypted**: DTLS-SRTP বাধ্যতামূলক
- **Adaptive**: bandwidth/CPU অনুযায়ী quality adjust
- **NAT traversal**: ICE/STUN/TURN দিয়ে firewall এর behind-এও কাজ করে
- **Standardized**: W3C + IETF

---

## 🏗️ Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                      WebRTC Full Stack (Bangla)                       │
└──────────────────────────────────────────────────────────────────────┘

   ┌─────────────────┐                         ┌─────────────────┐
   │   Browser A     │                         │   Browser B     │
   │  (Pathao Care   │                         │   (Doctor)      │
   │    Patient)     │                         │                 │
   ├─────────────────┤                         ├─────────────────┤
   │ getUserMedia()  │                         │ getUserMedia()  │
   │   ↓             │                         │   ↓             │
   │ MediaStream     │                         │ MediaStream     │
   │   ↓             │                         │   ↓             │
   │ RTCPeerConn     │                         │ RTCPeerConn     │
   └────────┬────────┘                         └────────┬────────┘
            │                                            │
            │  ──── (1) Signaling Channel ──────         │
            │       (WebSocket / SSE / HTTP)             │
            ├────────────────►  ┌─────────────┐ ◄────────┤
            │                   │  Signaling  │          │
            │                   │   Server    │          │
            │                   └─────────────┘          │
            │       SDP offer / answer + ICE candidates  │
            │                                            │
            │  ──── (2) ICE / STUN / TURN ──────         │
            │     (NAT traversal, candidate pair)        │
            ├────────────────►  ┌─────────────┐          │
            │                   │ STUN Server │ (Google) │
            │                   └─────────────┘          │
            │                   ┌─────────────┐          │
            │                   │ TURN Server │ (coturn) │
            │                   │  (relay)    │          │
            │                   └─────────────┘          │
            │                                            │
            │ ══(3) Media Plane══════════════════════════│
            │     SRTP / SRTCP audio + video             │
            │     SCTP-over-DTLS data channel            │
            │ ◄══════════════════════════════════════════│
            └────────────────────────────────────────────┘

   Direct P2P (or via TURN relay if NAT impossible to traverse)
```

### তিনটি প্লেন

| Plane | Protocol | Purpose |
|-------|----------|---------|
| **Signaling** | আপনার পছন্দ (WebSocket recommended) | offer/answer, ICE exchange, room mgmt |
| **Connectivity (ICE)** | STUN, TURN | NAT traversal, candidate gathering |
| **Media + Data** | SRTP (media), SCTP/DTLS (data) | encrypted real-time bytes |

---

## 🧊 ICE / STUN / TURN

**ICE** = Interactive Connectivity Establishment। এটি একটি framework যা peer দুটির মধ্যে সম্ভাব্য সব path test করে best path বেছে নেয়।

### Candidate Types

```
┌────────────────────────────────────────────────────────┐
│ ICE Candidate Types                                     │
├────────────────────────────────────────────────────────┤
│                                                         │
│ host       → device-এর local IP (192.168.0.5:5000)    │
│              same network এ থাকলে কাজ করে              │
│                                                         │
│ srflx      → server reflexive (STUN দিয়ে আবিষ্কৃত    │
│              public IP — 103.x.x.x:54321)              │
│              সাধারণ NAT-এ কাজ করে                      │
│                                                         │
│ prflx      → peer reflexive (peer থেকে দেখা public IP) │
│                                                         │
│ relay      → TURN server-এর IP                          │
│              symmetric NAT/firewall এ শেষ ভরসা          │
│                                                         │
└────────────────────────────────────────────────────────┘
```

### NAT Traversal — Hole Punching

```
Patient (192.168.1.5) ── Router ── Internet ── Router ── Doctor (10.0.0.4)
                          NAT-A                NAT-B
                          103.5.5.5            117.7.7.7

Step 1: Both query STUN server → know own public IP
        Patient: 103.5.5.5:43210
        Doctor:  117.7.7.7:51000

Step 2: Exchange via signaling
Step 3: Both send packets simultaneously
        → NAT mapping তৈরি হয়, packet pass করে
        → "Hole punched"

Symmetric NAT (mobile carrier-grade NAT)?
   → প্রতিটি destination-এ আলাদা port mapping
   → Hole punching ব্যর্থ
   → TURN relay লাগবে
```

### STUN

**Session Traversal Utilities for NAT** — খুবই lightweight, শুধু "তোমার public IP/port কী?" বলে দেয়।

```
stun:stun.l.google.com:19302         (free, কিন্তু production-এ unreliable)
stun:stun.cloudflare.com:3478        (Cloudflare free)
```

### TURN

**Traversal Using Relays around NAT** — যখন direct path অসম্ভব, TURN server media relay করে। **Bandwidth heavy → costly**।

```
A ──► TURN ──► B          (full duplex)
      relays
```

**TURN over TCP/TLS (turns://)** — corporate firewall শুধু 443 খোলা থাকলে।

```
turn:turn.example.com:3478?transport=udp     # default
turn:turn.example.com:3478?transport=tcp     # firewall friendly
turns:turn.example.com:5349?transport=tcp    # TLS-wrapped (port 443 ও use করা যায়)
```

### TURN Authentication

- **Long-term credentials**: hardcoded username/password — risky।
- **Ephemeral REST credentials** (recommended): server time-limited HMAC token issue করে।

```javascript
// Node — ephemeral credential generator
import crypto from 'crypto';

function turnCredentials(secret, ttlSec = 3600, name = 'pathao-user') {
  const expiry = Math.floor(Date.now() / 1000) + ttlSec;
  const username = `${expiry}:${name}`;
  const password = crypto.createHmac('sha1', secret)
    .update(username).digest('base64');
  return { username, password, ttl: ttlSec };
}
```

---

## 📡 Signaling

**WebRTC signaling protocol specify করে না** — আপনি যেকোনো transport ব্যবহার করতে পারেন:
- WebSocket (most common, recommended) — `14-networking/websockets.md` দেখুন
- Server-Sent Events (one-way, with HTTP POST upload) — `06-api-design/server-sent-events.md`
- HTTP long polling
- MQTT, XMPP

### Signaling Messages

| Message | Direction | Purpose |
|---------|-----------|---------|
| `offer` | A → B | "I want to call you, here is my SDP" |
| `answer` | B → A | "OK, here is my SDP" |
| `ice-candidate` | both | trickle ICE candidates |
| `bye` | either | end call |
| custom | as needed | room join/leave, presence |

### Trickle ICE

দুটি option:
1. **Vanilla ICE**: সব candidate gather শেষ → তারপর offer/answer (slow)।
2. **Trickle ICE**: candidate পেতে পেতে stream করো (fast, modern default)।

---

## 🧩 Components

### RTCPeerConnection Lifecycle

```
┌─────────────────────────────────────────────────────────────┐
│ Caller (Patient)                Callee (Doctor)              │
├─────────────────────────────────────────────────────────────┤
│ new RTCPeerConnection({iceServers})                          │
│   ↓                                                          │
│ getUserMedia → addTrack(track, stream)                       │
│   ↓                                                          │
│ pc.createOffer()  ─────► SDP offer                           │
│ pc.setLocalDescription(offer)                                │
│                                                              │
│  ── send offer via signaling ──►                             │
│                                                              │
│                              setRemoteDescription(offer)     │
│                              addTrack (callee side)          │
│                              createAnswer() → SDP answer     │
│                              setLocalDescription(answer)     │
│                                                              │
│  ◄── send answer via signaling ──                            │
│                                                              │
│ setRemoteDescription(answer)                                 │
│                                                              │
│  ── ICE candidates exchanged (trickle) ──                    │
│                                                              │
│ DTLS handshake → keys exchanged                              │
│ ICE complete → connected                                     │
│ ontrack fires → remote video plays                           │
└─────────────────────────────────────────────────────────────┘
```

### SDP — Session Description Protocol

```
v=0
o=- 1234567890 2 IN IP4 127.0.0.1
s=-
t=0 0
a=group:BUNDLE 0 1
m=audio 9 UDP/TLS/RTP/SAVPF 111         ← audio m-line, codec 111 (Opus)
c=IN IP4 0.0.0.0
a=rtcp-mux
a=ice-ufrag:abc123
a=ice-pwd:xyz...
a=fingerprint:sha-256 AB:CD:...           ← DTLS cert fingerprint
a=setup:actpass
a=mid:0
a=sendrecv
a=rtpmap:111 opus/48000/2
m=video 9 UDP/TLS/RTP/SAVPF 96 97         ← video m-line, codec 96/97
...
a=rtpmap:96 VP8/90000
a=rtpmap:97 H264/90000
```

বিস্তৃত knob — bandwidth (`b=AS:...`), simulcast (`a=simulcast:`), header extensions ইত্যাদি।

### Media Tracks

```javascript
// Camera + mic
const stream = await navigator.mediaDevices.getUserMedia({
  audio: { echoCancellation: true, noiseSuppression: true },
  video: { width: 1280, height: 720, frameRate: 30, facingMode: 'user' }
});

// Screen share
const screenStream = await navigator.mediaDevices.getDisplayMedia({
  video: { displaySurface: 'monitor' },
  audio: true                   // Chrome only
});

// Track manipulation
const videoTrack = stream.getVideoTracks()[0];
await videoTrack.applyConstraints({ width: 640, height: 480 });
videoTrack.enabled = false;     // mute (still alive)
videoTrack.stop();              // permanently
```

### RTCDataChannel

SCTP-over-DTLS — TCP-like reliability with UDP-like speed, configurable।

```javascript
// Reliable, ordered (default — like TCP)
const dc1 = pc.createDataChannel('chat');

// Unreliable, unordered (game state, low latency)
const dc2 = pc.createDataChannel('game', {
  ordered: false,
  maxRetransmits: 0
});

// Partially reliable
const dc3 = pc.createDataChannel('telemetry', {
  ordered: true,
  maxPacketLifeTime: 1000     // ms; পুরোনো হলে drop
});

dc1.onopen = () => dc1.send('hello');
dc1.onmessage = e => console.log('got', e.data);
```

### Simulcast & SVC

**Simulcast**: একই source-এর multiple resolution একসাথে পাঠানো। SFU subscriber-এর bandwidth অনুযায়ী একটা বেছে দেয়।

```javascript
const sender = pc.addTransceiver(videoTrack, {
  sendEncodings: [
    { rid: 'low', maxBitrate: 150_000, scaleResolutionDownBy: 4 },
    { rid: 'mid', maxBitrate: 500_000, scaleResolutionDownBy: 2 },
    { rid: 'high', maxBitrate: 2_000_000 }
  ]
});
```

**SVC (Scalable Video Coding)**: একটা stream-এর ভিতরে multiple layer (temporal/spatial)। AV1-এ ভালো support।

---

## 🎼 Codecs ও Bandwidth

### Audio Codecs

| Codec | Bitrate | Quality | Notes |
|-------|---------|---------|-------|
| **Opus** | 6-510 kbps | চমৎকার | Default, mandatory in WebRTC |
| G.711 (PCMU/PCMA) | 64 kbps | Telephone | PSTN gateway-এর জন্য |
| G.722 | 64 kbps | Wideband | Legacy |

Opus = always use করুন unless interop দরকার।

### Video Codecs

| Codec | License | Quality | HW Accel | Notes |
|-------|---------|---------|----------|-------|
| **VP8** | royalty-free | OK | অধিকাংশ device | Universal baseline |
| **VP9** | royalty-free | ভালো | অনেক | Bandwidth ৫০% কম |
| **H.264** | patented | ভালো | প্রায় সব device | iOS-এ favored |
| **AV1** | royalty-free | best | নতুন device | Future |

### Bandwidth Math (Pathao Care doctor consult)

```
Per peer (1-on-1):
- 720p @ 30fps:    ~1.5 Mbps up + 1.5 Mbps down = ~3 Mbps
- 480p @ 25fps:    ~600 kbps each direction
- Audio (Opus):    ~40 kbps each direction

বাংলাদেশী 4G typical: 5-15 Mbps → সাধারণত OK
বাংলাদেশী 3G: 1-3 Mbps → 360p/audio-only fallback লাগে
```

### Bandwidth Estimation (GCC)

WebRTC built-in **Google Congestion Control** — packet timing analyze করে available bandwidth estimate, bitrate adjust।

```javascript
// Manual cap (e.g., for low-cost data plans)
const params = sender.getParameters();
params.encodings[0].maxBitrate = 500_000;   // 500 kbps
await sender.setParameters(params);
```

---

## 🌐 Beyond P2P — Mesh, SFU, MCU

### Mesh Topology

প্রতি peer অন্য সবার সাথে individual connection।

```
       A
      ╱│╲
     ╱ │ ╲
    B──┼──D
     ╲ │ ╱
      ╲│╱
       C
```

- **Pros**: কোনো server media handle করে না, server খরচ কম।
- **Cons**: O(N²) connection, N=4-5 এর বেশি হলে client CPU/bandwidth-এ চাপ।
- **Use**: ছোট group call, P2P file transfer।

### SFU (Selective Forwarding Unit)

প্রতিটি peer একটাই connection (server-এর সাথে) রাখে। Server stream গুলো **decrypt না করে** simply route করে।

```
   A ──┐
       │
   B ──┼──► [ SFU ] ──► সবাইকে forward
       │
   C ──┘
```

- **Pros**: scalable (১০০+ peer), bandwidth-efficient client-side।
- **Cons**: server চাই; encryption layer (insertable streams) ছাড়া E2EE নয়।
- **Implementations**: **Mediasoup** (Node, recommended), **Janus** (C, plugin-rich), **Pion** (Go, library), **LiveKit** (cloud + open), **Jitsi Videobridge** (Java)।
- **Use**: production video conference (১০০% case).

### MCU (Multipoint Control Unit / Mixer)

Server সব stream decode করে, mix করে, একটা mixed stream পাঠায়।

```
   A ──┐
       │
   B ──┼──► [ MCU mixes ] ──► single composite stream
       │
   C ──┘
```

- **Pros**: client-side খুবই simple।
- **Cons**: server CPU-heavy (transcoding), latency বেশি, expensive।
- **Use**: legacy systems, recording-heavy use case, telephony bridge।

### Comparison

| | Mesh | SFU | MCU |
|-|------|-----|-----|
| Server cost | shun্য | মাঝারি | উচ্চ |
| Client uplink | N-1 streams | 1 stream | 1 stream |
| Client downlink | N-1 | up to N-1 | 1 mixed |
| Latency | কম | কম | বেশি |
| E2EE | সহজ | insertable streams চাই | প্রায় অসম্ভব |
| Recording | কঠিন | ভালো | চমৎকার |
| Max participants | ~4 | 100+ | 100+ |

### Recording / Transcription

SFU-তে server-side recorder একটা virtual peer হিসেবে join করে subscribe করে। Output: mixed MP4, individual track, বা WebM।

Pipeline:
```
SFU → server-side recorder → S3
                          → Whisper / Deepgram → transcript
                          → translation API → multilingual subtitle
```

---

## 🔒 Security

### DTLS-SRTP — Mandatory

- **DTLS** handshake establishes keys।
- **SRTP** encrypts media RTP packets দিয়ে those keys।
- **SCTP-over-DTLS** encrypts data channel।
- Certificate fingerprint SDP-তে exchange — MITM defense।

### Permissions

Browser user-কে explicit permission চায় (camera/mic/screen)। Permission persist policy site-wise।

```javascript
const status = await navigator.permissions.query({name: 'camera'});
// 'granted', 'denied', 'prompt'
```

### Identity Providers

WebRTC স্পেক identity assertion support করে (BrowserID/Mozilla Persona-এর মতো) কিন্তু practice-এ rarely used। প্রায় সব app নিজের signaling auth-এ depend।

### TURN Authentication

আগেই দেখেছেন — ephemeral REST credential preferred। Long-term password credentials leak হলে আপনার TURN server-এ অপরিচিত stream relay হবে → bandwidth bill blast।

### E2EE (End-to-End)

SFU operator media দেখতে পারলে কিছু সমস্যা (compliance)। **Insertable Streams API** (Chrome) দিয়ে frame-level extra encryption — Signal-এর video call এভাবে।

---

## 🔧 Implementations

### Browser

Native API: `RTCPeerConnection`, `getUserMedia`, `RTCDataChannel`।

### Wrapper Library (Browser/Node)

- **simple-peer** (https://github.com/feross/simple-peer) — lightweight P2P
- **peerjs** (https://peerjs.com) — built-in signaling server option

### Server-side SFU

| Library | Lang | Notes |
|---------|------|-------|
| **Mediasoup** | Node | TS-friendly, battle-tested, used by Discord-style apps |
| **Janus** | C | plugin-based, mature |
| **Pion** | Go | embeddable, modern, used by Twilio |
| **LiveKit** | Go | cloud + self-host, Pathao Care suitable |
| **Jitsi Videobridge** | Java | Jitsi Meet-এর backbone |
| **Ant Media** | Java | low-latency streaming |

### Mobile

- **react-native-webrtc** — React Native module
- **flutter_webrtc** — Flutter
- **iOS WebRTC.framework** (Google built)
- **Android `org.webrtc:google-webrtc`**

### Cloud Service (Build vs Buy)

| Service | Notes |
|---------|-------|
| **Twilio Video** | enterprise-grade, expensive |
| **Agora** | popular in Asia |
| **Daily.co** | dev-friendly |
| **LiveKit Cloud** | open core, sensible pricing |
| **100ms** | India-based, popular in BD |
| **Vonage** | enterprise |

---

## 💻 Code: Browser Video Call

### Server (Node + WebSocket signaling)

```javascript
// signaling-server.js
import { WebSocketServer } from 'ws';

const wss = new WebSocketServer({ port: 8080 });
const rooms = new Map();   // roomId → Set<ws>

wss.on('connection', (ws) => {
  let roomId = null;

  ws.on('message', (raw) => {
    const msg = JSON.parse(raw);

    if (msg.type === 'join') {
      roomId = msg.roomId;
      if (!rooms.has(roomId)) rooms.set(roomId, new Set());
      rooms.get(roomId).add(ws);
      ws.send(JSON.stringify({
        type: 'joined',
        peers: rooms.get(roomId).size
      }));
      return;
    }

    // Forward offer/answer/ice to other peer(s) in room
    if (roomId && rooms.has(roomId)) {
      for (const peer of rooms.get(roomId)) {
        if (peer !== ws && peer.readyState === 1) {
          peer.send(raw.toString());
        }
      }
    }
  });

  ws.on('close', () => {
    if (roomId && rooms.has(roomId)) {
      rooms.get(roomId).delete(ws);
      if (rooms.get(roomId).size === 0) rooms.delete(roomId);
    }
  });
});

console.log('Signaling server :8080');
```

### Client (Browser, ~120 lines)

```html
<!-- index.html -->
<!DOCTYPE html>
<html lang="bn">
<head><meta charset="utf-8"><title>Pathao Care Consult</title></head>
<body>
  <h2>ডাক্তার-রোগী ভিডিও কল</h2>
  <input id="room" placeholder="রুম আইডি" value="room-101">
  <button id="start">কল শুরু</button>
  <div>
    <video id="local" autoplay muted playsinline width="320"></video>
    <video id="remote" autoplay playsinline width="320"></video>
  </div>
  <script type="module" src="/client.js"></script>
</body>
</html>
```

```javascript
// client.js
const ICE_SERVERS = {
  iceServers: [
    { urls: 'stun:stun.cloudflare.com:3478' },
    {
      urls: ['turn:turn.example.com:3478?transport=udp',
             'turns:turn.example.com:5349?transport=tcp'],
      username: 'pathao-user',
      credential: 'ephemeral-token-here'
    }
  ]
};

const localVideo = document.getElementById('local');
const remoteVideo = document.getElementById('remote');
const startBtn = document.getElementById('start');
const roomInput = document.getElementById('room');

let pc, ws, localStream;

startBtn.onclick = async () => {
  // 1. Get media
  localStream = await navigator.mediaDevices.getUserMedia({
    audio: { echoCancellation: true, noiseSuppression: true },
    video: { width: 1280, height: 720, facingMode: 'user' }
  });
  localVideo.srcObject = localStream;

  // 2. Setup peer connection
  pc = new RTCPeerConnection(ICE_SERVERS);
  localStream.getTracks().forEach(t => pc.addTrack(t, localStream));

  pc.ontrack = (e) => { remoteVideo.srcObject = e.streams[0]; };

  pc.onicecandidate = (e) => {
    if (e.candidate) sendSignal({ type: 'ice', candidate: e.candidate });
  };

  pc.oniceconnectionstatechange = () => {
    console.log('ICE state:', pc.iceConnectionState);
    if (pc.iceConnectionState === 'failed') pc.restartIce();
  };

  // 3. Connect signaling
  ws = new WebSocket('wss://signal.pathao-care.local');
  ws.onopen = () => sendSignal({ type: 'join', roomId: roomInput.value });

  ws.onmessage = async (e) => {
    const msg = JSON.parse(e.data);

    if (msg.type === 'joined') {
      // 2nd person to join → make offer
      if (msg.peers === 2) {
        const offer = await pc.createOffer();
        await pc.setLocalDescription(offer);
        sendSignal({ type: 'offer', sdp: offer });
      }
    } else if (msg.type === 'offer') {
      await pc.setRemoteDescription(msg.sdp);
      const answer = await pc.createAnswer();
      await pc.setLocalDescription(answer);
      sendSignal({ type: 'answer', sdp: answer });
    } else if (msg.type === 'answer') {
      await pc.setRemoteDescription(msg.sdp);
    } else if (msg.type === 'ice') {
      try { await pc.addIceCandidate(msg.candidate); }
      catch (err) { console.warn('ICE fail', err); }
    }
  };
};

function sendSignal(obj) {
  ws.send(JSON.stringify(obj));
}

// Quality monitoring (every 5s)
setInterval(async () => {
  if (!pc) return;
  const stats = await pc.getStats();
  stats.forEach(report => {
    if (report.type === 'inbound-rtp' && report.kind === 'video') {
      console.log('packetsLost:', report.packetsLost,
                  'jitter:', report.jitter,
                  'fps:', report.framesPerSecond);
    }
  });
}, 5000);
```

### Test on local network only?

`getUserMedia` requires HTTPS (or `localhost`)। dev-এ `mkcert` দিয়ে local cert বানান।

---

## 🌀 Code: coturn Setup (Ubuntu 22.04)

```bash
sudo apt update && sudo apt install -y coturn
sudo systemctl enable coturn
```

```bash
# /etc/turnserver.conf

listening-port=3478
tls-listening-port=5349
listening-ip=0.0.0.0
relay-ip=YOUR_PUBLIC_IP

external-ip=YOUR_PUBLIC_IP

# Authentication — REST API style (ephemeral)
use-auth-secret
static-auth-secret=YOUR_LONG_RANDOM_SECRET

realm=turn.pathao-care.com
server-name=turn.pathao-care.com

# TLS — Let's Encrypt
cert=/etc/letsencrypt/live/turn.pathao-care.com/fullchain.pem
pkey=/etc/letsencrypt/live/turn.pathao-care.com/privkey.pem

# Security
no-multicast-peers
no-cli                                       # disable telnet admin
no-tlsv1
no-tlsv1_1
cipher-list="ECDHE+AESGCM:ECDHE+CHACHA20:DHE+AESGCM"

# Quotas
total-quota=300                              # concurrent sessions
user-quota=12
max-bps=2000000                              # 2 Mbps per user

# Logging
log-file=/var/log/turnserver.log
verbose

# Block private ranges (don't relay to internal infra!)
denied-peer-ip=10.0.0.0-10.255.255.255
denied-peer-ip=172.16.0.0-172.31.255.255
denied-peer-ip=192.168.0.0-192.168.255.255
denied-peer-ip=169.254.0.0-169.254.255.255
denied-peer-ip=127.0.0.0-127.255.255.255
denied-peer-ip=::1
denied-peer-ip=fe80::-fe80::ffff:ffff:ffff:ffff
```

```bash
# Open firewall
sudo ufw allow 3478/udp
sudo ufw allow 3478/tcp
sudo ufw allow 5349/tcp
sudo ufw allow 49152:65535/udp                # relay range

sudo systemctl restart coturn

# Verify
sudo journalctl -u coturn -f
turnutils_uclient -v -u testuser -w testpass turn.pathao-care.com
```

### TURN Bandwidth Cost

```
1-on-1 720p call relayed: ~3 Mbps total = ~1.4 GB/hour
TURN server bandwidth: ~3 Mbps in + 3 Mbps out = 6 Mbps charged

AWS egress: $0.09/GB → 1 hour call ~ $0.13
1000 hours/day → $130/day → $4000/month

→ TURN traffic যত কম হয় তত ভালো
→ STUN দিয়ে যত বেশি pair direct হয় তত সাশ্রয়ী
→ DigitalOcean/Hetzner-এ self-host অনেক সস্তা (BD market-এর জন্য suitable)
```

---

## ☁️ Code: LiveKit Quickstart (Pathao Care)

LiveKit হলো একটি modern open-source SFU (Go) — self-host বা cloud।

### Server install (Docker)

```yaml
# docker-compose.yml
services:
  livekit:
    image: livekit/livekit-server:latest
    command: --config /etc/livekit.yaml
    ports:
      - "7880:7880"        # WebSocket signaling
      - "7881:7881"        # TCP fallback
      - "50000-60000:50000-60000/udp"   # media
    volumes:
      - ./livekit.yaml:/etc/livekit.yaml
```

```yaml
# livekit.yaml
port: 7880
rtc:
  tcp_port: 7881
  port_range_start: 50000
  port_range_end: 60000
  use_external_ip: true
  stun_servers:
    - stun.cloudflare.com:3478
keys:
  pathao-key-id: very-long-shared-secret-here
turn:
  enabled: true
  domain: turn.pathao-care.com
```

### Backend (Node) — token issue

```javascript
// server/livekit-token.js
import { AccessToken } from 'livekit-server-sdk';

export function makeConsultToken({ userId, role, roomName }) {
  const at = new AccessToken('pathao-key-id', 'very-long-shared-secret-here', {
    identity: userId,
    name: role === 'doctor' ? `Dr. ${userId}` : `Patient ${userId}`,
    ttl: 60 * 60                      // 1 hour
  });
  at.addGrant({
    room: roomName,
    roomJoin: true,
    canPublish: true,
    canSubscribe: true,
    canPublishData: true,
    recorder: false
  });
  return at.toJwt();
}
```

### Client (React) — Pathao Care patient

```jsx
// PatientCallScreen.jsx
import { LiveKitRoom, VideoConference } from '@livekit/components-react';
import '@livekit/components-styles';

export function PatientCallScreen({ token, room }) {
  return (
    <LiveKitRoom
      serverUrl="wss://livekit.pathao-care.com"
      token={token}
      connect={true}
      video={true}
      audio={true}
      onDisconnected={() => alert('কল শেষ হয়েছে')}
      data-lk-theme="default"
    >
      <VideoConference />
    </LiveKitRoom>
  );
}
```

বাকি hard work (SFU, recording, screen share UI) LiveKit handle করে।

---

## 🔭 Operational Concerns

### Quality Monitoring (`getStats`)

```javascript
async function reportQuality(pc) {
  const stats = await pc.getStats();
  const m = { rtt: 0, packetsLost: 0, jitter: 0, bytesSent: 0 };
  stats.forEach(r => {
    if (r.type === 'remote-inbound-rtp') {
      m.rtt = r.roundTripTime;
      m.packetsLost = r.packetsLost;
      m.jitter = r.jitter;
    }
    if (r.type === 'outbound-rtp') m.bytesSent = r.bytesSent;
  });
  // ship to Datadog/Prometheus
  navigator.sendBeacon('/metrics', JSON.stringify(m));
}
```

KPI:
- **RTT** > 300ms → noticeable lag
- **packetLoss** > 5% → quality drop
- **jitter** > 30ms → audio glitch
- **resolution drops** → bandwidth-throttled

### Mobile Network in BD

- 4G handover (cell tower change) → IP change → ICE restart দরকার:
  ```javascript
  pc.restartIce();   // new ICE gathering, no signaling renegotiation needed (since RFC 5245)
  ```
- 3G fallback: video off, audio Opus low-bitrate
- Adaptive bitrate auto-handle most case

### Server-side Capacity Planning

| | 1-on-1 Mesh | SFU (per server) |
|-|-------------|------------------|
| 100 sessions | client-only | ~100 streams in/out, depends on CPU |
| Bandwidth | 0 server | (N peers) × 1.5 Mbps each direction |
| CPU | minimal | medium (forwarding, no transcode) |

LiveKit/Mediasoup single 8-core node ~ ৩০০-৫০০ concurrent participant।

### Recording

- Composite recording (MCU-style): server-side ffmpeg pipeline।
- Track-individual: SFU per-track WebM/MP4।
- Egress to S3, post-process for transcription।

---

## 🆚 WebRTC vs Alternatives

| | WebRTC | WebSocket + Media Server | HLS / DASH | RTMP | WebTransport |
|-|--------|--------------------------|------------|------|--------------|
| Latency | <500ms | 200ms-2s | 6-30s | 2-5s | <500ms |
| Audience | up to ~100 | 1000s | unlimited | 1000s | ? |
| Encoding | client | client | server | client | client |
| P2P? | ✅ | ❌ | ❌ | ❌ | ❌ |
| Browser native | ✅ | ✅ (player needed) | ✅ (HLS.js) | ❌ | partial |
| Use case | calling, conferencing | chat + video | live broadcast | OBS streaming | future |

**Rule of thumb**:
- Few-to-few interactive → WebRTC SFU
- One-to-many large audience → HLS (server transcodes from RTMP/WebRTC ingest)
- Two-way data only, modern HTTP/3 → WebTransport (still experimental)

### Hybrid Pattern

10 Minute School live class:
- Teacher → WebRTC ingest → Media server transcodes → HLS to thousands of student
- Student chat → WebSocket
- Doubt asking → optional WebRTC P2P or audio-only

---

## 🇧🇩 Bangladesh Scenarios

### ১. Pathao Care — Telemedicine Doctor Consult

- 1-on-1 video, 30 min average
- LiveKit Cloud / self-host (cost-aware: BD-hosted TURN = সস্তা data)
- Recording for compliance (DGHS, BMDC)
- Mobile-first (>80% Android)
- 3G fallback: audio + low-res video

### ২. 10 Minute School — Live Class

- Teacher publish via WebRTC ingest
- Backend transcode → HLS (CDN — Cloudflare)
- Student watch via HLS player (low cost, scales to ১০ হাজার)
- Doubt: WebRTC audio P2P / SFU

### ৩. bKash — Agent Video KYC

- Customer-agent video call for KYC (NID match)
- Mandatory recording (regulator-compliant)
- E2E encrypted (insertable streams) — sensitive data
- BD data residency: TURN/SFU on local cloud (Akij Cloud, Brain Station 23)

### ৪. Foodpanda — Customer-Rider Issue Resolution

- WebRTC audio call (cheap, no number reveal)
- Number masking (Twilio Voice / Plivo + WebRTC bridge)
- Anonymity, low cost, BD telco friendly

### ৫. Daraz — Live Commerce

- Seller live → ১০০০+ viewer
- HLS ingest from seller's WebRTC stream
- Real-time chat: WebSocket
- Reaction/heart: SSE or WebSocket

---

## ⚠️ Pitfalls

❌ **TURN ছাড়া production launch** — symmetric NAT-এ ফেইল হবে।
❌ **Free Google STUN-এ depend** — production-এ unreliable।
❌ **Long-term TURN credential hardcode** — leak হলে bandwidth bill।
❌ **CPU monitoring না করা** — SFU আগে CPU bound হয় bandwidth এর আগে।
❌ **Mesh-এ ৪+ peer** → প্রতি client crash।
❌ **`new RTCPeerConnection()` re-create বার বার** → ICE restart use করুন।
❌ **`getUserMedia` permission ছাড়া HTTP-তে** → silently fail।
❌ **HD হার্ডকোড** → low-end device-এ frame drop। `applyConstraints` দিয়ে adaptive।
❌ **Bandwidth cap না দেওয়া** → BD data plan-এ user bill phulko।
❌ **Recording compliance miss** → BMDC/RJSC requirement check।
❌ **Echo cancellation off** → অসহ্য feedback।
❌ **Signaling-এ end auth check না করা** → strangers join your room।
❌ **getStats ignore করা** → quality issue silent।
❌ **One-to-many-তে WebRTC mesh** → use HLS।
❌ **Mobile background-এ keep-alive পরিকল্পনা নেই** → call drop।

---

## ✅ Senior Engineer WebRTC Checklist

**Architecture:**
- [ ] মিশন: 1-on-1, group, broadcast — সঠিক topology বেছেছেন
- [ ] SFU (LiveKit/Mediasoup/Janus) যদি 4+ peer
- [ ] HLS pipeline yes/no decided

**STUN/TURN:**
- [ ] Production-grade TURN (Cloudflare, Twilio, self-host coturn)
- [ ] TURN over TCP/TLS port 443 fallback
- [ ] Ephemeral REST credentials
- [ ] Bandwidth cost projection আছে

**Signaling:**
- [ ] WebSocket-based (or SSE)
- [ ] Authentication on connect
- [ ] Room-level authorization
- [ ] Reconnect logic + ICE restart

**Media:**
- [ ] Opus audio, VP8/H.264/VP9 video
- [ ] Simulcast for SFU
- [ ] Adaptive bitrate (GCC) verified
- [ ] Bandwidth caps for low-data plan

**Monitoring:**
- [ ] `getStats` shipping to APM (Datadog/Prometheus)
- [ ] RTT/jitter/loss/fps dashboards
- [ ] Alert: TURN relay percentage spike (NAT issue)

**Security:**
- [ ] DTLS-SRTP verified
- [ ] HTTPS/WSS only
- [ ] Permissions UI clear (camera/mic icon)
- [ ] E2EE (insertable streams) যদি sensitive
- [ ] TURN access logged

**Mobile/BD:**
- [ ] react-native-webrtc / flutter_webrtc tested
- [ ] 3G fallback (audio-only)
- [ ] Background/foreground transitions handled
- [ ] Battery impact tested
- [ ] Local TURN PoP (Dhaka) for সাশ্রয়ী relay

**Recording/Compliance:**
- [ ] Server-side recording yes/no
- [ ] User consent UI
- [ ] Data retention policy
- [ ] BD regulatory: BMDC (medical), BB (financial)

---

> **মনে রাখুন**: WebRTC একটি massive stack — `RTCPeerConnection` শুধু টিপ। Signaling, TURN, SFU choice, codecs, monitoring — production-এ এগুলোই সাফল্য বা failure ঠিক করে। ছোট MVP-এ self-host coturn + mediasoup, বড় হলে LiveKit Cloud / 100ms / Agora দিয়ে scale করুন। বাংলাদেশে bandwidth দাম ও mobile network reality WebRTC architecture-এ বড় factor।
