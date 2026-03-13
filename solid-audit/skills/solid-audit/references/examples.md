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

### Java — Before (Violation)

```java
class UserService {
    public List<Map<String, Object>> parseCsv(String path) { /* CSV parsing */ }
    public void saveToDb(List<Map<String, Object>> users) { /* DB writes */ }
    public void sendWelcomeEmail(String email) { /* SMTP send */ }
}
```

### Java — After (Fixed)

```java
class CsvParser {
    public List<Map<String, Object>> parse(String path) { /* CSV parsing */ }
}

class UserRepository {
    public void save(List<Map<String, Object>> users) { /* DB writes */ }
}

class EmailService {
    public void sendWelcome(String email) { /* SMTP send */ }
}
```

### Go — Before (Violation)

```go
type UserService struct{}

func (s *UserService) ParseCSV(path string) []map[string]string { /* CSV parsing */ }
func (s *UserService) SaveToDB(users []map[string]string) { /* DB writes */ }
func (s *UserService) SendWelcomeEmail(email string) { /* SMTP send */ }
```

### Go — After (Fixed)

```go
type CSVParser struct{}
func (p *CSVParser) Parse(path string) []map[string]string { /* CSV parsing */ }

type UserRepository struct{}
func (r *UserRepository) Save(users []map[string]string) { /* DB writes */ }

type EmailService struct{}
func (e *EmailService) SendWelcome(email string) { /* SMTP send */ }
```

### C# — Before (Violation)

```csharp
class UserService {
    public List<Dictionary<string, string>> ParseCsv(string path) { /* CSV parsing */ }
    public void SaveToDb(List<Dictionary<string, string>> users) { /* DB writes */ }
    public void SendWelcomeEmail(string email) { /* SMTP send */ }
}
```

### C# — After (Fixed)

```csharp
class CsvParser {
    public List<Dictionary<string, string>> Parse(string path) { /* CSV parsing */ }
}

class UserRepository {
    public void Save(List<Dictionary<string, string>> users) { /* DB writes */ }
}

class EmailService {
    public void SendWelcome(string email) { /* SMTP send */ }
}
```

### Kotlin — Before (Violation)

```kotlin
class UserService {
    fun parseCsv(path: String): List<Map<String, String>> = TODO()
    fun saveToDb(users: List<Map<String, String>>) = TODO()
    fun sendWelcomeEmail(email: String) = TODO()
}
```

### Kotlin — After (Fixed)

```kotlin
class CsvParser {
    fun parse(path: String): List<Map<String, String>> = TODO()
}

class UserRepository {
    fun save(users: List<Map<String, String>>) = TODO()
}

class EmailService {
    fun sendWelcome(email: String) = TODO()
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

### Java — Before (Violation)

```java
String exportReport(List<?> data, String format) {
    if (format.equals("pdf")) return renderPdf(data);
    else if (format.equals("csv")) return renderCsv(data);
    else if (format.equals("json")) return renderJson(data);
    else throw new IllegalArgumentException("Unknown format: " + format);
}
```

### Java — After (Fixed)

```java
interface Renderer { String render(List<?> data); }

Map<String, Renderer> renderers = Map.of(
    "pdf", new PdfRenderer(),
    "csv", new CsvRenderer(),
    "json", new JsonRenderer()
);

String exportReport(List<?> data, String format) {
    Renderer r = renderers.get(format);
    if (r == null) throw new IllegalArgumentException("Unknown format: " + format);
    return r.render(data);
}
```

### Go — Before (Violation)

```go
func exportReport(data []interface{}, format string) (string, error) {
    if format == "pdf" { return renderPDF(data), nil }
    if format == "csv" { return renderCSV(data), nil }
    return "", fmt.Errorf("unknown format: %s", format)
}
```

### Go — After (Fixed)

```go
type Renderer interface { Render(data []interface{}) string }
var renderers = map[string]Renderer{"pdf": &PDFRenderer{}, "csv": &CSVRenderer{}}

