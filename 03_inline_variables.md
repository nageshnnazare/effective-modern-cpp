# Chapter 3: Inline Variables

**Book:** *C++17 - The Complete Guide*, first edition, Nicolai M. Josuttis (published 2019-12-20, 452 pages)  
**Chapter:** 3 - Inline variables  
**Book page span:** approximately **pp. 25-31** (sections **3.1**-**3.5**)

This chapter explains **`inline` global / static data**, why header-only definitions become **ODR-safe**, and how C++17 changes **`constexpr static` members** from "often only declarations" into **real definitions**.

---

## 3.1 Motivation for inline variables (book p.25)

### 3.1.1 The problem: non-`constexpr` static members

Consider:

```cpp
// Thing.hpp
struct Thing {
  static std::string message; // declaration only
};
```

You **still** need **exactly one definition** in exactly one translation unit (TU):

```cpp
// Thing.cpp
#include "Thing.hpp"
std::string Thing::message{"hello"};
```

If you attempt to **define** a non-constexpr `static` data member **inside** the class in pre-C++17, you hit language restrictions. If you **define** a variable with external linkage in a header included by multiple `.cpp` files **without** `inline`, you typically get **linker multiple definition errors**.

### 3.1.2 Header-only libraries and repeated definitions

Include guards prevent **multiple inclusion in one TU**, but they do **not** prevent **different TUs** each compiling a **definition** that came from the same header.

```
TU A.cpp includes header.hpp  ──► compiles definition for Logger::level
TU B.cpp includes header.hpp  ──► compiles definition for Logger::level
                                        │
                                        ▼
                                  link step: [ERROR] duplicate symbol
```

### 3.1.3 Classic workarounds (pre-C++17)

The book surveys patterns such as:

- **`static const` integral constants** initialized in-class (limited type set).
- **`constexpr` values** where applicable.
- **Inline function** returning `static` local:

```cpp
inline std::string& msg() {
  static std::string m{"OK"};
  return m;
}
```

- **`static` member function** returning reference to function-local static.
- **Variable templates** (C++14) for some use cases.
- Small **class template** tricks encapsulating storage.

Each workaround trades **syntax noise**, **ABI**, or **constexpr-ness** for **ODR correctness**.

---

## 3.2 Using inline variables (book p.27)

### 3.2.1 Inside classes: `inline static` data members

C++17 allows:

```cpp
#include <string>

struct Config {
  inline static std::string msg{"OK"};
};
```

Every TU that **odr-uses** the member agrees the **definition** is the one from the class definition, similar to **inline functions**.

### 3.2.2 Namespace-scope inline globals

```cpp
#include <string>

inline std::string buildId{"dev-1234"};
```

You can place this in a header shared by multiple TUs **without** a `.cpp` definition.

**Book semantics reminder:** Like inline functions, the definition must be **identical** in every TU that sees it (same tokens / token-wise equivalence rules).

### 3.2.3 Atomics

```cpp
#include <atomic>

inline std::atomic<bool> ready{false};
```

This models a **process-wide** flag with header-friendly definition, subject to your platform's atomic width and alignment requirements.

### 3.2.4 Completeness and initialization order

The book warns: the **type must be complete** where the inline variable is **defined** (initialized). Do not rely on intricate **static initialization order** across translation units unless you follow safe idioms (`constexpr`, function-local statics, etc.).

---

## 3.3 `constexpr` now implies `inline` for static members (book p.28)

### 3.3.1 What changed

In C++17, a **`static constexpr` data member** initialized inside the class is a **definition**, not merely a declaration that still needs an out-of-class duplicate for ODR reasons in many cases.

**Before C++17** (common pitfall):

```cpp
struct S {
  static constexpr int n = 5; // declaration-only in many contexts
};

void take_const_ref(const int&);

void odr_use_before_cpp17() ; // Might require out-of-class definition:
// constexpr int S::n;
```

**C++17:** ODR-use scenarios such as **binding to `const int&`** or taking **`&S::n`** are resolved **without** a redundant out-of-class definition in typical toolchains.

**ASCII:**

```
C++14 and earlier                    C++17 onward
────────────────                    ────────────

static constexpr int n = 5;         static constexpr int n = 5;
(often) needs TU definition         is a definition (implies inline)
for some ODR-uses                   for static data members
```

### 3.3.2 Deprecation of redundant redeclarations

The book notes that **out-of-class `constexpr` definitions** may become **redundant**; modern codebases often delete them. Always consult your toolchains when upgrading standards.

---

## 3.4 Inline variables and `thread_local` (book p.29)

### 3.4.1 Declaring per-thread storage

```cpp
#include <string>

struct Worker {
  inline static thread_local std::string name;
};
```

**Meaning:**

