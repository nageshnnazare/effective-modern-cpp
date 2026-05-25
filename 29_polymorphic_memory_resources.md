# Chapter 29: Polymorphic Memory Resources (PMR)

Tutorial aligned with *C++17 - The Complete Guide* by Nicolai M. Josuttis (Addison-Wesley). **Page references** use **p.** for this chapter’s span **pp. 341-364**.

C++17 introduces **polymorphic memory resources** in `<memory_resource>` (and allocator-aware containers in corresponding headers, e.g. `<vector>` via `std::pmr::vector`). The core idea: **separate allocation strategy from container type** by passing a `std::pmr::memory_resource*` (or using the **default resource**) instead of encoding the allocator type in the container’s template parameters.

---

## Conceptual overview: memory resource hierarchy

Josuttis motivates PMR by showing that you can keep `std::pmr::vector`, `std::pmr::string`, etc., and only swap the **resource** used for allocation (p. 342, 347).

```
┌─────────────────────────────────────────────────────────────────┐
│                    Client code (containers)                     │
│  std::pmr::vector<T>, std::pmr::string, std::pmr::deque<T>, ... │
└────────────────────────────┬────────────────────────────────────┘
                             │ allocates/deallocates through
                             v
┌─────────────────────────────────────────────────────────────────┐
│              std::pmr::polymorphic_allocator<T>                 │
│ (type-erased interface to memory_resource; same vector<T> type) │
└────────────────────────────┬────────────────────────────────────┘
                             │ calls virtual hooks
                             v
┌─────────────────────────────────────────────────────────────────┐
│                 std::pmr::memory_resource (base)                │
│   do_allocate(), do_deallocate(), do_is_equal()                 │
└──────┬──────────────┬──────────────┬──────────────┬─────────────┘
       │              │              │              │
       v              v              v              v
┌────────────┐ ┌─────────────┐ ┌──────────────┐ ┌──────────────────┐
│ new_delete │ │ null_memory │ │ monotonic_   │ │ (un)synchronized │
│ _resource  │ │ _resource   │ │ buffer_      │ │ _pool_resource   │
└────────────┘ └─────────────┘ └──────────────┘ └──────────────────┘
     global         throws       fast bump       pooled chunks
     new/delete    bad_alloc      allocator      (thread opt.)
```

*[Diagram: conceptual hierarchy as in Josuttis’s presentation of PMR, pp. 341-349.]*

---

## 29.1 Using standard memory resources (p. 342)

### Motivation: custom allocation without changing container type

**Traditional approach:** `std::vector<int, MyAlloc<int>>` changes the **type** of the vector. Code that only knows `std::vector<int>` cannot interoperate without templates or conversions.

**PMR approach:** `std::pmr::vector<int>` always uses a **`polymorphic_allocator<int>`**, which holds a pointer to a `memory_resource`. You can construct the vector with `&pool` where `pool` is any `memory_resource` subclass. The **static type** stays `std::pmr::vector<int>`; **behavior** depends on the resource (p. 342).

### pmr containers and namespace `std::pmr`

Standard **allocator-aware** typedefs/wrappers include (non-exhaustive):

- `std::pmr::vector`
- `std::pmr::deque`
- `std::pmr::list`
- `std::pmr::forward_list`
- `std::pmr::set`, `std::pmr::map`, multiset/multimap
- `std::pmr::unordered_set`, `std::pmr::unordered_map`
- `std::pmr::string` (typedef to `std::basic_string<char, std::char_traits<char>, std::pmr::polymorphic_allocator<char>>`)

**All PMR facilities** in the standard library for this facility live under **namespace `std::pmr`** (p. 342).

---

## 29.1.1 Motivating example (p. 342)

### `monotonic_buffer_resource` for fast allocation

A **monotonic buffer resource** allocates by bumping a pointer forward in a buffer. **Individual deallocations** are typically **no-ops** (or batched): the whole arena is released when the resource is destroyed or reset (Josuttis details behavior and caveats around upstream resources on **p. 349**).

**Pattern from the book (conceptual):** back a monotonic resource with a **stack buffer**, pass its address to `std::pmr::vector`, and use the vector normally.

