# 📘 Kafka সিকিউরিটি (Security)

## 📑 সূচিপত্র
- [📖 Kafka সিকিউরিটি কেন গুরুত্বপূর্ণ?](#-kafka-সিকিউরিটি-কেন-গুরুত্বপূর্ণ)
- [🔐 অথেনটিকেশন (Authentication)](#-অথেনটিকেশন-authentication)
- [🔒 অথরাইজেশন (ACLs)](#-অথরাইজেশন-acls)
- [🔑 এনক্রিপশন](#-এনক্রিপশন)
- [🏢 মাল্টি-টেনান্সি](#-মাল্টি-টেনান্সি)
- [📊 সিকিউরিটি আর্কিটেকচার ডায়াগ্রাম](#-সিকিউরিটি-আর্কিটেকচার-ডায়াগ্রাম)
- [💻 কনফিগারেশন উদাহরণ](#-কনফিগারেশন-উদাহরণ)
- [📋 সিকিউরিটি চেকলিস্ট](#-সিকিউরিটি-চেকলিস্ট)
- [🎯 বেস্ট প্র্যাকটিস](#-বেস্ট-প্র্যাকটিস)
- [🔗 পরবর্তী টপিক](#-পরবর্তী-টপিক)

---

## 📖 Kafka সিকিউরিটি কেন গুরুত্বপূর্ণ?

### 🏠 বাংলা উপমা

> ধরুন আপনার বাড়িতে একটি সিন্দুক (vault) আছে যেখানে টাকা-পয়সা, জমির দলিল, গহনা রাখা হয়।
> এই সিন্দুকে যদি কোনো তালাই না থাকে — যে কেউ এসে খুলে নিতে পারবে।
> Kafka ডিফল্টভাবে ঠিক এরকম — **কোনো সিকিউরিটি নেই!**
>
> সিকিউরিটি যোগ করা মানে সিন্দুকে একাধিক তালা লাগানো:
> 🔐 প্রথম তালা = **অথেনটিকেশন** (কে আসছে?)
> 🔒 দ্বিতীয় তালা = **অথরাইজেশন** (কী করতে পারবে?)
> 🔑 তৃতীয় তালা = **এনক্রিপশন** (ডেটা পথে নিরাপদ থাকবে)

### ⚠️ Default Kafka-তে কোনো সিকিউরিটি নেই

ডিফল্ট Kafka ইনস্টলেশনে:
- কোনো **authentication** নেই — যে কেউ connect করতে পারে
- কোনো **authorization** নেই — যে কেউ যেকোনো topic read/write করতে পারে
- কোনো **encryption** নেই — ডেটা plaintext এ যায়

এটা development এনভায়রনমেন্টে ঠিক আছে, কিন্তু **production এ মারাত্মক ঝুঁকি!**

### 🇧🇩 বাংলাদেশ কনটেক্সটে কেন গুরুত্বপূর্ণ?

**bKash/Nagad ট্রানজেকশন ডেটা:**
- প্রতিদিন কোটি কোটি টাকার লেনদেন হয়
- ইউজারের ফোন নম্বর, ব্যালেন্স, ট্রানজেকশন হিস্ট্রি — সব সংবেদনশীল
- Bangladesh Bank এর ডেটা প্রোটেকশন নিয়ম মানতে হবে

**Daraz অর্ডার সিস্টেম:**
- কাস্টমারের ঠিকানা, পেমেন্ট তথ্য
- মার্চেন্টদের ব্যবসায়িক ডেটা

**Pathao রাইড ডেটা:**
- রিয়েল-টাইম লোকেশন ট্র্যাকিং
- রাইডার ও যাত্রীর ব্যক্তিগত তথ্য

**Grameenphone ডেটা:**
- কোটি কোটি গ্রাহকের কল ও ডেটা ব্যবহার তথ্য
- BTRC রেগুলেশন অনুযায়ী ডেটা সুরক্ষা বাধ্যতামূলক

---

## 🔐 অথেনটিকেশন (Authentication)

অথেনটিকেশন মানে **"তুমি কে?"** — এই প্রশ্নের উত্তর নিশ্চিত করা।

Kafka চারটি প্রধান অথেনটিকেশন মেকানিজম সাপোর্ট করে:

```
┌──────────────────────────────────────────────────────┐
│            Kafka Authentication Methods              │
├──────────────┬───────────────┬────────────┬──────────┤
│  SASL/PLAIN  │  SASL/SCRAM   │  SSL/TLS   │  OAuth   │
│  (সহজ)       │  (ভালো)       │ (শক্তিশালী)│  (আধুনিক)│
│  ⭐⭐         │  ⭐⭐⭐        │  ⭐⭐⭐⭐    │  ⭐⭐⭐⭐  │
└──────────────┴───────────────┴────────────┴──────────┘
```

---

### 🅰️ SASL/PLAIN

**SASL/PLAIN** হলো সবচেয়ে সহজ অথেনটিকেশন — Username ও Password দিয়ে লগইন।

**কিভাবে কাজ করে:**
1. Client username ও password পাঠায়
2. Broker যাচাই করে
3. মিললে connection অনুমোদন হয়

**Broker কনফিগারেশন (server.properties):**
```properties
# SASL/PLAIN সক্রিয় করা
listeners=SASL_PLAINTEXT://0.0.0.0:9092
security.inter.broker.protocol=SASL_PLAINTEXT
sasl.mechanism.inter.broker.protocol=PLAIN
sasl.enabled.mechanisms=PLAIN

# JAAS কনফিগারেশন
listener.name.sasl_plaintext.plain.sasl.jaas.config=\
  org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="admin" \
  password="admin-secret" \
  user_admin="admin-secret" \
  user_orderservice="order-pass-123" \
  user_analytics="analytics-pass-456";
```

**Client কনফিগারেশন:**
```properties
security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="orderservice" \
  password="order-pass-123";
```

#### ❌ খারাপ: SASL/PLAIN বিনা TLS

```
Client ──── "user:password" (plaintext!) ────→ Broker
                    ↑
            হ্যাকার এখানে পাসওয়ার্ড দেখতে পারে! 😱
```

যদি TLS ছাড়া SASL/PLAIN ব্যবহার করেন, পাসওয়ার্ড **নেটওয়ার্কে plaintext** এ যায়।
যে কেউ packet sniff করে পাসওয়ার্ড চুরি করতে পারবে।

#### ✅ ভালো: SASL/PLAIN সাথে TLS

```
Client ──── TLS tunnel (encrypted) ────→ Broker
              "user:password" (ভিতরে নিরাপদ)
```

```properties
# TLS দিয়ে SASL/PLAIN — সঠিক পদ্ধতি
listeners=SASL_SSL://0.0.0.0:9093
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
ssl.keystore.location=/var/kafka/ssl/kafka.keystore.jks
ssl.keystore.password=keystorepass
ssl.truststore.location=/var/kafka/ssl/kafka.truststore.jks
ssl.truststore.password=truststorepass
```

---

### 🅱️ SASL/SCRAM (SHA-256/512)

**SCRAM** = Salted Challenge Response Authentication Mechanism

PLAIN এর চেয়ে অনেক বেশি নিরাপদ কারণ **পাসওয়ার্ড কখনো নেটওয়ার্কে পাঠানো হয় না!**

**কিভাবে কাজ করে (Challenge-Response):**
```
Client                              Broker
  │                                    │
  │ ── 1. "আমি orderservice" ────────→ │
  │                                    │
  │ ←── 2. চ্যালেঞ্জ (random nonce) ── │
  │                                    │
  │ ── 3. response (hash+nonce) ─────→ │
  │                                    │
  │    Broker নিজে hash calculate করে │
  │    মিললে → ✅ authenticated        │
  │ ←── 4. success ─────────────────── │
```

**ZooKeeper-এ ইউজার তৈরি:**
```bash
# SCRAM-SHA-256 দিয়ে ইউজার তৈরি
bin/kafka-configs.sh --zookeeper localhost:2181 \
  --alter --add-config 'SCRAM-SHA-256=[password=bkash-order-secret]' \
  --entity-type users \
  --entity-name bkash-order-service

# SCRAM-SHA-512 (আরো শক্তিশালী)
bin/kafka-configs.sh --zookeeper localhost:2181 \
  --alter --add-config 'SCRAM-SHA-512=[password=nagad-txn-secret]' \
  --entity-type users \
  --entity-name nagad-transaction-service
```

**Broker কনফিগারেশন:**
```properties
listeners=SASL_SSL://0.0.0.0:9093
security.inter.broker.protocol=SASL_SSL
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-512
sasl.enabled.mechanisms=SCRAM-SHA-512

listener.name.sasl_ssl.scram-sha-512.sasl.jaas.config=\
  org.apache.kafka.common.security.scram.ScramLoginModule required \
  username="admin" \
  password="admin-scram-secret";
```

**Client কনফিগারেশন:**
```properties
security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
  username="bkash-order-service" \
  password="bkash-order-secret";
```

---

### 🅲️ SSL/TLS মিউচুয়াল অথেনটিকেশন

SSL/TLS মিউচুয়াল অথেনটিকেশনে **উভয় পক্ষই** সার্টিফিকেট দিয়ে নিজেদের পরিচয় প্রমাণ করে।

```
Client (certificate আছে)  ←──── TLS Handshake ────→  Broker (certificate আছে)
        ↓                                                      ↓
  "আমার certificate দেখো"                         "আমারটাও দেখো"
        ↓                                                      ↓
  উভয়ে verify করলো ✅ ──────── Secure Connection ──────────────→
```

#### ধাপে ধাপে সেটআপ:

**ধাপ ১: Certificate Authority (CA) তৈরি:**
```bash
# নিজস্ব CA তৈরি (production এ সত্যিকার CA ব্যবহার করুন)
openssl req -new -x509 -keyout ca-key -out ca-cert -days 365 \
  -subj "/CN=KafkaCA/O=bKashTech/L=Dhaka/C=BD" \
  -passout pass:ca-password
```

**ধাপ ২: Broker Keystore তৈরি:**
```bash
# Broker এর keystore (নিজের certificate)
keytool -keystore kafka.server.keystore.jks -alias broker1 \
  -validity 365 -genkey -keyalg RSA -storepass broker-ks-pass \
  -dname "CN=broker1.kafka.bkash.internal,O=bKashTech,L=Dhaka,C=BD"

# CSR (Certificate Signing Request) তৈরি
keytool -keystore kafka.server.keystore.jks -alias broker1 \
  -certreq -file broker1.csr -storepass broker-ks-pass

# CA দিয়ে sign করা
openssl x509 -req -CA ca-cert -CAkey ca-key \
  -in broker1.csr -out broker1-signed.crt \
  -days 365 -CAcreateserial -passin pass:ca-password

# Keystore এ CA cert ও signed cert import
keytool -keystore kafka.server.keystore.jks -alias CARoot \
  -import -file ca-cert -storepass broker-ks-pass -noprompt
keytool -keystore kafka.server.keystore.jks -alias broker1 \
  -import -file broker1-signed.crt -storepass broker-ks-pass
```

**ধাপ ৩: Truststore তৈরি:**
```bash
# Truststore এ CA cert যোগ (যাদের trust করবো)
keytool -keystore kafka.server.truststore.jks -alias CARoot \
  -import -file ca-cert -storepass truststore-pass -noprompt
```

**ধাপ ৪: Broker কনফিগারেশন:**
```properties
listeners=SSL://0.0.0.0:9093
security.inter.broker.protocol=SSL

ssl.keystore.location=/var/kafka/ssl/kafka.server.keystore.jks
ssl.keystore.password=broker-ks-pass
ssl.key.password=broker-ks-pass
ssl.truststore.location=/var/kafka/ssl/kafka.server.truststore.jks
ssl.truststore.password=truststore-pass

# মিউচুয়াল TLS সক্রিয়
ssl.client.auth=required
```

---

### 🅳 OAuth/OIDC (আধুনিক টোকেন-বেজড অথেনটিকেশন)

OAuth 2.0 / OpenID Connect হলো সবচেয়ে আধুনিক পদ্ধতি। বড় প্রতিষ্ঠানে যেখানে ইতিমধ্যে **Identity Provider** (Keycloak, Auth0, Okta) আছে, সেখানে এটি উপযুক্ত।

```
Client                    Identity Provider           Kafka Broker
  │                        (Keycloak/Auth0)               │
  │ ── 1. credentials ──→       │                         │
  │ ←── 2. JWT token ────       │                         │
  │                              │                         │
  │ ── 3. JWT token ─────────────────────────────────────→ │
  │                              │                         │
  │                    ←── 4. token verify ──→             │
  │                              │                         │
  │ ←── 5. authenticated ──────────────────────────────── │
```

**Broker কনফিগারেশন (Kafka 3.x+):**
```properties
listeners=SASL_SSL://0.0.0.0:9093
sasl.enabled.mechanisms=OAUTHBEARER

listener.name.sasl_ssl.oauthbearer.sasl.jaas.config=\
  org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required \
  clientId="kafka-broker" \
  clientSecret="broker-secret" \
  scope="kafka" \
  tokenEndpointUrl="https://keycloak.bkash.internal/realms/kafka/protocol/openid-connect/token";

# JWT token validation
listener.name.sasl_ssl.oauthbearer.sasl.oauthbearer.jwks.endpoint.url=\
  https://keycloak.bkash.internal/realms/kafka/protocol/openid-connect/certs
listener.name.sasl_ssl.oauthbearer.sasl.oauthbearer.expected.audience=kafka-cluster
```

---

## 🔒 অথরাইজেশন (ACLs)

অথরাইজেশন মানে **"তুমি কী কী করতে পারবে?"** — এটি নিয়ন্ত্রণ করা।

Kafka **ACL (Access Control List)** ব্যবহার করে অথরাইজেশন করে।

### ACL এর মূল ধারণা

**Resource Types (রিসোর্স টাইপ):**
| Resource        | বর্ণনা                          |
|-----------------|----------------------------------|
| Topic           | Kafka topic                      |
| Group           | Consumer group                   |
| Cluster         | পুরো cluster operations          |
| TransactionalId | Transactional producer ID        |

**Operations (অপারেশন):**
| Operation | বর্ণনা                           |
|-----------|-----------------------------------|
| Read      | Topic থেকে পড়া                   |
| Write     | Topic এ লেখা                     |
| Create    | নতুন topic তৈরি                  |
| Delete    | Topic মুছে ফেলা                  |
| Alter     | কনফিগারেশন পরিবর্তন             |
| Describe  | Topic/cluster তথ্য দেখা          |

### ACL সেটআপ উদাহরণ

**প্রথমে Broker-এ ACL authorizer সক্রিয় করুন:**
```properties
# server.properties
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
super.users=User:admin
allow.everyone.if.no.acl.found=false
```

#### উদাহরণ ১: order-service কে order-events topic-এ write অনুমতি

```bash
# bKash এর order-service শুধু order-events টপিকে লিখতে পারবে
bin/kafka-acls.sh --bootstrap-server localhost:9093 \
  --command-config admin.properties \
  --add \
  --allow-principal User:bkash-order-service \
  --operation Write \
  --operation Describe \
  --topic order-events
```

#### উদাহরণ ২: analytics-service কে সব topic থেকে read অনুমতি

```bash
# Daraz এর analytics-service সব topic পড়তে পারবে (prefix match)
bin/kafka-acls.sh --bootstrap-server localhost:9093 \
  --command-config admin.properties \
  --add \
  --allow-principal User:daraz-analytics \
  --operation Read \
  --operation Describe \
  --topic '*' \
  --resource-pattern-type prefixed

# Consumer group ও দরকার
bin/kafka-acls.sh --bootstrap-server localhost:9093 \
  --command-config admin.properties \
  --add \
  --allow-principal User:daraz-analytics \
  --operation Read \
  --group analytics-consumer-group
```

#### উদাহরণ ৩: নির্দিষ্ট group কে deny করা

```bash
# অবিশ্বস্ত third-party সার্ভিসকে ব্লক করা
bin/kafka-acls.sh --bootstrap-server localhost:9093 \
  --command-config admin.properties \
  --add \
  --deny-principal User:untrusted-vendor \
  --operation Read \
  --operation Write \
  --topic payment-transactions
```

#### ACL তালিকা দেখা:

```bash
# সব ACL দেখা
bin/kafka-acls.sh --bootstrap-server localhost:9093 \
  --command-config admin.properties \
  --list

# নির্দিষ্ট topic এর ACL
bin/kafka-acls.sh --bootstrap-server localhost:9093 \
  --command-config admin.properties \
  --list --topic order-events
```

### ❌ খারাপ: সবাইকে সব পারমিশন দেওয়া

```properties
# এটা করবেন না! 🚫
allow.everyone.if.no.acl.found=true
# যেকোনো সার্ভিস যেকোনো topic এ যা ইচ্ছা তাই করতে পারবে
```

বাংলা উপমা: ব্যাংকের ক্যাশিয়ার, গার্ড, ক্লিনার — সবাইকে vault এর চাবি দেওয়ার মতো!

### ✅ ভালো: Principle of Least Privilege

```
┌────────────────────────────────────────────────────┐
│         Pathao Kafka ACL ডিজাইন                    │
├──────────────────┬─────────────────────────────────┤
│ Service          │ Permissions                      │
├──────────────────┼─────────────────────────────────┤
│ ride-service     │ Write: ride-events              │
│                  │ Read: driver-locations           │
├──────────────────┼─────────────────────────────────┤
│ payment-service  │ Write: payment-events           │
│                  │ Read: ride-events (completed)    │
├──────────────────┼─────────────────────────────────┤
│ analytics        │ Read: * (সব topic)              │
│                  │ Write: কিছুই না ❌               │
├──────────────────┼─────────────────────────────────┤
│ notification-svc │ Read: payment-events            │
│                  │ Read: ride-events               │
│                  │ Write: notification-events       │
└──────────────────┴─────────────────────────────────┘
```

প্রতিটি সার্ভিস **শুধুমাত্র যতটুকু দরকার ততটুকুই** পারমিশন পায়।

---

## 🔑 এনক্রিপশন

### 🔄 In-Transit Encryption (TLS)

ডেটা যখন নেটওয়ার্কে চলাচল করে তখন তা এনক্রিপ্ট করা।

```
Without TLS:                          With TLS:
Client ── "password123" ──→ Broker    Client ── "x8F#k2..." ──→ Broker
              ↑                                     ↑
     হ্যাকার পড়তে পারে              হ্যাকার কিছুই বুঝবে না
```

#### Client ↔ Broker TLS কনফিগারেশন

**Broker (server.properties):**
```properties
listeners=SSL://0.0.0.0:9093
advertised.listeners=SSL://broker1.kafka.internal:9093

ssl.keystore.location=/var/kafka/ssl/kafka.server.keystore.jks
ssl.keystore.password=server-ks-pass
ssl.key.password=server-key-pass
ssl.truststore.location=/var/kafka/ssl/kafka.server.truststore.jks
ssl.truststore.password=server-ts-pass

ssl.enabled.protocols=TLSv1.3,TLSv1.2
ssl.protocol=TLSv1.3
```

#### Broker ↔ Broker (Inter-Broker) TLS

```properties
# Inter-broker communication ও TLS দিয়ে secure করা
security.inter.broker.protocol=SSL
ssl.keystore.location=/var/kafka/ssl/kafka.server.keystore.jks
ssl.keystore.password=server-ks-pass
ssl.truststore.location=/var/kafka/ssl/kafka.server.truststore.jks
ssl.truststore.password=server-ts-pass
```

#### Certificate তৈরির সম্পূর্ণ ধাপ

```bash
#!/bin/bash
# Kafka TLS সার্টিফিকেট তৈরির স্ক্রিপ্ট

PASS="changeit"
VALIDITY=365
CA_CN="KafkaCA"

# ধাপ ১: CA তৈরি
openssl req -new -x509 -keyout ca-key.pem -out ca-cert.pem \
  -days $VALIDITY -subj "/CN=$CA_CN/O=MyCompany/L=Dhaka/C=BD" \
  -passout pass:$PASS

# ধাপ ২: প্রতিটি broker এর জন্য keystore তৈরি
for i in 1 2 3; do
  BROKER="broker${i}"

  # Keystore তৈরি
  keytool -keystore ${BROKER}.keystore.jks -alias ${BROKER} \
    -validity $VALIDITY -genkey -keyalg RSA -keysize 2048 \
    -storepass $PASS -keypass $PASS \
    -dname "CN=${BROKER}.kafka.internal,O=MyCompany,L=Dhaka,C=BD"

  # CSR তৈরি
  keytool -keystore ${BROKER}.keystore.jks -alias ${BROKER} \
    -certreq -file ${BROKER}.csr -storepass $PASS

  # CA দিয়ে sign
  openssl x509 -req -CA ca-cert.pem -CAkey ca-key.pem \
    -in ${BROKER}.csr -out ${BROKER}.signed.crt \
    -days $VALIDITY -CAcreateserial -passin pass:$PASS

  # Keystore এ import
  keytool -keystore ${BROKER}.keystore.jks -alias CARoot \
    -import -file ca-cert.pem -storepass $PASS -noprompt
  keytool -keystore ${BROKER}.keystore.jks -alias ${BROKER} \
    -import -file ${BROKER}.signed.crt -storepass $PASS

  # Truststore তৈরি
  keytool -keystore ${BROKER}.truststore.jks -alias CARoot \
    -import -file ca-cert.pem -storepass $PASS -noprompt
done

echo "✅ সব broker এর certificate তৈরি সম্পন্ন!"
```

### 💾 At-Rest Encryption (ডিস্কে এনক্রিপশন)

**Kafka-তে built-in at-rest encryption নেই!** কারণ:
- Kafka ডিজাইন করা হয়েছে **high throughput** এর জন্য
- প্রতিটি message encrypt/decrypt করলে performance কমে যায়
- Kafka মনে করে — disk-level encryption OS বা hardware এর কাজ

**সমাধান:**

| পদ্ধতি | বর্ণনা | উদাহরণ |
|--------|--------|--------|
| OS-level encryption | ডিস্ক পার্টিশন encrypt | LUKS (Linux) |
| Cloud disk encryption | ক্লাউড প্রোভাইডারের encryption | AWS EBS Encryption |
| Application-level | Producer encrypt, Consumer decrypt | Custom encryptor |

**LUKS দিয়ে Kafka data directory encrypt:**
```bash
# Kafka data partition encrypt করা (Linux)
cryptsetup luksFormat /dev/sdb
cryptsetup open /dev/sdb kafka-data
mkfs.ext4 /dev/mapper/kafka-data
mount /dev/mapper/kafka-data /var/kafka/data
```

**Application-level encryption (বিশেষ সংবেদনশীল ডেটার জন্য):**
```
Producer → encrypt(message) → Kafka → Consumer → decrypt(message)
```

bKash এর পেমেন্ট ডেটার মতো সংবেদনশীল তথ্যের ক্ষেত্রে **application-level encryption** অতিরিক্ত নিরাপত্তা দেয়।

---

## 🏢 মাল্টি-টেনান্সি (Multi-Tenancy)

একটি Kafka cluster কে একাধিক টিম বা সার্ভিসের মধ্যে নিরাপদে ভাগ করা।

### Topic Naming Convention

```
<tenant>.<domain>.<event-type>

উদাহরণ:
  daraz.orders.created
  daraz.orders.shipped
  bkash.payments.completed
  bkash.payments.refunded
  pathao.rides.requested
  pathao.rides.completed
```

### মাল্টি-টেনান্ট আর্কিটেকচার

```
┌─────────────────────────────────────────────────┐
│              Kafka Cluster                       │
│                                                  │
│  ┌──────────────────┐  ┌──────────────────┐     │
│  │    Tenant A       │  │    Tenant B       │     │
│  │    (Daraz)        │  │    (Pathao)       │     │
│  │                   │  │                   │     │
│  │  topics:          │  │  topics:          │     │
│  │   a.orders        │  │   b.rides         │     │
│  │   a.products      │  │   b.drivers       │     │
│  │   a.payments      │  │   b.payments      │     │
│  │                   │  │                   │     │
│  │  ACLs ✅          │  │  ACLs ✅          │     │
│  │  Quotas ✅        │  │  Quotas ✅        │     │
│  │                   │  │                   │     │
│  │  Network: 50MB/s  │  │  Network: 30MB/s  │     │
│  │  Requests: 500/s  │  │  Requests: 300/s  │     │
│  └──────────────────┘  └──────────────────┘     │
│                                                  │
└─────────────────────────────────────────────────┘
```

### ACL দিয়ে Tenant Isolation

```bash
# Daraz শুধু "daraz." prefix যুক্ত topic access করতে পারবে
bin/kafka-acls.sh --bootstrap-server localhost:9093 \
  --command-config admin.properties \
  --add \
  --allow-principal User:daraz-app \
  --operation Read --operation Write --operation Create \
  --topic daraz. \
  --resource-pattern-type prefixed

# Pathao শুধু "pathao." prefix যুক্ত topic access করতে পারবে
bin/kafka-acls.sh --bootstrap-server localhost:9093 \
  --command-config admin.properties \
  --add \
  --allow-principal User:pathao-app \
  --operation Read --operation Write --operation Create \
  --topic pathao. \
  --resource-pattern-type prefixed
```

### Quotas (রিসোর্স সীমাবদ্ধতা)

```bash
# Daraz এর জন্য bandwidth quota
bin/kafka-configs.sh --bootstrap-server localhost:9093 \
  --command-config admin.properties \
  --alter --add-config \
  'producer_byte_rate=52428800,consumer_byte_rate=52428800,request_percentage=50' \
  --entity-type users --entity-name daraz-app

# Pathao এর জন্য bandwidth quota
bin/kafka-configs.sh --bootstrap-server localhost:9093 \
  --command-config admin.properties \
  --alter --add-config \
  'producer_byte_rate=31457280,consumer_byte_rate=31457280,request_percentage=30' \
  --entity-type users --entity-name pathao-app
```

---

## 📊 সিকিউরিটি আর্কিটেকচার ডায়াগ্রাম

সম্পূর্ণ Kafka সিকিউরিটি আর্কিটেকচার — সব লেয়ার একসাথে:

```
                         Kafka সিকিউরিটি আর্কিটেকচার
                         ═══════════════════════════════

 External Clients                    Kafka Cluster                    
 ════════════════                    ═════════════                    

 ┌─────────────┐                                                     
 │ bKash App   │                                                     
 │ (Producer)  │──┐                                                  
 └─────────────┘  │                                                  
                  │    ┌──────────┐    ┌───────────┐    ┌──────────┐
 ┌─────────────┐  ├───→│ 🔑 TLS   │───→│ 🔐 SASL   │───→│ 🔒 ACL   │
 │ Daraz App   │──┤    │ Encrypt  │    │ Auth      │    │ Check    │
 │ (Producer)  │  │    │ Layer    │    │ Layer     │    │ Layer    │
 └─────────────┘  │    └──────────┘    └───────────┘    └────┬─────┘
                  │                                          │      
 ┌─────────────┐  │                                          ▼      
 │ Analytics   │──┘                              ┌──────────────────┐
 │ (Consumer)  │                                 │   Kafka Broker   │
 └─────────────┘                                 │    (broker-1)    │
                                                 └────────┬─────────┘
                                                          │         
                                                  Inter-Broker TLS  
                                                          │         
                                                 ┌────────▼─────────┐
                                                 │   Kafka Broker   │
                                                 │    (broker-2)    │
                                                 └────────┬─────────┘
                                                          │         
                                                  Inter-Broker TLS  
                                                          │         
                                                 ┌────────▼─────────┐
                                                 │   Kafka Broker   │
                                                 │    (broker-3)    │
                                                 └──────────────────┘


 ফ্লো সারাংশ:
 ═══════════════
 Client → TLS (encryption) → SASL (authentication) → ACL (authorization) → Broker
 Broker ←→ Inter-Broker TLS ←→ Broker
```

---

## 💻 কনফিগারেশন উদাহরণ

### সম্পূর্ণ Secured server.properties

```properties
# ============================
# Kafka Secured Broker Config
# bKash Production Environment
# ============================

# --- Broker Basics ---
broker.id=1
log.dirs=/var/kafka/data

# --- Listeners (SASL_SSL for clients, SSL for inter-broker) ---
listeners=SASL_SSL://0.0.0.0:9093
advertised.listeners=SASL_SSL://broker1.kafka.bkash.internal:9093
security.inter.broker.protocol=SASL_SSL

# --- SASL Configuration ---
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-512
sasl.enabled.mechanisms=SCRAM-SHA-512

listener.name.sasl_ssl.scram-sha-512.sasl.jaas.config=\
  org.apache.kafka.common.security.scram.ScramLoginModule required \
  username="inter-broker" \
  password="inter-broker-secret-2024";

# --- SSL/TLS Configuration ---
ssl.keystore.type=JKS
ssl.keystore.location=/var/kafka/ssl/broker1.keystore.jks
ssl.keystore.password=keystore-pass-2024
ssl.key.password=key-pass-2024
ssl.truststore.type=JKS
ssl.truststore.location=/var/kafka/ssl/broker1.truststore.jks
ssl.truststore.password=truststore-pass-2024

ssl.enabled.protocols=TLSv1.3,TLSv1.2
ssl.protocol=TLSv1.3
ssl.client.auth=required

# --- ACL Configuration ---
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
super.users=User:admin;User:inter-broker
allow.everyone.if.no.acl.found=false

# --- ZooKeeper (secured) ---
zookeeper.connect=zk1:2181,zk2:2181,zk3:2181/kafka
zookeeper.set.acl=true
```

### PHP Client কনফিগারেশন (php-rdkafka)

```php
<?php
// bKash Payment Service - Kafka Producer (PHP)

$config = new RdKafka\Conf();

// SASL/SCRAM authentication
$config->set('security.protocol', 'SASL_SSL');
$config->set('sasl.mechanism', 'SCRAM-SHA-512');
$config->set('sasl.username', 'bkash-payment-service');
$config->set('sasl.password', 'payment-service-secret');

// TLS configuration
$config->set('ssl.ca.location', '/etc/kafka/ssl/ca-cert.pem');
$config->set('ssl.certificate.location', '/etc/kafka/ssl/client-cert.pem');
$config->set('ssl.key.location', '/etc/kafka/ssl/client-key.pem');
$config->set('ssl.key.password', 'client-key-pass');

// Broker list
$config->set('metadata.broker.list', 'broker1.kafka.bkash.internal:9093,broker2.kafka.bkash.internal:9093');

$producer = new RdKafka\Producer($config);
$topic = $producer->newTopic('bkash.payments.completed');

$paymentEvent = json_encode([
    'transaction_id' => 'TXN-2024-001',
    'amount' => 5000,
    'currency' => 'BDT',
    'sender' => '01712XXXXXX',
    'receiver' => '01819XXXXXX',
    'timestamp' => date('c')
]);

$topic->produce(RD_KAFKA_PARTITION_UA, 0, $paymentEvent);
$producer->flush(10000);

echo "✅ পেমেন্ট ইভেন্ট পাঠানো হয়েছে\n";
```

### JavaScript/Node.js Client কনফিগারেশন (KafkaJS)

```javascript
// Daraz Order Service - Kafka Consumer (Node.js)
const { Kafka } = require('kafkajs');
const fs = require('fs');

const kafka = new Kafka({
  clientId: 'daraz-order-service',
  brokers: [
    'broker1.kafka.daraz.internal:9093',
    'broker2.kafka.daraz.internal:9093'
  ],

  // SASL/SCRAM authentication
  sasl: {
    mechanism: 'scram-sha-512',
    username: 'daraz-order-service',
    password: 'order-service-secret-2024'
  },

  // TLS configuration
  ssl: {
    rejectUnauthorized: true,
    ca: [fs.readFileSync('/etc/kafka/ssl/ca-cert.pem', 'utf-8')],
    cert: fs.readFileSync('/etc/kafka/ssl/client-cert.pem', 'utf-8'),
    key: fs.readFileSync('/etc/kafka/ssl/client-key.pem', 'utf-8'),
    passphrase: 'client-key-pass'
  }
});

const consumer = kafka.consumer({ groupId: 'daraz-order-consumers' });

const run = async () => {
  await consumer.connect();
  await consumer.subscribe({ topic: 'daraz.orders.created', fromBeginning: false });

  await consumer.run({
    eachMessage: async ({ topic, partition, message }) => {
      const order = JSON.parse(message.value.toString());
      console.log(`📦 নতুন অর্ডার পাওয়া গেছে: ${order.orderId}`);
      // অর্ডার প্রসেস করা...
    }
  });
};

run().catch(console.error);
```

---

## 📋 সিকিউরিটি চেকলিস্ট

Production এ Kafka deploy করার আগে এই চেকলিস্ট অনুসরণ করুন:

### অথেনটিকেশন
- [ ] SASL/SCRAM অথবা mTLS সক্রিয় করা হয়েছে
- [ ] SASL/PLAIN ব্যবহার করলে অবশ্যই TLS সাথে আছে
- [ ] Inter-broker authentication সক্রিয়
- [ ] ZooKeeper authentication সক্রিয়
- [ ] ডিফল্ট credentials পরিবর্তন করা হয়েছে

### অথরাইজেশন
- [ ] ACL authorizer সক্রিয়
- [ ] `allow.everyone.if.no.acl.found=false` সেট করা
- [ ] Super users সীমিত রাখা হয়েছে
- [ ] প্রতিটি সার্ভিসের জন্য আলাদা user ও ACL
- [ ] Principle of least privilege অনুসরণ করা হচ্ছে

### এনক্রিপশন
- [ ] Client-to-broker TLS সক্রিয়
- [ ] Inter-broker TLS সক্রিয়
- [ ] TLS 1.2 বা 1.3 ব্যবহার করা হচ্ছে
- [ ] Certificates এর মেয়াদ মনিটর করা হচ্ছে
- [ ] Disk encryption সক্রিয় (LUKS বা cloud encryption)

### নেটওয়ার্ক
- [ ] Kafka ports ফায়ারওয়াল দিয়ে সীমাবদ্ধ
- [ ] ZooKeeper পাবলিক নেটওয়ার্কে expose করা হয়নি
- [ ] Private network/VPC ব্যবহার করা হচ্ছে

### মনিটরিং ও অডিট
- [ ] Authentication failure লগ করা হচ্ছে
- [ ] ACL পরিবর্তন লগ করা হচ্ছে
- [ ] Unauthorized access attempt alert সেট করা

---

## 🎯 বেস্ট প্র্যাকটিস

### ১. সবসময় SASL + TLS একসাথে ব্যবহার করুন
```
❌ SASL_PLAINTEXT  → পাসওয়ার্ড plaintext এ যায়
❌ PLAINTEXT       → কোনো সিকিউরিটি নেই
✅ SASL_SSL        → authentication + encryption
✅ SSL (mTLS)      → certificate-based + encryption
```

### ২. প্রতিটি সার্ভিসের জন্য আলাদা credential
```
❌ সব সার্ভিস একই user: "app-user"
✅ আলাদা আলাদা user:
   - bkash-order-service
   - bkash-payment-service
   - bkash-notification-service
```

### ৩. Certificate rotation পরিকল্পনা রাখুন
```
- Certificate এর মেয়াদ ৯০-৩৬৫ দিন
- মেয়াদ শেষ হওয়ার ৩০ দিন আগে renew করুন
- Automated renewal (cert-manager/Vault) ব্যবহার করুন
```

### ৪. ZooKeeper ও secure করুন
```properties
# ZooKeeper ACL সক্রিয় করুন
zookeeper.set.acl=true

# ZooKeeper এ SASL ব্যবহার করুন
# ZooKeeper একটি critical component — এটি unsecured রাখলে
# পুরো Kafka cluster compromise হতে পারে!
```

### ৫. Credential Management
```
❌ কনফিগ ফাইলে হার্ডকোড করা password
✅ Environment variable থেকে পড়া
✅ HashiCorp Vault / AWS Secrets Manager ব্যবহার
✅ Kubernetes Secrets (encrypted at rest)
```

### ৬. Network Segmentation
```
Internet → Load Balancer → App Servers → Kafka (Private Network)
                                              ↓
                                         ZooKeeper (আরো restricted)
```

Kafka ও ZooKeeper কখনো সরাসরি internet এ expose করবেন না!

---

## 🔗 পরবর্তী টপিক

| টপিক | লিংক |
|-------|-------|
| ⬅️ আগের টপিক | [Kafka Basics](kafka-basics.md) |
| ➡️ পরের টপিক | [Kafka মনিটরিং](kafka-monitoring.md) |

---

> 📝 **নোট:** এই ডকুমেন্টটি বাংলাদেশি ডেভেলপারদের জন্য তৈরি।
> প্রোডাকশনে deploy করার আগে আপনার organization এর security team এর সাথে পরামর্শ করুন।
> সিকিউরিটি কোনো one-time কাজ নয় — এটি একটি **চলমান প্রক্রিয়া**।
