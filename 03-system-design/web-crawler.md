# 🕷️ ওয়েব ক্রলার (Web Crawler Design)

## 📌 সংজ্ঞা ও মৌলিক ধারণা

**ওয়েব ক্রলার** (স্পাইডার/বট) হলো একটি প্রোগ্রাম যা স্বয়ংক্রিয়ভাবে ইন্টারনেটে ওয়েবপেজ ভিজিট করে, কন্টেন্ট সংগ্রহ করে এবং লিঙ্ক ফলো করে নতুন পেজ আবিষ্কার করে। Google, Bing, Baidu-এর সার্চ ইঞ্জিনের ভিত্তি হলো ওয়েব ক্রলার।

### ওয়েব ক্রলারের কাজ

```
১. Seed URL থেকে শুরু করো
২. পেজ ডাউনলোড করো
৩. কন্টেন্ট পার্স করো (HTML থেকে তথ্য বের করো)
৪. নতুন লিঙ্ক বের করো
৫. ডুপ্লিকেট চেক করো
৬. নতুন লিঙ্কগুলো Queue-তে রাখো
৭. পুনরায় ২-৬ ধাপ চালাও
```

---

## 🎯 বাস্তব উদাহরণ — সংবাদপত্র সংগ্রাহক

```
কল্পনা করুন, আপনি বাংলাদেশের সব পত্রিকার খবর সংগ্রহ করতে চান:

  শুরু: prothomalo.com, thedailystar.net, kalerkantho.com

  ক্রলার যা করে:
    ১. prothomalo.com → হোমপেজ পড়ো
    ২. সব লিঙ্ক বের করো → /politics, /sports, /tech ...
    ৩. প্রতিটি লিঙ্কে যাও → খবরের শিরোনাম, বিষয়বস্তু সংগ্রহ করো
    ৪. আবার নতুন লিঙ্ক পাওয়া যায় → Queue-তে রাখো
    ৫. একই পেজে দুবার যেও না! (ডিডুপ্লিকেশন)

  সতর্কতা:
    → সার্ভারে অতিরিক্ত চাপ দিও না (Politeness)
    → robots.txt মেনে চলো
```

---

## 📊 সিস্টেম আর্কিটেকচার

```
            ওয়েব ক্রলার আর্কিটেকচার
            ==========================

  ┌───────────┐     ┌──────────────┐     ┌──────────────┐
  │ Seed URLs │────→│ URL Frontier │────→│ URL          │
  │           │     │ (Queue)      │     │ Fetcher      │
  └───────────┘     └──────────────┘     └──────┬───────┘
                           ▲                     │
                           │                     ▼
                    ┌──────┴──────┐      ┌──────────────┐
                    │ URL Filter  │      │ HTML Parser  │
                    │ (Dedup +    │      │ (Content     │
                    │  robots.txt)│      │  Extractor)  │
                    └──────┬──────┘      └──────┬───────┘
                           │                     │
                           │              ┌──────┴───────┐
                           │              │ Link         │
                           └──────────────│ Extractor    │
                                          └──────┬───────┘
                                                 │
                                          ┌──────┴───────┐
                                          │ Storage      │
                                          │ (DB/S3)      │
                                          └──────────────┘
```

---

## 🔄 BFS ক্রলিং (Breadth-First Search)

```
BFS দিয়ে ক্রল করলে প্রতিটি লেভেলের সব পেজ আগে ভিজিট হয়:

  Level 0: prothomalo.com
             │
  Level 1: /politics  /sports  /tech  /business
             │            │
  Level 2: /politics/1  /politics/2  /sports/cricket  ...

  সুবিধা:
    ✅ গুরুত্বপূর্ণ পেজ আগে ক্রল হয় (হোমপেজের কাছের)
    ✅ ক্রল ডেপথ নিয়ন্ত্রণ করা সহজ

  DFS vs BFS:
    DFS → একটি পথ গভীরে যায় → Spider Trap-এ আটকে যেতে পারে
    BFS → সব পথ সমানভাবে → আরও সম্পূর্ণ ক্রল ✅
```

