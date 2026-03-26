# Java Clean Code Principles (2026)

A practical guide to writing clean, maintainable Java — covering foundational practices and modern Java features. Based on a review of common best practices with additions and modifications for modern Java development.

---

## 1. Naming: Be Precise, Follow Conventions

Consistent, descriptive naming is foundational. But the real trap isn't `xyz` — it's names that *seem* descriptive but are actually vague.

**Avoid vague "manager" and "data" names:**

```java
// Bad — what does "process" mean? What "data"?
public class DataManager {
    public void processData(List<String> data) { ... }
}

// Good — says exactly what it does
public class InvoiceExporter {
    public void exportToCsv(List<Invoice> invoices) { ... }
}
```

**Booleans should read like a yes/no question:**

```java
// Bad
boolean status;
boolean flag;

// Good
boolean isActive;
boolean hasExpired;
boolean canRetry;
```

**Avoid abbreviations unless they're universally understood:**

```java
// Bad
int empCnt;
String txnId;
LocalDate regDt;

// Good
int employeeCount;
String transactionId;
LocalDate registrationDate;
```

### Java naming conventions (camelCase & friends)

Java has well-established casing rules. Breaking them confuses every developer who reads your code and breaks IDE tooling.

```java
// Classes & interfaces — PascalCase (UpperCamelCase)
public class OrderService { }
public interface PaymentGateway { }
public record TradeExecution(...) { }
public sealed interface TradeResult { }

// Methods & variables — camelCase (lowerCamelCase)
public void calculateTotalPrice() { }
int itemCount = 0;
String customerEmail;

// Constants — SCREAMING_SNAKE_CASE
private static final int MAX_RETRY_COUNT = 3;
private static final String DEFAULT_CURRENCY = "USD";

// Packages — all lowercase, no underscores
// com.helcik.invoice.service
// com.helcik.trade.execution

// Enums — PascalCase type, SCREAMING_SNAKE_CASE values
public enum OrderStatus {
    PENDING, PROCESSING, COMPLETED, CANCELLED
}

// Type parameters — single uppercase letter (or short word)
public class Repository<T, ID> { }
public interface Converter<S, T> { }
```

### Common naming pitfalls

- **Inconsistent acronyms** — use `HttpClient` and `parseJson`, not `HTTPClient` or `parseJSON`. Treat acronyms as words in camelCase.
- **Hungarian notation** — don't prefix types like `strName` or `iCount`. The type system already tells you this.
- **Meaningless suffixes** — `UserData`, `UserInfo`, `UserObject` all say the same thing. Pick one and be consistent.
- **Verb vs noun confusion** — classes are nouns (`InvoiceExporter`), methods are verbs (`exportInvoice`). Don't name a class `ExportInvoices` or a method `invoice`.

> **Rule of thumb:** If you have to read the method body to understand what a name means, the name is wrong. Good names make the code read like a story — `orderService.validateAndSubmit(order)` tells you everything without opening the file.

---

## 2. Short Methods & Single Responsibility

Every method should do one thing. If it doesn't fit on your screen (~20-25 lines), it's probably doing too much.

**Before — one method doing three jobs:**

```java
public void handleOrder(Order order) {
    if (order == null || order.getItems().isEmpty()) {
        throw new IllegalArgumentException("Invalid order");
    }

    double total = 0;
    for (Item item : order.getItems()) {
        total += item.getPrice() * item.getQuantity();
    }

    if (order.getCustomer().isPremium()) {
        total *= 0.9;
    }

    Payment payment = new Payment(order.getCustomer(), total);
    paymentService.processPayment(payment);
}
```

**After — each concern is isolated and testable:**

```java
public void handleOrder(Order order) {
    validateOrder(order);
    double total = calculateOrderTotal(order);
    executePayment(order.getCustomer(), total);
}

private void validateOrder(Order order) {
    if (order == null || order.getItems().isEmpty()) {
        throw new IllegalArgumentException("Invalid order");
    }
}

private double calculateOrderTotal(Order order) {
    double total = order.getItems().stream()
        .mapToDouble(item -> item.getPrice() * item.getQuantity())
        .sum();
    return order.getCustomer().isPremium() ? total * 0.9 : total;
}

private void executePayment(Customer customer, double total) {
    paymentService.processPayment(customer, total);
}
```

---

## 3. Comments: Explain *Why*, Not *What* — Or Better Yet, Refactor

If you need a comment to explain what code does, that's a signal to rename or extract a method. Reserve comments for things code structurally *cannot* express.

### Comments that should be refactored away

```java
// Bad — the comment IS the method name you should have written
// Check if the user is eligible for a discount
if (user.getAge() > 18 && user.getOrders().size() > 5 && user.isPremiumMember()) {
    applyDiscount(order);
}

// Good — extract the condition; the method name replaces the comment
if (user.isEligibleForDiscount()) {
    applyDiscount(order);
}
```

