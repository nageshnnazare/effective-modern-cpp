# Chapter 28: Other Small Library Features and Modifications

*Reference: Nicolai M. Josuttis, **C++17 - The Complete Guide**, Chapter 28, pp. 327-336.*

This chapter collects **miscellaneous** library tweaks: `uncaught_exceptions`, `shared_ptr` extensions for arrays and casts, `weak` helpers, numeric/chrono additions, and broader `constexpr` / `noexcept` marking across the standard library.

---

## 28.1 `std::uncaught_exceptions()` pattern (pp. 327-329)

```cpp
#include <exception>

int n = std::uncaught_exceptions();  // number of active exceptions being unwound
```

**Difference vs the deprecated singular API**

- **`std::uncaught_exception()` (deprecated / removed in C++17):** returned **`bool`**: "is there any active unwinding?" Nested exceptions made this brittle when you needed a **stack depth**-style answer.
- **`std::uncaught_exceptions()` (since C++11; idiomatic in C++17):** returns an **integral count** of uncaught exceptions currently in flight---- **`0`** in normal execution, increments as unwinding proceeds.

### `UncaughtExceptionDetector` class (complete pattern)

**Idea:** the **constructor** stores **`std::uncaught_exceptions()`** at scope entry; the **destructor** compares the **current** count. If the current count is **greater** than the saved value, this object's destructor is running **while unwinding** an exception that started **after** construction----often you must **suppress** logging that throws, **avoid** `throw` in the destructor, or branch transaction commit vs rollback.

```cpp
#include <exception>

class UncaughtExceptionDetector {
    const int entry_count_;
public:
    UncaughtExceptionDetector()
        : entry_count_(std::uncaught_exceptions()) {}

    ~UncaughtExceptionDetector() {
        if (std::uncaught_exceptions() > entry_count_) {
            // destructor reached during stack unwind after construction
            // [OK] e.g. swallow secondary errors, avoid throw, mark rollback-only path
        } else {
            // normal destructor (no new in-flight uncaught exception vs construction)
            // [OK] success-path cleanup may still throw if your design allows
        }
    }

    UncaughtExceptionDetector(const UncaughtExceptionDetector&) = delete;
    UncaughtExceptionDetector& operator=(const UncaughtExceptionDetector&) = delete;
};
```

**Replaces:** **[OOPS]** any remaining use of **`uncaught_exception()`** (singular)----migrate to **`uncaught_exceptions()`** and the **count-delta** idiom.

**Use cases:**

- Logging / RAII guards that must not throw in destructor during exception handling.
- Transaction rollbacks that distinguish success path cleanup vs failure.

---

## 28.2 Shared Pointer Improvements (p. 329)

Header: `<memory>`.

---

### 28.2.1 Shared pointers to raw C arrays (p. 329-330)

**Before C++17:** owning a raw array with **`shared_ptr`** required a **custom deleter** that called **`delete[]`**. [ERROR] `std::shared_ptr<int>(new int[10])` uses **`delete`** on the first element----**undefined behavior**.

```cpp
// Pre-C++17 style (still valid conceptually)
auto p_old = std::shared_ptr<int>(new int[10], [](int* x) { delete[] x; });
```

**Since C++17:** use the **`shared_ptr<T[]>`** partial specialization---- **`delete[]`** is selected automatically; **`operator[]`** is provided for indexing.

```cpp
#include <memory>

std::shared_ptr<int[]> p{new int[10]};
p[3] = 42;
```

```cpp
std::shared_ptr<std::string[]> ps{new std::string[10]};
ps[0] = "example";
```

- [OK] **Subscript** syntax **`p[i]`** for array specializations.
- [OOPS] There is still **`no`** `std::make_shared` for arbitrary **C arrays** with the same ergonomics as scalars in C++17----you may **`new T[n]`** into the smart constructor as shown, or keep **`vector`/`unique_ptr<T[]>`** when that fits better.

**Preferred:** `std::make_shared` is not provided for C arrays in C++17 the same way as scalars; initialize carefully.

---

### 28.2.2 `reinterpret_pointer_cast` (p. 330)

