# The Big Picture & Analogy

**Distributed Error Handling** is a set of techniques for dealing with the reality that **"in a system where multiple services talk over a network, failure isn't a matter of 'if' but 'when.'"** Networks drop, services slow down abnormally, services crash mid-request — these things happen constantly, and systems must be designed to **"fail gracefully"** instead of collapsing entirely.

**Analogy:** Think of **calling a restaurant to book a table**.

- **Retry** = if the line is busy, you don't give up immediately — you **try calling again after a bit** — but not by dialing frantically nonstop (that would just keep the line busier); instead you wait a moment before trying again (backoff).
- **Timeout** = if nobody picks up **after 30 seconds**, you shouldn't hold the line forever — you hang up and decide what to do next. You set a "limit of patience" in advance.
- **Fallback** = if you really can't get through to your favorite restaurant (tried multiple times, still no luck), you have a **"backup plan"** — call a second restaurant instead, or order delivery instead of dining out — never leaving yourself with nothing to eat just because your first choice didn't answer.

These 3 concepts work together to make a system **"resilient to unavoidable failure"** in a world where everything is connected over networks.

---

# Why Do We Need It?

**The problem with a system that lacks good error handling for distributed calls:**

```java
public OrderResponseDTO createOrder(OrderRequestDTO request) {
    // calling PaymentService directly, with zero protection
    PaymentResult payment = restTemplate.postForObject(
        "http://payment-service/payments", request, PaymentResult.class);
    // if PaymentService becomes abnormally slow (e.g. its internal database is lagging)
    // → this thread will "hang" waiting indefinitely, with no timeout
    // → OrderService's connection pool gradually gets exhausted
    // → OrderService itself starts responding slowly too, even though the real problem is PaymentService!
    
    return processOrder(payment);
}
```

- **No Timeout → Cascading Failure:** if the downstream service slows down with no bound on wait time, the caller's thread hangs indefinitely. Once this happens across many concurrent requests, the caller's connection pool fills up completely — **a service with no problems of its own starts failing too**, purely because it's waiting on a broken dependency. This is **cascading failure**, one of the most dangerous patterns in distributed systems.
- **No Retry → failing too easily on temporary issues:** a network hiccup that would resolve itself within 1 second instead causes the entire request to fail immediately, even though a retry would have succeeded.
- **No Fallback → unnecessarily poor user experience:** if `RecommendationService` (far less critical than payment) goes down, the entire product page errors out too, when it really should just "skip showing recommendations" while everything else works normally.

**Why Retry + Timeout + Fallback solve this:**
- **Timeout** prevents cascading failure by capping how long the caller waits, then "releasing" resources once that limit is exceeded.
- **Retry** handles transient failures that often resolve themselves if tried again.
- **Fallback** lets the system **degrade gracefully** instead of failing entirely when a less-critical dependency goes down.

---

# Core Logic & How It Works

## Timeout — Setting a Limit of Patience

```java
@Bean
public RestTemplate restTemplate() {
    return new RestTemplateBuilder()
        .setConnectTimeout(Duration.ofSeconds(2))  // max time to establish a connection
        .setReadTimeout(Duration.ofSeconds(5))     // max time to wait for a response
        .build();
}
```

**How to choose a timeout value:** you need to balance "waiting long enough for a normal operation to succeed" against "not waiting so long it hurts user experience or ties up resources." Start by measuring the downstream service's typical **p99 latency** (the time within which 99% of requests succeed), then set the timeout slightly above that — not a random guess.

## Retry — Trying Again Intelligently (Not Frantically)

**Naive Retry (dangerous!):**
```java
// ❌ wrong — retries immediately, no delay
for (int i = 0; i < 5; i++) {
    try {
        return callPaymentService(request);
    } catch (Exception e) {
        // retry right away
    }
}
```
**Why this is dangerous:** if the downstream service is already overloaded, hammering it with immediate retries from every client at once makes things **worse** (called a **retry storm**) — like frantically redialing a call center that's already jammed with calls, making it even harder for the system to recover.

