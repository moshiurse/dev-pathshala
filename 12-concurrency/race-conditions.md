# 🏁 রেস কন্ডিশন (Race Conditions)

## ধারণা

রেস কন্ডিশন হলো এমন একটি পরিস্থিতি যেখানে দুই বা ততোধিক প্রসেস/থ্রেড একই শেয়ারড রিসোর্সে একই সময়ে অ্যাক্সেস করে এবং ফলাফল নির্ভর করে কোন প্রসেসটি আগে এক্সিকিউট হয়। এটি অপ্রত্যাশিত ও ভুল ফলাফল তৈরি করতে পারে।

### মূল কম্পোনেন্ট

- **Critical Section**: কোডের সেই অংশ যেখানে শেয়ারড রিসোর্স অ্যাক্সেস করা হয়
- **Mutex (Mutual Exclusion)**: একটি লকিং মেকানিজম — একটি সময়ে শুধুমাত্র একটি থ্রেড অ্যাক্সেস পায়
- **Semaphore**: একটি কাউন্টিং মেকানিজম — নির্দিষ্ট সংখ্যক থ্রেড একসাথে অ্যাক্সেস পেতে পারে

---

## 🏠 বাস্তব জীবনের উদাহরণ

ধরুন, একটি ATM মেশিনে আপনার অ্যাকাউন্টে ১০,০০০ টাকা আছে। আপনি ও আপনার স্ত্রী একই সময়ে দুটি ভিন্ন ATM থেকে ৮,০০০ টাকা তুলতে চেষ্টা করছেন।

- **রেস কন্ডিশন ছাড়া**: প্রথমজন টাকা তুলবে, দ্বিতীয়জন "Insufficient Balance" পাবে।
- **রেস কন্ডিশনে**: উভয়ই ব্যালেন্স ১০,০০০ দেখবে, উভয়ই ৮,০০০ টাকা তুলে ফেলবে — মোট ১৬,০০০ টাকা!

**Mutex** হলো ATM এর লক — একবারে একজনই টাকা তুলতে পারবে।
**Semaphore** হলো ব্যাংকের কাউন্টার — ৩টি কাউন্টার থাকলে একসাথে ৩ জন সেবা পাবে।

---

## 💻 PHP কোড উদাহরণ

### সমস্যা: রেস কন্ডিশন (ফাইল-ভিত্তিক কাউন্টার)

```php
<?php
// ❌ রেস কন্ডিশন সমস্যা — দুটি প্রসেস একসাথে চালালে ভুল ফলাফল আসবে
function incrementCounter(): void
{
    $count = (int) file_get_contents('counter.txt');
    // এই গ্যাপে অন্য প্রসেস পুরোনো ভ্যালু পড়তে পারে
    usleep(100);
    file_put_contents('counter.txt', $count + 1);
}

incrementCounter();
```

### সমাধান: Mutex দিয়ে লকিং

```php
<?php
// ✅ ফাইল লক (Mutex) ব্যবহার করে রেস কন্ডিশন সমাধান
function safeIncrementCounter(): void
{
    $lockFile = fopen('counter.lock', 'c');

    // Exclusive Lock অর্জন — অন্য প্রসেস অপেক্ষা করবে
    if (flock($lockFile, LOCK_EX)) {
        $count = (int) file_get_contents('counter.txt');
        usleep(100);
        file_put_contents('counter.txt', $count + 1);

        // লক রিলিজ
        flock($lockFile, LOCK_UN);
    }

    fclose($lockFile);
}

safeIncrementCounter();
```

### Semaphore উদাহরণ (PHP sysvsem)

```php
<?php
// Semaphore দিয়ে সর্বোচ্চ ৩টি কানেকশন অনুমোদন
$semaphoreId = sem_get(1234, 3); // সর্বোচ্চ ৩টি প্রসেস

if (sem_acquire($semaphoreId)) {
    echo "রিসোর্স অ্যাক্সেস পাওয়া গেছে\n";
    // ক্রিটিক্যাল সেকশনের কাজ
    processRequest();
    sem_release($semaphoreId);
}

function processRequest(): void
{
    echo "রিকোয়েস্ট প্রসেস হচ্ছে...\n";
    sleep(2);
    echo "রিকোয়েস্ট সম্পন্ন!\n";
}
```

