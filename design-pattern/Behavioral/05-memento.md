# Memento

> **Category**: Behavioral | **Language**: Go

---

## What

**Memento** lets you save and restore the previous state of an object **without revealing the details** of its implementation. The object itself creates snapshots of its own state.

**3 parts:**

| Part | Role |
|------|------|
| **Originator** | The object whose state we want to save. Can create and restore from a Memento. |
| **Memento** | Lightweight snapshot of the Originator's state. Immutable to the outside. |
| **Caretaker** | Manages a stack of Mementos. Never inspects or modifies their contents. |

---

## Why / When

- **Undo/Redo functionality** — snapshot at various points, roll back to any previous state.
- You want to save/restore state without exposing internal implementation details.

---

## How

1. **Define the Originator** — the class whose state we want to save.
2. **Define the Memento** — hold the snapshot fields (unexported in Go). No setters, only getters.
3. **Implement Originator methods** — `CreateSnapshot()` and `Restore(m *Memento)`.
4. **Define the Caretaker** — holds a slice/stack of Mementos, with push (`AddSnapshot`) and pop (`GetPreviousSnapshot`).

---

## Example

```go
package main

import "fmt"

// Originator
type Editor struct {
	state string
}

func (o *Editor) Show() {
	fmt.Println("Current State:", o.state)
}

func (o *Editor) SetState(state string) {
	o.state = state
}

func (o *Editor) CreateSnapshot() *Memento {
	return NewMemento(o.state)
}

func (o *Editor) Restore(m *Memento) {
	if m == nil {
		return
	}
	o.state = m.GetSavedState()
}

// Memento — immutable snapshot
type Memento struct {
	savedState string // unexported: only Originator reads via getter
}

func NewMemento(state string) *Memento {
	return &Memento{savedState: state}
}

func (m *Memento) GetSavedState() string {
	return m.savedState
}

// Caretaker — manages snapshot history
type Caretaker struct {
	mementos []*Memento
}

func (c *Caretaker) AddSnapshot(m *Memento) {
	c.mementos = append(c.mementos, m)
}

func (c *Caretaker) GetPreviousSnapshot() *Memento {
	if len(c.mementos) <= 1 {
		c.mementos = []*Memento{}
		return nil
	}
	c.mementos = c.mementos[:len(c.mementos)-1]
	return c.mementos[len(c.mementos)-1]
}

// Client
func main() {
	caretaker := &Caretaker{}
	editor := &Editor{}

	editor.SetState("State 1")
	editor.Show()
	caretaker.AddSnapshot(editor.CreateSnapshot())

	editor.SetState("State 2")
	editor.Show()
	caretaker.AddSnapshot(editor.CreateSnapshot())

	editor.SetState("State 3")
	editor.Show()
	caretaker.AddSnapshot(editor.CreateSnapshot())

	fmt.Println("Restoring to previous state...")
	editor.Restore(caretaker.GetPreviousSnapshot())
	editor.Show()

	fmt.Println("Restoring to previous state...")
	editor.Restore(caretaker.GetPreviousSnapshot())
	editor.Show()
}

// Current State: State 1
// Current State: State 2
// Current State: State 3
// Restoring to previous state...
// Current State: State 2
// Restoring to previous state...
// Current State: State 1
```

---

## Pitfalls

- Storing too many snapshots can consume a lot of memory — implement a max history limit.
- Memento fields must be copies, not references — otherwise restoring a snapshot and then modifying the Originator mutates the snapshot too.

---

## Memory Aid

> Memento = Ctrl+Z history. Each snapshot is a save point. The Caretaker is the undo stack. The Editor is the document.
