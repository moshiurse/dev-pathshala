# 🔐 এনক্রিপশন (Encryption) — Symmetric, Asymmetric ও হ্যাশিং

## 📌 ধারণা (Concept)

এনক্রিপশন হলো ডেটাকে এমনভাবে রূপান্তর করা যাতে শুধু অনুমোদিত ব্যক্তিরাই এটি পড়তে পারে। এনক্রিপশনের তিনটি প্রধান ধারণা:

- **Symmetric Encryption:** একটি কী (key) দিয়ে এনক্রিপ্ট ও ডিক্রিপ্ট উভয়ই করা হয়
- **Asymmetric Encryption:** দুটি কী ব্যবহার — Public Key (এনক্রিপ্ট) ও Private Key (ডিক্রিপ্ট)
- **Hashing:** একমুখী রূপান্তর — মূল ডেটা ফেরত পাওয়া যায় না (পাসওয়ার্ড সংরক্ষণে ব্যবহৃত)

## 🏠 বাস্তব জীবনের উদাহরণ

### Symmetric Encryption
একটি তালা ও চাবি — একই চাবি দিয়ে তালা খোলা ও বন্ধ করা যায়। সমস্যা হলো চাবি অন্যকে দিতে হলে নিরাপদে পাঠাতে হবে।

### Asymmetric Encryption
পোস্ট বক্স — যে কেউ চিঠি ফেলতে পারে (public key), কিন্তু শুধু আপনার কাছেই চাবি আছে বক্স খোলার (private key)।

### Hashing
আঙুলের ছাপ — একটি মানুষ থেকে ছাপ তৈরি করা যায়, কিন্তু ছাপ থেকে মানুষ তৈরি করা যায় না। শুধু মিলিয়ে দেখা যায়।

---

## 🔵 Symmetric Encryption

### PHP — AES-256-GCM

```php
class SymmetricEncryption {
    private string $key;
    private string $cipher = 'aes-256-gcm';

    public function __construct(string $key) {
        // কী অবশ্যই ৩২ বাইট (256 bit) হতে হবে
        $this->key = $key;
    }

    public function encrypt(string $plainText): string {
        $iv = random_bytes(openssl_cipher_iv_length($this->cipher));
        $tag = '';

        $cipherText = openssl_encrypt(
            $plainText,
            $this->cipher,
            $this->key,
            OPENSSL_RAW_DATA,
            $iv,
            $tag,
            '', // AAD (Additional Authenticated Data)
            16  // tag length
        );

        // IV + Tag + CipherText একসাথে সংরক্ষণ
        return base64_encode($iv . $tag . $cipherText);
    }

    public function decrypt(string $encrypted): string {
        $data = base64_decode($encrypted);
        $ivLength = openssl_cipher_iv_length($this->cipher);

        $iv = substr($data, 0, $ivLength);
        $tag = substr($data, $ivLength, 16);
        $cipherText = substr($data, $ivLength + 16);

        $plainText = openssl_decrypt(
            $cipherText,
            $this->cipher,
            $this->key,
            OPENSSL_RAW_DATA,
            $iv,
            $tag
        );

        if ($plainText === false) {
            throw new RuntimeException('ডিক্রিপশন ব্যর্থ — ডেটা পরিবর্তিত হয়েছে');
        }

        return $plainText;
    }
}

// ব্যবহার
$key = random_bytes(32); // নিরাপদ কী তৈরি
$enc = new SymmetricEncryption($key);
$encrypted = $enc->encrypt('গোপন তথ্য');
$decrypted = $enc->decrypt($encrypted);
```

### JavaScript (Node.js) — AES-256-GCM

```javascript
import { createCipheriv, createDecipheriv, randomBytes } from 'crypto';

class SymmetricEncryption {
    #key;

    constructor(key) {
        this.#key = key; // 32 bytes
    }

    encrypt(plainText) {
        const iv = randomBytes(16);
        const cipher = createCipheriv('aes-256-gcm', this.#key, iv);

        let encrypted = cipher.update(plainText, 'utf8', 'hex');
        encrypted += cipher.final('hex');
        const authTag = cipher.getAuthTag().toString('hex');

        return `${iv.toString('hex')}:${authTag}:${encrypted}`;
    }

    decrypt(encryptedData) {
        const [ivHex, authTagHex, encrypted] = encryptedData.split(':');

        const decipher = createDecipheriv(
            'aes-256-gcm',
            this.#key,
            Buffer.from(ivHex, 'hex')
        );
        decipher.setAuthTag(Buffer.from(authTagHex, 'hex'));

        let decrypted = decipher.update(encrypted, 'hex', 'utf8');
        decrypted += decipher.final('utf8');
        return decrypted;
    }
}

// ব্যবহার
const key = randomBytes(32);
const enc = new SymmetricEncryption(key);
const encrypted = enc.encrypt('গোপন তথ্য');
const decrypted = enc.decrypt(encrypted);
```

---

## 🔴 Asymmetric Encryption

### PHP — RSA