- **`thread_local`**: each thread gets **distinct storage**.
- **`static`**: storage is associated with the **program / class**, not with each instance (`Worker` has no per-object overhead for `name`).
- **`inline`**: safe **single definition** when the class definition appears in multiple TUs.

### 3.4.2 Full example: global, thread-local, and instance names

This program illustrates **three storage durations** side by side. **Typical output** is shown after the code; exact thread IDs and pointer values vary by platform.

```cpp
#include <atomic>
#include <iostream>
#include <string>
#include <thread>

std::atomic<int> gCounter{0};

struct MyData {
  inline static std::string gName{"global-inline-static"};
  inline static thread_local std::string tName;
  std::string lName;

  explicit MyData(std::string local) : lName{std::move(local)} {}

  void bump_and_print(const char* who) {
    ++gCounter;
    if (tName.empty()) {
      tName = std::string{"thread-"} + who + "-" + std::to_string(gCounter.load());
    }
    std::cout << who
              << " | gName=" << gName
              << " | tName=" << tName
              << " | lName=" << lName
              << '\n';
  }
};

int main() {
  MyData a{"obj-a"};
  MyData b{"obj-b"};

  a.bump_and_print("main");
  b.bump_and_print("main");

  std::thread t{[&] {
    MyData c{"obj-c"};
    c.bump_and_print("side");
    MyData d{"obj-d"};
    d.bump_and_print("side");
  }};

  t.join();

  a.bump_and_print("main-after");
}
```

**Typical output (exact counters can vary by thread scheduling; structure is stable):**

```
main | gName=global-inline-static | tName=thread-main-1 | lName=obj-a
main | gName=global-inline-static | tName=thread-main-1 | lName=obj-b
side | gName=global-inline-static | tName=thread-side-3 | lName=obj-c
side | gName=global-inline-static | tName=thread-side-3 | lName=obj-d
main-after | gName=global-inline-static | tName=thread-main-1 | lName=obj-a
```

**Reading the output:** In the **main thread**, `tName` is created on the **first** `bump_and_print` and then **reused** for other `MyData` objects in that thread. In the **side thread**, the same pattern repeats with a **different** `tName` string.

| Field | Storage | Sharing rule |
|------|---------|--------------|
| `gName` | `inline static` | **ALL** threads and objects see **one** string object for the program |
| `tName` | `inline static thread_local` | **One** string **per thread**, shared by **all** `MyData` code running on that thread |
| `lName` | non-static member | **Each `MyData` instance** has its own string |

### 3.4.3 Thread-local memory layout (conceptual)

```
Process address space
┌────────────────────────────────────────────┐
│  .data / .bss: inline static gName         │  ◄── single object
├────────────────────────────────────────────┤
│  Thread 1 TLS block                        │
│    inline static thread_local tName        │  ◄── instance #1
├────────────────────────────────────────────┤
│  Thread 2 TLS block                        │
│    inline static thread_local tName        │  ◄── instance #2
└────────────────────────────────────────────┘

Stack frames (conceptual)
┌──────────────────┐
│ main() frame     │  MyData a.lName, MyData b.lName  (distinct)
└──────────────────┘
┌──────────────────┐
│ thread entry     │  MyData c.lName, MyData d.lName  (distinct)
└──────────────────┘
```

---

## 3.5 Afternotes (book)

Inline variables were **motivated** by **David Krauss** and **proposed** by **Hal Finkel** and **Richard Smith**, as the book notes.

---

## ODR diagram: inline fixes duplicate definitions

```
Without inline in header              With inline in header
────────────────────────              ─────────────────────

 header.hpp                           header.hpp
 ┌──────────────┐                     ┌───────────────────┐
 │ int x = 1;   │                     │ inline int x = 1; │
 └──────────────┘                     └───────────────────┘
        │                                    │
   ┌────┴────┐                          ┌────┴────┐
   ▼         ▼                          ▼         ▼
 A.cpp     B.cpp                    A.cpp     B.cpp
 [ERROR]   duplicate x             [OK]      merged x
 at link                               at link
```

---

## Pitfalls checklist

| Topic | Failure mode | Safer approach |
|------|--------------|----------------|
| Header definitions | Multiple `int x = 3;` | `inline int x = 3;` |
| `static` locals | Hiding globals unintentionally | Prefer class `inline static` for named config |
| ST order | Cross-TU init order fiasco | `constexpr`, function local static, `call_once` |
| `thread_local` lifetime | Access after thread ends | Join threads before reading TLS from outside |

---

## See also

- `00_index.md` (full book roadmap)  
- `02_if_switch_init.md` (RAII patterns complementary to globals)  

Reference: Nicolai M. Josuttis, *C++17 - The Complete Guide* (first edition, 2019-12-20).
