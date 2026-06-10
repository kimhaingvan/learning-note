# Regular Expressions (Regex)

> **Category**: Backend Fundamental | **Level**: Intermediate

---

## Table of Contents

1. [What](#1-what)
2. [Why](#2-why)
3. [How — Syntax Cheat Sheet](#3-how--syntax-cheat-sheet)
4. [How — Common Patterns](#4-how--common-patterns)
5. [How — Regex in Go](#5-how--regex-in-go)
6. [References](#6-references)

---

## 1. What

**Regular Expression (Regex)** is a sequence of characters that defines a **search pattern**. It is used for **matching**, **finding**, **replacing**, and **validating** text.

A regex engine reads the pattern and walks through the input string character by character, trying to match the pattern.

---

## 2. Why

- **Validation**: email, phone number, URL, IP address, password strength.
- **Search & Replace**: find patterns in logs, code, or documents.
- **Parsing**: extract data from unstructured text (log files, HTML, CSV).
- **Routing**: web frameworks use regex to match URL paths.
- **Lexing/Tokenizing**: compilers and interpreters use regex to break source code into tokens.

---

## 3. How — Syntax Cheat Sheet

### Characters

| Pattern | Meaning | Example | Matches |
|---------|---------|---------|---------|
| `\d` | One digit `[0-9]` | `file_\d\d` | `file_25` |
| `\w` | Word character `[a-zA-Z0-9_]` | `\w-\w\w\w` | `A-b_1` |
| `\s` | Whitespace (space, tab, newline) | `a\sb\sc` | `a b c` |
| `\D` | Not a digit | `\D\D\D` | `ABC` |
| `\W` | Not a word character | `\W\W\W` | `*-+` |
| `\S` | Not a whitespace | `\S\S\S\S` | `Yoyo` |
| `.` | Any character except newline | `a.c` | `abc` |

### Quantifiers

| Quantifier | Meaning | Example | Matches |
|------------|---------|---------|---------|
| `+` | One or more (greedy) | `\d+` | `12345` |
| `*` | Zero or more (greedy) | `A*B` | `AAAB` or `B` |
| `?` | Zero or one | `colou?r` | `color` or `colour` |
| `{n}` | Exactly n times | `\d{3}` | `123` |
| `{n,m}` | Between n and m times | `\d{2,4}` | `12`, `123`, `1234` |
| `{n,}` | n or more times | `\w{3,}` | `hello` |
| `+?`, `*?` | Lazy (non-greedy) version | `\d+?` | `1` in `12345` |

### Character Classes

| Pattern | Meaning | Example | Matches |
|---------|---------|---------|---------|
| `[abc]` | One of a, b, or c | `[aeiou]` | One vowel |
| `[a-z]` | Range: lowercase letters | `[a-z]+` | `hello` |
| `[^abc]` | NOT a, b, or c | `[^0-9]+` | Non-digit text |
| `[a-zA-Z0-9]` | Alphanumeric | `[a-zA-Z0-9]+` | `Hello123` |

### Anchors & Boundaries

| Anchor | Meaning | Example | Matches |
|--------|---------|---------|---------|
| `^` | Start of string/line | `^Hello` | `Hello` at line start |
| `$` | End of string/line | `world$` | `world` at line end |
| `\b` | Word boundary | `\bcat\b` | `cat` but not `cats` |
| `\B` | Not a word boundary | `\Bcat\B` | `cat` in `concatenate` |

### Groups & Logic

| Pattern | Meaning | Example | Matches |
|---------|---------|---------|---------|
| `(...)` | Capturing group | `(abc)+` | `abcabc` |
| `(?:...)` | Non-capturing group | `(?:abc)+` | `abcabc` (no capture) |
| `\|` | OR / alternation | `cat\|dog` | `cat` or `dog` |
| `\1` | Backreference to group 1 | `(\w)\1` | `aa`, `bb` |

### Lookarounds

| Pattern | Name | Example | Matches |
|---------|------|---------|---------|
| `(?=...)` | Positive lookahead | `\d(?=px)` | `5` in `5px` |
| `(?!...)` | Negative lookahead | `\d(?!px)` | `5` in `5em` |
| `(?<=...)` | Positive lookbehind | `(?<=\$)\d+` | `100` in `$100` |
| `(?<!...)` | Negative lookbehind | `(?<!\$)\d+` | `100` in `€100` |

---

## 4. How — Common Patterns

```
# Email
^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$

# URL
https?://[^\s/$.?#].[^\s]*

# IPv4
^(\d{1,3}\.){3}\d{1,3}$

# Phone (international)
^\+?[1-9]\d{1,14}$

# Date (YYYY-MM-DD)
^\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])$

# Password (min 8 chars, 1 upper, 1 lower, 1 digit, 1 special)
^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]{8,}$

# UUID v4
^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$
```

---

## 5. How — Regex in Go

Go uses the `regexp` package (RE2 syntax — no backreferences, no lookarounds).

### Basic Usage

```go
package main

import (
    "fmt"
    "regexp"
)

func main() {
    // Compile pattern (returns error if invalid)
    re, err := regexp.Compile(`\d+`)
    if err != nil {
        panic(err)
    }

    // MustCompile panics on invalid pattern — use for known-good patterns
    re = regexp.MustCompile(`\d+`)

    // Match check
    fmt.Println(re.MatchString("hello 123"))  // true
    fmt.Println(re.MatchString("hello world")) // false

    // Find first match
    fmt.Println(re.FindString("abc 123 def 456")) // "123"

    // Find all matches
    fmt.Println(re.FindAllString("abc 123 def 456", -1)) // ["123", "456"]

    // Replace
    result := re.ReplaceAllString("order 123 and 456", "XXX")
    fmt.Println(result) // "order XXX and XXX"
}
```

### Capturing Groups

```go
re := regexp.MustCompile(`(\w+)@(\w+)\.(\w+)`)
match := re.FindStringSubmatch("user@example.com")
// match[0] = "user@example.com"  (full match)
// match[1] = "user"              (group 1)
// match[2] = "example"           (group 2)
// match[3] = "com"               (group 3)
```

### Named Groups

```go
re := regexp.MustCompile(`(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})`)
match := re.FindStringSubmatch("2024-01-15")
names := re.SubexpNames()

for i, name := range names {
    if i != 0 && name != "" {
        fmt.Printf("%s: %s\n", name, match[i])
    }
}
// year: 2024
// month: 01
// day: 15
```

### Validation Example

```go
func isValidEmail(email string) bool {
    re := regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)
    return re.MatchString(email)
}
```

> **Tip**: Compile regex patterns once (package-level `var`) and reuse. `regexp.MustCompile` is safe for concurrent use.

```go
var emailRegex = regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)

func isValidEmail(email string) bool {
    return emailRegex.MatchString(email)
}
```

---

## 6. References

### Docs

- [RexEgg — Regex Quick-Start Cheat Sheet](http://www.rexegg.com/regex-quickstart.html)
- [Go Regex — yourbasic.org](https://yourbasic.org/golang/regular-expressions/)
- [Go `regexp` package](https://pkg.go.dev/regexp)

### Practice

- [HackerRank — Regex Domain](https://www.hackerrank.com/domains/regex)
- [LeetCode — Regex Problems](https://leetcode.com/problemset/all/?topicSlugs=regex)
- [regex101.com — Online Regex Tester](https://regex101.com/)
