# 🔌 API কনজাম্পশন প্যাটার্নস (API Consumption Patterns)

> **লেভেল:** Advanced | **ভাষা:** PHP (Guzzle) + JavaScript (fetch, Axios, React Query, SWR)  
> **প্রেক্ষাপট:** বাংলাদেশ — Daraz, Pathao, bKash, Nagad, Prothom Alo, Grameenphone

---

## 📌 সংজ্ঞা ও পরিচিতি

**API Consumption** হলো ক্লায়েন্ট সাইড থেকে সার্ভারের API-তে রিকোয়েস্ট পাঠানো, রেসপন্স গ্রহণ করা, ডেটা ক্যাশ করা, এরর হ্যান্ডেল করা এবং ব্যবহারকারীকে সর্বোত্তম অভিজ্ঞতা দেওয়ার সামগ্রিক প্রক্রিয়া। শুধু `fetch()` কল করাই যথেষ্ট নয় — প্রোডাকশন-গ্রেড অ্যাপ্লিকেশনে caching, retry, cancellation, pagination, এবং real-time data sync-এর মতো জটিল প্যাটার্ন দরকার হয়।

```
┌─────────────────────────────────────────────────────────────────┐
│                    API Consumption Layer                         │
│                                                                 │
│  ┌───────────┐  ┌──────────┐  ┌─────────┐  ┌───────────────┐  │
│  │  Fetching  │  │ Caching  │  │  Retry  │  │  Real-time    │  │
│  │  Library   │  │ Strategy │  │ & Retry │  │  Subscription │  │
│  └─────┬─────┘  └────┬─────┘  └────┬────┘  └───────┬───────┘  │
│        │              │             │               │           │
│        ▼              ▼             ▼               ▼           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              API Client / Repository Layer               │   │
│  └──────────────────────┬──────────────────────────────────┘   │
│                         │                                       │
└─────────────────────────┼───────────────────────────────────────┘
                          │  HTTP / WebSocket / SSE
                          ▼
              ┌───────────────────────┐
              │     Backend API       │
              │  (REST / GraphQL /    │
              │   gRPC / WebSocket)   │
              └───────────────────────┘
```

---

## 🏠 বাস্তব জীবনের উদাহরণ

কল্পনা করুন **Daraz** অ্যাপ ওপেন করেছেন:
1. **হোম পেজ লোড** → প্রোডাক্ট লিস্ট API কল (caching + prefetch)
2. **সার্চ করলেন** → debounced API কল + request cancellation (আগের সার্চ বাতিল)
3. **স্ক্রল করছেন** → infinite scroll pagination (cursor-based)
4. **অর্ডার দিলেন** → optimistic update (UI তাৎক্ষণিক আপডেট, পরে সার্ভার কনফার্ম)
5. **অর্ডার ট্র্যাকিং** → real-time polling বা WebSocket
6. **নেটওয়ার্ক চলে গেল** → retry with exponential backoff + offline cache

প্রতিটি ধাপে ভিন্ন ভিন্ন consumption pattern কাজ করছে!

---

# 📖 ১. বেসিক Fetch প্যাটার্নস (Basic Fetch Patterns)

## 🔗 Fetch API (Browser Native)

```javascript
// ✅ Good: সঠিক error handling সহ fetch
async function getProducts() {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), 10000); // ১০ সেকেন্ড timeout

  try {
    const response = await fetch('https://api.daraz.com.bd/products', {
      method: 'GET',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`,
        'Accept-Language': 'bn-BD'
      },
      signal: controller.signal
    });

    clearTimeout(timeoutId);

    if (!response.ok) {
      throw new Error(`HTTP Error: ${response.status} - ${response.statusText}`);
    }

    const data = await response.json();
    return data;
  } catch (error) {
    if (error.name === 'AbortError') {
      console.error('রিকোয়েস্ট সময়সীমা অতিক্রম করেছে');
    } else {
      console.error('API কল ব্যর্থ:', error.message);
    }
    throw error;
  }
}
```

```
Fetch API রিকোয়েস্ট ফ্লো:
┌──────────┐   fetch()    ┌──────────┐   HTTP    ┌──────────┐
│  Browser │────────────▶│  Fetch   │─────────▶│  Server  │
│   Code   │             │   API    │          │          │
│          │◀────────────│          │◀─────────│          │
│          │  Promise    │ (native) │ Response │          │
└──────────┘  resolve    └──────────┘          └──────────┘
                │
                ▼
        ⚠️ fetch() শুধু network error-এ reject করে!
        ⚠️ 404/500 কে "error" মনে করে না!
        ⚠️ response.ok ম্যানুয়ালি চেক করতে হবে!
```

## 🔗 Axios (থার্ড-পার্টি লাইব্রেরি)

```javascript
// ✅ Good: Axios instance with interceptors
import axios from 'axios';

const apiClient = axios.create({
  baseURL: 'https://api.pathao.com/v1',
  timeout: 15000,
  headers: {
    'Content-Type': 'application/json',
    'X-Client-Version': '2.5.0'
  }
});

// Request Interceptor — প্রতিটি রিকোয়েস্টে token যুক্ত করা
apiClient.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('pathao_token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Response Interceptor — 401 হলে token refresh
apiClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;

    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;
      const newToken = await refreshToken();
      originalRequest.headers.Authorization = `Bearer ${newToken}`;
      return apiClient(originalRequest);
    }

    return Promise.reject(error);
  }
);

// ব্যবহার
const rides = await apiClient.get('/rides/history');
```

## 🔗 PHP Guzzle (Server-to-Server)

```php
<?php
// ✅ Good: Guzzle দিয়ে bKash Payment API কল

use GuzzleHttp\Client;
use GuzzleHttp\Exception\RequestException;
use GuzzleHttp\HandlerStack;
use GuzzleHttp\Middleware;
use GuzzleHttp\Psr7\Request;
use GuzzleHttp\Psr7\Response;

$stack = HandlerStack::create();

// Retry middleware — ৩ বার retry করবে
$stack->push(Middleware::retry(
    function (int $retries, Request $request, ?Response $response, ?\Throwable $exception) {
        if ($retries >= 3) return false;
        if ($exception instanceof \GuzzleHttp\Exception\ConnectException) return true;
        if ($response && $response->getStatusCode() >= 500) return true;
        return false;
    },
    function (int $retries) {
        return 1000 * pow(2, $retries); // Exponential backoff: 1s, 2s, 4s
    }
));

$client = new Client([
    'base_uri' => 'https://tokenized.sandbox.bka.sh/v1.2.0-beta/',
    'timeout'  => 30,
    'handler'  => $stack,
    'headers'  => [
        'Content-Type' => 'application/json',
        'Accept'       => 'application/json',
    ]
]);

