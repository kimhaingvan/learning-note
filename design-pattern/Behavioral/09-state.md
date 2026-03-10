# State

> **Category**: Behavioral | **Language**: Go

---

## What

**State** lets an object alter its behavior when its internal state changes. The object appears to change its class. Each state is encapsulated in its own struct, and state transitions are handled by the states themselves.

---

## Why / When

- When an object behaves differently depending on its current state, and the number of states is large.
- When state-specific code is scattered across multiple `if/else` or `switch` blocks.

---

## How

1. **Define the State interface** — methods for every action the context can perform.
2. **Create Concrete States** — each implements the interface; each method either acts or transitions the context to another state.
3. **Create the Context** — holds a current State; delegates all actions to it.
4. **States transition the context** — inside their methods, they call `context.SetState(nextState)`.

---

## Example

```go
package main

import "fmt"

// State interface
type CoffeeMachineByState interface {
	InsertCoin(machine *CoffeeMachine)
	SelectCoffee(machine *CoffeeMachine)
	Brew(machine *CoffeeMachine)
}

// Context
type CoffeeMachine struct {
	state CoffeeMachineByState
}

func NewCoffeeMachine() *CoffeeMachine {
	machine := &CoffeeMachine{}
	machine.state = &FreeState{}
	return machine
}

func (m *CoffeeMachine) SetState(state CoffeeMachineByState) {
	m.state = state
}

func (m *CoffeeMachine) InsertCoin()    { m.state.InsertCoin(m) }
func (m *CoffeeMachine) SelectCoffee()  { m.state.SelectCoffee(m) }
func (m *CoffeeMachine) Brew()          { m.state.Brew(m) }

// ── Concrete States ──

// FreeState — waiting for a coin
type FreeState struct{}

func (s *FreeState) InsertCoin(machine *CoffeeMachine) {
	fmt.Println("Coin inserted. Please select your coffee.")
	machine.SetState(&SelectCoffeeState{})
}

func (s *FreeState) SelectCoffee(machine *CoffeeMachine) {
	fmt.Println("Please insert a coin first.")
}

func (s *FreeState) Brew(machine *CoffeeMachine) {
	fmt.Println("Please insert a coin first.")
}

// SelectCoffeeState — coin in, waiting for selection
type SelectCoffeeState struct{}

func (s *SelectCoffeeState) InsertCoin(machine *CoffeeMachine) {
	fmt.Println("Coin already inserted. Please select your coffee.")
}

func (s *SelectCoffeeState) SelectCoffee(machine *CoffeeMachine) {
	fmt.Println("Coffee selected. Brewing now...")
	machine.SetState(&DeliverCoffeeState{})
}

func (s *SelectCoffeeState) Brew(machine *CoffeeMachine) {
	fmt.Println("Please select a coffee first.")
}

// DeliverCoffeeState — brewing and delivering
type DeliverCoffeeState struct{}

func (s *DeliverCoffeeState) InsertCoin(machine *CoffeeMachine) {
	fmt.Println("Please wait, delivering your coffee.")
}

func (s *DeliverCoffeeState) SelectCoffee(machine *CoffeeMachine) {
	fmt.Println("Already brewing. Please wait.")
}

func (s *DeliverCoffeeState) Brew(machine *CoffeeMachine) {
	fmt.Println("Here is your coffee! ☕")
	machine.SetState(&FreeState{})
}

// Client
func main() {
	machine := NewCoffeeMachine()

	machine.SelectCoffee() // Please insert a coin first.
	machine.Brew()         // Please insert a coin first.

	machine.InsertCoin()   // Coin inserted. Please select your coffee.
	machine.InsertCoin()   // Coin already inserted. Please select your coffee.

	machine.SelectCoffee() // Coffee selected. Brewing now...
	machine.Brew()         // Here is your coffee! ☕

	machine.InsertCoin()   // Coin inserted. Please select your coffee.
}
```

---

## State vs Strategy

| Aspect | State | Strategy |
|--------|-------|----------|
| **Intent** | Object behavior changes as internal state changes | Client picks an algorithm at runtime |
| **Transitions** | States know about and transition to each other | Strategies are unaware of each other |
| **Who decides** | States transition themselves | Client sets the strategy |

---

## Pitfalls

- State explosion — too many states make the pattern unwieldy. Consider a state machine library instead.
- States holding mutable data can create subtle bugs during transitions.

---

## Memory Aid

> State = vending machine. Insert coin → new state. Select item → new state. Dispense → back to idle. Each state only allows certain actions and decides the next state.