---

## 💻 JavaScript কোড উদাহরণ

### সমস্যা: রেস কন্ডিশন (Async অপারেশন)

```javascript
// ❌ রেস কন্ডিশন — দুটি async অপারেশন একই ডেটা আপডেট করছে
let balance = 10000;

async function withdraw(amount) {
    const currentBalance = balance;
    // নেটওয়ার্ক ল্যাটেন্সি সিমুলেশন
    await new Promise(resolve => setTimeout(resolve, 100));

    if (currentBalance >= amount) {
        balance = currentBalance - amount;
        console.log(`${amount} টাকা তোলা হয়েছে। অবশিষ্ট: ${balance}`);
    } else {
        console.log('অপর্যাপ্ত ব্যালেন্স!');
    }
}

// দুটো withdraw একসাথে চলবে — রেস কন্ডিশন!
Promise.all([withdraw(8000), withdraw(8000)]);
```

### সমাধান: Mutex প্যাটার্ন

```javascript
// ✅ Mutex ক্লাস দিয়ে রেস কন্ডিশন সমাধান
class Mutex {
    constructor() {
        this._locked = false;
        this._queue = [];
    }

    async acquire() {
        return new Promise(resolve => {
            if (!this._locked) {
                this._locked = true;
                resolve();
            } else {
                this._queue.push(resolve);
            }
        });
    }

    release() {
        if (this._queue.length > 0) {
            const next = this._queue.shift();
            next();
        } else {
            this._locked = false;
        }
    }
}

const mutex = new Mutex();
let balance = 10000;

async function safeWithdraw(amount) {
    await mutex.acquire();
    try {
        const currentBalance = balance;
        await new Promise(resolve => setTimeout(resolve, 100));

        if (currentBalance >= amount) {
            balance = currentBalance - amount;
            console.log(`${amount} টাকা তোলা হয়েছে। অবশিষ্ট: ${balance}`);
        } else {
            console.log('অপর্যাপ্ত ব্যালেন্স!');
        }
    } finally {
        mutex.release();
    }
}

// এবার সঠিকভাবে কাজ করবে
Promise.all([safeWithdraw(8000), safeWithdraw(8000)]);
```

---

## ✅ কখন ব্যবহার করবেন

- শেয়ারড রিসোর্স (ফাইল, ডাটাবেজ, মেমরি) একাধিক প্রসেস থেকে অ্যাক্সেস করার সময়
- ব্যাংকিং/ফিনান্সিয়াল ট্রানজ্যাকশনে ডেটা কনসিস্টেন্সি নিশ্চিত করতে
- ক্যাশে আপডেটের সময় (Cache Stampede প্রতিরোধে)
- ইনভেন্টরি ম্যানেজমেন্ট সিস্টেমে স্টক আপডেটের সময়

## ❌ কখন ব্যবহার করবেন না

- যখন রিসোর্স শেয়ারড নয় (প্রতি রিকোয়েস্টে আলাদা ডেটা)
- রিড-অনলি অপারেশনে (কোনো ডেটা পরিবর্তন হচ্ছে না)
- অত্যধিক লকিং পারফরম্যান্স কমিয়ে দিতে পারে — অপ্রয়োজনে ব্যবহার করবেন না
- যদি ডাটাবেজ লেভেল লকিং (SELECT FOR UPDATE) যথেষ্ট হয়, তাহলে অ্যাপ্লিকেশন লেভেল Mutex অপ্রয়োজনীয়

---

## 🔑 মূল পয়েন্ট

| কনসেপ্ট | বিবরণ |
|----------|--------|
| Race Condition | একাধিক প্রসেসের শেয়ারড রিসোর্সে অনিয়ন্ত্রিত অ্যাক্সেস |
| Critical Section | কোডের সেই অংশ যেখানে শেয়ারড রিসোর্স পরিবর্তন হয় |
| Mutex | বাইনারি লক — একবারে একটি প্রসেস |
| Semaphore | কাউন্টিং লক — নির্দিষ্ট সংখ্যক প্রসেস |
| Atomic Operation | অবিভাজ্য অপারেশন — মাঝপথে থামানো যায় না |