**Exponential Backoff (the correct approach):**
```java
public PaymentResult callWithRetry(PaymentRequest request) {
    int maxRetries = 3;
    long baseDelayMs = 200;

    for (int attempt = 0; attempt <= maxRetries; attempt++) {
        try {
            return paymentClient.processPayment(request);
        } catch (TransientException e) { // only retry errors that are worth retrying!
            if (attempt == maxRetries) {
                throw e; // exhausted all attempts → genuinely give up
            }
            long delay = baseDelayMs * (long) Math.pow(2, attempt); // 200ms, 400ms, 800ms...
            long jitter = ThreadLocalRandom.current().nextLong(delay / 2); // add randomness
            sleep(delay + jitter);
        }
    }
    throw new IllegalStateException("Unreachable");
}
```

**3 key principles of correct Retry design:**
1. **Exponential Backoff:** increase the wait time by a multiplier with each retry attempt (200ms → 400ms → 800ms), instead of waiting a fixed amount every time — giving the downstream service time to recover.
2. **Jitter:** add a small amount of randomness to the wait time — preventing multiple clients from retrying **at exactly the same moment**, which would just create another traffic spike.
3. **Only retry errors worth retrying:** you must distinguish between errors likely to succeed on retry (network timeouts, 503 Service Unavailable) and errors that will never succeed no matter how many times you retry (400 Bad Request, 401 Unauthorized — retrying these just wastes time getting the same error again).

## Fallback — A Backup Plan When Everything Fails

```java
public List<ProductRecommendation> getRecommendations(Long userId) {
    try {
        return recommendationClient.getPersonalized(userId); // calling a service that might fail
    } catch (Exception e) {
        log.warn("Recommendation service unavailable, using fallback", e);
        return getPopularProductsFallback(); // backup plan — generic popular products instead of personalized ones
    }
}
```

**How to choose a good fallback:** a fallback should be something **"slightly worse, but still functional"** — not a blank error message. For example, showing "popular products" instead of "products recommended for you" is still far better than showing nothing at all.

## Circuit Breaker — Preventing Pointless Retries

**The problem Circuit Breakers solve:** if the downstream service has genuinely gone down (not just temporarily slow), retrying repeatedly is completely pointless, and just makes the caller waste time waiting for timeouts over and over.

```
State: CLOSED (normal) → calls the service normally
       │
       │ (failure rate exceeds a threshold, e.g. 50% of the last 10 requests)
       ▼
State: OPEN (circuit broken) → doesn't call the service at all, returns fallback immediately
       │
       │ (waits out a cooldown period, e.g. 30 seconds)
       ▼
State: HALF_OPEN (testing) → tries a few calls to see if it's recovered
       │
       ├─ succeeds → returns to CLOSED
       └─ fails → returns to OPEN
```

**Comparison:** just like a home electrical circuit breaker — if there's a repeated short circuit, the breaker "cuts the power" entirely instead of letting it short-circuit repeatedly until a fire starts, and waits until it's safe before allowing power back on.

---

# Trade-offs & When to Use

**When to use Retry:**
- Only for **transient** errors — network blips, 503s, connection timeouts.
- Operations that are **idempotent** (calling repeatedly produces the same result, with no duplicated side effects) — e.g. a `GET` request is always safe to retry, but retrying `POST /payments` carelessly could charge the customer twice!

**When NOT to retry:**
- **Client errors** (4xx) — validation failures, unauthorized — retrying produces the exact same result every time.
- Operations that are **not idempotent**, without an idempotency key protecting them — risking serious duplicated side effects.

**When to use Fallback:**
- Dependencies that are **"non-critical to the core business flow"** — recommendations, analytics, non-essential enrichment data.
- **Never use fallback** for genuinely critical operations (like payment) — the fallback for "payment charge failed" must never be "treat it as paid anyway." It should fail explicitly instead.

**Trade-offs:**
- **Retry:** gains resilience against transient issues, at the cost of increased latency (if a retry actually happens) and risk of a retry storm if backoff isn't designed properly.
- **Timeout too short:** fails unnecessarily often, even for operations that would have succeeded with a bit more patience. **Timeout too long:** risks cascading failure, as described above.

---

# Real-World Scenario — Full Pseudo-Code for Retry Logic

