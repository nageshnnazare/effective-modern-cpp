# Chapter 35: Deprecated and Removed Features

Tutorial aligned with *C++17 - The Complete Guide* by Nicolai M. Josuttis. **Page references**: **pp. 409-412**.

C++17 **removes** some legacy features and **deprecates** others to steer code toward safer, clearer modern idioms (p. 409-412).

---

## 35.1 Core language features (p. 409)

### 35.1.1 Dynamic exception specifications (p. 409)

| Form | C++17 status |
|------|----------------|
| `throw()` | **Deprecated** (use **`noexcept`**). Still parseable in C++17 for migration. |
| `throw(T, U, ...)` | **Removed** -- ill-formed. |

**Migration:**

```cpp
// Old (deprecated / removed forms)
// void f() throw();
// void g() throw(std::bad_alloc);

// Modern
void f() noexcept;
void g();  // let noexcept be partial if allocations may throw, or mark noexcept(false) explicitly only if needed
```

---

### 35.1.2 Keyword `register` (p. 409)

- **Removed** as a **storage-class specifier** meaning "put in register".
- **`register`** remains a **reserved keyword** -- you cannot use it as an identifier.

```cpp
// register int x;  // ill-formed in C++17
int y = 0;           // [OK]
```

---

### 35.1.3 `++` for `bool` (p. 410)

Incrementing **`bool`** was already deprecated; **C++17 removes** it.

```cpp
bool b = false;
// b++;  // ill-formed
b = true;  // [OK]
```

---

### 35.1.4 Trigraphs (p. 410)

**Trigraphs** are **removed** in C++17.

**What they were:** sequences starting with `??` that the preprocessor could rewrite into other characters, meant long ago for keyboards missing certain punctuation.

**Mapping (historical only----do not use):**

```
+--------+----------------+
| trigraph | replacement |
+--------+----------------+
| ??=      | #            |
| ??/      | \            |
| ??'      | ^            |
| ??(      | [            |
| ??)      | ]            |
| ??!      | |            |
| ??<      | {            |
| ??>      | }            |
| ??-      | ~            |
+--------+----------------+
```

**Why removed (p. 410):** they **confused programmers** (including security reviews: surprising tokenization), and **real-world use was rare**.

---

### 35.1.5 `static constexpr` redeclaration (p. 410)

Redundant **out-of-class** definitions for **`static constexpr`** data members became unnecessary when **inline** variables were unnecessary in many patterns; C++17 **deprecates** redundant **redeclarations** in certain forms (Josuttis summarizes cleanup around `static constexpr` ODR uses; p. 410).

**Direction:** prefer **`static constexpr`** definitions **inside** the class when integral/enumerable; C++17 **`inline` variables** also reduce TU boilerplate for non-integral cases elsewhere in the standard.

---

## 35.2 Library features (p. 410)

### 35.2.1 `auto_ptr` (p. 410)

**Removed.** Migrate to move-only unique ownership or explicit shared ownership:

- **`std::unique_ptr<T>`** for **exclusive** ownership with **move semantics** (usual `auto_ptr` replacement).
- **`std::shared_ptr<T>`** when **shared** lifetime is intentional (reference counting).

```cpp
// std::auto_ptr<int> p;   // removed
std::unique_ptr<int> q = std::make_unique<int>(42);
std::shared_ptr<int> s = std::make_shared<int>(42);
```

---

### 35.2.2 `random_shuffle()` (p. 410)

**Removed.** Use **`std::shuffle`** with an explicit **uniform random bit generator** (URBG), e.g. **`std::mt19937`** seeded from **`std::random_device`**.

```cpp
#include <algorithm>
#include <random>
#include <vector>

void modern_shuffle(std::vector<int>& v) {
    std::shuffle(v.begin(), v.end(), std::mt19937{std::random_device{}()});
}
```

- [OK] **Determinism:** pass a **fixed seed** if you need reproducible shuffles (tests).
- [OOPS] **`rand()`**-style implicit generators were exactly the problem----`random_shuffle`'s old default was easy to misuse.

---

### 35.2.3 `std::unary_function` / `std::binary_function` (p. 411)

**Removed.** Modern adapters **inherit nothing**; use **`using`** declarations for **`argument_type`**, **`result_type`** only if you truly need compatibility types. Prefer **`std::function`**, transparent function objects, or generic lambdas.

---

