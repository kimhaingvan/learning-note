# Proxy

> **Category**: Structural | **Language**: Go

---

## What

**Proxy** provides a substitute or placeholder for another object. The proxy controls access to the original object, allowing you to perform something **before or after** the request reaches the original.

---

## Why / When

- **Lazy initialization** — create the real object only when first needed.
- **Access control** — check permissions before forwarding.
- **Caching** — return cached results instead of calling the real object.
- **Logging / Monitoring** — record calls transparently.

---

## How

1. **Define the common interface** for the Real Object.
2. **Create and implement the Real Object.**
3. **Create a Proxy struct** — holds a field referencing the Real Object.
4. **Implement the same interface in the Proxy** — add gatekeeper logic (cache, auth, logging), then delegate to the Real Object.
5. **Client uses only the interface** — does not know whether it talks to the proxy or the real object.

> **Proxy vs Decorator**: Proxy has "gatekeeper" semantics — may block or shortcut calls. Decorator adds/wraps behavior and always calls through.

---

## Example

```go
package main

import "fmt"

// Common interface
type IStorage interface {
	GetImage(id string) (Image, error)
}

type Image struct {
	Id   string
	Name string
}

// Real Object
type StorageService struct{}

func (s *StorageService) GetImage(id string) (Image, error) {
	return Image{
		Id:   id,
		Name: fmt.Sprintf("Image-Name-%s", id),
	}, nil
}

// Proxy — adds caching
type CachingProxy struct {
	storage      IStorage
	cachedImages map[string]Image
}

func (p *CachingProxy) GetImage(id string) (Image, error) {
	if image, ok := p.cachedImages[id]; ok {
		fmt.Println("Image found in cache")
		return image, nil
	}
	fmt.Println("Image not found in cache, fetching from storage")
	image, err := p.storage.GetImage(id)
	if err != nil {
		return Image{}, err
	}
	p.cachedImages[id] = image
	return image, nil
}

// Client uses the interface
func main() {
	storage := &StorageService{}
	proxy := &CachingProxy{
		storage:      storage,
		cachedImages: make(map[string]Image),
	}

	image, _ := proxy.GetImage("1")
	fmt.Println("Fetched Image:", image)
	// Image not found in cache, fetching from storage
	// Fetched Image: {1 Image-Name-1}

	image, _ = proxy.GetImage("1")
	fmt.Println("Fetched Image:", image)
	// Image found in cache
	// Fetched Image: {1 Image-Name-1}
}
```

---

## Pitfalls

- Proxy adding latency on top of every call (e.g., cache lookup overhead).
- Stale cache — proxy must have a cache invalidation strategy.
- Client assuming the proxy behaves identically to the real object in all edge cases.

---

## Memory Aid

> Proxy = security guard at the door. Same uniform as the real service, but checks your ID (or serves from cache) before letting you through.
