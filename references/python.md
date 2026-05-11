# Python Refactoring Reference

Language-specific patterns and examples for the code-refactor methodology.

## Type Safety Boundaries

Route all raw dict access through a single parsing boundary. Everything downstream uses typed results:

```python
from typing import TypedDict, Literal

class TextEntry(TypedDict):
    type: Literal["text"]
    content: str

class SheetEntry(TypedDict):
    type: Literal["sheet"]
    columns: dict[str, str]
    rows: list[dict[str, str]]

Entry = TextEntry | SheetEntry

def parse_entry(raw: dict) -> Entry:
    """THE boundary — all raw access goes through here."""
    if raw["type"] == "text":
        return TextEntry(type="text", content=str(raw["content"]))
    if raw["type"] == "sheet":
        return SheetEntry(type="sheet", columns=raw["columns"], rows=raw["rows"])
    raise ValueError(f"Unknown type: {raw['type']}")

# Downstream: use parse_entry result — no more raw dict.get()
```

## Code Smells & Fixes

### 1. Long Method

```python
# BEFORE: god function
def process_order(order_id: str) -> dict:
    order = db.orders.find_one({"_id": order_id})
    if not order: raise ValueError("Not found")
    if order["status"] != "pending": raise ValueError("Invalid status")
    total = sum(item["price"] * item["qty"] for item in order["items"])
    if order.get("coupon"): total *= 0.9
    db.orders.update_one({"_id": order_id}, {"$set": {"total": total}})
    email_service.send(order["user_email"], f"Order {order_id} processed")
    return {"order_id": order_id, "total": total}

# AFTER: focused sub-functions
def process_order(order_id: str) -> dict:
    order = _fetch_order(order_id)
    _validate_pending(order)
    total = _calculate_total(order)
    _save_total(order_id, total)
    _notify_user(order, total)
    return {"order_id": order_id, "total": total}
```

### 2. Duplicated Code

When function B is function A + one extra step, compose:

```python
# BEFORE
def push_sheet() -> SheetEntry:
    entry = {"columns": {}, "rows": {}}
    # ... setup logic ...
    return entry

def push_sheet_from_csv(csv: str) -> SheetEntry:
    entry = {"columns": {}, "rows": {}}  # duplicate setup
    # ... same setup logic ...
    parse_sheet_from_csv(csv, entry)
    return entry

# AFTER: compose
entry = push_sheet()
parse_sheet_from_csv(csv, entry)
```

### 3. Long Parameter List

```python
# BEFORE
def create_user(email: str, password: str, name: str, age: int, city: str, country: str, phone: str): ...

# AFTER: bundle related params
from dataclasses import dataclass

@dataclass
class UserData:
    email: str
    password: str
    name: str
    age: int | None = None
    address: Address | None = None
    phone: str | None = None

def create_user(data: UserData) -> User: ...
```

### 4. Nested Conditionals → Guard Clauses

```python
# BEFORE: arrow code
def process(order: Order | None) -> Result:
    if order:
        if order.user:
            if order.user.is_active:
                if order.total > 0:
                    return process_order(order)
                else: return Result.error("Invalid total")
            else: return Result.error("User inactive")
        else: return Result.error("No user")
    else: return Result.error("No order")

# AFTER: guard clauses
def process(order: Order | None) -> Result:
    if not order: return Result.error("No order")
    if not order.user: return Result.error("No user")
    if not order.user.is_active: return Result.error("User inactive")
    if order.total <= 0: return Result.error("Invalid total")
    return process_order(order)
```

### 5. Feature Envy

Method uses another object's data more than its own — move it:

```python
# BEFORE: Order reaches into User's fields
class Order:
    def calculate_discount(self, user: User) -> float:
        if user.membership_level == "gold": return self.total * 0.2
        if user.account_age_days > 365: return self.total * 0.1
        return 0

# AFTER: User owns discount rate logic
class User:
    def get_discount_rate(self) -> float:
        if self.membership_level == "gold": return 0.2
        if self.account_age_days > 365: return 0.1
        return 0

class Order:
    def calculate_discount(self, user: User) -> float:
        return self.total * user.get_discount_rate()
```

### 6. Primitive Obsession

```python
# BEFORE
def send_email(to: str, subject: str, body: str) -> None: ...

# AFTER: domain type with validation
import re

class Email:
    def __init__(self, value: str) -> None:
        if not re.match(r"^[^\s@]+@[^\s@]+\.[^\s@]+$", value):
            raise ValueError(f"Invalid email: {value}")
        self.value = value

def send_email(to: Email, subject: str, body: str) -> None: ...
```

### 7. Magic Numbers/Strings

