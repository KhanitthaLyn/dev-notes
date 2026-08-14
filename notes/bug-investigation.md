# Question 
Your system has processed 800 million requests without a problem.
Then one request corrupts a user's data. You can't reproduce it.
What do you investigate first?

====

# Answer

# The Big Picture & Analogy

Imagine 800 million flights landing safely, then flight #800,000,001 has an incident. An air-crash investigator won't say "the plane is broken" — it flew successfully 800 million times. They'll ask: **"What was different about this specific flight?"** — weather, cargo weight, timing.

That's the core insight here: when a bug happens **1 in 800 million** times and can't be reproduced, it's almost never a plain deterministic logic bug — it's a bug tied to **timing** or a **rare system state**. In other words: the smell of a **race condition / concurrency bug**.

# Why Do We Need It? (Why This Mindset Matters)

A junior facing this usually makes two mistakes:
- Tries to reproduce with the exact same input repeatedly (wasted effort — since it's not deterministic, the input isn't the real variable)
- Blames "bad luck" or an "unpreventable edge case" and closes the ticket

The key insight: **if you can't reproduce it with the same input, the real variable isn't input data — it's timing or system state at that moment**, something juniors often overlook because it's invisible in the request payload itself.

# Core Logic & How It Works (Investigation Order)

**Step 1: First hypothesis — always concurrency**
At a ratio of 1:800,000,000, this is a timing edge case, not a data edge case. Check first:
- Is there **shared mutable state** accessed by multiple threads/requests concurrently (static variables, in-memory caches, non-thread-safe singletons)?
- Is there a non-atomic **check-then-act** pattern (like the UUID race condition we covered)?
- Does the connection/thread pool **reuse objects without resetting state** (e.g. `SimpleDateFormat` shared as `static` in Java, which isn't thread-safe)?

**Step 2: Look at detailed logs around that exact window — not just the error log**
- Compare the failing request's timestamp against **other concurrent requests** happening at the same instant — don't just examine the failing request in isolation
- Check if a deploy, config change, or traffic spike happened right around that time (sometimes it's a pool scaling up/down at just the wrong moment)

**Step 3: Inspect non-deterministic external dependencies**
- Retry logic resending duplicate requests (same pattern as the UUID scenario earlier)
- Distributed writes with partial failure (write succeeds on one replica, fails on another)
- Cache invalidation racing against a data update (stale read mid-write)

**Step 4: Capture state snapshots proactively, instead of chasing reproduction**
Since you can't reproduce it, shift focus from "make it happen again" to **"capture everything the next time it happens."** Add detailed structured logging / distributed tracing at the suspected hotspot, then wait for the next occurrence — fully instrumented this time.

# Trade-offs & When to Use (Context Changes the Answer)

| Situation | First suspect |
|---|---|
| High-concurrency / multithreaded system | Race condition, shared mutable state |
| Distributed system (multiple services/DBs) | Partial failure, eventual consistency, network partition |
| A deploy happened shortly before the issue | Bad deploy with a delayed effect (e.g. slow memory leak) |
| Bug only appears during peak traffic | Resource exhaustion (connection pool exhausted, thread starvation) |

# Real-World Scenario (Ecommerce Domain)

Say the Order Service has this bug: "1 in 800 million orders has a wrong `totalPrice`."

```java
// ❌ Prime suspect: shared mutable state that isn't thread-safe
public class PriceCalculator {
    private static final SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
    // SimpleDateFormat is NOT thread-safe! If multiple requests call this
    // concurrently, its internal Calendar state gets corrupted randomly
    
    public BigDecimal calculateTotal(Order order) {
        String dateStr = sdf.format(order.getDate()); // race happens right here
        ...
    }
}
```

This would be impossible to reproduce by firing the same request repeatedly, because it requires **two requests colliding at the exact same millisecond** to surface — which matches the "1 in 800 million" signature perfectly.

# Lead's Key Takeaway

> **"Can't reproduce + extremely low occurrence rate = suspect concurrency first, not logic."**
>
> A good Lead doesn't debug by re-running the same input over and over. They ask the right question first: **"What else was running concurrently at that exact moment?"** — because this class of bug doesn't hide in the input. It hides in the **timing** where two parts of the system collide without proper locking or synchronization.
