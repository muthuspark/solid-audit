# SOLID Violation Examples

Reference before/after code samples for all five SOLID principles.
Used by `/solid-audit` to calibrate violation detection.

---

## S — Single Responsibility Principle

### Python — Before (Violation)

```python
class UserService:
    def parse_csv(self, path: str) -> list[dict]:
        with open(path) as f:
            return [row for row in csv.DictReader(f)]

    def save_to_db(self, users: list[dict]) -> None:
        for user in users:
            db.execute("INSERT INTO users VALUES (?)", [user["email"]])

    def send_welcome_email(self, email: str) -> None:
        smtp.sendmail("noreply@app.com", email, "Welcome!")
```

**Violation**: `UserService` handles CSV parsing, database persistence, and email sending — three unrelated responsibilities.

### Python — After (Fixed)

```python
class CsvParser:
    def parse(self, path: str) -> list[dict]:
        with open(path) as f:
            return [row for row in csv.DictReader(f)]

class UserRepository:
    def save(self, users: list[dict]) -> None:
        for user in users:
            db.execute("INSERT INTO users VALUES (?)", [user["email"]])

class EmailService:
    def send_welcome(self, email: str) -> None:
        smtp.sendmail("noreply@app.com", email, "Welcome!")
```

### TypeScript — Before (Violation)

```typescript
class OrderService {
  processOrder(order: Order): void {
    // validate
    if (!order.items.length) throw new Error("Empty order");
    // persist
    db.query("INSERT INTO orders ...", [order]);
    // notify
    emailClient.send(order.userEmail, "Order confirmed");
  }
}
```

### TypeScript — After (Fixed)

```typescript
class OrderValidator {
  validate(order: Order): void {
    if (!order.items.length) throw new Error("Empty order");
  }
}

class OrderRepository {
  save(order: Order): void {
    db.query("INSERT INTO orders ...", [order]);
  }
}

class OrderNotifier {
  notify(order: Order): void {
    emailClient.send(order.userEmail, "Order confirmed");
  }
}
```

---

## O — Open/Closed Principle

### Python — Before (Violation)

```python
def export_report(data: list, format: str) -> str:
    if format == "pdf":
        return render_pdf(data)
    elif format == "csv":
        return render_csv(data)
    elif format == "json":
        return render_json(data)
    else:
        raise ValueError(f"Unknown format: {format}")
```

**Violation**: Adding a new format requires modifying `export_report` — the function is not closed for modification.

### Python — After (Fixed)

```python
from typing import Protocol

class ReportRenderer(Protocol):
    def render(self, data: list) -> str: ...

_renderers: dict[str, ReportRenderer] = {
    "pdf": PdfRenderer(),
    "csv": CsvRenderer(),
    "json": JsonRenderer(),
}

def export_report(data: list, format: str) -> str:
    if format not in _renderers:
        raise ValueError(f"Unknown format: {format}")
    return _renderers[format].render(data)
```

### TypeScript — Before (Violation)

```typescript
function getDiscount(user: User): number {
  if (user.type === "premium") return 0.2;
  if (user.type === "student") return 0.1;
  if (user.type === "employee") return 0.3;
  return 0;
}
```

### TypeScript — After (Fixed)

```typescript
interface DiscountStrategy {
  getDiscount(): number;
}

const discountStrategies: Record<string, DiscountStrategy> = {
  premium: { getDiscount: () => 0.2 },
  student: { getDiscount: () => 0.1 },
  employee: { getDiscount: () => 0.3 },
};

function getDiscount(user: User): number {
  return discountStrategies[user.type]?.getDiscount() ?? 0;
}
```

---

## L — Liskov Substitution Principle

### Python — Before (Violation)

```python
class Bird:
    def fly(self) -> str:
        return "flying"

class Penguin(Bird):
    def fly(self) -> str:
        raise NotImplementedError("Penguins cannot fly")
```

**Violation**: `Penguin` cannot be substituted for `Bird` — calling `fly()` raises an exception the base class contract doesn't declare.

### Python — After (Fixed)

```python
class Bird:
    def move(self) -> str:
        return "moving"

class FlyingBird(Bird):
    def fly(self) -> str:
        return "flying"

class Penguin(Bird):
    def swim(self) -> str:
        return "swimming"
```

### TypeScript — Before (Violation)

```typescript
class Rectangle {
  setWidth(w: number): void { this.width = w; }
  setHeight(h: number): void { this.height = h; }
  area(): number { return this.width * this.height; }
}

class Square extends Rectangle {
  setWidth(w: number): void { this.width = w; this.height = w; }
  setHeight(h: number): void { this.width = h; this.height = h; }
}
```

### TypeScript — After (Fixed)