```python
# BEFORE
if user.status == 2: ...
discount = total * 0.15

# AFTER
from enum import IntEnum

class UserStatus(IntEnum):
    ACTIVE = 1
    INACTIVE = 2
    SUSPENDED = 3

DISCOUNT_RATE_PREMIUM = 0.15
if user.status == UserStatus.INACTIVE: ...
discount = total * DISCOUNT_RATE_PREMIUM
```

### 8. Dead Code

Delete unused functions, imports, variables, and commented-out code. Common Python culprits: `__init__.py` re-export chains, `# TODO: remove after v2` blocks, `pass`-only method overrides.

### 9. Large Class

```python
# BEFORE: god object with 50+ methods
class UserManager:
    def create_user(self, data: dict) -> User: ...
    def send_email(self, to: str, subject: str, body: str) -> None: ...
    def generate_report(self, type: str, params: dict) -> Report: ...
    def handle_payment(self, amount: float, method: str) -> Payment: ...

# AFTER: single responsibility
class UserService:
    def create(self, data: dict) -> User: ...
class EmailService:
    def send(self, to: str, subject: str, body: str) -> None: ...
class ReportService:
    def generate(self, type: str, params: dict) -> Report: ...
class PaymentService:
    def process(self, amount: float, method: str) -> Payment: ...
```

### 10. Inappropriate Intimacy

```python
# BEFORE: digging into nested internals
class OrderProcessor:
    def process(self, order: Order) -> None:
        street = order.user.profile.address.street  # Law of Demeter violation
        config = order.repository.connection.config  # breaking encapsulation

# AFTER: ask, don't dig
class OrderProcessor:
    def process(self, order: Order) -> None:
        address = order.get_shipping_address()
        order.save()
```

## Inline Known-Behavior Calls

When a function branches internally but every caller knows the branch:

```python
# replace_current branches on type, but callers know the answer:
def replace_current(self, content: str, current: Entry | None) -> None:
    if current and current["type"] == "text":
        current["content"] = content  # in-place overwrite
    else:
        self.push_text(content)

# write() KNOWS it's text → always takes the if-branch
# append_text() KNOWS it's NOT text → always takes the else-branch
```

Both callers should inline their known branch. Keep `replace_current` only for callers that genuinely don't know the type.

## Design Patterns

### Strategy

```python
from abc import ABC, abstractmethod

class ShippingStrategy(ABC):
    @abstractmethod
    def calculate(self, order: Order) -> float: ...

class StandardShipping(ShippingStrategy):
    def calculate(self, order: Order) -> float:
        return 0.0 if order.total > 50 else 5.99

class ExpressShipping(ShippingStrategy):
    def calculate(self, order: Order) -> float:
        return 9.99 if order.total > 100 else 14.99

def calculate_shipping(order: Order, strategy: ShippingStrategy) -> float:
    return strategy.calculate(order)
```

### Chain of Responsibility

```python
from abc import ABC, abstractmethod

class Validator(ABC):
    def __init__(self) -> None:
        self._next: Validator | None = None

    def set_next(self, next_validator: Validator) -> Validator:
        self._next = next_validator
        return next_validator

    def validate(self, user: User) -> str | None:
        error = self._do_validate(user)
        if error: return error
        return self._next.validate(user) if self._next else None

    @abstractmethod
    def _do_validate(self, user: User) -> str | None: ...

class EmailRequired(Validator):
    def _do_validate(self, user: User) -> str | None:
        return "Email required" if not user.email else None

class AgeValidator(Validator):
    def _do_validate(self, user: User) -> str | None:
        return "Must be 18+" if user.age < 18 else None

# Build and use
validator = EmailRequired()
validator.set_next(AgeValidator())
error = validator.validate(user)
```

### Command

Replaces monolithic action handlers (giant `if action == "X"` blocks). Encapsulates each action as an object with `execute()` and optional `undo()`:

```python
from abc import ABC, abstractmethod

class Command(ABC):
    @abstractmethod
    def execute(self) -> None: ...
    def undo(self) -> None: ...  # optional

class CreateUserCommand(Command):
    def __init__(self, repo: UserRepo, data: dict) -> None:
        self.repo = repo
        self.data = data
        self._created_id: str | None = None

    def execute(self) -> None:
        user = self.repo.create(self.data)
        self._created_id = user.id

    def undo(self) -> None:
        if self._created_id:
            self.repo.delete(self._created_id)

class UpdateEmailCommand(Command):
    def __init__(self, repo: UserRepo, user_id: str, new_email: str) -> None:
        self.repo = repo
        self.user_id = user_id
        self.new_email = new_email
        self._old_email: str | None = None

    def execute(self) -> None:
        user = self.repo.get(self.user_id)
        self._old_email = user.email
        self.repo.update(self.user_id, email=self.new_email)

    def undo(self) -> None:
        if self._old_email:
            self.repo.update(self.user_id, email=self._old_email)

# Invoker
class ActionExecutor:
    def __init__(self) -> None:
        self._history: list[Command] = []

    def run(self, cmd: Command) -> None:
        cmd.execute()
        self._history.append(cmd)

    def undo_last(self) -> None:
        if self._history:
            self._history.pop().undo()
```

