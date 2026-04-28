# 🔍 কোয়েরি এক্সিকিউশন প্ল্যান (Query Execution Plan)

## 📌 সংজ্ঞা ও মূল ধারণা

### Execution Plan কী?

Query execution plan হলো database engine-এর তৈরি একটি **step-by-step blueprint** যা দেখায় কীভাবে একটি SQL query execute হবে — কোন index ব্যবহার হবে, কোন join algorithm প্রয়োগ হবে, কত row scan হবে, এবং আনুমানিক খরচ (cost) কত হবে।

**EXPLAIN** command ব্যবহার করে এই plan দেখা যায়। এটি query optimization-এর সবচেয়ে গুরুত্বপূর্ণ হাতিয়ার — query কেন ধীর সেটা বুঝতে execution plan পড়া আবশ্যক।

### EXPLAIN vs EXPLAIN ANALYZE

| Command | কী করে |
|---|---|
| **EXPLAIN** | Query execute না করেই **আনুমানিক** plan দেখায় (estimated cost, rows) |
| **EXPLAIN ANALYZE** | Query **আসলে execute করে** actual time, actual rows দেখায় |

```sql
-- শুধু plan দেখা (query execute হয় না)
EXPLAIN SELECT * FROM users WHERE email = 'rahim@example.com';

-- Query execute করে actual performance data সহ plan
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'rahim@example.com';

-- MySQL-এ বিস্তারিত format
EXPLAIN FORMAT=JSON SELECT * FROM users WHERE email = 'rahim@example.com';

-- PostgreSQL-এ বিস্তারিত format
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT * FROM users WHERE email = 'rahim@example.com';
```

---

## 🏠 বাস্তব উদাহরণ

**Google Maps-এ গন্তব্যে পৌঁছানোর route:**

আপনি ঢাকা থেকে চট্টগ্রাম যাবেন। Google Maps বিভিন্ন route দেখায়:
- **Route 1**: ঢাকা-চট্টগ্রাম মহাসড়ক → 5 ঘণ্টা (সরাসরি, কিন্তু traffic বেশি)
- **Route 2**: কুমিল্লা হয়ে → 6 ঘণ্টা (ঘুরপথ, কিন্তু traffic কম)
- **Route 3**: ফেরি দিয়ে → 7 ঘণ্টা (সবচেয়ে ধীর)

Google Maps **সবচেয়ে দ্রুত route** বেছে নেয়। Database optimizer-ও ঠিক এভাবে কাজ করে — সম্ভাব্য সব execution plan বিবেচনা করে সবচেয়ে কম cost-এর plan নির্বাচন করে।

- **Route** = Execution plan
- **ঘণ্টা** = Estimated cost
- **Traffic data** = Table statistics (row count, index cardinality)
- **Google Maps optimizer** = Query optimizer

---

## 📖 Execution Plan পড়া (MySQL)

### MySQL EXPLAIN Output

```sql
EXPLAIN SELECT u.name, o.total
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.status = 'active'
AND o.created_at > '2024-01-01';
```

```
┌────┬─────────────┬───────┬──────┬─────────────┬──────────┬─────────┬──────────┬──────┬─────────────────────────┐
│ id │ select_type │ table │ type │ possible    │ key      │ key_len │ ref      │ rows │ Extra                   │
│    │             │       │      │ _keys       │          │         │          │      │                         │
├────┼─────────────┼───────┼──────┼─────────────┼──────────┼─────────┼──────────┼──────┼─────────────────────────┤
│  1 │ SIMPLE      │ u     │ ref  │ PRIMARY,    │ idx_     │ 4       │ const    │  500 │ Using where             │
│    │             │       │      │ idx_status  │ status   │         │          │      │                         │
│  1 │ SIMPLE      │ o     │ ref  │ idx_user_id │ idx_     │ 4       │ app.u.id │    3 │ Using where; Using index│
│    │             │       │      │             │ user_id  │         │          │      │                         │
└────┴─────────────┴───────┴──────┴─────────────┴──────────┴─────────┴──────────┴──────┴─────────────────────────┘
```

