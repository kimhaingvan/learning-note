# Adapter

> **Category**: Structural | **Language**: Go

---

## What

Adapter allows objects with **incompatible interfaces** to collaborate by introducing a middle layer (the Adapter) that translates between them.

---

## Why / When

- You want to convert or translate between two or more objects with incompatible interfaces.
- You cannot (or choose not to) modify the existing class to match your interface.
- Trade-off: increases overall code complexity — sometimes it is simpler to just change the service class directly.

---

## How

**Principle**: The adapter implements the **target interface** and wraps the **adaptee**. The adapter's methods call the adaptee's methods and convert the output.

1. **Define the Target interface** — the interface the client depends on.
2. **Identify the Adaptee** — an existing component with an incompatible API.
3. **Create an Adapter struct** — holds a field of the adaptee type, implements the target interface.
4. **Implement all target methods** — delegate to the adaptee, converting inputs/outputs as needed.
5. **Client uses the adapter via the target interface.**

---

## Example

```go
package main

import "fmt"

// Adaptee — existing component with incompatible interface
type EnglishPerson struct{}

func (w *EnglishPerson) Hello() string {
	return "Hello Viet Nam"
}

// Target interface — what the client expects
type IVietNamPersonAdapter interface {
	XinChao(string) string
}

// Adapter — bridges EnglishPerson to IVietNamPersonAdapter
type EnglishToVietnameseAdapter struct {
	EnglishPerson EnglishPerson
}

func (w *EnglishToVietnameseAdapter) XinChao(msg string) string {
	if w.EnglishPerson.Hello() == msg {
		return "DICH: Xin chao Viet Nam"
	}
	return "Xin chao"
}

// Client uses only the target interface
func main() {
	eng := &EnglishPerson{}

	adapter := &EnglishToVietnameseAdapter{
		EnglishPerson: *eng,
	}

	fmt.Println(adapter.XinChao("Hello Viet Nam"))
	// DICH: Xin chao Viet Nam
}
```

---

## Pitfalls

- Over-adapting — if you need many adapters, the design may need rethinking.
- Adapters that silently swallow or transform data can hide bugs.

---

## Memory Aid

> Adapter = power plug converter. Two incompatible plugs, one middle piece that makes them fit.
