# TypeScript Refactoring Reference

Language-specific patterns and examples for the code-refactor methodology.

## Type Safety Boundaries

Route all raw/untyped access through a single parsing boundary. Use discriminated unions so downstream code benefits from type narrowing:

```typescript
// BEFORE: scattered raw access across methods
get currentType(): ContentType {
    return this.entry.get('type') as ContentType;  // raw .get() + cast
}
write(text: string): void {
    const columns = this.entry.get('columns') as Map<string, unknown>;  // another raw access
}

// AFTER: single parsing boundary, everything downstream uses typed result
type Entry = TextEntry | SheetEntry | RichtextEntry;

function parseEntry(raw: unknown): Entry {
    if (raw instanceof XmlText) return { type: 'text', content: raw };
    if (raw instanceof Map && raw.get('type') === 'sheet')
        return { type: 'sheet', columns: raw.get('columns'), rows: raw.get('rows') };
    if (raw instanceof XmlFragment)
        return { type: 'richtext', content: raw };
    throw new Error(`Unknown entry type`);
}

// Downstream uses discriminated union with type narrowing
function write(entry: Entry, text: string): void {
    switch (entry.type) {
        case 'text': return writeText(entry.content, text);
        case 'sheet': return writeSheet(entry.columns, text);
        case 'richtext': return writeRichtext(entry.content, text);
    }
}
```

## Code Smells & Fixes

### 1. Long Method

```typescript
// BEFORE: god function
async function processOrder(orderId: string): Promise<OrderResult> {
    const order = await db.orders.findOne({ _id: orderId });
    if (!order) throw new Error('Not found');
    if (order.status !== 'pending') throw new Error('Invalid status');
    const total = order.items.reduce((sum, item) => sum + item.price * item.qty, 0);
    const finalTotal = order.coupon ? total * 0.9 : total;
    await db.orders.updateOne({ _id: orderId }, { $set: { total: finalTotal } });
    await emailService.send(order.userEmail, `Order ${orderId} processed`);
    return { orderId, total: finalTotal };
}

// AFTER: focused sub-functions
async function processOrder(orderId: string): Promise<OrderResult> {
    const order = await fetchOrder(orderId);
    validatePending(order);
    const total = calculateTotal(order);
    await saveTotal(orderId, total);
    await notifyUser(order, total);
    return { orderId, total };
}
```

### 2. Duplicated Code

When function B is function A + one extra step, compose:

```typescript
// BEFORE: duplicated setup logic
function pushSheet(): SheetEntry {
    const entry = { columns: new Map(), rows: new Map() };
    // ... setup logic ...
    return entry;
}
function pushSheetFromCsv(csv: string): SheetEntry {
    const entry = { columns: new Map(), rows: new Map() };  // duplicate
    // ... same setup logic ...
    parseSheetFromCsv(csv, entry);
    return entry;
}

// AFTER: compose
const entry = pushSheet();
parseSheetFromCsv(csv, entry);
```

### 3. Long Parameter List

```typescript
// BEFORE
function createUser(
    email: string, password: string, name: string,
    age: number, city: string, country: string, phone: string
): User { ... }

// AFTER: bundle into parameter object
interface UserData {
    email: string;
    password: string;
    name: string;
    age?: number;
    address?: Address;
    phone?: string;
}
function createUser(data: UserData): User { ... }
```

### 4. Nested Conditionals → Guard Clauses

```typescript
// BEFORE: arrow code
function process(order: Order | null): Result {
    if (order) {
        if (order.user) {
            if (order.user.isActive) {
                if (order.total > 0) return processOrder(order);
                else return { error: 'Invalid total' };
            } else return { error: 'User inactive' };
        } else return { error: 'No user' };
    } else return { error: 'No order' };
}

// AFTER: guard clauses
function process(order: Order | null): Result {
    if (!order) return { error: 'No order' };
    if (!order.user) return { error: 'No user' };
    if (!order.user.isActive) return { error: 'User inactive' };
    if (order.total <= 0) return { error: 'Invalid total' };
    return processOrder(order);
}
```

### 5. Feature Envy

