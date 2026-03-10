# Prototype

> **Category**: Creational | **Language**: Go

---

## What

**Prototype** lets you copy existing objects without making your code dependent on their concrete structs or classes.

---

## Why

- Some fields may be private and not visible from outside — you cannot copy them manually.
- Cloning avoids re-running expensive initialization logic.
- The client does not need to know the concrete type to make a copy.

---

## When

- You need an exact copy of an object but cannot (or should not) depend on its concrete type.
- Creating an object from scratch is expensive; cloning a pre-built prototype is cheaper.

---

## How

1. **Define the prototype interface** — declare the `clone()` method.
2. **Create concrete structs** — all implement the prototype interface including `clone()`.

---

## Example

```go
package main

import "fmt"

// Step 1: Prototype interface
type Inode interface {
	print(string)
	clone() Inode
}

// Step 2: Concrete structs

type File struct {
	name string
}

func (f *File) print(indentation string) {
	fmt.Println(indentation + f.name)
}

func (f *File) clone() Inode {
	return &File{name: f.name + "_clone"}
}

type Folder struct {
	children []Inode
	name     string
}

func (f *Folder) print(indentation string) {
	fmt.Println(indentation + f.name)
	for _, i := range f.children {
		i.print(indentation + indentation)
	}
}

func (f *Folder) clone() Inode {
	cloneFolder := &Folder{name: f.name + "_clone"}
	var tempChildren []Inode
	for _, i := range f.children {
		copy := i.clone()
		tempChildren = append(tempChildren, copy)
	}
	cloneFolder.children = tempChildren
	return cloneFolder
}
```

```go
func main() {
	file1 := &File{name: "File1"}
	file2 := &File{name: "File2"}
	file3 := &File{name: "File3"}

	folder1 := &Folder{
		children: []Inode{file1},
		name:     "Folder1",
	}

	folder2 := &Folder{
		children: []Inode{folder1, file2, file3},
		name:     "Folder2",
	}
	fmt.Println("\nPrinting hierarchy for Folder2")
	folder2.print("  ")

	cloneFolder := folder2.clone()
	fmt.Println("\nPrinting hierarchy for clone Folder")
	cloneFolder.print("  ")
}

// Printing hierarchy for Folder2
//   Folder2
//     Folder1
//         File1
//     File2
//     File3
//
// Printing hierarchy for clone Folder
//   Folder2_clone
//     Folder1_clone
//         File1_clone
//     File2_clone
//     File3_clone
```

---

## Pitfalls

- **Shallow vs deep clone** — if you only copy pointer fields, both original and clone share the same underlying data. Always deep-clone nested objects (as shown in the `Folder.clone()` above).
- Circular references make cloning tricky — can cause infinite recursion.

---

## Memory Aid

> Prototype = "copy yourself." Each object knows how to duplicate itself, including its private internals.