### 35.2.4 `ptr_fun()`, `mem_fun()`, binders (p. 411)

**Removed.** Replace with:

- **Lambdas**
- **`std::bind`** (still available, though often discouraged vs `lambda`)
- **`std::function`**

---

### 35.2.5 Allocator support for `std::function` (p. 411)

Certain **`std::function` constructor** forms that took allocators were **removed** because the abstraction could not guarantee **exact** allocator propagation semantics users expected (Josuttis / standard rationale summary; p. 411).

**Practical impact:** if you relied on **`function` with allocator**, redesign storage (custom type-erasing wrapper, PMR-aware alternatives, etc.).

---

### 35.2.6 IOStream aliases (p. 411)

Some **deprecated** `ios`/`streambuf` typedef aliases were marked **deprecated** to migrate code to **`basic_*`** template spellings or modern typedefs (p. 411). Consult your implementation's deprecation warnings.

---

### 35.2.7 Other deprecated items (p. 412)

Josuttis groups additional **deprecations** (not all removed in C++17 yet) steering code toward **explicit traits**, **`invoke`**, and safer utilities:

| Deprecated API | Guidance |
|----------------|----------|
| **`std::iterator` base class template** | Write **`iterator_category`**, **`value_type`**, **`difference_type`**, **`pointer`**, **`reference`** (as needed) **explicitly** in your iterator type instead of inheriting `std::iterator<...>`. |
| **`std::result_of`** | Use **`std::invoke_result`** / **`std::invoke_result_t`** (C++17). |
| **`std::is_literal_type`** | No one-to-one replacement in the type traits system; **narrow** your query (`is_trivially_copyable`, `is_standard_layout`, **`constexpr` friendliness**) to what you actually need. |
| **`std::raw_storage_iterator`** | Prefer **`std::allocator_traits`**, **`uninitialized_*` algorithms**, **`std::vector`**, or containers; avoid raw-storage iterator patterns from C++98. |
| **`std::get_temporary_buffer` / `std::return_temporary_buffer`** | Replace with **`std::vector`**, **`std::unique_ptr<T[]>`**, or custom allocators----these global scratch-buffer hooks are fragile. |
| **`std::not1` / `std::not2`** | Replace with **`std::not_fn`** (C++17), usually wrapping a function object or **`std::function`**. |

```cpp
#include <type_traits>

template <class F, class... Args>
using result_t = std::invoke_result_t<F, Args...>;  // not result_of
```

---

## Summary table: deprecated / removed (Josuttis Chapter 35)

| Item | Status (C++17) | Replacement / guidance |
|------|----------------|-------------------------|
| `throw(Type...)` | Removed | `noexcept` / natural exception spec |
| `throw()` | Deprecated | `noexcept` |
| `register` | Removed as specifier | ordinary automatic `int x` |
| `bool++` | Removed | assign `true`/`false` |
| Trigraphs | Removed | use raw `#`, `\`, etc. |
| Redundant `static constexpr` ODR defs | Deprecated | inline in-class / `inline` variables |
| `std::auto_ptr` | Removed | `std::unique_ptr` |
| `std::random_shuffle` | Removed | `std::shuffle` + URNG |
| `unary_function` / `binary_function` | Removed | self-contained typedefs / lambdas |
| `ptr_fun` / `mem_fun` / old binders | Removed | lambdas / `bind` / function objects |
| `std::function` allocator ctor support | Removed | custom type erasure / redesign |
| `std::iterator` | Deprecated | manual typedefs / CRTP traits |
| `std::result_of` | Deprecated | `std::invoke_result` / `_t` |
| `std::is_literal_type` | Deprecated | targeted trait combo / redesign |
| `std::raw_storage_iterator` | Deprecated | `allocator_traits` / `uninitialized_*` / containers |
| `get_temporary_buffer` / `return_temporary_buffer` | Deprecated | `vector` / `unique_ptr` / allocators |
| `std::not1` / `std::not2` | Deprecated | `std::not_fn` |

*[Rows synthesize §35.1-35.2, pp. 409-412.]*

---

## Migration strategy

1. Enable **high warning levels** and **deprecation warnings** on your compiler.
2. Replace **removed** APIs first -- they fail compilation.
3. Schedule **deprecated** API removal before upgrading to **C++20+**, where some deprecations become removals.

---

## Reference

Josuttis: **Chapter 35**, **pp. 409-412**.
