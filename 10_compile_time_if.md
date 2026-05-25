# Chapter 10: Compile-Time `if`

**Book:** *C++17 - The Complete Guide*, Nicolai M. Josuttis  
**Chapter:** 10 - Compile-Time `if`  
**Book page span:** **pp. 95-105** (sections **10.1**-**10.4**)

This tutorial explains `if constexpr` as a **template metaprogramming control structure** that **discards** non-taken branches at template instantiation time, unlike runtime `if` which still requires all branches to be **valid C++** for the instantiation.

---

## 10.1 Motivation for Compile-Time `if` (p.96)

### The classic problem: runtime `if` does not disable compilation

Consider a naive generic `asString` sketch where you want:

- arithmetic types [OK] -> `std::to_string`
- `std::string` [OK] -> pass through / return directly
- otherwise [ERROR] -> maybe `static_assert`

If you write ordinary runtime `if` with type traits **inside a function template**, **both branches must compile** for every `T`, because the compiler still parses and instantiates the full function body unless you cut instantiation off.

**Broken shape (conceptual):**

```cpp
#include <string>
#include <type_traits>

template<typename T>
std::string badAsString(T val) {
  if (std::is_same_v<std::decay_t<T>, std::string>) {
    return val;  // ill-formed when T is int, even if "never runs"
  } else if (std::is_arithmetic_v<std::decay_t<T>>) {
    return std::to_string(val);
  } else {
    return val;  // also ill-formed for many T
  }
}
```

**Fix direction (p.96):** use `if constexpr` so the compiler **discards** branches that cannot apply for a given `T`.

```cpp
#include <string>
#include <type_traits>

template<typename T>
std::string asString(T&& val) {
  if constexpr (std::is_same_v<std::decay_t<T>, std::string>) {
    return std::forward<T>(val);
  } else if constexpr (std::is_arithmetic_v<std::decay_t<T>>) {
    return std::to_string(val);
  } else {
    // Could static_assert here if unreachable for unsupported T
    static_assert(!sizeof(T*), "unsupported type");
    return {};  // unreachable in well-formed programs if static_assert fires
  }
}
```

**Reading guide:** Replace the `static_assert` line with your preferred policy (compile-time error message, constraints / `requires`, etc.) per your codebase standards.

### Mental model

```
Runtime if: every branch must type-check in the instantiated template
──────────────────────────────────────────────────────────────────────
  instantiation for T = int
     ├─ then-branch must compile
     └─ else-branch must compile


Compile-time if: non-selected branches are DISCARDED (not instantiated)
────────────────────────────────────────────────────────────────────────
  instantiation for T = int
     └─ only compatible branches participate in instantiation
```

---

## 10.2 Using Compile-Time `if` (p.98)

### Mixing with runtime conditions (p.98)

You can nest runtime predicates inside `if constexpr` arms or combine them as needed, provided you respect the instantiation rules.

```cpp
template<typename T>
void demo(bool flag, T x) {
  if constexpr (sizeof(T) > 1) {
    if (flag) {
      // runtime if inside a constexpr-selected branch
    }
  } else {
    // ...
  }
}
```

### Not a preprocessor replacement (p.98)

`if constexpr` is **not** `#if`. It exists in function bodies (and certain other contexts per standard evolution) and participates in **C++ semantics**, including scoping, name lookup rules, and instantiation. You cannot use it “top-level” as if it were the preprocessor.

---

## 10.2.1 Caveats (p.98)

### Caveat A: impacts `return` type deduction with `auto`

If different `if constexpr` branches return **different types**, `auto` / `decltype(auto)` deduction for the function can fail or become surprising because the compiler still reasons about **what `return` statements remain** after discarding.

**Pattern:** keep return type consistent, or use explicit trailing return type, or structure with helpers.

### Caveat B: `else` matters even if `then` always returns

If after discarding you still have **two `return` statements** that produce incompatible deduction, you can get **ambiguous return type deduction** errors. Adding a trailing `else` / restructuring ends the ambiguity (book discussion around **p.98**).

Sketch:

```cpp
template<typename T>
auto foo(T) {              // careful with auto return
  if constexpr (std::is_integral_v<T>) {
    return 0;
  }
  // Missing else branch can still interact poorly with other returns depending on code
  return 0.0;
}
```

(Adapt the book’s precise example; treat this as a hazard checklist.)

### Caveat C: NO short-circuiting of instantiation for `&&` in the condition

This is a **major** gotcha:

```cpp
struct MyType { static constexpr int i = 42; };

template<typename T>
void bad() {
  // [ERROR] pattern: T::i is parsed in an ill-formed way for many T
  // if constexpr (std::is_same_v<MyType, T> && T::i == 42) { }
}

template<typename T>
void good() {
  if constexpr (std::is_same_v<MyType, T>) {
    if constexpr (T::i == 42) {
      // [OK]: inner depends on T only after MyType match
    }
  }
}
```

**ASCII rule card:**