### গুরুত্বপূর্ণ Column ব্যাখ্যা

| Column | ব্যাখ্যা | ভালো/খারাপ |
|---|---|---|
| **type** | Access method | `system` > `const` > `eq_ref` > `ref` > `range` > `index` > `ALL` |
| **key** | ব্যবহৃত index | NULL হলে ❌ কোনো index ব্যবহার হচ্ছে না |
| **rows** | আনুমানিক scan হবে কত row | কম = ভালো |
| **Extra** | অতিরিক্ত তথ্য | "Using index" ✅ / "Using filesort" ⚠️ / "Using temporary" ⚠️ |

### type Column-এর মান (ভালো → খারাপ)

```
┌─────────────────────────────────────────────────────────┐
│              Access Type (ভালো → খারাপ)                 │
│                                                         │
│  const    ──► Primary key / unique index lookup          │
│              "WHERE id = 5" (1 row)                     │
│                                                         │
│  eq_ref   ──► JOIN-এ unique index match                 │
│              "JOIN users ON orders.user_id = users.id"  │
│                                                         │
│  ref      ──► Non-unique index lookup                   │
│              "WHERE status = 'active'" (multiple rows)  │
│                                                         │
│  range    ──► Index range scan                          │
│              "WHERE age BETWEEN 20 AND 30"              │
│                                                         │
│  index    ──► Full index scan (সব index entry পড়া)     │
│              ⚠️ Table scan-এর চেয়ে সামান্য ভালো        │
│                                                         │
│  ALL      ──► Full table scan (সব row পড়া)             │
│              ❌ সবচেয়ে ধীর, index নেই                   │
└─────────────────────────────────────────────────────────┘
```

---

## 📖 Execution Plan পড়া (PostgreSQL)

### PostgreSQL EXPLAIN ANALYZE Output

```sql
EXPLAIN ANALYZE
SELECT u.name, o.total
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.status = 'active'
AND o.created_at > '2024-01-01';
```

```
Hash Join  (cost=125.00..450.25 rows=500 width=36) (actual time=1.2..5.8 rows=487 loops=1)
  Hash Cond: (o.user_id = u.id)
  ->  Seq Scan on orders o  (cost=0.00..280.00 rows=3000 width=16) (actual time=0.01..2.1 rows=2876 loops=1)
        Filter: (created_at > '2024-01-01')
        Rows Removed by Filter: 7124
  ->  Hash  (cost=100.00..100.00 rows=2000 width=24) (actual time=0.8..0.8 rows=1950 loops=1)
        Buckets: 2048  Batches: 1  Memory Usage: 120kB
        ->  Index Scan using idx_users_status on users u  (cost=0.29..100.00 rows=2000 width=24)
              Index Cond: (status = 'active')
Planning Time: 0.15 ms
Execution Time: 6.2 ms
```

### গুরুত্বপূর্ণ Node Types

| Node Type | ব্যাখ্যা | Performance |
|---|---|---|
| **Seq Scan** | Full table scan | ❌ বড় table-এ ধীর |
| **Index Scan** | Index ব্যবহার করে row fetch | ✅ ভালো |
| **Index Only Scan** | শুধু index থেকেই data পাওয়া (covering index) | ✅✅ সবচেয়ে ভালো |
| **Bitmap Index Scan** | Index থেকে matching page ID সংগ্রহ, তারপর একসাথে fetch | ✅ মাঝামাঝি |
| **Nested Loop** | প্রতিটি outer row-এর জন্য inner table scan | ছোট dataset-এ ✅ |
| **Hash Join** | Inner table-এর hash table তৈরি করে match | মাঝারি/বড় dataset-এ ✅ |
| **Merge Join** | দুই sorted input merge করে | বড় sorted dataset-এ ✅ |
| **Sort** | Result sort করা | ⚠️ Memory ব্যবহার করে |

### Cost পড়া

