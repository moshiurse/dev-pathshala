# 🔌 WebSockets

> WebSocket হলো একটি full-duplex কমিউনিকেশন প্রোটোকল যা ক্লায়েন্ট ও সার্ভারের মধ্যে persistent connection বজায় রাখে।
> রিয়েল-টাইম চ্যাট, লাইভ আপডেট, গেমিং — এসব ক্ষেত্রে WebSocket অপরিহার্য।

---

## 📖 সূচিপত্র

- [WebSocket কী?](#-websocket-কী)
- [HTTP vs WebSocket](#-http-vs-websocket)
- [Handshake Process](#-handshake-process)
- [WebSocket Frame Structure](#-websocket-frame-structure)
- [Use Cases](#-use-cases)
- [Socket.IO vs Raw WebSocket](#-socketio-vs-raw-websocket)
- [Node.js উদাহরণ](#-nodejs-উদাহরণ)
- [PHP Ratchet উদাহরণ](#-php-ratchet-উদাহরণ)
- [Server-Sent Events (SSE)](#-server-sent-events-sse)
- [তুলনা: WebSocket vs SSE vs Long Polling](#-তুলনা)
- [কখন ব্যবহার করবেন](#-কখন-ব্যবহার-করবেন--করবেন-না)

---

## 📌 WebSocket কী?

WebSocket (RFC 6455) একটি **full-duplex** (দ্বিমুখী) কমিউনিকেশন প্রোটোকল। HTTP এর মতো request-response নয় — একবার সংযোগ স্থাপিত হলে, উভয় পক্ষ যেকোনো সময় মেসেজ পাঠাতে পারে।

```
HTTP (Half-Duplex / Request-Response):
──────────────────────────────────────
Client                    Server
  │── Request ──────────►│
  │◄── Response ──────────│
  │                       │  (অপেক্ষা...)
  │── Request ──────────►│
  │◄── Response ──────────│
  │                       │  (অপেক্ষা...)

  ❌ সার্ভার নিজে থেকে মেসেজ পাঠাতে পারে না
  ❌ প্রতিটি request এ নতুন HTTP header overhead

WebSocket (Full-Duplex):
────────────────────────
Client                    Server
  │── HTTP Upgrade ─────►│  (Handshake)
  │◄── 101 Switching ────│
  │                       │
  │══ Persistent Connection ══│
  │                       │
  │── Message ──────────►│  (ক্লায়েন্ট যখন খুশি)
  │◄── Message ──────────│  (সার্ভারও যখন খুশি!)
  │◄── Message ──────────│  (সার্ভার push!)
  │── Message ──────────►│
  │◄── Message ──────────│
  │                       │

  ✅ উভয় দিক থেকে যেকোনো সময় মেসেজ পাঠানো যায়
  ✅ কম overhead (ছোট frame header)
  ✅ Persistent connection
```

---

## 📊 HTTP vs WebSocket

```
┌────────────────────┬──────────────────┬──────────────────┐
│     বৈশিষ্ট্য       │      HTTP        │    WebSocket     │
├────────────────────┼──────────────────┼──────────────────┤
│ কমিউনিকেশন         │ Half-duplex      │ Full-duplex      │
│                    │ (Request/Resp)   │ (যেকোনো দিক)     │
├────────────────────┼──────────────────┼──────────────────┤
│ Connection         │ প্রতি request এ   │ Persistent       │
│                    │ নতুন (বা keep-   │ (একবার খুলে      │
│                    │ alive)           │ রাখা)            │
├────────────────────┼──────────────────┼──────────────────┤
│ Header Overhead    │ বড় (প্রতিবার)    │ ছোট (2-14 bytes) │
├────────────────────┼──────────────────┼──────────────────┤
│ Server Push        │ ❌ (HTTP/1.1)    │ ✅               │
├────────────────────┼──────────────────┼──────────────────┤
│ Protocol           │ http:// https:// │ ws:// wss://     │
├────────────────────┼──────────────────┼──────────────────┤
│ Stateful           │ ❌ Stateless     │ ✅ Stateful      │
├────────────────────┼──────────────────┼──────────────────┤
│ Latency            │ উচ্চ (header +   │ নিম্ন (already   │
│                    │ new connection)  │ connected)       │
├────────────────────┼──────────────────┼──────────────────┤
│ Use Case           │ REST API, Web    │ Chat, Gaming,    │
│                    │ Pages, File DL   │ Live Updates     │
└────────────────────┴──────────────────┴──────────────────┘
```

---

## 🔗 Handshake Process

WebSocket সংযোগ একটি সাধারণ HTTP request দিয়ে শুরু হয়, তারপর **Upgrade** হয়।

```
Client                                    Server
  │                                         │
  │  HTTP GET /chat HTTP/1.1               │
  │  Host: chat.pathao.com                  │
  │  Upgrade: websocket          ──────────►│
  │  Connection: Upgrade                    │
  │  Sec-WebSocket-Key: dGhlIH...          │
  │  Sec-WebSocket-Version: 13             │
  │  Sec-WebSocket-Protocol: chat          │
  │                                         │
  │  HTTP/1.1 101 Switching Protocols      │
  │  Upgrade: websocket          ◄──────────│
  │  Connection: Upgrade                    │
  │  Sec-WebSocket-Accept: s3pPL...        │
  │                                         │
  │  ══════ WebSocket Connection ══════    │
  │  (এখন থেকে HTTP নয়, WebSocket!)        │
  │                                         │
```

### Handshake বিস্তারিত

```
Step 1: Client → Server (HTTP Upgrade Request)
────────────────────────────────────────────────
GET /chat HTTP/1.1
Host: chat.pathao.com
Upgrade: websocket                    ← প্রোটোকল পরিবর্তন চাই
Connection: Upgrade                   ← সংযোগ upgrade করো
Sec-WebSocket-Key: dGhlIHNhbXBs...   ← র‍্যান্ডম base64 key
Sec-WebSocket-Version: 13            ← WebSocket ভার্সন
Origin: https://pathao.com           ← কোন সাইট থেকে
Sec-WebSocket-Protocol: chat, json   ← সাব-প্রোটোকল পছন্দ

Step 2: Server → Client (101 Switching Protocols)
─────────────────────────────────────────────────
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: HSmrc0sM...    ← Key এর SHA-1 hash
Sec-WebSocket-Protocol: chat         ← নির্বাচিত সাব-প্রোটোকল

Sec-WebSocket-Accept কীভাবে তৈরি হয়:
────────────────────────────────────
Accept = Base64(SHA1(Key + "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"))
```

---

## 📖 WebSocket Frame Structure

```
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-------+-+-------------+-------------------------------+
 |F|R|R|R| opcode|M| Payload len |    Extended payload length    |
 |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
 |N|V|V|V|       |S|             |   (if payload len==126/127)   |
 | |1|2|3|       |K|             |                               |
 +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
 |     Extended payload length continued, if payload len == 127  |
 + - - - - - - - - - - - - - - - +-------------------------------+
 |                               |Masking-key, if MASK set to 1  |
 +-------------------------------+-------------------------------+
 | Masking-key (continued)       |          Payload Data         |
 +-------------------------------- - - - - - - - - - - - - - - - +
 :                     Payload Data continued ...                :
 + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
 |                     Payload Data continued ...                |
 +---------------------------------------------------------------+

 FIN: শেষ ফ্রেম কিনা (1 = হ্যাঁ)
 Opcode:
   0x0 = Continuation
   0x1 = Text
   0x2 = Binary
   0x8 = Close
   0x9 = Ping
   0xA = Pong
 MASK: ক্লায়েন্ট → সার্ভার সবসময় masked (1)

 মিনিমাম হেডার: মাত্র ২ বাইট! (HTTP এর ~800 বাইটের তুলনায়)
```

---

## 🎯 Use Cases

### ১. রিয়েল-টাইম চ্যাট (Pathao Chat)

```
রাইডার অ্যাপ                  সার্ভার                  কাস্টমার অ্যাপ
    │                          │                          │
    │  ws://chat.pathao.com    │    ws://chat.pathao.com  │
    │═══════ Connected ════════│══════ Connected ══════════│
    │                          │                          │
    │── "আমি ৫ মিনিটে আসছি" ──►│                          │
    │                          │── "আমি ৫ মিনিটে আসছি" ──►│
    │                          │                          │
    │                          │◄── "OK, ধন্যবাদ" ────────│
    │◄── "OK, ধন্যবাদ" ────────│                          │
    │                          │                          │
```

### ২. লাইভ লোকেশন ট্র্যাকিং

```
রাইডার GPS                    সার্ভার                  কাস্টমার ম্যাপ
    │                          │                          │
    │── {lat:23.81, lng:90.41}►│                          │
    │                          │── {lat:23.81, lng:90.41}►│
    │── {lat:23.82, lng:90.42}►│                          │  ← রিয়েল-টাইম
    │                          │── {lat:23.82, lng:90.42}►│    ম্যাপ আপডেট!
    │── {lat:23.83, lng:90.43}►│                          │
    │                          │── {lat:23.83, lng:90.43}►│
```

### ৩. লাইভ ক্রিকেট স্কোর

```
ESPNCricinfo Server                    ইউজারের ব্রাউজার
       │                                    │
       │── {"score":"150/3", "overs":"20"}──►│  ← বল-বাই-বল আপডেট!
       │── {"event":"FOUR!", "batsman":"..."}}►│
       │── {"score":"154/3", "overs":"20.1"}►│
       │── {"event":"WICKET!", "bowler":"..."}►│
       │── {"score":"154/4", "overs":"20.2"}►│
```

### ৪. স্টক/ক্রিপ্টো প্রাইস (Nagad/bKash Exchange Rate)

```
Exchange Server                        ট্রেডার ড্যাশবোর্ড
       │                                    │
       │── {"BDT_USD": 110.50} ────────────►│
       │── {"BDT_USD": 110.55} ────────────►│  ← প্রতি ms এ আপডেট!
       │── {"BDT_USD": 110.48} ────────────►│
       │── {"BDT_USD": 110.52} ────────────►│
```

---

## 📊 Socket.IO vs Raw WebSocket

```
┌────────────────────┬──────────────────┬──────────────────┐
│     বৈশিষ্ট্য       │  Raw WebSocket   │    Socket.IO     │
├────────────────────┼──────────────────┼──────────────────┤
│ প্রোটোকল           │ WebSocket only   │ WS + Fallbacks   │
│                    │                  │ (polling, etc)   │
├────────────────────┼──────────────────┼──────────────────┤
│ Auto Reconnect     │ ❌ নিজে করতে হয়  │ ✅ Built-in      │
├────────────────────┼──────────────────┼──────────────────┤
│ Rooms/Namespaces   │ ❌ নিজে করতে হয়  │ ✅ Built-in      │
├────────────────────┼──────────────────┼──────────────────┤
│ Event System       │ ❌ শুধু message   │ ✅ Custom events │
├────────────────────┼──────────────────┼──────────────────┤
│ Binary Support     │ ✅               │ ✅               │
├────────────────────┼──────────────────┼──────────────────┤
│ Broadcasting       │ ❌ নিজে করতে হয়  │ ✅ Built-in      │
├────────────────────┼──────────────────┼──────────────────┤
│ Acknowledgments    │ ❌               │ ✅ Built-in      │
├────────────────────┼──────────────────┼──────────────────┤
│ Overhead           │ কম               │ বেশি (protocol)  │
├────────────────────┼──────────────────┼──────────────────┤
│ Library Size       │ ০ (Native)       │ ~50KB            │
├────────────────────┼──────────────────┼──────────────────┤
│ ব্যবহার             │ সিম্পল কেসে      │ কমপ্লেক্স অ্যাপে  │
└────────────────────┴──────────────────┴──────────────────┘
```

---

## 💻 Node.js উদাহরণ

### Raw WebSocket (ws library)

```javascript
// =============================================
// WebSocket Server (Node.js - ws library)
// বাস্তব উদাহরণ: Pathao-style চ্যাট সিস্টেম
// =============================================

const WebSocket = require('ws');
const http = require('http');

const server = http.createServer();
const wss = new WebSocket.Server({ server });

// সব সংযুক্ত ক্লায়েন্ট ট্র্যাক করা
const clients = new Map(); // userId → WebSocket

wss.on('connection', (ws, req) => {
    const userId = req.url.split('?userId=')[1];
    clients.set(userId, ws);
    console.log(`ইউজার ${userId} সংযুক্ত হয়েছে। মোট: ${clients.size}`);
    
    ws.on('message', (data) => {
        const message = JSON.parse(data);
        
        switch (message.type) {
            case 'chat':
                handleChat(userId, message);
                break;
            case 'location':
                handleLocation(userId, message);
                break;
            case 'typing':
                handleTyping(userId, message);
                break;
        }
    });
    
    ws.on('close', () => {
        clients.delete(userId);
        console.log(`ইউজার ${userId} বিচ্ছিন্ন। মোট: ${clients.size}`);
    });
    
    // Heartbeat (সংযোগ জীবিত কিনা চেক)
    ws.isAlive = true;
    ws.on('pong', () => { ws.isAlive = true; });
});

// চ্যাট মেসেজ হ্যান্ডেল
function handleChat(senderId, message) {
    const { receiverId, text } = message;
    const receiverWs = clients.get(receiverId);
    
    if (receiverWs && receiverWs.readyState === WebSocket.OPEN) {
        receiverWs.send(JSON.stringify({
            type: 'chat',
            from: senderId,
            text: text,
            timestamp: Date.now()
        }));
    }
}

// লোকেশন আপডেট (রাইডার → কাস্টমার)
function handleLocation(riderId, message) {
    const { customerId, lat, lng } = message;
    const customerWs = clients.get(customerId);
    
    if (customerWs && customerWs.readyState === WebSocket.OPEN) {
        customerWs.send(JSON.stringify({
            type: 'location',
            riderId,
            lat,
            lng,
            timestamp: Date.now()
        }));
    }
}

// টাইপিং ইন্ডিকেটর
function handleTyping(senderId, message) {
    const { receiverId } = message;
    const receiverWs = clients.get(receiverId);
    
    if (receiverWs && receiverWs.readyState === WebSocket.OPEN) {
        receiverWs.send(JSON.stringify({
            type: 'typing',
            from: senderId
        }));
    }
}

// Heartbeat — ৩০ সেকেন্ড পর পর চেক
const heartbeat = setInterval(() => {
    wss.clients.forEach((ws) => {
        if (!ws.isAlive) {
            return ws.terminate();
        }
        ws.isAlive = false;
        ws.ping();
    });
}, 30000);

wss.on('close', () => clearInterval(heartbeat));

server.listen(8080, () => {
    console.log('WebSocket সার্ভার চালু: ws://localhost:8080');
});
```

### Raw WebSocket Client (Browser)

```javascript
// =============================================
// WebSocket Client (Browser)
// =============================================

class PathaoChat {
    constructor(userId) {
        this.userId = userId;
        this.ws = null;
        this.reconnectAttempts = 0;
        this.maxReconnectAttempts = 5;
    }
    
    connect() {
        this.ws = new WebSocket(`wss://chat.pathao.com?userId=${this.userId}`);
        
        this.ws.onopen = () => {
            console.log('সংযুক্ত! ✅');
            this.reconnectAttempts = 0;
        };
        
        this.ws.onmessage = (event) => {
            const message = JSON.parse(event.data);
            
            switch (message.type) {
                case 'chat':
                    this.onChatMessage(message);
                    break;
                case 'location':
                    this.onLocationUpdate(message);
                    break;
                case 'typing':
                    this.onTypingIndicator(message);
                    break;
            }
        };
        
        this.ws.onclose = (event) => {
            console.log(`সংযোগ বিচ্ছিন্ন (code: ${event.code})`);
            this.tryReconnect();
        };
        
        this.ws.onerror = (error) => {
            console.error('WebSocket error:', error);
        };
    }
    
    sendMessage(receiverId, text) {
        if (this.ws.readyState === WebSocket.OPEN) {
            this.ws.send(JSON.stringify({
                type: 'chat',
                receiverId,
                text
            }));
        }
    }
    
    sendLocation(customerId, lat, lng) {
        if (this.ws.readyState === WebSocket.OPEN) {
            this.ws.send(JSON.stringify({
                type: 'location',
                customerId,
                lat,
                lng
            }));
        }
    }
    
    // Auto-Reconnect (Exponential Backoff)
    tryReconnect() {
        if (this.reconnectAttempts >= this.maxReconnectAttempts) {
            console.log('সংযোগ পুনঃস্থাপন ব্যর্থ ❌');
            return;
        }
        
        const delay = Math.min(1000 * Math.pow(2, this.reconnectAttempts), 30000);
        this.reconnectAttempts++;
        
        console.log(`${delay / 1000}s পর পুনঃসংযোগ চেষ্টা #${this.reconnectAttempts}...`);
        setTimeout(() => this.connect(), delay);
    }
    
    onChatMessage(message) {
        console.log(`${message.from}: ${message.text}`);
    }
    
    onLocationUpdate(message) {
        console.log(`রাইডার অবস্থান: ${message.lat}, ${message.lng}`);
    }
    
    onTypingIndicator(message) {
        console.log(`${message.from} টাইপ করছে...`);
    }
    
    disconnect() {
        if (this.ws) {
            this.ws.close(1000, 'ইউজার লগআউট করেছে');
        }
    }
}

// ব্যবহার
const chat = new PathaoChat('customer-123');
chat.connect();
chat.sendMessage('rider-456', 'কোথায় আছেন?');
```

### Socket.IO (Node.js)

```javascript
// =============================================
// Socket.IO Server
// বাস্তব উদাহরণ: Daraz-style লাইভ নোটিফিকেশন
// =============================================

const { Server } = require('socket.io');
const http = require('http');

const httpServer = http.createServer();
const io = new Server(httpServer, {
    cors: {
        origin: 'https://www.daraz.com.bd',
        methods: ['GET', 'POST']
    }
});

// Namespace: /orders
const ordersNs = io.of('/orders');

ordersNs.on('connection', (socket) => {
    const userId = socket.handshake.auth.userId;
    console.log(`ইউজার ${userId} orders namespace এ যোগ দিয়েছে`);
    
    // ইউজারকে তার নিজের room এ যোগ করা
    socket.join(`user:${userId}`);
    
    // অর্ডার স্ট্যাটাস আপডেট
    socket.on('track-order', (orderId) => {
        socket.join(`order:${orderId}`);
        console.log(`ইউজার ${userId} অর্ডার ${orderId} ট্র্যাক করছে`);
    });
    
    socket.on('disconnect', () => {
        console.log(`ইউজার ${userId} বিচ্ছিন্ন`);
    });
});

// Namespace: /chat (কাস্টমার সার্ভিস)
const chatNs = io.of('/chat');

chatNs.on('connection', (socket) => {
    const userId = socket.handshake.auth.userId;
    
    // চ্যাট room এ যোগ দেওয়া
    socket.on('join-room', (roomId) => {
        socket.join(roomId);
        chatNs.to(roomId).emit('user-joined', {
            userId,
            message: `${userId} যোগ দিয়েছে`
        });
    });
    
    // মেসেজ পাঠানো
    socket.on('send-message', (data, callback) => {
        chatNs.to(data.roomId).emit('new-message', {
            from: userId,
            text: data.text,
            timestamp: Date.now()
        });
        
        // Acknowledgment (মেসেজ পৌঁছেছে কিনা)
        callback({ status: 'delivered' });
    });
});

// সার্ভার থেকে নোটিফিকেশন push করা
function notifyOrderStatus(userId, orderId, status) {
    ordersNs.to(`user:${userId}`).emit('order-update', {
        orderId,
        status,
        message: getStatusMessage(status),
        timestamp: Date.now()
    });
    
    ordersNs.to(`order:${orderId}`).emit('order-update', {
        orderId,
        status,
        message: getStatusMessage(status),
        timestamp: Date.now()
    });
}

function getStatusMessage(status) {
    const messages = {
        'confirmed':  'আপনার অর্ডার নিশ্চিত হয়েছে ✅',
        'packed':     'প্রোডাক্ট প্যাক করা হয়েছে 📦',
        'shipped':    'ডেলিভারি ম্যানের কাছে হস্তান্তর 🚚',
        'out_for_delivery': 'আপনার এলাকায় ডেলিভারি চলছে 🏃',
        'delivered':  'ডেলিভারি সম্পন্ন! 🎉'
    };
    return messages[status] || status;
}

httpServer.listen(3000, () => {
    console.log('Socket.IO সার্ভার চালু: http://localhost:3000');
});
```

### Socket.IO Client (Browser)

```javascript
// =============================================
// Socket.IO Client
// =============================================

import { io } from 'socket.io-client';

// Orders namespace
const ordersSocket = io('https://api.daraz.com.bd/orders', {
    auth: {
        userId: 'customer-123',
        token: 'jwt-token-here'
    },
    reconnection: true,
    reconnectionDelay: 1000,
    reconnectionAttempts: 5
});

ordersSocket.on('connect', () => {
    console.log('Orders namespace এ সংযুক্ত ✅');
    
    // অর্ডার ট্র্যাক করা
    ordersSocket.emit('track-order', 'ORD-456');
});

ordersSocket.on('order-update', (data) => {
    console.log(`অর্ডার ${data.orderId}: ${data.message}`);
    showNotification(data.message); // ব্রাউজার নোটিফিকেশন
});

// Chat namespace
const chatSocket = io('https://api.daraz.com.bd/chat', {
    auth: { userId: 'customer-123' }
});

chatSocket.on('connect', () => {
    chatSocket.emit('join-room', 'support-room-789');
});

// Acknowledgment সহ মেসেজ পাঠানো
chatSocket.emit('send-message', {
    roomId: 'support-room-789',
    text: 'আমার অর্ডার কোথায়?'
}, (response) => {
    console.log('মেসেজ স্ট্যাটাস:', response.status); // 'delivered'
});

chatSocket.on('new-message', (data) => {
    console.log(`${data.from}: ${data.text}`);
});
```

---

## 💻 PHP Ratchet উদাহরণ

```php
<?php
// composer require cboden/ratchet

// =============================================
// PHP WebSocket Server (Ratchet)
// বাস্তব উদাহরণ: লাইভ ক্রিকেট স্কোর সিস্টেম
// =============================================

use Ratchet\MessageComponentInterface;
use Ratchet\ConnectionInterface;
use Ratchet\Server\IoServer;
use Ratchet\Http\HttpServer;
use Ratchet\WebSocket\WsServer;

require __DIR__ . '/vendor/autoload.php';

class CricketScoreServer implements MessageComponentInterface
{
    protected \SplObjectStorage $clients;
    protected array $rooms;
    
    public function __construct()
    {
        $this->clients = new \SplObjectStorage();
        $this->rooms = [];
    }
    
    public function onOpen(ConnectionInterface $conn): void
    {
        $this->clients->attach($conn);
        echo "নতুন সংযোগ #{$conn->resourceId}\n";
    }
    
    public function onMessage(ConnectionInterface $from, $msg): void
    {
        $data = json_decode($msg, true);
        
        switch ($data['action']) {
            case 'subscribe':
                $this->subscribeToMatch($from, $data['matchId']);
                break;
                
            case 'score_update':
                $this->broadcastScore($data);
                break;
                
            case 'commentary':
                $this->broadcastCommentary($data);
                break;
        }
    }
    
    private function subscribeToMatch(ConnectionInterface $conn, string $matchId): void
    {
        if (!isset($this->rooms[$matchId])) {
            $this->rooms[$matchId] = new \SplObjectStorage();
        }
        $this->rooms[$matchId]->attach($conn);
        
        $conn->send(json_encode([
            'type' => 'subscribed',
            'matchId' => $matchId,
            'message' => "ম্যাচ $matchId এ সাবস্ক্রাইব করা হয়েছে ✅"
        ]));
    }
    
    private function broadcastScore(array $data): void
    {
        $matchId = $data['matchId'];
        if (!isset($this->rooms[$matchId])) return;
        
        $scoreUpdate = json_encode([
            'type' => 'score',
            'matchId' => $matchId,
            'team' => $data['team'],
            'runs' => $data['runs'],
            'wickets' => $data['wickets'],
            'overs' => $data['overs'],
            'batsman' => $data['batsman'],
            'bowler' => $data['bowler'],
            'timestamp' => time()
        ]);
        
        foreach ($this->rooms[$matchId] as $client) {
            $client->send($scoreUpdate);
        }
    }
    
    private function broadcastCommentary(array $data): void
    {
        $matchId = $data['matchId'];
        if (!isset($this->rooms[$matchId])) return;
        
        $commentary = json_encode([
            'type' => 'commentary',
            'matchId' => $matchId,
            'text' => $data['text'],
            'event' => $data['event'] // FOUR, SIX, WICKET, etc.
        ]);
        
        foreach ($this->rooms[$matchId] as $client) {
            $client->send($commentary);
        }
    }
    
    public function onClose(ConnectionInterface $conn): void
    {
        $this->clients->detach($conn);
        foreach ($this->rooms as $room) {
            if ($room->contains($conn)) {
                $room->detach($conn);
            }
        }
        echo "সংযোগ #{$conn->resourceId} বন্ধ\n";
    }
    
    public function onError(ConnectionInterface $conn, \Exception $e): void
    {
        echo "এরর: {$e->getMessage()}\n";
        $conn->close();
    }
}

// সার্ভার চালু
$server = IoServer::factory(
    new HttpServer(
        new WsServer(
            new CricketScoreServer()
        )
    ),
    8080
);

echo "ক্রিকেট স্কোর WebSocket সার্ভার চালু: ws://localhost:8080\n";
$server->run();
```

---

## 📡 Server-Sent Events (SSE)

SSE হলো সার্ভার থেকে ক্লায়েন্টে **একমুখী** (unidirectional) ডেটা পাঠানোর পদ্ধতি।

```
WebSocket (Full-Duplex):       SSE (Server → Client only):
────────────────────────       ──────────────────────────
Client ◄═════════► Server      Client ◄──────── Server
                               Client ──HTTP──► Server (শুধু প্রথমবার)

SSE ব্যবহার:
- লাইভ নোটিফিকেশন
- স্টক প্রাইস আপডেট
- নিউজ ফিড আপডেট
- সার্ভার লগ স্ট্রিমিং
```

### Node.js SSE Server

```javascript
// =============================================
// SSE Server (Node.js)
// বাস্তব উদাহরণ: Prothom Alo লাইভ নিউজ ফিড
// =============================================

const http = require('http');

const clients = new Set();

const server = http.createServer((req, res) => {
    if (req.url === '/events') {
        // SSE হেডার সেট করা
        res.writeHead(200, {
            'Content-Type': 'text/event-stream',
            'Cache-Control': 'no-cache',
            'Connection': 'keep-alive',
            'Access-Control-Allow-Origin': '*'
        });
        
        clients.add(res);
        console.log(`নতুন SSE ক্লায়েন্ট। মোট: ${clients.size}`);
        
        // Heartbeat (প্রতি ৩০ সেকেন্ডে)
        const heartbeat = setInterval(() => {
            res.write(': heartbeat\n\n');
        }, 30000);
        
        req.on('close', () => {
            clients.delete(res);
            clearInterval(heartbeat);
            console.log(`ক্লায়েন্ট বিচ্ছিন্ন। মোট: ${clients.size}`);
        });
    }
});

// নিউজ publish করা
function publishNews(news) {
    const data = JSON.stringify(news);
    
    for (const client of clients) {
        client.write(`event: news\n`);
        client.write(`data: ${data}\n`);
        client.write(`id: ${news.id}\n\n`);
    }
}

// উদাহরণ: প্রতি ১০ সেকেন্ডে ব্রেকিং নিউজ
setInterval(() => {
    publishNews({
        id: Date.now(),
        title: 'ব্রেকিং: বাংলাদেশ টেস্ট ম্যাচে জয়!',
        category: 'sports',
        timestamp: new Date().toISOString()
    });
}, 10000);

server.listen(3000, () => {
    console.log('SSE সার্ভার চালু: http://localhost:3000/events');
});
```

### SSE Client (Browser)

```javascript
// =============================================
// SSE Client (Browser)
// =============================================

const eventSource = new EventSource('https://api.prothomalo.com/events');

// নিউজ ইভেন্ট
eventSource.addEventListener('news', (event) => {
    const news = JSON.parse(event.data);
    console.log(`📰 ${news.title}`);
    
    // Auto-Reconnect built-in!
    // lastEventId ব্যবহার করে missed events পায়
});

eventSource.onerror = (error) => {
    console.log('SSE সংযোগ বিচ্ছিন্ন, auto-reconnect হবে...');
    // EventSource automatically reconnects!
};

// সংযোগ বন্ধ
// eventSource.close();
```

### PHP SSE Server

```php
<?php
// =============================================
// SSE Server (PHP)
// বাস্তব উদাহরণ: bKash ট্রানজ্যাকশন নোটিফিকেশন
// =============================================

header('Content-Type: text/event-stream');
header('Cache-Control: no-cache');
header('Connection: keep-alive');
header('Access-Control-Allow-Origin: *');

// Output buffering বন্ধ করা
ob_implicit_flush(true);
if (ob_get_level() > 0) ob_end_flush();

$lastId = isset($_SERVER['HTTP_LAST_EVENT_ID']) 
    ? (int) $_SERVER['HTTP_LAST_EVENT_ID'] 
    : 0;

while (true) {
    // ডাটাবেস থেকে নতুন ট্রানজ্যাকশন চেক
    $transactions = getNewTransactions($lastId);
    
    foreach ($transactions as $txn) {
        echo "event: transaction\n";
        echo "data: " . json_encode([
            'txnId' => $txn['id'],
            'amount' => $txn['amount'],
            'type' => $txn['type'],
            'message' => "{$txn['amount']} টাকা {$txn['type']} সফল"
        ]) . "\n";
        echo "id: {$txn['id']}\n\n";
        
        $lastId = $txn['id'];
    }
    
    // Heartbeat
    echo ": heartbeat\n\n";
    
    flush();
    sleep(2);
    
    if (connection_aborted()) break;
}

function getNewTransactions(int $lastId): array
{
    // ডাটাবেস query সিমুলেশন
    return []; 
}
```

---

## 📊 তুলনা

```
┌────────────────────┬──────────────┬──────────────┬──────────────┐
│     বৈশিষ্ট্য       │  WebSocket   │     SSE      │ Long Polling │
├────────────────────┼──────────────┼──────────────┼──────────────┤
│ Direction          │ Bidirectional│ Server→Client│ Simulated    │
│                    │ (Full-duplex)│ (One-way)    │ bidirectional│
├────────────────────┼──────────────┼──────────────┼──────────────┤
│ Protocol           │ ws:// wss:// │ HTTP         │ HTTP         │
├────────────────────┼──────────────┼──────────────┼──────────────┤
│ Auto Reconnect     │ ❌ নিজে      │ ✅ Built-in  │ ❌ নিজে      │
├────────────────────┼──────────────┼──────────────┼──────────────┤
│ Binary Data        │ ✅           │ ❌ (text)    │ ✅           │
├────────────────────┼──────────────┼──────────────┼──────────────┤
│ Max Connections    │ No limit*    │ 6 per domain │ 6 per domain │
│ (HTTP/1.1)         │              │ (browser)    │ (browser)    │
├────────────────────┼──────────────┼──────────────┼──────────────┤
│ Firewall Friendly  │ ⚠️ কিছু      │ ✅ HTTP       │ ✅ HTTP       │
│                    │ ব্লক করে     │ তাই সমস্যা নেই│ তাই সমস্যা নেই│
├────────────────────┼──────────────┼──────────────┼──────────────┤
│ Overhead           │ Low          │ Medium       │ High         │
├────────────────────┼──────────────┼──────────────┼──────────────┤
│ Use Case           │ চ্যাট, গেম,   │ নোটিফিকেশন,  │ লিগ্যাসি,     │
│                    │ লাইভ ট্র্যাক  │ নিউজ ফিড     │ ফলব্যাক       │
└────────────────────┴──────────────┴──────────────┴──────────────┘
```

```
কোন পদ্ধতি কখন?

"সার্ভার থেকে ক্লায়েন্টে আপডেট পাঠাতে হবে?"
         │
    ┌────▼────┐
    │ হ্যাঁ      │
    └────┬────┘
         │
"ক্লায়েন্টও সার্ভারে মেসেজ পাঠাবে?"
         │
    ┌────┴────┐
    │         │
   হ্যাঁ      না
    │         │
    ▼         ▼
 WebSocket   SSE
 (চ্যাট,      (নোটিফিকেশন,
  গেম,        লাইভ স্কোর,
  কোলাব)      নিউজ ফিড)
```

---

## ❌ Bad / ✅ Good Patterns

```
❌ ভুল: প্রতিটি ছোট আপডেটের জন্য HTTP polling
─────────────────────────────────────────────
setInterval(async () => {
    const res = await fetch('/api/notifications');
    // প্রতি ১ সেকেন্ডে HTTP request!
    // সার্ভারে অকারণ লোড, bandwidth waste
}, 1000);

✅ সঠিক: WebSocket বা SSE ব্যবহার করুন
─────────────────────────────────────────
const ws = new WebSocket('wss://api.example.com/ws');
ws.onmessage = (event) => {
    // সার্ভার যখন আপডেট আছে তখনই পাঠায়
    // কোনো unnecessary request নেই
};

──────────────────────────────────────────────

❌ ভুল: REST API তে WebSocket ব্যবহার করা
──────────────────────────────────────────
// CRUD operations এ WebSocket অপ্রয়োজনীয়
ws.send(JSON.stringify({ action: 'getUser', id: 123 }));

✅ সঠিক: CRUD operations এ HTTP ব্যবহার করুন
──────────────────────────────────────────
const user = await fetch('/api/users/123').then(r => r.json());

──────────────────────────────────────────────

❌ ভুল: Reconnection logic না রাখা
─────────────────────────────────
const ws = new WebSocket('wss://...');
// connection drop হলে... চুপচাপ বসে থাকবে!

✅ সঠিক: Exponential backoff সহ reconnection
──────────────────────────────────────────
function connect(attempt = 0) {
    const ws = new WebSocket('wss://...');
    ws.onclose = () => {
        const delay = Math.min(1000 * 2 ** attempt, 30000);
        setTimeout(() => connect(attempt + 1), delay);
    };
    ws.onopen = () => connect.attempt = 0;
}
```

---

## 🎯 কখন ব্যবহার করবেন / করবেন না

### WebSocket ব্যবহার করুন:

```
✅ রিয়েল-টাইম চ্যাট অ্যাপ (Pathao, Messenger)
✅ অনলাইন গেমিং (multiplayer)
✅ লাইভ কোলাবরেশন (Google Docs)
✅ লাইভ লোকেশন ট্র্যাকিং
✅ ট্রেডিং/স্টক প্ল্যাটফর্ম
✅ IoT রিয়েল-টাইম ড্যাশবোর্ড
```

### SSE ব্যবহার করুন:

```
✅ সার্ভার থেকে একমুখী আপডেট
✅ নোটিফিকেশন সিস্টেম
✅ লাইভ নিউজ/স্কোর ফিড
✅ সার্ভার লগ স্ট্রিমিং
✅ যেখানে HTTP proxy/firewall সমস্যা
```

### কোনোটিই ব্যবহার করবেন না:

```
❌ সাধারণ CRUD operations → HTTP REST ব্যবহার করুন
❌ কম ফ্রিকোয়েন্সি আপডেট (প্রতি ঘণ্টায়) → HTTP polling যথেষ্ট
❌ ফাইল আপলোড → HTTP multipart form ব্যবহার করুন
```

---

## 📊 সারসংক্ষেপ

```
┌───────────────────────────────────────────────────────┐
│             WebSocket সারসংক্ষেপ                       │
├───────────────────────────────────────────────────────┤
│                                                       │
│  WebSocket = ফোন কল 📞 (দুজনেই একসাথে কথা বলতে পারে)  │
│  SSE = রেডিও 📻 (শুধু শুনতে পারবেন)                    │
│  HTTP = চিঠি 📨 (প্রশ্ন পাঠাও, উত্তর পাও)              │
│                                                       │
│  মূল পয়েন্ট:                                           │
│  - WebSocket: Full-duplex, persistent connection      │
│  - HTTP Upgrade হ্যান্ডশেক দিয়ে শুরু হয়                │
│  - Socket.IO: WebSocket এর উপর সুবিধা যোগ করে        │
│  - SSE: সিম্পল সার্ভার → ক্লায়েন্ট push               │
│  - সবসময় reconnection logic রাখুন                    │
│  - Heartbeat/Ping-Pong দিয়ে connection চেক করুন      │
│                                                       │
└───────────────────────────────────────────────────────┘
```

---

> 💡 **পরবর্তী**: [DNS](./dns.md) — ডোমেইন নেম সিস্টেম কীভাবে কাজ করে
