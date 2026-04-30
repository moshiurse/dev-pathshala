# 🔄 TCP ও UDP (Transport Layer Protocols)

> TCP এবং UDP হলো ইন্টারনেটের ট্রান্সপোর্ট লেয়ারের দুটি মূল প্রোটোকল।
> TCP নির্ভরযোগ্যতা দেয়, UDP গতি দেয়। কখন কোনটি ব্যবহার করবেন সেটি জানা অত্যন্ত গুরুত্বপূর্ণ।

---

## 📖 সূচিপত্র

- [TCP বিস্তারিত](#-tcp-transmission-control-protocol)
- [UDP বিস্তারিত](#-udp-user-datagram-protocol)
- [Three-Way Handshake](#-three-way-handshake)
- [Flow Control](#-flow-control)
- [Congestion Control](#-congestion-control)
- [TCP vs UDP তুলনা](#-tcp-vs-udp-তুলনা)
- [সকেট প্রোগ্রামিং](#-সকেট-প্রোগ্রামিং)
- [কখন ব্যবহার করবেন](#-কখন-ব্যবহার-করবেন--করবেন-না)

---

## 📌 TCP (Transmission Control Protocol)

TCP হলো একটি **connection-oriented**, **reliable**, **ordered** প্রোটোকল। এটি নিশ্চিত করে যে প্রতিটি বাইট সঠিক ক্রমে গন্তব্যে পৌঁছেছে।

```
┌─────────────────────────────────────────────────────────┐
│                    TCP এর বৈশিষ্ট্য                       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ✅ Connection-oriented (আগে সংযোগ স্থাপন)               │
│  ✅ Reliable delivery (প্রতিটি প্যাকেট পৌঁছানো নিশ্চিত)   │
│  ✅ Ordered (সঠিক ক্রমে ডেলিভারি)                         │
│  ✅ Error checking (চেকসাম)                               │
│  ✅ Flow control (প্রাপকের ক্ষমতা অনুযায়ী পাঠানো)         │
│  ✅ Congestion control (নেটওয়ার্ক জ্যাম এড়ানো)            │
│  ❌ Slower than UDP (অতিরিক্ত overhead)                   │
│  ❌ More bandwidth (হেডার বড়: ২০-৬০ বাইট)                │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### TCP Header Structure

```
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |          Source Port          |       Destination Port        |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                        Sequence Number                       |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                    Acknowledgment Number                     |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |  Data |       |C|E|U|A|P|R|S|F|                              |
 | Offset| Rsrvd |W|C|R|C|S|S|Y|I|            Window            |
 |       |       |R|E|G|K|H|T|N|N|                              |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |           Checksum            |         Urgent Pointer        |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                    Options (if any)                          |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

 মিনিমাম হেডার সাইজ: ২০ বাইট
 ফ্ল্যাগসমূহ:
   SYN - সংযোগ শুরু
   ACK - প্রাপ্তি স্বীকার
   FIN - সংযোগ শেষ
   RST - সংযোগ রিসেট
   PSH - ডেটা তাৎক্ষণিক ডেলিভারি
   URG - জরুরি ডেটা
```

---

## 📌 UDP (User Datagram Protocol)

UDP হলো একটি **connectionless**, **unreliable** (গ্যারান্টি নেই), **fast** প্রোটোকল। এটি ডেটা পাঠায় এবং পৌঁছাল কিনা যাচাই করে না।

```
┌─────────────────────────────────────────────────────────┐
│                    UDP এর বৈশিষ্ট্য                       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ✅ Connectionless (সংযোগ স্থাপনের দরকার নেই)            │
│  ✅ Fast (কম overhead)                                   │
│  ✅ Lightweight (ছোট হেডার: মাত্র ৮ বাইট)                │
│  ✅ Broadcast/Multicast সাপোর্ট                          │
│  ✅ No head-of-line blocking                             │
│  ❌ Unreliable (প্যাকেট হারাতে পারে)                     │
│  ❌ Unordered (ভুল ক্রমে আসতে পারে)                      │
│  ❌ No flow control                                      │
│  ❌ No congestion control                                │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### UDP Header Structure

```
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |          Source Port          |       Destination Port        |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |            Length             |           Checksum            |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

 হেডার সাইজ: মাত্র ৮ বাইট! (TCP এর ২০+ বাইটের তুলনায়)
```

---

## 🔗 Three-Way Handshake

TCP সংযোগ স্থাপনের জন্য তিন ধাপের হ্যান্ডশেক ব্যবহার করে। এটাকে "SYN-SYN/ACK-ACK" বলে।

```
  Client (bKash App)              Server (bKash API)
       │                                │
       │  ──── SYN (seq=100) ──────►    │  Step 1: ক্লায়েন্ট বলে
       │       "আমি কানেক্ট করতে চাই"    │         "আস্সালামু আলাইকুম"
       │                                │
       │  ◄── SYN-ACK (seq=300,        │  Step 2: সার্ভার বলে
       │       ack=101) ──              │         "ওয়ালাইকুম আসসালাম,
       │       "ঠিক আছে, আমিও রেডি"     │          আমিও রেডি"
       │                                │
       │  ──── ACK (seq=101,           │  Step 3: ক্লায়েন্ট বলে
       │       ack=301) ──────►         │         "চলো শুরু করি!"
       │                                │
       │  ═══ Connection Established ═══│
       │                                │
       │  ◄────── Data Transfer ──────► │
       │      (bKash ট্রানজ্যাকশন ডেটা)   │
       │                                │
```

### বাস্তব উদাহরণ (bKash লেনদেন)

```
আপনার ফোন (bKash App)                bKash সার্ভার
        │                                   │
  Step 1│ SYN: "আমি ৫০০ টাকা পাঠাতে চাই"    │
        │──────────────────────────────────►│
        │                                   │
  Step 2│ SYN-ACK: "ঠিক আছে, PIN দাও"       │
        │◄──────────────────────────────────│
        │                                   │
  Step 3│ ACK: "PIN: ****"                   │
        │──────────────────────────────────►│
        │                                   │
        │══ সংযোগ স্থাপিত, ডেটা আদান-প্রদান ══│
        │                                   │
```

### Four-Way Termination (সংযোগ বন্ধ)

```
  Client                            Server
    │                                  │
    │  ──── FIN ─────────────────►    │  "আমার কাজ শেষ"
    │                                  │
    │  ◄──── ACK ────────────────     │  "বুঝলাম"
    │                                  │
    │  ◄──── FIN ────────────────     │  "আমারও কাজ শেষ"
    │                                  │
    │  ──── ACK ─────────────────►    │  "ঠিক আছে, বিদায়"
    │                                  │
    │     [TIME_WAIT: 2*MSL]          │
    │                                  │
```

---

## 📊 Flow Control

Flow Control নিশ্চিত করে যে প্রেরক প্রাপকের ক্ষমতার বেশি ডেটা পাঠায় না। TCP **Sliding Window** মেকানিজম ব্যবহার করে।

```
প্রেরকের বাফার (Sliding Window)
┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
│  1  │  2  │  3  │  4  │  5  │  6  │  7  │  8  │  9  │ 10  │
└─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
 ▲ACK         ▲পাঠানো              ▲Window Edge
 প্রাপ্ত        কিন্তু ACK            পাঠানো
              পাওয়া যায়নি           যাবে না

Window Size = 5 (একসাথে ৫টি সেগমেন্ট পাঠাতে পারবে)

প্রাপকের বাফার (Receive Window)
┌─────┬─────┬─────┬─────┬─────┐
│     │     │     │     │     │  ← খালি জায়গা = Window Size
└─────┴─────┴─────┴─────┴─────┘

সময়ের সাথে Window কীভাবে চলে:
──────────────────────────────────────────

t=0: Window=[1,2,3,4,5]     পাঠানো: 1,2,3,4,5
t=1: ACK 1,2 পাওয়া গেল
     Window=[3,4,5,6,7]     পাঠানো: 6,7
t=2: ACK 3,4,5 পাওয়া গেল
     Window=[6,7,8,9,10]    পাঠানো: 8,9,10
```

### বাস্তব উদাহরণ

```
ধরুন Daraz সার্ভার আপনার ব্রাউজারে একটি বড় ইমেজ পাঠাচ্ছে:

Daraz Server                              আপনার Browser
(প্রেরক)                                    (প্রাপক)
    │                                          │
    │  ── Segment 1 (1KB) ──────────────►     │
    │  ── Segment 2 (1KB) ──────────────►     │  Window=3
    │  ── Segment 3 (1KB) ──────────────►     │
    │                                          │
    │  ◄── ACK 1,2,3 + Window=5 ──────────   │  বাফার খালি হয়েছে
    │                                          │
    │  ── Segment 4 ────────────────────►     │
    │  ── Segment 5 ────────────────────►     │  Window=5
    │  ── Segment 6 ────────────────────►     │
    │  ── Segment 7 ────────────────────►     │
    │  ── Segment 8 ────────────────────►     │
    │                                          │
    │  ◄── ACK 4-8 + Window=2 ────────────   │  বাফার প্রায় ভর্তি!
    │                                          │
    │  ── Segment 9 ────────────────────►     │  Window=2
    │  ── Segment 10 ───────────────────►     │
    │      (আর পাঠাবে না, Window শেষ)          │
```

---

## 📊 Congestion Control

Congestion Control নেটওয়ার্কে অতিরিক্ত ট্রাফিক থেকে রক্ষা করে। TCP বেশ কিছু অ্যালগরিদম ব্যবহার করে:

```
Congestion Window (cwnd) এর পরিবর্তন:

                    ┃ Congestion
cwnd                ┃ Detected!
  ▲                 ┃
  │            ╱╲   ┃
  │           ╱  ╲  ┃    Congestion
  │          ╱    ╲ ┃    Avoidance
  │         ╱      ╲┃    (Linear বৃদ্ধি)
  │        ╱        ┃╲
  │       ╱         ┃ ╲
  │      ╱  Slow    ┃  ╲_____ ssthresh = cwnd/2
  │     ╱   Start   ┃        ╱
  │    ╱  (Exp.     ┃       ╱ Slow Start
  │   ╱   বৃদ্ধি)    ┃      ╱  আবার শুরু
  │  ╱              ┃     ╱
  │ ╱               ┃    ╱
  │╱                ┃   ╱
  └─────────────────┃──────────────────► সময়
                    ┃

পর্যায়সমূহ:
1. Slow Start: cwnd দ্বিগুণ হয় প্রতি RTT এ (1→2→4→8→16...)
2. Congestion Avoidance: cwnd = cwnd + 1 প্রতি RTT এ
3. Congestion Detection: প্যাকেট লস হলে cwnd কমে যায়
```

### অ্যালগরিদম তুলনা

```
┌──────────────────┬───────────────────────────────────────┐
│   অ্যালগরিদম      │              বৈশিষ্ট্য                 │
├──────────────────┼───────────────────────────────────────┤
│ TCP Tahoe        │ লস হলে cwnd=1, Slow Start থেকে শুরু  │
│ TCP Reno         │ ৩টি dup ACK → cwnd অর্ধেক (Fast      │
│                  │ Recovery)                             │
│ TCP Cubic        │ Linux ডিফল্ট, cubic function ব্যবহার  │
│ TCP BBR          │ Google তৈরি, bandwidth estimation    │
│ (Bottleneck      │ ভিত্তিক, লস-ভিত্তিক নয়               │
│  Bandwidth &     │                                       │
│  RTT)            │                                       │
└──────────────────┴───────────────────────────────────────┘
```

---

## 📊 TCP vs UDP তুলনা

```
┌────────────────────┬──────────────────┬──────────────────┐
│      বৈশিষ্ট্য      │       TCP        │       UDP        │
├────────────────────┼──────────────────┼──────────────────┤
│ সংযোগ             │ Connection-      │ Connectionless   │
│                    │ oriented         │                  │
├────────────────────┼──────────────────┼──────────────────┤
│ নির্ভরযোগ্যতা       │ Reliable         │ Unreliable       │
│                    │ (ACK, Retrans.)  │ (Best effort)    │
├────────────────────┼──────────────────┼──────────────────┤
│ ক্রম               │ Ordered          │ Unordered        │
├────────────────────┼──────────────────┼──────────────────┤
│ গতি               │ ধীর (overhead)    │ দ্রুত (কম overhead)│
├────────────────────┼──────────────────┼──────────────────┤
│ হেডার সাইজ         │ ২০-৬০ bytes      │ ৮ bytes          │
├────────────────────┼──────────────────┼──────────────────┤
│ Flow Control      │ ✅ আছে           │ ❌ নেই            │
├────────────────────┼──────────────────┼──────────────────┤
│ Congestion Control│ ✅ আছে           │ ❌ নেই            │
├────────────────────┼──────────────────┼──────────────────┤
│ Broadcast         │ ❌ নেই            │ ✅ আছে           │
├────────────────────┼──────────────────┼──────────────────┤
│ Error Recovery    │ ✅ Retransmit     │ ❌ Drop           │
├────────────────────┼──────────────────┼──────────────────┤
│ Use Cases         │ HTTP, FTP,       │ DNS, VoIP,       │
│                    │ SMTP, SSH,       │ Video Stream,    │
│                    │ Database         │ Gaming, DHCP     │
└────────────────────┴──────────────────┴──────────────────┘
```

### বাংলাদেশী সেবা অনুযায়ী ব্যবহার

```
┌──────────────────┬─────────┬────────────────────────────┐
│    সার্ভিস         │ প্রোটোকল │         কারণ               │
├──────────────────┼─────────┼────────────────────────────┤
│ bKash লেনদেন      │  TCP    │ প্রতিটি টাকার তথ্য নিশ্চিত    │
│                  │         │ পৌঁছাতে হবে                  │
├──────────────────┼─────────┼────────────────────────────┤
│ Daraz ওয়েবসাইট   │  TCP    │ HTML/CSS/JS পুরোটা দরকার   │
├──────────────────┼─────────┼────────────────────────────┤
│ GP Video Call    │  UDP    │ সামান্য ফ্রেম হারালে চলে,    │
│                  │         │ দ্রুততা জরুরি                │
├──────────────────┼─────────┼────────────────────────────┤
│ Pathao Live      │  UDP    │ রাইডারের অবস্থান রিয়েল-টাইম │
│ Tracking         │  +TCP   │ ম্যাপে দেখাতে হবে            │
├──────────────────┼─────────┼────────────────────────────┤
│ PUBG Mobile      │  UDP    │ গেমিং-এ ল্যাটেন্সি কম দরকার │
│ (BD Server)      │         │                            │
├──────────────────┼─────────┼────────────────────────────┤
│ DNS Query        │  UDP    │ ছোট query, দ্রুত response   │
│ (bd ISPs)        │         │ দরকার                      │
└──────────────────┴─────────┴────────────────────────────┘
```

---

## 💻 সকেট প্রোগ্রামিং

### PHP TCP সকেট

```php
<?php
// =============================================
// TCP Server (PHP)
// বাস্তব উদাহরণ: সিম্পল চ্যাট সার্ভার
// =============================================

$host = '127.0.0.1';
$port = 8080;

// সকেট তৈরি
$socket = socket_create(AF_INET, SOCK_STREAM, SOL_TCP);
if ($socket === false) {
    die("সকেট তৈরি ব্যর্থ: " . socket_strerror(socket_last_error()));
}

// বাইন্ড করা (IP ও Port এ সংযুক্ত)
socket_bind($socket, $host, $port);

// লিসেন শুরু (সর্বোচ্চ ৫টি সংযোগ Queue তে)
socket_listen($socket, 5);

echo "TCP সার্ভার চালু: $host:$port\n";

while (true) {
    // ক্লায়েন্টের সংযোগ গ্রহণ (Three-way handshake এখানে হয়)
    $client = socket_accept($socket);
    
    // ক্লায়েন্ট থেকে ডেটা পড়া
    $message = socket_read($client, 1024);
    echo "প্রাপ্ত: $message\n";
    
    // রেসপন্স পাঠানো
    $response = "সার্ভার বলছে: আপনার বার্তা পেয়েছি!";
    socket_write($client, $response, strlen($response));
    
    // সংযোগ বন্ধ
    socket_close($client);
}

socket_close($socket);
```

```php
<?php
// =============================================
// TCP Client (PHP)
// =============================================

$host = '127.0.0.1';
$port = 8080;

$socket = socket_create(AF_INET, SOCK_STREAM, SOL_TCP);

// সার্ভারে সংযোগ (Three-way Handshake)
socket_connect($socket, $host, $port);

// বার্তা পাঠানো
$message = "আস্সালামু আলাইকুম, সার্ভার!";
socket_write($socket, $message, strlen($message));

// রেসপন্স পড়া
$response = socket_read($socket, 1024);
echo "সার্ভার বলেছে: $response\n";

socket_close($socket);
```

### PHP UDP সকেট

```php
<?php
// =============================================
// UDP Server (PHP)
// বাস্তব উদাহরণ: লোকেশন ট্র্যাকিং (Pathao-style)
// =============================================

$socket = socket_create(AF_INET, SOCK_DGRAM, SOL_UDP);
socket_bind($socket, '127.0.0.1', 9090);

echo "UDP সার্ভার চালু: 127.0.0.1:9090\n";
echo "রাইডারের লোকেশন আপডেট অপেক্ষায়...\n";

while (true) {
    // কোনো সংযোগ স্থাপন ছাড়াই ডেটা গ্রহণ
    socket_recvfrom($socket, $buf, 1024, 0, $remoteIP, $remotePort);
    
    $location = json_decode($buf, true);
    echo "রাইডার #{$location['rider_id']}: ";
    echo "Lat={$location['lat']}, Lng={$location['lng']}\n";
    
    // ACK পাঠানো (অ্যাপ্লিকেশন লেভেলে, UDP তে built-in নেই)
    $ack = json_encode(['status' => 'received']);
    socket_sendto($socket, $ack, strlen($ack), 0, $remoteIP, $remotePort);
}
```

```php
<?php
// =============================================
// UDP Client (PHP) - রাইডার অ্যাপ সিমুলেশন
// =============================================

$socket = socket_create(AF_INET, SOCK_DGRAM, SOL_UDP);

// প্রতি ৩ সেকেন্ডে লোকেশন পাঠানো
while (true) {
    $location = json_encode([
        'rider_id' => 42,
        'lat' => 23.8103 + (rand(-100, 100) / 10000),
        'lng' => 90.4125 + (rand(-100, 100) / 10000),
        'timestamp' => time()
    ]);
    
    // কোনো সংযোগ ছাড়াই ডেটা পাঠানো (Fire and forget!)
    socket_sendto($socket, $location, strlen($location), 0, '127.0.0.1', 9090);
    
    echo "লোকেশন পাঠানো হয়েছে\n";
    sleep(3);
}
```

### Node.js TCP সকেট

```javascript
// =============================================
// TCP Server (Node.js)
// বাস্তব উদাহরণ: bKash-style Transaction Server
// =============================================

const net = require('net');

const server = net.createServer((socket) => {
    console.log(`ক্লায়েন্ট সংযুক্ত: ${socket.remoteAddress}:${socket.remotePort}`);
    
    socket.on('data', (data) => {
        const request = JSON.parse(data.toString());
        console.log('ট্রানজ্যাকশন রিকোয়েস্ট:', request);
        
        // ট্রানজ্যাকশন প্রসেস
        const response = {
            status: 'success',
            txnId: 'TXN' + Date.now(),
            amount: request.amount,
            message: `${request.amount} টাকা সফলভাবে পাঠানো হয়েছে`
        };
        
        // TCP নিশ্চিত করে এই ডেটা পুরোটা পৌঁছাবে
        socket.write(JSON.stringify(response));
    });
    
    socket.on('end', () => {
        console.log('ক্লায়েন্ট সংযোগ বিচ্ছিন্ন');
    });
    
    socket.on('error', (err) => {
        console.error('সকেট এরর:', err.message);
    });
});

server.listen(8080, '127.0.0.1', () => {
    console.log('TCP সার্ভার চালু: 127.0.0.1:8080');
});
```

```javascript
// =============================================
// TCP Client (Node.js)
// =============================================

const net = require('net');

const client = new net.Socket();

client.connect(8080, '127.0.0.1', () => {
    console.log('সার্ভারে সংযুক্ত');
    
    const transaction = {
        sender: '01712345678',
        receiver: '01898765432',
        amount: 500,
        pin: '****'
    };
    
    client.write(JSON.stringify(transaction));
});

client.on('data', (data) => {
    const response = JSON.parse(data.toString());
    console.log('রেসপন্স:', response);
    client.destroy();
});

client.on('close', () => {
    console.log('সংযোগ বন্ধ');
});
```

### Node.js UDP সকেট

```javascript
// =============================================
// UDP Server (Node.js)
// বাস্তব উদাহরণ: গেম সার্ভার (Low Latency)
// =============================================

const dgram = require('dgram');
const server = dgram.createSocket('udp4');

const players = new Map();

server.on('message', (msg, rinfo) => {
    const data = JSON.parse(msg.toString());
    const playerId = `${rinfo.address}:${rinfo.port}`;
    
    // প্লেয়ার পজিশন আপডেট (কোনো handshake নেই!)
    players.set(playerId, {
        x: data.x,
        y: data.y,
        health: data.health,
        lastUpdate: Date.now()
    });
    
    // সব প্লেয়ারের পজিশন ব্রডকাস্ট
    const gameState = JSON.stringify(Object.fromEntries(players));
    
    for (const [id, _] of players) {
        const [ip, port] = id.split(':');
        server.send(gameState, parseInt(port), ip);
    }
});

server.on('listening', () => {
    const addr = server.address();
    console.log(`গেম সার্ভার চালু: ${addr.address}:${addr.port}`);
});

server.bind(9090);
```

```javascript
// =============================================
// UDP Client (Node.js) - গেম ক্লায়েন্ট
// =============================================

const dgram = require('dgram');
const client = dgram.createSocket('udp4');

// প্রতি 50ms এ পজিশন পাঠানো (20 FPS)
setInterval(() => {
    const position = {
        x: Math.random() * 1000,
        y: Math.random() * 1000,
        health: 100
    };
    
    const message = Buffer.from(JSON.stringify(position));
    
    // UDP: পাঠাও, পৌঁছালো কিনা দেখো না!
    client.send(message, 9090, '127.0.0.1');
}, 50);

client.on('message', (msg) => {
    const gameState = JSON.parse(msg.toString());
    // গেম স্টেট রেন্ডার করো
});
```

---

## ❌ Bad / ✅ Good Patterns

### ❌ ভুল: আর্থিক লেনদেনে UDP ব্যবহার

```javascript
// ❌ BAD: টাকা পাঠানোর জন্য UDP ব্যবহার করা
const dgram = require('dgram');
const client = dgram.createSocket('udp4');

const transaction = {
    from: '01712345678',
    to: '01898765432',
    amount: 50000  // ৫০,০০০ টাকা!
};

// UDP তে এই প্যাকেট হারিয়ে গেলে?
// ৫০,০০০ টাকা কাটা হবে কিন্তু পৌঁছাবে না!
client.send(JSON.stringify(transaction), 9090, 'bkash-server.com');
```

### ✅ সঠিক: আর্থিক লেনদেনে TCP ব্যবহার

```javascript
// ✅ GOOD: TCP নিশ্চিত করে প্রতিটি বাইট পৌঁছাবে
const net = require('net');
const tls = require('tls');

// TLS over TCP — সিকিউর ও রিলায়েবল
const client = tls.connect(443, 'api.bkash.com', {
    rejectUnauthorized: true
}, () => {
    const transaction = {
        from: '01712345678',
        to: '01898765432',
        amount: 50000,
        idempotencyKey: 'unique-txn-123' // ডুপ্লিকেট প্রতিরোধ
    };
    
    client.write(JSON.stringify(transaction));
});

client.on('data', (data) => {
    const response = JSON.parse(data.toString());
    if (response.status === 'success') {
        console.log('লেনদেন সফল:', response.txnId);
    }
});
```

### ❌ ভুল: ভিডিও স্ট্রিমিং এ TCP ব্যবহার

```javascript
// ❌ BAD: লাইভ ভিডিও তে TCP ব্যবহার
// সমস্যা: একটি প্যাকেট হারালে পুরো স্ট্রিম থেমে যাবে (HOL blocking)
const net = require('net');
const client = new net.Socket();

client.connect(8080, 'live-stream.com', () => {
    // TCP তে প্যাকেট ১০ হারালে, ১১, ১২, ১৩... সব অপেক্ষা করবে
    // ইউজার buffering দেখবে!
});
```

### ✅ সঠিক: ভিডিও স্ট্রিমিং এ UDP ব্যবহার

```javascript
// ✅ GOOD: UDP ব্যবহার — হারানো ফ্রেম skip করে পরেরটা দেখানো
const dgram = require('dgram');
const client = dgram.createSocket('udp4');

client.on('message', (frame) => {
    // প্যাকেট ১০ হারালেও ১১, ১২, ১৩ সাথে সাথে দেখাবে
    // ইউজার সামান্য glitch দেখবে, কিন্তু buffering হবে না!
    renderVideoFrame(frame);
});
```

---

## 🎯 কখন ব্যবহার করবেন / করবেন না

### TCP ব্যবহার করুন যখন:

```
✅ ওয়েব ব্রাউজিং (HTTP/HTTPS)
   → পুরো পেজ লোড হওয়া দরকার

✅ ইমেইল (SMTP, IMAP, POP3)
   → ইমেইলের প্রতিটি অক্ষর দরকার

✅ ফাইল ট্রান্সফার (FTP, SFTP)
   → ফাইল corrupt হলে চলবে না

✅ ডাটাবেস কানেকশন (MySQL, PostgreSQL)
   → SQL query ও result পুরোটা দরকার

✅ আর্থিক লেনদেন (bKash, Nagad)
   → একটি বাইট হারালেও বিপদ

✅ SSH / Remote Access
   → কমান্ড সঠিকভাবে পৌঁছাতে হবে
```

### UDP ব্যবহার করুন যখন:

```
✅ ভিডিও/অডিও স্ট্রিমিং
   → সামান্য quality loss acceptable

✅ অনলাইন গেমিং
   → ল্যাটেন্সি সর্বনিম্ন রাখতে হবে

✅ VoIP (ভয়েস কল)
   → দ্রুত ডেলিভারি জরুরি

✅ DNS Queries
   → ছোট ডেটা, দ্রুত উত্তর দরকার

✅ IoT সেন্সর ডেটা
   → প্রচুর ডেটা, কিছু হারালে সমস্যা নেই

✅ Live Location (Pathao)
   → পুরানো location এর চেয়ে নতুনটা জরুরি
```

---

## 📊 TCP Connection States

```
                              ┌───────────┐
                  Passive Open│  LISTEN   │
                 ┌────────────┤           │
                 │            └─────┬─────┘
                 │              SYN │ rcvd
                 │            ┌─────▼─────┐
                 │            │ SYN_RCVD  │
                 │            └─────┬─────┘
                 │              ACK │ rcvd
    ┌────────┐   │            ┌─────▼─────┐
    │ CLOSED │   │            │ESTABLISHED│◄── ডেটা আদান-প্রদান
    └────┬───┘   │            └─────┬─────┘
     SYN │ sent  │              FIN │ rcvd
    ┌────▼───┐   │            ┌─────▼─────┐
    │SYN_SENT│   │            │ CLOSE_WAIT│
    └────┬───┘   │            └─────┬─────┘
   SYN+ACK│rcvd  │              FIN │ sent
    ┌────▼───┐   │            ┌─────▼─────┐
    │ESTABLISHED│ │            │ LAST_ACK  │
    └────┬───┘   │            └─────┬─────┘
     FIN │ sent  │              ACK │ rcvd
    ┌────▼────┐  │            ┌─────▼─────┐
    │FIN_WAIT1│  │            │  CLOSED   │
    └────┬────┘  │            └───────────┘
     ACK │ rcvd  │
    ┌────▼────┐  │
    │FIN_WAIT2│  │
    └────┬────┘  │
     FIN │ rcvd  │
    ┌────▼────┐  │
    │TIME_WAIT│──┘ (2*MSL timeout)
    └────┬────┘
         │
    ┌────▼────┐
    │ CLOSED  │
    └─────────┘
```

---

## 📊 সারসংক্ষেপ

```
┌───────────────────────────────────────────────────────┐
│                TCP vs UDP সারসংক্ষেপ                   │
├───────────────────────────────────────────────────────┤
│                                                       │
│  TCP = রেজিস্ট্রি ডাক 📬                               │
│  - সংযোগ স্থাপন → ডেটা আদান-প্রদান → সংযোগ বন্ধ       │
│  - প্রতিটি প্যাকেট পৌঁছানো নিশ্চিত                       │
│  - ধীর কিন্তু নির্ভরযোগ্য                                │
│                                                       │
│  UDP = সাধারণ চিঠি 📨                                   │
│  - শুধু পাঠাও, পৌঁছালো কিনা দেখো না                     │
│  - দ্রুত কিন্তু অনির্ভরযোগ্য                              │
│  - রিয়েল-টাইম অ্যাপে উত্তম                              │
│                                                       │
│  মূল সিদ্ধান্ত: "ডেটা হারানো কি গ্রহণযোগ্য?"              │
│  - না → TCP ব্যবহার করুন                                │
│  - হ্যাঁ → UDP ব্যবহার করুন                               │
│                                                       │
└───────────────────────────────────────────────────────┘
```

---

> 💡 **পরবর্তী**: [HTTP প্রোটোকল](./http-protocol.md) — HTTP/1.1, HTTP/2, HTTP/3 গভীর আলোচনা
