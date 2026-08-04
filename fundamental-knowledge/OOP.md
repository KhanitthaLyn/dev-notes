# 💡 1. The Big Picture & Analogy

**OOP (Object-Oriented Programming)** is a way of designing code where you model things in your system as **Objects** 
units that bundle together **data (attributes)** and **behavior (methods)** in one place, instead of keeping data and functions separate.

**Analogy:** Think of building a house from a **blueprint**.

- **Class** = the blueprint (defines what the house should have — rooms, doors, color scheme)
- **Object** = an actual house built from that blueprint

You can build many houses from the same blueprint. Each house has its own color, size, owner (this is **state/data**), 
but every house has doors, windows, and an electrical system that behave the same way (this is **behavior**).

In backend terms: the `User` class is the blueprint, and `user1`, `user2` — each with their own id, email — are the actual Objects created from that blueprint.

---

# 🧩 2. Why Do We Need It?

**The problem before OOP (Procedural Programming):**

- Data and the logic that manipulates it are completely separate. You'd have a plain `struct/data` like `User { name, email }`,
- and then loose, scattered functions like `sendEmail(user)`, `validateUser(user)` living in different files across the codebase.
- As business logic grows more complex (e.g. an ecommerce system with User, Product, Order, Payment), the code becomes tangled
- changing one field can silently break dozens of functions that depend on it, without any clear warning.
- It's hard to reuse code and scale across a large team, because there's no clear boundary defining who's allowed to modify what data.

**Why OOP solves this:**

- Data + behavior live together in one unit (Object) — logic related to User stays inside the `User` class, not scattered across the project.
- It creates clear boundaries through **encapsulation** — you control who can access or modify data.
- It maps naturally to the real world and to your database schema (1 table ≈ 1 class ≈ 1 entity), making it easier for a backend team to communicate using a shared vocabulary.

---

# ⚙️ 3. Core Logic & How It Works

OOP rests on **4 Pillars** you need to know cold:

### 1) Encapsulation
Hide internal state (private fields) and expose access only through public methods (getter/setter) — this prevents data from being modified without validation or control.

### 2) Abstraction
The caller doesn't need to know *how* something works internally — just *what* to call and *what* they get back. For example, `order.calculateTotal()` 
the caller doesn't need to know how tax and discounts are computed inside.

### 3) Inheritance
A child class inherits properties/methods from a parent class, reducing duplicated code. Example: `AdminUser extends User`.

### 4) Polymorphism
Different object types respond differently to the same method call. For example, `calculateShippingCost()` on `PhysicalProduct` vs `DigitalProduct` behaves differently, 
but is called through the same method name.

---

# ⚖️ 4. Trade-offs & When to Use

**When to use:**
- Backend systems with complex business domains (ecommerce, banking, booking systems) — OOP naturally models relationships between entities.
- Large teams that need clear boundaries between modules, preventing accidental cross-team data mutation.

**When NOT to use:**
- Short scripts, simple data pipelines, or stateless data-transformation work — a functional/procedural approach is often faster to write and easier to read.
- Watch out for over-engineering with deep class hierarchies (deep inheritance) — this makes debugging and reasoning about the code much harder.

**Trade-offs:**
- **You gain:** high maintainability, reusability, a natural mapping to the domain model, and a shared vocabulary across the team (User, Product, Order).
- **You pay:** more boilerplate code (getters/setters, constructors), and a steeper learning curve for beginners.

---

# 🛠️ 5. Real-World Scenario / Mini Example

Say you're building an ecommerce backend and need to design `User` and `Product` classes:

```java
public class User {
    private Long id;
    private String name;
    private String email;

    public User(Long id, String name, String email) {
        this.id = id;
        this.name = name;
        this.email = email;
    }

    public String getEmail() { return email; }

    public void setEmail(String email) {
        if (email == null || !email.contains("@")) {
            throw new IllegalArgumentException("Invalid email format");
        }
        this.email = email;
    }

    @Override
    public String toString() {
        return "User{id=" + id + ", name='" + name + "', email='" + email + "'}";
    }
}

public class Product {
    private Long id;
    private String name;
    private double price;
    private int stockQuantity;

    public Product(Long id, String name, double price, int stockQuantity) {
        this.id = id;
        this.name = name;
        this.price = price;
        this.stockQuantity = stockQuantity;
    }

    public boolean isInStock() {
        return stockQuantity > 0;
    }

    public void reduceStock(int quantity) {
        if (quantity > stockQuantity) {
            throw new IllegalStateException("Not enough stock");
        }
        this.stockQuantity -= quantity;
    }

    @Override
    public String toString() {
        return "Product{id=" + id + ", name='" + name + "', price=" + price + "}";
    }
}
```

**Notice:**
- `email` is private → it can only be set through `setEmail()`, which includes validation (**Encapsulation**)
- `reduceStock()` hides the stock-checking logic inside — the caller just says how much to reduce (**Abstraction**)
- `toString()` **overrides** the method inherited from Java's `Object` class (the parent of every class in Java), defining how the object gets printed — a small example of **Polymorphism**

---

# 🧠 6. Lead's Key Takeaway

1. **OOP isn't just about hitting all 4 pillars — it's about designing clear boundaries of responsibility.** The moment logic related to User starts living outside the `User` class, you've already lost the benefit of encapsulation.
2. **A class should speak the language of the business, not just mirror a database table.** Design methods with business meaning (e.g. `reduceStock()` instead of just `setStockQuantity()`) — this pays off enormously in long-term maintainability.
