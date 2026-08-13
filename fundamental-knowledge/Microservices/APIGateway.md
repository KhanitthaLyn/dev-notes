# The Big Picture & Analogy

An **API Gateway** is a **"single entry point"** that every external request must pass through before reaching the actual microservices behind it — it acts as an intermediary handling routing, security, and traffic control for every service, so each service doesn't have to implement this repeatedly on its own.

**Analogy:** Think of **the lobby of a large office building housing multiple companies**.

- Without a lobby (no Gateway): visitors have to know exactly which floor and which door each company is on themselves, and each company would need its own security guard checking IDs (authentication duplicated everywhere) — messy and highly redundant.
- With a lobby (with a Gateway): all visitors enter through one lobby, where a guard checks IDs at the entrance **once** (centralized authentication), and the receptionist then **routes** them to the correct company based on who they're there to see. If too many people are trying to get in, the lobby can limit how many enter (rate limiting), without every individual company inside having to guard their own door.

An API Gateway does exactly this in a microservices system — it's the "receptionist" standing at the front door, handling every request **before** distributing it to the correct service inside.

---

# Why Do We Need It?

**The problem before API Gateway (clients calling services directly):**

```
Client → UserService (has to implement auth itself)
Client → ProductService (has to implement auth itself)
Client → OrderService (has to implement auth itself)
```

- **The client has to know the address of every service:** with 20 microservices, the client (mobile app, frontend) would need to store 20 different URLs — whenever a service scales up or moves address, every client needs updating. Extremely fragile.
- **Cross-cutting concerns get duplicated in every service:** authentication, rate limiting, logging all have to be re-implemented in every service (violating the DRY principle you already learned) — changing how JWT tokens get validated means updating every service simultaneously.
- **The client has to know internal system structure:** e.g. needing to know "call ProductService first, then call InventoryService" — this exposes internal details the client shouldn't need to know, violating system-level encapsulation (similar to what you learned in OOP).
- **Security risk:** if every service is exposed publicly and directly, the attack surface becomes huge, making it hard to maintain consistent security everywhere.

**Why API Gateway solves this:**
- The client only needs to know **"one single point"** (the Gateway URL) — no need to know the internal service structure at all. The Gateway hides all that complexity.
- Cross-cutting concerns (auth, rate limiting, logging) are handled **once, at the Gateway** — no duplicating them in every service.
- It reduces attack surface, since there's only one point that needs to be locked down thoroughly (this is where defense in depth begins).

---

# Core Logic & How It Works

## Core Responsibilities of an API Gateway

### 1) **Routing** — sending requests to the correct service

```
Client Request: GET /api/products/123
                        │
                        ▼
                  API Gateway
                        │
              (looks at the path prefix "/products")
                        │
                        ▼
              routes to ProductService
              (http://product-service:8081/products/123)
```

```yaml
# example routing configuration (Spring Cloud Gateway)
routes:
  - id: product-service-route
    uri: http://product-service:8081
    predicates:
      - Path=/api/products/**
  - id: order-service-route
    uri: http://order-service:8082
    predicates:
      - Path=/api/orders/**
  - id: user-service-route
    uri: http://user-service:8083
    predicates:
      - Path=/api/users/**
```

**What's happening here:** the client has no idea what port `product-service` runs on, or even how many instances of it are running — the Gateway handles routing (and usually load balancing too, if there are multiple instances) for all of it.

### 2) **Authentication/Authorization** — verifying identity before letting requests through

```java
@Component
public class AuthenticationFilter implements GlobalFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = extractToken(exchange.getRequest());

        if (token == null || !jwtValidator.isValid(token)) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete(); // stops here, never reaches the service inside
        }

        // attach user info decoded from the token to a header passed on to the service inside
        ServerHttpRequest modifiedRequest = exchange.getRequest().mutate()
            .header("X-User-Id", jwtValidator.extractUserId(token))
            .build();

        return chain.filter(exchange.mutate().request(modifiedRequest).build());
    }
}
```

**The key point:** services behind the Gateway (`OrderService`, `ProductService`) **never validate JWT tokens themselves** — they trust the `X-User-Id` header the Gateway attaches, because the Gateway is the only point exposed to external traffic (the internal network is closed off, preventing external traffic from reaching services directly).

### 3) **Rate Limiting** — limiting request volume to prevent system overload

```java
@Bean
public RedisRateLimiter redisRateLimiter() {
    // allow 10 requests/second per user, burst capacity up to 20
    return new RedisRateLimiter(10, 20);
}
```

```yaml
routes:
  - id: product-service-route
    uri: http://product-service:8081
    predicates:
      - Path=/api/products/**
    filters:
      - name: RequestRateLimiter
        args:
          redis-rate-limiter.replenishRate: 10
          redis-rate-limiter.burstCapacity: 20
```

**Why do this at the Gateway instead of at each individual service:** if a user fires requests too frequently, the Gateway **blocks them right at the source**, before the traffic ever reaches the services inside — preventing the internal services from having to absorb load that shouldn't have gotten through in the first place (just like a lobby guard stopping overcrowding at the entrance, instead of leaving every company inside to fend for themselves).

---

# Trade-offs & When to Use

**When to use an API Gateway:**
- Systems with **more than 1 microservice** that clients need to call — this is essentially a standard requirement for microservices architecture.
- When you need to centralize cross-cutting concerns (auth, rate limiting, logging, monitoring) in one place.

