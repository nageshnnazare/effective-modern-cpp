# Chapter 30: `new` and `delete` with Over-Aligned Data

Tutorial aligned with *C++17 - The Complete Guide* by Nicolai M. Josuttis. **Page references**: **pp. 365-383**.

This chapter covers **over-aligned types**: objects whose alignment exceeds the **default alignment** for dynamic allocation on your platform. C++17 fixed a longstanding portability gap: **global `new`** may now allocate from a distinct **over-aligned** path so that **`alignas(N)`** types get storage that actually respects **N** (p. 365-366).

---

## 30.1 Using `new` with alignments (p. 365)

### `alignas` and the alignment requirement

```cpp
struct Ordinary {
    int x;
};

struct alignas(32) Over32 {
    int y;
};

static_assert(alignof(Ordinary) <= __STDCPP_DEFAULT_NEW_ALIGNMENT__);
// Over32 may have alignof == 32 > default new alignment on some platforms.
```

**Before C++17:** `new Over32` was not guaranteed to return memory aligned to 32 on all implementations; platforms might only guarantee **`__STDCPP_DEFAULT_NEW_ALIGNMENT__`** for ordinary `::operator new(std::size_t)` (often 16 or 8 bytes) (p. 365).

**Since C++17:** allocation for types with extended alignment uses the **over-aligned allocation path** so the returned pointer satisfies **`alignof(T)`** (p. 365).

---

## 30.1.1 Distinct dynamic / heap memory arenas (p. 366)

Implementations may use **two conceptual arenas** (not necessarily separate heaps, but distinct entry points):

1. **Default-aligned** allocations: plain `::operator new(size_t)` / `delete(void*)`.
2. **Over-aligned** allocations: `::operator new(size_t, std::align_val_t)` and matching **`delete`**.

**ASCII diagram: alignment and slot in a buffer**

```
Address increases -->

.. |  pad  |<------ object of type T, alignof(T)=32 ------>|
   ^        ^
   |        start address is a multiple of 32
   unused bytes to satisfy alignment (internal or at allocation boundary)

For dynamic storage, the runtime ensures the returned pointer p satisfies:
  reinterpret_cast<std::uintptr_t>(p) % alignof(T) == 0
```

Josuttis explains why splitting the interface matters for **implementers** and for **replacing** global `new` correctly (p. 366-370).

---

## 30.1.2 Passing the alignment with `new` expression (p. 367)

The compiler, when allocating an over-aligned type **T**, passes alignment as **`std::align_val_t(alignof(T))`** to the allocating `operator new`.

**Key overloads (conceptual):**

```cpp
// Allocation
void* operator new(std::size_t count, std::align_val_t al);
void* operator new[](std::size_t count, std::align_val_t al);

// Deallocation (must match how allocation was obtained)
void operator delete(void* ptr, std::size_t count, std::align_val_t al) noexcept;
void operator delete[](void* ptr, std::size_t count, std::align_val_t al) noexcept;
```

**Notes:**

- The **`std::size_t`**-aware `delete` overloads support **sized deallocation** where enabled; Josuttis ties this to matching **allocation** and **alignment** for correctness (p. 367).
- **Mismatch** (wrong `delete` for pointer origin) remains **undefined behavior**.

### Minimal usage example

```cpp
#include <new>
#include <cstdlib>

struct alignas(64) CacheLineBlock {
    int data[16];
};

void use_overaligned() {
    auto* p = new CacheLineBlock{};
    // ... use *p ...
    delete p;  // Compiler selects aligned delete for over-aligned class types (C++17)
}
```

---

## 30.2 Implementing `operator new()` for aligned memory (p. 370)

### Pre-C++17 workarounds

Portable C++ before the aligned `new` path often used:

- **`std::aligned_alloc`** / **C11** (where available) with manual lifetime management.
- **`posix_memalign`** / platform APIs.

These workarounds complicate **constructors**, **exceptions**, and **array** semantics (p. 370).

### Type-specific `operator new()` with alignment

You can provide **class-scoped** allocation hooks:

