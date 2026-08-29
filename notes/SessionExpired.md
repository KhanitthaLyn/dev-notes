# Question

Users occasionally see:
"Your session expired" immediately after login.
Only happens in some regions.
What could cause this?

# Answer

# The Big Picture & Analogy

Imagine you're meeting two friends who live in different cities. One friend's watch is accurate; the other's watch runs 10 minutes fast without them knowing.

You tell both: **"Meet at exactly 3pm, no earlier."** The friend with the fast watch will think **"it's already past 3pm"** even though the actual meeting time hasn't arrived yet — because their watch doesn't match the true reference time.

This is exactly what happens with multiple servers across regions: **each machine has its own system clock, and if those clocks aren't precisely synchronized (even off by just a few seconds), anything relying on "time" for a decision (like JWT expiration) breaks immediately.**

# Why Do We Need It? (Why "Region" Is a Key Clue)

A JWT (JSON Web Token) has key claims like:
```json
{
  "sub": "user123",
  "iat": 1735689600,  // issued at
  "exp": 1735693200   // expiration
}
```

Checking whether a token has expired means comparing `exp` against **"the current time of the server doing the check"** (`System.currentTimeMillis()` or equivalent).

If the system is deployed across multiple regions (e.g. us-east, ap-southeast, eu-west) and **a server in one region has a clock running fast relative to what it should be** — that server will mistakenly believe **"current time has already passed exp,"** even though the token was just issued moments ago.

This is exactly why the problem **"only happens in certain regions"** — clock skew doesn't affect all machines equally; it depends on how well NTP sync is working in each data center.

# Core Logic & How It Works (Likely Causes, Ranked)

**Cause A: Clock skew between the server that issued the token and the one validating it (most likely)**
```
Server A (region: ap-southeast) issues a token: iat=1000, exp=1300 (300-second lifetime)
Server B (region: us-east) validates the token: its system clock runs 350 seconds ahead of Server A

Server B sees "current time" (by its own clock) = 1350
Compared against exp=1300 -> the system thinks "expired 50 seconds ago," even though the token was issued mere seconds earlier
```
This happens when a load balancer or global routing sends the "issue token" request and the "validate token" request to different servers/regions, and NTP (Network Time Protocol) sync between those two machines isn't precise enough.

**Cause B: Session/token stored in a region-specific cache that isn't synced**
```
Login -> session is written to a Redis instance in region A
Next request -> load balancer routes to a server in region B
Region B tries reading the session from its own Redis replica -> hasn't synced yet (replication lag)
-> sees no session -> interprets it as "expired" or "invalid"
```
Same pattern as our earlier Horizontal Scaling discussion — **incomplete statelessness** if state (session) still ties itself to a particular region.

**Cause C: Timezone parsing bug**
```java
// Dangerous: relies on the server's default timezone instead of always fixing to UTC
LocalDateTime expiry = LocalDateTime.now().plusMinutes(30);
```
If servers across different regions have mismatched default timezones (one set to UTC, another to a local timezone), expiration calculations shift by the timezone offset.

**Cause D: CDN/edge cache incorrectly caching a login response**
Some edge locations might accidentally cache the login endpoint's response (if cache-control headers are misconfigured), causing clients to receive an actually-already-expired token.

# Trade-offs & When to Use (Fixes)

**Main fix: ensure every server relies on the same trustworthy time source**

| Approach | Description | When to use |
|---|---|---|
| **Enforce UTC everywhere** | Every server, every timestamp calculation must use UTC only, never local server timezone | Always — this is baseline |
| **Tighter NTP sync** | Configure NTP to sync more frequently (e.g. every minute instead of every hour), and monitor clock drift | When deploying across multiple regions/data centers |
| **Add clock skew tolerance (leeway)** | Allow a small buffer when validating `exp` (e.g. accept a token "expired" by up to 30 seconds) | An easy, fast-to-implement defensive measure |
| **Centralized time service** | Use a shared time source (e.g. the same NTP pool for all regions) instead of letting each machine sync independently | Large systems where time-based logic is critical |

**Code fix for JWT validation leeway:**
```java
JwtParser parser = Jwts.parserBuilder()
    .setSigningKey(secretKey)
    .setAllowedClockSkewSeconds(60) // tolerate up to 60 seconds of clock skew
    .build();
```

# Real-World Scenario (Ecommerce Domain)

Say an ecommerce system deploys multi-region (e.g. AWS ap-southeast-1 and us-west-2 for better latency across continents):

```
A user in Thailand logs in via the load balancer -> lands on a server in ap-southeast-1
Token issued with exp calculated from that machine's clock

The user immediately clicks "view cart" -> load balancer routes to a server in us-west-2
(could be due to failover or a traffic-balancing algorithm)

The us-west-2 server's clock runs 2 minutes ahead of ap-southeast-1 (imprecise NTP sync)
-> validates the token and sees "already expired," even though login happened less than a minute ago
```

**Debugging step a Lead should take:** examine the actual `iat` and `exp` in the failing JWT payload against **the timestamp of the server performing validation** (not the one that issued it) — if the numbers differ by only a few seconds/minutes, that's a clear signal of clock skew, not a logic bug.

# Lead's Key Takeaway

> **"Whenever a bug involves 'time' and only shows up on specific servers/regions, suspect clock synchronization first — before looking for a bug in business logic."**
>
> A good Lead knows that **distributed systems have no perfectly accurate "central clock" by nature** — every server has its own clock that can always drift somewhat (even with NTP in place). Systems that rely on comparing time across servers (token expiration, distributed lock timeouts, event ordering) must therefore **always build in tolerance for clock skew, rather than assuming every server's clock matches exactly.**
