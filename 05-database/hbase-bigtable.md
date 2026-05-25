# 📊 HBase ও Google Bigtable — Wide-Column Store Deep Dive

> **"Bigtable হলো Google-এর distributed storage-এর foundation — HBase হলো এর open-source incarnation।"**
>
> Google Search indexing, Gmail, YouTube analytics, Facebook Messenger — সব wide-column store ব্যবহার করে।

---

## 📌 Google Bigtable — The Origin

### Bigtable কী?

Google Bigtable হলো Google-এর internal **distributed, sorted, multi-dimensional map**। ২০০৬ সালে published paper ("Bigtable: A Distributed Storage System for Structured Data") distributed database design-এ revolution এনেছিল।

### Bigtable Data Model

```
┌──────────────────────────────────────────────────────────────┐
│  Bigtable = Sparse, Distributed, Persistent, Multi-dimensional│
│             Sorted Map                                         │
│                                                               │
│  (row_key, column_family:qualifier, timestamp) → value        │
│                                                               │
│  Example — Web Page Index:                                    │
│                                                               │
│  Row Key: "com.google.www" (reversed URL for locality)       │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ Row Key          │ contents:html    │ anchor:cnn.com    │  │
│  │                  │ (column family)  │ (column family)   │  │
│  ├──────────────────┼──────────────────┼───────────────────┤  │
│  │ com.google.www   │ t5: "<html>..."  │ t9: "Google"      │  │
│  │                  │ t3: "<html>..."  │ t8: "Search"      │  │
│  │                  │ (versions)       │ (versions)        │  │
│  └──────────────────┴──────────────────┴───────────────────┘  │
│                                                               │
│  ✅ Sparse: প্রতিটি row-তে সব column থাকতে হবে না            │
│  ✅ Sorted: row key lexicographically sorted                  │
│  ✅ Versioned: প্রতিটি cell-এ multiple timestamps             │
└──────────────────────────────────────────────────────────────┘
```

### Bigtable Architecture

```
┌──────────────────────────────────────────────────────────────┐
│              Google Bigtable Architecture                      │
│                                                               │
│  ┌────────────────────────────────────────────────────┐      │
│  │                 Client Library                       │      │
│  │   Caches tablet locations, batches mutations        │      │
│  └───────────────────────┬────────────────────────────┘      │
│                          │                                    │
│  ┌───────────────────────▼────────────────────────────┐      │
│  │              Master Server                          │      │
│  │   - Tablet assignment to tablet servers             │      │
│  │   - Load balancing                                  │      │
│  │   - GC of files in GFS                             │      │
│  │   - Schema changes (column family creation)         │      │
│  └───────────────────────┬────────────────────────────┘      │
│                          │                                    │
│  ┌───────────────────────▼────────────────────────────┐      │
│  │              Tablet Servers (×1000s)                │      │
│  │                                                     │      │
│  │   ┌──────────┐  ┌──────────┐  ┌──────────┐         │      │
│  │   │Tablet Srv│  │Tablet Srv│  │Tablet Srv│         │      │
│  │   │  1       │  │  2       │  │  3       │         │      │
│  │   │ 10-20    │  │ 10-20    │  │ 10-20    │         │      │
│  │   │ tablets  │  │ tablets  │  │ tablets  │         │      │
│  │   └──────────┘  └──────────┘  └──────────┘         │      │
│  └────────────────────────┬───────────────────────────┘      │
│                           │                                   │
│  ┌────────────────────────▼───────────────────────────┐      │
│  │         GFS (Google File System) / Colossus         │      │
│  │   - SSTable files stored here                       │      │
│  │   - Replication handled at storage layer            │      │
│  └────────────────────────────────────────────────────┘      │
│                                                               │
│  ┌────────────────────────────────────────────────────┐      │
│  │              Chubby (Distributed Lock)               │      │
│  │   - Tablet server liveness                          │      │
│  │   - Master election                                 │      │
│  │   - Schema storage                                  │      │
│  └────────────────────────────────────────────────────┘      │
└──────────────────────────────────────────────────────────────┘
```

### Tablet — Basic Unit

