# Question
One product page. One database call, you'd assume. Loop over 200 related items. Lazy loading quietly fires 200 more queries.
Page takes 9 seconds to load. Nobody touched the query count in code review. It multiplied at runtime, not in the diff.
What's the fix?

# Answer

# The Big Picture & Analogy

Imagine ordering at a restaurant. You ask the waiter for "one menu" (first query = fetch the main product). They bring it over.

But the menu lists 200 "recommended items." Instead of **walking to the kitchen once and asking about all 200 prices in a single trip**, the waiter **walks to the kitchen 200 separate times**, asking about one item each trip — 200 round-trips, when one trip would've been enough.

That's exactly what's happening in the code: **1 main query + N sub-queries fired one-by-one inside a loop = N+1 queries.**

# Why Do We Need It? (Why This Happens Despite "Normal-Looking" Code)

The root cause is **Lazy Loading** — a feature designed so the ORM (like Hibernate) "doesn't fetch a related entity until it's actually accessed," saving memory/performance when you don't need that relation.

The catch: when you write a loop like this:
```java
List<Product> products = productRepo.findRelatedItems(productId); // Query #1: fetches 200 items

for (Product p : products) {
    String categoryName = p.getCategory().getName(); // 💥 Lazy load fires here!
}
```
The line `p.getCategory().getName()` **looks like completely ordinary code** — just reading an object property. There's no visible "database" or "query" anywhere. That's exactly why **code review misses it** — the SQL isn't visible in the code at all. Hibernate silently generates SQL behind the scenes, every single time `.getCategory()` is called on an object whose relation hasn't been loaded yet.

# Core Logic & How It Works

What actually happens under the hood:

```
Query #1: SELECT * FROM products WHERE related_to = ?  -> returns 200 rows
          (each product's category is still an empty "proxy object")

Loop iteration 1: p.getCategory().getName() 
  -> Hibernate sees category isn't loaded -> fires Query #2: SELECT * FROM categories WHERE id = 1

Loop iteration 2: p.getCategory().getName()
  -> fires Query #3: SELECT * FROM categories WHERE id = 2

... repeats for all 200 iterations

Total: 1 (main query) + 200 (per-item queries) = 201 queries
```

Even if each query only takes 40-50ms, multiplying by 200 (and most run **sequentially, not in parallel**) adds up to several seconds instantly — which lines up exactly with the "9 seconds" in the question.

# Trade-offs & When to Use

**Main fix: Targeted Eager Fetch (JOIN FETCH)**

```java
@Query("SELECT p FROM Product p JOIN FETCH p.category WHERE p.relatedTo = :productId")
List<Product> findRelatedItemsWithCategory(@Param("productId") Long productId);
```
This tells Hibernate to "fetch the category together, in one query, via a SQL JOIN" — 201 queries becomes **1 query**.

**If you need multiple relations at once: Entity Graph**
```java
@EntityGraph(attributePaths = {"category", "reviews", "images"})
List<Product> findRelatedItems(Long productId);
```

| Approach | When to use | Trade-off |
|---|---|---|
| **JOIN FETCH** | You know for certain this endpoint always needs that relation (e.g. a product page that always shows category) | Fetching too much at once risks duplicated rows (Cartesian product) if joining multiple collections simultaneously |
| **Batch Fetching** (`@BatchSize`) | You don't want to rewrite every query, prefer a global fix | Still multiple queries — but batched (e.g. `WHERE id IN (1,2,3...50)`), much better than N+1 but not as optimal as JOIN FETCH |
| **DTO Projection** | You don't need the full entity, just a few fields | Fastest option — pulls only the columns actually used, no need to hydrate the whole entity graph |

**When NOT to eager-fetch everything:** if some endpoint never touches a given relation (e.g. an API that only needs the product name), defaulting to eager-fetch everything makes that endpoint **slower for no reason**. That's why fetching should be scoped per use-case, not set as `EAGER` globally at the entity level.

# Real-World Scenario (Ecommerce Domain)

```java
// ❌ N+1 Problem
public List<ProductDto> getRelatedProducts(Long productId) {
    List<Product> products = productRepo.findByRelatedTo(productId); // Query #1
    return products.stream()
        .map(p -> new ProductDto(p.getName(), p.getCategory().getName())) // N queries!
        .collect(Collectors.toList());
}

// ✅ Fixed with JOIN FETCH — down to a single query
@Query("SELECT p FROM Product p JOIN FETCH p.category WHERE p.relatedTo = :id")
List<Product> findByRelatedToWithCategory(@Param("id") Long id);

public List<ProductDto> getRelatedProducts(Long productId) {
    List<Product> products = productRepo.findByRelatedToWithCategory(productId); // one query
    return products.stream()
        .map(p -> new ProductDto(p.getName(), p.getCategory().getName())) // reads from memory now, no query fired
        .collect(Collectors.toList());
}
```

**Bonus tip for Leads:** enable `spring.jpa.properties.hibernate.generate_statistics=true` in your dev environment to log query counts per request — or use an APM tool like **Datadog / New Relic** or **p6spy** to monitor query counts in real time in production. This bug "shows up at runtime, not in the diff," exactly as the question states — static code review alone will never catch it.

# Lead's Key Takeaway

> **"Query count must be a deliberate design decision, not an accidental side effect of a loop."**
>
> A good Lead doesn't rely on code review to catch this (the code looks perfectly normal). Instead, they build **query count monitoring** into performance testing/APM from day one, and hold to a personal rule: "every time I loop over a DB-fetched collection and access a relation inside it, I stop and ask — has that relation already been fetched?"