try {
    $response = $client->post('tokenized/checkout/payment/create', [
        'headers' => [
            'Authorization' => $idToken,
            'X-APP-Key'     => config('bkash.app_key'),
        ],
        'json' => [
            'mode'           => '0011',
            'payerReference' => $userId,
            'amount'         => '500.00',
            'currency'       => 'BDT',
            'intent'         => 'sale',
        ]
    ]);

    $result = json_decode($response->getBody(), true);
} catch (RequestException $e) {
    Log::error('bKash API ব্যর্থ', [
        'status'  => $e->getResponse()?->getStatusCode(),
        'body'    => $e->getResponse()?->getBody()->getContents(),
        'message' => $e->getMessage(),
    ]);
    throw new PaymentException('পেমেন্ট প্রসেস করা যায়নি');
}
```

## 🔗 AbortController — রিকোয়েস্ট বাতিল করা

```javascript
// Daraz সার্চ — টাইপ করার সময় আগের রিকোয়েস্ট বাতিল
let currentController = null;

async function searchProducts(query) {
  // আগের রিকোয়েস্ট বাতিল করো
  if (currentController) {
    currentController.abort();
  }

  currentController = new AbortController();

  try {
    const res = await fetch(
      `https://api.daraz.com.bd/search?q=${encodeURIComponent(query)}`,
      { signal: currentController.signal }
    );
    return await res.json();
  } catch (err) {
    if (err.name !== 'AbortError') throw err;
    // AbortError মানে আমরাই বাতিল করেছি — কিছু করার দরকার নেই
  }
}
```

```
AbortController ফ্লো:
             টাইপ: "ph"         টাইপ: "pho"        টাইপ: "phone"
                │                    │                    │
                ▼                    ▼                    ▼
          ┌──────────┐         ┌──────────┐         ┌──────────┐
          │ fetch #1 │──abort──│ fetch #2 │──abort──│ fetch #3 │──▶ ফলাফল
          │ (বাতিল)  │  ──────▶│ (বাতিল)  │  ──────▶│ (সফল)   │
          └──────────┘         └──────────┘         └──────────┘
```

---

# 📖 ২. ডেটা ফেচিং লাইব্রেরি (Data Fetching Libraries)

## 📊 TanStack Query (React Query)

**React Query** হলো সার্ভার-স্টেট ম্যানেজমেন্টের জন্য সবচেয়ে জনপ্রিয় লাইব্রেরি। এটি caching, background refetching, stale data management, mutation, এবং cache invalidation স্বয়ংক্রিয়ভাবে পরিচালনা করে।

```
React Query ইন্টার্নাল আর্কিটেকচার:
┌──────────────────────────────────────────────────┐
│                  React Component                  │
│           useQuery('products', fetchFn)           │
└───────────────────────┬──────────────────────────┘
                        │
                        ▼
┌──────────────────────────────────────────────────┐
│                  Query Client                     │
│  ┌────────────┐  ┌───────────┐  ┌────────────┐  │
│  │   Query    │  │  Mutation  │  │  Infinite  │  │
│  │   Cache    │  │   Cache   │  │   Query    │  │
│  └─────┬──────┘  └─────┬─────┘  └──────┬─────┘  │
│        │               │               │         │
│        ▼               ▼               ▼         │
│  ┌─────────────────────────────────────────┐     │
│  │     Background Refetch Scheduler        │     │
│  │  (staleTime, refetchInterval, focus)    │     │
│  └────────────────────┬────────────────────┘     │
└───────────────────────┼──────────────────────────┘
                        │
                        ▼  HTTP Request
                ┌───────────────┐
                │  Backend API  │
                └───────────────┘
```

```javascript
// ✅ Good: React Query দিয়ে Prothom Alo নিউজ ফিড
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

// ক্যাশিং সহ ডেটা ফেচ
function useNewsFeed(category) {
  return useQuery({
    queryKey: ['news', category],
    queryFn: () => fetch(`/api/news?cat=${category}`).then(r => r.json()),
    staleTime: 5 * 60 * 1000,       // ৫ মিনিট পর্যন্ত fresh
    gcTime: 30 * 60 * 1000,         // ৩০ মিনিট ক্যাশে রাখো
    refetchOnWindowFocus: true,      // ট্যাবে ফিরলে refetch
    retry: 2,                        // ব্যর্থ হলে ২ বার retry
  });
}

// Mutation — নিউজে কমেন্ট করা
function useAddComment() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (comment) =>
      fetch('/api/comments', {
        method: 'POST',
        body: JSON.stringify(comment),
        headers: { 'Content-Type': 'application/json' }
      }),
    // সফল হলে নিউজ ক্যাশ invalidate করো
    onSuccess: (_, variables) => {
      queryClient.invalidateQueries({ queryKey: ['news', variables.newsId] });
    }
  });
}

// কম্পোনেন্টে ব্যবহার
function NewsFeed({ category }) {
  const { data, isLoading, isError, error } = useNewsFeed(category);
  const addComment = useAddComment();

  if (isLoading) return <p>লোড হচ্ছে...</p>;
  if (isError) return <p>ত্রুটি: {error.message}</p>;

  return (
    <div>
      {data.articles.map(article => (
        <NewsCard key={article.id} article={article} />
      ))}
    </div>
  );
}
```

## 📊 SWR (Stale-While-Revalidate)

Vercel-এর তৈরি SWR লাইব্রেরি `stale-while-revalidate` HTTP ক্যাশ কৌশল অনুসরণ করে — আগে ক্যাশ থেকে stale ডেটা দেখায়, তারপর ব্যাকগ্রাউন্ডে fresh ডেটা আনে।

```javascript
// ✅ Good: SWR দিয়ে Grameenphone ব্যালেন্স চেক
import useSWR from 'swr';

const fetcher = (url) => fetch(url).then(r => r.json());

function useBalance() {
  return useSWR('/api/gp/balance', fetcher, {
    refreshInterval: 30000,        // প্রতি ৩০ সেকেন্ডে রিফ্রেশ
    revalidateOnFocus: true,       // ট্যাবে ফিরলে revalidate
    dedupingInterval: 5000,        // ৫ সেকেন্ডের মধ্যে duplicate রিকোয়েস্ট বাতিল
    errorRetryCount: 3,
  });
}

function BalanceWidget() {
  const { data, error, isLoading, mutate } = useBalance();

  return (
    <div>
      <h3>আপনার ব্যালেন্স</h3>
      {isLoading && <Spinner />}
      {data && <p>৳ {data.balance}</p>}
      <button onClick={() => mutate()}>রিফ্রেশ করুন</button>
    </div>
  );
}
```

## 📊 RTK Query (Redux Toolkit Query)

```javascript
// ✅ Good: RTK Query দিয়ে Nagad API
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

