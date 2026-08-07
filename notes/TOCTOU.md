You generated a high-entropy UUIDv4. You verified it does not exist anywhere in the database.
You inserted the record. The database throws a duplicate key error. Network is secure. No other traffic is hitting the DB.

How did this happen?


It sounds "impossible" at first, but it points to a fundamental bug every Lead needs to internalize: **the Check-then-Act (TOCTOU) race condition.**

1. The Big Picture & Analogy

Imagine walking up to a bathroom stall. The door isn't locked, so you see it's empty and walk toward it. But at the exact same moment, someone else also saw the unlocked door and walked in too.

Nobody "checked wrong" — at the moment you checked, it really was empty. The problem is that **there's a time gap between checking and acting**, 
and someone can slip into that gap. That's exactly what happened to the UUID in this scenario.

2. Why Do We Need It? (Why This Happens)

Most people (including whoever wrote this puzzle) follow this pattern:

```
1. Generate UUID
2. SELECT ... WHERE id = uuid   -- check if it exists
3. If not found -> INSERT
```

The problem: **step 2 and step 3 are not one atomic operation.** They're two separate calls, with nothing locking the gap between them. Even if that gap is a few milliseconds, it's still exploitable.

The question claims "no other traffic hitting the DB" — that's the trap. **"No other users" does not mean "no duplicate requests from your own client."**

3. Core Logic & How It Works (The Real Causes)

**Cause A: Client-side retry (most common)**
- First request INSERTs successfully into the DB
- But the response gets lost (network timeout / connection drop)
- The client (or a retry library — Axios interceptor, gRPC retry policy) thinks it failed and **retries the same request**
- The retry reuses the **same UUID**, generated once before the retry loop started
- → Second INSERT hits the same Primary Key = Duplicate Key Error

**Cause B: Read Replica Lag**
- If the SELECT check reads from a **Read Replica** but the INSERT writes to **Primary**
- The check can say "not found" simply because the replica hasn't caught up yet (replication lag)
- But the record already exists on Primary from an earlier attempt

**Cause C: Message Queue Redelivery**
- If this logic runs inside a queue consumer (Kafka, SQS)
- The message got processed, but the consumer failed to ack before a timeout
- The queue redelivers it (at-least-once delivery) → duplicate insert attempt

4. Trade-offs & When to Use

> This isn't really "when to use" — it's "how to prevent it." Here's the fix comparison:

| Approach | Explanation |
|---|---|
| ❌ **Don't trust Check-then-Insert** | SELECT-before-INSERT does NOT prevent duplicates — it's inherently non-atomic |
| ✅ **Let the DB constraint be the single source of truth** | Rely on `UNIQUE`/Primary Key constraints, and catch the exception on insert instead of pre-checking |
| ✅ **Idempotency Keys** | If clients might retry, have them send an idempotency key; the server checks if that key was already processed and returns the cached result instead of inserting again |
| ✅ **`INSERT ... ON CONFLICT DO NOTHING`** (Postgres) or `INSERT IGNORE` (MySQL) | Let the database handle the conflict atomically in a single statement — no separate check/insert steps |

5. Real-World Scenario (Ecommerce Domain)

Say your system generates an **Order ID** at checkout:

```
❌ Problematic approach:
1. orderId = generateUUID()
2. if (!orderRepo.existsById(orderId)) {
3.     orderRepo.save(new Order(orderId, ...))
4. }
```

If the frontend spinner hangs and the user double-clicks "Checkout," or mobile network hiccups trigger an automatic retry 
the same order tries to insert twice with the same `orderId` (because the frontend stored it in state and never regenerated it for the retry).

```
✅ Correct approach:
1. orderId = generateUUID()  // generated once client-side, sent as idempotency key
2. try {
3.     orderRepo.save(new Order(orderId, ...))
4. } catch (DuplicateKeyException e) {
5.     return existingOrder(orderId)  // no error — just return the existing order
6. }
```

6. Lead's Key Takeaway

> **"Check-then-Act" is never 100% safe unless it's atomic — regardless of how much or how little traffic there is.**
>
> A good Lead doesn't ask "what's the probability of a race condition?" They ask "if it happens, does the system break?"
>  and design so that **DB constraints + exception handling** are the trusted last line of defense, not application-level logic that assumes a prior check guarantees safety.
