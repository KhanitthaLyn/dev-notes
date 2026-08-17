# Question
An API suddenly started returning: 500 Internal Server Error Only for one customer.
All other customers worked fine. What would you investigate first?

# Answer

# The Big Picture & Analogy

Imagine a restaurant serving 1,000 customers a day with no issues, except **one specific customer gets sick every single time they order the exact same dish**, while everyone else eating that dish is fine.

The chef wouldn't conclude "the kitchen is randomly broken" (since everyone else is fine). They'd think: **"what's different about this specific customer?"** — an allergy? A special order modification unique to them? Some condition the kitchen never accounted for?

This is the key distinction from the earlier "800 million requests" question:
- **800 million requests** → errors scattered randomly, not tied to any specific user → smells like **timing/concurrency**
- **This question** → errors happen **consistently to the same one user**, while everyone else works fine → smells like a **data-specific bug**

# Why Do We Need It? (Why Separate These Two Patterns)

A junior facing this might chase generic bug-hunting steps — checking recent deploys, server resources, network — wasting time, because **if it were a genuine infra issue, it would affect everyone, not just one person.**

The most important signal here: **"consistent for this one user, but never for anyone else."** That's the classic signature of **"this user's own data is triggering an edge case in code that was never tested against it."**

# Core Logic & How It Works (Investigation Order)

**Step 1: Look at the actual error log/stack trace for this failing request first**
This is always the right starting point for a deterministic 500 error — the stack trace will point directly to the exception type and failing line (e.g. `NullPointerException`, `ArrayIndexOutOfBounds`). Unlike the "800 million" case, where logs might not help because it's a timing issue, here the logs are gold.

**Step 2: Compare "this user's data" against a working user's data**
Ask what's "unusual" or "older" about this specific user:
- **Null/missing field:** this user signed up long ago, before a schema field existed (e.g. `phoneNumber` is null), but newer code assumes it's always populated
- **Unusually large data volume:** this user has an abnormally large order history (e.g. 50,000 orders), causing a loop/aggregation to time out or overflow
- **Special characters (encoding issue):** the user's name contains emoji, non-ASCII characters, or SQL-like characters that break parsing/encoding
- **Unique plan/permission tier:** the user is on a legacy plan or has a custom feature flag that the normal code path doesn't handle
- **Unusual timezone/locale:** the user set a strange locale or timezone that breaks date parsing (e.g. an unusual negative offset)

**Step 3: Try to reproduce it with this specific user (not a generic user)**
Unlike the 800-million case, which was impossible to reproduce — this one **should reproduce reliably with the same user**, since it's a deterministic, data-tied bug. Pull that user's actual data (or clone it into staging) and re-run the request to confirm your hypothesis.

**Step 4: Check for feature flags or A/B tests targeting this specific user segment**
Sometimes the issue isn't the user's data at all — they just happen to be in an experiment group hitting a new, buggy code path.

# Trade-offs & When to Use (Context Changes the Hypothesis)

| Signal observed | What to suspect |
|---|---|
| Errors scattered randomly, not tied to any user, can't reproduce | Concurrency/timing (like the 800M case) |
| Error happens to **the same user, every time**, reliably reproducible | **Data-specific bug** (this case) |
| Error affects **a group of users** sharing a trait (e.g. everyone who signed up before 2023) | Incomplete schema migration / legacy data |
| Error hits every user right after a new deploy | Bad deploy / config error |

# Real-World Scenario (Ecommerce Domain)

Say `GET /users/{id}/order-summary` — which calculates a user's total order spend — breaks for exactly one user:

```java
// ❌ Likely cause: this user has old orders where the discount field is null,
// because they signed up before the "discount code" feature even existed
public BigDecimal calculateTotalSpent(User user) {
    return orderRepo.findByUserId(user.getId())
        .stream()
        .map(order -> order.getPrice().subtract(order.getDiscount())) 
        // 💥 NullPointerException if an old order has a null discount field
        .reduce(BigDecimal.ZERO, BigDecimal::add);
}
```

```java
// ✅ Fix: handle null explicitly instead of assuming data is always complete
public BigDecimal calculateTotalSpent(User user) {
    return orderRepo.findByUserId(user.getId())
        .stream()
        .map(order -> {
            BigDecimal discount = order.getDiscount() != null ? order.getDiscount() : BigDecimal.ZERO;
            return order.getPrice().subtract(discount);
        })
        .reduce(BigDecimal.ZERO, BigDecimal::add);
}
```

This is a classic pattern: an old user carrying data from before a schema change — one of the most common causes of "only this one user" bugs.

# Lead's Key Takeaway

> **"If a bug reliably reproduces for the same single user every time, look at that user's data first — not infrastructure, not timing."**
>
> A good Lead reads the *pattern* of the bug before diving into debugging — **"random occurrence" vs. "consistently targets the same entity"** are two signals that point to completely opposite investigation paths. Recognizing which one you're looking at up front saves enormous amounts of time.
