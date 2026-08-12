# The Big Picture & Analogy

A **Transaction** is "a group of operations that must all succeed, or none of them do" (all-or-nothing) — there's no in-between state left half-done.

**Analogy:** Think of **transferring money between bank accounts**.

- Transferring from Account A to Account B requires 2 steps: (1) deduct from A, (2) add to B.
- If step 1 succeeds (A gets deducted) but step 2 fails (power outage, B's system crashes) — the money **vanishes from existence**, nobody has it! This is exactly the disaster a transaction prevents.
- A **Transaction** is like "a lawyer-certified contract" — if any step of the deal fails, everything done in that deal gets **entirely canceled (rollback)**, as if it never happened. There's no way for the deal to end up "half-done."

In Spring's `@Transactional`: if a method wrapped in a transaction throws an exception partway through, every database operation performed inside that method gets **completely rolled back**, as if it never ran at all.

---

# Why Do We Need It?

**The problem before Transactions (or using them incorrectly):**

```java
public void createOrder(OrderRequestDTO request) {
    Order order = new Order(request.getUserId(), calculateTotal(request));
    orderRepository.save(order); // ✅ order saved successfully

    for (CartItemDTO item : request.getItems()) {
        stockService.reduceStock(item.getProductId(), item.getQuantity()); 
        // 💥 if the 2nd one fails (e.g. insufficient stock) — the order that was already saved still exists!
        // the customer gets an order confirmation, but the stock was never actually reduced = inconsistent data
    }
}
```

- **Partial failure causes data inconsistency:** some operations succeed while others fail, leaving the database in a state that doesn't make sense — e.g. an Order exists but stock was never deducted, or a customer was charged but no order was created.
- **Manually rolling back yourself is very difficult:** if you have to undo each operation manually on error (writing your own compensating logic), the code becomes complex and highly bug-prone.
- **Concurrent requests colliding:** without proper transaction isolation, two simultaneous requests might read/write conflicting data, producing incorrect results (e.g. stock going negative because two people bought the last item at the same time).

**Why Transactions (`@Transactional`) solve this:**
- They guarantee that every operation within the same scope either all succeed together, or **all get canceled** — the database is never left in a "half-done" state.
- Spring manages the transaction boundary automatically via a single annotation, with no manual commit/rollback code needed.

---

# Core Logic & How It Works

## ACID — The Fundamental Principles of a Transaction

### **A — Atomicity**
Every operation in a transaction must either all succeed, or all fail — no in-between state.

### **C — Consistency**
A transaction must always move the database from "one valid state" to "another valid state" — never leaving data violating a constraint or business rule.

### **I — Isolation**
Multiple concurrently running transactions must never see each other's "in-progress" state — each transaction behaves as if it's running alone.

### **D — Durability**
Once a transaction has committed successfully, the data must **not disappear even if the system crashes** immediately afterward (it's genuinely written to disk).

## How `@Transactional` Works

```java
@Service
public class OrderService {

    @Transactional // Spring wraps this method in a proxy
    public Order createOrder(OrderRequestDTO request) {
        Order order = new Order(request.getUserId());
        orderRepository.save(order);

        for (CartItemDTO item : request.getItems()) {
            stockService.reduceStock(item.getProductId(), item.getQuantity());
        }

        return order;
    }
}
```

**What happens behind the scenes (the AOP Proxy Pattern):**

```
1. The client calls orderService.createOrder()
2. The Spring proxy intercepts the call before it reaches the real method
3. The proxy opens a database transaction (BEGIN)
4. The actual method runs — performing save(), reduceStock(), etc.
5a. If no exception occurs → the proxy issues COMMIT (permanently saves everything)
5b. If a RuntimeException occurs → the proxy issues ROLLBACK (undoes everything done in step 4)
```

**The single most important thing to watch out for — the Self-Invocation Problem:**

```java
@Service
public class OrderService {

    public void placeOrder(OrderRequestDTO request) {
        this.createOrder(request); // calling directly via "this" within the same class!
    }

    @Transactional
    public Order createOrder(OrderRequestDTO request) { ... }
}
```

**Why this is dangerous:** `@Transactional` works via a **proxy** that wraps the object from the outside. If you call a method via `this.` from within the same class, you're calling the method directly, **completely bypassing the proxy** — `@Transactional` on `createOrder()` will **not activate**, because the proxy never gets a chance to intercept this call (you'd need to call it through a separate bean from outside, or inject a self-reference via the Spring context).

## Rollback — Default Behavior That's Commonly Misunderstood

```java
@Transactional
public void createOrder(OrderRequestDTO request) {
    orderRepository.save(order);
    if (someCondition) {
        throw new IOException("Something failed"); // checked exception
    }
}
```

**Critical detail:** `@Transactional` **by default only rolls back on `RuntimeException` (unchecked)** — if a checked exception is thrown (e.g. `IOException`), the transaction will **commit normally, NOT roll back!** This is because Spring assumes checked exceptions represent "predictable, recoverable errors," not severe problems requiring a rollback.

```java
// To force rollback on checked exceptions too:
@Transactional(rollbackFor = Exception.class)
public void createOrder(OrderRequestDTO request) throws IOException {
    // now any exception thrown here triggers a rollback
}
```

## Propagation — Transactions Within Transactions

```java
@Transactional(propagation = Propagation.REQUIRED) // default
public void methodA() {
    methodB(); // if methodB also has @Transactional, it joins the same transaction as A
}
```

**Commonly used values:**
- `REQUIRED` (default) — uses an existing transaction if there is one, otherwise creates a new one.
- `REQUIRES_NEW` — always creates a brand new transaction (suspending the existing one first) — used when you want this operation to commit independently of the main transaction, even if the main transaction later rolls back (e.g. saving an audit log that should persist even if the order fails).

---

# Trade-offs & When to Use

**When to use `@Transactional`:**
- Any method in the **Service layer** that performs **more than one write operation** that must succeed together (e.g. save order + reduce stock).
- Methods with both reads and writes that need to see consistent data throughout (e.g. reading stock first, then deciding whether to reduce it).

**When NOT to use / things to watch for:**
- Don't unnecessarily add `@Transactional` to methods that only read (use `@Transactional(readOnly = true)` instead if needed, which helps with performance optimization).
- Don't open transactions that are **too long** (e.g. calling a slow external API inside one) — this holds database connections open for a long time, hurting throughput across the whole system.

**Trade-offs:**
- **You gain:** guaranteed data consistency, automatic rollback on failure, no need to write compensating logic yourself.
- **You pay:** long transactions lock resources for extended periods (hurting concurrency), and misunderstanding propagation/rollback rules can lead to hard-to-detect bugs (e.g. self-invocation silently disabling the transaction with no warning at all).

## Consistency in Monolith vs Microservices

**In a Monolith (1 database):** `@Transactional` works straightforwardly, since every operation lives in the same database — full ACID guarantees apply.

**In Microservices (each service has its own separate database):** Spring's `@Transactional` **cannot span across services**, since each database is genuinely separate — there's no single transaction covering everything. You need other patterns instead, such as:
- **Saga Pattern:** breaks the process into multiple steps, each with its own "compensating action." If a step fails, the compensating actions of previous steps are called to manually "undo" them (not a real database rollback).
- **Eventual Consistency:** accepts that data won't sync immediately, but will eventually sync via events/message queues.

---

# Real-World Scenario / Mini Example

A full order creation flow, saving the order and reducing stock within a single transaction:

```java
@Service
public class OrderService {
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);
    private final OrderRepository orderRepository;
    private final StockRepository stockRepository;

    public OrderService(OrderRepository orderRepository, StockRepository stockRepository) {
        this.orderRepository = orderRepository;
        this.stockRepository = stockRepository;
    }

    @Transactional(rollbackFor = Exception.class) // roll back on every exception, checked included
    public OrderResponseDTO createOrder(OrderRequestDTO request) {
        log.info("Starting order creation for userId={}", request.getUserId());

        // Step 1: validate stock first (from the HashMap you learned earlier — O(1) lookup)
        for (CartItemDTO item : request.getItems()) {
            int available = stockRepository.getStock(item.getProductId());
            if (available < item.getQuantity()) {
                // thrown here → no database write has happened yet, nothing to roll back
                throw new InsufficientStockException("Product " + item.getProductId());
            }
        }

        // Step 2: save the order (write #1)
        double total = calculateTotal(request);
        Order order = new Order(request.getUserId(), total);
        Order savedOrder = orderRepository.save(order);
        log.info("Order saved: id={}", savedOrder.getId());

        // Step 3: reduce stock one item at a time (writes #2, #3, #4...)
        for (CartItemDTO item : request.getItems()) {
            boolean success = stockRepository.reduceStock(item.getProductId(), item.getQuantity());
            if (!success) {
                // 💥 if this fails (e.g. a race condition emptied the stock in the meantime)
                // Spring will roll back the order saved in Step 2 as well!
                // the database returns to exactly as if nothing had happened
                throw new InsufficientStockException("Product " + item.getProductId() + " (race condition)");
            }
            log.info("Stock reduced: productId={}, quantity={}", item.getProductId(), item.getQuantity());
        }

        log.info("Order creation completed successfully: orderId={}", savedOrder.getId());
        return new OrderResponseDTO(savedOrder.getId(), savedOrder.getTotal());

        // if execution reaches this point with no exception → Spring commits the transaction automatically
        // if an exception occurs anywhere above → Spring rolls back the entire transaction automatically
    }

    private double calculateTotal(OrderRequestDTO request) {
        return request.getItems().stream()
            .mapToDouble(item -> stockRepository.getPrice(item.getProductId()) * item.getQuantity())
            .sum();
    }
}
```

**Testing the rollback behavior:**

```java
@SpringBootTest
public class OrderServiceTransactionTest {

    @Autowired private OrderService orderService;
    @Autowired private OrderRepository orderRepository;
    @Autowired private StockRepository stockRepository;

    @Test
    void shouldRollbackOrderWhenStockReductionFails() {
        // Given: product 102 has only 1 unit of stock
        long orderCountBefore = orderRepository.count();

        OrderRequestDTO request = new OrderRequestDTO();
        request.setItems(List.of(new CartItemDTO(102L, 5))); // requesting 5, but only 1 available

        // When: attempting to create an order (should fail)
        assertThrows(InsufficientStockException.class, () -> orderService.createOrder(request));

        // Then: no new order should have been created at all, since the transaction was fully rolled back
        long orderCountAfter = orderRepository.count();
        assertEquals(orderCountBefore, orderCountAfter); // confirms the rollback worked — no leftover order
    }
}
```

**Notice the power of `@Transactional`:** if Step 3 fails while reducing the second item's stock (assuming the first one already succeeded), Spring will roll back **both** the order saved in Step 2 **and** the stock already reduced in Step 3's first item, restoring everything to exactly as it was before — without writing a single line of compensating logic yourself.

---

# Lead's Key Takeaway

1. **`@Transactional` isn't just "a label" — it's a declaration of the "boundary of success" (unit of work).** Before adding this annotation, ask yourself: "if everything in here only partially succeeds, would the database be left in an acceptable state?" If the answer is "no," that's your signal a transaction boundary is needed.
2. **Understand that full ACID transaction guarantees only apply within a single database.** The moment you cross into multiple services/databases (microservices), no `@Transactional` can help anymore — you have to shift your thinking to patterns like Saga, or accept eventual consistency. This is a classic interview question that separates people who genuinely understand transactions from those who just add the annotation out of habit.