```
WRONG:  if constexpr (A<T> && B<T>)   // B<T> must still be valid for discarded path? (no short-circuit)
        ────────────────────────────
                │
                └── Often ill-formed: dependent parts still seen

RIGHT:  if constexpr (A<T>) {
            if constexpr (B<T>) { ... }
        }
```

**Book point:** Template metaprogramming `&&` does **not** grant the “don’t compile rhs” protection you expect from runtime boolean `&&` during instantiation the way novices expect.

### Caveat D: discarded statements still have constraints (p.98)

Even discarded branches:

- Must be **syntactically** valid as a template (with some exceptions around certain dependent contexts).
- **Non-dependent** names are still checked.

So you cannot hide an obviously ill-formed construct behind `if constexpr (false)` if it uses non-dependent illegal code.

### Caveat E: `static_assert(false)` is not a generic “disable branch” trick (p.98)

A naively written `static_assert(false)` can fail **even if** the branch appears discarded, when it is not dependent in the right way. Prefer **dependent** `static_assert` patterns like `static_assert(!sizeof(T))` inside constrained contexts, or C++20 concepts, etc.

---

## 10.2.2 Other Examples (p.101)

### Perfect return of generic values: handling `void`

Generic wrappers often need to “maybe return” a value unless it is `void`.

Illustrative structure (book pattern):

```cpp
#include <utility>
#include <type_traits>

template<typename Callable, typename Arg>
decltype(auto) invokeMaybe(Callable&& c, Arg&& arg) {
  if constexpr (std::is_void_v<std::invoke_result_t<Callable, Arg>>) {
    std::forward<Callable>(c)(std::forward<Arg>(arg));
  } else {
    return std::forward<Callable>(c)(std::forward<Arg>(arg));
  }
}
```

### Tag dispatching replacement: `advance` sketch (p.101)

**Before:** multiple overloads `advanceImpl(...)` selected via iterator tag types.

**After:** one function with `if constexpr` over iterator category checks.

Conceptual sketch:

```cpp
#include <iterator>
#include <type_traits>

template<typename Iterator, typename Dist>
void advanceLike(Iterator& it, Dist n) {
  using Cat = typename std::iterator_traits<Iterator>::iterator_category;

  if constexpr (std::is_same_v<Cat, std::random_access_iterator_tag>) {
    it += n;
  } else if constexpr (std::is_same_v<Cat, std::bidirectional_iterator_tag>) {
    // ...
  } else {
    // input/output iterator path
    // ...
  }
}
```

**Trade-off:** tag dispatch remains valuable for extension in some libraries; `if constexpr` trades **fewer symbols** for **less open** customization unless you mix approaches.

---

## 10.3 Compile-Time `if` with Initialization (p.103)

C++17 allows the `if` / `if constexpr` form with init-statement:

```cpp
template<typename T>
void demoInit(T x) {
  if constexpr (auto obj = x; std::is_same_v<decltype(obj), T>) {
    (void)obj;
  }
}
```

Here `obj` lives in both the condition arm’s scope rules (as usual for init-statement `if`).

For **value-based** constexpr checks, prefer conditions that are manifestly constant for the instantiation:

```cpp
template<typename T>
void demoValue(T) {
  if constexpr (sizeof(T) == 4) {
    // ...
  }
}
```

The book discusses init-statement forms such as `if constexpr (auto obj = foo(x); ...)` on **p.103**; keep your condition in the “converted constant expression” rules of your language mode.

---

## 10.4 Using Outside Templates (p.104)

`if constexpr` can appear in non-template functions, but **discarding applies only when the condition is itself a value-dependent constexpr context** as described in the book.

**Critical consequence:** you **cannot** use non-template `if constexpr` to “delete” ill-formed non-dependent code the way template instantiation discarding does.

**Benefit called out (p.104):** even where discarding is limited, **discarded code may not become part of the executable** when the condition folds to a compile-time constant.

---

## ASCII diagram: compile-time if flow

```
Parse template function
        │
        ▼
Instantiate for T = T0
        │
        ▼
Evaluate if constexpr condition (compile-time)
                        │
        ┌───────────────┴────────────────┐
        │ true                           │ false
        ▼                                ▼
Selected branch                   Other branch
instantiated normally              DISCARDED (mostly not instantiated)
        │                                │
        ▼                                ▼
Continue                           (still: syntax / non-dependent checks)
```

---

## Practical checklist

1. Use `if constexpr` when you need **different code** for different `T` and runtime `if` would force **all branches** to compile (**p.96**).
2. Nest `if constexpr` instead of `&&` when the RHS depends on prior traits (**p.98**).
3. Watch **`auto` return** interplay with multiple returns (**p.98**).
4. Do not treat `static_assert(false)` as a magic off-switch (**p.98**).
5. `if constexpr` + init-statements can localize temporaries (**p.103**).

---

**Primary reference:** Josuttis, *C++17 - The Complete Guide*, Chapter 10, **pp. 95-105**.
