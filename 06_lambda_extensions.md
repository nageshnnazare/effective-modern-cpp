# Chapter 6: Lambda Extensions

**Book reference:** *C++17 - The Complete Guide* by Nicolai M. Josuttis (pages 47-53)

Lambdas in C++17 gain two headline features discussed in this chapter:

1. **`constexpr` lambdas** (implicit or explicit).
2. **`*this` capture** (capture the current object *by value*), fixing lifetime bugs from pointer captures.

There is also brief coverage of capturing by **const reference** with `std::as_const` (p. 53).

---

## 6.1 `constexpr` Lambdas (p. 47)

### Implicit `constexpr` when possible

Since C++17, a lambda's call operator is **`constexpr` automatically** if the lambda satisfies the requirements for a `constexpr` function (roughly: what it does must be usable in constant evaluation).

**Example (book intent):**

```cpp
auto squared = [](auto val) { return val * val; };

// Use in compile-time context:
#include <array>
std::array<int, squared(5)> a{};   // [OK] size 25
```

Here `squared(5)` is a **constant expression** because the lambda's `operator()` is `constexpr`.

### What disables `constexpr`

If the lambda body uses features **not allowed** in `constexpr` functions, it is **not** `constexpr`. Common disablers:

* **`static` local variables** inside the lambda body (state that cannot live in `constexpr` evaluation the same way).
* Non-`constexpr` functions called from the body.
* Other C++ `constexpr` restrictions of your language mode.

```cpp
auto bad = [] {
    static int x = 0;   // typically makes operator() non-constexpr
    return x;
};
// std::array<int, bad()> b;  // [ERROR]
```

### Explicit `constexpr`

You can write `constexpr` explicitly for clarity or to get a diagnostic if it fails:

```cpp
auto f = [](auto val) constexpr { return val * val; };
```

With an explicit return type:

```cpp
auto g = [](int val) constexpr -> int { return val * val; };
```

### Closure type

The **closure type**'s `operator()` is `constexpr` when the rules are met -- there is no separate "runtime lambda" type vs "constexpr lambda" type; it is one type with a `constexpr` `operator()`.

### `constexpr` lambda vs `constexpr` lambda *object*

```cpp
constexpr auto squared = [](auto val) constexpr { return val * val; };
//  ^^^^^^^^^
//  The variable 'squared' is constexpr -- the closure itself must be
//  literal and usable at compile time.
```

Compare with:

```cpp
auto squared = [](auto val) { return val * val; };
// still may have constexpr operator() if possible, but 'squared' variable
// is not constexpr unless you mark it constexpr and the closure qualifies
```

---

## 6.1.1 Using `constexpr` Lambdas (p. 49)

### Compile-time string hash (djb2 style)

The book shows using a `constexpr` callable to hash string literals at compile time.

```cpp
constexpr unsigned long long hashed(const char* s, int h = 5381) {
    return *s ? hashed(s + 1, ((h << 5) + h) + *s) : h;
}
```

Or with a **constexpr lambda** wrapping the same idea (illustrative):

```cpp
constexpr auto hashed_fn = [](const char* s) constexpr -> unsigned long long {
    unsigned long long h = 5381;
    while (*s) {
        h = ((h << 5) + h) + static_cast<unsigned char>(*s);
        ++s;
    }
    return h;
};
```

**Enum with compile-time values** (using the `constexpr` function `hashed` above):

```cpp
enum class Hashed : unsigned long long {
    beer = hashed("beer"),
    wine = hashed("wine"),
};
```

You can write the same algorithm as a **`constexpr` lambda** stored in a `constexpr` variable and call that from `switch`, but **enumerators** need a single constant expression -- the free function form is often simplest.

**`switch` on runtime string-ish input** still compares to **compile-time** constants:

```cpp
// int main(int argc, char* argv[]) {
//     switch (hashed(argv[1])) {
//         case hashed("beer"): ...
//         case hashed("wine"): ...
//     }
// }
```

(Real code should handle invalid strings; this mirrors the book's teaching example.)

### `std::array` initialization

```cpp
#include <array>

constexpr auto make_table = [] constexpr {
    std::array<int, 10> a{};
    for (int i = 0; i < 10; ++i)
        a[i] = i * i;
    return a;
};
constexpr auto table = make_table();
```

### Parameterized lambda with another constexpr lambda

Nested `constexpr` callables compose when everything stays in the `constexpr` subset.

---

## 6.2 Passing Copies of `this` to Lambdas (p. 50)

### The problem: `[this]` captures a **pointer**

```cpp
struct Widget {
    std::string name;

    void demo() {
        auto lam = [this] { return name; };
        // lam stores pointer to *this
    }
};
```

If the lambda **outlives** the `Widget`, dereferencing the captured pointer is **[ERROR]** / undefined behavior.

`[=]` may capture `this` implicitly in ways that surprise readers (book warns).

### C++14 workaround: capture a copy of the object

```cpp
auto lam = [thisCopy = *this] { return thisCopy.name; };
```

### C++17: `*this` capture

```cpp
struct Data {
    std::string name;

    auto make_lambda() {
        return [*this] { return name; };  // copy of *this in closure
    }
};
```

The closure holds a **copy** of the object; the lambda can run later **safely** with respect to the original object's lifetime (subject to the usual rules for copying members).

### Combining captures

* `[&, *this]` [OK] -- default by-reference members, but object as value copy.
* `[this, *this]` [ERROR] -- conflicting/duplicate specification of object capture (book).

### Thread example

```cpp
#include <thread>

class Data {
    // ...
public:
    void startThreadWithCopyOfThis() {
        std::thread t{[*this]() mutable {
            // use members of the *copy*
        }};
        t.detach();
    }
};
```

**Without `*this`:** a lambda stored in a `std::thread` that captured `[this]` could run after the object is destroyed [ERROR] [undefined behavior].

---

## 6.3 Capturing by `const` Reference (p. 53)

Sometimes you want capture by reference but **read-only**, even when the source object is non-`const`. **`std::as_const`** helps:

```cpp
#include <utility>

void example(std::string& s) {
    auto lam = [&s = std::as_const(s)] {
        // s inside lambda is const std::string&
    };
}
```

This avoids accidentally mutating the outer object through a reference capture.

---

## 6.4 Afternotes

* **`constexpr` lambdas** were advanced by **Faisal Vali** et al.
* **`*this` capture** was advanced by **H. Carter Edwards** et al.

Both features close gaps that library authors and application developers hit regularly in C++14.

---

## Quick reference

| Feature | Syntax / idea | Result |
|--------|----------------|--------|
| Implicit constexpr | plain lambda body | `operator()` constexpr if valid |
| Explicit constexpr | `[]() constexpr { ... }` | diagnostic if invalid |
| Copy *this | `[*this]` | closure stores subobject copy |
| Pointer this | `[this]` | closure stores pointer -- lifetime bugs |
| const ref capture | `[&x = std::as_const(y)]` | read-only alias |

---

## Self-check questions

1. Why does `static` inside a lambda often break compile-time use?
2. When does `[*this]` allocate more than `[this]`?
3. Why is `[this, *this]` ill-formed?

(Answers follow directly from the sections above.)
