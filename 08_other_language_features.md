# Chapter 8: Other Language Features

**Book reference:** *C++17 - The Complete Guide* by Nicolai M. Josuttis (pages 61-73)

This chapter collects several smaller but important C++17 changes: **nested namespace** syntax, **specified evaluation order**, relaxed **scoped enum** initialization, **`auto` brace init** fixes, **hex float** literals, **`u8` character literals**, **exception specifications in types**, **`static_assert` without message**, and **`__has_include`**.

---

## 8.1 Nested Namespaces (p. 61)

### Single-line syntax

Instead of:

```cpp
namespace A {
namespace B {
namespace C {
    void f();
} } }
```

C++17 allows:

```cpp
namespace A::B::C {
    void f();
}
```

### Limitation

**`inline` namespace** does not participate in this shorthand in C++17 the way you might wish for `A::inline B::C` -- the book notes **no inline namespace nesting support** in this compact form (evolved in later standards for some related use cases; stick to C++17 facts here).

---

## 8.2 Defined Expression Evaluation Order (p. 62)

### The problem

Historically, many expressions had **unspecified evaluation order** between subexpressions. That led to **undefined behavior** when subexpressions mutated the same object without a sequence point rule.

**Book-style example pitfall:**

```cpp
// Ill-advised pre-C++17 (could be UB depending on argument evaluation)
s.replace(0, 8, "").replace(s.find("even"), 4, "sometimes");
```

If the second `replace` reads `s` while the first has not completed according to the old rules, disaster.

Similarly:

```cpp
std::cout << f() << g() << h();
```

The order of `f()`, `g()`, `h()` was **unspecified**.

### What C++17 guarantees (selected cases)

Evaluation is more constrained for several operators (left-to-right where stated):

* **Indexing:** `e1[e2]` -- `e1` evaluated before `e2`.
* **Member access:** `e1.e2`, `e1->e2`, `e1.*e2`, `e1->*e2` -- `e1` before `e2`.
* **Shift:** `e1 << e2`, `e1 >> e2` -- `e1` before `e2`.
* **Assignment (built-in):** value computations and side effects of the **right-hand side** occur **before** the **write** through the **left-hand side** lvalue for operators such as `=`, `*=`, `/=`, etc. (full list in the book and standard).

**Assignment ordering (ASCII):**

```
  e1 = e2;

  [1] evaluate / compute RHS  e2
  [2] evaluate LHS lvalue      e1
  [3] store RHS value into object designated by e1
```

**`new`:** allocation is **before** evaluation of constructor arguments as specified (simplifies reasoning).

### What stays unspecified

Arguments of **the same function call** are still **unordered** relative to one another:

```cpp
e1.f(a1, a2, a3);   // a1, a2, a3 evaluation order [unspecified]
```

### Guaranteed example with increments

```cpp
int i = 0;
std::cout << ++i << ' ' << --i << '\n';
// C++17: guaranteed output: "1 0" (book)
```

### Backward incompatibility: exceptions and output

Tighter ordering can change observable behavior when exceptions and I/O interleave (book mentions `vector::at()` and output ordering). Code that **relied** on unspecified order may **change** its diagnostics -- treat this as a portability and logging concern.

### ASCII diagram: evaluation order rules (overview)

```
                    expression with subexpressions
                                   │
                                   ▼
              ┌────────────────────────────────────────┐
              │  Is it a single function call's args?  │
              └───────────────────┬────────────────────┘
                                  │
                     yes ◄────────┘────────► no
                      │                      │
                      ▼                      ▼
           ┌──────────────────┐    ┌─────────────────────────┐
           │ Order between a1,│    │ Apply C++17 rules for   │
           │ a2, a3:          │    │ operator / new / etc.   │
           │  [unspecified]   │    │ e.g. e1[e2]: e1 then e2 │
           └──────────────────┘    └────────────┬────────────┘
                                                │
                                                ▼
                                     assignment: RHS before LHS
                                     (built-in compound assign)
```

---

## 8.3 Relaxed Enum Initialization from Integral Values (p. 65)

### Fixed underlying type, brace init from integer

```cpp
enum MyInt : char { };
MyInt i1{42};   // [OK] since C++17
```

### Copy initialization still restricted

```cpp
// MyInt i2 = 42;   // [ERROR] still not allowed
```

Only **direct list initialization** gets the relaxation (book).

### Scoped enums

