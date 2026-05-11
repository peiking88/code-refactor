# Java Refactoring Reference

Language-specific patterns and examples for the code-refactor methodology.

## Type Safety Boundaries

Route all raw/untyped access through a single parsing boundary. Use sealed classes with pattern matching so downstream code benefits from exhaustive type checking:

```java
// BEFORE: scattered instanceof + casting
void write(String content) {
    if (entry instanceof TextEntry) {
        ((TextEntry) entry).getContent();  // cast everywhere
    } else if (entry instanceof SheetEntry) {
        ((SheetEntry) entry).getColumns(); // another cast
    }
}

// AFTER: sealed hierarchy with single factory method
public sealed interface Entry permits TextEntry, SheetEntry {
    static Entry from(Map<String, Object> raw) {
        return switch ((String) raw.get("type")) {
            case "text" -> new TextEntry((String) raw.get("content"));
            case "sheet" -> new SheetEntry(
                (Map<String, String>) raw.get("columns"),
                (List<Map<String, String>>) raw.get("rows"));
            default -> throw new IllegalArgumentException("Unknown type: " + raw.get("type"));
        };
    }
}

record TextEntry(String content) implements Entry {}
record SheetEntry(Map<String, String> columns, List<Map<String, String>> rows) implements Entry {}

// Downstream: exhaustive pattern matching, no casts needed
switch (entry) {
    case TextEntry(var content) -> writeText(content);
    case SheetEntry(var cols, var rows) -> writeSheet(cols, rows);
}
```

## Code Smells & Fixes

### 1. Long Method

```java
// BEFORE: god method
public OrderResult processOrder(String orderId) {
    var order = db.orders().find(Filters.eq("_id", orderId)).first();
    if (order == null) throw new IllegalStateException("Not found");
    if (!"pending".equals(order.getStatus())) throw new IllegalStateException("Invalid status");
    double total = order.getItems().stream().mapToDouble(i -> i.price() * i.qty()).sum();
    if (order.getCoupon() != null) total *= 0.9;
    db.orders().updateOne(Filters.eq("_id", orderId), Updates.set("total", total));
    emailService.send(order.getUserEmail(), "Order " + orderId + " processed");
    return new OrderResult(orderId, total);
}

// AFTER: focused sub-methods
public OrderResult processOrder(String orderId) {
    var order = fetchOrder(orderId);
    validatePending(order);
    var total = calculateTotal(order);
    saveTotal(orderId, total);
    notifyUser(order, total);
    return new OrderResult(orderId, total);
}
```

### 2. Duplicated Code

When method B is method A + one extra step, compose:

```java
// BEFORE: duplicated setup logic
SheetEntry pushSheet() {
    var entry = new SheetEntry(new HashMap<>(), new ArrayList<>());
    // ... setup logic ...
    return entry;
}
SheetEntry pushSheetFromCsv(String csv) {
    var entry = new SheetEntry(new HashMap<>(), new ArrayList<>()); // duplicate
    // ... same setup logic ...
    parseSheetFromCsv(csv, entry);
    return entry;
}

// AFTER: compose
var entry = pushSheet();
parseSheetFromCsv(csv, entry);
```

### 3. Long Parameter List

```java
// BEFORE
void createUser(String email, String password, String name,
                int age, String city, String country, String phone) { }

// AFTER: parameter object via record
record UserData(String email, String password, String name,
                int age, Address address, String phone) {}
void createUser(UserData data) { }
```

### 4. Nested Conditionals → Guard Clauses

```java
// BEFORE: arrow code
Result process(Order order) {
    if (order != null) {
        if (order.user() != null) {
            if (order.user().isActive()) {
                if (order.total() > 0) return processOrder(order);
                else return Result.error("Invalid total");
            } else return Result.error("User inactive");
        } else return Result.error("No user");
    } else return Result.error("No order");
}

// AFTER: guard clauses
Result process(Order order) {
    if (order == null) return Result.error("No order");
    if (order.user() == null) return Result.error("No user");
    if (!order.user().isActive()) return Result.error("User inactive");
    if (order.total() <= 0) return Result.error("Invalid total");
    return processOrder(order);
}
```

### 5. Feature Envy

```java
// BEFORE: Order reaches into User's fields
class Order {
    double calculateDiscount(User user) {
        if (user.membershipLevel().equals("gold")) return total * 0.2;
        if (user.accountAgeDays() > 365) return total * 0.1;
        return 0;
    }
}

// AFTER: User owns discount rate logic
class User {
    double getDiscountRate() {
        if (membershipLevel.equals("gold")) return 0.2;
        if (accountAgeDays > 365) return 0.1;
        return 0;
    }
}
class Order {
    double calculateDiscount(User user) {
        return total * user.getDiscountRate();
    }
}
```

### 6. Primitive Obsession

