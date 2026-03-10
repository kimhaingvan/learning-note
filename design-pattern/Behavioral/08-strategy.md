# Strategy

> **Category**: Behavioral | **Language**: Go

---

## What

**Strategy** defines a family of algorithms, encapsulates each one behind an interface, and makes them **interchangeable at runtime**. The client (context) delegates work to the strategy object instead of implementing multiple algorithms itself.

---

## Why / When

- When you want to switch algorithms at runtime (e.g., different payment methods, sort algorithms, compression formats).
- When you have multiple classes that differ only in how they execute some behavior.

---

## How

1. **Define the Strategy interface** — a single method representing the algorithm.
2. **Implement Concrete Strategies** — each provides a different algorithm.
3. **Create the Context** — holds a Strategy reference and delegates to it.
4. **Client sets the strategy** at runtime.

---

## Example

```go
package main

import "fmt"

// Strategy interface
type PaymentStrategy interface {
	Pay(amount float64) string
}

// Concrete Strategy: Credit Card
type CreditCardPayment struct {
	CardNumber string
}

func (c *CreditCardPayment) Pay(amount float64) string {
	return fmt.Sprintf("Paid $%.2f via Credit Card ending %s",
		amount, c.CardNumber[len(c.CardNumber)-4:])
}

// Concrete Strategy: PayPal
type PayPalPayment struct {
	Email string
}

func (p *PayPalPayment) Pay(amount float64) string {
	return fmt.Sprintf("Paid $%.2f via PayPal (%s)", amount, p.Email)
}

// Context
type PaymentProcessor struct {
	strategy PaymentStrategy
}

func (pp *PaymentProcessor) SetStrategy(strategy PaymentStrategy) {
	pp.strategy = strategy
}

func (pp *PaymentProcessor) Checkout(amount float64) string {
	if pp.strategy == nil {
		return "No payment method selected"
	}
	return pp.strategy.Pay(amount)
}

// Client
func main() {
	processor := &PaymentProcessor{}

	// Pay with Credit Card
	processor.SetStrategy(&CreditCardPayment{CardNumber: "4111111111111234"})
	fmt.Println(processor.Checkout(99.99))
	// Paid $99.99 via Credit Card ending 1234

	// Switch to PayPal at runtime
	processor.SetStrategy(&PayPalPayment{Email: "user@example.com"})
	fmt.Println(processor.Checkout(49.50))
	// Paid $49.50 via PayPal (user@example.com)
}
```

---

## Strategy vs Bridge

| Aspect | Strategy | Bridge |
|--------|----------|--------|
| **Intent** | Swap algorithms at runtime | Separate abstraction from implementation |
| **Scope** | One behavior dimension | Two independently varying dimensions |
| **Change** | Context delegates to a strategy | Abstraction and implementor evolve independently |

---

## Pitfalls

- If you only have two strategies that rarely change, an `if/else` is simpler.
- Clients must know enough about strategies to pick the right one — the selector logic has to live somewhere.

---

## Memory Aid

> Strategy = GPS route options. Same destination, different algorithms: fastest, shortest, scenic. Pick one at departure time, and the navigator follows that strategy.
