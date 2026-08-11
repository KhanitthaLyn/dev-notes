# The Big Picture & Analogy

**Layered Architecture** is a way of dividing a backend system into "layers," each with a clearly defined responsibility. Data flows through each layer in sequence — layers don't get skipped or mixed together carelessly.

**Analogy:** Think of a **restaurant with clearly divided roles**.

- **Controller** = the waiter — takes the customer's order (HTTP request), translates it into language the kitchen understands, then brings the finished dish back out to the customer (HTTP response) — **doesn't cook anything themselves**
- **Service** = the head chef — decides how the dish gets made, handles all the business logic (e.g. "if the customer wants extra spicy, how many extra chilies do we add?") — **doesn't go grab ingredients from storage themselves**
- **Repository** = the storekeeper — knows exactly where ingredients live in storage and hands them over on request — **doesn't know or care what dish they're going into**
- **DTO** = the order slip — a standardized format used to communicate between departments, not the raw ingredients themselves (it doesn't expose unnecessary internal details)

Everyone does their own job without stepping on anyone else's — the waiter doesn't need to know what the sauce is made of, and the head chef doesn't need to know which table the customer is sitting at.

---

# Why Do We Need It?

**The problem before clear Layered Architecture:**

```java
@RestController
public class ProductController {
    @PostMapping("/products")
    public Product createProduct(@RequestBody Product product) {
        // validation logic mixed in here
        if (product.getPrice() < 0) throw new RuntimeException("Invalid price");

        // business logic mixed in here
        product.setCreatedAt(LocalDateTime.now());

        // database access mixed in here too!
        Connection conn = DriverManager.getConnection(url);
        PreparedStatement stmt = conn.prepareStatement("INSERT INTO products...");
        // ...

        return product; // returns the raw database entity directly, exposing every field to the client!
    }
}
```

- The Controller does everything at once (HTTP handling + validation + business logic + database access) → it becomes a **God Class**, exactly what you learned to avoid in SOLID.
- **Testing becomes very hard**, because testing the business logic requires a real database connection too — there's no way to isolate it.
- **The database entity gets exposed directly to the client** — if the entity has sensitive fields (e.g. internal cost price, audit fields), customers see everything. And if the database schema changes, the API response changes right along with it, unintentionally (a breaking change).
- Switching from REST to gRPC, or changing database providers, requires tearing everything apart because everything is tangled together.

**Why Layered Architecture solves this:**
- It separates concerns based on "reason to change" (exactly matching the SRP you learned) — HTTP concerns changing doesn't affect business logic; business logic changing doesn't affect how the database is accessed.
- DTOs create a "protective wall" between the internal data model and what the client sees — the database schema can change while the API contract stays the same (if designed well).

---

# Core Logic & How It Works

## The 3 Main Layers and Their Responsibilities

### **Controller Layer** — the HTTP entry point
**Responsibility:** receive requests, convert the request body into an object, call the Service, and convert the result into an HTTP response.
**Should NOT do:** business logic, or direct database access.

```java
@RestController
@RequestMapping("/products")
public class ProductController {
    private final ProductService productService;

    public ProductController(ProductService productService) {
        this.productService = productService;
    }

    @PostMapping
    public ResponseEntity<ProductResponseDTO> createProduct(@RequestBody ProductRequestDTO request) {
        ProductResponseDTO created = productService.createProduct(request);
        return ResponseEntity.status(201).body(created);
    }
}
```

### **Service Layer** — where business logic lives
**Responsibility:** validate data against business rules, make logical decisions, coordinate across multiple Repositories if needed, and convert data between Entity ↔ DTO.
**Should NOT do:** know about HTTP (status codes, request/response objects), or write raw SQL.

```java
@Service
public class ProductService {
    private final ProductRepository productRepository;

    public ProductService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    public ProductResponseDTO createProduct(ProductRequestDTO request) {
        // business validation
        if (request.getPrice() < 0) {
            throw new InvalidProductException("Price cannot be negative");
        }

        // DTO -> Entity
        Product product = new Product(request.getName(), request.getPrice());
        Product saved = productRepository.save(product);

        // Entity -> DTO (for the response)
        return new ProductResponseDTO(saved.getId(), saved.getName(), saved.getPrice());
    }
}
```

### **Repository Layer** — the data access point
**Responsibility:** CRUD operations against the database, and nothing else.
**Should NOT do:** any business logic, and it shouldn't know about DTOs at all (it only works with Entities).

```java
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {
    // Spring Data JPA generates the implementation automatically
    List<Product> findByActiveTrue();
}
```

## DTO (Data Transfer Object) — Why We Need It

**Entity** (used internally, with the database) vs. **DTO** (used to communicate externally) — two completely different jobs:

```java
// Entity — mirrors the full database structure
@Entity
public class Product {
    @Id
    private Long id;
    private String name;
    private double price;
    private double internalCostPrice;  // sensitive data — should never reach the client!
    private LocalDateTime createdAt;
    private String createdByAdminId;   // internal audit field
}

// Request DTO — only the fields the client should send in when creating a product
public class ProductRequestDTO {
    private String name;
    private double price;
    // no id, createdAt — the client shouldn't be setting these
}

// Response DTO — only the fields the client should see
public class ProductResponseDTO {
    private Long id;
    private String name;
    private double price;
    // no internalCostPrice, createdByAdminId — kept hidden internally
}
```

**Mapping between Entity ↔ DTO** happens only at the **Service layer**:

```
Client → [RequestDTO] → Controller → Service → [Entity] → Repository → Database
Client ← [ResponseDTO] ← Controller ← Service ← [Entity] ← Repository ← Database
```

---

# Trade-offs & When to Use

**When to use Layered Architecture:**
- General backend systems with low-to-moderate business logic complexity — a solid default for nearly every project.
- Teams with multiple people working in parallel — work can be clearly divided by layer (one person on Repository, another on Service).

**When to reconsider / things to watch for:**
- Very small projects (scripts, prototypes) — building 3 layers + DTOs might be over-engineering for a simple 2-3 endpoint CRUD.
- **Anemic Service Layer** — if the Service is just a pass-through from Controller to Repository with zero actual logic (`return repository.findAll()`), that's a signal you might be adding unnecessary layers.

**Trade-offs:**
- **You gain:** clear separation of concerns, easy testing (each layer can be mocked independently), and minimal impact on other layers when changing the database or API protocol.
- **You pay:** significantly more files (Entity, 2x DTO, Controller, Service, Repository just for one feature), and repetitive mapping code between Entity/DTO (unless you use a library like MapStruct to help).

---

# Real-World Scenario / Mini Example

**Full Product CRUD with DTO mapping:**

```java
// ===== Entity =====
@Entity
public class Product {
    @Id @GeneratedValue
    private Long id;
    private String name;
    private double price;
    private boolean active;
    private LocalDateTime createdAt;

    // constructors, getters, setters
    public Product() {}
    public Product(String name, double price) {
        this.name = name;
        this.price = price;
        this.active = true;
        this.createdAt = LocalDateTime.now();
    }
    // getters/setters...
}

// ===== DTOs =====
public class ProductRequestDTO {
    private String name;
    private double price;
    // getters/setters
}

public class ProductResponseDTO {
    private Long id;
    private String name;
    private double price;

    public ProductResponseDTO(Long id, String name, double price) {
        this.id = id; this.name = name; this.price = price;
    }
    // getters
}

// ===== Repository =====
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {
    List<Product> findByActiveTrue();
}

// ===== Service — where business logic + mapping happens =====
@Service
public class ProductService {
    private final ProductRepository productRepository;

    public ProductService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    public ProductResponseDTO createProduct(ProductRequestDTO request) {
        if (request.getPrice() < 0) {
            throw new InvalidProductException("Price cannot be negative");
        }
        Product product = new Product(request.getName(), request.getPrice());
        Product saved = productRepository.save(product);
        return toResponseDTO(saved);
    }

    public ProductResponseDTO getProduct(Long id) {
        Product product = productRepository.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(id)); // using Optional, as learned before
        return toResponseDTO(product);
    }

    public List<ProductResponseDTO> getAllActiveProducts() {
        return productRepository.findByActiveTrue().stream() // using Streams, as learned before
            .map(this::toResponseDTO)
            .collect(Collectors.toList());
    }

    public void deleteProduct(Long id) {
        if (!productRepository.existsById(id)) {
            throw new ProductNotFoundException(id);
        }
        productRepository.deleteById(id);
    }

    // mapping logic centralized in one place — not scattered across the codebase
    private ProductResponseDTO toResponseDTO(Product product) {
        return new ProductResponseDTO(product.getId(), product.getName(), product.getPrice());
    }
}

// ===== Controller — thin, no logic =====
@RestController
@RequestMapping("/products")
public class ProductController {
    private final ProductService productService;

    public ProductController(ProductService productService) {
        this.productService = productService;
    }

    @PostMapping
    public ResponseEntity<ProductResponseDTO> create(@RequestBody ProductRequestDTO request) {
        return ResponseEntity.status(201).body(productService.createProduct(request));
    }

    @GetMapping("/{id}")
    public ResponseEntity<ProductResponseDTO> get(@PathVariable Long id) {
        return ResponseEntity.ok(productService.getProduct(id));
    }

    @GetMapping
    public ResponseEntity<List<ProductResponseDTO>> getAll() {
        return ResponseEntity.ok(productService.getAllActiveProducts());
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        productService.deleteProduct(id);
        return ResponseEntity.noContent().build();
    }
}
```

**Notice the flow of data:**
- Incoming requests arrive as `ProductRequestDTO` (no id, no createdAt — the client shouldn't set these)
- Service converts it into a `Product` entity to persist to the database
- Service converts it back into a `ProductResponseDTO` before returning (no `internalCostPrice` or audit fields, if any exist)
- Controller has no idea what logic happens inside — it just routes the request to the Service and wraps the result as an HTTP response

---

#  Lead's Key Takeaway

1. **Test whether a layer is "overreaching" with a simple question: if the communication channel changed (REST → gRPC) or the database changed (MySQL → MongoDB), would this layer's code need to change?** If the Service needs edits when switching REST → gRPC, it knows too much about HTTP. If the Service needs edits when switching databases, it knows too much about SQL.
2. **A DTO isn't just "a copy of the Entity" — it's an intentionally designed contract between your system and the outside world.** A DTO should only ever contain the fields the client actually needs. Separating Entity from DTO lets you change your internal data model freely, without breaking the API contract clients depend on.