```java
// Bad — narrates the obvious
// Create a new user
User user = new User();
// Set the name
user.setName("Arek");
// Save the user
userRepository.save(user);
```

### Comments that ARE valuable: explaining *why*

**Business context that code can't express:**

```java
// FX rates from the ECB feed are delayed by ~15 minutes,
// so we add a buffer margin to protect against stale quotes
private static final double FX_RATE_BUFFER = 0.02;
```

**Non-obvious constraints or workarounds:**

```java
// The payment gateway rejects amounts below 0.50 USD due to processing fees,
// so we bundle micro-transactions into a single charge
if (pendingAmount < MINIMUM_CHARGE_THRESHOLD) {
    batchQueue.add(transaction);
    return;
}
```

**Deliberate tradeoffs:**

```java
// Using ArrayList instead of LinkedList here despite frequent insertions
// because profiling showed cache locality gives us 3x throughput on our
// typical dataset sizes (< 10k elements)
private final List<Trade> activeTrades = new ArrayList<>();
```

### Records as self-documenting code

```java
// Old style — needs Javadoc to explain what this class represents
/**
 * Represents the result of a discount calculation.
 * Contains the original price, discount rate applied,
 * and the final discounted price.
 */
public class DiscountResult {
    private final double originalPrice;
    private final double discountRate;
    private final double finalPrice;
    // constructor, getters, equals, hashCode, toString... 40+ lines
}

// Modern — the record IS the documentation
public record DiscountResult(double originalPrice, double discountRate, double finalPrice) {}
```

### Rule of thumb

Before writing a comment, ask: *"Can I rename something, extract a method, or restructure the code so this comment becomes redundant?"* If yes, do that. Comments are for: business context, external constraints, historical reasons, and performance justifications.

---

## 4. Eliminate Magic Numbers and Strings

Unnamed constants make code unreadable and fragile.

```java
// Bad — what does 0.1 represent?
public double calculateDiscount(double price) {
    return price * 0.1;
}

// Good — named constant
private static final double STANDARD_DISCOUNT_RATE = 0.10;

public double calculateDiscount(double price) {
    return price * STANDARD_DISCOUNT_RATE;
}
```

**Use enums for related values, sealed interfaces when behavior varies:**

```java
// Enum for simple value sets
public enum OrderStatus {
    PENDING, PROCESSING, COMPLETED, CANCELLED
}

// Sealed interface when each type has different behavior (Java 17+)
public sealed interface PaymentMethod permits CreditCard, BankTransfer, Crypto {
    BigDecimal calculateFee(BigDecimal amount);
}

public record CreditCard(String last4) implements PaymentMethod {
    public BigDecimal calculateFee(BigDecimal amount) {
        return amount.multiply(new BigDecimal("0.029"));
    }
}
```

---

## 5. Exception Handling: Be Specific, Fail Fast

Catch specific exceptions. Never catch generic `Exception`. Provide context that helps debugging.

```java
// Bad — swallows everything, no useful info
try {
    userService.registerUser(user);
} catch (Exception e) {
    System.out.println("An error occurred.");
}

// Good — specific catches with appropriate responses
try {
    userService.registerUser(user);
} catch (UserAlreadyExistsException e) {
    log.warn("Duplicate registration attempt for email: {}", user.getEmail());
    return ResponseEntity.conflict().body("User already exists");
} catch (DatabaseConnectionException e) {
    log.error("DB failure during registration: {}", e.getMessage(), e);
    throw new ServiceUnavailableException("Cannot process request", e);
}
```

**Additional principles:**

- Always use `try-with-resources` for `Closeable`/`AutoCloseable` resources
- In Spring Boot, centralize with `@ControllerAdvice` instead of scattering catch blocks
- Never use exceptions for control flow (e.g., catching `NoSuchElementException` instead of calling `isEmpty()`)
- Fail fast — validate inputs at the boundary, not deep in the call stack

---

## 6. Streams & Lambdas: Readable Over Clever

Method references are preferred when they map 1:1 to a method. But the bigger concern is complex stream chains that should be broken up.

