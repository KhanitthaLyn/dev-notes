# The Big Picture & Analogy

**Service Boundaries** is about deciding **"where the line" between each service should be drawn** — not just splitting by database table names, but splitting according to **"business responsibility boundaries" (business capability)**, based on concepts from **DDD (Domain-Driven Design)**.

**Analogy:** Think of **dividing departments within a company**.

- A good division: the HR department handles everything about employees, the Sales department handles everything about selling, the Finance department handles everything about money — each department has **"its own language"** and can decide matters within its own scope independently, without needing to constantly ask other departments.
- A bad division: splitting by "letters A-M handle this, N-Z handle that" — with no business rationale behind it, so every task constantly crosses departments, creating chaotic coordination.

Splitting services works the same way — **Bounded Context** (a key DDD term) is "a scope within which the same word always means the same thing." For example, the word "Product" in `ProductService` (concerned with name, description, images) versus the word "Product" in `InventoryService` (concerned only with stock quantity) **may not be the exact same object in every detail**, even though they share the same name.

---

# Why Do We Need It?

**The problem with drawing service boundaries incorrectly (extremely common in teams new to microservices):**

```
❌ Splitting directly by database table (Technical split, not Business split):
- UserTableService (just CRUD on the users table)
- OrderTableService (just CRUD on the orders table)
- ProductTableService (just CRUD on the products table)
```

- **Each service has no business logic of its own** — it becomes just a "thin wrapper around a database table." The real business logic ends up scattered across some caller or orchestrator, violating the most fundamental principle of microservices.
- **Every use case has to call multiple services at once**, e.g. "creating an order" requires calling `UserTableService` (check the user), `ProductTableService` (check the product), `OrderTableService` (save the order) — resulting in a huge number of network calls for a single operation. Slow and fragile (more calls = more points of failure).
- **Changing one small business rule impacts multiple services:** if the business changes a rule like "an order must validate the user's credit limit before creation," where should this logic live? If split by table, it becomes unclear who actually owns this business rule.

**Why Domain-driven boundaries solve this:**
- Splitting by **"who decides what"** (business capability) instead of "where the data lives" — each service has complete business logic of its own, not just a CRUD wrapper.
- Reduces cross-service network calls, since most related operations naturally fall within the same bounded context already.
- It's clear which service owns which business rule — reducing confusion about where to make changes.

---

# Core Logic & How It Works

## Principles for Finding a Bounded Context — Ask These Questions

### 1) **"Who owns this decision?"** (Business Capability Ownership)

```
Question: Who decides how much this product costs?
→ Answer: the team/department managing the catalog ⟹ this is ProductService

Question: Who decides whether this order should be approved?
→ Answer: the team/department managing orders ⟹ this is OrderService
```

### 2) **"Does the same word mean something different in each context?"** (Ubiquitous Language)

This is where people most commonly go wrong — the word **"Product"** doesn't mean the same thing everywhere:

```
ProductService sees "Product" as = { name, description, images, category, price }
InventoryService sees "Product" as = { sku, warehouseLocation, stockQuantity }
OrderService sees "Product" as = { productId, priceAtPurchase } (just a snapshot at purchase time)
```