---

## 🤝 Politeness (ভদ্রতা নীতি)

```
একটি সার্ভারকে অতিরিক্ত রিকোয়েস্ট দিলে:
  → সার্ভার ক্র্যাশ হতে পারে
  → আপনার IP ব্লক হতে পারে
  → এটি অনৈতিক

ভদ্রতা নিয়ম:
  ১. robots.txt মেনে চলো
  ২. একই ডোমেইনে পরপর রিকোয়েস্টে বিরতি দাও (১-৫ সেকেন্ড)
  ৩. Crawl-Delay হেডার মেনে চলো
  ৪. ব্যস্ত সময়ে ক্রল কমাও (রাতে বাড়াও)
  ৫. User-Agent সঠিকভাবে সেট করো

robots.txt উদাহরণ:
  User-agent: *
  Disallow: /admin/
  Disallow: /private/
  Crawl-delay: 2          ← ২ সেকেন্ড বিরতি
```

---

## 🔗 URL Frontier (ইউআরএল ফ্রন্টিয়ার)

```
URL Frontier হলো ক্রলারের "করণীয় তালিকা":

  ┌─────────────────────────────────────────┐
  │           URL Frontier                   │
  │                                          │
  │  Priority Queue:                         │
  │    ├── উচ্চ: হোমপেজ, নিউজ (প্রতি ঘণ্টায়)  │
  │    ├── মাঝারি: ক্যাটাগরি পেজ (প্রতি দিন)   │
  │    └── নিম্ন: আর্কাইভ (মাসে একবার)        │
  │                                          │
  │  Per-Host Queue:                         │
  │    ├── prothomalo.com → [url1, url2...]  │
  │    ├── thedailystar.net → [url3, url4...]│
  │    └── kalerkantho.com → [url5, url6...] │
  │                                          │
  │  → প্রতি হোস্টে আলাদা Queue = Politeness │
  └─────────────────────────────────────────┘
```

---

## 🔁 ডিডুপ্লিকেশন (Deduplication)

```
একই পেজ বারবার ক্রল করা অপচয়। দুই স্তরে ডিডুপ করো:

১. URL ডিডুপ — একই URL দুবার ক্রল করো না
   → Hash Set বা Bloom Filter ব্যবহার করো
   → Bloom Filter: মেমরি সাশ্রয়ী, সামান্য false positive আছে

২. Content ডিডুপ — ভিন্ন URL একই কন্টেন্ট
   → Fingerprint (MD5/SHA) তুলনা করো
   → SimHash/MinHash — কাছাকাছি কন্টেন্ট ধরতে পারে

উদাহরণ:
  /news/story?id=123
  /news/story?id=123&ref=home   ← একই কন্টেন্ট!
  /bn/news/story?id=123         ← একই কন্টেন্ট, ভিন্ন URL!
```

---

## 💻 PHP কোড উদাহরণ

