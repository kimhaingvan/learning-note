# Observer

> **Category**: Behavioral | **Language**: Go

---

## What

**Observer** (a.k.a. **Pub/Sub**) defines a one-to-many dependency so that when one object changes state, all dependent objects are **notified and updated automatically**.

**2 key roles:**

| Role | Responsibility |
|------|----------------|
| **Publisher** | Maintains a list of subscribers and notifies them when state changes |
| **Subscriber** | Registers interest in specific events and reacts when notified |

---

## Why / When

- When an event in one object must trigger updates in multiple other objects, and you don't know how many or which ones ahead of time.
- When some objects should observe others for a limited time or at runtime.

---

## How

1. **Define the Subscriber interface** — `HandleEvent(eventName, body string)`.
2. **Define the Publisher interface** — `Subscribe`, `Unsubscribe`, `Notify`.
3. **Implement the Publisher** — uses an `event → []Subscriber` map.
4. **Implement Concrete Subscribers** — each reacts to events differently.
5. **Wire up and trigger** — subscribe, then publish events.

---

## Example

```go
package main

import "fmt"

// Subscriber interface
type ISubscriber interface {
	HandleEvent(eventName, body string)
}

// Publisher interface
type IPublisher interface {
	Subscribe(eventName string, subscriber ISubscriber)
	Unsubscribe(eventName string, subscriber ISubscriber)
	Notify(eventName string, body string)
}

// Concrete Publisher
type EventManager struct {
	listeners map[string][]ISubscriber
}

func NewEventManager() *EventManager {
	return &EventManager{
		listeners: make(map[string][]ISubscriber),
	}
}

func (e *EventManager) Subscribe(eventName string, subscriber ISubscriber) {
	e.listeners[eventName] = append(e.listeners[eventName], subscriber)
}

func (e *EventManager) Unsubscribe(eventName string, subscriber ISubscriber) {
	subs := e.listeners[eventName]
	for i, s := range subs {
		if s == subscriber {
			e.listeners[eventName] = append(subs[:i], subs[i+1:]...)
			return
		}
	}
}

func (e *EventManager) Notify(eventName, body string) {
	for _, sub := range e.listeners[eventName] {
		sub.HandleEvent(eventName, body)
	}
}

// Concrete Subscribers
type Logger struct{ Name string }

func (l *Logger) HandleEvent(eventName, body string) {
	fmt.Printf("[%s] Logged event '%s': %s\n", l.Name, eventName, body)
}

type Emailer struct{ Recipient string }

func (e *Emailer) HandleEvent(eventName, body string) {
	fmt.Printf("Email to %s — event '%s': %s\n", e.Recipient, eventName, body)
}

// Client
func main() {
	manager := NewEventManager()

	logger := &Logger{Name: "AuditLog"}
	emailer := &Emailer{Recipient: "admin@example.com"}

	manager.Subscribe("user.created", logger)
	manager.Subscribe("user.created", emailer)
	manager.Subscribe("order.placed", logger)

	manager.Notify("user.created", "User alice joined")
	// [AuditLog] Logged event 'user.created': User alice joined
	// Email to admin@example.com — event 'user.created': User alice joined

	manager.Notify("order.placed", "Order #42 placed")
	// [AuditLog] Logged event 'order.placed': Order #42 placed

	manager.Unsubscribe("user.created", emailer)
	manager.Notify("user.created", "User bob joined")
	// [AuditLog] Logged event 'user.created': User bob joined
}
```

---

## Pitfalls

- **Memory leaks** — subscribers that are never unsubscribed keep receiving events and prevent garbage collection.
- **Notification order** — don't depend on the order subscribers are called.
- **Cascading updates** — subscriber A handles an event by firing another event that subscriber B handles, which fires another event … leading to infinite loops.

---

## Memory Aid

> Observer = newspaper subscription. Subscribe to topics you care about. When a new edition is published, every subscriber gets their copy automatically.