```
┌──────────────────────────────────────────────────────────────┐
│                   Tablet Structure                             │
│                                                               │
│  Tablet = একটি table-এর contiguous row range                 │
│  Example: rows "aaa" to "czz" = one tablet                   │
│                                                               │
│  ┌──────────────────────────────────────────────┐            │
│  │  Memtable (in-memory, sorted)                 │            │
│  │   recent writes buffered here                 │            │
│  └──────────────────────┬───────────────────────┘            │
│                         │ flush when full                     │
│                         ▼                                     │
│  ┌──────────────────────────────────────────────┐            │
│  │  SSTable files (on GFS, immutable)            │            │
│  │  ┌────────┐  ┌────────┐  ┌────────┐          │            │
│  │  │SSTable1│  │SSTable2│  │SSTable3│          │            │
│  │  │(older) │  │        │  │(newer) │          │            │
│  │  └────────┘  └────────┘  └────────┘          │            │
│  │                                               │            │
│  │  Periodic compaction: merge SSTables          │            │
│  └──────────────────────────────────────────────┘            │
│                                                               │
│  Read path: Memtable → SSTable (newest first) → merge        │
│  Write path: WAL (commit log) → Memtable                     │
└──────────────────────────────────────────────────────────────┘
```

### Google Cloud Bigtable (Managed Service)

আজ Google Cloud Bigtable হিসেবে managed service পাওয়া যায়:

| বৈশিষ্ট্য | মান |
|-----------|-----|
| **Latency** | Single-digit ms (P99 < 10ms) |
| **Throughput** | 10K+ reads/writes per node per second |
| **Storage** | Petabytes |
| **Nodes** | 3 to 1000+ |
| **Use cases** | IoT time-series, financial data, analytics, ML features |
| **Integration** | BigQuery, Dataflow, Spark, HBase API compatible |

---

## 📌 Apache HBase — Open-Source Bigtable

### HBase কী?

HBase হলো Bigtable paper-এর open-source implementation, Hadoop ecosystem-এর অংশ। HDFS-এর ওপর built — random, real-time read/write access দেয় big data-তে।

### কেন HBase?

```
┌──────────────────────────────────────────────────────────────┐
│  HDFS alone:                                                  │
│    ✅ Huge sequential reads/writes (batch processing)        │
│    ❌ Random access (single row lookup) — অসম্ভব            │
│    ❌ Low-latency point queries — designed for batch         │
│                                                               │
│  HBase on HDFS:                                              │
│    ✅ Random read/write on top of HDFS                       │
│    ✅ Millisecond latency for point lookups                  │
│    ✅ Auto-sharding (region splitting)                       │
│    ✅ Strong consistency (single row)                        │
│    ✅ Billions of rows × millions of columns × versions      │
└──────────────────────────────────────────────────────────────┘
```