```java
// Fine — clean and readable
List<String> activeUserNames = users.stream()
    .filter(User::isEnabled)
    .map(User::getName)
    .sorted()
    .toList(); // Java 16+ — prefer over Collectors.toList()

// Bad — too much happening in one chain
var result = orders.stream()
    .filter(o -> o.getStatus() == COMPLETED && o.getTotal().compareTo(threshold) > 0)
    .flatMap(o -> o.getItems().stream())
    .collect(Collectors.groupingBy(
        Item::getCategory,
        Collectors.summarizingDouble(i -> i.getPrice().doubleValue())));

// Better — extract the complex parts
Predicate<Order> isHighValueCompleted = order ->
    order.getStatus() == COMPLETED && order.getTotal().compareTo(threshold) > 0;

var result = orders.stream()
    .filter(isHighValueCompleted)
    .flatMap(order -> order.getItems().stream())
    .collect(Collectors.groupingBy(
        Item::getCategory,
        Collectors.summarizingDouble(item -> item.getPrice().doubleValue())));
```

**When NOT to use streams:** If you need indexes, early `break`, mutable accumulators, or checked exceptions — a `for` loop is cleaner.

---

## 7. Testing: Untested Code Isn't Clean Code

Tests are not a separate concern from clean code — they enable it. You can't safely refactor without them.

**Use descriptive test names:**

```java
// Outdated convention
@Test
public void testComputeDiscount() { ... }

// Modern — reads like a specification
@Test
@DisplayName("Should apply 10% discount for premium customers")
void shouldApply10PercentDiscount_whenCustomerIsPremium() {
    var calculator = new DiscountCalculator();
    double result = calculator.computeDiscount(100.0, 0.10);
    assertThat(result).isCloseTo(10.0, within(0.001)); // AssertJ
}
```

**Use parameterized tests to cover edge cases:**

```java
@ParameterizedTest
@CsvSource({
    "100.0, 0.10, 10.0",
    "50.0,  0.00, 0.0",
    "200.0, 0.25, 50.0",
    "0.0,   0.10, 0.0"
})
void shouldComputeDiscountCorrectly(double price, double rate, double expected) {
    assertThat(calculator.computeDiscount(price, rate)).isCloseTo(expected, within(0.001));
}
```

**Follow Arrange-Act-Assert:**

```java
@Test
void shouldRejectExpiredCoupon() {
    // Arrange
    var coupon = new Coupon("SAVE10", LocalDate.of(2025, 1, 1));
    var order = new Order(BigDecimal.valueOf(100));

    // Act & Assert
    assertThatThrownBy(() -> order.applyCoupon(coupon))
        .isInstanceOf(ExpiredCouponException.class)
        .hasMessageContaining("SAVE10");
}
```

---

## 8. Simplicity Over Cleverness (YAGNI)

Don't build abstractions you don't need yet.

```java
// Over-engineered — nobody needs this
Function<Double, Function<Integer, Double>> calculate = p -> q -> p * q;

// Just write this
public double computeOrderAmount(double price, int quantity) {
    return price * quantity;
}
```

**More realistic over-engineering to avoid:**

- Creating `AbstractDiscountStrategyFactory` when you have one discount type
- Building a plugin system when you have two implementations
- Using the Strategy pattern for a two-branch `if`/`else`

Extract an abstraction when you have **three** concrete cases, not before.

---

## 9. Dependency Injection & Composition

Inject dependencies through constructors. Code to interfaces. Prefer composition over inheritance.

```java
// Bad — tight coupling, untestable
public class OrderService {
    private PaymentService paymentService = new PaymentService();
}

// Good — injectable, testable, swappable
public class OrderService {
    private final PaymentGateway paymentGateway;

    public OrderService(PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }
}
```

**Since Spring 4.3+**, single-constructor classes don't need `@Autowired`. The principle applies even without Spring — it's about designing classes that receive their dependencies rather than creating them.

**Composition over inheritance:**

```java
// Fragile inheritance hierarchy
class Animal { }
class FlyingAnimal extends Animal { void fly() { } }
class SwimmingFlyingAnimal extends FlyingAnimal { void swim() { } } // uh oh

// Composition via interfaces with default methods
interface Flyer { default void fly() { /* ... */ } }
interface Swimmer { default void swim() { /* ... */ } }

class Duck implements Flyer, Swimmer { }
```

---

## 10. Refactor Continuously (With Tests)

Refactoring without tests is just moving code around and hoping. The cycle is: write tests → refactor → verify tests still pass.

**Common safe refactorings:**

- Extract Method — pull a block into a named method
- Introduce Parameter Object — replace 3+ related parameters with a record
- Replace conditional with polymorphism — use sealed interfaces + pattern matching
- Inline trivial methods — if a method just delegates, remove the indirection

---

## 11. Embrace Modern Java Features

These aren't optional syntax sugar — they eliminate entire categories of code smells.

### Records (Java 16+)

Immutable data carriers with free `equals`, `hashCode`, `toString`.

```java
// Replaces 40+ lines of POJO boilerplate
public record TradeExecution(
    String symbol,
    BigDecimal price,
    int quantity,
    Instant executedAt
) {
    // Compact constructor for validation
    public TradeExecution {
        Objects.requireNonNull(symbol, "Symbol required");
        if (quantity <= 0) throw new IllegalArgumentException("Quantity must be positive");
    }
}
```