```
Index Scan  (cost=0.29..100.00 rows=2000 width=24)
                    │       │      │        │
                    │       │      │        └─ প্রতি row-এর size (bytes)
                    │       │      └─ আনুমানিক output row সংখ্যা
                    │       └─ মোট cost (সব row fetch করতে)
                    └─ startup cost (প্রথম row পেতে)
```

---

## 🕵️ Index ব্যবহার Detect করা

### Index ব্যবহার হচ্ছে না — সাধারণ কারণ

```sql
-- ❌ কারণ 1: Column-এ function ব্যবহার (index ignore হয়)
SELECT * FROM users WHERE YEAR(created_at) = 2024;
-- ✅ সমাধান: Range query ব্যবহার করুন
SELECT * FROM users WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';

-- ❌ কারণ 2: Implicit type casting
SELECT * FROM users WHERE phone = 01712345678;  -- phone VARCHAR, কিন্তু number দেওয়া
-- ✅ সমাধান: সঠিক type দিন
SELECT * FROM users WHERE phone = '01712345678';

-- ❌ কারণ 3: LIKE-এ leading wildcard
SELECT * FROM products WHERE name LIKE '%phone%';
-- ✅ সমাধান: Full-text search ব্যবহার করুন
SELECT * FROM products WHERE MATCH(name) AGAINST('phone');

-- ❌ কারণ 4: OR condition-এ mixed indexed/non-indexed column
SELECT * FROM users WHERE email = 'x@y.com' OR age > 25;
-- ✅ সমাধান: UNION ব্যবহার করুন
SELECT * FROM users WHERE email = 'x@y.com'
UNION
SELECT * FROM users WHERE age > 25;

-- ❌ কারণ 5: NOT IN / NOT EXISTS (index ব্যবহার সীমিত)
SELECT * FROM users WHERE id NOT IN (SELECT user_id FROM blocked_users);
-- ✅ সমাধান: LEFT JOIN ব্যবহার করুন
SELECT u.* FROM users u
LEFT JOIN blocked_users b ON u.id = b.user_id
WHERE b.user_id IS NULL;
```

---

## 💻 PHP Code Example — PDO দিয়ে EXPLAIN ব্যবহার