**This isn't a mistake — this is correct design!** Each service only stores the data it **"needs to know"** to do its own job. There doesn't need to be one perfect, universal Product entity shared everywhere (that's Monolith-style thinking, which quietly couples services together).

### 3) **High Cohesion, Low Coupling — Within vs Across Boundaries**

```
✅ High cohesion within the same service:
OrderService knows about: create order, cancel order, calculate total, apply discount
→ everything related to "deciding about orders" lives together

✅ Low coupling across services:
OrderService doesn't need to know how ProductService internally stores product data,
it just knows which API to call to get the product's price back
```

## Example: Splitting Ecommerce into Services (User / Product / Order)

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   UserService    │     │  ProductService   │     │  OrderService   │
├─────────────────┤     ├──────────────────┤     ├─────────────────┤
│ - Register user  │     │ - Manage catalog   │     │ - Create order  │
│ - Authenticate   │     │ - Set price        │     │ - Calculate     │
│ - Manage profile │     │ - Manage category  │     │   total         │
│ - Manage address │     │ - Search products  │     │ - Track status  │
└─────────────────┘     └──────────────────┘     └─────────────────┘
        │                        │                         │
        │                        │                         │
   UserDB (own)            ProductDB (own)            OrderDB (own)
```

**The key dividing line:** `OrderService` **does NOT store** full product data (name, description, images) — it only stores `productId` + `priceAtPurchase` (a snapshot), which is all it needs. When full product details are required, it calls `ProductService` via API.

---

# Trade-offs & When to Use

**Signs the boundaries are drawn correctly:**
- Each service can describe "its own job" in a single clear sentence, without mentioning other services.
- Most business rules related to a given entity live **entirely within** one service, not scattered across services.
- Changing a service's internal implementation (e.g. switching databases) has zero impact on other services.

**Signs the boundaries are drawn incorrectly (need refactoring):**
- **Chatty communication:** a single use case requires calling multiple services too often (a sign of the "distributed monolith" discussed earlier).
- **Shared database:** two services accessing the same database — meaning they should probably actually be one service, or the boundaries need rethinking.
- **Circular dependency:** Service A calls B, and B calls back into A — usually a sign that responsibility was split in the wrong place (just like circular dependency issues covered earlier in DI).

**Trade-offs of Domain-driven splitting vs Technical layer splitting:**
- **Domain-driven split:** cohesive business logic within each service, fewer network calls — but requires investing time upfront to genuinely understand the domain before splitting (harder to start with).
- **Technical split (by table):** looks easy to start with, but turns into a distributed monolith over time — the most common pitfall for teams new to microservices.

---

# Real-World Scenario / Mini Example

**Splitting Ecommerce into 3 Services with a designed API contract between them:**

```java
// ===== UserService — owns everything about "identity" and "profile" =====
@RestController
@RequestMapping("/users")
public class UserController {
    // Business capability: managing user identity and personal information
    @PostMapping
    public UserResponseDTO register(@RequestBody RegisterRequestDTO request) { ... }

    @GetMapping("/{id}")
    public UserResponseDTO getUser(@PathVariable Long id) { ... }

    @PostMapping("/{id}/addresses")
    public AddressResponseDTO addAddress(@PathVariable Long id, @RequestBody AddressDTO address) { ... }
}
// UserDB stores: users, addresses, credentials
// Doesn't know about: product prices, what's in an order

// ===== ProductService — owns everything about the "catalog" =====
@RestController
@RequestMapping("/products")
public class ProductController {
    // Business capability: managing products sold in the system
    @GetMapping("/{id}")
    public ProductResponseDTO getProduct(@PathVariable Long id) { ... }

    @GetMapping("/search")
    public List<ProductResponseDTO> search(@RequestParam String keyword) { ... }

    @PutMapping("/{id}/price")
    public void updatePrice(@PathVariable Long id, @RequestBody UpdatePriceDTO price) { ... }
}
// ProductDB stores: products, categories, prices
// Doesn't know about: who ordered what, which user is viewing which order

// ===== OrderService — owns everything about "placing orders" =====
@RestController
@RequestMapping("/orders")
public class OrderController {
    private final ProductServiceClient productClient; // calls ProductService via API
    private final UserServiceClient userClient;         // calls UserService via API

    // Business capability: all decisions related to orders
    @PostMapping
    public OrderResponseDTO createOrder(@RequestBody OrderRequestDTO request) {
        // call UserService to validate the user exists and is active
        UserResponseDTO user = userClient.getUser(request.getUserId());

        // call ProductService to fetch current prices (never duplicate prices inside OrderService)
        List<ProductResponseDTO> products = productClient.getProducts(request.getProductIds());

        // OrderService's own business logic — calculate, validate, save
        Order order = buildOrder(user, products, request);
        return orderRepository.save(order);
    }
}
// OrderDB stores: orders, order_items (just productId + priceAtPurchase snapshot)
// Does NOT store: full product details, full user profile data
```

**Key design details worth noting:**

1. **`OrderService.OrderItem`** only stores `productId` + `priceAtPurchase` — it **does NOT store** `productName`, `productDescription`, because that's `ProductService`'s responsibility (if a product name needs to show up in order history, call `ProductService` at query time, or maintain a read-only cache).

2. **Each service has complete business logic of its own**, not just CRUD — `OrderService` has its own calculation and validation logic for orders, not just plain database saves.

3. **Cross-service calls only happen when genuinely necessary** (creating an order needs to know about the user + products), not for every operation.

**Test whether the boundary is correct with this question:** "If ProductService switches its entire database from PostgreSQL to MongoDB, does `OrderService` need any code changes?" — the answer should be **"no changes needed at all"**, because `OrderService` only talks to `ProductService` through its API contract, with zero knowledge of internal implementation. If the answer is "yes, changes are needed," that means the boundary is leaking somewhere.

---

# Lead's Key Takeaway

1. **Split services by "who owns the decision," not by "where the data lives."** The question to keep asking: "who decides this business rule?" If you can clearly answer that it belongs to exactly one service, the boundary is drawn correctly. If the answer is unclear, or the rule has to be split between 2 services, that's a design problem.
2. **Duplicated data across services isn't a problem — it's the correct outcome of separating bounded contexts.** `OrderService` storing `productId` + `priceAtPurchase`, duplicating information also found in `ProductService`, is **not a normalization violation** like in the Database Modeling topic, because it's data from two different contexts (ProductService cares about "current price," OrderService cares about "price at the moment of purchase"). Understanding this distinction is exactly what separates people who genuinely understand DDD from those still thinking with a single-database mindset.