func exportReport(data []interface{}, format string) (string, error) {
    r, ok := renderers[format]
    if !ok { return "", fmt.Errorf("unknown format: %s", format) }
    return r.Render(data), nil
}
```

### C# — Before (Violation)

```csharp
string ExportReport(List<object> data, string format) {
    if (format == "pdf") return RenderPdf(data);
    else if (format == "csv") return RenderCsv(data);
    else throw new ArgumentException($"Unknown format: {format}");
}
```

### C# — After (Fixed)

```csharp
interface IRenderer { string Render(List<object> data); }

var renderers = new Dictionary<string, IRenderer> {
    ["pdf"] = new PdfRenderer(),
    ["csv"] = new CsvRenderer(),
};

string ExportReport(List<object> data, string format) {
    if (!renderers.TryGetValue(format, out var r))
        throw new ArgumentException($"Unknown format: {format}");
    return r.Render(data);
}
```

### Kotlin — Before (Violation)

```kotlin
fun exportReport(data: List<Any>, format: String): String = when (format) {
    "pdf" -> renderPdf(data)
    "csv" -> renderCsv(data)
    else -> throw IllegalArgumentException("Unknown format: $format")
}
```

### Kotlin — After (Fixed)

```kotlin
interface Renderer { fun render(data: List<Any>): String }

val renderers: Map<String, Renderer> = mapOf(
    "pdf" to PdfRenderer(),
    "csv" to CsvRenderer(),
)

fun exportReport(data: List<Any>, format: String): String =
    renderers[format]?.render(data) ?: throw IllegalArgumentException("Unknown format: $format")
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

### Java — Before (Violation)

```java
class Bird {
    public String fly() { return "flying"; }
}

class Penguin extends Bird {
    @Override
    public String fly() { throw new UnsupportedOperationException("Penguins cannot fly"); }
}
```

### Java — After (Fixed)

```java
interface Shape { double area(); }
class Rectangle implements Shape { public double area() { return width * height; } }
class Square implements Shape { public double area() { return side * side; } }
```

### Go — Before (Violation)

```go
type Animal struct{}
func (a *Animal) Fly() string { return "flying" }

type Penguin struct{ Animal }
// Penguin embeds Animal but cannot fly — embedding the wrong behavior
```

### Go — After (Fixed)

```go
// No inheritance — each type satisfies only what it can
type Flyer interface { Fly() string }
type Swimmer interface { Swim() string }

type Eagle struct{}
func (e *Eagle) Fly() string { return "flying" }

type Penguin struct{}
func (p *Penguin) Swim() string { return "swimming" }
```

### C# — Before (Violation)

```csharp
class Rectangle {
    public virtual void SetWidth(double w) { Width = w; }
    public virtual void SetHeight(double h) { Height = h; }
    public double Area() => Width * Height;
}

class Square : Rectangle {
    public override void SetWidth(double w) { Width = w; Height = w; }  // breaks LSP
    public override void SetHeight(double h) { Width = h; Height = h; }
}
```

### C# — After (Fixed)

```csharp
interface IShape { double Area(); }
class Rectangle : IShape { public double Area() => Width * Height; }
class Square : IShape { public double Area() => Side * Side; }
```

### Kotlin — Before (Violation)

```kotlin
open class Bird {
    open fun fly(): String = "flying"
}

class Penguin : Bird() {
    override fun fly(): String = throw UnsupportedOperationException("Penguins cannot fly")
}
```

### Kotlin — After (Fixed)

```kotlin
interface Shape { fun area(): Double }
class Rectangle(val width: Double, val height: Double) : Shape {
    override fun area() = width * height
}
class Square(val side: Double) : Shape {
    override fun area() = side * side
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

### Java — Before (Violation)

```java
interface Worker {
    void work();
    void eat();
    void sleep();
    int reportHours();
    void requestVacation();
}

class Robot implements Worker {
    public void work() { /* impl */ }
    public void eat() { throw new UnsupportedOperationException(); }
    public void sleep() { throw new UnsupportedOperationException(); }
    public int reportHours() { return 0; }
    public void requestVacation() { throw new UnsupportedOperationException(); }
}
```

### Java — After (Fixed)

```java
interface Workable { void work(); }
interface HumanNeeds { void eat(); void sleep(); void requestVacation(); }
interface Reportable { int reportHours(); }

