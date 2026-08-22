# Question
Think Like a System Designer. Your app suddenly gets 10 million users. Which scaling method is typically preferred?

Vertical Scaling and Horizontal Scaling

Increase in processing power VS Increase in number of machines

# Answer

# The Big Picture & Analogy

Imagine a restaurant with a huge line of customers. Two ways to fix it:

- **Vertical Scaling** = keep the same chef, but send them to advanced training so they cook 10x faster (one chef, but much more skilled)
- **Horizontal Scaling** = hire 10 more chefs to work alongside in the same kitchen (multiple chefs, each with the same skill level)

The problem with the first option: **a chef can only get so skilled.** No matter how good they get, there's a physical limit (two hands, limited stove space). And if that chef gets sick, **the restaurant closes immediately** — there's no backup.

# Why Do We Need It? (Why Horizontal Is the Standard Answer)

**Vertical Scaling (Scale Up)** = upgrading the same server to be more powerful (more CPU, RAM, SSD on that one machine)

It has two layered problems:
- **Hardware ceiling:** a single machine has a physical limit. No matter how much you pay, upgrades eventually stop (there's no single server powerful enough to handle 10 million users alone)
- **Single Point of Failure (SPOF):** if that server goes down (hardware failure, network issue, bad deploy), **the entire system goes down**, since there's no backup

**Horizontal Scaling (Scale Out)** = adding more server machines instead of upgrading one

This solves both problems simultaneously: no fixed hardware ceiling (you can keep adding machines as demand grows), and no SPOF (if one machine fails, others keep serving traffic).

# Core Logic & How It Works

Remember the Alex Xu Chapter 1 workflow?

```
Client → Load Balancer → [Web Server 1, Web Server 2, Web Server 3, ...]
```

**Key components that make Horizontal Scaling actually work:**

1. **Load Balancer** — distributes traffic evenly across multiple servers (Round Robin, Least Connections, etc.). It's the "single door" clients see, while behind it work is spread across many machines.

2. **Stateless Web Tier** — each server must **not store session/state locally.** If request #1 lands on Server A, but request #2 (same user) lands on Server B — if state only lived on Server A, Server B wouldn't recognize that user at all. Session data must live in a centralized store instead (e.g. Redis).

3. **The database needs to scale too** — via Read Replicas (for read-heavy workloads) or Sharding (for write-heavy workloads), which we've already covered in Chapter 1.

# Trade-offs & When to Use

| Aspect | Vertical Scaling | Horizontal Scaling |
|---|---|---|
| **Ease of implementation** | Very simple — just upgrade hardware, no code changes | More complex — requires load balancing, statelessness, cross-node data consistency |
| **Ceiling** | Clear hardware ceiling | Practically no ceiling (keep adding machines) |
| **Fault tolerance** | Very low (SPOF) | High (one machine fails, others cover) |
| **Cost per unit of performance** | Rises non-linearly (very powerful servers get exponentially pricier) | More cost-effective at scale (many commodity machines are cheaper) |
| **Consistency** | Simple (state/data lives in one place, no sync needed) | More complex (requires managing distributed state, cross-node consistency) |

**When to use Vertical:** small systems, predictable and moderate traffic, or MVP/prototype stages where investing in distributed infrastructure upfront isn't yet worth it — **"simple first, scale later."**

**When to use Horizontal:** systems that need to handle high, unpredictable traffic (like a "sudden 10 million users" scenario), require high availability, or are production systems where downtime isn't acceptable.

# Real-World Scenario (Ecommerce Domain)

Say it's **Flash Sale / 11.11 day**, and traffic spikes 20x unexpectedly:

```
❌ With Vertical Scaling:
- You'd need to pause the system to resize the instance (downtime!)
- Eventually, there's no instance type powerful enough to handle this level of traffic at all

✅ With Horizontal Scaling + Auto Scaling Group:
- The system monitors CPU/traffic and automatically spins up new servers (e.g. 3 → 30 instances)
- The Load Balancer distributes traffic to new instances immediately, no downtime
- Once traffic drops after the sale ends, it scales back down automatically, saving cost
```

This is exactly why major ecommerce platforms (Shopee, Lazada, Amazon) all build on **Horizontal Scaling + Auto Scaling** as their architectural foundation — never Vertical Scaling alone.

# Lead's Key Takeaway

> **"Vertical Scaling is a short-term shortcut with a clear ceiling; Horizontal Scaling is an investment in an architecture that can scale nearly without limit — at the cost of added complexity."**
>
> A good Lead doesn't treat Horizontal Scaling as "always better." Instead, they recognize that **designing for statelessness from day 1** (even while the system is still small and doesn't need to scale yet) is one of the highest-value investments possible — because retrofitting a system with state baked into individual servers into a stateless one later is **far harder and riskier than designing it correctly from the start.**