```
FUNCTION callServiceWithResilience(request):
    
    // ===== Configuration =====
    maxRetries = 3
    baseDelayMs = 200
    timeoutMs = 3000
    circuitBreakerThreshold = 0.5  // 50% fail rate
    
    // ===== Step 1: Check Circuit Breaker State first =====
    IF circuitBreaker.state == OPEN:
        LOG "Circuit breaker OPEN — skipping call, using fallback"
        RETURN getFallbackResponse()
    
    // ===== Step 2: Attempt the service call with retry logic =====
    FOR attempt FROM 0 TO maxRetries:
        TRY:
            // call with a timeout
            response = callWithTimeout(request, timeoutMs)
            
            // success → inform the circuit breaker this call succeeded
            circuitBreaker.recordSuccess()
            RETURN response
            
        CATCH TimeoutException OR ConnectionException AS e:
            // this is a transient error — worth retrying
            circuitBreaker.recordFailure()
            
            IF attempt == maxRetries:
                LOG "Max retries exceeded, giving up"
                BREAK  // exit the loop and fall through to fallback
            
            // calculate delay using exponential backoff + jitter
            delay = baseDelayMs * (2 ^ attempt)
            jitter = RANDOM(0, delay / 2)
            LOG "Attempt {attempt} failed, retrying after {delay + jitter}ms"
            SLEEP(delay + jitter)
            CONTINUE  // try the next attempt
            
        CATCH ClientException AS e:  // e.g. 400, 401, 404
            // this error will never succeed no matter how many times we retry — don't retry at all
            LOG "Client error, not retrying: {e.message}"
            THROW e  // rethrow immediately, skip the retry loop entirely
    
    // ===== Step 3: All retries exhausted, still failed → use Fallback =====
    LOG "All retries exhausted, using fallback"
    RETURN getFallbackResponse()


FUNCTION getFallbackResponse():
    // example: return old cached data, or a safe default response
    IF cache.has(request.key):
        RETURN cache.get(request.key)  // stale data is better than nothing
    ELSE:
        RETURN DEFAULT_SAFE_RESPONSE  // e.g. empty list, default value
```

**A real implementation example using Resilience4j (the standard library for Java):**

```java
@Service
public class RecommendationService {
    private final RecommendationClient client;

    @CircuitBreaker(name = "recommendationService", fallbackMethod = "fallbackRecommendations")
    @Retry(name = "recommendationService")
    @TimeLimiter(name = "recommendationService")
    public CompletableFuture<List<Product>> getRecommendations(Long userId) {
        return CompletableFuture.supplyAsync(() -> client.getPersonalized(userId));
    }

    // Fallback method — signature must match the main method + accept an extra Exception param
    private CompletableFuture<List<Product>> fallbackRecommendations(Long userId, Exception e) {
        log.warn("Falling back to popular products for userId={}", userId, e);
        return CompletableFuture.completedFuture(popularProductsCache.getPopular());
    }
}
```

```yaml
# application.yml — configuring resilience patterns
resilience4j:
  retry:
    instances:
      recommendationService:
        max-attempts: 3
        wait-duration: 200ms
        exponential-backoff-multiplier: 2
        retry-exceptions:
          - java.net.ConnectException
          - java.util.concurrent.TimeoutException
        ignore-exceptions:
          - com.ecommerce.exception.ClientException  # never retry client errors

  circuitbreaker:
    instances:
      recommendationService:
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
        sliding-window-size: 10

  timelimiter:
    instances:
      recommendationService:
        timeout-duration: 3s
```

**Notice how all 3 annotations work together:** `@TimeLimiter` caps the wait time → `@Retry` handles retries on transient failures → `@CircuitBreaker` cuts the connection if failures happen too often → and if everything fails, `fallbackRecommendations` gets called instead.

---

# Lead's Key Takeaway

1. **Retry, Timeout, and Fallback aren't 3 separate concepts — they're "3 layers of protection" that must always work together.** Timeout prevents infinite waiting, Retry handles temporary issues, and Fallback catches things when everything genuinely fails. Missing any one of these creates a dangerous blind spot — e.g. having retry without timeout means a thread can hang indefinitely starting from the very first retry attempt.
2. **The question to ask before adding any resilience pattern is: "Is this operation idempotent, and is this error transient or permanent?"** These two questions decide everything: idempotent + transient = safe to retry; not idempotent or permanent error = never retry blindly, since it could cause more harm than good (e.g. duplicate charges, or wasting time retrying an error that will never resolve).
