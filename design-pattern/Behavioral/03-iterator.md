# Iterator

> **Category**: Behavioral | **Language**: Go

---

## What

**Iterator** provides a uniform way to traverse elements of a collection **without exposing its underlying representation** (list, stack, tree, etc.).

---

## Why / When

- You have **custom data structures** (trees, graphs, linked lists) and need to walk them uniformly.
- You want to offer **multiple traversal strategies** (forward, reverse, filtered) over the same data.

---

## How

1. **Define the Iterator interface** — methods to fetch next/previous, track position, check end.
2. **Create a concrete Iterator** implementing the interface — holds the collection and cursor.
3. **Client uses the Iterator interface** to traverse.

---

## Example

```go
package main

import "fmt"

type User struct {
	Name string
	Age  int
}

// Iterator interface
type Iterator interface {
	GetNext() *User
	GetPrevious() *User
	HasNext() bool
}

// Concrete Iterator
type UserIterator struct {
	users []*User
	index int
}

func (u *UserIterator) GetNext() *User {
	if u.HasNext() {
		user := u.users[u.index]
		u.index++
		return user
	}
	return nil
}

func (u *UserIterator) GetPrevious() *User {
	if u.index > 0 {
		u.index--
		return u.users[u.index]
	}
	return nil
}

func (u *UserIterator) HasNext() bool {
	return u.index < len(u.users)
}

// Client
func main() {
	user1 := &User{Name: "a", Age: 30}
	user2 := &User{Name: "b", Age: 20}

	userIterator := &UserIterator{
		users: []*User{user1, user2},
		index: 0,
	}

	for userIterator.HasNext() {
		user := userIterator.GetNext()
		fmt.Println(user.Name, user.Age)
	}
	// a 30
	// b 20
}
```

---

## Note on Go

Go's built-in `range` keyword handles most iteration needs. The Iterator pattern is most useful in Go when:
- The data structure is not a slice/map (e.g., a tree, graph, or external paginated API).
- You need stateful, resumable iteration.

---

## Pitfalls

- Using Iterator when a simple `range` loop suffices (over-engineering).
- Not handling concurrent modification — if the collection changes during iteration, results are undefined.

---

## Memory Aid

> Iterator = bookmark in a book. It remembers where you are, lets you go forward or back, and doesn't care whether the book is hardcover or digital.
