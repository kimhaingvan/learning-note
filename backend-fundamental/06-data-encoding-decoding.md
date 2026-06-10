# Data Encoding and Decoding

> **Category**: Backend Fundamental | **Level**: Intermediate

---

## Table of Contents

1. [What](#1-what)
2. [Base64](#2-base64)
3. [URL Encoding](#3-url-encoding)
4. [Binary Encoding](#4-binary-encoding)
5. [UUID vs ULID](#5-uuid-vs-ulid)
6. [Comparison](#6-comparison)

---

## 1. What

**Encoding** is the process of converting data from one format to another for **safe transmission or storage**. Unlike encryption, encoding is **not for security** — it is **reversible** and uses no secret key.

| Concept | Purpose | Reversible | Key needed |
|---------|---------|------------|------------|
| **Encoding** | Safe transport/storage | ✅ Yes | ❌ No |
| **Encryption** | Confidentiality | ✅ Yes | ✅ Yes |
| **Hashing** | Integrity/verification | ❌ No | ❌ No |

---

## 2. Base64

### What

**Base64** is a **binary-to-text encoding scheme** that represents binary data using **only 64 printable ASCII characters** (`A-Z`, `a-z`, `0-9`, `+`, `/`) plus `=` for padding.

### Why

- Embed binary data (images, files) in text-based formats (JSON, HTML, email).
- Transmit binary data over protocols that only support text (SMTP, HTTP headers).
- Encode cryptographic outputs (keys, hashes, tokens) for safe storage.

### How — Encoding Process

**Example: Encode `"Man"` → `"TWFu"`**

**Step 1**: Convert characters to ASCII decimal values.

| Character | ASCII Decimal |
|-----------|---------------|
| M | 77 |
| a | 97 |
| n | 110 |

**Step 2**: Convert decimal to 8-bit binary.

| Character | Decimal | Binary (8-bit) |
|-----------|---------|----------------|
| M | 77 | `01001101` |
| a | 97 | `01100001` |
| n | 110 | `01101110` |

**Step 3**: Concatenate and split into 6-bit chunks.

```
8-bit: 01001101 01100001 01101110
6-bit: 010011 010110 000101 101110
```

**Step 4**: Convert 6-bit chunks to Base64 characters using the Base64 table.

| 6-bit Binary | Decimal | Base64 Character |
|--------------|---------|------------------|
| 010011 | 19 | T |
| 010110 | 22 | W |
| 000101 | 5 | F |
| 101110 | 46 | u |

**Step 5**: Add padding (`=`) if the input is not a multiple of 3 bytes.

| Input Length mod 3 | Padding |
|--------------------|---------|
| 0 | No padding |
| 1 | `==` (2 padding chars) |
| 2 | `=` (1 padding char) |

**Result**: `"Man"` → `"TWFu"`

### How — Decoding Process

**Example: Decode `"TWFu"` → `"Man"`**

**Step 1**: Convert Base64 characters to decimal values.

| Base64 Character | Decimal | 6-bit Binary |
|------------------|---------|--------------|
| T | 19 | `010011` |
| W | 22 | `010110` |
| F | 5 | `000101` |
| u | 46 | `101110` |

**Step 2**: Concatenate 6-bit values and regroup into 8-bit chunks.

```
6-bit: 010011 010110 000101 101110
8-bit: 01001101 01100001 01101110
```

**Step 3**: Convert 8-bit binary to ASCII characters.

| Binary | Decimal | ASCII |
|--------|---------|-------|
| 01001101 | 77 | M |
| 01100001 | 97 | a |
| 01101110 | 110 | n |

**Result**: `"TWFu"` → `"Man"`

### Size Overhead

Base64 increases data size by ~**33%** (3 bytes input → 4 characters output).

### Variants

| Variant | Characters | Padding | Use Case |
|---------|------------|---------|----------|
| **Standard** | `A-Z a-z 0-9 + /` | `=` | General purpose |
| **URL-safe** | `A-Z a-z 0-9 - _` | Optional | URLs, filenames |
| **No-padding** | Same as above | None | JWTs, compact tokens |

### Go Example

```go
package main

import (
    "encoding/base64"
    "fmt"
)

func main() {
    data := "Hello, World!"

    // Standard Base64
    encoded := base64.StdEncoding.EncodeToString([]byte(data))
    fmt.Println("Encoded:", encoded) // SGVsbG8sIFdvcmxkIQ==

    decoded, _ := base64.StdEncoding.DecodeString(encoded)
    fmt.Println("Decoded:", string(decoded)) // Hello, World!

    // URL-safe Base64
    urlEncoded := base64.URLEncoding.EncodeToString([]byte(data))
    fmt.Println("URL-safe:", urlEncoded)

    // Raw (no padding) — used in JWTs
    rawEncoded := base64.RawURLEncoding.EncodeToString([]byte(data))
    fmt.Println("Raw URL-safe:", rawEncoded)
}
```

---

## 3. URL Encoding

### What

**URL encoding** (percent-encoding) replaces unsafe/reserved characters in a URL with `%` followed by their **hex ASCII value**.

### Why

URLs can only contain a limited set of ASCII characters. Characters like spaces, `&`, `=`, `?`, `#` have special meanings and must be encoded.

### How

```
Character → ASCII hex → %HEX

Space     → 0x20 → %20  (or +)
&         → 0x26 → %26
=         → 0x3D → %3D
?         → 0x3F → %3F
#         → 0x23 → %23
/         → 0x2F → %2F
```

**Example**:
```
Input:  "hello world & go=fun"
Output: "hello%20world%20%26%20go%3Dfun"
```

### Go Example

```go
package main

import (
    "fmt"
    "net/url"
)

func main() {
    // Encode query parameter
    encoded := url.QueryEscape("hello world & go=fun")
    fmt.Println(encoded) // hello+world+%26+go%3Dfun

    // Decode
    decoded, _ := url.QueryUnescape(encoded)
    fmt.Println(decoded) // hello world & go=fun

    // Build a full URL with query params
    u, _ := url.Parse("https://api.example.com/search")
    q := u.Query()
    q.Set("q", "golang concurrency")
    q.Set("page", "1")
    u.RawQuery = q.Encode()
    fmt.Println(u.String()) // https://api.example.com/search?page=1&q=golang+concurrency
}
```

---

## 4. Binary Encoding

### What

**Binary encoding** represents data directly in binary format — compact and efficient for machine-to-machine communication, but not human-readable.

### Common Binary Encoding Schemes

| Scheme | Description | Use Case |
|--------|-------------|----------|
| **Big-endian** | Most significant byte first | Network protocols (network byte order) |
| **Little-endian** | Least significant byte first | x86/ARM CPUs, file formats |
| **Varint** | Variable-length integer encoding | Protobuf, databases |

### Go Example

```go
package main

import (
    "bytes"
    "encoding/binary"
    "fmt"
)

func main() {
    // Write uint32 in Big-Endian
    buf := new(bytes.Buffer)
    binary.Write(buf, binary.BigEndian, uint32(256))
    fmt.Printf("Big-endian:    %x\n", buf.Bytes()) // 00000100

    // Write uint32 in Little-Endian
    buf.Reset()
    binary.Write(buf, binary.LittleEndian, uint32(256))
    fmt.Printf("Little-endian: %x\n", buf.Bytes()) // 00010000

    // Read back
    var value uint32
    binary.Read(bytes.NewReader(buf.Bytes()), binary.LittleEndian, &value)
    fmt.Println("Value:", value) // 256
}
```

---

## 5. UUID vs ULID

### What

Both are standards for generating **unique identifiers** in distributed systems.

| Feature | UUIDv4 | ULID |
|---------|--------|------|
| **Full name** | Universally Unique Identifier | Universally Unique Lexicographically Sortable Identifier |
| **Example** | `d3b07384-d113-4f65-a8c4-9a5a7351dba6` | `01H2GYY4XG8PVNQ9F6H3KC8X5D` |
| **Size** | 128 bits | 128 bits |
| **Format** | Hexadecimal (36 chars with dashes) | Base32 (26 chars) |
| **Sortability** | ❌ No order | ✅ Ordered (time-based) |
| **Timestamp** | ❌ No | ✅ First 48 bits store millisecond time |
| **Randomness** | ✅ Fully random (122 bits) | ✅ Partially random (80 bits) |
| **Readability** | ❌ Hard to read | ✅ More readable, shorter |
| **Database indexing** | ❌ Slower (random → page splits in B-tree) | ✅ Faster (monotonic → sequential inserts) |

### How UUIDv4 Works

```
Generate 128 random bits
Set version bits (bits 48-51 = 0100 for v4)
Set variant bits (bits 64-65 = 10)
Format: xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx
        (y is 8, 9, a, or b)
```

### How ULID Works

```
 01AN4Z07BY      79KA1307SR9X4MV3
|----------|    |----------------|
 Timestamp          Randomness
 48 bits            80 bits
 (ms since epoch)   (random)

Total: 128 bits, encoded in Crockford's Base32 (26 chars)
```

- **Timestamp (first 48 bits)**: Milliseconds since UNIX epoch → enables sorting by creation time.
- **Random part (last 80 bits)**: Ensures uniqueness within the same millisecond.
- **Sorting**: ULIDs generated later are always lexicographically greater.

### When to Use Which

| Use Case | UUID | ULID |
|----------|------|------|
| Distributed unique IDs | ✅ Yes | ✅ Yes |
| Sortable database primary key | ❌ No | ✅ Yes |
| Human-friendly IDs | ❌ No | ✅ Yes |
| Time-ordered events/logs | ❌ No | ✅ Yes |
| Legacy system compatibility | ✅ Yes | ⚠️ Less support |
| Cryptographic randomness | ✅ Yes | ✅ Yes |

### Go Example

```go
package main

import (
    "fmt"
    "github.com/google/uuid"
    "github.com/oklog/ulid/v2"
    "math/rand"
    "time"
)

func main() {
    // UUID v4
    id := uuid.New()
    fmt.Println("UUID:", id.String())
    // UUID: d3b07384-d113-4f65-a8c4-9a5a7351dba6

    // ULID
    entropy := ulid.Monotonic(rand.New(rand.NewSource(time.Now().UnixNano())), 0)
    ulidID := ulid.MustNew(ulid.Timestamp(time.Now()), entropy)
    fmt.Println("ULID:", ulidID.String())
    // ULID: 01H2GYY4XG8PVNQ9F6H3KC8X5D

    // Extract timestamp from ULID
    ts := ulid.Time(ulidID.Time())
    fmt.Println("Created at:", ts)
}
```

> **Tip**: If you use UUIDs as primary keys in PostgreSQL, consider **UUIDv7** (time-ordered UUID, standardized in RFC 9562) as a modern alternative that combines UUID compatibility with ULID-like sortability.

---

## 6. Comparison

| Encoding | Format | Size Overhead | Human-Readable | Use Case |
|----------|--------|---------------|----------------|----------|
| **Base64** | Text (ASCII) | +33% | ✅ Yes | Embed binary in text |
| **URL encoding** | Text (ASCII) | Variable | ✅ Yes | Safe URLs |
| **Hex** | Text (ASCII) | +100% | ✅ Yes | Display binary (hashes, keys) |
| **Binary** | Binary | None | ❌ No | Wire protocols, file formats |
| **Protobuf varint** | Binary | Compact | ❌ No | Efficient integer encoding |