```php
<?php

class WebCrawler
{
    private array $visited = [];
    private SplQueue $frontier;
    private array $domainLastAccess = [];
    private int $politenessDelay;

    public function __construct(int $politenessDelay = 2)
    {
        $this->frontier = new SplQueue();
        $this->politenessDelay = $politenessDelay;
    }

    public function addSeedUrls(array $urls): void
    {
        foreach ($urls as $url) {
            $this->frontier->enqueue($url);
        }
    }

    public function crawl(int $maxPages = 100): array
    {
        $results = [];

        while (!$this->frontier->isEmpty() && count($results) < $maxPages) {
            $url = $this->frontier->dequeue();

            if ($this->isVisited($url)) continue;
            if (!$this->isAllowedByRobots($url)) continue;

            // ভদ্রতা বিরতি
            $this->respectPoliteness($url);

            $html = $this->fetchPage($url);
            if ($html === null) continue;

            $this->visited[$this->normalizeUrl($url)] = true;

            // কন্টেন্ট পার্স করো
            $content = $this->extractContent($html);
            $results[] = ['url' => $url, 'content' => $content];

            // নতুন লিঙ্ক বের করো (BFS)
            $links = $this->extractLinks($html, $url);
            foreach ($links as $link) {
                if (!$this->isVisited($link)) {
                    $this->frontier->enqueue($link);
                }
            }
        }

        return $results;
    }

    private function fetchPage(string $url): ?string
    {
        $ch = curl_init($url);
        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_FOLLOWREDIRECTS => true,
            CURLOPT_TIMEOUT => 10,
            CURLOPT_USERAGENT => 'BDCrawler/1.0 (+https://example.com/bot)',
        ]);
        $html = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        return ($httpCode === 200) ? $html : null;
    }

    private function extractLinks(string $html, string $baseUrl): array
    {
        $dom = new DOMDocument();
        @$dom->loadHTML($html);
        $links = [];

        foreach ($dom->getElementsByTagName('a') as $anchor) {
            $href = $anchor->getAttribute('href');
            $absoluteUrl = $this->toAbsoluteUrl($href, $baseUrl);
            if ($absoluteUrl) $links[] = $absoluteUrl;
        }

        return $links;
    }

    private function extractContent(string $html): string
    {
        $dom = new DOMDocument();
        @$dom->loadHTML($html);
        return $dom->getElementsByTagName('title')[0]?->textContent ?? '';
    }

    private function respectPoliteness(string $url): void
    {
        $domain = parse_url($url, PHP_URL_HOST);
        if (isset($this->domainLastAccess[$domain])) {
            $elapsed = time() - $this->domainLastAccess[$domain];
            if ($elapsed < $this->politenessDelay) {
                sleep($this->politenessDelay - $elapsed);
            }
        }
        $this->domainLastAccess[$domain] = time();
    }

    private function isVisited(string $url): bool
    {
        return isset($this->visited[$this->normalizeUrl($url)]);
    }

    private function normalizeUrl(string $url): string
    {
        $parsed = parse_url($url);
        return ($parsed['scheme'] ?? 'https') . '://' . ($parsed['host'] ?? '') . ($parsed['path'] ?? '/');
    }

    private function isAllowedByRobots(string $url): bool
    {
        // robots.txt পার্স করে চেক করো (সিম্পলিফাইড)
        return true;
    }

    private function toAbsoluteUrl(string $href, string $base): ?string
    {
        if (str_starts_with($href, 'http')) return $href;
        if (str_starts_with($href, '/')) {
            $parsed = parse_url($base);
            return ($parsed['scheme'] ?? 'https') . '://' . ($parsed['host'] ?? '') . $href;
        }
        return null;
    }
}

// ব্যবহার
$crawler = new WebCrawler(politenessDelay: 2);
$crawler->addSeedUrls(['https://example.com']);
$results = $crawler->crawl(maxPages: 50);
```

---

## 💻 JavaScript কোড উদাহরণ

