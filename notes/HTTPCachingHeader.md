# Question

Can you spot the problem?
Users complain they're getting stale data.
The API works perfectly.
Why?
`GET /api/products
Cache-Control: public, max-age=86400
Or an even stronger version with more`

# Answer
# The Big Picture & Analogy

Imagine ordering pizza from a shop, and telling the delivery driver: **"Keep this pizza and hand it to anyone who asks for it over the next 24 hours, without ever checking back with the shop."**

The problem: if the shop **changes its menu, adds a new topping, or raises prices** during those 24 hours — customers who get pizza from that driver (holding the old stock) **will have absolutely no way of knowing the menu changed**, because the driver never checked back in as instructed.

This is exactly what happens with `Cache-Control: public, max-age=86400` — we're telling **everyone in the chain** (browsers, CDNs, intermediate proxies): **"Store this response and use it for 24 hours (86400 seconds), without ever asking the server again."**

# Why Do We Need It? (Why "the API is correct" but users see stale data)

This header has two critical parts:

**`public`** — declares this response is **not private data**, and can be cached at **any point along the delivery path**, not just the individual user's browser, but also:
- Shared proxy caches (e.g. a corporate network's central proxy)
- **CDNs** (Cloudflare, CloudFront, Fastly)
- Some ISP-level caches

**`max-age=86400`** — states that the cached copy is **"considered fresh" for up to 86400 seconds (24 hours)** — during this window, **anyone holding this response will never hit the origin server again, not even once.**

**Here's the answer to the puzzle:** a team testing the API directly (e.g. via Postman or curl, bypassing cache) will see that **"the API is correct, returning the latest data,"** because these tools often skip caching or use cache-busting headers without realizing it. But a **real user browsing normally** gets a response that a **CDN or browser cached 20 hours ago**, which hasn't yet exceeded its `max-age` — even though the actual product data in the database has since changed.

# Core Logic & How It Works

Let's trace the actual HTTP caching flow:

```
First request:
Browser -> CDN (no cache) -> Origin Server -> responds with 
"Cache-Control: public, max-age=86400"

CDN caches this response immediately (since it's public)
The browser caches it too

Second request (5 minutes later, same or different user):
Browser checks local cache first -> max-age not yet exceeded -> uses the cached value, doesn't even send a request
(or if browser cache expired but the CDN's hasn't -> CDN answers from its own cache, never touching Origin)

** Meanwhile, if an admin changes a product's price in the database **
The Origin Server has correct, updated data — but no one ever asks it, since everyone's using stale cache

Not until the full 86400 seconds elapses does a fresh request actually reach Origin
```

**Key insight juniors often miss:** `max-age` doesn't mean "revalidate every 24 hours to check for changes" — it means **"never hit the server at all until the time expires."** It's a 100% trust arrangement, with zero verification in between.

# Trade-offs & When to Use (The Fix)

**Fix A: Reduce max-age to match how frequently the data actually changes**
```
Cache-Control: public, max-age=300  // 5 minutes instead of 24 hours
```
Good for data that doesn't change frequently, but isn't truly static forever.

**Fix B: Use `stale-while-revalidate` (a better option)**
```
Cache-Control: public, max-age=60, stale-while-revalidate=86400
```
Meaning: use cache for the first 60 seconds (fast), but afterward, **serve the user the stale cached copy immediately while simultaneously fetching fresh data from origin in the background** — the next request gets fresh data, without the first user needing to wait.

**Fix C: Active cache invalidation (most precise, most complex)**
```java
// when an admin updates a product's price -> purge cache immediately
public void updateProductPrice(Long productId, BigDecimal newPrice) {
    productRepo.updatePrice(productId, newPrice);
    cdnService.purgeCache("/api/products/" + productId); // instruct the CDN to purge immediately
}
```
Most accurate, but requires infrastructure supporting real-time purge commands (e.g. Cloudflare API, Fastly Instant Purge).

**Fix D: ETag + conditional requests (efficient revalidation)**
```
Response Header: ETag: "abc123-v5"

Next request:
GET /api/products
If-None-Match: "abc123-v5"

If unchanged -> server responds 304 Not Modified (no body sent, saves bandwidth)
If changed -> server responds 200 with fresh data + a new ETag
```

| Approach | Freshness | Performance | Complexity |
|---|---|---|---|
| **Reduce max-age** | Better, but a staleness window remains | Some cache-hit rate loss | Very low (just tweak the header) |
| **stale-while-revalidate** | Very good (user never waits, data updates quickly) | Very high (still fast) | Moderate |
| **Active purge** | Best possible (precise, real-time) | Very high | High (needs supporting infra) |
| **ETag revalidation** | Accurate + saves bandwidth | Moderate (still hits server every time, but small response) | Moderate |

# Real-World Scenario (Ecommerce Domain)

The exact scenario this problem describes: **a Flash Sale where prices change in real time**

```
Problem: /api/products has Cache-Control: public, max-age=86400
Flash Sale starts at 8:00pm, Product A's price drops from 1000 to 500
But a user who visited at 7:00pm (before the sale) already has an old cached response
Once the sale starts, that user still sees the price as 1000 (CDN cache hasn't expired)
The user misses the discount, or worse, checks out and hits a price mismatch (caught during backend validation at submit)
```

```java
// Fix suited for Flash Sales: differentiate cache strategy by data volatility
@GetMapping("/products/{id}")
public ResponseEntity<ProductDto> getProduct(@PathVariable Long id) {
    Product product = productService.findById(id);
    
    CacheControl cacheControl = product.isOnFlashSale() 
        ? CacheControl.maxAge(30, TimeUnit.SECONDS)  // Flash sale: very short cache
        : CacheControl.maxAge(1, TimeUnit.HOURS);    // normal products: longer cache is fine
    
    return ResponseEntity.ok()
        .cacheControl(cacheControl)
        .body(productMapper.toDto(product));
}
```

# Lead's Key Takeaway

> **"When users complain about 'stale' data even though the API tests correctly 100% of the time, always suspect the caching layer first — especially CDN/browser caches sitting 'in transit,' which teams often forget to check because they focus only on the backend."**
>
> A good Lead never sets `max-age` on the assumption "longer is always better for performance," without considering **"how frequently the underlying data actually changes."** Always ask first: **"if this data changes suddenly, how long is it acceptable for users to see stale data?"** — then set the cache duration to match that answer, rather than maxing it out purely for performance's sake.