```typescript
// BEFORE: Order reaches into User's fields
class Order {
    calculateDiscount(user: User): number {
        if (user.membershipLevel === 'gold') return this.total * 0.2;
        if (user.accountAgeDays > 365) return this.total * 0.1;
        return 0;
    }
}

// AFTER: User owns discount rate logic
class User {
    getDiscountRate(): number {
        if (this.membershipLevel === 'gold') return 0.2;
        if (this.accountAgeDays > 365) return 0.1;
        return 0;
    }
}
class Order {
    calculateDiscount(user: User): number {
        return this.total * user.getDiscountRate();
    }
}
```

### 6. Primitive Obsession

```typescript
// BEFORE
function sendEmail(to: string, subject: string, body: string): void { ... }

// AFTER: branded type with validation
type Email = string & { readonly __brand: 'Email' };
function createEmail(value: string): Email {
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value))
        throw new Error(`Invalid email: ${value}`);
    return value as Email;
}
function sendEmail(to: Email, subject: string, body: string): void { ... }
```

### 7. Magic Numbers/Strings

```typescript
// BEFORE
if (user.status === 2) { ... }

// AFTER
const UserStatus = { ACTIVE: 1, INACTIVE: 2, SUSPENDED: 3 } as const;
type UserStatus = (typeof UserStatus)[keyof typeof UserStatus];
if (user.status === UserStatus.INACTIVE) { ... }
```

### 8. Dead Code

Delete unused imports, exports, and commented-out blocks. Common TS culprits: `// TODO: remove after v2`, conditional exports only used in tests, `@deprecated` functions with zero callers.

### 9. Large Class

```typescript
// BEFORE: god class
class UserManager {
    createUser(data: UserData): User { ... }
    sendEmail(to: string, subject: string, body: string): void { ... }
    generateReport(type: string, params: ReportParams): Report { ... }
    handlePayment(amount: number, method: string): Payment { ... }
}

// AFTER: single responsibility
class UserService {
    create(data: UserData): User { ... }
}
class EmailService {
    send(to: string, subject: string, body: string): void { ... }
}
class ReportService {
    generate(type: string, params: ReportParams): Report { ... }
}
class PaymentService {
    process(amount: number, method: string): Payment { ... }
}
```

### 10. Inappropriate Intimacy

```typescript
// BEFORE: chained property access through multiple objects
class OrderProcessor {
    process(order: Order): void {
        const street = order.user.profile.address.street;  // Law of Demeter violation
    }
}

// AFTER: single public method on the owning object
class Order {
    getShippingAddress(): string { return this.user.profile.address.street; }
}
class OrderProcessor {
    process(order: Order): void {
        const street = order.getShippingAddress();
    }
}
```

## Parameter Objects & Type Derivation

When 2+ params always travel together:

```typescript
// BEFORE: destructure fields from a type that already bundles them
function serializeSheet(columns: Map<string, string>, rows: Map<string, unknown>): string { ... }

// AFTER: accept the bundling type directly
interface SheetBinding {
    columns: Map<string, string>;
    rows: Map<string, unknown>;
}
function serializeSheet(entry: SheetBinding): string { ... }
```

Derive types instead of duplicating fields:

```typescript
// BEFORE: columns/rows duplicated in both types
type SheetBinding = { columns: Map<string, string>; rows: Map<string, unknown> };
type SheetEntry = { type: 'sheet'; columns: Map<string, string>; rows: Map<string, unknown>; createdAt: number };

// AFTER: intersection
type SheetEntry = SheetBinding & { type: 'sheet'; createdAt: number };
```

## Collapsing Duplicate Branches

When a switch has branches doing the same thing with different inputs, collapse via shared method:

```typescript
// BEFORE: two branches both "flatten to string and push"
switch (entry.type) {
    case 'richtext': pushText(richtextToPlaintext(entry.content) + text); break;
    case 'sheet': pushText(sheetToCsv(entry) + text); break;
}

// AFTER: read() already does "flatten to string"
pushText(this.read() + text);
```

## Design Patterns

### Strategy

```typescript
interface ShippingStrategy {
    calculate(order: Order): number;
}
class StandardShipping implements ShippingStrategy {
    calculate(order: Order): number { return order.total > 50 ? 0 : 5.99; }
}
class ExpressShipping implements ShippingStrategy {
    calculate(order: Order): number { return order.total > 100 ? 9.99 : 14.99; }
}

function calculateShipping(order: Order, strategy: ShippingStrategy): number {
    return strategy.calculate(order);
}
```

