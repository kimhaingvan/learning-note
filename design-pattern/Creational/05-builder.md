# Builder

> **Category**: Creational | **Language**: Go

---

## What

Builder lets you construct a complex object **step by step** through **separated, small methods**, using only the methods you really need.

---

## Why / When

- Avoid constructors with many parameters.
- Construction of various representations involves similar steps that differ only in the details.

---

## How

1. **Define the struct** you want to build (usually complex).
2. **Declare methods** on the struct to assemble it step by step.

Go offers two idiomatic approaches:

---

### Solution 1: Method Chaining

```go
package main

import "errors"

type Car struct {
	Color   string
	Seats   int
	Sunroof bool
}

func NewCar() *Car { return &Car{} }

func (b *Car) SetPaint(color string) *Car {
	b.Color = color
	return b
}

func (b *Car) SetSeats(n int) *Car {
	b.Seats = n
	return b
}

func (b *Car) SetSunroof(on bool) *Car {
	b.Sunroof = on
	return b
}

func (b *Car) Build() (Car, error) {
	if b.Seats <= 0 {
		return Car{}, errors.New("seats must be >0")
	}
	return *b, nil
}
```

### Solution 2: Functional Options

```go
package main

import "errors"

type Car2 struct {
	Color   string
	Seats   int
	Sunroof bool
}

type CarOption func(*Car2) error

func WithColor(c string) CarOption { return func(car *Car2) error { car.Color = c; return nil } }
func WithSeats(n int) CarOption    { return func(car *Car2) error { car.Seats = n; return nil } }
func WithSunroof() CarOption       { return func(car *Car2) error { car.Sunroof = true; return nil } }

func NewCar2(opts ...CarOption) (Car2, error) {
	c := Car2{Seats: 4} // defaults
	for _, o := range opts {
		if err := o(&c); err != nil {
			return Car2{}, err
		}
	}
	if c.Seats <= 0 {
		return Car2{}, errors.New("seats must be >0")
	}
	return c, nil
}
```

---

## Example — Client Code

```go
package main

import "fmt"

func main() {
	// Method chaining
	car, _ := NewCar().
		SetPaint("red").
		SetSeats(2).
		SetSunroof(true).
		Build()

	// Functional options
	opts := []CarOption{
		WithColor("red"),
		WithSeats(2),
		WithSunroof(),
	}
	car2, _ := NewCar2(opts...)

	fmt.Println(car)  // {red 2 true}
	fmt.Println(car2) // {red 2 true}
}
```

---

## Method Chaining vs Functional Options

| | Method Chaining | Functional Options |
|---|---|---|
| **Readability** | Good for short chains | Good for config-heavy setup |
| **Extensibility** | Add method to struct | Add a new option function |
| **Validation** | Deferred to `Build()` | Can validate per-option |
| **Idiomatic Go** | Common | Very common (e.g., gRPC, HTTP servers) |

---

## Pitfalls

- Forgetting a `Build()` or validation step — resulting in incomplete objects.
- Builder pattern is overkill for simple structs with 2-3 fields.

---

## Memory Aid

> Builder = "construct step by step, pick only what you need." Like ordering a custom burger — choose bun, patty, toppings one at a time.