### Sealed Classes (Java 17+)

Exhaustive type hierarchies the compiler can verify.

```java
public sealed interface TradeResult permits Success, Failure, Pending {
    record Success(TradeExecution execution) implements TradeResult {}
    record Failure(String reason, Exception cause) implements TradeResult {}
    record Pending(String orderId) implements TradeResult {}
}
```

### Pattern Matching in Switch (Java 21+)

```java
// Compiler ensures all cases are covered for sealed types
public String describe(TradeResult result) {
    return switch (result) {
        case Success s   -> "Executed %s shares of %s".formatted(s.execution().quantity(), s.execution().symbol());
        case Failure f   -> "Failed: " + f.reason();
        case Pending p   -> "Pending order: " + p.orderId();
    };
}
```

### Text Blocks (Java 15+)

```java
// Clean embedded SQL, JSON, HTML
String query = """
    SELECT symbol, price, quantity
    FROM trades
    WHERE executed_at > :since
      AND status = 'COMPLETED'
    ORDER BY executed_at DESC
    """;
```

### `var` for Local Type Inference (Java 10+)

```java
// Good — type is obvious from the right side
var users = userRepository.findAllActive();
var mapper = new ObjectMapper();

// Bad — type is not obvious, use explicit type
var result = service.process(data); // What type is result?
```

---

## 12. Immutability by Default

Mutable state is the root of countless bugs. Default to immutable and opt into mutability only when needed.

```java
// Use records for data
public record Money(BigDecimal amount, Currency currency) {}

// Use unmodifiable collections
List<String> symbols = List.of("AAPL", "GOOG", "MSFT");
Map<String, Integer> counts = Map.of("trades", 42, "errors", 0);

// Mark fields final
private final List<Order> orders;
```

---

## 13. Optional: Use Idiomatically

`Optional` is for return types that may have no value. Not for fields, not for parameters.

```java
// Bad
Optional<User> user = findUser(id);
if (user.isPresent()) {
    return user.get().getName();
}

// Good — chain operations
return findUser(id)
    .map(User::getName)
    .orElse("Unknown");

// Good — throw when absence is a bug
User user = findUser(id)
    .orElseThrow(() -> new UserNotFoundException("No user with id: " + id));

// Never do these:
private Optional<String> name;          // Don't use as field
public void setName(Optional<String> n) // Don't use as parameter
```

---

## 14. Structured Logging Over System.out

`System.out.println` has no place in production code. Use structured logging.

```java
// Bad — no level, no context, no searchability
System.out.println("Order processed successfully!");
System.out.println("Error: " + e.getMessage());

// Good — structured, leveled, searchable
log.info("Order processed: orderId={}, customerId={}, total={}",
    order.getId(), order.getCustomerId(), total);

log.error("Payment failed: orderId={}, gateway={}", orderId, gateway, exception);
```

Use MDC (Mapped Diagnostic Context) for request-scoped context like correlation IDs, user IDs, and session identifiers.

---

## 15. Null Safety Strategy

Design APIs that never return null. Use annotations to make intent explicit.

```java
// API contract: this never returns null
public @NonNull List<Trade> findTrades(@NonNull String symbol) {
    // return empty list, not null
    return trades.getOrDefault(symbol, List.of());
}

// Use Optional for genuinely optional return values
public Optional<User> findByEmail(String email) { ... }

// Validate at boundaries
public void createOrder(@NonNull Customer customer, @NonNull List<Item> items) {
    Objects.requireNonNull(customer, "Customer must not be null");
    Objects.requireNonNull(items, "Items must not be null");
    // safe to proceed without null checks deeper in the call stack
}
```

---

## Quick Reference

| Principle | One-liner |
|---|---|
| Naming | Be Precise, Follow Conventions |
| Short methods | One job, ~20 lines max, testable in isolation |
| Comments | Explain *why*, not *what* — or refactor instead |
| No magic values | Named constants, enums, sealed types |
| Exception handling | Specific catches, fail fast, centralize in Spring |
| Streams | Readable over clever; extract complex predicates |
| Testing | Untested code isn't clean; AAA pattern; parameterized tests |
| YAGNI | Extract abstractions at three cases, not one |
| DI & composition | Constructor injection; interfaces over inheritance |
| Refactor | Continuously, but only with test coverage |
| Modern Java | Records, sealed classes, pattern matching, text blocks |
| Immutability | Default to final, records, unmodifiable collections |
| Optional | Return types only; map/flatMap/orElseThrow |
| Logging | SLF4J + MDC, never System.out |
| Null safety | @NonNull, empty collections, validate at boundaries |