export const nagadApi = createApi({
  reducerPath: 'nagadApi',
  baseQuery: fetchBaseQuery({
    baseUrl: 'https://api.nagad.com.bd/v1',
    prepareHeaders: (headers, { getState }) => {
      const token = getState().auth.token;
      if (token) headers.set('Authorization', `Bearer ${token}`);
      return headers;
    },
  }),
  tagTypes: ['Transaction', 'Balance'],
  endpoints: (builder) => ({
    getTransactions: builder.query({
      query: ({ page, limit }) => `/transactions?page=${page}&limit=${limit}`,
      providesTags: ['Transaction'],
    }),
    sendMoney: builder.mutation({
      query: (body) => ({
        url: '/send-money',
        method: 'POST',
        body,
      }),
      invalidatesTags: ['Transaction', 'Balance'],
    }),
  }),
});

export const { useGetTransactionsQuery, useSendMoneyMutation } = nagadApi;
```

## 📊 TanStack Query vs SWR vs RTK Query তুলনা

```
┌────────────────────┬───────────────────┬─────────────┬──────────────┐
│    বৈশিষ্ট্য       │  TanStack Query   │     SWR     │  RTK Query   │
├────────────────────┼───────────────────┼─────────────┼──────────────┤
│ ক্যাশিং            │ ✅ শক্তিশালী      │ ✅ ভালো     │ ✅ ভালো      │
│ DevTools           │ ✅ বিল্ট-ইন       │ ❌ নেই      │ ✅ Redux DT  │
│ Infinite Query     │ ✅ বিল্ট-ইন       │ ✅ আছে      │ ⚠️ ম্যানুয়াল│
│ Optimistic Update  │ ✅ সহজ           │ ✅ mutate()  │ ✅ আছে      │
│ SSR Support        │ ✅ চমৎকার        │ ✅ ভালো     │ ⚠️ জটিল     │
│ Bundle Size        │ ~13KB            │ ~4KB        │ ~12KB        │
│ Redux প্রয়োজন     │ ❌ না            │ ❌ না       │ ✅ হ্যাঁ     │
│ Learning Curve     │ মাঝারি           │ সহজ        │ কঠিন        │
└────────────────────┴───────────────────┴─────────────┴──────────────┘
```

---

# 📖 ৩. ক্যাশিং স্ট্র্যাটেজি (Caching Strategies)

## 📊 ক্যাশিং লেয়ার ওভারভিউ

```
ক্যাশিং লেয়ার (ক্লায়েন্ট থেকে সার্ভার):

  ┌──────────────────────────────────────────────────────┐
  │ Layer 1: In-Memory Cache (React Query / SWR)         │
  │   → সবচেয়ে দ্রুত, পেজ রিলোডে হারিয়ে যায়            │
  ├──────────────────────────────────────────────────────┤
  │ Layer 2: Browser HTTP Cache (Cache-Control, ETag)    │
  │   → ব্রাউজার নিজে ম্যানেজ করে                        │
  ├──────────────────────────────────────────────────────┤
  │ Layer 3: Service Worker Cache (Offline-first)        │
  │   → অফলাইনেও কাজ করে, আমরা নিয়ন্ত্রণ করি           │
  ├──────────────────────────────────────────────────────┤
  │ Layer 4: CDN Cache (Cloudflare, AWS CloudFront)      │
  │   → সার্ভারের কাছে, ভৌগোলিকভাবে বিতরিত              │
  ├──────────────────────────────────────────────────────┤
  │ Layer 5: Server Cache (Redis / Memcached)            │
  │   → ডেটাবেস লোড কমায়                                 │
  └──────────────────────────────────────────────────────┘
```

## HTTP Cache Headers

```javascript
// সার্ভার থেকে Cache-Control হেডার পাঠানো (Express.js)
app.get('/api/products', (req, res) => {
  res.set({
    'Cache-Control': 'public, max-age=300, stale-while-revalidate=60',
    'ETag': '"v1-abc123"',
    'Last-Modified': 'Wed, 15 Jan 2025 10:00:00 GMT'
  });
  res.json(products);
});
```

```php
<?php
// PHP Laravel — ETag দিয়ে conditional response
public function index(Request $request)
{
    $products = Product::active()->get();
    $etag = md5($products->toJson());

    if ($request->header('If-None-Match') === '"' . $etag . '"') {
        return response('', 304); // Not Modified — ব্যান্ডউইথ বাঁচলো!
    }

    return response()->json($products)
        ->header('ETag', '"' . $etag . '"')
        ->header('Cache-Control', 'public, max-age=600');
}
```

## Service Worker Caching (Offline-First)

```javascript
// ✅ Good: Prothom Alo PWA — অফলাইনেও নিউজ পড়া যায়
self.addEventListener('fetch', (event) => {
  if (event.request.url.includes('/api/news')) {
    event.respondWith(
      caches.open('news-cache-v1').then(async (cache) => {
        try {
          // প্রথমে নেটওয়ার্ক থেকে আনো
          const networkResponse = await fetch(event.request);
          cache.put(event.request, networkResponse.clone());
          return networkResponse;
        } catch (error) {
          // নেটওয়ার্ক না থাকলে ক্যাশ থেকে দেখাও
          const cachedResponse = await cache.match(event.request);
          if (cachedResponse) return cachedResponse;
          return new Response(JSON.stringify({ error: 'অফলাইন — ক্যাশে ডেটা নেই' }), {
            headers: { 'Content-Type': 'application/json' }
          });
        }
      })
    );
  }
});
```

```
Stale-While-Revalidate প্যাটার্ন:

    ব্যবহারকারী              ক্যাশ                সার্ভার
        │                      │                     │
        │──── রিকোয়েস্ট ──────▶│                     │
        │◀── stale ডেটা ────── │ (তাৎক্ষণিক!)       │
        │    (পুরাতন কিন্তু    │                     │
        │     দ্রুত)           │──── revalidate ────▶│
        │                      │◀── fresh ডেটা ──────│
        │◀── আপডেট ডেটা ──────│                     │
        │    (ব্যাকগ্রাউন্ডে  │                     │
        │     UI আপডেট)        │                     │
```

---

# 📖 ৪. রিট্রাই ও রেজিলিয়েন্স প্যাটার্নস (Retry & Resilience)

## 📊 Exponential Backoff

নেটওয়ার্ক ব্যর্থ হলে তাৎক্ষণিক retry করলে সার্ভারে অতিরিক্ত চাপ পড়ে। Exponential backoff প্রতিটি retry-তে অপেক্ষার সময় দ্বিগুণ করে।

```javascript
// ✅ Good: Exponential Backoff with Jitter
async function fetchWithRetry(url, options = {}, maxRetries = 3) {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      const response = await fetch(url, options);
      if (response.ok) return response;

      // 4xx error — retry করে লাভ নেই (4xx মানে ক্লায়েন্ট ভুল)
      if (response.status >= 400 && response.status < 500) {
        throw new Error(`Client Error: ${response.status}`);
      }

      throw new Error(`Server Error: ${response.status}`);
    } catch (error) {
      if (attempt === maxRetries) throw error;

      // Exponential backoff + jitter
      const baseDelay = Math.pow(2, attempt) * 1000; // 1s, 2s, 4s
      const jitter = Math.random() * 1000;           // 0-1s র‍্যান্ডম
      const delay = baseDelay + jitter;

      console.log(`Retry ${attempt + 1}/${maxRetries} — ${delay}ms পর আবার চেষ্টা`);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}