### HBase Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                  HBase Architecture                            │
│                                                               │
│  ┌────────────────────────────────────────────────────┐      │
│  │                 Client                              │      │
│  │   HBase client → ZooKeeper → Region Server         │      │
│  └───────────────────────┬────────────────────────────┘      │
│                          │                                    │
│  ┌───────────────────────▼────────────────────────────┐      │
│  │              HMaster                                │      │
│  │   - Region assignment                               │      │
│  │   - Load balancing (region moves)                   │      │
│  │   - DDL operations (create/delete table)            │      │
│  │   - Failover: standby HMaster takes over           │      │
│  └───────────────────────┬────────────────────────────┘      │
│                          │                                    │
│  ┌───────────────────────▼────────────────────────────┐      │
│  │         ZooKeeper Ensemble (3/5 nodes)              │      │
│  │   - Region Server liveness (ephemeral nodes)        │      │
│  │   - HMaster election                               │      │
│  │   - META table location                            │      │
│  │   - Cluster configuration                          │      │
│  └───────────────────────┬────────────────────────────┘      │
│                          │                                    │
│  ┌───────────────────────▼────────────────────────────┐      │
│  │          Region Servers (DataNodes)                  │      │
│  │                                                     │      │
│  │   ┌─────────────────────────────────────────┐       │      │
│  │   │  Region Server 1                        │       │      │
│  │   │                                         │       │      │
│  │   │  ┌─────────┐  ┌─────────┐  ┌────────┐  │       │      │
│  │   │  │ Region A │  │Region B │  │Region C│  │       │      │
│  │   │  │(rows a-d)│  │(rows e-k)│  │(rows l-p)│       │      │
│  │   │  └─────────┘  └─────────┘  └────────┘  │       │      │
│  │   │                                         │       │      │
│  │   │  ┌─────────────────────────────────┐    │       │      │
│  │   │  │  Block Cache (read cache, LRU)  │    │       │      │
│  │   │  └─────────────────────────────────┘    │       │      │
│  │   │                                         │       │      │
│  │   │  ┌─────────────────────────────────┐    │       │      │
│  │   │  │  WAL (Write Ahead Log)          │    │       │      │
│  │   │  └─────────────────────────────────┘    │       │      │
│  │   └─────────────────────────────────────────┘       │      │
│  └─────────────────────────────────────────────────────┘      │
│                          │                                    │
│  ┌───────────────────────▼────────────────────────────┐      │
│  │                    HDFS                             │      │
│  │   HFiles (SSTables), WAL files stored here         │      │
│  │   3× replication at storage layer                   │      │
│  └────────────────────────────────────────────────────┘      │
└──────────────────────────────────────────────────────────────┘
```

### Region — HBase-এর Basic Unit

```
┌──────────────────────────────────────────────────────────────┐
│                   Region Internal Structure                    │
│                                                               │
│  Region = Table-এর একটি contiguous row range                 │
│           (Bigtable-এর Tablet-এর equivalent)                  │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  MemStore (per column family, in-memory)              │    │
│  │   - Write buffer                                      │    │
│  │   - Sorted by row key                                 │    │
│  │   - Flush to HFile when size threshold hit (128MB)    │    │
│  └─────────────────────────┬────────────────────────────┘    │
│                            │ flush                            │
│                            ▼                                  │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  HFiles (on HDFS, immutable)                          │    │
│  │  ┌────────┐  ┌────────┐  ┌────────┐                  │    │
│  │  │HFile 1 │  │HFile 2 │  │HFile 3 │                  │    │
│  │  └────────┘  └────────┘  └────────┘                  │    │
│  │                                                       │    │
│  │  Compaction:                                          │    │
│  │    Minor: ছোট HFiles merge → বড় HFile                │    │
│  │    Major: সব HFiles merge + delete markers clean      │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                               │
│  Read Path:                                                  │
│    Block Cache → MemStore → HFiles (bloom filter check)      │
│                                                               │
│  Write Path:                                                 │
│    Client → WAL (durability) → MemStore (speed)              │
└──────────────────────────────────────────────────────────────┘
```

### Auto-Region Splitting

```
┌──────────────────────────────────────────────────────────────┐
│               Region Splitting                                │
│                                                               │
│  Before split:                                               │
│    Region 1: rows "a" to "z" (10GB — threshold hit!)        │
│                                                               │
│  After split:                                                │
│    Region 1a: rows "a" to "m" (5GB) → RegionServer 1        │
│    Region 1b: rows "n" to "z" (5GB) → RegionServer 2        │
│                                                               │
│  Split is automatic — no manual sharding!                    │
│  HMaster rebalances regions across servers                   │
└──────────────────────────────────────────────────────────────┘
```

---

## 📊 Bigtable vs HBase তুলনা

| বৈশিষ্ট্য | Google Bigtable | Apache HBase |
|-----------|----------------|--------------|
| **Type** | Managed service (GCP) | Open-source, self-managed |
| **Storage** | Colossus (Google FS) | HDFS |
| **Coordination** | Chubby | ZooKeeper |
| **Operations** | Google manages | আপনাকে manage করতে হবে |
| **Cost** | Pay-per-use | Infrastructure cost |
| **Scaling** | Node add/remove (minutes) | Manual region balancing |
| **Consistency** | Strong (single row) | Strong (single row) |
| **Multi-row txn** | ❌ | ❌ (Phoenix দিয়ে limited) |
| **API** | gRPC, HBase-compatible | Java API, REST, Thrift |
| **Ecosystem** | BigQuery, Dataflow | Hadoop, Spark, Hive |

---

## 💻 HBase PHP Example (via Thrift/REST)

```php
<?php
// HBase REST API client (since PHP doesn't have native HBase client)

class HBaseClient
{
    private string $baseUrl;
    private \GuzzleHttp\Client $http;

    public function __construct(string $host = 'localhost', int $port = 8080)
    {
        $this->baseUrl = "http://{$host}:{$port}";
        $this->http = new \GuzzleHttp\Client(['base_uri' => $this->baseUrl]);
    }

