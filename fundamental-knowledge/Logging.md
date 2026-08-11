# The Big Picture & Analogy

**Logging** is recording "events that happen in the system" as evidence, so you can debug issues later without guessing. **Structured Logging** means recording in a machine-readable format (like JSON) instead of free-form text.

**Analogy:** Think of **CCTV cameras in a shopping mall**.

- **Unstructured logging** is like a camera that just records plain video — events are captured, sure, but when something goes wrong (an item goes missing), you have to sit and scrub through the entire day's footage manually looking for what happened. Painfully slow.
- **Structured Logging** is like a camera system with **automatic tagging** — every recording includes timestamp, camera number, and zone, so you can immediately filter: "show me Zone B footage between 14:00-14:05."
- **Correlation ID** is like a **case number** — no matter how many cameras capture the same event (a customer walking from Zone A to B to C), every clip gets tagged with the same case number, letting you stitch them together into the full story easily.

In backend systems: 1 HTTP request might travel through multiple services (Controller → Service → Repository → external API). The **Correlation ID** is the "case number" attached to that request the whole way through — letting you grep for every log related to that request with a single command when debugging.

---

# Why Do We Need It?

**The problem before good logging:**

```java
System.out.println("Order created"); // no context at all — which order, who created it, when
```

- **No context** — you just see "Order created" but don't know which order, which user, or when — nearly useless when debugging a production issue.
- **No log levels** — everything gets printed the same way, with no way to distinguish severe errors from ordinary debug info, making log files cluttered and hard to find signal in the noise.
- **Can't trace a request across multiple services** — in systems with multiple microservices, or even multiple layers within a single monolith, figuring out "what did this request touch, and where" is extremely hard without a linking mechanism (correlation).
- Many production incidents take a long time to fix because there's **not enough logging**, or **too much logging without structure** to find the pattern in.

**Why Structured Logging + Correlation ID solve this:**
- Structured logs (JSON) can be queried directly through log aggregation tools (e.g. ELK Stack, Datadog) — like "find all logs where userId=123 and level=ERROR."
- Correlation ID lets you trace a single request across multiple services/layers with one query — turning "finding a needle in a haystack" into "grep once, done."
- Correct log levels let you gauge severity instantly, without reading every single line.

---

# Core Logic & How It Works

## Log Levels — Severity Ranking

```
TRACE < DEBUG < INFO < WARN < ERROR
```

| Level | When to use | Example |
|---|---|---|
| **TRACE** | Finest-grained detail, for very deep debugging | Every variable value at each step in a loop |
| **DEBUG** | Info to help during development debugging | "Calculated total: 150.00" |
| **INFO** | Significant events worth tracking normally | "Order created: id=123" |
| **WARN** | Abnormal situation, but the system can still continue | "Stock low for product 456" |
| **ERROR** | A real problem impacting functionality | "Payment failed for order 123" |