```cpp
#include <memory_resource>
#include <vector>
#include <cstddef>

void example_monotonic_inline_buffer() {
    alignas(std::max_align_t) char buf[1000];
    std::pmr::monotonic_buffer_resource pool{buf, sizeof(buf)};

    std::pmr::vector<int> v{&pool};
    v.reserve(200);
    for (int i = 0; i < 100; ++i) {
        v.push_back(i);
    }
    // [OK] Fast pushes: allocator does not round-trip to global heap for every element
    //      in the same way a general-purpose allocator might.
    //
    // When `pool` goes out of scope, memory in `buf` is no longer owned by live objects
    // from a C++ object lifetime perspective only if usage matches the resource rules;
    // Josuttis stresses understanding lifetimes (p. 342-349).
}
```

### Performance intuition: no individual deallocation

For many hot loops, **avoiding per-element deallocation** and reducing synchronization can dominate. A monotonic arena:

- **Allocates** by advancing an offset (very cheap).
- **May not reclaim** partial blocks until the resource is destroyed or you adopt the upstream/reset patterns described in **§29.1.3** (p. 349).

**Trade-off:** if you need **frequent releases** of arbitrary subsets of objects, a monotonic strategy alone may be wrong; **pool resources** may fit better (p. 347).

---

## 29.1.2 Standard memory resources (p. 347)

Josuttis summarizes the **built-in** memory resources:

| Resource | Role (summary) |
|----------|------------------|
| `std::pmr::new_delete_resource()` | Uses **global** `::operator new` / `::operator delete` (p. 347). |
| `std::pmr::null_memory_resource()` | Every allocation throws **`std::bad_alloc`** (p. 347). Useful for enforcing "no heap". |
| `std::pmr::monotonic_buffer_resource` | **Fast** bump-pointer style; **does not** individually deallocate in the usual sense (p. 347). |
| `std::pmr::unsynchronized_pool_resource` | **Pooled** blocks; **not thread-safe** (p. 347). |
| `std::pmr::synchronized_pool_resource` | **Pooled** blocks; **thread-safe** (p. 347). |

### Default resource: `get_default_resource()` / `set_default_resource()`

```cpp
#include <memory_resource>

void demo_default_resource() {
    std::pmr::memory_resource* old = std::pmr::get_default_resource();

    alignas(std::max_align_t) char buf[4096];
    std::pmr::monotonic_buffer_resource local{buf, sizeof(buf)};

    std::pmr::set_default_resource(&local);

    std::pmr::vector<int> v;  // uses current default resource
    v.push_back(1);

    std::pmr::set_default_resource(old);
}
```

**Notes (Josuttis, p. 347 and surrounding):**

- The **default resource** is a process-wide pointer (thread interactions and ordering are your responsibility when changing it).
- PMR containers default-constructed without an explicit resource use **`get_default_resource()`**.

---

## 29.1.3 Memory resources in detail (p. 349)

### `monotonic_buffer_resource` with upstream resource

When the **initial buffer** is exhausted, a monotonic resource can obtain more memory from an **upstream** `memory_resource*`. Josuttis explains the chaining model (p. 349).

**Layout intuition:**

```
monotonic_buffer_resource
┌──────────────────────────────────────────────────────────┐
│ current_buffer  ──────>  [  used  |  free...........  ]  │
│ next_offset                                              │
│ upstream  ───> another memory_resource (e.g. new_delete) │
└──────────────────────────────────────────────────────────┘
        │
        │  when initial arena exhausted, may allocate
        │  additional chunks from upstream (p. 349)
        v
   [ optional extension blocks from upstream ]
```

### Pool resources with pool options

`std::pmr::pool_options` can tune **maximum blocks per chunk** and **desired chunk size** (Josuttis, p. 349). Pool resources:

- Serve allocations from **pools** of fixed (or graded) block sizes.
- Coalesce returns into freelists (implementation-defined exact strategy).

**Layout intuition (pools):**

```
synchronized_pool_resource / unsynchronized_pool_resource
┌─────────────────────────────────────────────────────────────┐
│  pool for size class S1   :  [ free list ] <-> [ chunks ]   │
│  pool for size class S2   :  [ free list ] <-> [ chunks ]   │
│  ...                                                        │
│  upstream resource                                          │
└─────────────────────────────────────────────────────────────┘
```

