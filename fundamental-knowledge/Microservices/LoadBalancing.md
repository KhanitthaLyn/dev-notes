# The Big Picture & Analogy

A **Load Balancer (LB)** is a "traffic distributor" that sits in front of multiple servers, deciding which server instance each incoming request should be sent to — so that no single server gets overloaded while others sit idle.

**Analogy:** Think of **a cashier directing customers to multiple payment counters**.

- Without someone directing traffic: customers walk up to whichever counter they like — some counters end up with huge lines while others sit empty, because customers don't actually know which counter is free.
- With someone directing traffic (a Load Balancer): they stand at the entrance, watch which counter is least busy, and **send the customer there** — every counter gets a roughly even load, none too crowded, none too empty.

**Horizontal Scaling** means **adding more payment counters** (adding server instances) as customer volume grows, instead of trying to make a single cashier work faster and faster (Vertical Scaling — upgrading one server to be more powerful). The Load Balancer is what makes horizontal scaling actually work — because adding more servers is pointless if nothing distributes traffic to the new ones.

---

# Why Do We Need It?

**The problem before Load Balancers (Single Server):**

```
Client → Server (single instance)
```

- **Single Point of Failure:** if this server goes down, **the entire system becomes unreachable**, with no backup.
- **Severely limited scaling:** even upgrading hardware (CPU, RAM) more and more (Vertical Scaling) hits a **practical ceiling** — the most powerful hardware on the market still has limits, and the more powerful it gets, the more disproportionately expensive it becomes (diminishing returns).
- **Downtime during deployment:** deploying a new version means temporarily shutting down this single server = the entire system is down during that time.

**Why Load Balancer + Horizontal Scaling solve this:**
- Multiple servers run simultaneously — if one crashes, the others keep handling traffic (**high availability**).
- Scaling has essentially no ceiling — need to handle more traffic? Just keep adding server instances (far cheaper than buying a single extreme-spec machine).
- Deployments can happen with zero downtime — take down one server at a time to update it (rolling update), while the others keep serving traffic.

---

# Core Logic & How It Works

## Load Balancing Algorithms — How to Decide Which Server Gets a Request

### 1) **Round Robin** — cycles through servers in order
```
Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A (cycles back around)
```
Simplest approach, but doesn't account for whether a server is currently busy or not.

### 2) **Least Connections** — sends to whichever server has the fewest active connections
```
Server A: 50 active connections
Server B: 20 active connections  ← sent here, since it has the fewest
Server C: 45 active connections
```
Smarter than Round Robin, since it accounts for each server's actual current load rather than just cycling in sequence.

### 3) **Weighted Round Robin** — more powerful servers get more requests
```
Server A (powerful — high spec): weight 3 → gets 3 shares of all requests
Server B (standard spec): weight 1 → gets 1 share
```
Used when servers have unequal specs (e.g. during a hardware migration).

## Health Checks — The Critical Piece Often Overlooked

```java
@RestController
public class HealthController {
    @GetMapping("/health")
    public ResponseEntity<String> healthCheck() {
        // check whether the database connection is still alive, whether critical dependencies are ready
        if (databaseIsHealthy() && dependenciesAreHealthy()) {
            return ResponseEntity.ok("UP");
        }
        return ResponseEntity.status(503).body("DOWN");
    }
}
```

**The Load Balancer periodically pings `/health` on every server (e.g. every 10 seconds):**
```
LB → GET /health → Server A → "UP" (200)   → keeps sending traffic normally
LB → GET /health → Server B → "DOWN" (503) → immediately stops sending traffic to Server B
LB → GET /health → Server B → "UP" (200)   → (once recovered) starts sending traffic back to it
```

**Why this is so important:** without health checks, the Load Balancer keeps sending traffic to a dead server (**a blind spot**), causing every request routed to it to fail — when it should have been avoided in favor of a healthy server from the start.

## Layer 4 vs Layer 7 Load Balancing

**Layer 4 (Transport Layer):** decides based purely on IP + Port, without looking at request content — very fast, but not very smart.
**Layer 7 (Application Layer):** can read HTTP headers, paths, cookies — enabling more sophisticated decisions, like routing based on URL path (`/api/products` to the Product server, `/api/orders` to the Order server).

---

# Trade-offs & When to Use

**What makes Load Balancing actually work in practice — connecting to a previous topic:**

