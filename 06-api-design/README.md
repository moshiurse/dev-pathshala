# 📘 API ডিজাইন

> সুন্দর, স্কেলেবল ও সিকিউর API তৈরির নীতি ও কৌশল।
> প্রতিটি টপিকে PHP (Laravel) ও JavaScript (Express/Node) উদাহরণ, OpenAPI spec, এবং production best practices সহ গভীর বিশ্লেষণ।

---

## 📂 টপিক সূচি

| # | বিষয় | ফাইল | মূল বিষয়বস্তু |
|---|-------|------|--------------|
| ১ | REST API ডিজাইন | [rest-api.md](./rest-api.md) | Fielding's 6 Constraints, Richardson Maturity, HATEOAS, Pagination, Idempotency |
| ২ | GraphQL | [graphql.md](./graphql.md) | Schema/SDL, Queries/Mutations/Subscriptions, Lighthouse + Apollo, DataLoader, Federation |
| ৩ | gRPC | [grpc.md](./grpc.md) | Protocol Buffers, 4 Service Types, HTTP/2, Interceptors, mTLS, Microservices |
| ৪ | API অথেনটিকেশন | [authentication.md](./authentication.md) | JWT, OAuth 2.0, HMAC, mTLS, RBAC/ABAC, Token Refresh, MFA |
| ৫ | API ভার্সনিং | [versioning.md](./versioning.md) | URI/Header/Query Versioning, Deprecation Lifecycle, Backward Compatibility |
| ৬ | এরর হ্যান্ডলিং | [error-handling.md](./error-handling.md) | RFC 7807, Error Codes System, Localization (বাংলা), Sentry, Graceful Degradation |
| ৭ | API ডকুমেন্টেশন | [documentation.md](./documentation.md) | OpenAPI 3.0, Swagger/Scribe/Scramble, Postman, SDK Generation, Developer Portal |
| ৮ | রেট লিমিটিং | [rate-limiting.md](./rate-limiting.md) | Token Bucket, Sliding Window, API Throttling |
| ৯ | পেজিনেশন স্ট্র্যাটেজি | [pagination-strategies.md](./pagination-strategies.md) | Offset vs Cursor, Keyset Pagination |
| ১০ | ওয়েবহুক | [webhook.md](./webhook.md) | Retry Strategies, Signature Verification, Idempotency |
| ১১ | API কনসাম্পশন প্যাটার্ন | [api-consumption-patterns.md](./api-consumption-patterns.md) | React Query, SWR, Caching, Retry, Polling, WebSocket, Guzzle |
| ১২ | Backend-for-Frontend (BFF) | [bff-pattern.md](./bff-pattern.md) | Per-client API, Aggregation, Transformation, Auth offload, Daraz Web vs Mobile BFF |
| ১৩ | Server-Sent Events (SSE) | [server-sent-events.md](./server-sent-events.md) | text/event-stream, EventSource, Last-Event-ID, Nginx buffering, Pathao Foods order tracking |
| ১৪ | GraphQL Federation | [graphql-federation.md](./graphql-federation.md) | Apollo Federation v2, Subgraphs, @key/@requires/@provides, Supergraph composition |
| ১৫ | API Gateway | [api-gateway.md](./api-gateway.md) | Kong/Envoy/AWS APIGW, Routing, JWT, Rate Limit, Aggregation, Circuit Breaker, mTLS, Daraz/Pathao/bKash |

---

## 🗺️ API ডিজাইন সম্পর্ক

```
    ┌─────────────────────────────────────┐
    │         API Protocol Layer           │
    │  REST ◄──► GraphQL ◄──► gRPC        │
    └──────────────┬──────────────────────┘
                   │
    ┌──────────────▼──────────────────────┐
    │        Cross-cutting Concerns        │
    │                                      │
    │  Authentication ─── Versioning       │
    │       │                  │           │
    │       ▼                  ▼           │
    │  Error Handling ─── Documentation    │
    └──────────────────────────────────────┘
```

---

## 🆚 API Protocol তুলনা

| মানদণ্ড | REST | GraphQL | gRPC |
|---------|------|---------|------|
| Protocol | HTTP/1.1+ | HTTP | HTTP/2 |
| Data Format | JSON | JSON | Protobuf (binary) |
| Schema | OpenAPI (optional) | SDL (required) | .proto (required) |
| Use Case | Public API | Flexible queries | Microservices |
| Performance | ভালো | ভালো | সবচেয়ে দ্রুত |

---

## 📖 প্রতিটি টপিকে যা পাবেন

- ✅ বাংলায় বিস্তারিত ব্যাখ্যা ও ASCII ডায়াগ্রাম
- ✅ PHP (Laravel) + JavaScript (Express/Node) কোড উদাহরণ
- ✅ Production-ready implementation patterns
- ✅ Security best practices
- ✅ Advanced scenarios ও Comparison tables
- ✅ বাংলাদেশ context (bKash API, SSLCommerz, BD mobile number validation)
