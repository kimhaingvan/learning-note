# Password Hashing

> **Category**: Backend Fundamental | **Level**: Intermediate

---

## Table of Contents

1. [What](#1-what)
2. [Why Not SHA-256 for Passwords?](#2-why-not-sha-256-for-passwords)
3. [How Password Hashing Works](#3-how-password-hashing-works)
4. [bcrypt](#4-bcrypt)
5. [scrypt](#5-scrypt)
6. [Argon2](#6-argon2)
7. [Comparison](#7-comparison)
8. [Code Examples](#8-code-examples)

---

## 1. What

**Password hashing** is the process of converting a plaintext password into a fixed-length, irreversible string using a **purposely slow** hash function combined with a **random salt**.

Unlike general-purpose hashing (SHA-256), password hashing algorithms are designed to be **computationally expensive** — making brute-force attacks impractical.

---

## 2. Why Not SHA-256 for Passwords?

| Problem | Explanation |
|---------|-------------|
| **Too fast** | SHA-256 can compute billions of hashes/sec on modern GPUs — brute-force is trivial |
| **No built-in salt** | Without a salt, identical passwords produce identical hashes → vulnerable to **rainbow table** attacks |
| **No cost factor** | Cannot adjust difficulty as hardware gets faster |

**Password hashing algorithms solve all three problems**: they are slow, include a salt, and have a configurable cost factor.

---

## 3. How Password Hashing Works

### Registration (Store)

```
password + random_salt → slow_hash_function(password, salt, cost) → stored_hash
Store: stored_hash (contains salt + cost + hash)
```

### Login (Verify)

```
input_password + extract_salt_from_stored_hash → slow_hash_function(input, salt, cost) → computed_hash
Compare: computed_hash == stored_hash? → Valid / Invalid
```

### Key Concepts

| Concept | Meaning |
|---------|---------|
| **Salt** | Random bytes added to the password before hashing — ensures identical passwords produce different hashes |
| **Cost factor** | Controls how slow/expensive the hash computation is — increase as hardware improves |
| **Key stretching** | Running the hash function many iterations to increase computation time |

---

## 4. bcrypt

### What

**bcrypt** is a password hashing function based on the **Blowfish** cipher. It has been the industry standard since 1999.

### How

1. Generate a **16-byte random salt**.
2. Derive an encryption key from the password using **Eksblowfish** (expensive key schedule Blowfish).
3. Encrypt the string `"OrpheanBeholderScryDoubt"` **64 times** using the derived key.
4. Concatenate: `$2b$` + cost + `$` + salt + hash.

### Output Format

```
$2b$12$WApznUPhDubN0oeveSXHp.Ux5KnMEDCerlASptD2IVzrMFCIpKNPa
│  │  │                      │
│  │  │                      └─ 31 chars: salt (22) + hash (31) in base64
│  │  └─ cost factor (2^12 = 4096 iterations)
│  └─ version (2b)
└─ algorithm identifier
```

### Limitations

- Max input: **72 bytes** (longer passwords are truncated).
- CPU-hard only (not memory-hard) — vulnerable to GPU/ASIC attacks at scale.

---

## 5. scrypt

### What

**scrypt** is a password hashing function designed to be **both CPU-hard and memory-hard**, making it expensive to attack with specialized hardware (GPUs, ASICs).

### How

1. Derive an initial key from the password using **PBKDF2-HMAC-SHA256**.
2. Use that key to fill a **large memory array** (configurable size).
3. Perform **random memory-dependent lookups** — requires keeping the array in RAM.
4. Produce the final hash from the accessed memory.

### Parameters

| Parameter | Meaning | Typical Value |
|-----------|---------|---------------|
| `N` | CPU/memory cost (must be power of 2) | 32768 (2^15) |
| `r` | Block size (memory usage = 128 × N × r bytes) | 8 |
| `p` | Parallelism factor | 1 |
| `keyLen` | Output hash length | 32 bytes |

---

## 6. Argon2

### What

**Argon2** is the **winner of the 2015 Password Hashing Competition** and is considered the current best practice for password hashing. It comes in three variants:

| Variant | Designed For |
|---------|--------------|
| **Argon2d** | Maximizes GPU resistance (data-dependent memory access) — use for cryptocurrency |
| **Argon2i** | Resists side-channel attacks (data-independent access) — use for password hashing |
| **Argon2id** | Hybrid of both — **recommended for most use cases** |

### How (Argon2id)

1. First pass uses **data-independent** memory access (Argon2i behavior) — resists side-channel attacks.
2. Subsequent passes use **data-dependent** access (Argon2d behavior) — resists GPU attacks.
3. Fills a **configurable amount of memory** in multiple passes.

### Parameters

| Parameter | Meaning | Recommended (OWASP) |
|-----------|---------|----------------------|
| `memory` | Memory usage in KiB | 64 MiB (65536 KiB) |
| `iterations` | Number of passes over memory | 3 |
| `parallelism` | Number of threads | 4 |
| `saltLength` | Salt size | 16 bytes |
| `keyLength` | Output hash size | 32 bytes |

### Output Format

```
$argon2id$v=19$m=65536,t=3,p=4$c2FsdHNhbHQ$hash...
│        │    │               │             │
│        │    │               │             └─ Base64 encoded hash
│        │    │               └─ Base64 encoded salt
│        │    └─ parameters: memory=64MB, time=3, parallelism=4
│        └─ version
└─ algorithm
```

---

## 7. Comparison

| Feature | bcrypt | scrypt | Argon2id |
|---------|--------|--------|----------|
| **Year** | 1999 | 2009 | 2015 |
| **CPU-hard** | ✅ Yes | ✅ Yes | ✅ Yes |
| **Memory-hard** | ❌ No | ✅ Yes | ✅ Yes |
| **GPU/ASIC resistant** | ⚠️ Moderate | ✅ Yes | ✅ Yes |
| **Side-channel resistant** | ✅ Yes | ❌ No | ✅ Yes (Argon2id) |
| **Max input** | 72 bytes | Unlimited | Unlimited |
| **Configurable params** | Cost only | CPU + Memory + Parallelism | CPU + Memory + Parallelism |
| **Recommendation** | Legacy / still acceptable | Good | **Best practice** |

> **OWASP recommendation (2024)**: Use **Argon2id** as the first choice. If not available, use **bcrypt** with cost ≥ 10.

---

## 8. Code Examples

### Go — bcrypt

```go
package main

import (
    "fmt"
    "golang.org/x/crypto/bcrypt"
)

func main() {
    password := []byte("mySecurePassword123")

    // Hash password (cost = 12)
    hash, err := bcrypt.GenerateFromPassword(password, 12)
    if err != nil {
        panic(err)
    }
    fmt.Println("Hash:", string(hash))
    // $2a$12$LJ3m4ys3Lk0TdcnOoyCzku...

    // Verify password
    err = bcrypt.CompareHashAndPassword(hash, password)
    if err != nil {
        fmt.Println("Invalid password")
    } else {
        fmt.Println("Password is correct")
    }
}
```

### Go — Argon2id

```go
package main

import (
    "crypto/rand"
    "crypto/subtle"
    "encoding/base64"
    "fmt"
    "golang.org/x/crypto/argon2"
)

func hashPassword(password string) (string, error) {
    // Generate random salt
    salt := make([]byte, 16)
    if _, err := rand.Read(salt); err != nil {
        return "", err
    }

    // Hash with Argon2id
    // Parameters: time=3, memory=64MB, threads=4, keyLen=32
    hash := argon2.IDKey([]byte(password), salt, 3, 64*1024, 4, 32)

    // Encode for storage
    return fmt.Sprintf("$argon2id$v=19$m=65536,t=3,p=4$%s$%s",
        base64.RawStdEncoding.EncodeToString(salt),
        base64.RawStdEncoding.EncodeToString(hash),
    ), nil
}

func verifyPassword(password, encoded string) bool {
    // Parse encoded hash to extract salt and parameters
    // (simplified — production code should parse all fields)
    salt, _ := base64.RawStdEncoding.DecodeString("extracted_salt")
    expectedHash, _ := base64.RawStdEncoding.DecodeString("extracted_hash")

    hash := argon2.IDKey([]byte(password), salt, 3, 64*1024, 4, 32)

    // Constant-time comparison to prevent timing attacks
    return subtle.ConstantTimeCompare(hash, expectedHash) == 1
}

func main() {
    hash, _ := hashPassword("mySecurePassword123")
    fmt.Println(hash)
}
```

> **Key takeaway**: Never store passwords in plaintext. Never use general-purpose hashes (MD5, SHA-256) for passwords. Use bcrypt or Argon2id.
