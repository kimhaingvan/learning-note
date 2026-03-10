# Composite

> **Category**: Structural | **Language**: Go

---

## What

**Composite** lets you compose objects into **tree structures** and work with them as if they were individual objects.

**3 parts:**

| Part | Role |
|------|------|
| **Component** | Common interface for all objects in the tree |
| **Leaf** | Concrete object with no children |
| **Composite** | Concrete object that holds zero or more Components (leaf or other composites) |

---

## Why / When

- The core model of your app can be represented as a **tree**.
- You want client code to treat both simple (Leaf) and complex (Composite) elements **uniformly**.

---

## How

1. **Define the Component interface** — methods that make sense for both simple and complex elements.
2. **Implement Leaf objects** — simple elements, no children.
3. **Implement Composite objects** — can hold children, has `add`/`remove` methods, delegates work to children.
4. **Client operates through the Component interface.**

---

## Example

```go
package main

import "fmt"

// Component interface
type Component interface {
	search(string)
}

// Leaf
type File struct {
	name string
}

func (f *File) search(keyword string) {
	fmt.Printf("Searching for keyword %s in file %s\n", keyword, f.name)
}

func (f *File) getName() string {
	return f.name
}

// Composite
type Folder struct {
	components []Component
	name       string
}

func (f *Folder) search(keyword string) {
	fmt.Printf("Searching recursively for keyword %s in folder %s\n", keyword, f.name)
	for _, composite := range f.components {
		composite.search(keyword)
	}
}

func (f *Folder) add(c Component) {
	f.components = append(f.components, c)
}

// Client
func main() {
	file1 := &File{name: "File1"}
	file2 := &File{name: "File2"}
	file3 := &File{name: "File3"}

	folder1 := &Folder{name: "Folder1"}
	folder1.add(file1)

	folder2 := &Folder{name: "Folder2"}
	folder2.add(file2)
	folder2.add(file3)
	folder2.add(folder1)

	folder2.search("rose")
}
// Searching recursively for keyword rose in folder Folder2
// Searching for keyword rose in file File2
// Searching for keyword rose in file File3
// Searching recursively for keyword rose in folder Folder1
// Searching for keyword rose in file File1
```

---

## Pitfalls

- Forcing leaf-only methods (like `add`) into the Component interface — violates interface segregation.
- Deeply nested trees can cause stack overflows if recursive calls are unbounded.

---

## Memory Aid

> Composite = tree of uniform nodes. A folder contains files and other folders — treat them all the same way.
