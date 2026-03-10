# Abstract Factory

> **Category**: Creational | **Language**: Go

---

## What

An Abstract Factory supplies a **family of related objects** through one interface, keeping client code unaware of the concrete types.

- This is an interface whose methods each return *another* interface.
- Usually implemented by multiple Factory Methods.

---

## Why

- Create entire families of related objects (e.g., Nike shoes + Nike shirts) without mixing brands.
- Client code depends only on abstract interfaces — never on concrete `Nike` or `Adidas` types directly.

---

## When

- You need to produce **families of related products** (shoe + shirt for the same brand).
- You want to enforce that products from one family are used together.

---

## How

1. **Define product interfaces** for all product types.
2. **Define the factory interface** — creation methods for each product type, returning product interfaces.
3. **Create concrete product structs** implementing the product interfaces.
4. **Implement one concrete factory per variant** (one per brand/family).
5. **Define a factory initialization function** that picks the right factory based on a parameter.

---

## Example

```go
package main

import "fmt"

// ── Step 1: Product interfaces ──

type IShoe interface {
	setLogo(logo string)
	setSize(size int)
	getLogo() string
	getSize() int
}

type IShirt interface {
	setLogo(logo string)
	setSize(size int)
	getLogo() string
	getSize() int
}

// ── Step 2: Factory interface ──

type ISportsFactory interface {
	makeShoe() IShoe
	makeShirt() IShirt
}

// ── Step 3: Concrete products ──

type Shoe struct {
	logo string
	size int
}

func (s *Shoe) setLogo(logo string) { s.logo = logo }
func (s *Shoe) getLogo() string      { return s.logo }
func (s *Shoe) setSize(size int)     { s.size = size }
func (s *Shoe) getSize() int         { return s.size }

type Shirt struct {
	logo string
	size int
}

func (s *Shirt) setLogo(logo string) { s.logo = logo }
func (s *Shirt) getLogo() string      { return s.logo }
func (s *Shirt) setSize(size int)     { s.size = size }
func (s *Shirt) getSize() int         { return s.size }

// ── Step 4: Concrete factories ──

type Nike struct{}

func (n *Nike) makeShoe() IShoe {
	return &Shoe{logo: "nike", size: 14}
}

func (n *Nike) makeShirt() IShirt {
	return &Shirt{logo: "nike", size: 14}
}

type Adidas struct{}

func (a *Adidas) makeShoe() IShoe {
	return &Shoe{logo: "adidas", size: 14}
}

func (a *Adidas) makeShirt() IShirt {
	return &Shirt{logo: "adidas", size: 14}
}

// ── Step 5: Factory selector ──

func GetSportsFactory(brand string) (ISportsFactory, error) {
	if brand == "adidas" {
		return &Adidas{}, nil
	}
	if brand == "nike" {
		return &Nike{}, nil
	}
	return nil, fmt.Errorf("wrong brand type passed")
}
```

```go
func main() {
	adidasFactory, _ := GetSportsFactory("adidas")
	nikeFactory, _ := GetSportsFactory("nike")

	nikeShoe := nikeFactory.makeShoe()
	nikeShirt := nikeFactory.makeShirt()
	adidasShoe := adidasFactory.makeShoe()
	adidasShirt := adidasFactory.makeShirt()

	fmt.Printf("Logo: %s, Size: %d\n", nikeShoe.getLogo(), nikeShoe.getSize())
	fmt.Printf("Logo: %s, Size: %d\n", nikeShirt.getLogo(), nikeShirt.getSize())
	fmt.Printf("Logo: %s, Size: %d\n", adidasShoe.getLogo(), adidasShoe.getSize())
	fmt.Printf("Logo: %s, Size: %d\n", adidasShirt.getLogo(), adidasShirt.getSize())
	// Logo: nike, Size: 14
	// Logo: nike, Size: 14
	// Logo: adidas, Size: 14
	// Logo: adidas, Size: 14
}
```

---

## Abstract Factory vs Factory Method

| | Factory Method | Abstract Factory |
|---|---|---|
| **Returns** | One product via interface | A family of related products |
| **Structure** | Single function | Interface with multiple creation methods |
| **Use case** | Pick one variant | Ensure consistent product families |

---

## Pitfalls

- Over-engineering when you only have one product type — use Factory Method instead.
- Adding a new product type to the factory interface forces changes in every concrete factory.

---

## Memory Aid

> Abstract Factory = "factory of factories." One factory per brand, each producing a matching family of products.
