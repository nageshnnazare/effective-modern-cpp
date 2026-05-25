# Chapter 14: Extended Using Declarations

**Book:** *C++17 - The Complete Guide*, Nicolai M. Josuttis  
**Chapter:** 14 - Extended Using Declarations  
**Book page span:** **pp. 129-132** (sections **14.1**-**14.2**)

C++17 extends `using` so you can import **multiple** names from a base in one declaration (comma-separated), and — critically for libraries — supports **pack expansion** in `using Base<Ts>::member...` forms. This enables idioms like **inheritance-based overload sets** and concise **lambda overload** wrappers.

---

## 14.1 Using Variadic Using Declarations (p.129)

### Comma-separated `using`

Old style (verbose):

```cpp
struct Derived : Base {
  using Base::a;
  using Base::b;
  using Base::c;
};
```

C++17 style:

```cpp
struct Derived : Base {
  using Base::a, Base::b, Base::c;
};
```

**Benefit:** less repetition, especially in CRTP-heavy or policy-based designs.

---

## Lambda overload pattern (p.129)

### Problem

`std::visit` for `std::variant` wants a **callable** with `operator()` overloads for each alternative type. Lambdas each have a unique closure type, so you cannot list them as a single type without merging.

### Solution: inherit from all lambdas + variadic `using`

```cpp
#include <utility>

template<typename... Ts>
struct overload : Ts... {
  using Ts::operator()...;
};

template<typename... Ts>
overload(Ts...) -> overload<Ts...>;

#include <string>
#include <variant>

void demoOverload() {
  auto twice = overload{
    [](std::string& s) { s += s; },
    [](auto& v) { v *= 2; },
  };

  std::variant<int, double, std::string> v{7};
  std::visit(twice, v);  // uses the generic lambda for int

  v = 3.5;
  std::visit(twice, v);  // uses generic lambda for double

  v = std::string{"hi"};
  std::visit(twice, v);  // uses string-specific lambda
}
```

**Reading guide:**

- `overload` derives from each lambda type, so the closure `operator()` members live in base subobjects.
- `using Ts::operator()...` brings **each** `operator()` into the derived class’s overload set.
- The **deduction guide** `overload(Ts...) -> overload<Ts...>` (C++17 CTAD) saves writing template arguments.

### Application: `std::variant` visitors (p.129)

This pattern is the modern replacement for nested `struct Visitor : lam1, lam2` boilerplate.

---

## ASCII diagram: overload resolution with variadic `using`

```
Call:   visitor(arg)
            │
            ▼
┌───────────────────────────────────────────────┐
│ Name lookup finds overload::operator() names  │
│ imported from bases:                          │
│                                               │
│   overload                                    │
│     ├─ lambda_1::operator()(string&)          │
│     └─ lambda_2::operator()(auto& ) [generic] │
└───────────────────────┬───────────────────────┘
                        │
                        ▼
              Overload resolution picks best match
                  (exact > generic, etc.)
                        │
                        ▼
                   Selected operator()
```

**Key idea:** Without `using Ts::operator()...`, the `operator()` functions are **hidden** as base-class members in many call contexts; the `using` declarations **inject** them into one overload set in the derived type.

---

## 14.2 Variadic Using Declarations for Inheriting Constructors (p.130)

### Pattern

```cpp
template<typename T>
struct Base {
  explicit Base(T) {}
};

template<typename... Types>
class Multi : private Base<Types>... {
 public:
  using Base<Types>::Base...;  // inherit each base’s constructors
};
```

### Behavior story (p.130)

`MultiISB m1 = 42;` can invoke the **`Base<int>`** constructor path while defaulting / implicitly handling other bases per the book’s example naming (`MultiISB` stands in for a concrete typedef in the text).

**Caution:** inheriting many constructors can create **ambiguity** if constructors overlap in viable resolution; design base constructors carefully.

### Inheriting assignment operators (p.130)

The book notes you can also inherit assignment operators similarly where appropriate:

```cpp
using Base<Types>::operator=...;
```

(This is advanced; watch for slicing / deleted inherited assignment interactions.)

---

## Design guidance

1. Use comma `using` to shorten repetitive injections (**p.129**).
2. Use `using Ts::operator()...` for **`overload` lambdas** + `std::visit` (**p.129**).
3. Use `using Base<Ts>::Base...` for **variadic mixin** constructor forwarding (**p.130**).
4. When CTAD fails (C++17 edge cases), name `overload<Ts...>` explicitly.

---

## Exercise

1. Implement `overload` without CTAD (explicit `overload<decltype(l1), decltype(l2)>`).
2. Build a `Multi` template and trace which constructor is called for `Multi<int,double>{42}` vs brace forms.
3. Replace a visitor struct with three manual `operator()` overloads by a single `overload{...}`.

---

**Primary reference:** Josuttis, *C++17 - The Complete Guide*, Chapter 14, **pp. 129-132**.