### Chain of Responsibility

```typescript
interface ValidationRule { check(user: User): string | null; }

class EmailRequired implements ValidationRule {
    private next: ValidationRule | null = null;
    setNext(rule: ValidationRule): ValidationRule { this.next = rule; return rule; }
    check(user: User): string | null {
        if (!user.email) return "Email required";
        return this.next?.check(user) ?? null;
    }
}
class AgeValidator implements ValidationRule {
    private next: ValidationRule | null = null;
    setNext(rule: ValidationRule): ValidationRule { this.next = rule; return rule; }
    check(user: User): string | null {
        if (user.age < 18) return "Must be 18+";
        return this.next?.check(user) ?? null;
    }
}

const validator = new EmailRequired();
validator.setNext(new AgeValidator());
const error = validator.check(user);
```

### Command

Replaces monolithic action handlers. Encapsulates each action as an object with `execute()` and optional `undo()`:

```typescript
interface Command {
    execute(): void;
    undo?(): void;
}

class CreateUserCommand implements Command {
    private createdId: string | null = null;
    constructor(private repo: UserRepo, private data: UserData) {}
    execute(): void {
        const user = this.repo.create(this.data);
        this.createdId = user.id;
    }
    undo(): void {
        if (this.createdId) this.repo.delete(this.createdId);
    }
}

// Invoker tracks history for undo
class ActionExecutor {
    private history: Command[] = [];
    run(cmd: Command): void {
        cmd.execute();
        this.history.push(cmd);
    }
    undoLast(): void {
        const cmd = this.history.pop();
        cmd?.undo?.();
    }
}
```

### State

Replaces complex status-flag conditionals. Each state is a class; context delegates to current state:

```typescript
interface OrderState {
    submit(): void;
    cancel(): void;
}

class DraftState implements OrderState {
    constructor(private order: Order) {}
    submit(): void { this.order.state = new PendingState(this.order); }
    cancel(): void { this.order.state = new CancelledState(this.order); }
}

class PendingState implements OrderState {
    constructor(private order: Order) {}
    submit(): void { throw new Error("Already submitted"); }
    cancel(): void { this.order.state = new CancelledState(this.order); }
}

class Order {
    state: OrderState = new DraftState(this);
    submit(): void { this.state.submit(); }
    cancel(): void { this.state.cancel(); }
}
```

### Template Method

When functions share algorithm skeleton but differ in steps:

```typescript
abstract class DataExporter {
    export(filepath: string): void {
        const data = this.fetchData();
        const validated = this.validate(data);
        const formatted = this.format(validated);
        this.write(filepath, formatted);
    }
    protected abstract fetchData(): Record<string, unknown>[];
    protected abstract format(data: Record<string, unknown>[]): string;
    protected validate(data: Record<string, unknown>[]): Record<string, unknown>[] {
        return data; // default no-op
    }
    private write(filepath: string, content: string): void {
        Deno.writeTextFileSync(filepath, content);
    }
}

class CsvExporter extends DataExporter {
    protected fetchData(): Record<string, unknown>[] { ... }
    protected format(data: Record<string, unknown>[]): string { ... }
}
```

### Decorator

Adds behavior without subclass explosion. Each decorator wraps and extends:

```typescript
interface Notifier { send(message: string): void; }

class EmailNotifier implements Notifier {
    send(message: string): void { console.log(`Email: ${message}`); }
}

class NotifierDecorator implements Notifier {
    constructor(protected wrapped: Notifier) {}
    send(message: string): void { this.wrapped.send(message); }
}

class SmsDecorator extends NotifierDecorator {
    send(message: string): void {
        super.send(message);
        console.log(`SMS: ${message}`);
    }
}

class SlackDecorator extends NotifierDecorator {
    send(message: string): void {
        super.send(message);
        console.log(`Slack: ${message}`);
    }
}

// Compose: any combination without subclass explosion
const notifier = new SlackDecorator(new SmsDecorator(new EmailNotifier()));
notifier.send("Deploy complete");
```