```

```
Exponential Backoff ভিজ্যুয়ালাইজেশন:

  চেষ্টা ১     চেষ্টা ২        চেষ্টা ৩              চেষ্টা ৪
    ❌            ❌               ❌                   ✅
    │            │               │                    │
    ├──1s+jitter─┤               │                    │
    │            ├───2s+jitter───┤                    │
    │            │               ├────4s+jitter───────┤
    │            │               │                    │
  ──────────────────────────────────────────────────────▶ সময়

  ❌ Bad (কোনো বিরতি ছাড়া retry):
    ❌ ❌ ❌ ❌ ❌ ❌ ❌ → সার্ভার ধসে পড়তে পারে!
```

## 📊 Client-Side Circuit Breaker

Pathao-এর রাইড সার্ভিস যদি বারবার ব্যর্থ হয়, তাহলে ক্রমাগত কল করা বোকামি — বরং কিছুক্ষণ থামুন:

```javascript
// ✅ Good: সিম্পল Circuit Breaker
class CircuitBreaker {
  constructor({ failureThreshold = 5, resetTimeout = 30000 }) {
    this.failureCount = 0;
    this.failureThreshold = failureThreshold;
    this.resetTimeout = resetTimeout;
    this.state = 'CLOSED';      // CLOSED = স্বাভাবিক, OPEN = বন্ধ
    this.nextAttempt = 0;
  }

  async call(fn) {
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        throw new Error('Circuit OPEN — সার্ভিস সাময়িকভাবে অনুপলব্ধ');
      }
      this.state = 'HALF_OPEN'; // একটি test রিকোয়েস্ট পাঠাও
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  onSuccess() {
    this.failureCount = 0;
    this.state = 'CLOSED';
  }

  onFailure() {
    this.failureCount++;
    if (this.failureCount >= this.failureThreshold) {
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.resetTimeout;
    }
  }
}

// ব্যবহার — Pathao ride API
const rideBreaker = new CircuitBreaker({ failureThreshold: 3, resetTimeout: 60000 });

async function requestRide(data) {
  return rideBreaker.call(() =>
    fetch('/api/pathao/rides', { method: 'POST', body: JSON.stringify(data) })
  );
}
```

```
Circuit Breaker স্টেট ডায়াগ্রাম:

  ┌──────────┐    failure >= threshold    ┌──────────┐
  │  CLOSED  │──────────────────────────▶│   OPEN   │
  │ (স্বাভাবিক)│                           │  (বন্ধ)   │
  └─────▲────┘                            └─────┬────┘
        │                                       │
        │   সফল                        timeout শেষ
        │                                       │
        │            ┌──────────┐               │
        └────────────│HALF_OPEN │◀──────────────┘
                     │(পরীক্ষা)  │
                     └─────┬────┘
                           │
                        ব্যর্থ → আবার OPEN
```

## 📊 Optimistic vs Pessimistic Update

```
Optimistic Update (আশাবাদী আপডেট):
  ১. UI তাৎক্ষণিক আপডেট  →  ২. সার্ভারে পাঠাও  →  ৩. ব্যর্থ হলে rollback

  bKash Send Money উদাহরণ:
  "টাকা পাঠানো হচ্ছে..." → UI-তে ব্যালেন্স কমাও → সার্ভার কনফার্ম → ✅
                                                    → সার্ভার ব্যর্থ → ↩️ rollback

Pessimistic Update (সতর্ক আপডেট):
  ১. সার্ভারে পাঠাও  →  ২. অপেক্ষা  →  ৩. সফল হলে UI আপডেট

  Nagad Cash Out উদাহরণ:
  "প্রসেসিং..." (লোডিং দেখাও) → সার্ভার কনফার্ম → UI আপডেট → ✅
```

```javascript
// ✅ Good: React Query দিয়ে Optimistic Update — bKash Send Money
const sendMoney = useMutation({
  mutationFn: (data) => apiClient.post('/bkash/send', data),

  // সার্ভারে পাঠানোর আগেই UI আপডেট
  onMutate: async (newTransfer) => {
    await queryClient.cancelQueries({ queryKey: ['balance'] });
    const previousBalance = queryClient.getQueryData(['balance']);

    queryClient.setQueryData(['balance'], (old) => ({
      ...old,
      amount: old.amount - newTransfer.amount
    }));

    return { previousBalance }; // rollback-এর জন্য
  },

  // ব্যর্থ হলে আগের ব্যালেন্স ফেরত আনো
  onError: (err, variables, context) => {
    queryClient.setQueryData(['balance'], context.previousBalance);
    toast.error('টাকা পাঠানো ব্যর্থ হয়েছে — আবার চেষ্টা করুন');
  },

  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: ['balance'] });
  }
});
```

---

# 📖 ৫. পেজিনেশন প্যাটার্নস (Pagination Patterns)

## 📊 Offset-Based vs Cursor-Based

```
Offset-Based (সহজ কিন্তু সমস্যাযুক্ত):
  GET /api/products?page=3&limit=20
  → OFFSET 40 LIMIT 20

  সমস্যা — পেজ ৩ পড়ার সময় নতুন আইটেম যোগ হলে:
  ┌───┬───┬───┬───┬───┬───┬───┐
  │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │  ← আগে
  └───┴───┴───┴───┴───┴───┴───┘
  page1 ─┘     page2 ─┘

  ┌───┬───┬───┬───┬───┬───┬───┬───┐
  │ ★ │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │  ← নতুন আইটেম (★) যোগ হলো
  └───┴───┴───┴───┴───┴───┴───┴───┘
  page1 ──┘     page2 ──┘
  → আইটেম ৩ দুবার দেখাবে! 😬

Cursor-Based (নির্ভরযোগ্য):
  GET /api/products?after=cursor_abc&limit=20
  → WHERE id > cursor_abc LIMIT 20

  → নতুন আইটেম যোগ হলেও cursor সঠিক পজিশন ধরে রাখে ✅
```

```javascript
// ✅ Good: React Query দিয়ে Infinite Scroll — Daraz প্রোডাক্ট লিস্ট
import { useInfiniteQuery } from '@tanstack/react-query';

