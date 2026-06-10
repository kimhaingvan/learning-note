# Symmetric and Asymmetric Encryption

> **Category**: Backend Fundamental | **Level**: Intermediate

---

## Table of Contents

1. [What](#1-what)
2. [Symmetric Encryption (AES)](#2-symmetric-encryption-aes)
3. [Asymmetric Encryption (RSA)](#3-asymmetric-encryption-rsa)
4. [Comparison](#4-comparison)
5. [How They Work Together (TLS Handshake)](#5-how-they-work-together-tls-handshake)
6. [Code Examples](#6-code-examples)

---

## 1. What

| Type | How | Analogy |
|------|-----|---------|
| **Symmetric** | Same key to encrypt and decrypt | A locked box with one key — both sender and receiver share the same key |
| **Asymmetric** | Public key encrypts, private key decrypts | A mailbox — anyone can drop mail in (public key), only the owner can open it (private key) |

---

## 2. Symmetric Encryption (AES)

### What

**AES (Advanced Encryption Standard)** is a symmetric block cipher that encrypts data in fixed-size blocks (128 bits) using a shared secret key.

### How

```
Plaintext → [AES Encrypt with Key] → Ciphertext
Ciphertext → [AES Decrypt with Key] → Plaintext
```

1. **Key expansion**: Derive round keys from the original key.
2. **Initial round**: XOR plaintext with the first round key.
3. **Main rounds** (10/12/14 rounds depending on key size):
   - **SubBytes**: Substitute each byte using an S-box (non-linear substitution).
   - **ShiftRows**: Cyclically shift rows of the state matrix.
   - **MixColumns**: Mix columns using matrix multiplication in GF(2^8).
   - **AddRoundKey**: XOR with the round key.
4. **Final round**: Same as main round but without MixColumns.

### Key Sizes

| Key Size | Rounds | Security Level |
|----------|--------|----------------|
| 128 bits | 10 | Standard |
| 192 bits | 12 | High |
| 256 bits | 14 | Very High (government/military) |

### Block Cipher Modes

AES operates on 128-bit blocks. To encrypt data larger than one block, you need a **mode of operation**:

| Mode | Name | How | Use Case |
|------|------|-----|----------|
| **ECB** | Electronic Codebook | Each block encrypted independently | ❌ Never use — identical blocks → identical ciphertext |
| **CBC** | Cipher Block Chaining | Each block XORed with previous ciphertext block | File encryption |
| **CTR** | Counter | Turns block cipher into stream cipher using counter | Streaming, disk encryption |
| **GCM** | Galois/Counter Mode | CTR + authentication tag | **Recommended** — TLS, API encryption |

> **Always use AES-GCM** — it provides both **encryption** (confidentiality) and **authentication** (integrity).

### When to Use

- Encrypting data at rest (files, database fields).
- Encrypting data in transit (part of TLS).
- Session encryption after key exchange.
- Fast bulk data encryption.

---

## 3. Asymmetric Encryption (RSA)

### What

**RSA** is an asymmetric encryption algorithm that uses a **pair of keys**: a public key (shared openly) and a private key (kept secret). Based on the mathematical difficulty of **factoring the product of two large prime numbers**.

### How

```
Key Generation:
  1. Choose two large primes: p, q
  2. Compute n = p × q
  3. Compute φ(n) = (p-1)(q-1)
  4. Choose e (public exponent, commonly 65537)
  5. Compute d = e⁻¹ mod φ(n)  (private exponent)

Public Key:  (e, n)
Private Key: (d, n)

Encrypt: ciphertext = plaintext^e mod n
Decrypt: plaintext = ciphertext^d mod n
```

### Key Sizes

| Key Size | Security Level | Status |
|----------|----------------|--------|
| 1024 bits | ~80-bit security | ❌ Broken — do not use |
| 2048 bits | ~112-bit security | ✅ Minimum acceptable |
| 4096 bits | ~128-bit security | ✅ Recommended |

### When to Use

- **Key exchange**: Securely share a symmetric key.
- **Digital signatures**: Sign data with private key, verify with public key.
- **Certificate validation**: TLS/SSL certificates.
- **Small data encryption**: RSA can only encrypt data smaller than the key size.

> **RSA is slow** — never use it to encrypt large data directly. Use it to encrypt a symmetric key, then use AES for the bulk data (hybrid encryption).

---

## 4. Comparison

| Feature | Symmetric (AES) | Asymmetric (RSA) |
|---------|-----------------|-------------------|
| **Keys** | 1 shared key | 2 keys (public + private) |
| **Speed** | Fast (~1000x faster) | Slow |
| **Key distribution** | Problem: how to share the key securely? | Easy: public key can be shared openly |
| **Data size** | Any size | Limited by key size |
| **Use case** | Bulk data encryption | Key exchange, signatures |
| **Key size for 128-bit security** | 128 bits | 3072 bits |
| **Examples** | AES-128, AES-256, ChaCha20 | RSA, ECDSA, Ed25519 |

---

## 5. How They Work Together (TLS Handshake)

In practice, asymmetric encryption is used to **exchange keys**, then symmetric encryption handles the actual data:

```
Client                                Server
  │                                      │
  │──── ClientHello (supported ciphers) ─────►│
  │                                      │
  │◄──── ServerHello + Certificate (public key)│
  │                                      │
  │  1. Client generates random AES key  │
  │  2. Encrypts AES key with server's   │
  │     RSA public key                   │
  │──── Encrypted AES key ──────────────►│
  │                                      │
  │  3. Server decrypts with private key │
  │  4. Both sides now have the AES key  │
  │                                      │
  │◄════ AES-encrypted communication ═══►│
  │     (fast symmetric encryption)      │
```

---

## 6. Code Examples

### Go — AES-GCM Encryption/Decryption

```go
package main

import (
    "crypto/aes"
    "crypto/cipher"
    "crypto/rand"
    "encoding/hex"
    "fmt"
    "io"
)

func encrypt(plaintext []byte, key []byte) (string, error) {
    block, err := aes.NewCipher(key)
    if err != nil {
        return "", err
    }

    aesGCM, err := cipher.NewGCM(block)
    if err != nil {
        return "", err
    }

    // Generate random nonce
    nonce := make([]byte, aesGCM.NonceSize())
    if _, err := io.ReadFull(rand.Reader, nonce); err != nil {
        return "", err
    }

    // Encrypt and prepend nonce to ciphertext
    ciphertext := aesGCM.Seal(nonce, nonce, plaintext, nil)
    return hex.EncodeToString(ciphertext), nil
}

func decrypt(ciphertextHex string, key []byte) (string, error) {
    ciphertext, _ := hex.DecodeString(ciphertextHex)

    block, err := aes.NewCipher(key)
    if err != nil {
        return "", err
    }

    aesGCM, err := cipher.NewGCM(block)
    if err != nil {
        return "", err
    }

    // Extract nonce from ciphertext
    nonceSize := aesGCM.NonceSize()
    nonce, ciphertext := ciphertext[:nonceSize], ciphertext[nonceSize:]

    plaintext, err := aesGCM.Open(nil, nonce, ciphertext, nil)
    if err != nil {
        return "", err
    }

    return string(plaintext), nil
}

func main() {
    key := []byte("my-32-byte-secret-key-here!!1234") // AES-256 requires 32 bytes

    encrypted, _ := encrypt([]byte("Hello, World!"), key)
    fmt.Println("Encrypted:", encrypted)

    decrypted, _ := decrypt(encrypted, key)
    fmt.Println("Decrypted:", decrypted) // Hello, World!
}
```

### Go — RSA Key Generation, Encrypt & Decrypt

```go
package main

import (
    "crypto/rand"
    "crypto/rsa"
    "crypto/sha256"
    "fmt"
)

func main() {
    // Generate RSA key pair (2048 bits)
    privateKey, err := rsa.GenerateKey(rand.Reader, 2048)
    if err != nil {
        panic(err)
    }
    publicKey := &privateKey.PublicKey

    // Encrypt with public key (OAEP padding)
    plaintext := []byte("Hello, RSA!")
    ciphertext, err := rsa.EncryptOAEP(sha256.New(), rand.Reader, publicKey, plaintext, nil)
    if err != nil {
        panic(err)
    }
    fmt.Printf("Encrypted: %x\n", ciphertext)

    // Decrypt with private key
    decrypted, err := rsa.DecryptOAEP(sha256.New(), rand.Reader, privateKey, ciphertext, nil)
    if err != nil {
        panic(err)
    }
    fmt.Println("Decrypted:", string(decrypted)) // Hello, RSA!
}
```

### Go — RSA Digital Signature

```go
package main

import (
    "crypto"
    "crypto/rand"
    "crypto/rsa"
    "crypto/sha256"
    "fmt"
)

func main() {
    privateKey, _ := rsa.GenerateKey(rand.Reader, 2048)
    publicKey := &privateKey.PublicKey

    message := []byte("Important message")

    // Sign with private key
    hashed := sha256.Sum256(message)
    signature, err := rsa.SignPKCS1v15(rand.Reader, privateKey, crypto.SHA256, hashed[:])
    if err != nil {
        panic(err)
    }

    // Verify with public key
    err = rsa.VerifyPKCS1v15(publicKey, crypto.SHA256, hashed[:], signature)
    if err != nil {
        fmt.Println("Invalid signature")
    } else {
        fmt.Println("Signature is valid")
    }
}
```