class Robot implements Workable, Reportable {
    public void work() { /* impl */ }
    public int reportHours() { return 0; }
}
```

### Go — Before (Violation)

```go
type Worker interface {
    Work()
    Eat()
    Sleep()
    ReportHours() int
    RequestVacation()
}
```

### Go — After (Fixed)

```go
// Narrow to what each caller actually needs
type Worker interface { Work() }
type Reporter interface { ReportHours() int }
type HumanWorker interface { Worker; Eat(); Sleep(); RequestVacation() }
```

### C# — Before (Violation)

```csharp
interface IWorker {
    void Work();
    void Eat();
    void Sleep();
    int ReportHours();
    void RequestVacation();
}

class Robot : IWorker {
    public void Work() { /* impl */ }
    public void Eat() => throw new NotImplementedException();
    public void Sleep() => throw new NotImplementedException();
    public int ReportHours() => 0;
    public void RequestVacation() => throw new NotImplementedException();
}
```

### C# — After (Fixed)

```csharp
interface IWorkable { void Work(); }
interface IHumanNeeds { void Eat(); void Sleep(); void RequestVacation(); }
interface IReportable { int ReportHours(); }

class Robot : IWorkable, IReportable {
    public void Work() { /* impl */ }
    public int ReportHours() => 0;
}
```

### Kotlin — Before (Violation)

```kotlin
interface Worker {
    fun work()
    fun eat()
    fun sleep()
    fun reportHours(): Int
    fun requestVacation()
}

class Robot : Worker {
    override fun work() { /* impl */ }
    override fun eat() = throw UnsupportedOperationException()
    override fun sleep() = throw UnsupportedOperationException()
    override fun reportHours() = 0
    override fun requestVacation() = throw UnsupportedOperationException()
}
```

### Kotlin — After (Fixed)

```kotlin
interface Workable { fun work() }
interface HumanNeeds { fun eat(); fun sleep(); fun requestVacation() }
interface Reportable { fun reportHours(): Int }

class Robot : Workable, Reportable {
    override fun work() { /* impl */ }
    override fun reportHours() = 0
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

### Java — Before (Violation)

```java
class OrderService {
    private final PostgresDatabase db = new PostgresDatabase("localhost", 5432);

    public void placeOrder(Order order) {
        db.save(order);
    }
}
```

### Java — After (Fixed)

```java
interface Database { void save(Order order); Optional<Order> find(String id); }

class OrderService {
    private final Database db;
    public OrderService(Database db) { this.db = db; }
    // Note: Spring users can use @Autowired instead

    public void placeOrder(Order order) { db.save(order); }
}
```

### Go — Before (Violation)

```go
type OrderService struct {
    db *PostgresDatabase
}
func NewOrderService() *OrderService { return &OrderService{db: NewPostgresDatabase("localhost")} }
```

### Go — After (Fixed)

```go
type Database interface {
    Save(order Order) error
    Find(id string) (Order, error)
}

type OrderService struct { db Database }
func NewOrderService(db Database) *OrderService { return &OrderService{db: db} }
```

### C# — Before (Violation)

```csharp
class OrderService {
    private readonly PostgresDatabase _db = new("localhost", 5432);

    public void PlaceOrder(Order order) => _db.Save(order);
}
```

### C# — After (Fixed)

```csharp
interface IDatabase { void Save(Order order); Order? Find(string id); }

class OrderService {
    private readonly IDatabase _db;
    public OrderService(IDatabase db) { _db = db; }
    // ASP.NET Core: register IDatabase in IServiceCollection in Program.cs

    public void PlaceOrder(Order order) => _db.Save(order);
}
```

### Kotlin — Before (Violation)

```kotlin
class OrderService {
    private val db = PostgresDatabase("localhost", 5432)

    fun placeOrder(order: Order) = db.save(order)
}
```

### Kotlin — After (Fixed)

```kotlin
interface Database { fun save(order: Order); fun find(id: String): Order? }

class OrderService(private val db: Database) {
    // Android: inject Database via Hilt or Koin
    fun placeOrder(order: Order) = db.save(order)
}
```
