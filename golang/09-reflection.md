# Reflection in Go

> **Category**: Golang | **Level**: Intermediate

---

## What

The `reflect` package lets you **inspect and manipulate types and values at runtime** — when the type is not known at compile time.

Two core types:

| Type | Represents | How to get |
|------|------------|------------|
| `reflect.Type` | The **type** of a value | `reflect.TypeOf(x)` |
| `reflect.Value` | The **value** itself | `reflect.ValueOf(x)` |

---

## Why

Reflection enables:
- **Generic serialization** (JSON, YAML encoders) — traverse any struct's fields without knowing the type.
- **Dependency injection** — resolve and wire types automatically.
- **ORM / query builders** — map struct fields to database columns.
- **Testing utilities** — deep equality checks, fake data builders.

---

## When

Use reflection only when:
- The type is truly unknown at compile time
- The same logic must work for **any** struct without code generation

Preferring **generics** or interfaces when possible — they're faster and type-safe.

---

## `reflect.Type` — Inspecting a Type

```go
package main

import (
    "fmt"
    "reflect"
)

type Person struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
}

func main() {
    p := Person{Name: "Alice", Age: 30}
    t := reflect.TypeOf(p)

    fmt.Println(t.Name())    // Person
    fmt.Println(t.Kind())    // struct
    fmt.Println(t.NumField()) // 2

    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        fmt.Printf("  Field: %-10s Type: %-10s Tag: %q\n",
            field.Name, field.Type, field.Tag.Get("json"))
    }
}
// Output:
// Person
// struct
// 2
//   Field: Name       Type: string     Tag: "name"
//   Field: Age        Type: int        Tag: "age"
```

### `reflect.Type` Methods Reference

| Method | Returns | Description |
|--------|---------|-------------|
| `Kind()` | `reflect.Kind` | Underlying kind: `struct`, `ptr`, `slice`, `map`, `int`, `string`, … |
| `Name()` | `string` | Type name (empty for anonymous / pointer types) |
| `NumField()` | `int` | Number of struct fields (panics if not a struct) |
| `Field(i)` | `reflect.StructField` | The i-th struct field descriptor |
| `Elem()` | `reflect.Type` | Element type of pointer, slice, map, channel, or array |
| `NumIn()` | `int` | Number of function input parameters |
| `NumOut()` | `int` | Number of function output values |
| `Implements(u)` | `bool` | Whether the type implements interface `u` |

---

## `reflect.Value` — Inspecting and Modifying Values

```go
func main() {
    p := Person{Name: "Alice", Age: 30}
    v := reflect.ValueOf(p)

    fmt.Println(v.Kind())       // struct
    fmt.Println(v.NumField())   // 2

    for i := 0; i < v.NumField(); i++ {
        fmt.Printf("  %s = %v\n", v.Type().Field(i).Name, v.Field(i))
    }
}
// Output:
//   Name = Alice
//   Age = 30
```

### `reflect.Value` Methods Reference

| Method | Description |
|--------|-------------|
| `Kind()` | Returns the Kind of the value |
| `Bool()` | Returns the bool value |
| `Int()` | Returns the int64 value |
| `Uint()` | Returns the uint64 value |
| `Float()` | Returns the float64 value |
| `String()` | Returns the string value |
| `Bytes()` | Returns the `[]byte` value |
| `Len()` | Length of array, slice, map, string, or channel |
| `Index(i)` | The i-th element of array, slice, or string |
| `MapKeys()` | All keys of a map as `[]reflect.Value` |
| `MapIndex(key)` | Value for a map key |
| `Field(i)` | The i-th struct field value |
| `FieldByName(name)` | Struct field value by name |
| `Interface()` | Returns the value as `interface{}` |
| `CanSet()` | Whether the value can be modified |
| `CanAddr()` | Whether the value is addressable |
| `Elem()` | Dereferences a pointer; retrieves the element of an interface |
| `IsNil()` | Whether the value is nil (for pointer, chan, map, slice, func) |
| `IsValid()` | Whether the value is non-zero `reflect.Value` |

---

## Modifying Values via Reflection

You can only set a field if you pass a **pointer** and the field is addressable:

```go
func main() {
    p := Person{Name: "Alice", Age: 30}

    // Must pass a pointer to make fields settable
    v := reflect.ValueOf(&p).Elem()

    if v.FieldByName("Name").CanSet() {
        v.FieldByName("Name").SetString("Bob")
    }
    if v.FieldByName("Age").CanSet() {
        v.FieldByName("Age").SetInt(25)
    }

    fmt.Println(p) // {Bob 25}
}
```

---

## Inspecting Pointers

```go
func inspectPointer(i interface{}) {
    v := reflect.ValueOf(i)

    if v.Kind() == reflect.Ptr {
        fmt.Println("Is a pointer")
        fmt.Println("Points to:", v.Elem().Kind())
        fmt.Println("Value:", v.Elem())
    }
}

func main() {
    x := 42
    inspectPointer(&x)
}
// Output:
// Is a pointer
// Points to: int
// Value: 42
```

---

## Practical Example — Generic Pretty Printer

```go
func prettyPrint(i interface{}) {
    v := reflect.ValueOf(i)
    t := reflect.TypeOf(i)

    // Dereference pointer if needed
    if v.Kind() == reflect.Ptr {
        v = v.Elem()
        t = t.Elem()
    }

    if v.Kind() != reflect.Struct {
        fmt.Printf("%v\n", i)
        return
    }

    fmt.Printf("Type: %s\n", t.Name())
    for i := 0; i < v.NumField(); i++ {
        fmt.Printf("  %-15s = %v\n", t.Field(i).Name, v.Field(i))
    }
}

func main() {
    p := Person{Name: "Alice", Age: 30}
    prettyPrint(p)
    prettyPrint(&p)
}
```

---

## Pitfalls

- **`reflect.ValueOf(nil)` is invalid** — always check `v.IsValid()` before using a returned `reflect.Value`.

  ```go
  var x interface{} = nil
  v := reflect.ValueOf(x)
  fmt.Println(v.IsValid()) // false — don't call methods on this
  ```

- **Calling type-specific methods on the wrong Kind panics** — e.g., `v.Int()` on a `string` Kind.

- **Non-addressable values cannot be set** — pass a pointer and call `.Elem()` to get a settable value.

- **Reflection is slow** — avoid in hot paths. Cache `reflect.Type` results where possible.

- **Unexported fields** are not settable via reflection — `CanSet()` returns false.

---

## Memory Aid

> `TypeOf` = the label on the box (tells you what kind it is).
> `ValueOf` = the contents inside (lets you read and optionally change them — only if you brought the original, not a copy).

---

## What to Learn Next

- [10-memory.md](10-memory.md) — Stack vs heap, and why escape analysis matters
