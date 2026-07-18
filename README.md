# Spring Dependency Injection (DI) — Complete Notes

A practical guide to understanding `@Autowired`, injection types, and circular dependencies in Spring.

---

## Table of Contents
- [What is Dependency Injection](#what-is-dependency-injection)
- [Types of Injection](#types-of-injection)
  - [1. Field Injection](#1-field-injection)
  - [2. Setter Injection](#2-setter-injection)
  - [3. Constructor Injection](#3-constructor-injection-recommended)
- [Real-Time Example: E-commerce Order Flow](#real-time-example-e-commerce-order-flow)
- [Handling Multiple Implementations](#handling-multiple-implementations)
- [Circular Dependencies](#circular-dependencies)
- [Optional Dependencies & Runtime Swapping](#optional-dependencies--runtime-swapping)
- [Other Concepts to Learn](#other-concepts-to-learn)
- [Quick Reference Table](#quick-reference-table)

---

## What is Dependency Injection

Dependency Injection (DI) is a design pattern where objects **don't create their own dependencies**. Instead, the Spring **IoC (Inversion of Control) container** creates and "injects" them automatically.

> `@Autowired` tells Spring: *"find a matching bean and wire it in here."*

---

## Types of Injection

### 1. Field Injection

```java
@Service
public class OrderService {
    @Autowired
    private PaymentRepository paymentRepository;
}
```

**When to use:** Quick prototypes, demos, learning. ⚠️ Avoid in production.

**Why avoid it:**
- Field can't be `final` → object can exist in a half-broken state
- Hard to unit test (needs reflection or full Spring context to inject mocks)
- Hides dependencies — can't tell what a class needs without reading the whole body
- Circular dependencies get silently "resolved" via reflection tricks instead of failing fast

**How it works internally:**
1. Spring creates the object using its default no-arg constructor
2. *After* creation, Spring uses **reflection** to directly set the field — bypassing getters/setters
3. Works regardless of access modifier (`private`, package-private, etc.)

---

### 2. Setter Injection

```java
@Service
public class OrderService {
    private PaymentRepository paymentRepository;

    @Autowired
    public void setPaymentRepository(PaymentRepository paymentRepository) {
        this.paymentRepository = paymentRepository;
    }
}
```

**When to use:**
- Optional dependencies (`@Autowired(required = false)`)
- Rare cases where a dependency needs to be *changed* after object creation

**Downside:** Object can temporarily exist without its dependency set → risk of `NullPointerException` if used too early.

---

### 3. Constructor Injection (✅ Recommended)

```java
@Service
public class OrderService {
    private final PaymentRepository paymentRepository;
    private final NotificationService notificationService;

    // @Autowired is optional here if there's only one constructor (Spring 4.3+)
    public OrderService(PaymentRepository paymentRepository,
                         NotificationService notificationService) {
        this.paymentRepository = paymentRepository;
        this.notificationService = notificationService;
    }
}
```

**Why it's best:**
- Fields can be `final` → immutable, thread-safe
- Object is never in an incomplete state
- Easy to unit test — just call `new OrderService(mockRepo, mockNotif)`
- Circular dependencies fail **at startup** (fail-fast) instead of causing runtime bugs
- Dependencies are explicit and visible in the constructor signature

**With Lombok (removes boilerplate):**
```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final PaymentRepository paymentRepository;
    private final NotificationService notificationService;
    // no manual constructor needed!
}
```

---

## Real-Time Example: E-commerce Order Flow

```java
// Repository layer
@Repository
public interface PaymentRepository extends JpaRepository<Payment, Long> {
}

// Service layer
@Service
public class NotificationService {
    public void sendOrderConfirmation(String email) {
        System.out.println("Email sent to: " + email);
    }
}

// Business/Service layer — calls repository + another service
@Service
public class OrderService {
    private final PaymentRepository paymentRepository;
    private final NotificationService notificationService;

    public OrderService(PaymentRepository paymentRepository,
                         NotificationService notificationService) {
        this.paymentRepository = paymentRepository;
        this.notificationService = notificationService;
    }

    public void placeOrder(Order order) {
        paymentRepository.save(order.getPayment());
        notificationService.sendOrderConfirmation(order.getCustomerEmail());
    }
}

// Controller layer — calls the service
@RestController
@RequestMapping("/orders")
public class OrderController {
    private final OrderService orderService;

    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @PostMapping
    public ResponseEntity<String> createOrder(@RequestBody Order order) {
        orderService.placeOrder(order);
        return ResponseEntity.ok("Order placed!");
    }
}
```

Spring's IoC container wires `PaymentRepository` → `OrderService` → `OrderController` automatically, with no manual `new` calls anywhere.

---

## Handling Multiple Implementations

Use `@Qualifier` or `@Primary` when multiple beans of the same type exist.

```java
public interface PaymentGateway {
    void pay(double amount);
}

@Service
public class RazorpayGateway implements PaymentGateway {
    public void pay(double amount) { /* ... */ }
}

@Service
public class StripeGateway implements PaymentGateway {
    public void pay(double amount) { /* ... */ }
}

@Service
public class OrderService {
    private final PaymentGateway paymentGateway;

    public OrderService(@Qualifier("razorpayGateway") PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }
}
```

Alternatively, mark a default implementation with `@Primary` on the class.

**Choosing dynamically per request (thread-safe, no shared mutable state):**
```java
@Service
public class PaymentProcessor {
    private final Map<String, PaymentGateway> gateways;

    public PaymentProcessor(Map<String, PaymentGateway> gateways) {
        this.gateways = gateways; // Spring injects ALL PaymentGateway beans, keyed by bean name
    }

    public void processPayment(String gatewayName, double amount) {
        gateways.get(gatewayName).pay(amount);
    }
}
```

---

## Circular Dependencies

A circular dependency happens when two beans depend on each other:
```
A needs B  →  B needs A
```

### With Field Injection — app starts, but hides the bug

```java
@Service
public class TeacherService {
    @Autowired
    private StudentService studentService;
}

@Service
public class StudentService {
    @Autowired
    private TeacherService teacherService;
}
```

**What happens:** Spring resolves this using an internal "early bean reference" (three-level cache) trick:
1. Spring starts creating `TeacherService` (empty object, field not set yet)
2. Registers this half-built object in an internal cache
3. Starts creating `StudentService`, which needs `TeacherService` → gets the half-built reference
4. `StudentService` is now fully built
5. Spring goes back and finishes `TeacherService`, setting its field to the completed `StudentService`

⚠️ It "works," but the app quietly tolerates a design smell — bugs from this can surface later (e.g., in `@PostConstruct` methods running on a half-built object), and only in edge cases.

### With Constructor Injection — app fails immediately

```java
@Service
public class TeacherService {
    private final StudentService studentService;
    public TeacherService(StudentService studentService) {
        this.studentService = studentService;
    }
}

@Service
public class StudentService {
    private final TeacherService teacherService;
    public StudentService(TeacherService teacherService) {
        this.teacherService = teacherService;
    }
}
```

**Result:**
```
***************************
APPLICATION FAILED TO START
***************************
The dependencies of some of the beans in the application context form a cycle:
┌─────┐
|  teacherService defined in TeacherService.class
↑     ↓
|  studentService defined in StudentService.class
└─────┘
```

There's no "half-built object" trick possible — constructor arguments must be fully ready *before* the object is created.

### Why fail-fast is the good outcome

| | Field Injection | Constructor Injection |
|---|---|---|
| App starts? | ✅ Yes | ❌ No |
| Bug caught immediately? | ❌ No — hidden | ✅ Yes — at startup |
| Real problem fixed? | Masked | Forced to fix |

### How to actually fix it

**1. Refactor — extract shared logic into a third class**
```java
@Service
public class HomeworkService {
    public void process() {
        // shared logic that both services needed
    }
}
```

**2. Use `@Lazy` (quick fix, not ideal — usually still a design smell)**
```java
public TeacherService(@Lazy StudentService studentService) {
    this.studentService = studentService;
}
```
This injects a proxy and resolves the real bean only when actually used.

> **Takeaway:** A circular dependency is almost always a design smell — two classes are too tightly coupled. Constructor injection's fail-fast behavior forces you to notice and fix it immediately.

---

## Optional Dependencies & Runtime Swapping

### Case 1: Optional dependency (app works fine without it)

```java
@Service
public class OrderService {
    private final PaymentRepository paymentRepository; // mandatory
    private AnalyticsService analyticsService;          // optional

    public OrderService(PaymentRepository paymentRepository) {
        this.paymentRepository = paymentRepository;
    }

    @Autowired(required = false)
    public void setAnalyticsService(AnalyticsService analyticsService) {
        this.analyticsService = analyticsService;
    }

    public void placeOrder(Order order) {
        paymentRepository.save(order.getPayment());
        if (analyticsService != null) {
            analyticsService.trackOrder(order);
        }
    }
}
```
If `AnalyticsService` bean doesn't exist, the app **still starts** and just skips tracking.

### Case 2: Changing a dependency after creation (rare, use with caution)

```java
@Service
public class PaymentProcessor {
    private PaymentGateway paymentGateway;

    @Autowired
    public void setPaymentGateway(PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }

    public void processPayment(double amount) {
        paymentGateway.pay(amount);
    }
}
```

⚠️ **Thread-safety warning:** Spring beans are **singletons by default** — one instance shared across all requests. Mutating shared state at runtime (`setPaymentGateway()` called mid-flight) can cause one user's request to be processed with the wrong gateway under concurrent load.

**Safer alternative:** Inject a `Map<String, PaymentGateway>` once at startup and choose per-request (see [Handling Multiple Implementations](#handling-multiple-implementations)) instead of mutating shared object state.

---

## Other Concepts to Learn

| Concept | Why it matters |
|---|---|
| `@Component`, `@Service`, `@Repository`, `@Controller` | All are stereotypes of `@Component` — Spring scans and registers them as beans |
| Bean Scopes (`singleton`, `prototype`, `request`, `session`) | Controls how many instances of a bean exist |
| `@Configuration` + `@Bean` | Manually define beans (e.g., for third-party classes you can't annotate) |
| `@Lazy` | Delay bean creation until first use |
| `@Value` | Inject values from `application.properties` / `.yml` |
| `@Autowired(required = false)` | Makes a dependency optional |
| `@Inject` (JSR-330) | Java standard equivalent of `@Autowired`, framework-agnostic |
| `@Primary` | Marks a default bean when multiple candidates exist |
| `@Qualifier` | Explicitly picks which bean to inject by name |
| Lombok `@RequiredArgsConstructor` | Auto-generates constructor for all `final` fields |

---

## Quick Reference Table

| Scenario | Real Example | Recommended Injection |
|---|---|---|
| Mandatory dependency | Payment repo, DB connection | **Constructor** |
| Optional dependency | Analytics, feature-flagged services | Setter with `required = false` |
| Multiple implementations, chosen per request | Multiple payment gateways | Constructor + `Map<String, Bean>` |
| Need to mutate dependency at runtime | Rare — usually a design smell | Setter (use cautiously, watch thread-safety) |
| Quick prototype / demo | Learning exercises | Field (not for production) |

### Golden Rule
> **Always use constructor injection for mandatory dependencies.**
> Use `@Autowired` field/setter injection only for legacy code, quick tests, or genuinely optional dependencies.

---

*Notes compiled for learning Spring Dependency Injection concepts.*
