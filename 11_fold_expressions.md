# Chapter 11: Fold Expressions

**Book:** *C++17 - The Complete Guide*, Nicolai M. Josuttis  
**Chapter:** 11 - Fold Expressions  
**Book page span:** **pp. 107-118** (sections **11.1**-**11.2.3**)

Fold expressions compile **parameter packs** into repeated applications of a binary operator, generating concise code that used to require recursive templates or `(void)initializer_list<int>{(f(args),0)...}` tricks.

---

## 11.1 Motivation (p.108)

### Before folds: recursive instantiation

Historically, summing a pack:

```cpp
// Pre-C++17 style (conceptual)
template<typename T>
constexpr T sum(T v) { return v; }

template<typename T, typename... Ts>
constexpr T sum(T h, Ts... ts) {
  return h + sum(ts...);
}
```

This works, but it stresses compilers with deep instantiations for very large packs and spreads logic across multiple overloads.

### After folds: one expression

```cpp
template<typename... Ts>
constexpr auto sum(Ts... args) {
  return (args + ...);  // unary right fold by '+' (see table below)
}
```

**Book point:** folds are not “runtime loops”; they expand at compile time into a nested expression involving operators.

---

## 11.2 Using Fold Expressions (p.108)

Folds come in **unary** and **binary** forms, and **left** vs **right** variants.

### Unary folds

Given a pack `args` with elements `a1, a2, ..., aN`:

| Form | Expansion (conceptual) |
|------|------------------------|
| Unary left fold `(... op args)` | `((a1 op a2) op a3) ... op aN` |
| Unary right fold `(args op ...)` | `a1 op (a2 op (... op aN))` |

**Parentheses required** around the fold syntax as shown; the `...` placement distinguishes left vs right.

### Why associativity matters (p.108): strings

For non-associative operations (and for efficiency differences), order matters:

```cpp
#include <string>

template<typename... Strings>
std::string concatLeft(Strings&&... s) {
  // Illustrative only: check book for exact recommended pattern / move semantics
  return (std::string{} + ... + std::forward<Strings>(s)); // binary left fold from an initial value
}
```

String concatenation creates temporaries; left vs right folds change **temporary lifetimes** and performance characteristics.

**ASCII expansion compare (three args a,b,c):**

```
Unary LEFT fold:   (... + args)
────────────────────────────────
expands: ((a + b) + c)


Unary RIGHT fold:  (args + ...)
────────────────────────────────
expands: (a + (b + c))
```

For integers with `+`, results match; for strings, costs may differ.

---

## 11.2.1 Empty Parameter Packs (p.109)

### Unary folds and empty packs

If `args` is empty, unary folds have **built-in rules** for some operators:

| Operator op | Empty-pack result |
|-------------|-------------------|
| `&&` | `true` |
| `\|\|` | `false` |
| `,` (comma) | `void()` |
| Most others | **ill-formed** |

So:

```cpp
template<typename... Bs>
bool allTrue(Bs... bs) {
  return (bs && ...);  // empty pack -> true
}

template<typename... Bs>
bool anyTrue(Bs... bs) {
  return (bs || ...);  // empty pack -> false
}
```

### Binary folds (p.109)

Binary folds include an explicit **initial value**:

| Form | Conceptual expansion |
|------|------------------------|
| Binary left `(init op ... op args)` | `(((init op a1) op a2) op ... )` |
| Binary right `(args op ... op init)` | `(a1 op (... op (aN op init)))` |

#### `print()` via binary left fold over `<<` (p.109)

```cpp
#include <iostream>

template<typename... Args>
void print(const Args&... args) {
  // Binary left fold: (((std::cout << a1) << a2) << ...)
  (std::cout << ... << args);
}

// Often you append a terminator in a second statement or as part of another fold:
template<typename... Args>
void println(const Args&... args) {
  (std::cout << ... << args) << '\n';
}
```

**Reading guide:** Reproduce the book’s exact `print` snippet from **p.109** for pedagogical parity; the key idea is the binary fold with `std::cout` as the left “seed”.