Cast between `shared_ptr` instantiations using `reinterpret_cast` on the stored pointer.

**Typical uses:** low-level interoperability where the same owned storage must be viewed under another type. This has the same preconditions as `reinterpret_cast` on raw pointers: **alignment**, **lifetime**, and **strict-aliasing** rules still apply; the reference counts stay synchronized because the `shared_ptr` control block is shared.

Consult the standard wording and Josuttis **Section 28.2.2 (p. 330)** before applying this in production; a one-size-fits-all code snippet is intentionally omitted here because correctness is entirely domain-specific.

---

### 28.2.3 `weak_type` (p. 330)

```cpp
#include <memory>

static_assert(
    std::is_same_v<
        std::shared_ptr<int>::weak_type,
        std::weak_ptr<int>
    >);
```

Enables generic code to name `weak_ptr` matching a given `shared_ptr` without repeating `T`.

---

### 28.2.4 `weak_from_this` (pp. 330-332)

**Purpose:** inside a type derived from **`std::enable_shared_from_this<T>`**, obtain a **`std::weak_ptr<T>`** to **`*this`** without bumping the **strong** reference count.

**Why that matters:** **`shared_from_this()`** always promotes to **`shared_ptr`**----handy, but inappropriate when you only need a **non-owning** handle (e.g. registration in a callback registry). **`weak_from_this()`** is ideal when **circular `shared_ptr` graphs** would otherwise leak or when callees must **detect lifetime expiry**.

```cpp
#include <memory>

// Only valid once `std::shared_ptr<Node>` already owns *this (typical construction pattern).
struct Node : std::enable_shared_from_this<Node> {
    std::weak_ptr<Node> weak_self() { return weak_from_this(); }

    void register_callback() {
        std::weak_ptr<Node> w = weak_from_this();
        // Store handler that locks w before use; [OK] breaks cycles vs capturing shared_from_this()
    }
};
```

Calling `weak_from_this()` **before** the object is owned by a `shared_ptr` has undefined behavior (same precondition family as `shared_from_this()`).

---

## 28.3 Numeric Extensions (p. 332)

Header: `<numeric>` / `<cmath>` (respectively).

### `std::gcd` / `std::lcm`

```cpp
#include <numeric>

constexpr int g = std::gcd(48, 18);   // 6
constexpr int l = std::lcm(48, 18);   // 144
```

Works for integer types; `constexpr` friendly.

### `std::hypot` three-argument

```cpp
#include <cmath>

double r = std::hypot(3.0, 4.0, 12.0);  // sqrt(3^2+4^2+12^2)
```

### Mathematical special functions (C++17, pp. 332-334)

C++17 adds a large family of **named** functions (historically TR1-minded) to **`<cmath>`** for scientific / engineering work.

**Header:** **`<cmath>`** (also available under the C names in `<math.h>` / globally depending on your inclusion style).

**Complete inventory (Josuttis summary / ISO naming):**

| Category | Functions |
|----------|-----------|
| Associated orthogonal polynomials | **`assoc_laguerre`**, **`assoc_legendre`**, **`laguerre`**, **`legendre`** |
| Elliptic integrals | **`comp_ellint_1`**, **`comp_ellint_2`**, **`comp_ellint_3`**, **`ellint_1`**, **`ellint_2`**, **`ellint_3`** |
| Beta | **`beta`** |
| Bessel / Neumann / spherical | **`cyl_bessel_i`**, **`cyl_bessel_j`**, **`cyl_bessel_k`**, **`cyl_neumann`**, **`sph_bessel`**, **`sph_neumann`** |
| Exponential integrals / zeta | **`expint`**, **`riemann_zeta`** |
| Hermite polynomials | **`hermite`** |

**Representative prototypes (all in namespace `std`, overloads for `float` / `double` / `long double`):**

```
assoc_laguerre    assoc_legendre    beta
comp_ellint_1     comp_ellint_2     comp_ellint_3
cyl_bessel_i      cyl_bessel_j      cyl_bessel_k      cyl_neumann
ellint_1          ellint_2          ellint_3
expint            hermite           laguerre         legendre
riemann_zeta      sph_bessel        sph_neumann
```

