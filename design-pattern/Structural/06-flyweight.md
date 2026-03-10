# Flyweight

> **Category**: Structural | **Language**: Go

---

## What

Flyweight lets you share many objects with the same **common, immutable data (intrinsic state)** by storing shared instances in memory, instead of duplicating that data in each object.

**4 parts:**

| Part | Role |
|------|------|
| **Flyweight Interface** | Methods to access intrinsic (shared) data |
| **Concrete Flyweight** | Shared object containing common, immutable data |
| **Flyweight Factory** | Stores shared flyweight objects in a map; ensures one instance per unique intrinsic state |
| **Context** | Holds extrinsic (unique per-object) data + a reference to a shared Flyweight |

---

## Why / When

When your program must support a huge number of objects that barely fit into available RAM. The pattern reuses unchanged, immutable data instead of duplicating it across all objects.

---

## How

1. **Define the Flyweight interface** — methods to get intrinsic data.
2. **Divide fields into two parts**:
   - **Intrinsic**: unchanged data duplicated across many objects → shared.
   - **Extrinsic**: contextual data unique to each object → stays in Context.
3. **Define the Flyweight type** — stores intrinsic data, implements the interface.
4. **Implement the Flyweight Factory** — manages the pool; returns existing or creates new.
5. **Create Context objects** — hold extrinsic state + a shared Flyweight reference.

---

## Example

```go
package main

import "fmt"

// Flyweight interface
type IDress interface {
	GetColor() string
}

// Concrete Flyweight — immutable shared data
type Dress struct {
	color string
}

func (d *Dress) GetColor() string {
	return d.color
}

func NewDress(color string) IDress {
	return &Dress{color: color}
}

// Flyweight Factory — ensures one instance per color
type DressFactory struct {
	mapColorWithDress map[string]IDress
}

func NewDressFactory() *DressFactory {
	return &DressFactory{
		mapColorWithDress: make(map[string]IDress),
	}
}

func (s *DressFactory) GetColorSize() int {
	return len(s.mapColorWithDress)
}

func (s *DressFactory) GetDressByColor(color string) IDress {
	dress, exists := s.mapColorWithDress[color]
	if !exists {
		dress = NewDress(color)
		s.mapColorWithDress[color] = dress
	}
	return dress
}

// Context — extrinsic state + shared flyweight
type Player struct {
	name  string
	dress IDress // shared
}

func main() {
	factory := NewDressFactory()

	p1 := Player{name: "Alice", dress: factory.GetDressByColor("red")}
	p2 := Player{name: "Bob", dress: factory.GetDressByColor("red")}
	p3 := Player{name: "Carol", dress: factory.GetDressByColor("blue")}

	fmt.Println(p1.name, p1.dress.GetColor()) // Alice red
	fmt.Println(p2.name, p2.dress.GetColor()) // Bob red
	fmt.Println(p3.name, p3.dress.GetColor()) // Carol blue

	fmt.Println("Unique dresses created:", factory.GetColorSize()) // 2 (not 3)
}
```

---

## Pitfalls

- Shared flyweight data must be **truly immutable** — if any context mutates it, all contexts are affected.
- Adds code complexity for the factory. Only worth it when the number of shared objects is large.
- You trade RAM for CPU: looking up the factory map on every access has a small cost.

---

## Memory Aid

> Flyweight = shared wardrobe. 1,000 players in red use the same one red dress object, not 1,000 copies.
