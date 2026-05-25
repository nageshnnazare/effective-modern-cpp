# Chapter 21: Extensions of Type Traits

*Reference: Nicolai M. Josuttis, **C++17 - The Complete Guide**, Chapter 21, pp. 251-257.*

This chapter documents C++17 extensions to `<type_traits>`: the `_v` variable templates for existing traits, new traits for aggregates, invocation, logical metafunctions, swappability, and the `bool_constant` alias.

---

## 21.1 Type Traits Suffix `_v` (p. 251)

### Motivation

Before C++17, querying a type trait meant spelling `Trait<T>::value`:

```cpp
#include <type_traits>

template<typename T>
void check() {
    if (std::is_const<T>::value) {
        // ...
    }
}
```

C++17 adds **variable templates** (see also inline variables) so each Boolean trait gains a `_v` form that expands directly to the `constexpr bool` value.

### Definition pattern (as in the standard)

For each type trait `std::trait_name` that exposes `static constexpr bool value`, the standard provides:

```cpp
template<typename T>
inline constexpr bool is_const_v = is_const<T>::value;
```

Key points:

- **`inline constexpr` variables** (C++17): a single definition in headers; ODR-safe across translation units.
- **Name**: `trait_v` mirrors `trait::value`.
- **Applicability**: all traits that expose a `value` member of type `bool` get the `_v` alias (e.g. `is_integral_v`, `is_same_v`, `is_base_of_v`, ...).

### Usage examples

```cpp
#include <type_traits>
#include <iostream>

void demo() {
    static_assert(std::is_const_v<const int>);
    static_assert(!std::is_const_v<int>);

    constexpr bool same = std::is_same_v<int, int32_t>;  // platform-dependent typedef

    if constexpr (std::is_integral_v<double>) {
        // never taken
    } else {
        std::cout << "double is not integral\n";
    }
}
```

### Relationship to `_t` and class templates

| Form | Example | Role |
|------|---------|------|
| Class template | `std::remove_const<T>` | Type computation |
| `_t` alias | `std::remove_const_t<T>` | Shorthand for `typename remove_const<T>::type` |
| `_v` variable | `std::is_const_v<T>` | Shorthand for `is_const<T>::value` |

---

## 21.2 New Type Traits (p. 252)

### 21.2.1 `std::is_aggregate<T>`

**Purpose:** Detects whether `T` is an **aggregate** per the language definition (no user-provided/ inherited/ explicit constructors, no private/protected non-static data members, no virtual functions, etc., subject to standard wording).

```cpp
#include <type_traits>

struct Agg { int x; double y; };
struct NonAgg { NonAgg() {} int x; };

static_assert(std::is_aggregate_v<Agg>);
static_assert(!std::is_aggregate_v<NonAgg>);
```

Useful for SFINAE when you want brace-initialization or structured-binding-friendly types.

### 21.2.2 `std::has_unique_object_representations<T>`

**Purpose:** `true` if two objects of type `T` with equal value representations compare equal (no padding traps, no multiple representations for the same value). Informally: **memcmp-safe** equality for hashing raw bytes.

**[OK]** When `has_unique_object_representations_v<T>` is `true`, **reliable hashing** can hash the object representation (e.g. feeding `sizeof(T)` bytes starting at the object address) without colliding distinct values that `operator==` would treat as equal. **[OOPS]** Still verify `true` for your `T` on each platform; padding and floating-point representations can make this `false`.

```cpp
#include <type_traits>

struct Padded {
    char c;
    // possible padding
    int i;
};

static_assert(std::has_unique_object_representations_v<int>);
// Padded may be false on typical implementations due to padding
```

**Note:** Result can be `false` for types with padding or tricky floating-point representations. Prefer for trivially copyable types when designing hashers that read object representation.

### 21.2.3 Invocation traits

| Trait | Meaning (informal) |
|-------|---------------------|
| `std::is_invocable<C, Args...>` | `std::invoke(c, args...)` is well-formed |
| `std::is_invocable_r<R, C, Args...>` | Invocable and return convertible to `R` |
| `std::is_nothrow_invocable<C, Args...>` | `noexcept` `invoke` |
| `std::is_nothrow_invocable_r<R, C, Args...>` | Nothrow + return `R` |
| `std::invoke_result_t<C, Args...>` | Result type of `invoke` |

