# The Big Picture & Analogy

**Bean Validation** (JSR 380 — Hibernate Validator being the main implementation) is a standard for **"declaring data-checking rules" directly via annotations on fields**, instead of writing manual if-else checks.

**Analogy:** Think of a **security guard checking ID badges at a building entrance**.

- Without a guard: everyone walks in, and problems only surface later, inside the building (e.g. discovering someone unauthorized sitting in a confidential meeting room) — by the time you notice, it's already too late.
- With a guard at the door (Bean Validation): everyone must show their badge before entering. The guard checks it immediately — if it's invalid, **they're stopped right at the door**, never allowed to let the problem drift into the system at all.

`@NotNull`, `@Email`, `@Min`, etc. are "rule labels" attached to each field, telling the guard "this field must have a value," "this field must be a valid email format." The guard (Spring) checks these automatically **before** the data ever reaches the business logic.

---

# Why Do We Need It?

**The problem before Bean Validation (manual validation):**

```java
public UserResponseDTO createUser(UserRequestDTO request) {
    // you have to hand-write if-else checks for every field
    if (request.getEmail() == null || request.getEmail().isEmpty()) {
        throw new IllegalArgumentException("Email is required");
    }
    if (!request.getEmail().contains("@")) {
        throw new IllegalArgumentException("Invalid email format");
    }
    if (request.getName() == null || request.getName().length() < 2) {
        throw new IllegalArgumentException("Name must be at least 2 characters");
    }
    if (request.getAge() < 0 || request.getAge() > 150) {
        throw new IllegalArgumentException("Invalid age");
    }
    // ... pages of validation logic before you even reach the real business logic

    // the actual business logic starts here (buried deep behind all the validation)
}
```

- **Validation logic gets tangled with business logic**, making it hard to tell which parts are rules and which parts are actual system logic.
- **You have to repeat it everywhere** the same input type is received — if 5 endpoints all accept `UserRequestDTO`, you'd copy the same validation logic 5 times (or extract a method, but still have to call it at every single spot).
- **Very easy to forget a check** — since the compiler never enforces it, if a new developer adds a field and forgets to validate it, bad data slips straight into the system.
- Error messages end up inconsistent, since everyone writes validation in their own style.

**Why Bean Validation solves this:**
- Rules are declared **once, in one place** (on the DTO) and automatically reused everywhere that DTO is used — write it once, works on every endpoint.
- It clearly separates "validation rules" from "business logic" — reading the DTO alone immediately tells you what constraints each field has, without digging through service logic.
- Spring integrates it automatically at the Controller level via `@Valid` — validation happens before your method code even runs.

---

# Core Logic & How It Works

## Commonly Used Annotations

```java
public class UserRequestDTO {

    @NotNull(message = "Name is required")          // cannot be null
    @Size(min = 2, max = 50, message = "Name must be 2-50 characters") // string length
    private String name;

    @NotBlank(message = "Email is required")         // no null, no empty, no whitespace-only
    @Email(message = "Invalid email format")          // must be a valid email format
    private String email;

    @Min(value = 0, message = "Age must be positive")
    @Max(value = 150, message = "Age must be realistic")
    private int age;

    @NotNull
    @Positive(message = "Price must be positive")     // must be > 0
    private Double price;

    @Pattern(regexp = "^[0-9]{10}$", message = "Phone must be 10 digits") // custom regex
    private String phoneNumber;

    @Past(message = "Birth date must be in the past")  // date must be in the past
    private LocalDate birthDate;
}
```

**An important distinction to note:**
- `@NotNull` — only rejects null (an empty string `""` still passes)
- `@NotEmpty` — rejects null and empty (but `"   "`, just whitespace, still passes)
- `@NotBlank` — rejects null, empty, and whitespace-only (strictest — best fit for most Strings)

## Connecting to the Controller via `@Valid`

```java
@RestController
@RequestMapping("/users")
public class UserController {
    private final UserService userService;

    public UserController(UserService userService) { this.userService = userService; }

    @PostMapping
    public ResponseEntity<UserResponseDTO> createUser(@Valid @RequestBody UserRequestDTO request) {
        // if the request fails validation → Spring throws MethodArgumentNotValidException
        // before any of this method's code even runs!
        return ResponseEntity.status(201).body(userService.createUser(request));
    }
}
```

**The mechanism behind it:** `@Valid` tells Spring to **validate this object first** before passing it into the method, using AOP to intercept the incoming request. If validation fails, it immediately throws `MethodArgumentNotValidException` — the method body never executes if validation fails (this connects directly to the `GlobalExceptionHandler` you already learned — you can catch this exception and convert it into a standard error response).

## Nested Object Validation

```java
public class OrderRequestDTO {
    @NotNull
    private Long userId;

    @NotEmpty(message = "Order must have at least one item")
    @Valid  // critical! tells Spring to also validate each CartItemDTO inside, not just that the list isn't empty
    private List<CartItemDTO> items;
}

public class CartItemDTO {
    @NotNull
    private Long productId;

    @Min(value = 1, message = "Quantity must be at least 1")
    private int quantity;
}
```

**Watch out:** if you forget `@Valid` on `List<CartItemDTO> items`, the list itself is only checked with `@NotEmpty`, but **each individual `CartItemDTO` inside never gets validated at all** — this is an extremely common mistake.

## Custom Validation (when built-in annotations aren't enough)

