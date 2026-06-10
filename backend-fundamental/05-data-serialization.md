# Data Serialization

> **Category**: Backend Fundamental | **Level**: Intermediate

---

## Table of Contents

1. [What](#1-what)
2. [JSON](#2-json)
3. [XML](#3-xml)
4. [Protocol Buffers (Protobuf)](#4-protocol-buffers-protobuf)
5. [Comparison](#5-comparison)
6. [When to Use What](#6-when-to-use-what)

---

## 1. What

**Data serialization** is the process of converting in-memory data structures (objects, structs) into a format that can be **stored** or **transmitted**, and later **deserialized** back.

```
Struct/Object → [Serialize] → Bytes/Text → [Transmit/Store] → [Deserialize] → Struct/Object
```

---

## 2. JSON

### What

**JSON (JavaScript Object Notation)** is a lightweight, human-readable text format for data interchange. It is the de facto standard for REST APIs.

### Format

```json
{
  "name": "Alice",
  "age": 30,
  "active": true,
  "roles": ["admin", "user"],
  "address": {
    "city": "Hanoi",
    "country": "Vietnam"
  }
}
```

### Data Types

| Type | Example |
|------|---------|
| String | `"hello"` |
| Number | `42`, `3.14` |
| Boolean | `true`, `false` |
| Null | `null` |
| Array | `[1, 2, 3]` |
| Object | `{"key": "value"}` |

### Go Example

```go
package main

import (
    "encoding/json"
    "fmt"
)

type User struct {
    Name   string   `json:"name"`
    Age    int      `json:"age"`
    Active bool     `json:"active"`
    Roles  []string `json:"roles"`
}

func main() {
    // Serialize (Marshal)
    user := User{Name: "Alice", Age: 30, Active: true, Roles: []string{"admin"}}
    data, _ := json.Marshal(user)
    fmt.Println(string(data))
    // {"name":"Alice","age":30,"active":true,"roles":["admin"]}

    // Deserialize (Unmarshal)
    var decoded User
    json.Unmarshal(data, &decoded)
    fmt.Printf("%+v\n", decoded)
    // {Name:Alice Age:30 Active:true Roles:[admin]}
}
```

### Pros & Cons

| Pros | Cons |
|------|------|
| Human-readable | No schema enforcement |
| Widely supported | No comments allowed |
| Simple and lightweight | No binary data support (must Base64 encode) |
| Native in JavaScript | Verbose for large datasets |

---

## 3. XML

### What

**XML (eXtensible Markup Language)** is a text-based format that uses tags to define structure. It supports schemas (XSD), namespaces, and attributes.

### Format

```xml
<?xml version="1.0" encoding="UTF-8"?>
<user>
    <name>Alice</name>
    <age>30</age>
    <active>true</active>
    <roles>
        <role>admin</role>
        <role>user</role>
    </roles>
    <address city="Hanoi" country="Vietnam"/>
</user>
```

### Go Example

```go
package main

import (
    "encoding/xml"
    "fmt"
)

type User struct {
    XMLName xml.Name `xml:"user"`
    Name    string   `xml:"name"`
    Age     int      `xml:"age"`
    Active  bool     `xml:"active"`
}

func main() {
    // Serialize
    user := User{Name: "Alice", Age: 30, Active: true}
    data, _ := xml.MarshalIndent(user, "", "  ")
    fmt.Println(string(data))

    // Deserialize
    var decoded User
    xml.Unmarshal(data, &decoded)
    fmt.Printf("%+v\n", decoded)
}
```

### Pros & Cons

| Pros | Cons |
|------|------|
| Schema validation (XSD) | Very verbose |
| Namespaces for avoiding conflicts | Slower to parse than JSON |
| Attributes + elements | Complex to read/write |
| Mature tooling (XSLT, XPath) | Falling out of favor for APIs |

---

## 4. Protocol Buffers (Protobuf)

### What

**Protocol Buffers (Protobuf)** is a **binary** serialization format developed by **Google**. It is schema-defined, strongly typed, and much more compact and faster than JSON/XML.

### How

1. **Define** your data structure in a `.proto` file.
2. **Compile** the `.proto` file using `protoc` to generate code for your language.
3. **Use** the generated code to serialize/deserialize.

### Proto File

```protobuf
syntax = "proto3";

package user;

message User {
    string name = 1;    // field number, not default value
    int32 age = 2;
    bool active = 3;
    repeated string roles = 4;  // repeated = array/list

    Address address = 5;
}

message Address {
    string city = 1;
    string country = 2;
}
```

### Field Numbers

- Each field has a **unique number** (not a value) — used in binary encoding.
- Numbers **1–15** use 1 byte → use for frequently used fields.
- Numbers **16–2047** use 2 bytes.
- **Never reuse** a field number after removing a field.

### Binary Encoding

Protobuf encodes data as `field_number + wire_type + value`:

| Wire Type | Meaning | Used For |
|-----------|---------|----------|
| 0 | Varint | int32, int64, bool, enum |
| 1 | 64-bit | fixed64, double |
| 2 | Length-delimited | string, bytes, messages, repeated |
| 5 | 32-bit | fixed32, float |

### Go Example

```go
// After running: protoc --go_out=. user.proto

package main

import (
    "fmt"
    "google.golang.org/protobuf/proto"
    pb "myapp/proto/user" // generated package
)

func main() {
    // Serialize
    user := &pb.User{
        Name:   "Alice",
        Age:    30,
        Active: true,
        Roles:  []string{"admin"},
    }
    data, _ := proto.Marshal(user)
    fmt.Printf("Binary size: %d bytes\n", len(data)) // ~20 bytes vs ~60 for JSON

    // Deserialize
    var decoded pb.User
    proto.Unmarshal(data, &decoded)
    fmt.Println(decoded.GetName()) // Alice
}
```

### Pros & Cons

| Pros | Cons |
|------|------|
| Very compact (binary) | Not human-readable |
| Very fast serialize/deserialize | Requires schema (.proto) and code generation |
| Strong typing + schema evolution | Extra build step (protoc) |
| Cross-language support | Debugging is harder (binary format) |
| Backward/forward compatible | Steeper learning curve |

---

## 5. Comparison

| Feature | JSON | XML | Protobuf |
|---------|------|-----|----------|
| **Format** | Text | Text | Binary |
| **Human-readable** | ✅ Yes | ✅ Yes | ❌ No |
| **Schema** | Optional (JSON Schema) | XSD | Required (.proto) |
| **Size** | Medium | Large | Small (~3-10x smaller than JSON) |
| **Parse speed** | Medium | Slow | Fast (~5-100x faster than JSON) |
| **Typing** | Weak | Strong (with XSD) | Strong |
| **Language support** | Universal | Universal | Requires code generation |
| **Schema evolution** | Fragile | Fragile | Built-in (field numbers) |
| **Use case** | REST APIs, config | SOAP, enterprise | gRPC, microservices, high-perf |

### Size Comparison (same data)

```
JSON:    {"name":"Alice","age":30,"active":true}    → 42 bytes
XML:     <user><name>Alice</name><age>30</age>...</user>  → ~80 bytes
Protobuf: (binary)                                  → ~12 bytes
```

---

## 6. When to Use What

| Scenario | Recommended |
|----------|-------------|
| Public REST API | **JSON** |
| Browser/frontend communication | **JSON** |
| Configuration files | **JSON** or YAML |
| Enterprise/legacy systems | **XML** |
| Microservice-to-microservice (gRPC) | **Protobuf** |
| High-performance, low-latency | **Protobuf** |
| Mobile apps (bandwidth-sensitive) | **Protobuf** |
| Streaming data | **Protobuf** |