function useProductList(category) {
  return useInfiniteQuery({
    queryKey: ['products', category],
    queryFn: ({ pageParam }) =>
      fetch(`/api/products?category=${category}&cursor=${pageParam || ''}&limit=20`)
        .then(r => r.json()),
    getNextPageParam: (lastPage) => lastPage.nextCursor ?? undefined,
    staleTime: 5 * 60 * 1000,
  });
}

function ProductList({ category }) {
  const {
    data, fetchNextPage, hasNextPage,
    isFetchingNextPage, isLoading
  } = useProductList(category);

  // Intersection Observer দিয়ে infinite scroll
  const observerRef = useRef(null);
  const lastItemRef = useCallback((node) => {
    if (isFetchingNextPage) return;
    if (observerRef.current) observerRef.current.disconnect();

    observerRef.current = new IntersectionObserver((entries) => {
      if (entries[0].isIntersecting && hasNextPage) {
        fetchNextPage();
      }
    });

    if (node) observerRef.current.observe(node);
  }, [isFetchingNextPage, hasNextPage, fetchNextPage]);

  if (isLoading) return <p>লোড হচ্ছে...</p>;

  const allProducts = data.pages.flatMap(page => page.items);

  return (
    <div>
      {allProducts.map((product, i) => (
        <ProductCard
          key={product.id}
          ref={i === allProducts.length - 1 ? lastItemRef : null}
          product={product}
        />
      ))}
      {isFetchingNextPage && <p>আরও লোড হচ্ছে...</p>}
    </div>
  );
}
```

```php
<?php
// ✅ Good: PHP Laravel — Cursor Pagination (Daraz-স্টাইল)
public function index(Request $request)
{
    $products = Product::query()
        ->where('category_id', $request->category)
        ->where('is_active', true)
        ->orderBy('id', 'desc')
        ->cursorPaginate(20);

    return response()->json([
        'items'      => ProductResource::collection($products),
        'nextCursor' => $products->nextCursor()?->encode(),
        'hasMore'    => $products->hasMorePages(),
    ]);
}
```

---

# 📖 ৬. রিয়েল-টাইম ডেটা প্যাটার্নস (Real-time Data)

## 📊 Polling vs WebSocket vs SSE তুলনা

```
Short Polling:
  Client ──GET──▶ Server    "নতুন কিছু আছে?"  "না"
  Client ──GET──▶ Server    "নতুন কিছু আছে?"  "না"
  Client ──GET──▶ Server    "নতুন কিছু আছে?"  "হ্যাঁ! এই নাও"
  → সহজ কিন্তু অপচয় — বেশিরভাগ রিকোয়েস্ট খালি ফেরে

Long Polling:
  Client ──GET──▶ Server    (সার্ভার ধরে রাখে... অপেক্ষা... অপেক্ষা...)
  Client ◀──────  Server    "নতুন ডেটা!" (ডেটা পেলে রেসপন্স দেয়)
  Client ──GET──▶ Server    (আবার কানেক্ট)
  → কম অপচয়, কিন্তু সার্ভারে connection ধরে রাখতে হয়

WebSocket:
  Client ◀═══════▶ Server   (দ্বিমুখী persistent connection)
  Client ◀── push  Server   "নতুন মেসেজ!"
  Client ── send ▶ Server   "আমি টাইপ করছি..."
  Client ◀── push  Server   "আরেকটি মেসেজ!"
  → সবচেয়ে দ্রুত, দ্বিমুখী, কিন্তু জটিল ও ব্যয়বহুল

SSE (Server-Sent Events):
  Client ──GET──▶ Server    (একটি connection)
  Client ◀── event Server   "আপডেট ১"
  Client ◀── event Server   "আপডেট ২"
  Client ◀── event Server   "আপডেট ৩"
  → সার্ভার থেকে ক্লায়েন্ট — একমুখী, সহজ, HTTP-ভিত্তিক
```

```
কোনটি কখন ব্যবহার করবেন?

┌─────────────────────┬───────────────┬──────────────┬─────────────┐
│     ব্যবহারক্ষেত্র   │   Polling     │  WebSocket   │    SSE      │
├─────────────────────┼───────────────┼──────────────┼─────────────┤
│ Pathao রাইড ট্র্যাক  │               │      ✅      │             │
│ bKash ট্রানজেকশন    │     ✅        │              │             │
│ Prothom Alo লাইভ    │               │              │     ✅      │
│ চ্যাট অ্যাপ          │               │      ✅      │             │
│ ড্যাশবোর্ড মেট্রিক্স │               │              │     ✅      │
│ নোটিফিকেশন          │     ✅        │      ✅      │     ✅      │
└─────────────────────┴───────────────┴──────────────┴─────────────┘
```

```javascript
// ✅ Good: Short Polling — bKash ট্রানজেকশন স্ট্যাটাস চেক
function useTransactionStatus(txnId) {
  return useQuery({
    queryKey: ['txn-status', txnId],
    queryFn: () => fetch(`/api/bkash/txn/${txnId}`).then(r => r.json()),
    refetchInterval: (query) => {
      // সফল বা ব্যর্থ হলে polling বন্ধ
      const status = query.state.data?.status;
      if (status === 'completed' || status === 'failed') return false;
      return 3000; // ৩ সেকেন্ড পর পর চেক
    },
  });
}

// ✅ Good: WebSocket — Pathao রাইড ট্র্যাকিং
function useRideTracking(rideId) {
  const [location, setLocation] = useState(null);

  useEffect(() => {
    const ws = new WebSocket(`wss://ws.pathao.com/rides/${rideId}`);

    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      setLocation({ lat: data.lat, lng: data.lng });
    };

    ws.onclose = () => {
      // ৫ সেকেন্ড পর reconnect
      setTimeout(() => useRideTracking(rideId), 5000);
    };

    return () => ws.close();
  }, [rideId]);

  return location;
}

// ✅ Good: SSE — Prothom Alo লাইভ আপডেট
function useLiveNewsFeed() {
  const [articles, setArticles] = useState([]);

  useEffect(() => {
    const eventSource = new EventSource('/api/news/live');

    eventSource.addEventListener('new-article', (e) => {
      const article = JSON.parse(e.data);
      setArticles(prev => [article, ...prev]);
    });

    eventSource.onerror = () => {
      eventSource.close();
      setTimeout(() => useLiveNewsFeed(), 10000); // ১০ সেকেন্ড পর reconnect
    };

    return () => eventSource.close();
  }, []);

  return articles;
}
```

---

# 📖 ৭. API Layer আর্কিটেকচার (API Layer Architecture)

## 📊 Repository Pattern

সরাসরি কম্পোনেন্ট থেকে `fetch()` কল করা অগোছালো এবং রক্ষণাবেক্ষণে কঠিন। Repository Pattern API কলগুলোকে আলাদা লেয়ারে সংগঠিত করে।

```
❌ Bad: সরাসরি কম্পোনেন্টে fetch
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Component A  │     │ Component B  │     │ Component C  │
│  fetch(...)  │     │  fetch(...)  │     │  fetch(...)  │
│  fetch(...)  │     │  fetch(...)  │     │  fetch(...)  │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                    │
       └────────────────────┼────────────────────┘
                            ▼
                     ┌──────────┐
                     │   API    │
                     └──────────┘