```java
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = ValidSkuValidator.class)
public @interface ValidSku {
    String message() default "Invalid SKU format";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class ValidSkuValidator implements ConstraintValidator<ValidSku, String> {
    @Override
    public boolean isValid(String sku, ConstraintValidatorContext context) {
        return sku != null && sku.matches("^SKU-[A-Z0-9]{8}$"); // specific business rule
    }
}

// usage
public class ProductRequestDTO {
    @ValidSku
    private String sku;
}
```

---

# Trade-offs & When to Use

**When to use Bean Validation:**
- Validation that's a **"structural rule"** — e.g. a field must never be null, must match a valid email format, must fall within a length range — annotations are a great fit here, since these are fixed rules that don't depend on context.
- Should be applied to every DTO that receives client input, always (defense in depth — never trust external input without checking it).

**When NOT to use / when business logic is a better fit:**
- Validation that **depends on complex business rules or requires a database query** — e.g. "this email must not already exist in the system" — that belongs in the Service layer, not an annotation (a custom validator could technically do it, but that couples the validator to the database, which it shouldn't be).
- Validation that depends on multiple fields at once (cross-field validation, e.g. "startDate must come before endDate") — achievable with a class-level custom annotation, but often reads more clearly as plain business logic in the Service.

**Trade-offs:**
- **You gain:** concise, declaratively-placed validation logic that's easy to read, reusable across every endpoint sharing the same DTO, and dramatically less boilerplate.
- **You pay:** slightly harder debugging when validation gets complex (tracing through multiple stacked annotations), and you need to write custom validators when built-in annotations don't cover your case.

---

# Real-World Scenario / Mini Example

Adding full validation to `UserDTO`, with tests for invalid input:

```java
// ===== UserRequestDTO with validation =====
public class UserRequestDTO {

    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 50, message = "Name must be between 2 and 50 characters")
    private String name;

    @NotBlank(message = "Email is required")
    @Email(message = "Email format is invalid")
    private String email;

    @NotNull(message = "Age is required")
    @Min(value = 18, message = "User must be at least 18 years old")
    @Max(value = 120, message = "Age must be realistic")
    private Integer age;

    @Pattern(regexp = "^[0-9]{9,10}$", message = "Phone number must be 9-10 digits")
    private String phoneNumber; // optional field — no @NotBlank, since it's allowed to be empty

    // getters/setters
}

// ===== Controller using @Valid =====
@RestController
@RequestMapping("/users")
public class UserController {
    private final UserService userService;

    public UserController(UserService userService) { this.userService = userService; }

    @PostMapping
    public ResponseEntity<UserResponseDTO> createUser(@Valid @RequestBody UserRequestDTO request) {
        return ResponseEntity.status(201).body(userService.createUser(request));
    }
}

// ===== GlobalExceptionHandler catching validation errors (building on the previous topic) =====
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException e) {
        // gather error messages from every field that failed validation
        Map<String, String> fieldErrors = new HashMap<>();
        e.getBindingResult().getFieldErrors().forEach(error ->
            fieldErrors.put(error.getField(), error.getDefaultMessage())
        );

        ErrorResponse response = new ErrorResponse(
            "VALIDATION_ERROR",
            "Invalid input data",
            fieldErrors // returns per-field detail back to the client
        );
        return ResponseEntity.badRequest().body(response);
    }
}
```

**Testing with invalid input:**

```json
// Request with several violations at once
POST /users
{
  "name": "A",
  "email": "not-an-email",
  "age": 15,
  "phoneNumber": "123"
}
```

**Response received (before business logic is ever reached):**
```json
{
  "errorCode": "VALIDATION_ERROR",
  "message": "Invalid input data",
  "fieldErrors": {
    "name": "Name must be between 2 and 50 characters",
    "email": "Email format is invalid",
    "age": "User must be at least 18 years old",
    "phoneNumber": "Phone number must be 9-10 digits"
  }
}
```

**Unit test for validation:**

```java
@SpringBootTest
public class UserRequestDTOValidationTest {
    private Validator validator;

    @BeforeEach
    void setup() {
        ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
        validator = factory.getValidator();
    }

    @Test
    void shouldFailWhenEmailIsInvalid() {
        UserRequestDTO dto = new UserRequestDTO();
        dto.setName("John Doe");
        dto.setEmail("invalid-email"); // malformed
        dto.setAge(25);

        Set<ConstraintViolation<UserRequestDTO>> violations = validator.validate(dto);

        assertFalse(violations.isEmpty());
        assertTrue(violations.stream()
            .anyMatch(v -> v.getPropertyPath().toString().equals("email")));
    }

    @Test
    void shouldPassWithValidData() {
        UserRequestDTO dto = new UserRequestDTO();
        dto.setName("John Doe");
        dto.setEmail("john@example.com");
        dto.setAge(25);

        Set<ConstraintViolation<UserRequestDTO>> violations = validator.validate(dto);

        assertTrue(violations.isEmpty()); // no errors at all
    }
}
```

**Notice that** `userService.createUser()` **is never called at all** if the request fails validation — bad data is stopped at the "front door," never reaching the inner business logic.

---

# Lead's Key Takeaway

1. **Good validation means "rejecting bad data as early as possible" (fail fast).** The closer you catch it to the point data enters the system (at the DTO layer, before business logic), the more you prevent corrupted data from reaching the database or triggering side effects that are hard to roll back.
2. **Clearly separate "structural validation" (done via annotations) from "business rule validation" (done in the Service layer).** Rules that don't depend on the database or other system state should always live as annotations. Rules that require a query or depend on business context belong in the Service. This separation means anyone reading the code instantly knows where each rule lives and where to go fix it.