```php
<?php

class QueryAnalyzer
{
    private PDO $pdo;

    public function __construct(PDO $pdo)
    {
        $this->pdo = $pdo;
    }

    /**
     * MySQL query-র execution plan বিশ্লেষণ
     */
    public function explain(string $sql, array $params = []): array
    {
        $stmt = $this->pdo->prepare("EXPLAIN " . $sql);
        $stmt->execute($params);
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }

    /**
     * JSON format-এ বিস্তারিত plan
     */
    public function explainJson(string $sql, array $params = []): array
    {
        $stmt = $this->pdo->prepare("EXPLAIN FORMAT=JSON " . $sql);
        $stmt->execute($params);
        $result = $stmt->fetch(PDO::FETCH_ASSOC);
        return json_decode($result['EXPLAIN'], true);
    }

    /**
     * EXPLAIN ANALYZE — actual execution data (MySQL 8.0.18+)
     */
    public function explainAnalyze(string $sql, array $params = []): string
    {
        $stmt = $this->pdo->prepare("EXPLAIN ANALYZE " . $sql);
        $stmt->execute($params);
        $result = $stmt->fetch(PDO::FETCH_NUM);
        return $result[0];
    }

    /**
     * Slow query সনাক্তকরণ — execution plan-এ সমস্যা খুঁজুন
     */
    public function detectIssues(string $sql, array $params = []): array
    {
        $plans = $this->explain($sql, $params);
        $issues = [];

        foreach ($plans as $plan) {
            // Full table scan detect
            if ($plan['type'] === 'ALL') {
                $issues[] = [
                    'severity' => 'HIGH',
                    'table' => $plan['table'],
                    'issue' => "Full table scan (type=ALL) — rows: {$plan['rows']}",
                    'suggestion' => "WHERE clause-এর column-এ index যোগ করুন",
                ];
            }

            // Index ব্যবহার হচ্ছে না
            if (empty($plan['key'])) {
                $issues[] = [
                    'severity' => 'HIGH',
                    'table' => $plan['table'],
                    'issue' => "No index used (key=NULL)",
                    'suggestion' => "possible_keys: {$plan['possible_keys']} — সঠিক index তৈরি করুন",
                ];
            }

            // Filesort ব্যবহার (ORDER BY-এ index নেই)
            if (str_contains($plan['Extra'] ?? '', 'Using filesort')) {
                $issues[] = [
                    'severity' => 'MEDIUM',
                    'table' => $plan['table'],
                    'issue' => "Using filesort — ORDER BY clause-এ index নেই",
                    'suggestion' => "ORDER BY column-এ composite index যোগ করুন",
                ];
            }

            // Temporary table ব্যবহার (GROUP BY / DISTINCT)
            if (str_contains($plan['Extra'] ?? '', 'Using temporary')) {
                $issues[] = [
                    'severity' => 'MEDIUM',
                    'table' => $plan['table'],
                    'issue' => "Using temporary table — GROUP BY / DISTINCT optimize করুন",
                    'suggestion' => "GROUP BY column-এ index যোগ করুন বা query restructure করুন",
                ];
            }

            // অতিরিক্ত row scan
            if ((int) $plan['rows'] > 100000) {
                $issues[] = [
                    'severity' => 'WARNING',
                    'table' => $plan['table'],
                    'issue' => "High row estimate: {$plan['rows']} rows scanned",
                    'suggestion' => "আরো selective WHERE condition বা index ব্যবহার করুন",
                ];
            }
        }

        return $issues;
    }
}

// ব্যবহার
$pdo = new PDO('mysql:host=127.0.0.1;dbname=myapp;charset=utf8mb4', 'root', 'secret');
$analyzer = new QueryAnalyzer($pdo);

// Execution plan দেখা
$sql = "SELECT u.name, COUNT(o.id) as order_count
        FROM users u
        LEFT JOIN orders o ON u.id = o.user_id
        WHERE u.created_at > :date
        GROUP BY u.id
        ORDER BY order_count DESC
        LIMIT 10";

$plan = $analyzer->explain($sql, ['date' => '2024-01-01']);
print_r($plan);

// সমস্যা সনাক্তকরণ
$issues = $analyzer->detectIssues($sql, ['date' => '2024-01-01']);
foreach ($issues as $issue) {
    echo "[{$issue['severity']}] {$issue['table']}: {$issue['issue']}\n";
    echo "  → {$issue['suggestion']}\n\n";
}
```

---

## 💻 JavaScript Code Example — pg ও mysql2 দিয়ে EXPLAIN

