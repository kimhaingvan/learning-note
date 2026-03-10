# Decorator

> **Category**: Structural | **Language**: Go

---

## What

Decorator adds new behaviors to existing objects **dynamically** by wrapping them in a special wrapper struct, without modifying the original objects.

**3 parts:**

| Part | Role |
|------|------|
| **Component Interface** | Common interface that core objects and decorators implement |
| **Concrete Component** | The original object to be decorated |
| **Decorator** | Holds a reference to a Component and implements the same interface, adding behavior |

---

## Why / When

- You have many concrete structs implementing the same interface and want to assign **extra behaviors** to all of them (before or after their calls) — without changing those objects or breaking existing code.

---

## How

1. **Define the Component interface.**
2. **Create Concrete Components** implementing the interface.
3. **Create Decorator structs** — each holds a Component field.
4. **Implement the Component interface in each Decorator** — delegate to the wrapped component + inject extra behavior.
5. **Chain multiple decorators** by wrapping one inside another.

---

## Example

```go
package main

import "fmt"

// Component interface
type IPizza interface {
	DoPizza() string
}

// Concrete Components
type SeafoodPizza struct{}

func (c *SeafoodPizza) DoPizza() string {
	return "Seafood Pizza"
}

type ChickenPizza struct{}

func (c *ChickenPizza) DoPizza() string {
	return "Chicken Pizza"
}

// Decorators
type CheesePizzaDecorator struct {
	pizza IPizza
}

func (c *CheesePizzaDecorator) DoPizza() string {
	return c.pizza.DoPizza() + " with Cheese"
}

type TomatoPizzaDecorator struct {
	pizza IPizza
}

func (c *TomatoPizzaDecorator) DoPizza() string {
	return c.pizza.DoPizza() + " with Tomato"
}

func main() {
	seafoodPizza := &SeafoodPizza{}
	chickenPizza := &ChickenPizza{}

	cheeseSeafood := &CheesePizzaDecorator{pizza: seafoodPizza}
	tomatoChicken := &TomatoPizzaDecorator{pizza: chickenPizza}

	fmt.Println(cheeseSeafood.DoPizza()) // Seafood Pizza with Cheese
	fmt.Println(tomatoChicken.DoPizza()) // Chicken Pizza with Tomato

	// Chain: Seafood + Cheese + Tomato
	doubleDeco := &TomatoPizzaDecorator{pizza: cheeseSeafood}
	fmt.Println(doubleDeco.DoPizza()) // Seafood Pizza with Cheese with Tomato
}
```

---

## Decorator vs Proxy

| | Decorator | Proxy |
|---|---|---|
| **Purpose** | Add/modify behavior | Control access |
| **Stacking** | Multiple decorators can wrap each other | Usually one proxy layer |
| **Awareness** | Decorators dont manage the lifecycle of the wrapped object | Proxy often manages the lifecycle of its service object |

---

## Pitfalls

- Deep nesting of decorators makes debugging harder — stack traces become long.
- Order of wrapping matters — `Cheese(Tomato(pizza))` != `Tomato(Cheese(pizza))`.

---

## Memory Aid

> Decorator = gift wrapping. Each layer adds something on top without opening or changing the gift inside.
