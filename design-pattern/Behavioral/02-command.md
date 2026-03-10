# Command

> **Category**: Behavioral | **Language**: Go

---

## What

Encapsulate a request as a **stand-alone object**, decoupling the **Invoker** (who asks) from the **Receiver** (who does). Enables queuing, logging, undo/redo, and composite commands.

**4 parts:**

| Part | Role |
|------|------|
| **Receiver** | Object with actual business methods (e.g., `Light`) |
| **Command Interface** | Interface with `Execute()` (and optionally `Undo()`) |
| **ConcreteCommand** | Holds a reference to a Receiver; implements Command Interface by calling Receiver methods |
| **Invoker** | Holds Commands; triggers them without knowing business logic |

---

## Why / When

- **Decoupling** — Invokers don't depend on concrete actions, just on the Command interface.
- **Undo/Redo** — store a history of commands and call their `Undo()`.
- You need to **dynamically choose** which action to perform at runtime.

---

## How

1. **Define the Receiver** — object with actual business methods.
2. **Define the Command Interface** — `Execute()` and optionally `Undo()`.
3. **Define Concrete Commands** — hold a Receiver reference, implement the interface by calling Receiver methods.
4. **Define the Invoker** — holds Commands, triggers them uniformly.
5. **Client wiring** — create receivers, create commands, associate them, create invoker.

---

## Example

```go
package main

import "fmt"

// Receiver
type Light struct {
	IsLight bool
}

func (l *Light) TurnOn()  { l.IsLight = true }
func (l *Light) TurnOff() { l.IsLight = false }

// Command Interface
type ICommand interface {
	Execute()
}

// Concrete Commands
type TurnOnCommand struct {
	light *Light
}

func (c *TurnOnCommand) Execute() {
	c.light.TurnOn()
}

type TurnOffCommand struct {
	light *Light
}

func (c *TurnOffCommand) Execute() {
	c.light.TurnOff()
}

// Invoker
type RemoteControl struct {
	turnOffCommand ICommand
	turnOnCommand  ICommand
}

func (r *RemoteControl) TurnOn()  { r.turnOnCommand.Execute() }
func (r *RemoteControl) TurnOff() { r.turnOffCommand.Execute() }

// Client
func main() {
	light := &Light{}
	turnOnCommand := &TurnOnCommand{light: light}
	turnOffCommand := &TurnOffCommand{light: light}

	remoteControl := &RemoteControl{
		turnOnCommand:  turnOnCommand,
		turnOffCommand: turnOffCommand,
	}

	remoteControl.TurnOn()
	fmt.Println("Light status:", light.IsLight) // true

	remoteControl.TurnOff()
	fmt.Println("Light status:", light.IsLight) // false
}
```

---

## Pitfalls

- Over-engineering simple operations that don't need undo or queuing.
- Forgetting to implement `Undo()` when it is actually needed.
- Command objects that grow too large — they should be thin wrappers, not business logic containers.

---

## Memory Aid

> Command = remote control button. Each button is a small object that knows which device method to call. The remote doesn't know how the TV works — it just presses buttons.