```javascript
const { Pool } = require('pg');
const mysql = require('mysql2/promise');

// ═══════════════════════════════════════════
// PostgreSQL EXPLAIN Analyzer
// ═══════════════════════════════════════════
class PgQueryAnalyzer {
    constructor(pool) {
        this.pool = pool;
    }

    /**
     * EXPLAIN ANALYZE — actual execution data
     */
    async explainAnalyze(sql, params = []) {
        const explainSql = `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) ${sql}`;
        const result = await this.pool.query(explainSql, params);
        return result.rows[0]['QUERY PLAN'][0];
    }

    /**
     * EXPLAIN only — estimated plan (query execute হয় না)
     */
    async explain(sql, params = []) {
        const result = await this.pool.query(`EXPLAIN (FORMAT JSON) ${sql}`, params);
        return result.rows[0]['QUERY PLAN'][0];
    }

    /**
     * Plan বিশ্লেষণ করে সমস্যা খুঁজুন
     */
    async analyzeQuery(sql, params = []) {
        const plan = await this.explainAnalyze(sql, params);
        const issues = [];

        this._walkPlanTree(plan.Plan, issues);

        return {
            executionTimeMs: plan['Execution Time'],
            planningTimeMs: plan['Planning Time'],
            totalCost: plan.Plan['Total Cost'],
            issues,
        };
    }

    _walkPlanTree(node, issues, depth = 0) {
        // Seq Scan detect (Full table scan)
        if (node['Node Type'] === 'Seq Scan' && node['Actual Rows'] > 10000) {
            issues.push({
                severity: 'HIGH',
                nodeType: node['Node Type'],
                table: node['Relation Name'],
                rows: node['Actual Rows'],
                issue: `Sequential Scan on ${node['Relation Name']} — ${node['Actual Rows']} rows`,
                suggestion: `Filter column-এ index তৈরি করুন: ${node['Filter'] || 'N/A'}`,
            });
        }

        // Sort node detect (memory-intensive)
        if (node['Node Type'] === 'Sort' && node['Sort Method'] === 'external merge') {
            issues.push({
                severity: 'MEDIUM',
                nodeType: 'Sort',
                issue: 'External merge sort — memory-তে ধরেনি, disk ব্যবহার হচ্ছে',
                suggestion: 'work_mem বাড়ান বা ORDER BY column-এ index দিন',
            });
        }

        // Nested Loop-এ বেশি loops
        if (node['Node Type'] === 'Nested Loop' && node['Actual Loops'] > 1000) {
            issues.push({
                severity: 'WARNING',
                nodeType: 'Nested Loop',
                loops: node['Actual Loops'],
                issue: `Nested Loop-এ ${node['Actual Loops']} loops — JOIN optimize করুন`,
                suggestion: 'JOIN column-এ index দিন, বা Hash Join-এ force করুন',
            });
        }

        // Recursively analyze child nodes
        if (node.Plans) {
            for (const child of node.Plans) {
                this._walkPlanTree(child, issues, depth + 1);
            }
        }
    }
}

// ═══════════════════════════════════════════
// MySQL EXPLAIN Analyzer
// ═══════════════════════════════════════════
class MySqlQueryAnalyzer {
    constructor(pool) {
        this.pool = pool;
    }

    async explain(sql, params = []) {
        const [rows] = await this.pool.execute(`EXPLAIN ${sql}`, params);
        return rows;
    }

    async explainJson(sql, params = []) {
        const [rows] = await this.pool.execute(`EXPLAIN FORMAT=JSON ${sql}`, params);
        return JSON.parse(rows[0].EXPLAIN);
    }

    async analyzeQuery(sql, params = []) {
        const plans = await this.explain(sql, params);
        const issues = [];

        for (const plan of plans) {
            if (plan.type === 'ALL') {
                issues.push({
                    severity: 'HIGH',
                    table: plan.table,
                    issue: `Full table scan — ${plan.rows} rows estimated`,
                    suggestion: `${plan.table} table-এ WHERE column-এ index দিন`,
                });
            }

            if (!plan.key && plan.possible_keys) {
                issues.push({
                    severity: 'MEDIUM',
                    table: plan.table,
                    issue: 'Index আছে কিন্তু ব্যবহার হচ্ছে না',
                    suggestion: `FORCE INDEX (${plan.possible_keys}) চেষ্টা করুন বা query restructure করুন`,
                });
            }
        }

        return { plans, issues };
    }
}

// ═══════════════════════════════════════════
// ব্যবহার
// ═══════════════════════════════════════════
(async () => {
    // PostgreSQL
    const pgPool = new Pool({
        host: '127.0.0.1',
        database: 'myapp',
        user: 'app',
        password: 'secret',
    });

    const pgAnalyzer = new PgQueryAnalyzer(pgPool);

    const pgResult = await pgAnalyzer.analyzeQuery(
        `SELECT u.name, COUNT(o.id)
         FROM users u
         LEFT JOIN orders o ON u.id = o.user_id
         WHERE u.created_at > $1
         GROUP BY u.id
         ORDER BY COUNT(o.id) DESC
         LIMIT 10`,
        ['2024-01-01']
    );

    console.log(`Execution: ${pgResult.executionTimeMs}ms`);
    console.log(`Issues: ${pgResult.issues.length}`);
    pgResult.issues.forEach(i =>
        console.log(`  [${i.severity}] ${i.issue}\n    → ${i.suggestion}`)
    );

    // MySQL
    const mysqlPool = await mysql.createPool({
        host: '127.0.0.1',
        database: 'myapp',
        user: 'root',
        password: 'secret',
    });

    const mysqlAnalyzer = new MySqlQueryAnalyzer(mysqlPool);
    const mysqlResult = await mysqlAnalyzer.analyzeQuery(
        'SELECT * FROM users WHERE status = ? AND created_at > ?',
        ['active', '2024-01-01']
    );

    mysqlResult.issues.forEach(i =>
        console.log(`  [${i.severity}] ${i.table}: ${i.issue}`)
    );

    await pgPool.end();
    await mysqlPool.end();
})();
```