```typescript
interface Shape {
  area(): number;
}

class Rectangle implements Shape {
  constructor(private width: number, private height: number) {}
  area(): number { return this.width * this.height; }
}

class Square implements Shape {
  constructor(private side: number) {}
  area(): number { return this.side * this.side; }
}
```

---

## I — Interface Segregation Principle

### Python — Before (Violation)

```python
from abc import ABC, abstractmethod

class WorkerInterface(ABC):
    @abstractmethod
    def work(self) -> None: ...
    @abstractmethod
    def eat(self) -> None: ...
    @abstractmethod
    def sleep(self) -> None: ...
    @abstractmethod
    def report_hours(self) -> int: ...
    @abstractmethod
    def request_vacation(self) -> None: ...
    @abstractmethod
    def attend_meeting(self) -> None: ...
    @abstractmethod
    def submit_timesheet(self) -> None: ...

class Robot(WorkerInterface):
    def work(self) -> None: ...
    def eat(self) -> None: raise NotImplementedError
    def sleep(self) -> None: raise NotImplementedError
    def report_hours(self) -> int: return 0
    def request_vacation(self) -> None: raise NotImplementedError
    def attend_meeting(self) -> None: ...
    def submit_timesheet(self) -> None: raise NotImplementedError
```

**Violation**: `Robot` is forced to implement human-specific methods it doesn't need.

### Python — After (Fixed)

```python
from abc import ABC, abstractmethod

class Workable(ABC):
    @abstractmethod
    def work(self) -> None: ...

class HumanWorker(ABC):
    @abstractmethod
    def eat(self) -> None: ...
    @abstractmethod
    def sleep(self) -> None: ...
    @abstractmethod
    def request_vacation(self) -> None: ...

class Reportable(ABC):
    @abstractmethod
    def report_hours(self) -> int: ...
    @abstractmethod
    def submit_timesheet(self) -> None: ...

class Robot(Workable, Reportable):
    def work(self) -> None: ...
    def report_hours(self) -> int: return 0
    def submit_timesheet(self) -> None: ...
```

### TypeScript — Before (Violation)

```typescript
interface Printer {
  print(doc: Document): void;
  scan(doc: Document): Document;
  fax(doc: Document): void;
  email(doc: Document): void;
}

class SimplePrinter implements Printer {
  print(doc: Document): void { /* real impl */ }
  scan(doc: Document): Document { throw new Error("not supported"); }
  fax(doc: Document): void { throw new Error("not supported"); }
  email(doc: Document): void { throw new Error("not supported"); }
}
```

### TypeScript — After (Fixed)

```typescript
interface Printable {
  print(doc: Document): void;
}

interface Scannable {
  scan(doc: Document): Document;
}

interface Faxable {
  fax(doc: Document): void;
}

class SimplePrinter implements Printable {
  print(doc: Document): void { /* real impl */ }
}

class AllInOnePrinter implements Printable, Scannable, Faxable {
  print(doc: Document): void { /* impl */ }
  scan(doc: Document): Document { /* impl */ return doc; }
  fax(doc: Document): void { /* impl */ }
}
```

---

## D — Dependency Inversion Principle

### Python — Before (Violation)

```python
class OrderService:
    def __init__(self) -> None:
        self.db = PostgresDatabase(host="localhost", port=5432)
        self.mailer = SmtpMailer(host="smtp.example.com")

    def place_order(self, order: Order) -> None:
        self.db.save(order)
        self.mailer.send(order.user_email, "Order placed")
```

**Violation**: `OrderService` directly instantiates concrete `PostgresDatabase` and `SmtpMailer` — impossible to swap implementations or test without a real database and SMTP server.

### Python — After (Fixed)

```python
from typing import Protocol

class Database(Protocol):
    def save(self, order: Order) -> None: ...

class Mailer(Protocol):
    def send(self, to: str, subject: str) -> None: ...

class OrderService:
    def __init__(
        self,
        db: Database = PostgresDatabase(host="localhost", port=5432),
        mailer: Mailer = SmtpMailer(host="smtp.example.com"),
    ) -> None:
        self.db = db
        self.mailer = mailer

    def place_order(self, order: Order) -> None:
        self.db.save(order)
        self.mailer.send(order.user_email, "Order placed")
```

### TypeScript — Before (Violation)

```typescript
class ReportService {
  private db = new MySqlDatabase("localhost", 3306);

  generateReport(userId: string): Report {
    const data = this.db.query(`SELECT * FROM orders WHERE user_id = ?`, [userId]);
    return buildReport(data);
  }
}
```

### TypeScript — After (Fixed)

```typescript
interface Database {
  query(sql: string, params: unknown[]): unknown[];
}

class ReportService {
  constructor(private db: Database) {}

  generateReport(userId: string): Report {
    const data = this.db.query(`SELECT * FROM orders WHERE user_id = ?`, [userId]);
    return buildReport(data);
  }
}
```