    /**
     * Create table with column families
     */
    public function createTable(string $table, array $columnFamilies): bool
    {
        $schema = ['ColumnSchema' => array_map(fn($cf) => ['name' => $cf], $columnFamilies)];

        $response = $this->http->put("/{$table}/schema", [
            'json' => $schema,
            'headers' => ['Content-Type' => 'application/json'],
        ]);

        return $response->getStatusCode() === 201;
    }

    /**
     * Put (write) a row
     */
    public function put(string $table, string $rowKey, array $columns): bool
    {
        $cells = [];
        foreach ($columns as $column => $value) {
            $cells[] = [
                'column' => base64_encode($column),
                '$' => base64_encode($value),
            ];
        }

        $payload = [
            'Row' => [[
                'key' => base64_encode($rowKey),
                'Cell' => $cells,
            ]],
        ];

        $response = $this->http->put("/{$table}/{$rowKey}", [
            'json' => $payload,
            'headers' => ['Content-Type' => 'application/json'],
        ]);

        return $response->getStatusCode() === 200;
    }

    /**
     * Get a single row
     */
    public function get(string $table, string $rowKey): ?array
    {
        try {
            $response = $this->http->get("/{$table}/{$rowKey}", [
                'headers' => ['Accept' => 'application/json'],
            ]);
            $data = json_decode($response->getBody(), true);
            return $this->decodeRow($data);
        } catch (\Exception $e) {
            return null; // row not found
        }
    }

    /**
     * Scan rows by prefix
     */
    public function scan(string $table, string $startRow, string $endRow, int $limit = 100): array
    {
        $scannerUrl = "/{$table}/scanner";
        $scanSpec = [
            'startRow' => base64_encode($startRow),
            'endRow' => base64_encode($endRow),
            'batch' => $limit,
        ];

        $response = $this->http->put($scannerUrl, [
            'json' => $scanSpec,
            'headers' => ['Content-Type' => 'application/json'],
        ]);

        $scannerLocation = $response->getHeader('Location')[0];
        $results = [];

        // Fetch results
        try {
            $scanResponse = $this->http->get($scannerLocation, [
                'headers' => ['Accept' => 'application/json'],
            ]);
            $data = json_decode($scanResponse->getBody(), true);
            foreach ($data['Row'] ?? [] as $row) {
                $results[] = $this->decodeRow(['Row' => [$row]]);
            }
        } finally {
            $this->http->delete($scannerLocation); // close scanner
        }

        return $results;
    }

    private function decodeRow(array $data): array
    {
        $result = [];
        foreach ($data['Row'] ?? [] as $row) {
            $result['key'] = base64_decode($row['key']);
            foreach ($row['Cell'] ?? [] as $cell) {
                $column = base64_decode($cell['column']);
                $result['columns'][$column] = base64_decode($cell['$']);
            }
        }
        return $result;
    }
}

// Usage — IoT sensor data (ঢাকার বায়ু মান)
$hbase = new HBaseClient('hbase-rest.internal', 8080);

// Create table
$hbase->createTable('air_quality', ['readings', 'metadata']);

// Write sensor data (row key = sensor_id + reverse timestamp for recent-first)
$rowKey = 'sensor_dhaka_01_' . (PHP_INT_MAX - time());
$hbase->put('air_quality', $rowKey, [
    'readings:pm25' => '85.5',
    'readings:pm10' => '120.3',
    'readings:aqi' => '165',
    'readings:temperature' => '32.5',
    'metadata:location' => 'Farmgate, Dhaka',
    'metadata:device' => 'AirSense-3000',
]);

// Read latest readings for a sensor (scan by prefix)
$results = $hbase->scan('air_quality', 'sensor_dhaka_01_', 'sensor_dhaka_01_~', 10);
```

---

## 💻 JavaScript Example (HBase REST + Google Bigtable)

```javascript
import { Bigtable } from '@google-cloud/bigtable';

// Google Cloud Bigtable client
const bigtable = new Bigtable({ projectId: 'my-project' });
const instance = bigtable.instance('my-instance');
const table = instance.table('sensor-data');