Empty-pack behavior for binary folds is discussed relative to the chosen operator and whether an init value exists (**p.109**).

---

## 11.2.2 Supported Operators (p.112)

### Operators

All **binary** operators are foldable **except**:

- Member access `.`
- Pointer member `->`
- Subscript `[]`

### Folded function calls with comma (p.112)

You can invoke an expression for each element using comma folding:

```cpp
template<typename F, typename... Args>
void forEachArg(F f, Args&&... args) {
  (f(std::forward<Args>(args)), ...);
}
```

This runs `f` for each argument; order follows the fold direction chosen.

### Combining hash values: `hashCombine` pattern (p.112)

A common pattern is mixing hashes:

```cpp
#include <cstddef>

inline void hashCombine(std::size_t& seed, std::size_t h) {
  seed ^= h + 0x9e3779b97f4a7c15ULL + (seed << 6) + (seed >> 2);
}

template<typename... Ts>
std::size_t hashVal(const Ts&... vals) {
  std::size_t seed = 0;
  (hashCombine(seed, std::hash<Ts>{}(vals)), ...);
  return seed;
}
```

(Adjust hashing to match your standard library’s best practices; the book cites this as idiomatic “seed mixing”.)

### Folded calls for base classes (p.112)

Variadic CRTP / mixin bases sometimes expose `print()` methods; a fold over comma can call each:

```cpp
template<typename... Bases>
struct Many : Bases... {
  void printAll() {
    (..., Bases::print());
  }
};
```

Note the leading comma usage pattern `(... , expr)` vs `(expr , ...)` variants; match the book’s canonical example to ensure the empty-pack behavior you intend.

### Folded path traversals with `->*` (p.112)

The book shows traversing a chain of pointers-to-members through `->*` in a fold; this is powerful but easy to misuse if precedence is wrong.

---

## 11.2.3 Using for Types (p.117)

### `std::conjunction`-like patterns with packs

You can express “all types equal to `T1`” via folds over `std::is_same_v`:

```cpp
#include <type_traits>

template<typename T1, typename... Ts>
inline constexpr bool allSame = (std::is_same_v<T1, Ts> && ...);
```

This is the “homogeneous pack” predicate used conceptually in **`std::array` deduction guides** (**p.117**, ties back to **Ch. 9** in the book).

---

## Detailed ASCII reference table (forms and expansion)

```
Legend:  a1..aN are pack elements, OP is a binary operator, I is init expression

┌───────────────────────────────┬──────────────────────────────────────────────┐
│ Syntax                        │ Meaning (informal expansion)                 │
├───────────────────────────────┼──────────────────────────────────────────────┤
│ (... OP pack)                 │ Unary LEFT: ((a1 OP a2) OP ...) OP aN        │
│ (pack OP ...)                 │ Unary RIGHT: a1 OP (a2 OP (... OP aN))       │
├───────────────────────────────┼──────────────────────────────────────────────┤
│ (I OP ... OP pack)            │ Binary LEFT: (((I OP a1) OP a2) OP ...)      │
│ (pack OP ... OP I)            │ Binary RIGHT: (a1 OP (... OP (aN OP I)))     │
├───────────────────────────────┼──────────────────────────────────────────────┤
│ Empty unary &&                │ true                                         │
│ Empty unary ||                │ false                                        │
│ Empty unary ,                 │ void()                                       │
│ Other empty unary OP          │ ill-formed                                   │
└───────────────────────────────┴──────────────────────────────────────────────┘
```

### Precedence reminder (practical debugging)

```
Folds expand into raw operator expressions.
If you mix <<, +, and comparisons, add parentheses EXPLICITLY.
```

---

## Exercises (optional)

1. Implement `sum`, `product`, `allTrue` with unary folds; write static tests for empty pack behavior.
2. Benchmark string concatenation for left vs right folds on large packs.
3. Implement `hashVal` and compare distribution to `boost::hash_combine` patterns.

---

**Primary reference:** Josuttis, *C++17 - The Complete Guide*, Chapter 11, **pp. 107-118**.
