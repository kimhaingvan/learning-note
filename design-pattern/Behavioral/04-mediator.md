# Mediator

> **Category**: Behavioral | **Language**: Go

---

## What

**Mediator** reduces chaotic dependencies between objects by forcing them to communicate only through a **Mediator interface** instead of directly with each other.

**3 parts:**

| Part | Role |
|------|------|
| **Mediator Interface** | Methods for communication (e.g., `RequestLanding`, `Notify`) |
| **Concrete Mediator** | Coordinator that implements the interface and manages interactions |
| **Colleague Classes** | Objects that interact **only** via the Mediator, never directly |

---

## Why

Reduces inter-object dependencies — colleagues don't need references to every other colleague.

---

## When

- When it's hard to change some classes because they are **tightly coupled** to many others.
- When you can't reuse a component because it depends on too many other components.

---

## How

1. **Identify tightly coupled classes** that would benefit from being more independent.
2. **Declare the Mediator interface** — communication methods between objects.
3. **Implement the Concrete Mediator** — coordinates requests and maintains shared state.
4. **Colleague classes hold a Mediator reference** — all communication goes through it.

---

## Example

```go
package main

import "fmt"

// Mediator interface
type IMediator interface {
	RequestLanding(plane IPlane) bool
	RequestTakeOff(plane IPlane) bool
}

// Colleague interface
type IPlane interface {
	Land()
	TakeOff()
}

// Concrete Mediator
type ControlTower struct {
	isPlatformFree bool
	planeQueue     []IPlane
}

func (c *ControlTower) RequestLanding(plane IPlane) bool {
	if c.isPlatformFree {
		c.isPlatformFree = false
		return true
	}
	c.planeQueue = append(c.planeQueue, plane)
	return false
}

func (c *ControlTower) RequestTakeOff(plane IPlane) bool {
	if !c.isPlatformFree {
		c.isPlatformFree = true
		if len(c.planeQueue) > 0 {
			next := c.planeQueue[0]
			c.planeQueue = c.planeQueue[1:]
			next.Land()
		}
		return true
	}
	return false
}

// Colleague: Aircraft
type Aircraft struct {
	id, name string
	tower    IMediator
}

func NewAircraft(id, name string, tower IMediator) IPlane {
	return &Aircraft{id: id, name: name, tower: tower}
}

func (c *Aircraft) Land() {
	if c.tower.RequestLanding(c) {
		fmt.Println("Aircraft", c.name, "is landing.")
	} else {
		fmt.Println("Platform busy, aircraft", c.name, "is waiting.")
	}
}

func (c *Aircraft) TakeOff() {
	if c.tower.RequestTakeOff(c) {
		fmt.Println("Aircraft", c.name, "has taken off.")
	} else {
		fmt.Println("Platform busy, aircraft", c.name, "waiting to take off.")
	}
}

// Colleague: Drone
type Drone struct {
	id, name string
	tower    IMediator
}

func NewDrone(id, name string, tower IMediator) IPlane {
	return &Drone{id: id, name: name, tower: tower}
}

func (c *Drone) Land() {
	if c.tower.RequestLanding(c) {
		fmt.Println("Drone", c.name, "is landing.")
	} else {
		fmt.Println("Platform busy, Drone", c.name, "is waiting.")
	}
}

func (c *Drone) TakeOff() {
	if c.tower.RequestTakeOff(c) {
		fmt.Println("Drone", c.name, "has taken off.")
	} else {
		fmt.Println("Platform busy, Drone", c.name, "waiting.")
	}
}

// Client
func main() {
	tower := &ControlTower{
		isPlatformFree: true,
		planeQueue:     make([]IPlane, 0),
	}

	aircraft1 := NewAircraft("1", "Boeing 747", tower)
	drone1 := NewDrone("5", "DJI Phantom", tower)
	aircraft2 := NewAircraft("2", "Airbus A320", tower)

	aircraft1.Land()  // Boeing 747 is landing.
	drone1.Land()     // Platform busy, DJI Phantom is waiting.
	aircraft2.Land()  // Platform busy, Airbus A320 is waiting.
	aircraft1.TakeOff() // Boeing 747 has taken off. → DJI Phantom is landing.
}
```

---

## Pitfalls

- Mediator becoming a "god object" — keep it focused on coordination, not business logic.
- Circular callbacks — Mediator notifies colleague, colleague calls Mediator again.

---

## Memory Aid

> Mediator = air traffic control tower. Planes never talk to each other — they all talk to the tower, and the tower coordinates.
