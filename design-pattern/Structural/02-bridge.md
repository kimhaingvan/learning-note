# Bridge

> **Category**: Structural | **Language**: Go

---

## What

**Bridge** splits a large class or a set of closely related classes into two separate hierarchies — **data (Abstraction)** and **behavior (Implementation)** — which can be developed independently and integrated with each other.

---

## Key Concepts

| Term | Role |
|------|------|
| **Abstraction** | High-level struct holding data + a reference to an Implementor (the bridge). Defines *what* operations are available. |
| **Implementor** | Interface defining *how* operations are carried out. Provides primitive methods. |
| **Concrete Implementors** | Structs implementing the Implementor interface with different behavior. |
| **Bridge** | The Implementor field inside the Abstraction that connects data to behavior. |

---

## Why / When

- When you want to divide a monolithic class that has several variants of functionality (e.g., `TvBasicRemote`, `RadioBasicRemote`, `TvAdvanceRemote`, `RadioAdvanceRemote`).
- When you need to extend a class in several **orthogonal (independent) dimensions**.

---

## How

1. **Identify the two dimensions**: data (struct fields) vs behavior (methods).
2. **Define the Implementor interface** (behavior).
3. **Create Concrete Implementors**.
4. **Define the Abstraction** — struct holding data + an Implementor reference (the bridge).
5. **Optionally extend the Abstraction** by embedding the base struct.

---

## Example

```go
package main

import "fmt"

// ── Implementor interface (behavior) ──

type Renderer interface {
	Render(data ReportData) string
}

// ── Concrete Implementors ──

type HTMLRenderer struct{}

func (HTMLRenderer) Render(d ReportData) string { return "<html>" + d.Title + "</html>" }

type PDFRenderer struct{}

func (PDFRenderer) Render(d ReportData) string { return "<pdf>" + d.Title + "</pdf>" }

// ── Data ──

type ReportData struct {
	Title   string
	Content string
	Footer  string
}

// ── Abstraction (data + bridge) ──

type Report struct {
	data     ReportData
	renderer Renderer // ← bridge
}

func NewReport(d ReportData, r Renderer) *Report {
	return &Report{data: d, renderer: r}
}

func (r *Report) Print() {
	out := r.renderer.Render(r.data)
	fmt.Println(out)
}

// ── Extended Abstraction ──

type SignedReport struct {
	*Report
	signer string
}

func NewSignedReport(d ReportData, r Renderer, s string) *SignedReport {
	return &SignedReport{Report: NewReport(d, r), signer: s}
}

func (s *SignedReport) Print() {
	s.Report.Print()
	fmt.Println("Signed by:", s.signer)
}

func main() {
	data := ReportData{
		Title: "Monthly", Content: "Sales data...", Footer: "Confidential",
	}

	rpt1 := NewReport(data, HTMLRenderer{})
	rpt1.Print() // <html>Monthly</html>

	rpt2 := NewSignedReport(data, PDFRenderer{}, "Alice")
	rpt2.Print() // <pdf>Monthly</pdf> \n Signed by: Alice
}
```

---

## Bridge vs Strategy

| | Bridge | Strategy |
|---|---|---|
| **Focus** | Separate structure (data) from implementation (behavior) at design time | Swap algorithms at runtime |
| **Abstraction** | Has its own hierarchy that can be extended | Context is typically fixed |
| **Relationship** | Permanent composition | Interchangeable at runtime |

---

## Pitfalls

- Over-engineering simple classes — Bridge adds complexity. Only use when you truly have two independent dimensions.
- Mixing data and behavior in the Implementor (it should only define primitive operations).

---

## Memory Aid

> Bridge = two independent decks connected by a bridge. Data on one side, behavior on the other. Mix and match freely.