**Rule of thumb:** production usually only logs `INFO` and above (DEBUG/TRACE are disabled since they're noisy and consume storage). During development, DEBUG might be enabled for detailed tracing.

```java
private static final Logger log = LoggerFactory.getLogger(OrderService.class);

log.info("Order created: id={}, userId={}", order.getId(), order.getUserId());
log.warn("Stock running low for product: id={}, remaining={}", productId, stock);
log.error("Payment failed for order: id={}", orderId, exception); // pass the exception to capture the stack trace
```

**Watch out:** avoid string concatenation (`"Order: " + order.getId()`), since it wastes performance even when that log level is disabled (the string still gets built regardless). Use `{}` placeholders instead — SLF4J lazily evaluates them only if the log level is actually enabled.

## Structured Logging (JSON Format)

**Old style (plain text):**
```
2026-08-11 10:30:00 INFO OrderService - Order created: id=123, userId=456
```

**Structured (JSON) — much easier for tools to parse:**
```json
{
  "timestamp": "2026-08-11T10:30:00Z",
  "level": "INFO",
  "logger": "OrderService",
  "message": "Order created",
  "orderId": 123,
  "userId": 456,
  "correlationId": "abc-123-def-456"
}
```

In Spring Boot, **Logback + Logstash encoder** is commonly used to automatically output logs in this JSON format, making it easy to query through tools like Kibana/Datadog (filter directly by field, no text regex needed).

## Correlation ID — Linking Logs Across Layers/Services

**The idea:** generate a unique ID the moment a request first arrives, then "attach" that ID to every log produced throughout the entire duration of that request.

```java
// uses SLF4J's MDC (Mapped Diagnostic Context) — stores context per thread
@Component
public class CorrelationIdFilter extends OncePerRequestFilter {

    private static final String CORRELATION_ID_HEADER = "X-Correlation-Id";

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                     FilterChain chain) throws ServletException, IOException {
        String correlationId = request.getHeader(CORRELATION_ID_HEADER);
        if (correlationId == null) {
            correlationId = UUID.randomUUID().toString(); // generate new if absent (e.g. request from external client)
        }

        MDC.put("correlationId", correlationId); // add to MDC — every log in this thread will automatically include it
        response.setHeader(CORRELATION_ID_HEADER, correlationId); // return it to the client so they can track it too

        try {
            chain.doFilter(request, response); // let the request continue on to the Controller
        } finally {
            MDC.clear(); // critical! must always clear, or thread-pool reuse will leak the old correlationId across requests
        }
    }
}
```

**Once MDC is set,** every `log.info()`, `log.error()` call happening on the same thread (throughout that request, regardless of which layer it's in) automatically includes `correlationId` — with no need to manually pass it as a parameter.

---

# Trade-offs & When to Use

**When to log at INFO level:**
- Significant business events that need an audit trail — e.g. order created, payment processed, user registered.

**When to log at ERROR level:**
- Every time an exception is caught in `GlobalExceptionHandler` (especially unexpected exceptions) — always log with the full stack trace for later debugging.

**When NOT to log:**
- **Never log sensitive data** — passwords, credit card numbers, personal identification numbers must never appear in logs, even at DEBUG level (a serious security/compliance risk).
- Don't log everything at INFO until it floods the logs — if every line is INFO, it's effectively the same as having no levels at all. Choose only the genuinely significant events.

**Trade-offs:**
- **Structured logging:** massive query power via log aggregation tools, at the cost of larger log files (JSON is more verbose than plain text) and needing extra encoder/infrastructure setup.
- **Correlation ID:** invaluable traceability across services/layers when debugging production incidents, at the cost of needing to be careful about MDC leaking across threads if not cleared properly (especially with async code or thread pools).

---

# Real-World Scenario / Mini Example

Adding logging with correlationId to an ecommerce Controller and Service:

```java
// ===== Correlation ID Filter (setup once, used system-wide) =====
@Component
public class CorrelationIdFilter extends OncePerRequestFilter {
    private static final String CORRELATION_ID_HEADER = "X-Correlation-Id";

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                     FilterChain chain) throws ServletException, IOException {
        String correlationId = request.getHeader(CORRELATION_ID_HEADER);
        if (correlationId == null) correlationId = UUID.randomUUID().toString();

        MDC.put("correlationId", correlationId);
        response.setHeader(CORRELATION_ID_HEADER, correlationId);

        try {
            chain.doFilter(request, response);
        } finally {
            MDC.clear();
        }
    }
}

// ===== Controller — log the entry/exit points of a request =====
@RestController
@RequestMapping("/orders")
public class OrderController {
    private static final Logger log = LoggerFactory.getLogger(OrderController.class);
    private final OrderService orderService;

    public OrderController(OrderService orderService) { this.orderService = orderService; }

    @PostMapping
    public ResponseEntity<OrderResponseDTO> createOrder(@Valid @RequestBody OrderRequestDTO request) {
        log.info("Received order creation request: userId={}, itemCount={}", 
                  request.getUserId(), request.getItems().size());
        // no need to add correlationId to the message manually — MDC handles it automatically via the log pattern

        OrderResponseDTO response = orderService.createOrder(request);

        log.info("Order created successfully: orderId={}", response.getOrderId());
        return ResponseEntity.status(201).body(response);
    }
}

// ===== Service — log significant business logic =====
@Service
public class OrderService {
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);
    private final StockRepository stockRepository;

    public OrderService(StockRepository stockRepository) { this.stockRepository = stockRepository; }

    public OrderResponseDTO createOrder(OrderRequestDTO request) {
        log.debug("Validating stock for {} items", request.getItems().size());

        for (CartItemDTO item : request.getItems()) {
            int available = stockRepository.getStock(item.getProductId());
            if (available < item.getQuantity()) {
                log.warn("Insufficient stock: productId={}, requested={}, available={}",
                          item.getProductId(), item.getQuantity(), available);
                throw new InsufficientStockException("Product " + item.getProductId());
            }
        }

        try {
            Order order = processOrder(request);
            log.info("Order processed: orderId={}, total={}", order.getId(), order.getTotal());
            return toResponseDTO(order);

        } catch (PaymentException e) {
            // logging with the exception object — captures the full stack trace along with it
            log.error("Payment processing failed for userId={}", request.getUserId(), e);
            throw e;
        }
    }
}
```

**Logback configuration (`logback-spring.xml`) to attach correlationId to every log message automatically:**

```xml
<configuration>
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <!-- %X{correlationId} pulls the value from MDC into every log line -->
            <pattern>%d{yyyy-MM-dd HH:mm:ss} [%X{correlationId}] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>
    <root level="INFO">
        <appender-ref ref="CONSOLE" />
    </root>
</configuration>
```

**What you'd see in the log output (notice the same correlationId appears across every layer):**
```
2026-08-11 10:30:00 [abc-123-def] INFO  OrderController - Received order creation request: userId=456, itemCount=2
2026-08-11 10:30:00 [abc-123-def] DEBUG OrderService    - Validating stock for 2 items
2026-08-11 10:30:00 [abc-123-def] WARN  OrderService    - Insufficient stock: productId=789, requested=5, available=2
```

**The power of correlationId during real debugging:** if there's a production issue and a user reports their order failed, you just need the correlationId from the response header (which the client stored) and run one command: `grep "abc-123-def" app.log`. You immediately see **everything** that happened to that request from Controller to Service, in chronological order — no guessing which log lines belong to which request.

---

# Lead's Key Takeaway

1. **A good log is written for "a future person with zero context" to understand instantly.** When writing a log message, ask yourself: "if I found this line at 3am during on-call with no other context, would I understand what happened?" If you're not sure, always add the necessary fields (IDs, relevant values).
2. **Correlation ID isn't just "nice to have" — it's the difference between "debugged in 5 minutes" and "debugged over hours" in any system with multiple layers/services.** It should be a baseline requirement from day one of building the system, not something bolted on later once problems start — retrofitting it into an existing system is far harder than designing it in from the start.