- **Servers must be Stateless (connecting to the Design Thinking topic):** the Load Balancer distributes requests without guaranteeing which server a given user's next request will land on. If a server keeps state in memory (e.g. a shopping cart), this distribution pattern will cause data to disappear mid-session. **Load Balancing and Stateless Design are always paired together.**

**When to use Round Robin vs Least Connections:**
- Round Robin: good for requests that take roughly the same amount of time to process every time (e.g. simple APIs).
- Least Connections: good for requests with varying processing time (some heavier than others) — preventing an already-busy server from being handed even more work.

**Trade-offs:**
- **You gain:** high availability, near-limitless scaling, zero-downtime deployments.
- **You pay:** the Load Balancer itself becomes a critical point that must be highly available (often requiring a backup LB layer, or DNS-based failover), adds one extra network hop (slightly increased latency), and requires that every server genuinely be stateless before you get the full benefit.

---

# Real-World Scenario — A Flow Diagram for a Load-Balanced Service

```
┌───────────────────────────────────────────────────────────────┐
│              LOAD BALANCED ECOMMERCE SERVICE FLOW                │
└───────────────────────────────────────────────────────────────┘

                        Client Requests
                               │
                               ▼
                   ┌───────────────────────┐
                   │     Load Balancer       │
                   │  (Layer 7, Least Conn)  │
                   │                         │
                   │  Health Check every 10s │
                   │  ─────────────────────  │
                   │  Server A: UP  (20 conn)│
                   │  Server B: UP  (15 conn)│──┐ request sent here
                   │  Server C: DOWN         │  │ since it has the fewest connections
                   └───────────┬─────────────┘  │
                               │                 │
              ┌────────────────┼─────────────────┘
              │                │                
              ▼                ▼                
     ┌────────────┐   ┌────────────┐   ┌────────────┐
     │  Server A   │   │  Server B   │   │  Server C   │
     │ (stateless) │   │ (stateless) │   │  (DOWN —    │
     │             │   │             │   │  no traffic │
     │             │   │             │   │  routed)    │
     └──────┬──────┘   └──────┬──────┘   └─────────────┘
            │                 │
            └────────┬────────┘
                      │
                      ▼
            ┌───────────────────┐
            │  Shared Database    │
            │  + Redis Cache       │
            │  (the real state     │
            │   lives here, not     │
            │   in server memory)   │
            └───────────────────┘
```

**Key design points to notice:**

1. **Server C is automatically removed from rotation** because its health check failed — the Load Balancer won't send it traffic until the health check returns "UP" again.
2. **Servers A and B are stateless** — whether a user gets routed to A or B makes no difference to the result, since real state lives in the Database/Redis, not in server memory.
3. **The Least Connections algorithm** routes traffic to whichever server currently has the least load, not just cycling in fixed order.

**Explaining Horizontal Scaling in practice — a Flash Sale traffic spike scenario:**

```
Before the Flash Sale (normal traffic):
Load Balancer → [Server A, Server B]  (2 instances are enough)

During the Flash Sale (traffic spikes 10x):
Auto-scaling detects CPU/traffic exceeding a threshold
→ automatically adds instances
Load Balancer → [Server A, Server B, Server C, Server D, Server E]
(5 instances now handle the increased traffic — this works because servers 
 are stateless, needing no special data sync between new and existing instances)

After the Flash Sale (traffic returns to normal):
Auto-scaling scales back down to 2 instances to save cost
```

**This is exactly why Stateless Design (the previous topic) matters so much for Load Balancing:** if a server kept state in memory, suddenly spinning up new instances during a Flash Sale would leave those new instances **"knowing nothing"** about users currently active — but because servers are stateless, new instances are ready to accept traffic the moment they start, with no data sync required at all.

---

# Lead's Key Takeaway

1. **A Load Balancer isn't just "a traffic distributor" — it's a "health monitoring point" that gives the system partial self-healing ability.** A good health check lets the system automatically remove problematic servers from rotation without anyone manually watching over it. This is exactly the difference between a system that can "partially tolerate failure" and one that "collapses entirely when one point breaks."
2. **Load Balancing only works well when the design underneath it is correct — especially Stateless Design.** Trying to bolt a Load Balancer onto a system that still has state stuck in server memory won't deliver the full benefit (or worse, will create bizarre bugs). Always fix statelessness first — then horizontal scaling via a Load Balancer becomes automatically straightforward. This is exactly why these two topics (Stateless Design and Load Balancing) are always taught together when designing systems meant to genuinely scale.