✅ Good: Repository Pattern
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Component A  │     │ Component B  │     │ Component C  │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                    │
       ▼                    ▼                    ▼
┌──────────────────────────────────────────────────────┐
│              API Repository Layer                     │
│  ┌────────────┐  ┌─────────────┐  ┌──────────────┐  │
│  │ ProductRepo│  │  OrderRepo  │  │  PaymentRepo │  │
│  └─────┬──────┘  └──────┬──────┘  └───────┬──────┘  │
│        └────────────────┼──────────────────┘         │
│                         ▼                            │
│               ┌──────────────────┐                   │
│               │   API Client     │                   │
│               │  (Axios/Guzzle)  │                   │
│               └────────┬─────────┘                   │
└────────────────────────┼─────────────────────────────┘
                         ▼
                  ┌──────────┐
                  │   API    │
                  └──────────┘
```

```javascript
// ✅ Good: JavaScript — API Client ও Repository Pattern

// --- api/client.js ---
import axios from 'axios';

const apiClient = axios.create({
  baseURL: process.env.REACT_APP_API_URL,
  timeout: 15000,
});

apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

export default apiClient;

// --- api/repositories/productRepository.js ---
import apiClient from '../client';

export const productRepository = {
  getAll: (params) =>
    apiClient.get('/products', { params }).then(r => r.data),

  getById: (id) =>
    apiClient.get(`/products/${id}`).then(r => r.data),

  search: (query, signal) =>
    apiClient.get('/products/search', { params: { q: query }, signal }).then(r => r.data),

  create: (data) =>
    apiClient.post('/products', data).then(r => r.data),

  update: (id, data) =>
    apiClient.put(`/products/${id}`, data).then(r => r.data),

  delete: (id) =>
    apiClient.delete(`/products/${id}`).then(r => r.data),
};

// --- hooks/useProducts.js ---
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { productRepository } from '../api/repositories/productRepository';

export function useProducts(params) {
  return useQuery({
    queryKey: ['products', params],
    queryFn: () => productRepository.getAll(params),
  });
}

export function useCreateProduct() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: productRepository.create,
    onSuccess: () => queryClient.invalidateQueries({ queryKey: ['products'] }),
  });
}
```

```php
<?php
// ✅ Good: PHP — Repository Pattern (Guzzle)

// app/Services/ApiClient.php
namespace App\Services;

use GuzzleHttp\Client;

class ApiClient
{
    private Client $client;

    public function __construct()
    {
        $this->client = new Client([
            'base_uri' => config('services.api.base_url'),
            'timeout'  => 15,
            'headers'  => [
                'Accept'       => 'application/json',
                'Content-Type' => 'application/json',
            ],
        ]);
    }

    public function get(string $uri, array $query = []): array
    {
        $response = $this->client->get($uri, ['query' => $query]);
        return json_decode($response->getBody(), true);
    }

    public function post(string $uri, array $data): array
    {
        $response = $this->client->post($uri, ['json' => $data]);
        return json_decode($response->getBody(), true);
    }
}

// app/Repositories/PaymentRepository.php
namespace App\Repositories;

use App\Services\ApiClient;

class PaymentRepository
{
    public function __construct(private ApiClient $client) {}

    public function createPayment(array $data): array
    {
        return $this->client->post('/payments/create', $data);
    }

    public function getStatus(string $txnId): array
    {
        return $this->client->get("/payments/{$txnId}/status");
    }

    public function refund(string $txnId, float $amount): array
    {
        return $this->client->post("/payments/{$txnId}/refund", [
            'amount' => $amount,
        ]);
    }
}
```

## 📊 Error Normalization

```javascript
// ✅ Good: সব API error কে একটি consistent format-এ রূপান্তর
class ApiError {
  constructor(status, message, code, details = null) {
    this.status = status;
    this.message = message;
    this.code = code;
    this.details = details;
  }

  static fromAxiosError(error) {
    if (error.response) {
      return new ApiError(
        error.response.status,
        error.response.data?.message || 'সার্ভার ত্রুটি',
        error.response.data?.code || 'UNKNOWN_ERROR',
        error.response.data?.errors
      );
    }
    if (error.code === 'ECONNABORTED') {
      return new ApiError(408, 'রিকোয়েস্ট সময়সীমা অতিক্রম', 'TIMEOUT');
    }
    return new ApiError(0, 'নেটওয়ার্ক সংযোগ নেই', 'NETWORK_ERROR');
  }

  get isRetryable() {
    return this.status >= 500 || this.code === 'TIMEOUT' || this.code === 'NETWORK_ERROR';
  }
}
```

---

# 📖 ৮. পারফরম্যান্স প্যাটার্নস (Performance Patterns)

## 📊 Request Deduplication

একই API একই সময়ে একাধিকবার কল হলে শুধু একটি রিকোয়েস্ট পাঠাও:

```javascript
// ✅ Good: Request Deduplication
const pendingRequests = new Map();

async function deduplicatedFetch(url, options = {}) {
  const key = `${options.method || 'GET'}:${url}`;

  if (pendingRequests.has(key)) {
    return pendingRequests.get(key); // আগের Promise-ই ফেরত দাও
  }

  const promise = fetch(url, options)
    .then(r => r.json())
    .finally(() => pendingRequests.delete(key));

  pendingRequests.set(key, promise);
  return promise;
}

// তিনটি কম্পোনেন্ট একসাথে কল করলেও একটিমাত্র HTTP রিকোয়েস্ট যাবে
deduplicatedFetch('/api/user/profile');
deduplicatedFetch('/api/user/profile'); // ← duplicate — আগেরটির Promise পাবে
deduplicatedFetch('/api/user/profile'); // ← duplicate — আগেরটির Promise পাবে
```

## 📊 Parallel vs Waterfall Requests

```
❌ Waterfall (একটির পর একটি — ধীর):
  ┌──────────┐
  │ Products │ ─────────────────▶ 800ms
  └──────────┘
               ┌──────────┐
               │ Reviews  │ ───────────▶ 600ms
               └──────────┘
                            ┌──────────┐
                            │  User    │ ────▶ 300ms
                            └──────────┘
  Total: 800 + 600 + 300 = 1700ms 😢

✅ Parallel (একসাথে সব — দ্রুত):
  ┌──────────┐
  │ Products │ ─────────────────▶ 800ms
  ├──────────┤
  │ Reviews  │ ───────────▶ 600ms
  ├──────────┤
  │  User    │ ────▶ 300ms
  └──────────┘
  Total: max(800, 600, 300) = 800ms ✅
