# Chapter 33: Improvements for Implementing Generic Code

Tutorial aligned with *C++17 - The Complete Guide* by Nicolai M. Josuttis. **Page references**: **pp. 399-404**.

Three small **standard library** tools simplify **generic programming**:

1. **`std::invoke`**: uniform call syntax for callables and **member pointers** (p. 399).
2. **`std::bool_constant`**: ergonomic **boolean** `integral_constant` (p. 401).
3. **`std::void_t`**: helper for **SFINAE**-based detection (p. 403).

Headers: **`<functional>`**, **`<type_traits>`**.

---

## 33.1 `std::invoke<>` (p. 399)

### Purpose

`std::invoke(f, args...)` calls **`f`** with **`args...`** using the same **dispatch rules** as the INVOKE pseudo-operation used by **`std::bind`**, **`std::thread`**, etc.

It handles:

- **Free functions**: `std::invoke(f, a, b)` ~ `f(a, b)`
- **Function objects**: `std::invoke(g, a)`
- **Pointer-to-member functions**: `std::invoke(&C::mf, obj, args...)`
- **Pointer-to-member data**: `std::invoke(&C::md, obj)` ~ member access

### Examples

```cpp
#include <functional>

void free_func(int, double) {}

struct Widget {
    int value;
    void print() const {}
};

void demo_invoke() {
    std::invoke(free_func, 1, 2.0);

    Widget w{42};
    std::invoke(&Widget::print, w);
    int x = std::invoke(&Widget::value, w);
    (void)x;
}
```

### Why prefer `invoke` in generic code?

```cpp
template <class Callable, class... Args>
decltype(auto) call(Callable&& c, Args&&... args) {
    return std::invoke(std::forward<Callable>(c), std::forward<Args>(args)...);
}
```

Here **`c`** might be anything **INVOKE**-compatible; you avoid spelling `(*c)(...)` vs `c(...)` vs `(obj.*c)(...)` manually (p. 399).

---

### ASCII diagram: `invoke` dispatch (conceptual)

```
                    std::invoke(f, args...)
                               |
            ┌──────────────────┼──────────────────┐
            v                  v                  v
      free function      function object    pointer-to-member
            |                  |                  |
            f(args...)         f(args...)    (object.*pmf)(...)
                                            or (object.*pmd)
```

*[Dispatch union as described around INVOKE, Josuttis p. 399.]*

---

## 33.2 `std::bool_constant<>` (p. 401)

### Definition

C++17 adds:

```cpp
template<bool B>
using bool_constant = std::integral_constant<bool, B>;
```

Thus:

- `std::true_type`  == `bool_constant<true>`
- `std::false_type` == `bool_constant<false>`

### Example in a trait

```cpp
#include <type_traits>

template <class T>
struct is_pointer_trait : std::bool_constant<std::is_pointer<T>::value> {};
```

**Benefit:** less typing than `integral_constant<bool, ...>` in **metaprogramming** (p. 401).

---

## 33.3 `std::void_t<>` (p. 403)

### Definition

```cpp
template <class...>
using void_t = void;
```

It maps any **valid** type parameter pack to **`void`**. The power is in **SFINAE**: **invalid** instantiations **discard** the overload rather than error.

### Detection of `.begin()` (classic idiom)

```cpp
#include <type_traits>
#include <utility>
#include <vector>

template <class, class = void>
struct has_begin : std::false_type {};

template <class T>
struct has_begin<T, std::void_t<decltype(std::declval<T>().begin())>>
    : std::true_type {};

static_assert(has_begin<std::vector<int>>::value);
```

**How it works:**

- Attempt **`decltype(std::declval<T>().begin())`** inside **`void_t<...>`**.
- If **`T` has no `.begin()`**, substitution fails inside `void_t`.
- The **partial specialization** is **removed** from the candidate set -> the **primary** `false_type` template wins. [OK]

- If **valid**, specialization **`has_begin<T, void>`** matches -> **`true_type`**.

---

### ASCII diagram: `void_t` SFINAE funnel

```
Template specialization selection for has_begin<T>
┌──────────────────────────────────────────────────────────────┐
│ Primary: has_begin<T, void = void>  -->  std::false_type     │
└──────────────────────────────────────────────────────────────┘
                    ^
      if substitution fails in partial specialization
                    |
                    v
┌──────────────────────────────────────────────────────────────┐
│ Partial: has_begin<T, void_t<decltype(declval<T>().begin())>>│
│          -> std::true_type                                   │
└──────────────────────────────────────────────────────────────┘
      only considered when void_t<...> is well-formed
```

---

### Advanced note: pitfalls

- **Arity / constness / rvalue** sensitivity: `declval<T>()` may pick **`const`** overloads; tune `declval<T&>()` / `declval<const T&>()` to match intended concept (not unique to `void_t`, but frequent).

- In C++20, **constraints** often supersede `void_t` tricks; in **C++17**, **`void_t`** remains idiomatic (p. 403).

---

## Reference

Josuttis: **Chapter 33**, **pp. 399-404**.
