# Visitor

> **Category**: Behavioral | **Language**: Go

---

## What

**Visitor** lets you add new operations to existing object structures **without modifying their classes**. It achieves this through **double dispatch** — the element calls the visitor, passing itself.

---

## Why / When

- When you need to perform unrelated operations across a set of classes and don't want to pollute them with that logic.
- When the class hierarchy is stable (elements rarely change) but **operations change often**.

---

## How

1. **Define the Element interface** — an `Accept(visitor)` method.
2. **Implement Concrete Elements** — each calls the visitor's matching method inside `Accept`.
3. **Define the Visitor interface** — one `VisitX()` method per element type.
4. **Implement Concrete Visitors** — each provides a different operation for every element.

---

## Example

```go
package main

import "fmt"

// Element interface
type IAnimal interface {
	Accept(visitor IAnimalOperation)
}

// Visitor interface
type IAnimalOperation interface {
	VisitLion(lion *Lion)
	VisitTiger(tiger *Tiger)
	VisitDog(dog *Dog)
}

// Concrete Elements

type Lion struct{}

func (l *Lion) Accept(visitor IAnimalOperation) {
	visitor.VisitLion(l)
}

type Tiger struct{}

func (t *Tiger) Accept(visitor IAnimalOperation) {
	visitor.VisitTiger(t)
}

type Dog struct{}

func (d *Dog) Accept(visitor IAnimalOperation) {
	visitor.VisitDog(d)
}

// Concrete Visitor: JumpAnimalOperation

type JumpAnimalOperation struct{}

func (j *JumpAnimalOperation) VisitLion(lion *Lion) {
	fmt.Println("Lion jumps!")
}

func (j *JumpAnimalOperation) VisitTiger(tiger *Tiger) {
	fmt.Println("Tiger jumps!")
}

func (j *JumpAnimalOperation) VisitDog(dog *Dog) {
	fmt.Println("Dog jumps!")
}

// Concrete Visitor: SayAnimalOperation

type SayAnimalOperation struct{}

func (s *SayAnimalOperation) VisitLion(lion *Lion) {
	fmt.Println("Lion says: Roar!")
}

func (s *SayAnimalOperation) VisitTiger(tiger *Tiger) {
	fmt.Println("Tiger says: Growl!")
}

func (s *SayAnimalOperation) VisitDog(dog *Dog) {
	fmt.Println("Dog says: Woof!")
}

// Client
func main() {
	animals := []IAnimal{&Lion{}, &Tiger{}, &Dog{}}

	jumpOp := &JumpAnimalOperation{}
	sayOp := &SayAnimalOperation{}

	for _, animal := range animals {
		animal.Accept(jumpOp)
	}
	// Lion jumps!
	// Tiger jumps!
	// Dog jumps!

	for _, animal := range animals {
		animal.Accept(sayOp)
	}
	// Lion says: Roar!
	// Tiger says: Growl!
	// Dog says: Woof!
}
```

---

## Pitfalls

- Adding a new element type requires updating **every** Visitor interface — costly if the element hierarchy changes often.
- Visitors make sense only when operations change more frequently than the object structure.

---

## Memory Aid

> Visitor = tour guide. The animals (elements) stay in their enclosures. Different tour guides (visitors) come through and do different things — one describes them, another photographs them. The animals just say "I'm a Lion" (`Accept`) and the guide does the rest.