`std::result_of` is **deprecated** in C++17 in favor of `std::invoke_result` (clearer for pointers-to-member and `invoke` semantics).

```cpp
#include <type_traits>
#include <functional>

struct S {
    int f(double) { return 0; }
};

using MemFn = decltype(&S::f);

static_assert(std::is_invocable_v<MemFn, S*, double>);
static_assert(std::is_same_v<std::invoke_result_t<MemFn, S*, double>, int>);

int add(int a, int b) { return a + b; }
static_assert(std::is_invocable_r_v<long, decltype(add), int, int>);

int foo(int);
static_assert(std::is_same_v<std::invoke_result_t<decltype(foo), int>, int>);
```

### 21.2.4 `std::conjunction`, `std::disjunction`, `std::negation`

These model **compile-time boolean logic** on trait-like types (must be *BooleanTestable* / `_v`-friendly).

- **`conjunction<B1, B2, ...>`:** short-circuit **AND**---if `B1::value` is `false`, later traits are **not instantiated** (important for avoiding ill-formed instantiations).
- **`disjunction<B1, B2, ...>`:** short-circuit **OR**---stops at first `true`.
- **`negation<B>`:** `!B::value`.

**Short-circuit and incomplete types:** Because later operands are not instantiated once the result is fixed, **`conjunction` / `disjunction` avoid touching traits that would be ill-formed or undefined behavior** on incomplete types or in situations where eager `::value` evaluation would instantiate bogus templates. Prefer them over hand-written `&&` / `||` of trait `value`s when the second check only makes sense if the first succeeds. **[ERROR]** Using a trait that requires completeness on an incomplete `T` without short-circuiting can yield **undefined behavior** or hard errors.

```cpp
#include <type_traits>

template<typename T>
struct is_integral_and_signed
    : std::conjunction<std::is_integral<T>, std::is_signed<T>> {};

template<typename T>
constexpr bool is_integral_and_signed_v = is_integral_and_signed<T>::value;

static_assert(is_integral_and_signed_v<int>);
static_assert(!is_integral_and_signed_v<unsigned>);
```

**Short-circuit example (avoid bad instantiation):**

```cpp
#include <type_traits>

template<typename T>
struct has_foo_member {
    template<typename U>
    static auto test(int) -> decltype(std::declval<U>().foo(), std::true_type{});
    template<typename>
    static std::false_type test(...);
    static constexpr bool value = decltype(test<T>(0))::value;
};

template<typename T>
struct safe_check
    : std::conjunction<
          std::is_class<T>,
          std::integral_constant<bool, has_foo_member<T>::value>
      > {};
```

If `T` is not a class, the second operand of a plain `&&` might still be instantiated; `conjunction` avoids evaluating `has_foo_member<T>` when `is_class_v<T>` is false (when structured carefully---adapt pattern to your detection needs).

### 21.2.5 Swappability

| Trait | Meaning |
|-------|---------|
| `is_swappable<T>` | `swap(l, r)` valid in `std`/`found by ADL` for two lvalues of type `T` |
| `is_nothrow_swappable<T>` | `swap` is `noexcept` |
| `is_swappable_with<T, U>` | `swap` between `T` and `U` |
| `is_nothrow_swappable_with<T, U>` | Nothrow cross-type swap |

These align with Swappable requirements used throughout the library.

**`is_swappable` (same type, two lvalues):**

```cpp
#include <type_traits>
#include <string>

static_assert(std::is_swappable_v<std::string>);   // [OK] lvalue swap well-formed
static_assert(!std::is_swappable_v<void>);        // [OK] not swappable
static_assert(!std::is_swappable_v<char[]>);      // [OK] array type not Swappable like this
```

**`is_swappable_with` vs `is_swappable` (rvalues vs lvalues):**

```cpp
#include <type_traits>

// Two rvalues of type int: not swappable as int&& / int&& in the std::swap sense
static_assert(!std::is_swappable_with_v<int, int>);
// Two lvalue references: swappable
static_assert(std::is_swappable_with_v<int&, int&>);
```

`is_swappable<T>` is equivalent to `is_swappable_with_v<T&, T&>`.

### 21.2.6 `std::bool_constant<b>`

Alias for `std::integral_constant<bool, b>`:

```cpp
#include <type_traits>

using ok = std::bool_constant<true>;
using no = std::bool_constant<false>;
static_assert(ok::value == true);
```

---