**When NOT to over-engineer:**
- A Monolith or single-service system — a Gateway isn't necessary at all here (adds complexity with no benefit).
- Never put business logic inside the Gateway — the Gateway should only handle routing/security/traffic control. If business logic starts creeping in (e.g. calculating prices), the Gateway is overstepping its scope.

**Trade-offs:**
- **You gain:** a simple client (only needs to know one point), centralized security, and massively reduced duplicated code across services.
- **You pay:** it becomes a **Single Point of Failure** — if the Gateway goes down, **the entire system becomes unreachable**, even though every service inside is working perfectly fine (the Gateway must always be designed for high availability, e.g. running multiple instances behind a load balancer). It also adds one extra network hop (slightly increasing latency for every request).

---

# Real-World Scenario — A Full API Gateway Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    API GATEWAY REQUEST FLOW                       │
└─────────────────────────────────────────────────────────────────┘

  Client (Mobile App / Frontend)
       │
       │  GET /api/orders/123
       │  Header: Authorization: Bearer <JWT>
       ▼
┌─────────────────────────────────────────┐
│              API GATEWAY                  │
│                                           │
│  Step 1: Rate Limiting Check              │
│  ├─ has the user exceeded the limit?      │
│  └─ if so → return 429 Too Many Requests  │
│     (stops here, doesn't pass through)    │
│                                           │
│  Step 2: Authentication                   │
│  ├─ is the JWT token valid?               │
│  └─ if not → return 401 Unauthorized      │
│     (stops here, doesn't pass through)    │
│                                           │
│  Step 3: Authorization                    │
│  ├─ does the user have access to this     │
│  │  resource?                              │
│  └─ if not → return 403 Forbidden         │
│     (stops here, doesn't pass through)    │
│                                           │
│  Step 4: Routing                          │
│  ├─ which route matches "/api/orders/**"? │
│  └─ → routes to OrderService              │
│                                           │
│  Step 5: Request Transformation           │
│  └─ attaches an X-User-Id header decoded  │
│     from the JWT                          │
└──────────────────┬────────────────────────┘
                    │
                    │  (internal network only —
                    │   no way to reach it directly
                    │   from outside)
                    ▼
         ┌─────────────────────┐
         │    OrderService      │
         │  (doesn't need to     │
         │   validate the JWT    │
         │   itself — trusts the │
         │   X-User-Id header    │
         │   the Gateway attached)│
         └──────────┬───────────┘
                    │
                    ▼
              [Order Data]
                    │
                    ▼ (goes back through the Gateway to the Client)
                Client receives the Response
```

**A simple example implementation (Spring Cloud Gateway):**

```java
@Configuration
public class GatewayConfig {

    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("order-service", r -> r
                .path("/api/orders/**")
                .filters(f -> f
                    .filter(authenticationFilter)  // Step 2, 3
                    .requestRateLimiter(c -> c.setRateLimiter(redisRateLimiter())) // Step 1
                    .filter(userIdHeaderFilter))   // Step 5
                .uri("lb://order-service"))        // Step 4 — lb:// = load balanced

            .route("product-service", r -> r
                .path("/api/products/**")
                .filters(f -> f
                    .filter(authenticationFilter)
                    .requestRateLimiter(c -> c.setRateLimiter(redisRateLimiter())))
                .uri("lb://product-service"))

            .build();
    }
}
```

**Testing that the Gateway performs each responsibility correctly:**

```java
@Test
void shouldReturn401WhenNoAuthToken() {
    // no Authorization header sent
    webTestClient.get().uri("/api/orders/123")
        .exchange()
        .expectStatus().isUnauthorized(); // Gateway blocks it before reaching OrderService
}

@Test
void shouldReturn429WhenRateLimitExceeded() {
    // firing requests beyond the configured limit
    for (int i = 0; i < 25; i++) {
        webTestClient.get().uri("/api/orders/123")
            .header("Authorization", "Bearer " + validToken)
            .exchange();
    }
    // the 25th request should hit the rate limit
    webTestClient.get().uri("/api/orders/123")
        .header("Authorization", "Bearer " + validToken)
        .exchange()
        .expectStatus().isEqualTo(429);
}

@Test
void shouldRouteToCorrectServiceWithValidToken() {
    webTestClient.get().uri("/api/orders/123")
        .header("Authorization", "Bearer " + validToken)
        .exchange()
        .expectStatus().isOk(); // passes every step, successfully reaches OrderService
}
```

---

# Lead's Key Takeaway

1. **An API Gateway applies the SRP principle at the system architecture level — completely separating "infrastructure/security concerns" from "business logic."** The services inside don't need to know anything about auth or rate limiting at all — they just focus on their own business logic. This is the same idea as `@ControllerAdvice` separating error handling from Controllers, applied at a much larger scale — the Gateway separates cross-cutting concerns from services.
2. **The Gateway is simultaneously the most powerful and the most fragile point in the system — it must be designed for exceptionally high availability.** Every request has to pass through it, so if the Gateway goes down, it's not just one service becoming unavailable — **the entire system becomes unreachable**, even though every service behind it is working perfectly fine. This is exactly why, in real architectures, Gateways are almost always deployed as multiple instances behind a load balancer, never as a single instance.
