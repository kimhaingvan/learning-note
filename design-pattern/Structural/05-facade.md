# Facade

> **Category**: Structural | **Language**: Go

---

## What

**Facade** provides a **simplified interface** and hides the complex details of subsystems inside.

- The Facade is your "public front door" — a minimal, stable interface.
- All complex details are hidden behind the scenes.
- Expose only what is needed. Clients depend on the Facade interface only.

---

## Why

- **Loose coupling** — clients depend on the Facade; subsystems can evolve without breaking client code.
- **Simplicity** — clients get a clean API instead of wrestling with dozens of subsystem types.

---

## When

When you need a limited but straightforward interface to a complex subsystem.

---

## How

**Principle**: Expose what the client needs and hide the detail through interfaces.

1. **Define Subsystem Interfaces** — narrow interfaces per subsystem.
2. **Create and Implement Subsystem Objects.**
3. **Create the Facade Object** — fields: one interface per subsystem. Constructor: inject concrete subsystem instances. Facade methods: delegate to subsystems in sequence.
4. **Client uses only the Facade** — does not import or invoke subsystems directly.

---

## Example

```go
package main

import (
	"fmt"
	"log"
)

// ── Step 1: Subsystem interfaces ──

type IUserService interface {
	CheckUserByUsername(username string) error
}

type IProductService interface {
	CheckProductByID(productID string) error
}

// ── Step 2: Subsystem implementations ──

type UserService struct{}

func (s *UserService) CheckUserByUsername(username string) error {
	fmt.Println("Checking user:", username)
	return nil
}

type ProductService struct{}

func (s *ProductService) CheckProductByID(productID string) error {
	fmt.Println("Checking product:", productID)
	return nil
}

// ── Step 3: Facade ──

type ImportOrderGroup struct {
	userService    IUserService
	productService IProductService
}

func NewImportOrderGroup() *ImportOrderGroup {
	return &ImportOrderGroup{
		productService: &ProductService{},
		userService:    &UserService{},
	}
}

func (s *ImportOrderGroup) CheckExists(productID, userName string) error {
	// Facade delegates to subsystems — client does not know the details
	if err := s.productService.CheckProductByID(productID); err != nil {
		return err
	}
	if err := s.userService.CheckUserByUsername(userName); err != nil {
		return err
	}
	return nil
}

// ── Step 4: Client ──

func main() {
	facade := NewImportOrderGroup()
	if err := facade.CheckExists("product_id_1", "user_name_1"); err != nil {
		log.Fatal(err)
	}
	// Checking product: product_id_1
	// Checking user: user_name_1
}
```

---

## Pitfalls

- Facade becoming a "god object" that does too much — keep it thin.
- Exposing subsystem types through the Facade leaks the abstraction.

---

## Memory Aid

> Facade = hotel concierge. You say "I need a restaurant, tickets, and a taxi." The concierge handles three different services — you never call them directly.