```cpp
#include <new>
#include <cstdlib>

struct alignas(128) Widget {
    static void* operator new(std::size_t n);
    static void operator delete(void* p, std::size_t n) noexcept;

    int payload;
};

inline void* Widget::operator new(std::size_t n) {
    void* p = std::aligned_alloc(128, n);
    if (!p) {
        throw std::bad_alloc{};
    }
    return p;
}

inline void Widget::operator delete(void* p, std::size_t n) noexcept {
    // n may be used for sized delete matching; free must match aligned_alloc
    std::free(p);
    (void)n;
}
```

**C++17 extension:** for **over-aligned** types, you may also need **`operator new(std::size_t, std::align_val_t)`** -- Josuttis explains when the class-specific **aligned** overloads are required vs when the **global** aligned `new` suffices (p. 370-377).

### Placement `delete` and construction failure

If **`new (ptr) T(...)`** throws in a custom allocator routine, **placement delete** (if any) participates in cleanup strategies for **failed construction**. Josuttis revisits the **placement new** / **exception** interaction when layering aligned blocks (p. 370).

---

## 30.3 Implementing global `operator new()` (p. 378)

Replacing **global** allocation affects **all** dynamic storage. For **C++17**, a complete replacement strategy must consider:

- `::operator new(std::size_t)`
- `::operator new(std::size_t, std::align_val_t)`  // over-aligned path
- Array forms, `nothrow` variants as required by your replacement policy

**Sketch (illustrative only -- production code needs full overload set and POSIX/ABI care):**

```cpp
#include <new>
#include <cstdlib>

void* operator new(std::size_t size) {
    void* p = std::malloc(size);
    if (!p) {
        throw std::bad_alloc{};
    }
    return p;
}

void* operator new(std::size_t size, std::align_val_t al) {
    void* p = std::aligned_alloc(static_cast<std::size_t>(al), size);
    if (!p) {
        throw std::bad_alloc{};
    }
    return p;
}

void operator delete(void* p) noexcept {
    std::free(p);
}

void operator delete(void* p, std::size_t, std::align_val_t) noexcept {
    std::free(p);
}
```

Josuttis warns about **ABI**, **`aligned_alloc`** requirements (size multiple of alignment), and **matching** every overload your standard library may call (p. 378).

---

## 30.4 Tracking all `::new` calls (p. 380)

Josuttis gives a **complete** tracking example using **`inline` variables** (C++17) to hold counters or logs shared across translation units without ODR pitfalls of naive statics in headers (p. 380).

### Pattern: inline variables for global counters (C++17)

```cpp
// tracking_new.hpp (header)
#pragma once
#include <atomic>
#include <cstddef>

inline std::atomic<std::size_t> g_new_calls{0};
inline std::atomic<std::size_t> g_aligned_new_calls{0};

// In a single .cpp that defines replacement operators, increment inside new.
```

**Why `inline` variables (Josuttis ties to C++17 machinery, p. 380):**

- One **definition** across TUs with **ODR-safe** merge semantics.
- Suitable for **header-only** instrumentation kits.

A full tracker would increment **`g_new_calls`** in `operator new(size_t)` and **`g_aligned_new_calls`** in `operator new(size_t, std::align_val_t)`, then mirror in `delete` for balance checks (p. 380).

---

## Memory layout diagram: default vs over-aligned cells

```
Default heap block for T1 (alignof small)
┌──────────────────────────────┐
│ T1 object                    │
└──────────────────────────────┘

Over-aligned block for T2 (alignof=64)
┌── possibly unused padding ──┬──────────────────────────────┐
│                             │ T2 object (64-byte aligned)  │
└─────────────────────────────┴──────────────────────────────┘
         implementation may store size/metadata outside this view
```

*[Conceptual figure after Josuttis’s discussion of arenas and aligned slots, pp. 366-370.]*

---

## Checklist

- [OK] Rely on **C++17** `new`/`delete` for **`alignas` types** without manual platform align APIs when possible (p. 365).
- [OK] When replacing **global** `new`, supply **aligned** overloads too (p. 378).
- [ERROR] Do not `free` a pointer from **`aligned_alloc`** with the wrong deallocation function.
- [ERROR] Do not assume **`alignof(T) <= __STDCPP_DEFAULT_NEW_ALIGNMENT__`** for all types.

---

## Reference

Josuttis: **Chapter 30**, **pp. 365-383**.
