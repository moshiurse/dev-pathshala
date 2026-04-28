# 🔄 কনকারেন্সি ও ডিস্ট্রিবিউটেড সিস্টেম

কনকারেন্সি প্যাটার্ন ও ডিস্ট্রিবিউটেড সিস্টেমের মূল ধারণা। এই সেকশনে আমরা শিখবো কিভাবে একাধিক প্রসেস বা থ্রেড একসাথে কাজ করে, কিভাবে ডেটা কনসিস্টেন্সি বজায় রাখা যায়, এবং ডিস্ট্রিবিউটেড সিস্টেমে ট্রানজ্যাকশন ম্যানেজমেন্ট কিভাবে করা হয়।

---

## 📚 টপিক সমূহ

| # | টপিক | বিবরণ |
|---|-------|--------|
| ১ | [রেস কন্ডিশন](./race-conditions.md) | Race Conditions, Critical Section, Mutex, Semaphore |
| ২ | [ডেডলক](./deadlocks.md) | Deadlock Conditions, Detection, Prevention, Avoidance |
| ৩ | [সাগা প্যাটার্ন](./saga-pattern.md) | Distributed Transactions — Choreography vs Orchestration |
| ৪ | [ডিস্ট্রিবিউটেড কনসেনসাস](./distributed-consensus.md) | Raft, Paxos, Leader Election |
| ৫ | [কনকারেন্সি প্যাটার্নস](./concurrency-patterns.md) | Producer-Consumer, Reader-Writer, Thread Pool |
| ৬ | [ইভেন্ট লুপ](./event-loop.md) | JavaScript Event Loop, PHP Async, Non-blocking I/O |

---

## 🎯 কাদের জন্য?

এই সেকশনটি সিনিয়র সফটওয়্যার ইঞ্জিনিয়ারদের জন্য তৈরি যারা:
- হাই-কনকারেন্সি সিস্টেম ডিজাইন করতে চান
- ডিস্ট্রিবিউটেড সিস্টেমের চ্যালেঞ্জ বুঝতে চান
- রেস কন্ডিশন ও ডেডলক সমস্যা সমাধান করতে চান
- ইভেন্ট-ড্রিভেন আর্কিটেকচার বুঝতে চান

## 🔗 পূর্বশর্ত

- অপারেটিং সিস্টেমের বেসিক ধারণা (প্রসেস, থ্রেড)
- ডাটাবেজ ট্রানজ্যাকশনের ধারণা
- PHP ও JavaScript এর বেসিক জ্ঞান
