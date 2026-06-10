# Hashing and Encryption

> **Category**: Backend Fundamental | **Level**: Intermediate

---

## Table of Contents

1. [Hashing vs Encryption](#1-hashing-vs-encryption)
2. [Hashing — What & How](#2-hashing--what--how)
3. [SHA-256](#3-sha-256)
4. [SHA-3](#4-sha-3)
5. [SHA-256 vs SHA-3](#5-sha-256-vs-sha-3)
6. [Code Examples](#6-code-examples)

---

## 1. Hashing vs Encryption

| Feature | Hashing | Encryption |
|---------|---------|------------|
| **Direction** | One-way (irreversible) | Two-way (reversible) |
| **Purpose** | Verify integrity, store passwords | Protect data confidentiality |
| **Output** | Fixed-size digest | Variable-size ciphertext |
| **Key required?** | No (HMAC uses key) | Yes (symmetric or asymmetric) |
| **Can recover original?** | No | Yes (with the key) |
| **Example** | SHA-256, bcrypt | AES, RSA |

---

## 2. Hashing — What & How

### What

A **hash function** takes an input of **any size** and produces a **fixed-size** output (called a **digest** or **hash**). The same input always produces the same output.

### Properties of a Cryptographic Hash

| Property | Meaning |
|----------|---------|
| **Deterministic** | Same input → same output, always |
| **Fast** | Computes quickly for any input size |
| **Pre-image resistant** | Given hash `h`, hard to find input `m` such that `hash(m) = h` |
| **Second pre-image resistant** | Given input `m1`, hard to find `m2 ≠ m1` where `hash(m1) = hash(m2)` |
| **Collision resistant** | Hard to find any two inputs `m1 ≠ m2` where `hash(m1) = hash(m2)` |
| **Avalanche effect** | A tiny change in input → completely different output |

### How (General Steps)

```
Input → Padding → Break into blocks → Process blocks with compression function → Output digest
```

1. **Pad** the message to a fixed block size.
2. **Split** into fixed-size blocks.
3. **Process** each block through a compression function with an internal state.
4. **Output** the final internal state as the hash digest.

---

## 3. SHA-256

### What

**SHA-256** (Secure Hash Algorithm 256-bit) is a member of the **SHA-2** family, designed by the **NSA**. It produces a **256-bit (32-byte)** hash.

### How It Works

1. **Pre-processing**: Pad the message so its length is a multiple of 512 bits.
2. **Parse**: Break into 512-bit blocks.
3. **Initialize**: Set 8 hash values (`H0`–`H7`) from the fractional parts of square roots of the first 8 primes.
4. **Compression**: For each block, run **64 rounds** of bitwise operations (AND, OR, XOR, shifts, rotations) using 64 round constants.
5. **Output**: Concatenate the 8 final hash values → 256-bit digest.

```
Input: "hello"
SHA-256: 2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824
```

### When to Use

- File integrity checks (checksums).
- Digital signatures.
- Blockchain (Bitcoin uses SHA-256).
- HMAC for message authentication.

---

## 4. SHA-3

### What

**SHA-3** is the latest member of the Secure Hash Algorithm family, based on the **Keccak** sponge construction. It is **NOT** a replacement for SHA-2 — it is an **alternative** for diversity.

### How It Works (Sponge Construction)

Unlike SHA-2 (Merkle–Damgård), SHA-3 uses a **sponge function**:

1. **Absorb phase**: XOR input blocks into the internal state, then apply a permutation function.
2. **Squeeze phase**: Extract output blocks from the state until the desired hash length is reached.

```
┌─────────────────────────────────────────────┐
│          Sponge Construction                │
│                                             │
│  Input blocks → [Absorb] → [Squeeze] → Output
│                    ↕            ↕            │
│              Permutation   Permutation       │
└─────────────────────────────────────────────┘
```

### When to Use

- When you need an **alternative** to SHA-2 (defense in depth).
- Standards compliance requiring SHA-3.
- Variable-length output via SHAKE128/SHAKE256.

---

## 5. SHA-256 vs SHA-3

| Feature | SHA-256 | SHA-3 (SHA3-256) |
|---------|---------|-------------------|
| **Family** | SHA-2 | SHA-3 (Keccak) |
| **Structure** | Merkle–Damgård | Sponge |
| **Output size** | 256 bits | 256 bits (configurable) |
| **Block size** | 512 bits | 1088 bits (for SHA3-256) |
| **Rounds** | 64 | 24 |
| **Speed** | Generally faster in software | Faster in hardware |
| **Security** | No known practical attacks | No known practical attacks |
| **Adoption** | Widely used (TLS, Bitcoin, Git) | Growing, less legacy support |

---

## 6. Code Examples

### Go — SHA-256

```go
package main

import (
    "crypto/sha256"
    "fmt"
)

func main() {
    data := []byte("hello")

    // One-shot hash
    hash := sha256.Sum256(data)
    fmt.Printf("SHA-256: %x\n", hash)
    // SHA-256: 2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824

    // Streaming hash (for large data)
    h := sha256.New()
    h.Write([]byte("hel"))
    h.Write([]byte("lo"))
    fmt.Printf("SHA-256: %x\n", h.Sum(nil))
    // Same output
}
```

### Go — SHA-3

```go
package main

import (
    "fmt"
    "golang.org/x/crypto/sha3"
)

func main() {
    data := []byte("hello")

    // SHA3-256
    hash := sha3.Sum256(data)
    fmt.Printf("SHA3-256: %x\n", hash)

    // SHAKE256 (variable-length output)
    shake := sha3.NewShake256()
    shake.Write(data)
    output := make([]byte, 64) // 64 bytes output
    shake.Read(output)
    fmt.Printf("SHAKE256: %x\n", output)
}
```

### Go — HMAC (Hash-based Message Authentication Code)

```go
package main

import (
    "crypto/hmac"
    "crypto/sha256"
    "fmt"
)

func main() {
    key := []byte("my-secret-key")
    message := []byte("hello world")

    mac := hmac.New(sha256.New, key)
    mac.Write(message)
    signature := mac.Sum(nil)
    fmt.Printf("HMAC-SHA256: %x\n", signature)

    // Verify — use hmac.Equal (constant-time comparison to prevent timing attacks)
    mac2 := hmac.New(sha256.New, key)
    mac2.Write(message)
    expected := mac2.Sum(nil)
    fmt.Println("Valid:", hmac.Equal(signature, expected)) // true
}
```

> **Important**: Always use `hmac.Equal()` instead of `==` for comparing MACs — it prevents **timing attacks**.
