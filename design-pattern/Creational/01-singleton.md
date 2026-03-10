# Singleton

> **Category**: Creational | **Language**: Go

---

## What

A pattern that ensures a struct/class has **only one instance** and provides **global access** to that instance.

---

## Why

Coordinate shared, stateful resources (e.g., config, connection pool, logger) without passing them everywhere.

---

## When

When a class in your program should have just a **single instance** available to all clients — for example, a single database object shared by different parts of the program.

---

## How

1. Declare a **private** instance variable for the struct.
2. Declare a **public** creation method that:
   - Creates a new object on its **first call** (lazy initialization).
   - Returns the **cached instance** on all subsequent calls.

---

## Example

```go
package main

var dbInstance *Database

type Database struct {
	Name string
}

func getDatabaseInstance(databaseName string) *Database {
	// Initial
	if dbInstance == nil {
		dbInstance = &Database{Name: databaseName}
	}

	// Return cached instance
	return dbInstance
}
```

```go
package main

import "fmt"

func main() {
	// Create a singleton instance
	db1 := getDatabaseInstance("mysql")
	fmt.Println(db1.Name) // "mysql"

	// Try to create another instance
	db2 := getDatabaseInstance("postgres")
	fmt.Println(db2.Name) // Still "mysql" — same instance returned
}
```

---

## Pitfalls

- **Not thread-safe** — the example above races if called from multiple goroutines. Use `sync.Once` in production Go code.
- Singletons introduce **hidden global state** — makes testing harder.
- Overusing Singleton where dependency injection would be cleaner.

---

## Memory Aid

> One instance, one entry point, forever. Like a company CEO — only one at a time, everyone talks to the same person.