```javascript
const { JSDOM } = require('jsdom');
const crypto = require('crypto');

class WebCrawler {
  constructor(options = {}) {
    this.maxPages = options.maxPages || 100;
    this.politenessDelay = options.politenessDelay || 2000;
    this.frontier = [];           // URL Queue (BFS)
    this.visited = new Set();     // ডিডুপ — URL ট্র্যাকিং
    this.contentHashes = new Set(); // ডিডুপ — কন্টেন্ট ট্র্যাকিং
    this.domainTimers = new Map();
    this.results = [];
  }

  addSeeds(urls) {
    urls.forEach((url) => this.frontier.push(url));
  }

  async crawl() {
    while (this.frontier.length > 0 && this.results.length < this.maxPages) {
      const url = this.frontier.shift(); // BFS: Queue থেকে প্রথমটি নাও
      const normalized = this._normalizeUrl(url);

      if (this.visited.has(normalized)) continue;
      this.visited.add(normalized);

      // ভদ্রতা বিরতি
      await this._respectPoliteness(url);

      try {
        const html = await this._fetchPage(url);
        if (!html) continue;

        // কন্টেন্ট ডিডুপ
        const hash = crypto.createHash('md5').update(html).digest('hex');
        if (this.contentHashes.has(hash)) continue;
        this.contentHashes.add(hash);

        const { title, links } = this._parsePage(html, url);
        this.results.push({ url, title });

        // নতুন লিঙ্ক Queue-তে যোগ করো
        links.forEach((link) => {
          if (!this.visited.has(this._normalizeUrl(link))) {
            this.frontier.push(link);
          }
        });
      } catch (err) {
        console.error(`ক্রল ব্যর্থ: ${url}`, err.message);
      }
    }

    return this.results;
  }

  async _fetchPage(url) {
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), 10000);

    try {
      const res = await fetch(url, {
        signal: controller.signal,
        headers: { 'User-Agent': 'BDCrawler/1.0' },
      });
      clearTimeout(timeout);
      if (!res.ok) return null;
      return await res.text();
    } catch {
      clearTimeout(timeout);
      return null;
    }
  }

  _parsePage(html, baseUrl) {
    const dom = new JSDOM(html);
    const doc = dom.window.document;
    const title = doc.title || '';

    const links = [];
    doc.querySelectorAll('a[href]').forEach((a) => {
      try {
        const absolute = new URL(a.getAttribute('href'), baseUrl).href;
        links.push(absolute);
      } catch {}
    });

    return { title, links };
  }

  async _respectPoliteness(url) {
    const domain = new URL(url).hostname;
    const lastAccess = this.domainTimers.get(domain) || 0;
    const elapsed = Date.now() - lastAccess;

    if (elapsed < this.politenessDelay) {
      await new Promise((r) => setTimeout(r, this.politenessDelay - elapsed));
    }
    this.domainTimers.set(domain, Date.now());
  }

  _normalizeUrl(url) {
    try {
      const u = new URL(url);
      return `${u.protocol}//${u.hostname}${u.pathname}`;
    } catch {
      return url;
    }
  }
}

// ব্যবহার
(async () => {
  const crawler = new WebCrawler({ maxPages: 50, politenessDelay: 2000 });
  crawler.addSeeds(['https://example.com']);
  const results = await crawler.crawl();
  console.log(`${results.length}টি পেজ ক্রল হয়েছে`);
})();
```

---

## ✅ কখন ব্যবহার করবেন

| পরিস্থিতি | কেন |
|---|---|
| সার্চ ইঞ্জিন ইনডেক্সিং | ওয়েব পেজ আবিষ্কার ও ইনডেক্স |
| প্রাইস কম্পারিজন সাইট | বিভিন্ন সাইট থেকে দাম সংগ্রহ |
| ওয়েব আর্কাইভিং | ঐতিহাসিক কন্টেন্ট সংরক্ষণ |
| SEO অডিটিং | ব্রোকেন লিঙ্ক, মেটাডেটা চেক |
| ডেটা মাইনিং / ML ট্রেনিং | বিপুল ডেটা সংগ্রহ |

## ❌ কখন ব্যবহার করবেন না

| পরিস্থিতি | কেন |
|---|---|
| API আছে এমন সাইট | API ব্যবহার করো, ক্রল নয় |
| robots.txt নিষিদ্ধ | আইনি ও নৈতিক সমস্যা |
| রিয়েল-টাইম ডেটা দরকার | ক্রলিং ব্যাচ প্রসেস, রিয়েল-টাইম নয় |
| সিঙ্গেল পেজ অ্যাপ (SPA) | জাভাস্ক্রিপ্ট রেন্ডারিং জটিল |

---

## 🔑 মনে রাখার পয়েন্ট

1. **BFS** — গুরুত্বপূর্ণ পেজ আগে ক্রল হয়, Spider Trap এড়ায়
2. **Politeness** — robots.txt মানো, ডিলে দাও, User-Agent সেট করো
3. **URL Frontier** — Priority + Per-Host Queue = দক্ষ ও ভদ্র ক্রলিং
4. **Deduplication** — URL Hash + Content Hash দুই স্তরে ডিডুপ করো
5. ইন্টারভিউতে — স্কেলেবিলিটি, ফল্ট টলারেন্স, DNS ক্যাশিং আলোচনা করুন