```java
// BEFORE
void sendEmail(String to, String subject, String body) { }

// AFTER: domain type with validation
record Email(String value) {
    Email {
        if (!value.matches("^[^\\s@]+@[^\\s@]+\\.[^\\s@]+$"))
            throw new IllegalArgumentException("Invalid email: " + value);
    }
}
void sendEmail(Email to, String subject, String body) { }
```

### 7. Magic Numbers/Strings

```java
// BEFORE
if (user.getStatus() == 2) { ... }

// AFTER
enum UserStatus { ACTIVE(1), INACTIVE(2), SUSPENDED(3); }
private static final double DISCOUNT_RATE_PREMIUM = 0.15;
if (user.getStatus() == UserStatus.INACTIVE) { ... }
```

### 8. Dead Code

Delete unused methods, imports, and commented-out blocks. Common Java culprits: empty `@Override` stubs, `@Deprecated` methods with zero callers, `// FIXME: remove in v3` blocks.

### 9. Large Class

```java
// BEFORE: god class with 50+ methods
class UserManager {
    User createUser(UserData data) { ... }
    void sendEmail(String to, String subject, String body) { ... }
    Report generateReport(String type, Map<String, Object> params) { ... }
    Payment handlePayment(double amount, String method) { ... }
}

// AFTER: single responsibility
class UserService { User create(UserData data) { ... } }
class EmailService { void send(String to, String subject, String body) { ... } }
class ReportService { Report generate(String type, Map<String, Object> params) { ... } }
class PaymentService { Payment process(double amount, String method) { ... } }
```

### 10. Inappropriate Intimacy

```java
// BEFORE: chained access through nested objects
class OrderProcessor {
    void process(Order order) {
        String street = order.getUser().getProfile().getAddress().getStreet();  // Law of Demeter
    }
}

// AFTER: single public method on owning object
class Order {
    String getShippingAddress() { return user.getProfile().getAddress().getStreet(); }
}
class OrderProcessor {
    void process(Order order) {
        String street = order.getShippingAddress();
    }
}
```

## Exception Audit Examples

Three categories of exception misuse with Java-specific Before/After.

### Category 1: Missing Exception Handling

```java
// BEFORE: file read with no error boundary — IOException propagates unhandled
String loadConfig(String path) {
    return Files.readString(Path.of(path));
}

// AFTER: handle the real failure modes
String loadConfig(String path) {
    try {
        return Files.readString(Path.of(path));
    } catch (NoSuchFileException e) {
        logger.warn("Config not found at {}, using defaults", path);
        return DEFAULT_CONFIG;
    } catch (IOException e) {
        throw new ConfigException("Failed to read config: " + path, e);
    }
}
```

### Category 2: Unnecessary Exception Handling

```java
// BEFORE: exception-driven control flow (slow, unclear intent)
String getUserName(String userId) {
    try {
        return db.get(userId).getName();
    } catch (NullPointerException e) {
        return "Unknown";
    }
}

// AFTER: null check — fast, clear intent
String getUserName(String userId) {
    var user = db.get(userId);
    return user != null ? user.getName() : "Unknown";
}
```

```java
// BEFORE: bare catch in a batch loop — swallows errors silently
List<String> processBatch(List<String> items) {
    var results = new ArrayList<String>();
    for (String item : items) {
        try {
            results.add(transform(item));
        } catch (Exception e) {
            // silently skipped — data loss with no signal
        }
    }
    return results;
}

// AFTER: collect errors, report them
List<String> processBatch(List<String> items) {
    var results = new ArrayList<String>();
    var errors = new ArrayList<String>();
    for (String item : items) {
        try {
            results.add(transform(item));
        } catch (IllegalArgumentException e) {
            logger.error("Failed to transform {}: {}", item, e.getMessage());
            errors.add(item);
        }
    }
    if (!errors.isEmpty()) {
        logger.warn("Batch processing had {} failures out of {} items", errors.size(), items.size());
    }
    return results;
}
```

### Category 3: Wrong Exception Type

```java
// BEFORE: generic RuntimeException for argument validation
void setAge(int age) {
    if (age < 0) throw new RuntimeException("Age cannot be negative");
}

// AFTER: IllegalArgumentException — the standard type for invalid arguments
void setAge(int age) {
    if (age < 0) throw new IllegalArgumentException("Age cannot be negative: " + age);
}
```

```java
// BEFORE: lost cause chain — root cause destroyed
List<User> loadUsers(String path) {
    try {
        return parseUsers(Files.readString(Path.of(path)));
    } catch (IOException e) {
        throw new AppException("Failed to load users");  // original cause lost
    }
}

// AFTER: chain the original exception
List<User> loadUsers(String path) {
    try {
        return parseUsers(Files.readString(Path.of(path)));
    } catch (IOException e) {
        throw new AppException("Failed to load users from " + path, e);
    }
}
```

## Parameter Objects & Type Derivation

```java
// BEFORE: fields duplicated across types
record SheetBinding(Map<String, String> columns, Map<String, String> rows) {}
record SheetEntry(Map<String, String> columns, Map<String, String> rows, String type, Instant createdAt) {}

// AFTER: composition via record nesting
record SheetEntry(SheetBinding binding, Instant createdAt) implements Entry {
    @Override public String type() { return "sheet"; }
}
```