```php
// কী পেয়ার তৈরি
$config = [
    'private_key_bits' => 2048,
    'private_key_type' => OPENSSL_KEYTYPE_RSA,
];
$keyPair = openssl_pkey_new($config);

// Private key বের করা
openssl_pkey_export($keyPair, $privateKey);
// Public key বের করা
$publicKey = openssl_pkey_get_details($keyPair)['key'];

// এনক্রিপ্ট (Public Key দিয়ে)
$plainText = 'গোপন মেসেজ';
openssl_public_encrypt($plainText, $encrypted, $publicKey);
$encodedData = base64_encode($encrypted);

// ডিক্রিপ্ট (Private Key দিয়ে)
openssl_private_decrypt(base64_decode($encodedData), $decrypted, $privateKey);
echo $decrypted; // গোপন মেসেজ
```

### JavaScript (Node.js) — RSA

```javascript
import { generateKeyPairSync, publicEncrypt, privateDecrypt } from 'crypto';

// কী পেয়ার তৈরি
const { publicKey, privateKey } = generateKeyPairSync('rsa', {
    modulusLength: 2048,
    publicKeyEncoding: { type: 'spki', format: 'pem' },
    privateKeyEncoding: { type: 'pkcs8', format: 'pem' }
});

// এনক্রিপ্ট
const encrypted = publicEncrypt(publicKey, Buffer.from('গোপন মেসেজ'));
const encodedData = encrypted.toString('base64');

// ডিক্রিপ্ট
const decrypted = privateDecrypt(privateKey, Buffer.from(encodedData, 'base64'));
console.log(decrypted.toString()); // গোপন মেসেজ
```

---

## 🟣 হ্যাশিং (Hashing) — পাসওয়ার্ড সংরক্ষণ

### PHP — bcrypt ও Argon2

```php
// ✅ bcrypt দিয়ে হ্যাশ (ডিফল্ট ও সবচেয়ে সাধারণ)
$hash = password_hash('mypassword', PASSWORD_BCRYPT, ['cost' => 12]);

// ✅ Argon2id দিয়ে হ্যাশ (আরও নিরাপদ, PHP 7.3+)
$hash = password_hash('mypassword', PASSWORD_ARGON2ID, [
    'memory_cost' => 65536,  // 64 MB
    'time_cost' => 4,        // ৪ বার iteration
    'threads' => 3           // ৩টি থ্রেড
]);

// পাসওয়ার্ড যাচাই
if (password_verify('mypassword', $hash)) {
    echo 'পাসওয়ার্ড সঠিক!';
}

// হ্যাশ রি-হ্যাশ দরকার কিনা চেক (অ্যালগরিদম বা cost পরিবর্তন হলে)
if (password_needs_rehash($hash, PASSWORD_ARGON2ID, ['memory_cost' => 65536])) {
    $newHash = password_hash('mypassword', PASSWORD_ARGON2ID, ['memory_cost' => 65536]);
    // ডাটাবেসে নতুন হ্যাশ সেভ করুন
}
```

### JavaScript (Node.js) — bcrypt ও Argon2

```javascript
// bcrypt
import bcrypt from 'bcrypt';

const saltRounds = 12;
const hash = await bcrypt.hash('mypassword', saltRounds);
const isMatch = await bcrypt.compare('mypassword', hash);

// Argon2 (আরও নিরাপদ)
import argon2 from 'argon2';

const argonHash = await argon2.hash('mypassword', {
    type: argon2.argon2id,
    memoryCost: 65536,
    timeCost: 3,
    parallelism: 4
});

const isValid = await argon2.verify(argonHash, 'mypassword');
```

---

## 📊 তুলনা

| বৈশিষ্ট্য | Symmetric | Asymmetric | Hashing |
|-----------|-----------|------------|---------|
| কী সংখ্যা | ১টি | ২টি (Public + Private) | কী নেই |
| গতি | দ্রুত | ধীর | মাঝারি |
| উল্টানো যায়? | হ্যাঁ | হ্যাঁ | না |
| ব্যবহার | ডেটা এনক্রিপশন | কী বিনিময়, ডিজিটাল সিগনেচার | পাসওয়ার্ড সংরক্ষণ |
| উদাহরণ | AES-256 | RSA, ECDSA | bcrypt, Argon2 |

---

## ✅ কখন ব্যবহার করবেন

- **Symmetric:** বড় ডেটা এনক্রিপ্ট করতে, ডাটাবেসে সংবেদনশীল ফিল্ড এনক্রিপ্ট করতে
- **Asymmetric:** SSL/TLS, ডিজিটাল সিগনেচার, কী বিনিময়
- **bcrypt/Argon2:** পাসওয়ার্ড সংরক্ষণে সবসময়

## ❌ কখন ব্যবহার করবেন না

- **MD5/SHA1:** পাসওয়ার্ড হ্যাশিংয়ে কখনো ব্যবহার করবেন না (অত্যন্ত দুর্বল)
- **Symmetric:** কী বিনিময় সমস্যা থাকলে (পরিবর্তে Asymmetric ব্যবহার করুন)
- **নিজে ক্রিপ্টোগ্রাফি তৈরি করবেন না** — সবসময় প্রমাণিত লাইব্রেরি ব্যবহার করুন
- **ECB মোড:** কখনো ব্যবহার করবেন না — পরিবর্তে GCM বা CBC ব্যবহার করুন
