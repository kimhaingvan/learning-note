# Template Method

> **Category**: Behavioral | **Language**: Go

---

## What

**Template Method** defines the **skeleton of an algorithm** in a base function, deferring specific steps to subclasses (or, in Go, to interface implementations). The overall sequence is fixed; only individual steps vary.

---

## Why / When

- When several classes share the same algorithm structure but differ in certain steps.
- When you want to let clients extend only specific steps of an algorithm, not the whole thing.

---

## How

1. **Define an interface** with the varying steps (Go has no abstract classes, so use an interface).
2. **Write a top-level template function** that accepts the interface — this function defines the fixed algorithm skeleton.
3. **Implement concrete types** — each provides its own version of the varying steps.

---

## Example

```go
package main

import "fmt"

// Interface for the varying steps
type Exporter interface {
	ReadData() string
	FormatData(raw string) string
	WriteData(formatted string)
}

// Template function — defines the fixed algorithm skeleton
func Export(e Exporter) {
	raw := e.ReadData()
	formatted := e.FormatData(raw)
	e.WriteData(formatted)
}

// ── Concrete Implementation: CSV ──

type CSVExporter struct{}

func (c *CSVExporter) ReadData() string {
	fmt.Println("[CSV] Reading raw records from database...")
	return "Alice,30\nBob,25"
}

func (c *CSVExporter) FormatData(raw string) string {
	fmt.Println("[CSV] Formatting as CSV...")
	return "name,age\n" + raw
}

func (c *CSVExporter) WriteData(formatted string) {
	fmt.Println("[CSV] Writing to output.csv:")
	fmt.Println(formatted)
}

// ── Concrete Implementation: JSON ──

type JSONExporter struct{}

func (j *JSONExporter) ReadData() string {
	fmt.Println("[JSON] Reading raw records from database...")
	return `[{"name":"Alice","age":30},{"name":"Bob","age":25}]`
}

func (j *JSONExporter) FormatData(raw string) string {
	fmt.Println("[JSON] Pretty-printing JSON...")
	return "{\n  \"users\": " + raw + "\n}"
}

func (j *JSONExporter) WriteData(formatted string) {
	fmt.Println("[JSON] Writing to output.json:")
	fmt.Println(formatted)
}

// Client
func main() {
	fmt.Println("=== CSV Export ===")
	Export(&CSVExporter{})

	fmt.Println()

	fmt.Println("=== JSON Export ===")
	Export(&JSONExporter{})
}

// === CSV Export ===
// [CSV] Reading raw records from database...
// [CSV] Formatting as CSV...
// [CSV] Writing to output.csv:
// name,age
// Alice,30
// Bob,25
//
// === JSON Export ===
// [JSON] Reading raw records from database...
// [JSON] Pretty-printing JSON...
// [JSON] Writing to output.json:
// {
//   "users": [{"name":"Alice","age":30},{"name":"Bob","age":25}]
// }
```

---

## Template Method vs Strategy

| Aspect | Template Method | Strategy |
|--------|-----------------|----------|
| **Level** | Class-level (inheritance / interface embedding) | Object-level (composition) |
| **What varies** | Individual steps within a fixed skeleton | The entire algorithm |
| **Control** | Template function controls the flow | Client controls which strategy to use |

---

## Pitfalls

- In Go, there's no inheritance — the "template" is a standalone function, which can feel less natural than in OOP languages.
- Adding a new step to the template function forces all implementations to add it too (interface breaking change).
- If steps have no shared structure, you probably want Strategy instead.

---

## Memory Aid

> Template Method = cooking recipe. The recipe (template) says: first prep, then cook, then plate. *How* you prep and cook depends on the dish — but the order is always the same.