### Composite traits: pointer detection and integral-or-enum

**`IsPtr` — combine `is_null_pointer`, `is_member_pointer`, `is_pointer`:**

```cpp
#include <type_traits>

struct S {};

template<typename T>
struct IsPtr : std::disjunction<
    std::is_null_pointer<T>,
    std::is_member_pointer<T>,
    std::is_pointer<T>
> {};

template<typename T>
inline constexpr bool IsPtr_v = IsPtr<T>::value;

static_assert(IsPtr_v<int*>);
static_assert(IsPtr_v<void (S::*)()>);
static_assert(IsPtr_v<std::nullptr_t>);
static_assert(!IsPtr_v<int>);
```

**`IsIntegralOrEnum` — `conjunction` / `disjunction` / `negation`:**

```cpp
#include <type_traits>

template<typename T>
struct IsIntegralOrEnum : std::disjunction<
    std::is_integral<T>,
    std::is_enum<T>
> {};

template<typename T>
inline constexpr bool IsIntegralOrEnum_v = IsIntegralOrEnum<T>::value;

// Equivalent style using negation for "not floating-point" is usually redundant here;
// disjunction of integral | enum is the direct idiom Josuttis-style metaprograms use.
template<typename T>
struct IsIntegralOrEnumAlt : std::disjunction<
    std::conjunction<std::is_arithmetic<T>, std::negation<std::is_floating_point<T>>>,
    std::is_enum<T>
> {};

template<typename T>
inline constexpr bool IsIntegralOrEnumAlt_v = IsIntegralOrEnumAlt<T>::value;

enum class E : int { a };

static_assert(IsIntegralOrEnum_v<int>);
static_assert(IsIntegralOrEnum_v<E>);
static_assert(!IsIntegralOrEnum_v<double>);
static_assert(IsIntegralOrEnumAlt_v<int>);
static_assert(!IsIntegralOrEnumAlt_v<double>);
```

---

## Table: C++17 new type traits (summary)

| Trait | Primary use | Typical `_v` |
|------|-------------|--------------|
| `is_aggregate<T>` | Brace-init / structural typing checks | `is_aggregate_v<T>` |
| `has_unique_object_representations<T>` | Memcpy/memcmp style hashing | `has_unique_object_representations_v<T>` |
| `is_invocable<C, Args...>` | Callable with `invoke` | `is_invocable_v<C, Args...>` |
| `is_invocable_r<R, C, Args...>` | Callable + return constraint | `is_invocable_r_v<R, C, Args...>` |
| `is_nothrow_invocable<C, Args...>` | Nothrow call | `is_nothrow_invocable_v<...>` |
| `is_nothrow_invocable_r<R, C, Args...>` | Nothrow + `R` | `is_nothrow_invocable_r_v<...>` |
| `invoke_result<C, Args...>` | Return type of `invoke` | `invoke_result_t<C, Args...>` |
| `conjunction<B...>` | Lazy AND of traits | `conjunction<B...>::value` |
| `disjunction<B...>` | Lazy OR of traits | `disjunction<B...>::value` |
| `negation<B>` | NOT | `negation<B>::value` |
| `is_swappable<T>` | Swappable lvalues | `is_swappable_v<T>` |
| `is_nothrow_swappable<T>` | Nothrow swap | `is_nothrow_swappable_v<T>` |
| `is_swappable_with<T,U>` | Cross-type swap | `is_swappable_with_v<T,U>` |
| `is_nothrow_swappable_with<T,U>` | Nothrow cross swap | `is_nothrow_swappable_with_v<T,U>` |
| `bool_constant<b>` | Readable `integral_constant<bool,b>` | N/A (use `bool_constant`) |

*Note:* `conjunction` / `disjunction` / `negation` are **class templates** with a nested `::value`. There is no separate `std::conjunction_v` type trait in C++17; write `conjunction<B...>::value` or wrap it in your own variable template for project-local brevity.

---

## Quick reference: before / after

```
[OLD] std::is_same<T, U>::value     ->  [C++17] std::is_same_v<T, U>
[OLD] typename std::result_of<F(Args...)>::type
      ->  std::invoke_result_t<F, Args...>   // preferred semantics
```

---

## Further reading (same book)

- Inline variables and variable templates: earlier language chapters tie to the `_v` pattern (Josuttis discusses in context of C++17 library evolution around p. 251).
