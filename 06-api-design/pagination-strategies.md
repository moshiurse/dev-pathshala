# 📄 Pagination Strategies — API Pagination কৌশল বিস্তারিত গাইড

## 📖 সংজ্ঞা ও মূল ধারণা

Pagination হলো বড় dataset-কে ছোট ছোট অংশে (page) ভাগ করে client-কে পাঠানোর কৌশল। সঠিক pagination strategy নির্বাচন API-এর performance, scalability এবং user experience-এ সরাসরি প্রভাব ফেলে।

### তিনটি প্রধান কৌশল:
- **Offset Pagination** — `LIMIT` ও `OFFSET` ব্যবহার করে page নম্বর ভিত্তিক
- **Cursor Pagination** — শেষ record-এর unique identifier (cursor) ব্যবহার করে পরবর্তী page fetch
- **Keyset Pagination** — sorted column-এর মান ব্যবহার করে `WHERE` clause দিয়ে পরবর্তী page

---

## 🏗️ Pagination Strategies বিস্তারিত

### 1. Offset Pagination (সবচেয়ে সাধারণ)

```sql
-- Page 1 (প্রথম ২০টি record)
SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 0;

-- Page 2 (পরবর্তী ২০টি)
SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 20;

-- Page N
SELECT * FROM products ORDER BY id LIMIT 20 OFFSET (N-1) * 20;
```

```
Offset Pagination ডায়াগ্রাম:

ডেটাবেস: [1][2][3][4][5][6][7][8][9][10][11][12]....[1000000]
                                                         ↑
Page 1:   [1][2][3][4][5]  → OFFSET 0,  LIMIT 5        |
Page 2:   [6][7][8][9][10] → OFFSET 5,  LIMIT 5        |
Page 3:   [11][12]...      → OFFSET 10, LIMIT 5        |
                                                         |
Page 200000: → OFFSET 999995 → DB কে ৯৯৯,৯৯৫টি row    |
               skip করতে হয় → 🐌 অত্যন্ত ধীর!    ←---+
```

**সুবিধা:** সহজ implementation, যেকোনো page-এ সরাসরি যাওয়া যায়, মোট page সংখ্যা জানা যায়

**অসুবিধা:** বড় offset-এ performance খারাপ হয়, data পরিবর্তনে duplicate/missing record

### 2. Cursor Pagination (Facebook, Twitter style)

```sql
-- প্রথম page
SELECT * FROM products ORDER BY created_at DESC, id DESC LIMIT 20;

-- পরবর্তী page (শেষ record-এর cursor ব্যবহার করে)
SELECT * FROM products 
WHERE (created_at, id) < ('2024-01-15 10:30:00', 450)
ORDER BY created_at DESC, id DESC 
LIMIT 20;
```

```
Cursor Pagination ডায়াগ্রাম:

Request 1: GET /api/products?limit=5
Response:  [A][B][C][D][E]  → next_cursor = "cursor_E"
                       ↑
                    cursor

Request 2: GET /api/products?limit=5&after=cursor_E
Response:  [F][G][H][I][J]  → next_cursor = "cursor_J"
                       ↑
                    cursor

Request 3: GET /api/products?limit=5&after=cursor_J
Response:  [K][L]           → next_cursor = null (শেষ page)
```

**সুবিধা:** যেকোনো dataset size-এ consistent performance, real-time data-তে নির্ভুল

**অসুবিধা:** নির্দিষ্ট page-এ সরাসরি যাওয়া যায় না, মোট page সংখ্যা জানা কঠিন

### 3. Keyset Pagination

Cursor pagination-এর মতোই, কিন্তু cursor encode না করে সরাসরি column value ব্যবহার করে। যখন sort column indexed থাকে তখন সবচেয়ে ভালো performance দেয়।

```sql
-- প্রথম page
SELECT id, name, price FROM products 
ORDER BY price ASC, id ASC 
LIMIT 20;

-- পরবর্তী page (শেষ record: price=500, id=42)
SELECT id, name, price FROM products 
WHERE (price > 500) OR (price = 500 AND id > 42)
ORDER BY price ASC, id ASC 
LIMIT 20;
```

---

## 🎯 বাস্তব উদাহরণ (Real-life Analogy)

**Offset Pagination → বইয়ের পৃষ্ঠা:**
একটি বইতে পৃষ্ঠা নম্বর আছে। আপনি সরাসরি পৃষ্ঠা ৩৫০-তে যেতে পারেন। কিন্তু যদি বইটি ১০ লক্ষ পৃষ্ঠার হয় এবং আপনি পৃষ্ঠা ৯,৯৯,৯৯০-এ যেতে চান — তাহলে অনেক পৃষ্ঠা উল্টাতে হবে (যেমন DB-তে row skip করতে হয়)।

**Cursor Pagination → Facebook News Feed:**
আপনি Facebook scroll করছেন। আপনি জানেন না মোট কতটি post আছে বা আপনি কোন "পৃষ্ঠায়" আছেন। আপনি শুধু scroll করেন, আর নতুন content আসতে থাকে। শেষ দেখা post-এর পরের content load হয় — এটাই cursor pagination।

