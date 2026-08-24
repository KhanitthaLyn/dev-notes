# Question 
You add a new field to your API response.

- Old clients expect:
{ "name": "Alice" }

- New clients receive:
{ "name": "Alice", "email": "alice@example.com" }

Thousands of old apps are still live.
How do you evolve the API without breaking them?


# Answer

# The Big Picture & Analogy

Imagine you send a wedding invitation with just **"date, time, location."** Guests read those three lines and show up accordingly.

The following year, you start **adding a QR code for RSVP** to the invitation. Guests who received the old invitation without a QR code can still read those same three lines just fine — **completely unaffected**, because they never expected a QR code in the first place. They simply "ignore" the new addition.

This is the core principle of API design: **"a well-designed client should ignore fields it doesn't recognize."** If clients follow this, adding a new field breaks nothing at all.

# Why Do We Need It? (Separating What Actually Breaks vs. What Doesn't)

**What does NOT break (additive changes):**
- Adding a new field to a response (`email` in this example)
- Adding a new optional request parameter

**What DOES actually break (breaking changes):**
- Removing an existing field
- Renaming an existing field
- Changing an existing field's data type (e.g. `age: "25"` string → `age: 25` number)
- Changing an existing field's meaning (e.g. `status: "active"` used to mean user-active, now means something else)
- Changing a previously returned HTTP status code
- Adding a new required field to the request that old clients never send

**So why does the question frame this as a problem?** Because in the real world, there are **edge cases where even additive changes can break things**:
- An old client using **strict schema validation** (e.g. JSON Schema with `additionalProperties: false`) → throws an error the moment it sees an unfamiliar field
- An old client doing **exact-match comparison** against the whole response (e.g. comparing responses against a saved test snapshot) → a new field causes the test to fail

This is exactly why you still need a **versioning strategy** as a safety net, even when the change is theoretically non-breaking.

# Core Logic & How It Works

Here are the main API evolution strategies:

**Strategy A: Additive-Only Changes (the default approach you should always try first)**
```json
// before
{ "name": "Alice" }

// after — add a field, touch nothing else
{ "name": "Alice", "email": "alice@example.com" }
```
Golden rule: **"never remove, never rename, never change type — only add."** Follow this and 90% of API evolution never needs a new version at all.

**Strategy B: URL Versioning (when a genuine breaking change is unavoidable)**
```
GET /v1/users/123  -> { "name": "Alice" }
GET /v2/users/123  -> { "name": "Alice", "contact": { "email": "alice@example.com" } }
```
When you truly need a **non-backward-compatible restructure** (e.g. nesting `email` inside a `contact` object) — create an entirely new version, and run both versions side-by-side.

**Strategy C: Header-based Versioning**
```
GET /users/123
Accept: application/vnd.myapp.v2+json
```
A more elegant alternative to URL versioning, since the URL stays the same (nicer for REST purists), but the implementation is more complex.

**Strategy D: Deprecation Path (for sunsetting an old version)**
```
1. Announce v1 as deprecated (still functional)
2. Add warning headers: Deprecation: true, Sunset: Wed, 01 Jan 2027 00:00:00 GMT
3. Monitor which clients are still calling v1 (via logging/analytics)
4. Reach out to teams still on v1 to help them migrate
5. Retire v1 once confident nobody's using it anymore (or after the announced deadline passes)
```

# Trade-offs & When to Use

| Strategy | When to use | Trade-off |
|---|---|---|
| **Additive-only** | Default, always try this first | Limited — doesn't work for every case (sometimes real restructuring is unavoidable) |
| **URL Versioning** (`/v1`, `/v2`) | When a breaking change is genuinely necessary and you have diverse clients you can't fully control (public API, mobile apps that don't auto-update) | Must maintain multiple code versions simultaneously, increasing long-term complexity and cost |
| **Header-based Versioning** | When your team is mature and wants to keep URLs RESTful | More complex implementation, harder to debug (same URL, different behavior) |
| **GraphQL** (if starting fresh) | For new projects expecting frequent schema changes | Clients choose which fields to fetch, making additive changes essentially frictionless — but higher learning curve than REST |

**When NOT to version:** for a plain additive change (like this example), **don't rush to create a new version** — that just adds unnecessary maintenance overhead. Versioning should be reserved for genuinely breaking changes only.

# Real-World Scenario (Ecommerce Domain)

Say `GET /orders/{id}` currently returns:
```json
{ "orderId": "123", "totalPrice": 1500 }
```

**Case 1: Adding a `discountAmount` field (additive — no versioning needed)**
```json
{ "orderId": "123", "totalPrice": 1500, "discountAmount": 100 }
```
Old clients unaware of `discountAmount` are completely unaffected — deploy immediately, no announcement needed.

**Case 2: Changing `totalPrice` from Integer (cents) to Decimal (dollars) — breaking change, needs versioning**
```json
// v1 (before — stored in cents)
{ "orderId": "123", "totalPrice": 150000 }

// v2 (after — stored in dollars)  
{ "orderId": "123", "totalPrice": 1500.00 }
```
This **requires** a separate `/v2/orders/{id}`, because an old client parsing `totalPrice` as integer cents would compute completely wrong values if given decimal dollars instead — this is exactly the dangerous kind of silent bug: no errors thrown, just quietly wrong numbers.

**Deprecation in Spring Boot:**
```java
@RestController
@RequestMapping("/v1/orders")
@Deprecated
public class OrderControllerV1 {
    @GetMapping("/{id}")
    public ResponseEntity<OrderV1Dto> getOrder(@PathVariable String id) {
        response.getHeaders().add("Deprecation", "true");
        response.getHeaders().add("Sunset", "Wed, 01 Jan 2027 00:00:00 GMT");
        response.getHeaders().add("Link", "</v2/orders/" + id + ">; rel=\"successor-version\"");
        // ...
    }
}
```

# Lead's Key Takeaway

> **"Don't fear adding new fields — fear removing, renaming, or changing the type/meaning of existing ones. That's what actually constitutes a breaking change."**
>
> A good Lead establishes a **clear API contract** from the start (e.g. via an OpenAPI/Swagger spec) and sets a team rule: **"additive changes ship without versioning; anything else must go through the versioning process."** Most importantly, they teach client-side developers to write **defensive parsing code** from day one (ignore unknown fields, don't throw errors on unexpectedly missing fields where possible) — because this is what makes future API evolution as smooth as it can possibly be.