---

## 🛠️ Optimization Tips

### Index ভিত্তিক Optimization

```sql
-- 1. Composite index: WHERE + ORDER BY একসাথে cover করুন
-- Query: WHERE status = 'active' ORDER BY created_at DESC
CREATE INDEX idx_status_created ON users (status, created_at DESC);

-- 2. Covering index: SELECT-এর column-ও index-এ রাখুন
-- Query: SELECT name, email FROM users WHERE status = 'active'
CREATE INDEX idx_covering ON users (status, name, email);
-- Extra: "Using index" দেখাবে (table access প্রয়োজন নেই)

-- 3. Partial index (PostgreSQL): শুধু প্রয়োজনীয় row index করুন
CREATE INDEX idx_active_users ON users (email) WHERE status = 'active';

-- 4. Index-এর effectiveness যাচাই
-- MySQL
SELECT * FROM sys.schema_unused_indexes;    -- অব্যবহৃত index
SELECT * FROM sys.schema_redundant_indexes; -- redundant index

-- PostgreSQL
SELECT indexrelname, idx_scan FROM pg_stat_user_indexes
WHERE idx_scan = 0 ORDER BY pg_relation_size(indexrelid) DESC;
```

---

## ✅ কখন ব্যবহার করবেন

- **Slow query debugging**: কোনো query ধীর হলে EXPLAIN দিয়ে কারণ খুঁজুন
- **Index verification**: নতুন index তৈরির পর EXPLAIN দিয়ে ব্যবহার হচ্ছে কিনা যাচাই
- **Query optimization**: বিভিন্ন query variation-এর মধ্যে কোনটি দ্রুত তা তুলনা
- **Code review**: PR-এ নতুন query-র সাথে EXPLAIN output attach করা best practice
- **Performance monitoring**: Production-এ slow query log + EXPLAIN সংযুক্ত করে periodic audit

## ❌ কখন ব্যবহার করবেন না

- **EXPLAIN ANALYZE production-এ সাবধানে**: এটি query **আসলে execute করে**, write query (UPDATE/DELETE)-এ transaction-এ wrap করুন
- **Micro-optimization**: 1ms vs 2ms query নিয়ে সময় নষ্ট করবেন না, bottleneck খুঁজুন
- **Small dataset**: ১০০ row-এর table-এ full scan vs index scan তেমন পার্থক্য নেই
- **Constantly changing data**: Statistics outdated হলে EXPLAIN-এর estimate ভুল হতে পারে — `ANALYZE` table চালান আগে

---

## 🔑 মনে রাখার বিষয়

1. **EXPLAIN প্রথমে, optimize পরে**: আন্দাজে index দেবেন না, plan দেখে সিদ্ধান্ত নিন
2. **Actual vs Estimated**: EXPLAIN ANALYZE-এর actual rows ও estimated rows-এ বড় পার্থক্য থাকলে `ANALYZE` table চালান
3. **Query cost is relative**: Cost সংখ্যা database-specific, এক database-র cost অন্যটির সাথে তুলনা করা যায় না
4. **Index সব সমাধান নয়**: কখনো query restructure, denormalization, বা caching ভালো সমাধান
5. **Production monitoring**: Slow query log সক্রিয় রাখুন এবং নিয়মিত EXPLAIN review করুন
