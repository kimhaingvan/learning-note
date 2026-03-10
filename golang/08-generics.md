# Generics in Go

> **Category**: Golang | **Level**: Intermediate

---

## What

**Generics** (introduced in Go 1.18) allow you to write functions and types that work across **multiple types** without sacrificing type safety or resorting to `interface{}`.

Before generics:
```go
// Had to write one version per type, or use interface{} and lose type safety
func SumInts(nums []int) int      { ... }
func SumFloats(nums []float64) float64 { ... }
```

After generics:
```go
func Sum[T int | float64](nums []T) T { ... }
```

---

## Why

- **Code reuse without boilerplate** — write once, work with many types
- **Type safety preserved** — compiler catches type mismatches at compile time
- **No runtime overhead from type assertions** — eliminates the `interface{}` + type assertion pattern

---

## When

Use generics when:
- You find yourself writing the same function/type for multiple concrete types
- You need a reusable container (`Stack`, `Queue`, `Set`, `Result[T]`)
- You're building utility functions (`Map`, `Filter`, `Reduce`) for slices

Don't use generics just to be clever — if your code only works with one type, use that type directly.

---

## Syntax — Type Parameters

Type parameters appear in **square brackets** `[T constraint]`:

```go
// Single type parameter with the "any" constraint (accepts any type)
func Print[T any](val T) {
    fmt.Println(val)
}

// Two type parameters
func Map[T, U any](slice []T, f func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = f(v)
    }
    return result
}
```

---

## Constraints — Restricting Type Parameters

A **constraint** defines which types are allowed for a type parameter.

### Built-in Constraints

| Constraint | Meaning |
|------------|---------|
| `any` | Any type (alias for `interface{}`) |
| `comparable` | Types that support `==` and `!=` |
| `constraints.Ordered` | Types that support `<`, `<=`, `>`, `>=` (numbers + strings) |

```go
import "golang.org/x/exp/constraints"

func Min[T constraints.Ordered](a, b T) T {
    if a < b {
        return a
    }
    return b
}
```

### Custom Constraints Using Type Unions

```go
// Only allow int, int64, or float64
type Number interface {
    int | int64 | float64
}

func Sum[T Number](nums []T) T {
    var total T
    for _, n := range nums {
        total += n
    }
    return total
}

func main() {
    fmt.Println(Sum([]int{1, 2, 3}))          // 6
    fmt.Println(Sum([]float64{1.1, 2.2, 3.3})) // 6.6
}
```

### Constraint with Methods

```go
type Stringer interface {
    String() string
}

func PrintAll[T Stringer](items []T) {
    for _, item := range items {
        fmt.Println(item.String())
    }
}
```

---

## Generic Functions

### Map — Transform Each Element

```go
func Map[T, U any](s []T, f func(T) U) []U {
    result := make([]U, len(s))
    for i, v := range s {
        result[i] = f(v)
    }
    return result
}

func main() {
    nums := []int{1, 2, 3, 4}
    doubled := Map(nums, func(n int) int { return n * 2 })
    strs := Map(nums, func(n int) string { return fmt.Sprintf("item-%d", n) })
    fmt.Println(doubled) // [2 4 6 8]
    fmt.Println(strs)    // [item-1 item-2 item-3 item-4]
}
```

### Filter — Keep Matching Elements

```go
func Filter[T any](s []T, pred func(T) bool) []T {
    var result []T
    for _, v := range s {
        if pred(v) {
            result = append(result, v)
        }
    }
    return result
}

func main() {
    evens := Filter([]int{1, 2, 3, 4, 5, 6}, func(n int) bool { return n%2 == 0 })
    fmt.Println(evens) // [2 4 6]
}
```

### Contains — Check Membership

```go
func Contains[T comparable](s []T, target T) bool {
    for _, v := range s {
        if v == target {
            return true
        }
    }
    return false
}
```

---

## Generic Types

### Stack — Type-Safe Stack

```go
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, bool) {
    var zero T
    if len(s.items) == 0 {
        return zero, false
    }
    top := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return top, true
}

func (s *Stack[T]) Len() int {
    return len(s.items)
}

func main() {
    s := &Stack[int]{}
    s.Push(1); s.Push(2); s.Push(3)

    if v, ok := s.Pop(); ok {
        fmt.Println("Popped:", v) // Popped: 3
    }

    strStack := &Stack[string]{}
    strStack.Push("hello")
    strStack.Push("world")
}
```

### Result — Success / Error Container

```go
type Result[T any] struct {
    Value T
    Err   error
}

func NewResult[T any](v T, err error) Result[T] {
    return Result[T]{Value: v, Err: err}
}

func (r Result[T]) IsOK() bool { return r.Err == nil }
```

---

## Type Inference

Go can often infer type parameters from the arguments — you don't need to write them explicitly:

```go
// Explicit
result := Map[int, string]([]int{1, 2, 3}, strconv.Itoa)

// Inferred — cleaner
result := Map([]int{1, 2, 3}, strconv.Itoa)
```

---

## Pitfalls

- **Overusing generics** for simple cases — use concrete types when the function always handles one type.
- **`comparable` ≠ `ordered`** — `comparable` allows `==`/`!=` only; for `<`/`>`, use `constraints.Ordered`.
- **Type parameters on methods are not supported** — you can't write `func (s *Foo) Bar[T any]()`. Put the type parameter on the type instead.
- **`any` loses type information at runtime** — if you need runtime type info, use reflection.

---

## Memory Aid

> Generics = a template cookie cutter. Define the shape once; press it into any compatible dough (type).

---

## What to Learn Next

- [09-reflection.md](09-reflection.md) — Runtime type inspection when generics aren't enough
