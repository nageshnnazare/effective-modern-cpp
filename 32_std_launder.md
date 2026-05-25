# Chapter 32: `std::launder()`

Tutorial aligned with *C++17 - The Complete Guide* by Nicolai M. Josuttis. **Page references**: **pp. 393-398**.

Header: **`<new>`**. `std::launder` is a **low-level** tool that interacts with the C++ **object model** and **compiler optimizations**. It is primarily needed when **replacing an object** at a **fixed address** (often via **placement new**) where **`const`** or **reference** members cause the compiler to assume **immutable facts** about the old object's memory that **no longer hold** after **lifetime** changes (p. 393-396).

---

## 32.1 Motivation (p. 393)

### The optimization problem

Consider a type with a **`const`** member:

```cpp
#include <new>

struct X {
    const int n;
    explicit X(int v) : n{v} {}
};

void broken_assumptions() {
    alignas(X) unsigned char storage[sizeof(X)];
    X* p = ::new (storage) X{7};

    // Destroy and reconstruct at same address:
    p->~X();
    ::new (storage) X{42};
}
```

**Naive mental model:** `p` still points at the **new** `X`, so `p->n` should be **42**.

**Compiler model:** For **`const` members**, optimizers may legally **cache** `p->n` in registers or reuse a load across operations, **because** between well-defined access to **`p->n`**, the abstract machine promises **`const` subobjects** are immutable **for the current object's lifetime** under the rules they applied at that optimization point.

After placement-new **without** updating the **provenance** of pointers the compiler tracked, reads through **`p`** may still "see" **7** (p. 393).

**Analogous issue:** **reference** members similarly constrain what the compiler may assume about **aliasability** of subobjects (Josuttis ties this to object lifetimes; p. 393).

---

## ASCII diagram: stale compiler knowledge vs fresh object

```
Storage bytes at address A
┌───────────────────────────────────────┐
│ lifetime 1: X with const n == 7       │  Compiler may prove: load n -> 7
└───────────────────────────────────────┘
                |
                | placement new X{42}
                v
┌───────────────────────────────────────┐
│ lifetime 2: X with const n == 42      │  True dynamic value changed
└───────────────────────────────────────┘

Pointer p still numerically equals A, but *unless* the compiler is told to
re-establish the object identity it may reuse cached facts about lifetime 1.

std::launder(p) yields pointer to the *current* object of type X at A (p. 396).
```

---

## 32.2 How `launder()` solves the problem (p. 396)

### Core idea

```cpp
template <class T>
constexpr T* launder(T* p) noexcept;
```

**Effect:** `std::launder(p)` is a **barrier** that **rebinds** the compiler's **abstract object tracking** for a pointer into a **new object** that **occupies** the storage **`p` addresses** under strict preconditions (notably same **`T`**, compatible storage, no **const** access problems beyond what launder resolves).

### Correct usage sketch (conceptual)

```cpp
#include <new>

struct X {
    const int n;
    explicit X(int v) : n{v} {}
};

void with_launder() {
    alignas(X) unsigned char storage[sizeof(X)];
    X* p0 = ::new (storage) X{7};
    p0->~X();
    X* p1 = ::new (storage) X{42};
    X* q = std::launder(p1);
    int v = q->n;  // [OK] fresh read under Josuttis / standard rules (p. 396)
    (void)v;
}
```

Josuttis describes `launder` as **"laundering"** the pointer through an **optimization barrier** so the compiler cannot keep stale assumptions (p. 396).

**Note:** The illustrative pattern must follow **strict lifetime rules** -- you may not access **`p0->n`** after destruction in ways that end the lifetime incorrectly; the chapter stresses correctness of **placement new sequences** (p. 393-398).

---

## 32.3 Why / when `launder()` does not work (p. 397)

Josuttis lists important **limitations** (p. 397):

1. **Different types** at the same address:If you end the lifetime of **`T`** and create **`U`** in the same storage, **`launder` as `T*`** does not magically bridge **cross-type** reuse for general cases.

2. **Stricter alignment or size**: If the **new** object type has **stronger alignment** or **different size** requirements that violate storage assumptions, **`launder`** cannot rescue an invalid reuse.

3. **Not for everyday code**: Typical application developers rarely need **`launder`**; **containers** and **allocators** in the standard library implementation may need it when juggling **typed** nodes in **untyped** buffers (p. 397).

### Decision table (summary)

| Scenario | `launder` helpful? |
|----------|--------------------|
| Same type `T`, legal placement new cycle, **`const` / ref** confusion | Often **yes** |
| Reuse storage for **different** type | **No** (not a `launder` fix) |
| Alignment / storage not suitable for new object | **No** |
| Ordinary heap `new`/`delete` without tricky aliasing | **No** |

---

## Relationship to `std::start_lifetime_as` (note)

Later standards introduce additional facilities related to object lifetime and storage reuse. In **C++17**, **`std::launder`** is the primary spelling discussed in Josuttis for this problem class (p. 396).

---

## Reference

Josuttis: **Chapter 32**, **pp. 393-398**.