```cpp
enum class Weekday : int { mon, tue /*...*/ };
Weekday s1{0};  // [OK] since C++17 in direct-list-init
```

### Narrowing still rejected

```cpp
// MyInt i5{42.2};  // [ERROR] narrowing
```

---

## 8.4 Fixed Direct List Initialization with `auto` (p. 66)

### `auto a{42};` is now `int`

Before the defect report / C++17 retroactive fix, some compilers treated:

```cpp
auto a{42};  // was std::initializer_list<int> in C++11 wording -- [FIXED]
```

as `std::initializer_list<int>`. **C++17:** `a` is **`int`**.

### Multi-element braces with `auto` now ill-formed

```cpp
// auto b{1, 2, 3};  // [ERROR] since C++17
```

### Copy initialization unchanged

```cpp
auto c = {42};  // still std::initializer_list<int>
```

This is a **breaking change** relative to early C++11 implementations; Josuttis notes it was **adopted retroactively** even for `-std=c++11` modes in conforming implementations.

---

## 8.5 Hexadecimal Floating-Point Literals (p. 67)

Hex float literals use a **hex significand** and a **decimal exponent** for a **power of two**:

**Form:** `0x` _significand_ `p` _exponent_

Examples:

| Literal | Meaning | Decimal (conceptual) |
|---------|---------|----------------------|
| `0x1p4` | 1 * 2^4 | 16 |
| `0xAp2` | 10 * 2^2 | 40 |
| `0x1.4p+2` | 1.25 * 2^2 | 5 |

Fractional hex digits after `.` represent **binary fractions** (each hex digit is 4 bits).

---

## 8.6 UTF-8 Character Literals (p. 68)

```cpp
auto c = u8'6';   // type is char in C++17 (book)
```

**Prefixes (overview):**

| Prefix | Typical width / encoding |
|--------|---------------------------|
| `u8` | UTF-8, stored in `char` (single code unit for ASCII) |
| `u` | UTF-16 unit (`char16_t`) |
| `U` | UTF-32 unit (`char32_t`) |
| `L` | wide character (`wchar_t`) |

String literals have related prefixes; character literals follow analogous rules with distinct types.

---

## 8.7 Exception Specifications as Part of the Type (p. 69)

### Different types

These are **different function types**:

```cpp
void f() noexcept;
void g();
```

You **cannot** bind a pointer to `void() noexcept` with an address of `void()` that might throw:

```cpp
void (*fp)() noexcept = /* &f */ /* [OK] */;
// fp = g_might_throw;  // [ERROR]
```

### No overload by exception spec alone

You cannot overload **only** by `noexcept` on an otherwise identical signature.

### Conditional `noexcept`

```cpp
template<typename T>
void h() noexcept(sizeof(int) < 4);
```

### Generic library impact

Type traits and specializations multiply: Josuttis mentions **`std::is_function`** needing **48 partial specializations** in C++17, **doubled from 24**, because exception specifications become part of the type system in a finer-grained way.

---

## 8.8 Single-Argument `static_assert` (p. 72)

The message string is **optional**:

```cpp
template<typename T>
void require_default_constructible() {
    static_assert(std::is_default_constructible_v<T>);
}
```

---

## 8.9 Preprocessor Condition `__has_include` (p. 73)

```cpp
#if __has_include(<filesystem>)
#  include <filesystem>
#endif
```

**Pure preprocessor** feature: not usable as a normal constant expression in source code outside `#if`.

**Typical use:** probe optional headers across platforms before including.

---

## Chapter map (page anchor)

| Section | Topic | Approx. page |
|--------|--------|--------------|
| 8.1 | `namespace A::B::C` | p. 61 |
| 8.2 | Evaluation order | p. 62 |
| 8.3 | Enum from integrals | p. 65 |
| 8.4 | `auto{}` fix | p. 66 |
| 8.5 | Hex floats | p. 67 |
| 8.6 | `u8` char literal | p. 68 |
| 8.7 | Exception spec in type | p. 69 |
| 8.8 | `static_assert` | p. 72 |
| 8.9 | `__has_include` | p. 73 |

---

## Further practice

1. Write a small `switch` that needs `[[fallthrough]]` and compile with `-Wimplicit-fallthrough`.
2. Experiment with `MyInt{42}` vs `MyInt i = 42` and read the diagnostics.
3. Use `std::is_function_v<void() noexcept>` vs `void()` in a trait probe.

All examples should be validated with **`-std=c++17`** or newer on your compiler.
