# Factory Method

> **Category**: Creational | **Language**: Go

---

## What

A function that returns an object through an **interface**, letting the caller choose **which concrete struct** they get — not via hard-coded values, but via argument values.

---

## Why

- Avoid repeated constructor boilerplate.
- Add new variants without touching callers (just extend the factory).

---

## When

Multiple implementations share the same behavior (e.g., `BadmintonAthlete` and `FootballAthlete`).

---

## How

1. **Define a common interface** with a set of behaviors/functions.
2. **Create concrete structs** that implement the interface.
3. **Create a factory method** that returns the interface.
4. **Add a parameter** to the factory method to choose which concrete implementation to return.

---

## Example

```go
package main

import "fmt"

// Step 1: Common interface
type IAthlete interface {
	Run()
	GetPower() int
}

// Step 2: Concrete implementations
type FootballAthlete struct {
	Scored int
}

func (s *FootballAthlete) Run() {
	fmt.Println("Football Running")
}

func (s *FootballAthlete) GetPower() int {
	return s.Scored
}

type BadmintonAthlete struct {
	Smash int
}

func (s *BadmintonAthlete) Run() {
	fmt.Println("Badminton Running")
}

func (s *BadmintonAthlete) GetPower() int {
	return s.Smash
}

// Step 3 + 4: Factory method with parameter
func AthleteFactory(name string) (athlete IAthlete) {
	switch name {
	case "football":
		athlete = &FootballAthlete{Scored: 500}
	case "badminton":
		athlete = &BadmintonAthlete{Smash: 500}
	}
	return
}
```

```go
func main() {
	footballAthlete := AthleteFactory("football")
	badmintonAthlete := AthleteFactory("badminton")

	footballAthlete.Run()              // Football Running
	badmintonAthlete.Run()             // Badminton Running
	fmt.Println(footballAthlete.GetPower())  // 500
	fmt.Println(badmintonAthlete.GetPower()) // 500
}
```

---

## Pitfalls

- Factory grows into a big `switch` as you add types — consider a registry map instead.
- Returning `nil` silently when the parameter does not match any type — always handle the default case.

---

## Memory Aid

> Factory Method = "Tell me what you want, I'll build the right one." Caller picks by name, factory picks by type.
