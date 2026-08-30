# Question

your JWT tokens are stateless. a user gets hacked. you need to invalidate their token immediately. but you can't.
what now?
how do you solve it?

# Answer

# The Big Picture & Analogy

Imagine a JWT like **a pre-printed concert ticket** — the ticket has everything printed on it (attendee name, seat, expiration date). The door staff don't need to call anyone; they just **check the ticket and verify the signature isn't forged**, then let you in immediately. This is the speed statelessness provides.

But the problem: if that ticket gets **stolen**, the venue **can't "call back" that specific ticket** — there's no central system tracking "which tickets are still valid." The door staff will keep letting whoever holds the (stolen) ticket in, until it **"expires as printed"** — no earlier.

This is the core of the problem: **stateless means "no one is holding state to check against," which is great for performance, but means there's no lever to pull to "cancel it."**

# Why Do We Need It? (Why It's Genuinely Hard)

A server validating a JWT does exactly two things:
1. Checks the **signature** is correct (hasn't been tampered with)
2. Checks the **exp claim** hasn't passed yet

**Neither step involves asking a database "is this token still valid?"** — this is the entire point of JWT: avoiding a database query on every single request for speed (unlike session-based auth, which always queries the DB/Redis).

The problem: **the speed gained and the ability to revoke instantly are inherently in tension.** The more stateless you are, the harder revocation becomes.

# Core Logic & How It Works (Solutions From Simplest to Most Complete)

**Approach A: Short-lived Access Token + Refresh Token (should already be baseline)**
```
Access Token: very short-lived (5-15 minutes)
Refresh Token: long-lived (7-30 days), but stored in a database (has state)
```
When a user gets hacked: **revoke the refresh token** (delete it from the DB immediately) — the old access token still works until it naturally expires (max 15 minutes), but afterward, no new token can be issued since the refresh token is gone.

**Limitation:** there's still a 5-15 minute window where the old access token remains valid — if you need truly "instant" revocation (zero-second window), this alone isn't enough.

**Approach B: Token Blacklist/Denylist (achieves genuine instant revocation)**
```java
public boolean isTokenValid(String token) {
    Claims claims = jwtParser.parse(token);
    
    // always check blacklist first (adds one very fast lookup step)
    if (redisTemplate.hasKey("blacklist:" + claims.getJti())) {
        return false; // token has been revoked
    }
    
    return true; // signature + exp valid, and not blacklisted
}

// when a user is compromised
public void revokeToken(String jti) {
    long remainingTtl = calculateRemainingTime(jti);
    redisTemplate.opsForValue().set("blacklist:" + jti, "revoked", remainingTtl, TimeUnit.SECONDS);
    // TTL matches exactly how much time the token had left -> no need to keep the blacklist entry forever
}
```
**Key insight:** this is **"reinjecting a small amount of state" back into JWT, which was designed to be stateless** — but it's a worthwhile trade-off because:
- The blacklist only stores **tokens that have been revoked** (a tiny fraction compared to all valid tokens)
- Uses **Redis**, which is extremely fast (in-memory lookup) instead of a full database query — you don't lose most of JWT's performance benefit
- The blacklist entry's **TTL matches the token's original remaining lifetime exactly** — once the token would have naturally expired anyway, the blacklist entry is no longer needed (auto cleanup)

**Approach C: Token Versioning (an alternative that doesn't require Redis)**
```java
// add a tokenVersion column to the User table
public class User {
    private Long id;
    private Integer tokenVersion; // incremented whenever a full revocation is needed
}

// the JWT payload embeds tokenVersion when issued
{ "sub": "user123", "tokenVersion": 5, "exp": ... }

// validation checks whether the token's version matches the DB's current version
public boolean isTokenValid(Claims claims) {
    User user = userRepo.findById(claims.getSub());
    return claims.get("tokenVersion").equals(user.getTokenVersion());
}

// when a user is compromised -> revoke every token for this user instantly
public void revokeAllTokens(Long userId) {
    userRepo.incrementTokenVersion(userId); // tokenVersion 5 -> 6
    // every old token carrying tokenVersion=5 becomes invalid immediately
}
```
**Advantage:** revokes **all of that user's tokens at once** (perfect for the "user got hacked, log out everywhere" scenario), but **the downside is a mandatory DB query on every validation** — sacrificing much more of the statelessness benefit than the blacklist approach.

# Trade-offs & When to Use

| Approach | Revocation speed | Performance cost | Best for |
|---|---|---|---|
| **Short-lived token + refresh** | Slow (waits for access token to expire) | Very low (keeps full statelessness) | General systems that can tolerate a short delay |
| **Blacklist (Redis)** | **Instant** | Low (Redis lookups are very fast) | When genuinely instant revocation matters (banking, security-critical) |
| **Token versioning** | Instant + revokes all sessions at once | Moderate (requires a DB query every validation) | When you need "log out everywhere" simultaneously |

**The most important trade-off to internalize:** **pure JWT (100% stateless) can never truly support instant revocation by design.** Every fix here is about **"reinjecting some amount of state"** to gain revocation capability — an acknowledgment that **pure stateless JWT is really more of a theoretical ideal; real systems that need robust security always carry some degree of state.**

**When NOT to worry:** if the system doesn't genuinely need "instant" revocation (e.g. a low-risk internal tool), plain short-lived tokens + refresh tokens are perfectly sufficient — no need to add blacklist complexity that isn't actually required.

# Real-World Scenario (Ecommerce Domain)

Say an ecommerce system detects that a customer's account has been compromised (suspicious orders, unauthorized password changes):

```java
@Service
public class SecurityIncidentService {
    
    public void handleAccountCompromise(Long userId, String currentToken) {
        // 1. instantly revoke the current token via blacklist
        Claims claims = jwtParser.parseToken(currentToken);
        long remainingSeconds = claims.getExpiration().getTime() - System.currentTimeMillis();
        redisTemplate.opsForValue().set(
            "blacklist:" + claims.getId(), 
            "compromised", 
            remainingSeconds, 
            TimeUnit.MILLISECONDS
        );
        
        // 2. revoke all this user's refresh tokens (prevent new access tokens from being issued)
        refreshTokenRepo.deleteAllByUserId(userId);
        
        // 3. force a password reset
        userRepo.forcePasswordReset(userId);
        
        // 4. notify the user via email/SMS
        notificationService.sendSecurityAlert(userId);
    }
}
```

**Middleware checking the blacklist on every request:**
```java
@Component
public class JwtAuthFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request, ...) {
        String token = extractToken(request);
        Claims claims = jwtParser.parse(token);
        
        // check the blacklist before any business logic runs
        if (redisTemplate.hasKey("blacklist:" + claims.getId())) {
            response.setStatus(401);
            response.getWriter().write("{\"error\":\"UNAUTHORIZED\"}");
            return;
        }
        
        // proceed to normal business logic
        filterChain.doFilter(request, response);
    }
}
```

# Lead's Key Takeaway

> **"Pure statelessness is more of a theoretical ideal — real systems requiring robust security always carry some level of state. The question isn't 'can we be 100% stateless,' it's 'how much state do we reinject to balance performance against security.'"**
>
> A good Lead doesn't see a JWT blacklist as "defeating the purpose of JWT." Instead, they understand it as a **pragmatic trade-off** — sacrificing a small amount of statelessness (a fast Redis lookup, not a full database query) in exchange for a capability genuinely required for real security. Good system design isn't about rigidly adhering to any one principle absolutely — it's about knowing when to "bend" that principle for the security that's actually needed.
