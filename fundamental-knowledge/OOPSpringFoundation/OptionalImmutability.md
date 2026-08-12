# The Big Picture & Analogy

This topic covers 2 related concepts:
**Optional** is a "box that explicitly tells you it might be empty," replacing `null`, which gives no such warning.
**Immutability** is designing an object so its values **cannot change after creation**.

**Analogy for Optional:** Think about tracking an online order — when you check delivery status, a well-designed system doesn't just show a blank screen (like `null`). 
It explicitly tells you either **"Tracking info not found"** (`Optional.empty()`) or shows the actual status (`Optional.of(status)`). 
You immediately know you need to check before using the value, instead of hitting an error screen unexpectedly.
**Analogy for Immutability:** Think of a **printed receipt** — once it's printed, nobody can go back and change the numbers on it. 
If the price was wrong, you issue a new receipt instead of scratching out the old number. This is exactly what gives data safety and trustworthiness.

---

# Why Do We Need It?

## The Problem with `null`

```java
public User findUser(Long id) {
    return userRepository.get(id); // might return null if not found
}

// caller uses it without knowing it might be null
User user = findUser(999L);
System.out.println(user.getName()); // 💥 NullPointerException!
```

- `null` communicates nothing about "this value might not exist" — the compiler gives no warning. Developers have to *remember* to check `if (user != null)` themselves, which is **extremely easy to forget**.
- `NullPointerException (NPE)` is one of the top causes of production bugs in the Java world, because it happens **at runtime** and often crashes far away from where the `null` was originally created — making debugging painful.
- Tony Hoare (who invented `null`) famously called it his own **"billion-dollar mistake,"** given the sheer scale of damage NPEs have caused worldwide.

## The Problem with Mutability

```java
public class Address {
    private String city;
    public void setCity(String city) { this.city = city; } // can be changed anytime, from anywhere
}

Address addr = user.getAddress();
addr.setCity("Bangkok"); // this user's address can be modified from literally anywhere in the system!
```

- If an object can be modified from anywhere (mutable), it becomes very hard to track "who changed this data, and when" — especially in large systems with multiple threads or services touching the same object.
- This causes **unexpected side effects** — function A modifies an object that function B is currently using, so B ends up with a different value than expected, with no idea it was changed underneath it.
- In multi-threaded environments (like concurrent requests on a backend), shared mutable objects easily lead to **race conditions**.

**Why Optional + Immutability solve this:**
- Optional forces the caller to **explicitly consider the "no value" case** at compile time (via the type system), instead of discovering it at runtime.
- Immutable objects guarantee that "however it was created, it stays that way forever" — making them inherently safe from side effects and naturally thread-safe (no locks needed).

---

# Core Logic & How It Works

## Optional

**Creating an Optional:**
```java
Optional<User> present = Optional.of(user);        // must not be null
Optional<User> empty = Optional.empty();            // intentionally empty
Optional<User> maybeNull = Optional.ofNullable(x);  // might be null (use when unsure)
```

**Using Optional the right way:**
```java
Optional<User> userOpt = findUser(999L);

// Approach 1: isPresent() + get() — works, but not idiomatic
if (userOpt.isPresent()) {
    System.out.println(userOpt.get().getName());
}

// Approach 2: ifPresent() — cleaner
userOpt.ifPresent(user -> System.out.println(user.getName()));

// Approach 3: orElse() — provide a default value if empty
User user = userOpt.orElse(User.guest());

// Approach 4: orElseThrow() — throw a meaningful exception if empty
User user = userOpt.orElseThrow(() -> new UserNotFoundException(999L));

// Approach 5: map() — chain transformations safely
String name = userOpt.map(User::getName).orElse("Unknown");
```

