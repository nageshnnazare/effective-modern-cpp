# Chapter 12: Dealing with String Literals as Template Parameters

**Book:** *C++17 - The Complete Guide*, Nicolai M. Josuttis  
**Chapter:** 12 - Dealing with String Literals as Template Parameters  
**Book page span:** **pp. 119-120** (section **12.1**)

This short chapter is about **non-type template parameters (NTTP)** that involve **address constants** and the evolution of linkage rules for string literals used through pointers/references in template parameters.

---

## 12.1 Using Strings in Templates (p.119)

### What can be a non-type template parameter (conceptual list)

Across C++17 discussions, NTTP includes (among other categories) **integral** values, **pointer** values (subject to constant-ness rules), **reference** parameters in some forms, and **`nullptr_t`**.

**Strings themselves are not passed “by value” as template arguments** like integers: what you usually pass is a **pointer** to leading character of an array object with static storage duration, or similar address-constant expressions permitted by the language.

### Linkage and address constants: before vs C++17 (p.119)

**Before C++17:** using a pointer into a string literal or local static array as a template argument often required specific **linkage** arrangements so the address counted as a sufficiently stable **address constant** for the instantiation model.

**Since C++17:** objects with **no linkage** in certain `static` local contexts become usable where the book says “no linkage allowed (static local)” — check **p.119** for the standard rule Josuttis cites and the motivating examples.

### Examples that are valid across many modes (p.119)

**External linkage array (book notes [OK]** in all versions covered):**

```cpp
extern const char hello[] = "Hello World!";
template<const char* P>
struct Message {};  // body omitted in tutorials

void demoExtern() {
  Message<hello> m;
  (void)m;
}
```

**Internal linkage / namespace scope `const` array (book: [OK] since C++11 in the narrative):**

```cpp
const char hello11[] = "Hello World!";
template<const char* P>
struct Message {};

void demoInternal() {
  Message<hello11> m;
  (void)m;
}
```

**Static local since C++17 (book story):**

```cpp
template<const char* P>
struct Message {};

void demoStaticLocal() {
  static const char hello17[] = "Hello World!";
  Message<hello17> m;
  (void)m;
}
```

### You still cannot pass a raw literal token as `template` argument (p.119)

This remains wrong:

```cpp
// template<class T, T v> struct A; // illustration only

// Message<"hi"> x;  // [ERROR]: cannot pass string literal token directly as NTTP
```

The literal `"hi"` is not a template argument of form `const char*` at the language level without going through a named entity / approved constant expression path.

### `constexpr` function returning an address (p.119)

The book notes that since C++17 you can use patterns like a `constexpr` function yielding a pointer usable in template instantiation:

```cpp
constexpr const char* pNum() {
  return "42";
}

template<const char* P>
struct A {};

void demoConstexprFn() {
  A<pNum()> a;
  (void)a;
}
```

**Reading guide:** The exact validity still depends on the function being a **constexpr** function in the strict sense and the returned pointer pointing into objects with the permitted lifetime rules for template arguments.

---

## ASCII diagram: linkage / storage mental model

```
                    Template argument needs stable address constant
                                        │
            ┌───────────────────────────┼───────────────────────────┐
            │                           │                           │
            ▼                           ▼                           ▼
    extern const char s[]        const char s[]              static const char s[]
    at namespace scope           at namespace scope          inside function
    (classic)                    (C++11 story)               (C++17 story)
```

---

## Common mistakes checklist

1. Trying `Message<"text">` directly [ERROR] (**p.119**).
2. Assuming **any** `const char*` works: it must meet constant-expression + linkage constraints.
3. Confusing **class NTTP** (C++20 adds more categories) with **C++17** rules — stay within the book’s C++17 scope unless upgrading deliberately.

---

## Minimal exercises

1. Instantiate three `Message<&helloX>`-style templates for `extern`, namespace `const`, and `static` local forms; observe compiler errors when rules are violated.
2. Compare MSVC/GCC/Clang diagnostics for invalid NTTP pointers.

---

**Primary reference:** Josuttis, *C++17 - The Complete Guide*, Chapter 12, **pp. 119-120**.