```

```javascript
// ❌ Bad: Waterfall — একটি শেষ হওয়ার পর আরেকটি
async function loadDashboard() {
  const products = await fetch('/api/products').then(r => r.json());
  const reviews = await fetch('/api/reviews').then(r => r.json());
  const user = await fetch('/api/user').then(r => r.json());
  return { products, reviews, user };
}

// ✅ Good: Parallel — সব একসাথে
async function loadDashboard() {
  const [products, reviews, user] = await Promise.all([
    fetch('/api/products').then(r => r.json()),
    fetch('/api/reviews').then(r => r.json()),
    fetch('/api/user').then(r => r.json()),
  ]);
  return { products, reviews, user };
}

// ✅ Better: Promise.allSettled — একটি ব্যর্থ হলেও বাকিগুলো পাওয়া যায়
async function loadDashboard() {
  const results = await Promise.allSettled([
    fetch('/api/products').then(r => r.json()),
    fetch('/api/reviews').then(r => r.json()),
    fetch('/api/user').then(r => r.json()),
  ]);

  return {
    products: results[0].status === 'fulfilled' ? results[0].value : [],
    reviews:  results[1].status === 'fulfilled' ? results[1].value : [],
    user:     results[2].status === 'fulfilled' ? results[2].value : null,
  };
}
```

## 📊 Prefetching

```javascript
// ✅ Good: React Query Prefetch — Daraz প্রোডাক্ট হোভার করলে ডিটেইল প্রিফেচ
function ProductCard({ product }) {
  const queryClient = useQueryClient();

  const handleMouseEnter = () => {
    queryClient.prefetchQuery({
      queryKey: ['product', product.id],
      queryFn: () => productRepository.getById(product.id),
      staleTime: 5 * 60 * 1000,
    });
  };

  return (
    <Link
      to={`/product/${product.id}`}
      onMouseEnter={handleMouseEnter}
    >
      <h3>{product.name}</h3>
      <p>৳ {product.price}</p>
    </Link>
  );
}
```

## 📊 Request Batching

```javascript
// ✅ Good: একাধিক ছোট রিকোয়েস্ট একসাথে ব্যাচে পাঠানো
class RequestBatcher {
  constructor(batchFn, { maxSize = 20, delayMs = 50 } = {}) {
    this.batchFn = batchFn;
    this.maxSize = maxSize;
    this.delayMs = delayMs;
    this.queue = [];
    this.timer = null;
  }

  add(item) {
    return new Promise((resolve, reject) => {
      this.queue.push({ item, resolve, reject });

      if (this.queue.length >= this.maxSize) {
        this.flush();
      } else if (!this.timer) {
        this.timer = setTimeout(() => this.flush(), this.delayMs);
      }
    });
  }

  async flush() {
    clearTimeout(this.timer);
    this.timer = null;

    const batch = this.queue.splice(0, this.maxSize);
    const items = batch.map(b => b.item);

    try {
      const results = await this.batchFn(items);
      batch.forEach((b, i) => b.resolve(results[i]));
    } catch (error) {
      batch.forEach(b => b.reject(error));
    }
  }
}

// ব্যবহার — ২০টি ইউজার ডেটা একটি রিকোয়েস্টে আনা
const userBatcher = new RequestBatcher(
  (ids) => fetch('/api/users/batch', {
    method: 'POST',
    body: JSON.stringify({ ids }),
    headers: { 'Content-Type': 'application/json' }
  }).then(r => r.json())
);

// প্রতিটি কম্পোনেন্ট আলাদাভাবে কল করে, কিন্তু ব্যাচে যায়
const user1 = await userBatcher.add('user_001');
const user2 = await userBatcher.add('user_002');
```

---

# 📖 ৯. কখন ব্যবহার করবেন / করবেন না

## 🎯 ডেটা ফেচিং লাইব্রেরি নির্বাচন

```
আপনার প্রকল্পের জন্য কোনটি সঠিক?

                        শুরু করুন
                          │
                ┌─────────▼──────────┐
                │  Redux ব্যবহার     │
                │  করছেন?            │
                └──┬──────────────┬──┘
                 হ্যাঁ            না
                   │              │
                   ▼              │
            ┌──────────┐         │
            │ RTK Query│         │
            └──────────┘         │
                          ┌──────▼──────────┐
                          │  প্রকল্প কতটা   │
                          │  জটিল?          │
                          └──┬──────────┬───┘
                          সহজ         জটিল
                            │           │
                            ▼           ▼
                      ┌─────────┐  ┌────────────┐
                      │   SWR   │  │ React Query│
                      └─────────┘  └────────────┘
```

| পরিস্থিতি | প্রস্তাবিত টুল | কারণ |
|-----------|----------------|------|
| ছোট প্রকল্প, কম API কল | `fetch` + `useState` | কোনো dependency নেই |
| মাঝারি প্রকল্প, ক্যাশিং দরকার | SWR | সহজ, হালকা (~4KB) |
| বড় প্রকল্প, জটিল ক্যাশিং | TanStack Query | শক্তিশালী, DevTools আছে |
| Redux ইকোসিস্টেম | RTK Query | Redux-এর সাথে সামঞ্জস্যপূর্ণ |
| GraphQL API | Apollo Client | GraphQL-এ সেরা |
| PHP সার্ভার-টু-সার্ভার | Guzzle | PHP-র স্ট্যান্ডার্ড HTTP ক্লায়েন্ট |
| Node.js সার্ভার-টু-সার্ভার | Axios / `node-fetch` | সার্ভার সাইডে জনপ্রিয় |

## 🎯 রিয়েল-টাইম প্যাটার্ন নির্বাচন

| পরিস্থিতি | প্যাটার্ন | কারণ |
|-----------|----------|------|
| প্রতি ৩০ সেকেন্ডে আপডেট যথেষ্ট | Short Polling | সবচেয়ে সহজ |
| তাৎক্ষণিক একমুখী আপডেট | SSE | HTTP-ভিত্তিক, সহজ |
| দ্বিমুখী real-time (চ্যাট) | WebSocket | দ্রুত, দ্বিমুখী |
| অফলাইন সাপোর্ট দরকার | Service Worker + Cache | PWA-তে অপরিহার্য |

## 🎯 ক্যাশিং স্ট্র্যাটেজি নির্বাচন

| ডেটার ধরন | স্ট্র্যাটেজি | উদাহরণ |
|-----------|------------|--------|
| খুব কম পরিবর্তন হয় | Long cache (1 day+) | ক্যাটাগরি লিস্ট |
| মাঝে মাঝে পরিবর্তন | SWR + short cache | প্রোডাক্ট তালিকা |
| প্রায়ই পরিবর্তন | No cache / very short | ব্যালেন্স, অর্ডার স্ট্যাটাস |
| ব্যবহারকারী-নির্দিষ্ট | Private cache | প্রোফাইল, সেটিংস |

---

# 📖 ১০. অ্যান্টি-প্যাটার্নস (Anti-Patterns)

## ❌ Bad / ✅ Good

### ❌ ১. কম্পোনেন্টে সরাসরি API URL

```javascript
// ❌ Bad: হার্ডকোড URL — পরিবর্তন করা কঠিন
function ProductList() {
  const [data, setData] = useState([]);
  useEffect(() => {
    fetch('https://api.daraz.com.bd/v2/products?limit=20')
      .then(r => r.json())
      .then(setData);
  }, []);
}