**Warning:** never call `.get()` directly without checking first — that completely defeats the purpose of Optional (you've just replaced NPE with `NoSuchElementException`).

## Immutability

**Rules for building an Immutable Class:**
1. Make fields `private final`
2. No setters
3. Set all values only through the constructor
4. If a field is a mutable object (e.g. List, Date), **defensively copy** it on the way in and out

```java
public final class Address { // final class — prevents subclassing that could alter behavior
    private final String street;
    private final String city;
    private final String zipCode;

    public Address(String street, String city, String zipCode) {
        this.street = street;
        this.city = city;
        this.zipCode = zipCode;
    }

    public String getStreet() { return street; }
    public String getCity() { return city; }
    public String getZipCode() { return zipCode; }

    // no setter at all — to "change" it, you create a new object
    public Address withCity(String newCity) {
        return new Address(this.street, newCity, this.zipCode); // create new, original untouched
    }
}
```

The **"with" method pattern** is the standard way to "modify" an immutable object — you're actually creating a new object with a slightly different value, while the original is never touched.

---

# Trade-offs & When to Use

**Optional — When to use:**
- Return type of a method that "might not find anything," e.g. `findUserById()`, `findProductByCode()`
- Should NOT be used as a class field or method parameter (community consensus: Optional belongs **only as a return type**)

**Optional — When NOT to use:**
- Don't use it as a field in an entity/DTO — it needlessly complicates serialization/deserialization (e.g. converting to JSON)
- Don't use it as a substitute for an empty `List` — if a collection has no data, return an `empty list`, not `Optional<List<T>>`

**Immutability — When to use:**
- Value objects with no identity of their own — e.g. `Address`, `Money`, `DateRange` — ideal because they're compared by content, not identity
- Objects frequently shared across threads (e.g. config, DTOs passed between services)

**Immutability — When NOT to use:**
- Entities that need to track frequently changing state, e.g. an `Order` whose status changes over time (pending → paid → shipped) — forcing full immutability here means creating too many new objects, becoming over-engineering

**Trade-offs:**
- **Optional:** gains compiler-assisted null-safety, at the cost of a bit more boilerplate and slight performance overhead (wrapper object creation).
- **Immutability:** gains thread-safety for free, prevents side-effect bugs, and makes debugging much easier (state never changes mid-flight) — at the cost of creating new objects more often (GC overhead) and requiring a mindset shift (from "mutate the existing value" to "create a new value").

---

# Real-World Scenario / Mini Example

Ecommerce backend — refactoring `findUser` to return `Optional`, and building an immutable `Address`:

```java
// Immutable Address — a value object with no identity of its own
public final class Address {
    private final String street;
    private final String city;
    private final String zipCode;

    public Address(String street, String city, String zipCode) {
        this.street = street;
        this.city = city;
        this.zipCode = zipCode;
    }

    public String getStreet() { return street; }
    public String getCity() { return city; }
    public String getZipCode() { return zipCode; }

    public Address withCity(String newCity) {
        return new Address(this.street, newCity, this.zipCode);
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Address)) return false;
        Address other = (Address) o;
        return street.equals(other.street) && city.equals(other.city) && zipCode.equals(other.zipCode);
    }
    // compared by "content" — since it's a value object, 2 addresses with identical fields are equal
}

public class UserRepository {
    private Map<Long, User> users = new HashMap<>();

    // Before: returns null if not found — dangerous
    // public User findUser(Long id) {
    //     return users.get(id); // could be null
    // }

    // After: returns Optional — forces the caller to consider the "not found" case
    public Optional<User> findUser(Long id) {
        return Optional.ofNullable(users.get(id));
    }
}

public class UserService {

    public String getUserDisplayName(Long userId) {
        return userRepository.findUser(userId)
            .map(User::getName)
            .orElse("Guest User"); // safe, no possibility of NPE
    }

    public User getUserOrThrow(Long userId) {
        return userRepository.findUser(userId)
            .orElseThrow(() -> new UserNotFoundException(userId)); // clear about what went wrong
    }

    public void updateUserCity(Long userId, String newCity) {
        User user = getUserOrThrow(userId);
        Address updatedAddress = user.getAddress().withCity(newCity); // new Address created, original untouched
        // update user with the new address (via a method returning a new User, if User is also immutable)
    }
}
```

**Notice the difference:** previously `findUser()` returned `null`, and the caller had to remember to check. Once changed to `Optional<User>`, **the compiler forces the caller to handle the "not found" case** — code won't even compile if you try to call a method on `User` directly from `Optional<User>` without unwrapping it first.

---

# Lead's Key Takeaway

1. **Optional isn't just "a replacement for null" — it's a change to the type signature that communicates meaning more clearly.** Seeing `Optional<User>` as a return type tells anyone reading the code immediately that "this method might not find anything," without needing to read the implementation or documentation at all.
2. **Immutability trades "convenience while writing" for "safety while debugging."** The more complex a system gets — more threads, more services accessing the same data — the more you should make value objects immutable, because the cost of debugging a bug caused by unexpected mutation is far higher than the cost of creating a new object.