Use **unsynchronized** when **one thread** owns all allocations; use **synchronized** for **shared** pools (p. 347).

---

## 29.2 Defining custom memory resources (p. 355)

Derive from `std::pmr::memory_resource` and override:

1. **`do_allocate(std::size_t bytes, std::size_t alignment)`**  
2. **`do_deallocate(void* p, std::size_t bytes, std::size_t alignment)`**  
3. **`do_is_equal(const memory_resource& other) const noexcept`**

The **public** non-virtual API (`allocate`, `deallocate`, `is_equal`) enforces alignment and forwards to `do_*`.

### Example: tracking allocator (pattern)

```cpp
#include <memory_resource>
#include <cstdlib>
#include <atomic>
#include <cassert>

class tracking_resource : public std::pmr::memory_resource {
    std::pmr::memory_resource* upstream_;
    std::atomic<std::size_t> allocated_{0};
    std::atomic<std::size_t> deallocated_{0};

public:
    explicit tracking_resource(std::pmr::memory_resource* u =
                                   std::pmr::get_default_resource())
        : upstream_(u) {}

    std::size_t bytes_allocated() const { return allocated_.load(); }
    std::size_t bytes_deallocated() const { return deallocated_.load(); }

private:
    void* do_allocate(std::size_t bytes, std::size_t alignment) override {
        void* p = upstream_->allocate(bytes, alignment);
        allocated_ += bytes;
        return p;
    }

    void do_deallocate(void* p, std::size_t bytes, std::size_t alignment) override {
        upstream_->deallocate(p, bytes, alignment);
        deallocated_ += bytes;
    }

    bool do_is_equal(const memory_resource& other) const noexcept override {
        return this == &other;
    }
};
```

Josuttis walks through **equality semantics** and when two different objects should compare equal (**§29.2.1**, p. 358).

---

## 29.2.1 Equality of memory resources (p. 358)

**Definition (intuitive):** Two resources are **equal** if memory allocated from one may be **deallocated through** the other **with matching** `bytes` and `alignment`.

**Typical patterns:**

- **Same object:** `&a == &b` implies equality.
- **Singletons** like `new_delete_resource()` returned pointers compare equal per standard.
- **Distinct trackers** wrapping different state: usually **not equal** even if they share upstream, unless you explicitly design otherwise (p. 358).

If `is_equal` is wrong, **undefined behavior** can result when containers move memory between resources incorrectly.

---

## 29.3 PMR support for custom types (p. 360)

### Allocator-aware types and `polymorphic_allocator<T>`

For your own containers or nodes, you can:

- Store `std::pmr::polymorphic_allocator<T>` as the allocator type.
- Obtain `memory_resource*` via `.resource()` and propagate it to nested structures.

**`polymorphic_allocator<T>`** is **type-erased** at the resource level: it does not template your entire library on `MyAlloc<T>` (p. 360).

### Propagation rules (outline)

Josuttis connects PMR behavior to **`std::allocator_traits` propagation** idioms but specialized for polymorphic allocators: on **copy/move** of allocator-aware types, whether the resource pointer is copied/moved depends on `propagate_on_container_copy_assignment`, etc. For PMR containers in the STL, consult the standard’s ** mandated** behavior per container operation (p. 360).

**Practical guidance:**

- When **moving** `std::pmr::vector`, the allocator (resource pointer) is often **moved** with the representation; **copies** may allocate from the **target’s** resource or use propagate traits -- verify per operation in Josuttis / cppreference for your version.

---

## Quick checklist

- [OK] Use **`std::pmr::`** containers for a uniform static type with pluggable memory.
- [OK] Use **monotonic** arenas for **many small allocations** with **batch free** semantics.
- [OK] Use **pools** when you need **reuse** without global heap chatter; pick **sync or unsync** (p. 347).
- [ERROR] Do not mix **deallocate** across **non-equal** resources.
- [ERROR] Do not let a **monotonic** arena’s backing buffer disappear while objects still **use** it.

---

## Further reading (same chapter)

Josuttis: **Chapter 29**, pp. **341-364** – worked examples, edge cases (alignment, upstream lifetimes, custom resources), and integration guidance for real codebases.
