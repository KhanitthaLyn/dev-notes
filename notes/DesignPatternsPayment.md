# Question
How will you change PayPal payment gateway to Stripe payment gateway without changing the older code?
Which design pattern do you use?

# Answer

# The Big Picture & Analogy

Imagine a **wall outlet** and household **appliances** — the outlet doesn't care whether you plug in an iron, a fan, or a TV, as long as the plug on that appliance matches the **same standard** (two prongs or three).

If you want to switch from Iron Brand A to Iron Brand B, you don't need to **tear open the wall and rewire it** — you just unplug the old one and plug in the new one, because both use the **same plug standard.**

This is the core of the answer: we need to build a **"standard plug" (a common interface)** that the business logic depends on, while PayPal and Stripe are just **"different-brand appliances that fit the same socket."**

# Why Do We Need It? (The Problem It Solves)

Without this pattern, business logic (e.g. the Order Service) binds directly to PayPal:

```java
// ❌ Problem: Order Service knows PayPal directly
public class OrderService {
    private PayPalClient payPalClient = new PayPalClient();
    
    public void checkout(Order order) {
        payPalClient.sendPayment(order.getAmount(), order.getPayPalEmail());
        // ... other business logic
    }
}
```

The problem: switching to Stripe means changing **every place that directly calls PayPalClient** — risking missed spots, and making things **hard to test** since business logic is coupled to PayPal's concrete implementation rather than an interface, making mocking difficult.

This is exactly what the **Open/Closed Principle** addresses: code should be **"open to extension"** (adding a new provider) but **"closed to modification"** (no need to touch existing, working code).

# Core Logic & How It Works

**Answer: Combine the Strategy Pattern with the Adapter Pattern**

**Step 1: Define a common interface (the "contract")**
```java
public interface PaymentGateway {
    PaymentResult processPayment(BigDecimal amount, String currency, PaymentDetails details);
    boolean refund(String transactionId, BigDecimal amount);
}
```

**Step 2: Create an Adapter for each provider**
```java
// PayPal Adapter — "translates" PayPal's language into our common interface
public class PayPalAdapter implements PaymentGateway {
    private PayPalSDK payPalSDK; // PayPal-specific library
    
    @Override
    public PaymentResult processPayment(BigDecimal amount, String currency, PaymentDetails details) {
        // translate our input into what the PayPal SDK expects
        PayPalRequest request = PayPalRequest.builder()
            .amount(amount)
            .email(details.getPayPalEmail())
            .build();
        PayPalResponse response = payPalSDK.charge(request);
        return new PaymentResult(response.getTransactionId(), response.isSuccess());
    }
}

// Stripe Adapter — translates Stripe's language into the same interface
public class StripeAdapter implements PaymentGateway {
    private StripeClient stripeClient;
    
    @Override
    public PaymentResult processPayment(BigDecimal amount, String currency, PaymentDetails details) {
        ChargeCreateParams params = ChargeCreateParams.builder()
            .setAmount(amount.longValue())
            .setCurrency(currency)
            .setSource(details.getStripeToken())
            .build();
        Charge charge = stripeClient.charges().create(params);
        return new PaymentResult(charge.getId(), "succeeded".equals(charge.getStatus()));
    }
}
```

**Step 3: Business logic only calls the interface — never knows about PayPal or Stripe**
```java
public class OrderService {
    private final PaymentGateway paymentGateway; // depends on the interface, not an implementation
    
    public OrderService(PaymentGateway paymentGateway) { // dependency injection
        this.paymentGateway = paymentGateway;
    }
    
    public void checkout(Order order) {
        PaymentResult result = paymentGateway.processPayment(
            order.getAmount(), order.getCurrency(), order.getPaymentDetails()
        );
        // ... this code never changes, no matter how many times the provider switches
    }
}
```

**Step 4: Switch providers in exactly one place — the wiring/config layer**
```java
// change one line, from PayPal to Stripe
PaymentGateway gateway = new StripeAdapter(stripeClient); // was: new PayPalAdapter(...)
OrderService orderService = new OrderService(gateway);
```

# Trade-offs & When to Use

| Pattern | Role | When to use |
|---|---|---|
| **Adapter Pattern** | "Translates" an external provider's API into the interface we designed | Use when integrating with an external library/service whose API doesn't match what you need |
| **Strategy Pattern** | Swap algorithms/implementations at runtime through the same interface | Use when there are multiple "ways to do the same thing" that need to be interchangeable (payment provider, shipping provider, notification channel) |

**Trade-offs to know:**
- **Pros:** adding a new provider later (e.g. Omise, 2C2P) just means writing a new Adapter — no business logic changes needed. Testing gets much easier too (mock the `PaymentGateway` interface instead of mocking the real PayPal SDK)
- **Cons:** adds an abstraction layer, making the code slightly more indirect to trace through — for a small project genuinely certain it'll never switch providers, this might be over-engineering

**When NOT to use:** if you're 100% certain you'll only ever use one provider (rare in the payments world), building this abstraction upfront may not pay off. But in the payments domain specifically, the risk of switching providers (rising fees, expanding into a new market the current provider doesn't support) is high enough that this is closer to "necessary" than "nice to have."

# Real-World Scenario (Ecommerce Domain)

In our system, if we ever need to support both PayPal and Stripe simultaneously (letting customers choose), the Strategy Pattern handles it without any structural changes:

```java
@Service
public class PaymentGatewayFactory {
    private final Map<String, PaymentGateway> gateways;
    
    public PaymentGatewayFactory(List<PaymentGateway> gatewayList) {
        // Spring auto-injects every PaymentGateway implementation
        this.gateways = gatewayList.stream()
            .collect(Collectors.toMap(g -> g.getProviderName(), g -> g));
    }
    
    public PaymentGateway getGateway(String providerName) {
        return gateways.get(providerName); // "paypal" or "stripe"
    }
}

// In the checkout flow — pick the gateway based on the customer's runtime preference
PaymentGateway gateway = paymentGatewayFactory.getGateway(order.getPreferredProvider());
gateway.processPayment(...);
```

This is where the **Factory Pattern** (also on your checklist) complements the Strategy Pattern — the Factory's job is to "pick" the right Strategy at runtime.

# Lead's Key Takeaway

> **"Never let business logic know the 'brand' of an external service — it should only know the 'contract' (interface) we define ourselves."**
>
> A good Lead treats every external dependency (payment gateway, email provider, SMS provider, storage provider) as **"something that could always change"** and designs the abstraction upfront — not waiting until a real switch is forced and then refactoring under pressure. Refactoring after the system has grown large is far more painful than designing it right from the start.