**Keyset Pagination → লাইব্রেরির বই খোঁজা:**
আপনি "P" অক্ষর দিয়ে শুরু হওয়া বইগুলো দেখতে চান। আপনি সরাসরি "P" section-এ যান, সেখান থেকে শুরু করেন। ১০০০ বই skip করতে হয় না।

---

## 💻 PHP Code Example — Laravel-style Pagination

```php
<?php

// === Offset Pagination ===
class OffsetPaginator
{
    private PDO $db;

    public function __construct(PDO $db)
    {
        $this->db = $db;
    }

    public function paginate(string $table, int $page = 1, int $perPage = 20): array
    {
        $offset = ($page - 1) * $perPage;

        // মোট record সংখ্যা
        $countStmt = $this->db->query("SELECT COUNT(*) FROM {$table}");
        $total = (int) $countStmt->fetchColumn();

        // Data fetch
        $stmt = $this->db->prepare(
            "SELECT * FROM {$table} ORDER BY id LIMIT :limit OFFSET :offset"
        );
        $stmt->bindValue(':limit', $perPage, PDO::PARAM_INT);
        $stmt->bindValue(':offset', $offset, PDO::PARAM_INT);
        $stmt->execute();

        $data = $stmt->fetchAll(PDO::FETCH_ASSOC);

        return [
            'data'         => $data,
            'current_page' => $page,
            'per_page'     => $perPage,
            'total'        => $total,
            'last_page'    => (int) ceil($total / $perPage),
            'has_more'     => ($page * $perPage) < $total,
        ];
    }
}

// === Cursor Pagination ===
class CursorPaginator
{
    private PDO $db;

    public function __construct(PDO $db)
    {
        $this->db = $db;
    }

    public function paginate(
        string $table,
        ?string $cursor = null,
        int $limit = 20,
        string $sortColumn = 'created_at',
        string $direction = 'DESC'
    ): array {
        $params = [];
        $whereClause = '';

        if ($cursor !== null) {
            $decoded = $this->decodeCursor($cursor);
            $operator = $direction === 'DESC' ? '<' : '>';
            $whereClause = "WHERE ({$sortColumn}, id) {$operator} (:cursor_value, :cursor_id)";
            $params[':cursor_value'] = $decoded['value'];
            $params[':cursor_id'] = $decoded['id'];
        }

        $sql = "SELECT * FROM {$table} {$whereClause} 
                ORDER BY {$sortColumn} {$direction}, id {$direction} 
                LIMIT :limit";
        $params[':limit'] = $limit + 1; // ১টি বেশি আনি পরবর্তী page আছে কিনা জানতে

        $stmt = $this->db->prepare($sql);
        foreach ($params as $key => $value) {
            $stmt->bindValue($key, $value, is_int($value) ? PDO::PARAM_INT : PDO::PARAM_STR);
        }
        $stmt->execute();
        $data = $stmt->fetchAll(PDO::FETCH_ASSOC);

        $hasMore = count($data) > $limit;
        if ($hasMore) {
            array_pop($data); // অতিরিক্ত record বাদ দেওয়া
        }

        $lastItem = end($data);
        $nextCursor = $hasMore && $lastItem
            ? $this->encodeCursor($lastItem[$sortColumn], $lastItem['id'])
            : null;

        return [
            'data'        => $data,
            'next_cursor' => $nextCursor,
            'has_more'    => $hasMore,
        ];
    }

    private function encodeCursor(string $value, int $id): string
    {
        return base64_encode(json_encode(['value' => $value, 'id' => $id]));
    }

    private function decodeCursor(string $cursor): array
    {
        return json_decode(base64_decode($cursor), true);
    }
}

// ব্যবহার
$offsetResult = $offsetPaginator->paginate('products', page: 3, perPage: 15);
$cursorResult = $cursorPaginator->paginate('products', cursor: $request->query('cursor'));
```

---

## 🟨 JavaScript Code Example