// Create table with column families
async function setupTable() {
  const [tableExists] = await table.exists();
  if (!tableExists) {
    await table.create({
      families: [
        { name: 'readings', rule: { versions: 5 } },      // Keep 5 versions
        { name: 'metadata', rule: { age: { seconds: 0 } } }, // Keep forever
      ],
    });
  }
}

// Write sensor data
async function writeSensorData(sensorId, data) {
  // Row key design: sensor_id#reverse_timestamp (latest first)
  const reverseTs = (Number.MAX_SAFE_INTEGER - Date.now()).toString().padStart(16, '0');
  const rowKey = `${sensorId}#${reverseTs}`;

  const row = table.row(rowKey);
  await row.save({
    readings: {
      pm25: { value: String(data.pm25), timestamp: Date.now() * 1000 },
      pm10: { value: String(data.pm10), timestamp: Date.now() * 1000 },
      aqi: { value: String(data.aqi), timestamp: Date.now() * 1000 },
    },
    metadata: {
      location: { value: data.location },
      device: { value: data.device },
    },
  });
}

// Read latest N readings for a sensor
async function getLatestReadings(sensorId, limit = 10) {
  const [rows] = await table.getRows({
    start: `${sensorId}#`,
    end: `${sensorId}#~`,
    limit,
    filter: [
      { family: /readings/ },
      { column: { cellLimit: 1 } }, // latest version only
    ],
  });

  return rows.map(row => ({
    key: row.id,
    pm25: row.data.readings?.pm25?.[0]?.value,
    pm10: row.data.readings?.pm10?.[0]?.value,
    aqi: row.data.readings?.aqi?.[0]?.value,
    timestamp: row.data.readings?.pm25?.[0]?.timestamp,
  }));
}

// Batch write (high throughput)
async function batchWrite(sensorId, readings) {
  const mutations = readings.map(data => {
    const reverseTs = (Number.MAX_SAFE_INTEGER - data.timestamp).toString().padStart(16, '0');
    return {
      key: `${sensorId}#${reverseTs}`,
      data: {
        readings: {
          pm25: String(data.pm25),
          aqi: String(data.aqi),
        },
      },
    };
  });

  await table.mutate(mutations); // batch insert
}

// Usage
await setupTable();
await writeSensorData('dhaka_farmgate_01', {
  pm25: 85.5, pm10: 120.3, aqi: 165,
  location: 'Farmgate, Dhaka', device: 'AirSense-3000',
});

const latest = await getLatestReadings('dhaka_farmgate_01', 5);
console.log('Latest readings:', latest);
```

---

## 🎯 Use Cases ও When to Choose

### ✅ HBase/Bigtable ব্যবহার করুন যদি:

| Use Case | কেন |
|----------|-----|
| **Time-series data** | Row key-তে timestamp, sequential writes fast |
| **IoT sensor data** | Billions of rows, sparse columns |
| **Analytics counters** | Atomic increment, high write throughput |
| **Message storage** | Facebook Messenger uses HBase |
| **Web indexing** | Google's original use case |
| **Audit/event logs** | Append-heavy, immutable data |
| **ML feature store** | Low-latency feature retrieval |

### ❌ ব্যবহার করবেন না যদি:

| Condition | Alternative |
|-----------|-------------|
| Complex queries/joins | RDBMS, Elasticsearch |
| Small dataset (<10GB) | PostgreSQL, Redis |
| Transactions (multi-row ACID) | PostgreSQL, CockroachDB |
| Document-style flexible schema | MongoDB |
| Graph relationships | Neo4j |
| Full-text search | Elasticsearch |

---

## 🔑 মনে রাখার বিষয়

1. **Row key design is everything** — scan pattern, hotspot avoidance, data locality সব row key-এ নির্ভর করে
2. **Column families ≤ 3** — বেশি column family performance hurt করে (আলাদা HFile)
3. **Pre-split regions** — Sequential key (timestamp) হলে hotspot হবে; salt/hash prefix ব্যবহার করুন
4. **Bigtable = HBase managed** — API compatible, ops burden নেই
5. **Write-optimized (LSM Tree)** — Sequential writes blindingly fast, random reads need bloom filter + cache
6. **No secondary index** — Row key-ই primary access pattern; অন্য access চাইলে denormalize বা Phoenix/coprocessor
7. **Strong consistency (single row)** — Multi-row consistency চাইলে application-level handle করতে হবে
