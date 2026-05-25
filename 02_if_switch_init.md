# Chapter 2: `if` and `switch` with Initialization

**Book:** *C++17 - The Complete Guide*, first edition, Nicolai M. Josuttis (published 2019-12-20, 452 pages)  
**Chapter:** 2 - `if` and `switch` with initialization  
**Book page span:** approximately **pp. 21-24** (sections **2.1**-**2.3**)

This note expands the book's examples with **motivation**, **lifetime diagrams**, and **pitfalls** (especially unnamed temporaries with RAII types like `std::lock_guard`).

---

## 2.1 `if` with initialization (book p.21)

### 2.1.1 New syntax

C++17 allows an **initializer statement** before the condition:

```
if ( initializer ; condition ) { ... } else { ... }
      └────┬────┘   └────┬────┘
           │             └─ bool (or contextually convertible)
           └─ either an expression or a declaration
```

**ASCII control flow:**

```
    ┌──────────────┐
    │ Run init     │
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐    no     ┌──────────────┐
    │ condition?   │────────►  │ else branch  │
    └──────┬───────┘           └──────┬───────┘
           │ yes                      │
           ▼                          │
    ┌──────────────┐                  │
    │ then branch  │                  │
    └──────┬───────┘                  │
           │                          │
           └──────────┬───────────────┘
                      ▼
               ┌──────────────┐
               │ Init scope   │
               │ ends here    │
               └──────────────┘
```

### 2.1.2 Lifetime and visibility

Any **name introduced in the initializer** is in scope in:

- the **condition** expression, and
- **both** the **then** and **else** branches,

and **not** beyond the full `if` statement.

This mirrors the existing scoping rules of a `for` loop's init statement, but for `if`/`else`.

**Book-style logging example:**

```cpp
#include <fstream>
#include <string>
#include <vector>

std::ofstream getLogStrm();

void log_collection(const std::vector<std::string>& coll) {
  if (std::ofstream strm = getLogStrm(); coll.empty()) {
    strm << "<no data>\n";
  } else {
    for (const auto& elem : coll) {
      strm << elem << '\n';
    }
  }
  // strm is NOT in scope here
}
```

**Why this matters:** You can keep **resource setup** colocated with the decision, without polluting outer scope and without an extra nesting level.

### 2.1.3 Mutex locking idiom

```cpp
#include <mutex>
#include <deque>
#include <iostream>

std::mutex collMutex;
std::deque<int> coll;

void print_front_if_any() {
  if (std::lock_guard<std::mutex> lg{collMutex}; !coll.empty()) {
    std::cout << coll.front() << '\n';
  }
}
```

### 2.1.4 CRITICAL WARNING: unnamed temporaries with RAII

This is **wrong**:

```cpp
// [ERROR] logical bug (do not write this):
if (std::lock_guard<std::mutex>{collMutex}; !coll.empty()) {
  // ...
}
```

**Reason:** `std::lock_guard<std::mutex>{collMutex}` creates a **temporary** lock object. The temporary is **destroyed at the end of the full initializer expression**, **before** the condition and branches run. The mutex is **not** held during the branch body.

ASCII timeline:

```
time ──────────────────────────────────────────────────────────►

init:  tmp lock constructed ──► tmp lock destroyed   condition   branches
                               ↑
                               mutex unlocked HERE (too early)
```

Contrast with a **named** guard:

```
named lg constructed ──────────► condition ──► branches ──► lg destroyed
                                                          ↑
                                                          mutex unlocked HERE
```

**Rule of thumb:** If the RAII object must survive until the end of the `if` statement branches, **give it a name** in the initializer.

### 2.1.5 Multiple declarations in the initializer

You may declare **more than one variable** when they share a single declaration form that allows it (as in the book):

```cpp
int qqq1();
int qqq2();

void demo() {
  if (int x = qqq1(), y = qqq2(); x != y) {
    // use x and y
  }
}
```

The specifics mirror ordinary declaration rules: types must compose legally.

### 2.1.6 Combining with structured bindings (book pattern)

```cpp
#include <map>
#include <string>
#include <utility>

void insert_or_inspect(std::map<std::string, int>& coll) {
  if (auto [pos, ok] = coll.insert({"new", 42}); !ok) {
    const auto& [key, val] = *pos; // re-decompose the map element
    (void)key;
    (void)val;
    // Handle duplicate-key case with readable names
  }
}
```

**Why it is ergonomic:** `insert` returns `std::pair<iterator, bool>`. The binding gives **self-documenting names** immediately, and the condition checks **`ok`** without a separate `.second` line.

### 2.1.7 Before and after comparison (C++14 vs C++17)

**C++14** (extra scope / repeated names):

```cpp
{
  auto result = coll.insert({"new", 42});
  if (!result.second) {
    const auto& elem = *result.first;
    (void)elem;
  }
}
```

**C++17**:

```cpp
if (auto ins = coll.insert({"new", 42}); !ins.second) {
  const auto& elem = *ins.first;
  (void)elem;
}
// or with structured bindings:
// if (auto [pos, ok] = coll.insert(...); !ok) { ... }
```

**Gains:**

- Names live only where needed.
- You can initialize **streams**, **locks**, **status objects**, and **algorithm results** right next to the boolean test.

---

## 2.2 `switch` with initialization (book p.23)

### 2.2.1 Syntax

```
switch ( initializer ; condition ) { ... }
```

The **name introduced by the initializer** is visible in the **condition** and **throughout all cases** of the `switch`, including **scopes guarded by `case` labels**, per the usual `switch` scoping rules.

### 2.2.2 Filesystem example pattern (book style)

```cpp
#include <filesystem>
#include <iostream>

namespace fs = std::filesystem;

void describe(const char* name) {
  switch (fs::path p{name}; status(p).type()) {
    case fs::file_type::not_found:
      std::cout << p << " not found\n"; // p visible
      break;
    case fs::file_type::regular:
      std::cout << p << " is a file\n";
      break;
    default:
      std::cout << p << " is something else\n";
      break;
  }
}
```

**Key idea:** **`p`** is created once, then reused when printing diagnostics in multiple cases without re-parsing or re-storing the string.

### 2.2.3 Usual `switch` caveats still apply

- **`break` discipline**: use `[[fallthrough]]` only with intent (C++17 attribute; see book Part I attributes chapter).
- **Variable declarations in `case`**: still wrap in blocks `{}` if initialization is needed to avoid leaking names across cases.

---

## 2.3 Afternotes (book)

The init-statement forms for `if` and `switch` are associated in the book's discussion with proposal work by **Thomas Koppe**.

---

## Practical patterns cheat sheet

| Goal | Pattern |
|------|---------|
| Stream scoped to branches | `if (std::ofstream os = open(); cond) { ... } else { ... }` |
| Lock only around inspection + work | `if (std::lock_guard lk{m}; pred()) { ... }` |
| Parse once, branch on kind | `switch (auto tok = lexer.next(); tok.kind) { ... }` |
| Container insert + failure | `if (auto [it, ins] = m.emplace(k, v); !ins) { ... }` |

---

## See also

- `01_structured_bindings.md` (insert idiom details, `pair`/`tuple` semantics)  
- `00_index.md` (full book roadmap)

Reference: Nicolai M. Josuttis, *C++17 - The Complete Guide* (first edition, 2019-12-20).
