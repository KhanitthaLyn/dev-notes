# The Big Picture & Analogy

**Caching** means storing a "copy of already-computed/fetched data" somewhere that's faster to access, so you don't have to recompute or re-query the same data every time someone asks for it.

**Analogy:** Think of **jotting down a frequently-used recipe on a piece of paper stuck to your fridge**.

- Without a cache: every time you want to cook pad kra pao, you have to **open a thick cookbook, search for the recipe, and read the whole thing again** (querying the database every time) — even though it's the same recipe you've cooked 100 times already.
- With a cache: you write the frequently-used recipe on a piece of paper on your fridge (cache) — next time, you just **glance at the paper for a second**, no need to open the thick cookbook at all. Much faster.
- But if **the recipe in the cookbook gets updated** (the restaurant improves the recipe) while the paper on your fridge still has the old version — you'll unknowingly keep cooking with a **"stale"** recipe. This is **the core danger of caching.**

A cache doesn't make data "more correct" — it just makes it **"faster to access"** — and that's the trade-off you need to deeply understand before using it.

---

# Why Do We Need It?

**Problems caching solves:**

- **The database gets asked the same question repeatedly:** e.g. an ecommerce homepage showing "top 10 best-selling products" gets viewed thousands of times per minute, but the result only actually changes once a day — every repeated query makes the database do unnecessary work (connecting back to the Query Optimization topic — even with an index, it's still slower than reading from memory).
- **Expensive computations get repeated:** e.g. calculating a "monthly sales report" that requires aggregating millions of rows. If recomputed every time someone views it, that's extremely slow and burns CPU unnecessarily.
- **Slow or rate-limited external APIs:** e.g. calling an external currency exchange rate API that only updates once an hour. Calling it on every single order would be both slow and risk hitting rate limits.

**Why Caching solves this:**
- It changes the complexity from "query the database every time" (slow, resource-heavy) to "read from memory/cache" (much faster, often O(1)).
- It directly reduces load on the database and external services — letting the system scale better without needing more database servers.

---

# Core Logic & How It Works

## The Core Principle: Cache-Aside Pattern (the most common approach)

```java
public ProductResponseDTO getProduct(Long productId) {
    String cacheKey = "product:" + productId;

    // Step 1: always check the cache first
    ProductResponseDTO cached = redisTemplate.opsForValue().get(cacheKey);
    if (cached != null) {
        return cached; // Cache Hit — very fast, never touches the database
    }

    // Step 2: if not in cache (Cache Miss) → query the database
    Product product = productRepository.findById(productId)
        .orElseThrow(() -> new ProductNotFoundException(productId));
    ProductResponseDTO response = toDTO(product);

    // Step 3: store the result in the cache for next time, with a TTL set
    redisTemplate.opsForValue().set(cacheKey, response, Duration.ofMinutes(10));

    return response;
}
```

**Cache Hit vs Cache Miss:**
- **Cache Hit** = the data is found in the cache, no need to query the database at all (very fast).
- **Cache Miss** = data isn't found in the cache, so the database must be queried, and then the result is stored in the cache for next time.

## TTL (Time To Live) — Setting an Expiration on Cached Data

```java
redisTemplate.opsForValue().set(cacheKey, response, Duration.ofMinutes(10));
// this data will be automatically removed from the cache after 10 minutes
// the next request after the TTL expires becomes a Cache Miss, triggering a fresh database query
```

**Why TTL is necessary:** without a TTL at all, data would sit in the cache forever, even after the database has changed — TTL is a "damage limiter" that caps how stale the data can possibly get.

## Cache Invalidation — One of the Hardest Problems in Computer Science

```java
@Transactional
public void updateProductPrice(Long productId, double newPrice) {
    Product product = productRepository.findById(productId).orElseThrow();
    product.setPrice(newPrice);
    productRepository.save(product);

    // critical! must delete the old cache entry immediately when the data changes
    // otherwise the cache keeps showing the old price until the TTL finally expires
    redisTemplate.delete("product:" + productId);
}
```

**There's a famous saying in the industry:** *"There are only two hard things in Computer Science: cache invalidation and naming things."* Knowing "when to delete/update the cache" is harder than it sounds, especially when data can be changed from multiple places (e.g. an admin panel updating prices, a batch job updating stock). Miss even one spot, and the cache silently goes stale without anyone noticing.

---

# Trade-offs & When to Use

## Checklist: What Kind of Data Is a Good Fit for Caching

```
✅ Good fit for Caching:
- Read-heavy relative to writes — e.g. product catalog, category lists
- Data that doesn't change often, or where slight "staleness" is acceptable (eventual consistency is fine)
- Expensive computations (aggregates, reports) whose results can be reused

❌ Poor fit for Caching (or requires extreme caution):
- Data that must always be exactly real-time, e.g. stock quantity during checkout (a wrong cache value could cause overselling)
- Data that changes so frequently the cache barely ever hits (writes outpace TTL expiration) — adds complexity with no real benefit
- Highly sensitive data with severe consequences if wrong, e.g. an account's remaining balance
```

## Dangers of Caching to Watch Out For

**1. Stale Data (outdated data that no longer matches reality)**
```
A product is genuinely out of stock, but the cache still shows "in stock"
→ the user can place the order, but when the actual database is checked, there's nothing there → the order has to be canceled later, a terrible experience
```

**2. Cache Stampede (everyone querying the database at once)**
```
If a popular cache key (e.g. "homepage") expires right when traffic is at its highest
→ thousands of requests simultaneously hit a cache miss and hammer the database all at once
→ the database crashes from load it's never normally seen (even though the cache was supposed to prevent exactly this!)
```

**3. Cache Inconsistency Across Servers:** if using local cache (in-memory on each server) instead of a distributed cache (Redis) — each server instance has its own separate cache, so users see inconsistent data depending on which server their request happens to hit (connecting back to the Stateless Design topic — a local cache is "state" tied to a single server).

## Trade-offs

- **You gain:** massively improved performance (often 10-100x), reduced database load, reduced cost of external API calls.
- **You pay:** risk of stale data, increased complexity (invalidation logic to think through), and an additional point of failure (if the cache server goes down, you need a fallback path to the database).

---

# Real-World Scenario — Identifying 2 Places to Add Caching in Ecommerce

## Location 1: Product Catalog / Product Detail Page

```java
@Service
public class ProductService {
    private final RedisTemplate<String, ProductResponseDTO> redisTemplate;
    private final ProductRepository productRepository;

    public ProductResponseDTO getProduct(Long productId) {
        String cacheKey = "product:" + productId;

        ProductResponseDTO cached = redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            log.debug("Cache HIT for product {}", productId);
            return cached;
        }

        log.debug("Cache MISS for product {}", productId);
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
        ProductResponseDTO response = toDTO(product);

        // TTL of 30 minutes — product details (name, description, images) don't change often
        redisTemplate.opsForValue().set(cacheKey, response, Duration.ofMinutes(30));
        return response;
    }

    @Transactional
    public void updateProduct(Long productId, UpdateProductDTO update) {
        Product product = productRepository.findById(productId).orElseThrow();
        product.update(update);
        productRepository.save(product);

        // invalidate the cache immediately when the data changes
        redisTemplate.delete("product:" + productId);
    }
}
```

**Why this spot was chosen:** product detail data (name, description, images) is **read extremely often** (every single time someone views a product), but **changes infrequently** (admins update product info only a few times a day) — this is a textbook case of a high read:write ratio, making it an ideal caching candidate.

## Location 2: "Best Sellers" / Homepage Featured Products

```java
@Service
public class HomepageService {
    private final RedisTemplate<String, List<ProductResponseDTO>> redisTemplate;
    private final OrderRepository orderRepository;

    public List<ProductResponseDTO> getBestSellers() {
        String cacheKey = "homepage:bestsellers";

        List<ProductResponseDTO> cached = redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            return cached;
        }

        // a very expensive query — aggregating sales across millions of orders
        List<ProductResponseDTO> bestSellers = orderRepository.findTopSellingProducts(10);

        // TTL of 1 hour — best-selling products don't need to be perfectly real-time
        // being "1 hour stale" is acceptable, since it doesn't affect any business-critical decision
        redisTemplate.opsForValue().set(cacheKey, bestSellers, Duration.ofHours(1));
        return bestSellers;
    }
}
```

**Why this spot was chosen:** calculating "best sellers" requires aggregating a massive amount of order data (an expensive query), but **the result doesn't need to update every second** — the #1 best seller an hour ago vs. right now doesn't differ enough to meaningfully affect user experience — this is a case where **eventual consistency** is fully acceptable.

**A spot deliberately chosen NOT to cache — Stock Quantity at Checkout:**

```java
// ❌ never cache this!
public boolean checkStockForCheckout(Long productId, int quantity) {
    // must query the live database every single time, never use the cache
    // because if the cache is wrong (shows "in stock" when it's actually sold out)
    // = a real overselling incident = a serious business problem
    int available = productRepository.getCurrentStock(productId);
    return available >= quantity;
}
```

**This is a critical example of "knowing when NOT to use a cache"** — even though stock queries are called just as frequently, 100% accuracy matters far more than speed at this particular point.

---

# Lead's Key Takeaway

1. **Ask this question before adding a cache anywhere: "If this data is 1 minute (or 1 hour) stale, does it cause real business harm?"** If the answer is "no harm at all" (e.g. best sellers, product descriptions) → cache it freely. If the answer is "severe harm" (e.g. stock at checkout, account balances) → don't cache it, or invalidate it as precisely as possible.
2. **Caching isn't "free speed" — it's trading "real-time accuracy" for "speed."** Every time you add a cache, you must ask yourself "how am I handling stale data?" (Is the TTL appropriate? Is invalidation covered at every point the data can change?) — because a cache added without carefully thinking through invalidation creates bugs that are harder to debug than having no cache at all. "Why isn't this data updating?" is the classic question that follows every carelessly-cached system.
