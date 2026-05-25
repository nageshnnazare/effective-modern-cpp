# Chapter 34: Common C++17 Settings

Tutorial aligned with *C++17 - The Complete Guide* by Nicolai M. Josuttis. **Page references**: **pp. 407-408**.

This short chapter collects **macro values**, **C compatibility**, **signal handler** constraints, and **forward progress** wording that affect **portable** C++17 code (p. 407-408).

---

## 34.1 Value of `__cplusplus` (p. 407)

### Standard values

When compiling **C++17** in conforming modes, expect:

```cpp
#if __cplusplus == 201703L
// C++17 translation unit
#endif
```

**Historical reference values** (Josuttis, p. 407):

| Standard | Typical `__cplusplus` |
|----------|------------------------|
| C++98    | `199711L`              |
| C++11    | `201103L`              |
| C++14    | `201402L`              |
| C++17    | `201703L`              |

**Feature detection idiom:**

```cpp
#if __cplusplus >= 201703L
#include <optional>
// ...
#endif
```

**Caveats:**

- Some compilers expose **`__cplusplus`** as an **older** value unless you pass flags like **`-std=c++17`** / **`/std:c++17`**; always verify your toolchain docs.
- Prefer **feature-test macros** where available (C++20 formalizes more of this; C++17 code still leans on **`__cplusplus`** + vendor macros).

---

## 34.2 Compatibility to C11 (p. 407)

**C++17** updates normative references so the **C standard** baseline is **C11** rather than **C99** (p. 407).

### Practical impact

- **Headers** like **`<stdalign.h>`**, **`<stdbool.h>`** are **deprecated** in C++ (p. 407); prefer native C++ spellings:

```cpp
alignas(16) int x;     // instead of C header patterns
bool flag = true;      // C++ keyword bool
```

- **Not all C11** library features are **re-required** for C++ in identical form; Josuttis flags **gaps** and **divergences** (p. 407). When mixing **C and C++** TUs, audit **ABI** for `stdint`, `threads`, `aligned_alloc`, etc.

### ASCII note: standards stack

```
C++17  ──references──>  ISO C11  (baseline C library / atomics / threads overlap nuanced)
     \
      \── deprecates some C compatibility headers in favor of C++ keywords (p. 407)
```

---

## 34.3 Dealing with signal handlers (p. 408)

The C++ standard **tightens** what is safe inside a **signal handler** that uses `std::signal` (p. 408):

- Only **lock-free** **`std::atomic`** operations are safe for concurrent signaling patterns as described in the standard's limitations.
- Most **library** and **allocation** functions are **not async-signal-safe** in the POSIX sense; **do not** call arbitrary C++ in handlers.

### Minimal pattern (illustration only)

```cpp
#include <csignal>
#include <atomic>

std::atomic<int> g_sig{0};

extern "C" void handler(int signum) {
    // Must be lock-free for reliable signal handler use:
    static_assert(std::atomic<int>::is_always_lock_free ||
                  g_sig.is_lock_free());
    g_sig.store(signum, std::memory_order_relaxed);
}
```

Josuttis links this to **standard clarification** in C++17 rather than inventing new primitives (p. 408).

---

## 34.4 Forward progress guarantees (p. 408)

C++17 adds **forward progress** guarantees for **threads of execution**:

- A thread must **eventually** make **forward progress** (perform a **visible step**) unless it **blocks** in well-defined ways.
- This prevents "infinite starvation" models forbidden by the standard's concurrency rules (p. 408).

**Intuition for library authors:** spinlocks and **lock-free** structures must avoid **indefinite** starvation assumptions that contradict these rules.

---

## Checklist

- [OK] Gate C++17-only code with **`__cplusplus >= 201703L`** when needed (p. 407).
- [OK] Treat **C11** as the referenced C baseline; migrate off deprecated **C shim headers** (p. 407).
- [OK] Signal handlers: **extremely** limited C++ -- prefer atomics only where lock-free and design reviewed (p. 408).
- [ERROR] Do not treat **any** `std::` API as safe in a signal handler without verifying restrictions (p. 408).

---

## Reference

Josuttis: **Chapter 34**, **pp. 407-408**.