// ✅ Good: Repository + Environment Variable
function ProductList() {
  const { data } = useProducts({ limit: 20 });
}
```

### ❌ ২. Error Handling ভুলে যাওয়া

```javascript
// ❌ Bad: কোনো error handling নেই
const data = await fetch('/api/data').then(r => r.json());

// ✅ Good: সঠিক error handling
try {
  const response = await fetch('/api/data');
  if (!response.ok) throw new Error(`HTTP ${response.status}`);
  const data = await response.json();
} catch (error) {
  showToast('ডেটা লোড করা যায়নি — আবার চেষ্টা করুন');
  logError(error);
}
```

### ❌ ৩. Waterfall রিকোয়েস্ট (অপ্রয়োজনীয়)

```javascript
// ❌ Bad: একটির পর একটি — ধীর!
const user = await fetchUser();
const orders = await fetchOrders(); // user-এর উপর নির্ভর করে না!
const notifications = await fetchNotifications();

// ✅ Good: সমান্তরালে চালাও
const [user, orders, notifications] = await Promise.all([
  fetchUser(),
  fetchOrders(),
  fetchNotifications(),
]);
```

### ❌ ৪. retry ছাড়া গুরুত্বপূর্ণ API কল

```javascript
// ❌ Bad: bKash পেমেন্ট — একবার ব্যর্থ হলেই হাল ছেড়ে দেওয়া
const result = await fetch('/api/bkash/payment', { method: 'POST', body });

// ✅ Good: retry + timeout + error handling
const result = await fetchWithRetry('/api/bkash/payment', {
  method: 'POST',
  body,
  timeout: 30000,
}, 3); // ৩ বার retry
```

### ❌ ৫. অসীম রিফেচ (Infinite Refetch Loop)

```javascript
// ❌ Bad: object/array dependency — প্রতি রেন্ডারে নতুন reference!
function Products({ filters }) {
  const { data } = useQuery({
    queryKey: ['products', filters], // filters = { cat: 'phone' }
    queryFn: () => productRepository.getAll(filters),
  });
  // ⚠️ filters প্রতি রেন্ডারে নতুন object → queryKey পরিবর্তন → অসীম refetch!
}

// ✅ Good: stable queryKey ব্যবহার করুন
function Products({ category, page }) {
  const { data } = useQuery({
    queryKey: ['products', category, page], // primitive values — stable
    queryFn: () => productRepository.getAll({ category, page }),
  });
}
```

### ❌ ৬. Global State-এ সার্ভার ডেটা রাখা

```javascript
// ❌ Bad: Redux store-এ সার্ভার ডেটা — ম্যানুয়াল ক্যাশ, stale ডেটা সমস্যা
dispatch(setProducts(await fetchProducts()));
// কে invalidate করবে? কখন refetch হবে? stale কিনা কীভাবে বুঝবে?

// ✅ Good: React Query — সার্ভার স্টেট ম্যানেজমেন্ট লাইব্রেরি ব্যবহার করুন
const { data } = useQuery({ queryKey: ['products'], queryFn: fetchProducts });
// ক্যাশিং, refetch, stale management — সব অটোমেটিক!
```

### ❌ ৭. টোকেন রিফ্রেশ না করা

```php
<?php
// ❌ Bad: Token expire হলে ব্যবহারকারীকে আবার লগইন করাও
if ($response->getStatusCode() === 401) {
    return redirect('/login');
}

// ✅ Good: Automatic token refresh
if ($response->getStatusCode() === 401) {
    $newToken = $this->refreshToken();
    // আবার একই রিকোয়েস্ট পাঠাও নতুন token দিয়ে
    return $client->request($method, $uri, [
        'headers' => ['Authorization' => 'Bearer ' . $newToken],
    ]);
}
```

---

## 📊 সারসংক্ষেপ — সব প্যাটার্ন একনজরে

```
┌────────────────────────────────────────────────────────────────┐
│                API Consumption Patterns চিটশিট                 │
├────────────────────────┬───────────────────────────────────────┤
│ Basic Fetch            │ fetch + AbortController + timeout     │
│ HTTP Client            │ Axios (JS) / Guzzle (PHP)            │
│ Server State           │ React Query / SWR / RTK Query        │
│ Caching                │ In-memory → HTTP Cache → SW Cache    │
│ Retry                  │ Exponential backoff + jitter          │
│ Circuit Breaker        │ CLOSED → OPEN → HALF_OPEN            │
│ Optimistic Update      │ UI আগে → সার্ভার পরে → ব্যর্থে rollback│
│ Pagination             │ Cursor-based > Offset-based           │
│ Real-time              │ Polling / WebSocket / SSE             │
│ Architecture           │ Repository Pattern + API Client       │
│ Deduplication          │ Pending request Map                   │
│ Batching               │ Queue + flush timer                   │
│ Prefetch               │ queryClient.prefetchQuery()           │
│ Parallel               │ Promise.all / Promise.allSettled      │
│ Error Handling         │ Normalized ApiError class             │
└────────────────────────┴───────────────────────────────────────┘
```

---

## 🔗 সম্পর্কিত টপিকস

- [REST API ডিজাইন](./rest-api.md) — API তৈরির মৌলিক নীতি
- [GraphQL](./graphql.md) — Apollo Client দিয়ে consumption
- [এরর হ্যান্ডলিং](./error-handling.md) — API error কৌশল
- [পেজিনেশন স্ট্র্যাটেজি](./pagination-strategies.md) — সার্ভার-সাইড pagination
- [রেট লিমিটিং](./rate-limiting.md) — ক্লায়েন্ট সাইড throttling
- [ওয়েবহুক](./webhook.md) — সার্ভার-টু-সার্ভার real-time

---

> **মনে রাখুন:** ভালো API consumption শুধু `fetch()` কল নয় — এটি caching, retry, error handling, cancellation, এবং user experience-এর সমন্বয়। প্রোডাকশন অ্যাপ্লিকেশনে এই প্যাটার্নগুলো সঠিকভাবে প্রয়োগ করলে আপনার অ্যাপ দ্রুত, নির্ভরযোগ্য এবং ব্যবহারকারী-বান্ধব হবে। 🚀
