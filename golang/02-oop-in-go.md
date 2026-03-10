# OOP in Go

> **Category**: Golang | **Level**: Intermediate

---

## What — The 4 OOP Attributes

Traditional OOP languages (Java, C++) share four core attributes. Go implements all four, but often differently.

| Attribute | Classic OOP | Go Equivalent |
|-----------|-------------|---------------|
| **Encapsulation** | `private/protected/public` keywords | Package-level visibility (lowercase = unexported) |
| **Abstraction** | Abstract classes / interfaces | Interfaces (implicitly satisfied) |
| **Inheritance** | `extends` keyword, class hierarchy | Composition via **embedding** |
| **Polymorphism** | Method overriding, virtual dispatch | Interface values (dynamic dispatch) |

---

## Why

To achieve **modularity**, **code reuse**, and **clear system boundaries** — even though Go deliberately avoids classical inheritance. The result is simpler, less coupled code.

---

## When

Use OOP practices when you need:
- **Encapsulation** — hide implementation details behind a clean API
- **Polymorphism** — treat different types through a shared interface
- **Clear separation of concerns** — split responsibilities across packages and types

---

## How — Go's Implementation

### 1. Encapsulation — Package-Level Visibility

Go has no `private`/`public` keywords. Visibility is controlled by the **case of the first letter**:

| First letter | Visibility |
|-------------|------------|
| **Lowercase** | Unexported — hidden outside the package |
| **Uppercase** | Exported — accessible from any package |

```go
// Unexported: only accessible within the same package
func (s *GenerateBillItemsService) getMasterDataForProduct() {}

// Exported: accessible from any package
func (e *Product) IsOneTimeProduct() bool {
    return e.BillingScheduleID.Status != pgtype.Present
}
```

> **Tip:** Export only what callers need. Keep implementation details unexported to protect internal state.

---

### 2. Abstraction — Interfaces

A type **implicitly** satisfies an interface by implementing all its methods — no `implements` keyword needed.

```go
// Define: what behavior is required, not how it's done
type ITaxServiceForGenerateBillItem interface {
    GetTaxByID(ctx context.Context, db database.QueryExecer, taxID pgtype.Text) (tax entities.Tax, err error)
}
```

**Rules for good interfaces:**
- Define **small, focused** interfaces (1–3 methods)
- Name them by behavior: `-er` suffix (`Reader`, `Writer`, `Notifier`)
- Let types satisfy them **without knowing it**

---

### 3. Inheritance → Composition via Embedding

Go **does not** have classical inheritance. Instead, you **embed** one struct inside another to promote its fields and methods.

```go
type Engine struct {
    HP int
}

func (e Engine) Start() {
    fmt.Println("Engine starting, HP:", e.HP)
}

type Car struct {
    Engine  // ← embedding: Car "inherits" Engine's fields and methods
    Brand string
}

func main() {
    c := Car{Engine: Engine{HP: 200}, Brand: "Toyota"}
    c.Start()  // promoted method — called directly on Car
    fmt.Println(c.HP)   // promoted field — accessed directly
}
```

> **Key difference from inheritance:** `Car` does not *become* an `Engine`. It *has* an `Engine`. Composition is explicit and avoids fragile class hierarchies.

---

### 4. Polymorphism — Interface Values

Go supports **implicit interface satisfaction**: a type automatically satisfies an interface if it implements all the required methods — no declaration needed.

```go
type Notifier interface {
    Notify(msg string)
}

type Email struct{ Addr string }
func (e Email) Notify(m string) {
    fmt.Println("Email to", e.Addr, ":", m)
}

type SMS struct{ Num string }
func (s SMS) Notify(m string) {
    fmt.Println("SMS to", s.Num, ":", m)
}

// Alert works with ANY type that satisfies Notifier
func Alert(n Notifier) {
    n.Notify("Server down")
}

func main() {
    Alert(Email{Addr: "admin@example.com"})
    Alert(SMS{Num: "+84-123-456"})
}
```

**Two kinds of polymorphism in Go:**

| Kind | Mechanism | When resolved |
|------|-----------|--------------|
| **Static** | Each concrete type has its own methods, called directly | Compile time |
| **Dynamic** | Interface variable holds any value satisfying the interface | Runtime (virtual dispatch) |

---

## Implicit Implementation — Why It Matters

Go has **no** `implements` keyword. A type satisfies an interface just by having the right methods.

**Benefits:**
- **Backward compatibility**: Define interfaces for types from packages you don't control.
- **Interface segregation**: Tailor small interfaces to specific needs.
- **Loose coupling**: Types don't need to know which interfaces they satisfy.

```go
// Any type with Read() AND Write() automatically satisfies ReadWriter
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// Embeds both — composed interface
type ReadWriter interface {
    Reader
    Writer
}
```

---

## The Empty Interface — `interface{}` / `any`

Every type satisfies the empty interface (similar to `Object` in Java or `any` in TypeScript).

```go
func printAny(v interface{}) {
    fmt.Printf("Value: %v, Type: %T\n", v, v)
}

func main() {
    printAny(42)               // Value: 42, Type: int
    printAny("hello")          // Value: hello, Type: string
    printAny(true)             // Value: true, Type: bool
    printAny(Rectangle{3, 4}) // Value: {3 4}, Type: main.Rectangle
}
```

> ⚠️ Use sparingly — it sacrifices Go's static type safety. Prefer generics (Go 1.18+) when you need type-safe reusable code.

---

## Type Assertions and Type Switches

When working with interface values, you often need to access the underlying concrete type.

### Type Assertion

```go
str, ok := i.(string)
if ok {
    fmt.Printf("String value: %s\n", str)
}
// Always use the two-value form (ok) to avoid panics
```

### Type Switch

```go
func describe(i interface{}) {
    switch v := i.(type) {
    case int:
        fmt.Printf("Integer: %d\n", v)
    case string:
        fmt.Printf("String: %s\n", v)
    case bool:
        fmt.Printf("Boolean: %t\n", v)
    case Rectangle:
        fmt.Printf("Rectangle area: %.2f\n", v.Area())
    default:
        fmt.Printf("Unknown type: %T\n", v)
    }
}

func main() {
    describe("hello")         // String: hello
    describe(42)              // Integer: 42
    describe(true)            // Boolean: true
    describe(Rectangle{5, 7}) // Rectangle area: 35.00
    describe(3.14)            // Unknown type: float64
}
```

---

## Summary

| Concept | Key Takeaway |
|---------|-------------|
| **Structs** | Go's building blocks — group related data without class hierarchies |
| **Methods** | Give behavior to structs; value vs pointer receiver gives fine-grained control |
| **Embedding** | Enables composition — builds complex types from simpler ones |
| **Interfaces** | Define behavior contracts — focus on *what* types can do, not *what* they are |
| **Implicit implementation** | Reduces coupling — more adaptable and maintainable code |

---

## What to Learn Next

- [03-pointer-value-methods.md](03-pointer-value-methods.md) — Value vs pointer receivers in depth
- [08-generics.md](08-generics.md) — Type-safe code reuse without `interface{}`