- [OK] These wrap **ISO C / POSIX**-flavored math where the platform provides them.
- [OOPS] **Availability & accuracy** still depend on **STL vendor + OS math library + compiler flags**; freestanding / embedded builds may omit subsets.

---

## 28.4 `chrono` extensions: `floor` / `ceil` / `round` / `abs` (pp. 334-335)

`std::chrono::floor`, **`ceil`**, **`round`**, and **`abs`** are function templates that operate on **`duration`** and **`time_point`** values, converting toward a coarser or finer tick period without manual integer-divide boilerplate.

**Semantics (Josuttis p. 334-335):**

- **`std::chrono::floor<ToDuration>(d)`:** rounds the **duration** toward **minus infinity** (algebraic floor on the tick count after conversion).
- **`std::chrono::ceil<ToDuration>(d)`:** rounds toward **plus infinity**.
- **`std::chrono::round<ToDuration>(d)`:** rounds to **nearest** representable tick count in `ToDuration`; **ties to even** (banker's rounding) on the resulting integer tick.
- **`std::chrono::abs(d)`:** **absolute value** of a **duration** (if the duration is signed). [ERROR] Applying **`abs`** to an **unsigned** duration specialization is not the use case; check your `duration::rep`.

```cpp
#include <chrono>
#include <iostream>

int main() {
    using namespace std::chrono_literals;
    auto d = 3min + 1270ms;

    auto floored_to_sec = std::chrono::floor<std::chrono::seconds>(d);   // toward -inf
    auto ceiled_to_sec  = std::chrono::ceil<std::chrono::seconds>(d);    // toward +inf
    auto rounded_to_sec = std::chrono::round<std::chrono::seconds>(d);  // nearest, ties to even

    std::cout << "floor: " << floored_to_sec.count() << "s\n";
    std::cout << "ceil:  " << ceiled_to_sec.count()  << "s\n";
    std::cout << "round: " << rounded_to_sec.count() << "s\n";

    using Sec = std::chrono::seconds;
    Sec a{10}, b{-25};
    auto ad = std::chrono::abs(a - b);  // magnitude of difference as duration
    std::cout << "abs: " << ad.count() << "s\n";
}
```

**Why:** cleaner rounding of timestamps for logging / scheduling vs manual integer division tricks.

---

## 28.5 `constexpr` Extensions (p. 335)

Many library functions (not only language core) became `constexpr` in C++17. Examples (depending on standard version / clause):

- More `<algorithm>` / `<utility>` / container member functions usable in constant evaluation contexts.

Consult Josuttis **Section 28.5** for edition-specific enumerations: the trend is toward compile-time containers and algorithms when iterator operations are `constexpr`.

---

## 28.6 `noexcept` Extensions (p. 336)

Additional standard functions marked `noexcept` where specifications guarantee non-throwing (e.g., move operations on certain types, swap, destructors). This strengthens:

- `noexcept` operator queries in generic code.
- Strong exception guarantees for containers.

---

## Table: small library improvements (checklist)

| Feature | Header | Brief description |
|---------|--------|-------------------|
| `uncaught_exceptions()` | `<exception>` | Count active unwinding exceptions |
| `shared_ptr<T[]>` | `<memory>` | Array delete, `operator[]` |
| `reinterpret_pointer_cast` | `<memory>` | `reinterpret` for `shared_ptr` |
| `weak_type` | `<memory>` | Nested `weak_ptr` type alias |
| `weak_from_this` | `<memory>` | Weak self from `enable_shared_from_this` |
| `gcd` / `lcm` | `<numeric>` | Integer math helpers |
| `hypot(x,y,z)` | `<cmath>` | 3D Euclidean norm |
| Special math fns | `<cmath>` | Elliptic integrals, Bessel, etc. |
| `chrono::floor/ceil/round/abs` | `<chrono>` | Rounding durations / time points |
| More `constexpr` | many | Compile-time evaluation |
| More `noexcept` | many | Stronger noexcept contracts |

---

## Cross-reference

- **Chapter 25:** `clamp`, `sample`, `data` utilities (pp. 299-307).
- **Chapter 26:** container handle and string `data` (pp. 309-319).