## Collapsing Duplicate Branches

When a switch has branches doing the same thing with different inputs:

```java
// BEFORE: two branches both "flatten to string and push"
switch (entry) {
    case RichtextEntry r -> pushText(richtextToPlaintext(r.content()) + text);
    case SheetEntry s    -> pushText(sheetToCsv(s) + text);
}

// AFTER: read() already handles the conversion
pushText(read() + text);
```

## Design Patterns

### Strategy

```java
interface ShippingStrategy { double calculate(Order order); }

class StandardShipping implements ShippingStrategy {
    public double calculate(Order o) { return o.total() > 50 ? 0 : 5.99; }
}
class ExpressShipping implements ShippingStrategy {
    public double calculate(Order o) { return o.total() > 100 ? 9.99 : 14.99; }
}

double calculateShipping(Order order, ShippingStrategy strategy) {
    return strategy.calculate(order);
}
```

### Chain of Responsibility (Functional)

```java
interface ValidationRule { Optional<String> check(User user); }

List<ValidationRule> rules = List.of(
    u -> u.email() == null ? Optional.of("Email required") : Optional.empty(),
    u -> u.age() < 18 ? Optional.of("Must be 18+") : Optional.empty(),
    u -> u.country().equals("blocked") ? Optional.of("Country not supported") : Optional.empty()
);

Optional<String> error = rules.stream()
    .map(r -> r.check(user))
    .flatMap(Optional::stream)
    .findFirst();
```

### Command

Replaces monolithic action handlers. Encapsulates each action as an object with `execute()` and optional `undo()`:

```java
interface Command {
    void execute();
    default void undo() {}
}

class CreateUserCommand implements Command {
    private final UserRepo repo;
    private final UserData data;
    private String createdId;

    CreateUserCommand(UserRepo repo, UserData data) {
        this.repo = repo;
        this.data = data;
    }

    public void execute() {
        var user = repo.create(data);
        this.createdId = user.id();
    }

    public void undo() {
        if (createdId != null) repo.delete(createdId);
    }
}

class ActionExecutor {
    private final List<Command> history = new ArrayList<>();

    void run(Command cmd) {
        cmd.execute();
        history.add(cmd);
    }

    void undoLast() {
        if (!history.isEmpty()) history.removeLast().undo();
    }
}
```

### State

Replaces complex status-flag conditionals. Each state is a class; context delegates:

```java
interface OrderState {
    void submit(Order order);
    void cancel(Order order);
}

class DraftState implements OrderState {
    public void submit(Order order) { order.setState(new PendingState()); }
    public void cancel(Order order) { order.setState(new CancelledState()); }
}

class PendingState implements OrderState {
    public void submit(Order order) { throw new IllegalStateException("Already submitted"); }
    public void cancel(Order order) { order.setState(new CancelledState()); }
}

class Order {
    private OrderState state = new DraftState();

    void setState(OrderState s) { this.state = s; }
    void submit() { state.submit(this); }
    void cancel() { state.cancel(this); }
}
```

### Template Method

When functions share algorithm skeleton but differ in steps:

```java
abstract class DataExporter {
    public final void export(String filepath) throws IOException {
        var data = fetchData();
        var validated = validate(data);
        var formatted = format(validated);
        Files.writeString(Path.of(filepath), formatted);
    }

    protected abstract List<Map<String, Object>> fetchData();
    protected abstract String format(List<Map<String, Object>> data);

    protected List<Map<String, Object>> validate(List<Map<String, Object>> data) {
        return data; // default no-op, subclasses may override
    }
}

class CsvExporter extends DataExporter {
    protected List<Map<String, Object>> fetchData() { ... }
    protected String format(List<Map<String, Object>> data) { ... }
}
```

### Decorator

Adds behavior without subclass explosion. Each decorator wraps and extends:

```java
interface Notifier { void send(String message); }

class EmailNotifier implements Notifier {
    public void send(String message) { System.out.println("Email: " + message); }
}

abstract class NotifierDecorator implements Notifier {
    protected final Notifier wrapped;
    NotifierDecorator(Notifier wrapped) { this.wrapped = wrapped; }
    public void send(String message) { wrapped.send(message); }
}

class SmsDecorator extends NotifierDecorator {
    SmsDecorator(Notifier wrapped) { super(wrapped); }
    public void send(String message) {
        super.send(message);
        System.out.println("SMS: " + message);
    }
}

class SlackDecorator extends NotifierDecorator {
    SlackDecorator(Notifier wrapped) { super(wrapped); }
    public void send(String message) {
        super.send(message);
        System.out.println("Slack: " + message);
    }
}

// Compose: any combination without subclass explosion
var notifier = new SlackDecorator(new SmsDecorator(new EmailNotifier()));
notifier.send("Deploy complete");
```
