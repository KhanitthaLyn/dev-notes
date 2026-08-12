# The Big Picture & Analogy

**Database Modeling** is the process of translating "the real-world business domain" into a **schema (table structure)** that's accurate, efficient, and free of unnecessary data duplication.

**Analogy:** Think of **designing a filing cabinet system in an office**.

- Poor design: the same document gets copied and stuffed into multiple drawers. When the information changes (e.g. a customer's address changes), you have to go update every drawer that has a copy — if you miss one, the data contradicts itself (inconsistency).
- Good design: each drawer stores **only its own specific topic** (a customer drawer, an order drawer, a product drawer), and drawers reference each other via "index cards" (a card saying "see more detail in drawer X, item Y") instead of duplicating the data.

**Key** = the "national ID number" of each row in a table — a unique identifier used to reference across tables.
**Normalization** = the process of organizing data to eliminate unnecessary duplication (like organizing drawers so each topic lives in exactly one place, not scattered around).

---

# Why Do We Need It?

**The problem with a poorly designed (unnormalized) schema:**

```
Table: Orders
| order_id | customer_name | customer_email      | product_name | product_price |
|----------|---------------|----------------------|--------------|---------------|
| 1        | John Doe      | john@mail.com        | Laptop       | 25000         |
| 2        | John Doe      | john@mail.com         | Mouse        | 500           |
```

- **Redundancy:** John Doe's name and email are duplicated in every order row — wasting space and risking inconsistency.
- **Update Anomaly:** if John changes his email, you must update **every single row** containing his name. Miss even one → the data contradicts itself, and you no longer know which email is correct.
- **Insert Anomaly:** you can't add a new customer who hasn't placed an order yet, because this table always couples a customer to an order.
- **Delete Anomaly:** deleting a customer's only order also wipes out all their customer info, even though you never intended to delete the customer.

**Why Normalization + Key design solve this:**
- Each entity's data is separated according to "its own responsibility" (much like the SRP you already learned!) — the Customer table stores only customer data, the Order table stores only order data, and they connect via keys.
- Data gets edited in exactly one place (single source of truth) — change an email once in the Customer table, and every joined order automatically reflects the new value.

---

# Core Logic & How It Works

## Keys — The Heart of Connecting Data

### **Primary Key (PK)**
A unique identifier for each row in a table — every table should always have one.

```sql
CREATE TABLE customers (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,  -- PK
    name VARCHAR(255),
    email VARCHAR(255)
);
```

### **Foreign Key (FK)**
A field in one table that "references" the Primary Key of another table — this is the mechanism that creates relationships between tables.

```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    customer_id BIGINT,                     -- FK
    total DECIMAL(10,2),
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);
```

**FK enforces referential integrity:** the database won't allow inserting an order with a `customer_id` that doesn't actually exist in the customers table — preventing orphaned data.

### **Composite Key**
Used when multiple columns together are needed to establish uniqueness — e.g. junction tables (covered below in Many-to-Many).

## Normal Forms — A Progressive Cleanup Process

### **1NF (First Normal Form)** — No multi-valued cells

```
❌ Wrong:
| order_id | products          |
|----------|--------------------|
| 1        | Laptop, Mouse, Bag |   ← one cell holding multiple values

✅ Correct: split into one value per row (or a separate table)
| order_id | product |
|----------|---------|
| 1        | Laptop  |
| 1        | Mouse   |
| 1        | Bag     |
```

### **2NF (Second Normal Form)** — every non-key column must depend on the *entire* primary key (matters when there's a composite key)

```
❌ Wrong: (composite key = order_id + product_id)
| order_id | product_id | product_name | quantity |
|----------|------------|---------------|----------|
| 1        | 101        | Laptop        | 1        |
   product_name depends only on product_id, not on order_id too → violates 2NF

✅ Correct: move product_name into a separate Products table
```

### **3NF (Third Normal Form)** — no column may depend on another non-key column (transitive dependency)

```
❌ Wrong:
| order_id | customer_id | customer_city | customer_zip |
   customer_zip depends on customer_city, not directly on order_id → violates 3NF

✅ Correct: move customer_city, customer_zip into a Customers/Addresses table
```

**A simple rule of thumb for 3NF:** "does this field say something directly about *this* table, or is it actually about a different table entirely?"

## Relationship Types

### **One-to-Many (1:N)** — the most common
```sql
-- 1 Customer can have many Orders
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    customer_id BIGINT,  -- FK lives on the "many" side (Order)
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);
```

### **Many-to-Many (M:N)** — requires a Junction Table
```sql
-- 1 Order can have many Products, and 1 Product can appear in many Orders
CREATE TABLE order_items (           -- junction/join table
    order_id BIGINT,
    product_id BIGINT,
    quantity INT,
    PRIMARY KEY (order_id, product_id),  -- composite key
    FOREIGN KEY (order_id) REFERENCES orders(id),
    FOREIGN KEY (product_id) REFERENCES products(id)
);
```

**Why a junction table is required:** relational databases can't natively express M:N relationships directly — you need a middle table that stores each "pair" of the relationship (each row is one actual order-product pairing).

### **One-to-One (1:1)** — less common
```sql
-- 1 User has 1 UserProfile (split into a separate table for data queried less often)
CREATE TABLE user_profiles (
    id BIGINT PRIMARY KEY,
    user_id BIGINT UNIQUE,  -- UNIQUE is what makes this 1:1 instead of 1:N
    bio TEXT,
    FOREIGN KEY (user_id) REFERENCES users(id)
);
```

---

# Trade-offs & When to Use

**When to fully normalize (3NF or beyond):**
- Systems that **write frequently** and need high data correctness (e.g. banking, ecommerce orders/inventory) — preventing data inconsistency matters more than read speed.
- Systems where data changes often (e.g. frequently adjusted product prices) — normalization means updating in one place, done.

**When to denormalize (intentionally allowing some duplication):**
- Systems that are **read-heavy relative to writes**, where joining many tables makes queries too slow (e.g. dashboard analytics, reporting) — accept some duplication in exchange for read speed.
- **Real-world example:** storing `product_name` and `product_price` directly inside `order_items` (even though it looks "duplicated" from the Products table), because you need a "snapshot" of the price at the moment of purchase. If the product's price changes later, old orders shouldn't inherit the new price! This is **intentional** denormalization, driven by a business reason — not a mistake.

**Trade-offs:**
- **Heavy normalization:** high data integrity, no self-contradicting data, but at the cost of queries requiring multiple JOINs (slower as data grows).
- **Denormalization:** faster queries (fewer joins), but at the cost of data-inconsistency risk and extra logic needed to keep duplicated data in sync.

---

# Real-World Scenario / Mini Example

**Designing a full Ecommerce ERD (Entity Relationship Diagram):**

```sql
-- ===== Customers =====
CREATE TABLE customers (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ===== Addresses (1 Customer can have many Addresses — 1:N) =====
CREATE TABLE addresses (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    customer_id BIGINT NOT NULL,
    street VARCHAR(255),
    city VARCHAR(100),
    zip_code VARCHAR(20),
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);

-- ===== Categories =====
CREATE TABLE categories (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL
);

-- ===== Products (1 Category can have many Products — 1:N) =====
CREATE TABLE products (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    category_id BIGINT,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    stock_quantity INT DEFAULT 0,
    FOREIGN KEY (category_id) REFERENCES categories(id)
);

-- ===== Orders (1 Customer can have many Orders — 1:N) =====
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    customer_id BIGINT NOT NULL,
    shipping_address_id BIGINT NOT NULL,
    status VARCHAR(50) DEFAULT 'PENDING',
    total DECIMAL(10,2) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (customer_id) REFERENCES customers(id),
    FOREIGN KEY (shipping_address_id) REFERENCES addresses(id)
);

-- ===== Order Items (Junction Table for Order <-> Product : M:N) =====
CREATE TABLE order_items (
    order_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INT NOT NULL,
    price_at_purchase DECIMAL(10,2) NOT NULL,  -- intentional denormalization! stores a price snapshot at purchase time
    PRIMARY KEY (order_id, product_id),        -- composite key
    FOREIGN KEY (order_id) REFERENCES orders(id),
    FOREIGN KEY (product_id) REFERENCES products(id)
);

-- ===== Payments (1 Order has 1 Payment — 1:1) =====
CREATE TABLE payments (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_id BIGINT UNIQUE NOT NULL,           -- UNIQUE = enforces 1:1
    payment_method VARCHAR(50),
    status VARCHAR(50),
    paid_at TIMESTAMP,
    FOREIGN KEY (order_id) REFERENCES orders(id)
);
```

**All the relationships in this system:**

```
Customer (1) ──< (N) Address        [1 customer has many addresses]
Customer (1) ──< (N) Order          [1 customer has many orders]
Category (1) ──< (N) Product        [1 category has many products]
Order (1) ──< (N) OrderItem >── (N) Product   [M:N via a junction table]
Order (1) ──── (1) Payment          [1 order has exactly 1 payment]
```

**Key design decisions worth noting:**

1. **`order_items.price_at_purchase`** — this is intentional denormalization, because the price in the `products` table might change over time, but an old order must always retain the price it was purchased at (a critical business requirement).
2. **`orders.shipping_address_id`** — references the specific address chosen at checkout time, rather than querying through the customer directly, since a customer might have multiple addresses and pick a different one for each order.
3. **`payments.order_id UNIQUE`** — using a `UNIQUE` constraint on a foreign key is the standard technique for enforcing a 1:1 relationship (without `UNIQUE`, it would silently become 1:N instead).

---

# Lead's Key Takeaway

1. **Normalization isn't a rule to rigidly apply everywhere — it's the "starting point" you need to understand before you can denormalize deliberately, with good reason.** Always start designing at 3NF, then denormalize only in specific spots with a **clear business reason** (like a price snapshot) or a **measured, real performance reason** — never denormalize just to avoid writing a join.
2. **When designing an ERD, ask this question of every relationship: "Can one side have many of the other, and can the other side also have many of the first?"** The answer (neither = 1:1, one side can have many = 1:N, both sides can have many = M:N) immediately tells you where to place the foreign key, or whether you need a junction table. This is the most fundamental question in schema design, and it's the one most often answered wrong without practicing systematic thinking about it.