```javascript
// === Offset Pagination ===
class OffsetPaginator {
    constructor(db) {
        this.db = db; // knex বা অন্য query builder
    }

    async paginate(table, { page = 1, perPage = 20 } = {}) {
        const offset = (page - 1) * perPage;

        const [data, [{ total }]] = await Promise.all([
            this.db(table).orderBy('id').limit(perPage).offset(offset),
            this.db(table).count('* as total'),
        ]);

        return {
            data,
            meta: {
                currentPage: page,
                perPage,
                total: Number(total),
                lastPage: Math.ceil(total / perPage),
                hasMore: page * perPage < total,
            },
        };
    }
}

// === Cursor Pagination ===
class CursorPaginator {
    constructor(db) {
        this.db = db;
    }

    async paginate(table, { cursor = null, limit = 20, sortBy = 'created_at', order = 'desc' } = {}) {
        let query = this.db(table).orderBy(sortBy, order).orderBy('id', order);

        if (cursor) {
            const decoded = this.decodeCursor(cursor);
            const op = order === 'desc' ? '<' : '>';
            query = query.whereRaw(
                `(${sortBy}, id) ${op} (?, ?)`,
                [decoded.value, decoded.id]
            );
        }

        // limit + 1 দিয়ে পরবর্তী page আছে কি না যাচাই
        const data = await query.limit(limit + 1);
        const hasMore = data.length > limit;

        if (hasMore) data.pop();

        const lastItem = data[data.length - 1];
        const nextCursor = hasMore && lastItem
            ? this.encodeCursor(lastItem[sortBy], lastItem.id)
            : null;

        return {
            data,
            pageInfo: {
                hasNextPage: hasMore,
                endCursor: nextCursor,
            },
        };
    }

    encodeCursor(value, id) {
        return Buffer.from(JSON.stringify({ value, id })).toString('base64');
    }

    decodeCursor(cursor) {
        return JSON.parse(Buffer.from(cursor, 'base64').toString('utf-8'));
    }
}

// Express Route উদাহরণ
const express = require('express');
const app = express();

app.get('/api/products', async (req, res) => {
    const { page, cursor, limit = 20 } = req.query;
    const paginator = cursor !== undefined
        ? new CursorPaginator(db)
        : new OffsetPaginator(db);

    const result = cursor !== undefined
        ? await paginator.paginate('products', { cursor, limit: Number(limit) })
        : await paginator.paginate('products', { page: Number(page || 1), perPage: Number(limit) });

    res.json(result);
});
```

---

## 📊 Performance তুলনা

```
Dataset: ১০ লক্ষ record, PostgreSQL, indexed columns

Query                              | সময় (ms) | ব্যাখ্যা
-----------------------------------+-----------+--------------------
OFFSET 0 LIMIT 20                  |    2 ms   | শুরুতে দ্রুত
OFFSET 10000 LIMIT 20              |   15 ms   | মাঝারি
OFFSET 500000 LIMIT 20             |  450 ms   | ধীর — ৫ লক্ষ row skip
OFFSET 999980 LIMIT 20             |  890 ms   | অত্যন্ত ধীর
                                    |           |
Cursor (WHERE id > 999980 LIMIT 20)|    2 ms   | সবসময় দ্রুত! ✅
Keyset (WHERE (price,id) > (...))  |    3 ms   | Index থাকলে দ্রুত ✅
```

```
Performance গ্রাফ (Response Time vs Page Number):

সময়  │
(ms)  │
900   │                                          ╱ Offset
      │                                        ╱
600   │                                     ╱
      │                                  ╱
300   │                             ╱
      │                        ╱
100   │               ╱──
 10   │    ╱──────
  2   │──── ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ Cursor/Keyset
      └─────────────────────────────────────────────
      Page 1     100     1K      10K     50K    Page
```

---

## ✅ কখন ব্যবহার করবেন

| Strategy | ব্যবহারের ক্ষেত্র |
|----------|-------------------|
| **Offset** | Admin panel, ছোট dataset (<১ লক্ষ), page number দেখাতে হলে, pagination UI-তে "Go to page X" দরকার হলে |
| **Cursor** | Infinite scroll, mobile app feed, real-time data (social media), বড় dataset (>১ লক্ষ), API consumer-দের জন্য |
| **Keyset** | Analytics dashboard, time-series data, sorted data browsing, যেখানে sort column indexed আছে |

## ❌ কখন ব্যবহার করবেন না

| Strategy | এড়িয়ে চলুন যখন |
|----------|-----------------|
| **Offset** | Dataset লক্ষাধিক record, real-time data যেখানে ঘন ঘন insert/delete হয়, performance critical API |
| **Cursor** | ব্যবহারকারীকে নির্দিষ্ট page-এ যেতে হলে, মোট page/record সংখ্যা দেখাতে হলে, random access দরকার হলে |
| **Keyset** | Complex sorting (multiple dynamic columns), sort column-এ index না থাকলে, client-কে total count দিতে হলে |

---

## 🔗 API Response Format Best Practice

```json
// Offset Pagination Response
{
    "data": [...],
    "meta": {
        "current_page": 3,
        "per_page": 20,
        "total": 15420,
        "last_page": 771,
        "from": 41,
        "to": 60
    },
    "links": {
        "first": "/api/products?page=1",
        "prev": "/api/products?page=2",
        "next": "/api/products?page=4",
        "last": "/api/products?page=771"
    }
}

// Cursor Pagination Response
{
    "data": [...],
    "page_info": {
        "has_next_page": true,
        "has_previous_page": true,
        "start_cursor": "eyJ2YWx1ZSI6...",
        "end_cursor": "eyJ2YWx1ZSI6..."
    }
}
```

> 💡 **Senior Engineer Tip:** বেশিরভাগ production API-তে cursor pagination ব্যবহার করুন। Offset pagination শুধুমাত্র তখনই ব্যবহার করুন যখন page number দেখানো business requirement হয়। বড় dataset-এ offset pagination ব্যবহার করলে `COUNT(*)` query-ও expensive হতে পারে — সেক্ষেত্রে approximate count (যেমন PostgreSQL-এর `reltuples`) বিবেচনা করুন।