### State

Replaces complex status-flag conditionals. Each state is a class; the context delegates to the current state:

```python
from abc import ABC, abstractmethod

class OrderState(ABC):
    def __init__(self, order: Order) -> None:
        self.order = order

    @abstractmethod
    def submit(self) -> None: ...
    @abstractmethod
    def cancel(self) -> None: ...

class DraftState(OrderState):
    def submit(self) -> None:
        self.order.state = PendingState(self.order)
    def cancel(self) -> None:
        self.order.state = CancelledState(self.order)

class PendingState(OrderState):
    def submit(self) -> None:
        raise ValueError("Already submitted")
    def cancel(self) -> None:
        self.order.state = CancelledState(self.order)

class CancelledState(OrderState):
    def submit(self) -> None:
        raise ValueError("Cannot submit cancelled order")
    def cancel(self) -> None:
        raise ValueError("Already cancelled")

class Order:
    def __init__(self) -> None:
        self.state: OrderState = DraftState(self)
    def submit(self) -> None: self.state.submit()
    def cancel(self) -> None: self.state.cancel()
```

### Template Method

When multiple functions share the same algorithm skeleton but differ in steps:

```python
from abc import ABC, abstractmethod

class DataExporter(ABC):
    def export(self, filepath: str) -> None:
        data = self.fetch_data()          # subclass provides
        validated = self.validate(data)   # subclass provides (or use default)
        formatted = self.format(validated) # subclass provides
        self.write(filepath, formatted)    # shared

    @abstractmethod
    def fetch_data(self) -> list[dict]: ...
    @abstractmethod
    def format(self, data: list[dict]) -> str: ...

    def validate(self, data: list[dict]) -> list[dict]:
        return data  # default: no-op, subclasses may override

    def write(self, filepath: str, content: str) -> None:  # shared step
        with open(filepath, "w") as f:
            f.write(content)

class CsvExporter(DataExporter):
    def fetch_data(self) -> list[dict]: ...
    def format(self, data: list[dict]) -> str: ...

class JsonExporter(DataExporter):
    def fetch_data(self) -> list[dict]: ...
    def format(self, data: list[dict]) -> str: ...
    def validate(self, data: list[dict]) -> list[dict]:  # override default
        return [row for row in data if row.get("active")]
```

### Decorator

Adds behavior without subclass explosion. Each decorator wraps and extends:

```python
from abc import ABC, abstractmethod

class Notifier(ABC):
    @abstractmethod
    def send(self, message: str) -> None: ...

class EmailNotifier(Notifier):
    def send(self, message: str) -> None:
        print(f"Email: {message}")

class NotifierDecorator(Notifier):
    def __init__(self, wrapped: Notifier) -> None:
        self._wrapped = wrapped
    def send(self, message: str) -> None:
        self._wrapped.send(message)

class SmsDecorator(NotifierDecorator):
    def send(self, message: str) -> None:
        super().send(message)
        print(f"SMS: {message}")

class SlackDecorator(NotifierDecorator):
    def send(self, message: str) -> None:
        super().send(message)
        print(f"Slack: {message}")

# Compose: email + SMS + Slack, no subclass for each combination
notifier = SlackDecorator(SmsDecorator(EmailNotifier()))
notifier.send("Deploy complete")
```

## Simplification Examples

Common patterns where Python code can be simplified without changing behavior:

```python
# SIMPLIFY: Verbose dictionary building
# Before
result = {}
for item in items:
    result[item.id] = item.name
# After
result = {item.id: item.name for item in items}

# SIMPLIFY: Verbose list building
# Before
active_users = []
for user in users:
    if user.is_active:
        active_users.append(user)
# After
active_users = [user for user in users if user.is_active]

# SIMPLIFY: Redundant boolean return
# Before
def is_valid(data: str) -> bool:
    if len(data) > 0 and len(data) < 100:
        return True
    return False
# After
def is_valid(data: str) -> bool:
    return 0 < len(data) < 100

# SIMPLIFY: Unnecessary else after return
# Before
def get_status(order: Order) -> str:
    if order.is_paid:
        return "paid"
    else:
        return "pending"
# After
def get_status(order: Order) -> str:
    if order.is_paid:
        return "paid"
    return "pending"
```
