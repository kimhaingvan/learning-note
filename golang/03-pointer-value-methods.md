# Pointer Methods vs Value Methods

> **Category**: Golang | **Level**: Intermediate

---

## What

A **receiver** is the "subject" of a method — the type that owns it. Go lets you define methods with either a **value receiver** or a **pointer receiver**.

### Syntax Comparison

| Aspect | Value Receiver | Pointer Receiver |
|--------|---------------|-----------------|
| **Syntax** | `func (p Person) Method()` | `func (p *Person) Method()` |
| **Modifies original struct?** | ❌ No — works on a **copy** | ✅ Yes — works on the **actual object** |
| **Memory allocation** | Copies the entire struct | No copy — uses a reference |
| **Used with interfaces?** | Satisfies value-based interface | Satisfies both value and pointer-based interfaces |
| **Common for** | Immutable methods, small structs | Mutating methods, large structs |
| **Auto-handled by Go?** | ✅ Yes — auto-dereferences | ✅ Yes — can call pointer methods on values |

```go
type Person struct {
    Name string
}

// Value receiver — works on a copy
func (p Person) Greet() {
    fmt.Println("Hello,", p.Name)
}

// Pointer receiver — mutates the original
func (p *Person) Rename(newName string) {
    p.Name = newName
}
```

---

## Why

- **Value receiver**: safe for read-only methods — callers know nothing will change.
- **Pointer receiver**: necessary when the method needs to mutate the struct, or the struct is large and copying would be expensive.

---

## When — Which to Choose

| Use Value Receiver | Use Pointer Receiver |
|--------------------|---------------------|
| Method **does not modify** the struct | Method **modifies** the struct fields |
| Struct is **small** (a few fields) | Struct is **large** (copying is expensive) |
| Method represents a **read-only query** | Method represents a **command / mutation** |

> **Pro Tip:** Be **consistent**. If any method on a type uses a pointer receiver, prefer pointer receivers for **all** methods on that type. Mixed receivers cause confusion and interface satisfaction issues.

---

## How

### Auto-Dereferencing

Go automatically handles the conversion between value and pointer when it's safe:

```go
p := Person{Name: "Alice"}

// Go auto-takes the address: equivalent to (&p).Rename("Bob")
p.Rename("Bob")
fmt.Println(p.Name) // Bob

ptr := &Person{Name: "Charlie"}

// Go auto-dereferences: equivalent to (*ptr).Greet()
ptr.Greet() // Hello, Charlie
```

> Auto-dereferencing only works for **addressable variables** (local variables, struct fields). It does **not** work for non-addressable values like function return values or map entries.

---

### Interface Satisfaction Rules

This is where the difference really matters:

```go
type Greeter interface {
    Greet()
}

p := Person{Name: "Alice"}
ptr := &Person{Name: "Bob"}

// Value receiver method → both value and pointer satisfy the interface
var g Greeter
g = p    // ✅ works
g = ptr  // ✅ works

// Pointer receiver method → only pointer satisfies the interface
type Renamer interface {
    Rename(name string)
}
var r Renamer
r = ptr  // ✅ works
r = p    // ❌ compile error: Person does not implement Renamer
```

**Rule:** A pointer type `*T` has the method set of both `T` and `*T`. A value type `T` only has the method set of `T`.

---

## Example — Full Demo

```go
package main

import "fmt"

type Rectangle struct {
    Width, Height float64
}

// Value receiver — read-only: safe to copy
func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

// Value receiver — returns a new value (doesn't mutate)
func (r Rectangle) Scale(factor float64) Rectangle {
    return Rectangle{r.Width * factor, r.Height * factor}
}

// Pointer receiver — mutates in-place
func (r *Rectangle) ScaleInPlace(factor float64) {
    r.Width *= factor
    r.Height *= factor
}

func main() {
    r := Rectangle{Width: 3, Height: 4}

    fmt.Println("Area:", r.Area())           // 12

    r2 := r.Scale(2)
    fmt.Println("Scaled (new):", r2)         // {6 8}
    fmt.Println("Original unchanged:", r)    // {3 4} ← copy not modified

    r.ScaleInPlace(2)
    fmt.Println("Scaled in-place:", r)       // {6 8} ← original modified
}
```

---

## Pitfalls

- **Mixing receiver types** on the same struct causes interface satisfaction surprises.
- **Copying a struct with a Mutex** via a value receiver breaks the Mutex — always use pointer receivers when the struct contains a `sync.Mutex`.
- **Map values are not addressable** — you cannot call pointer receiver methods directly on a map value:

  ```go
  m := map[string]Person{"alice": {Name: "Alice"}}
  m["alice"].Rename("Bob") // ❌ compile error: cannot take address of map value
  ```

---

## Memory Aid

> Value receiver = photocopy. You work on the copy, original untouched.
> Pointer receiver = original document. Changes are permanent.

---

## What to Learn Next

- [04-context.md](04-context.md) — Cancellation, timeouts, request-scoped values
- [07-concurrency.md](07-concurrency.md) — Why pointer receivers matter with Mutex
