# Chapter 26: Container and String Extensions

*Reference: Nicolai M. Josuttis, **C++17 - The Complete Guide**, Chapter 26, pp. 309-319.*

C++17 extends associative containers with **node handles** (`extract` / splice-style insertion), refines **emplace** return types, adds `try_emplace` / `insert_or_assign`, permits certain containers with **incomplete** value types, and makes `std::string::data()` provide **mutable** access.

---

## 26.1 Node Handles (p. 309)

Applicable to **node-based** associative containers: `std::map`, `std::multimap`, `std::set`, `std::multiset`, and their `unordered_*` counterparts.

A **node handle** (`node_type`) owns a single element **outside** any container, enabling moves without allocating new nodes.

---

## 26.1.1 Modifying a Key (p. 309)

`std::map` keys are normally immutable in-place. **Extract -> mutate key -> reinsert** is the idiom.

```cpp
#include <map>
#include <string>
#include <iostream>

int main() {
    std::map<std::string, int> m{{"old", 42}};

    auto nh = m.extract("old");
    if (!nh.empty()) {
        nh.key() = "new";  // modify key while out of container
        auto result = m.insert(std::move(nh));  // returns insert_return_type: position, inserted, node
        (void)result.position;
    }
}
```

**Practical note:** Exact `insert` return type differs between map/set variants; always check your standard library docs for your container.

**Why it works:** While extracted, the element is not located by the container's tree / bucket structure, so key hash / ordering invariants can be rebuilt on insertion.

---

## 26.1.2 Moving Nodes Between Containers (p. 311)

Transfer **without** copying mapped values or reallocating a new node (when successful).

```cpp
#include <map>
#include <string>

void merge_one_entry(
    std::map<std::string, int>& dst,
    std::map<std::string, int>& src,
    std::map<std::string, int>::iterator it_in_src) {

    auto nh = src.extract(it_in_src);
    dst.insert(std::move(nh));
}
```

**Compatibility:** Source and destination must have **compatible** `node_type` (same `key_type`, `mapped_type`, allocator, hash/compare for unordered maps).

---

## 26.1.3 Merging Containers (p. 312)

`dst.merge(src)` moves **all** transferable nodes from `src` into `dst`.

- **Unique-key containers (`map`, `unordered_map`, `set`, ...):** If an element's key already exists in `dst`, that element **stays** in `src`.
- **`multimap` / `multiset`:** All can move (duplicates allowed).

```cpp
#include <map>
#include <iostream>

int main() {
    std::map<std::string, int> a{{"x", 1}, {"y", 2}};
    std::map<std::string, int> b{{"y", 99}, {"z", 3}};

    a.merge(b);
    // a has x,y,z ; y kept original in 'a' (already present), 99 remains in b perhaps
    // b retains nodes whose keys collided in dst
    std::cout << b.size() << '\n';
}
```

---

### ASCII: extract / reinsert lifecycle

```
┌──────────── map container ────────────┐
│  ...  [ key=k  | value=v ]  ...       │
└────────────────┬──────────────────────┘
                 │ extract(it)
                 ▼
        ┌─────────────────┐
        │  node handle nh │
        │  owns element   │
        └────────┬────────┘
                 │ nh.key() = new_key   (map)
                 │ or move to other map insert(nh map std::move(nh))
                 ▼
┌──────────── target container ─────────┐
│  ... newly inserted / merged node ... │
└───────────────────────────────────────┘
```

---

## 26.2 Emplace Improvements (p. 314)

### 26.2.1 Return Type of Emplace Functions (p. 314)

For containers like `vector`, `deque`, `list`, `forward_list`:

- `emplace`, `emplace_back`, `emplace_front` return **reference** to inserted element (instead of `void` where applicable previously). **[OK]** In particular, `emplace_back` returns `T&` since C++17 (earlier `std::vector::emplace_back` was `void`).

```cpp
#include <vector>
#include <string>

int main() {
    std::vector<std::string> v;
    std::string& ref = v.emplace_back("hello");
    ref[0] = 'H';
}
```

---

### 26.2.2 `try_emplace()` and `insert_or_assign()` (p. 314)

Both return **`std::pair<iterator, bool>`** (for `map`-like unique-key containers).

**`try_emplace`:** Inserts only if key missing; **does not move or construct** mapped value arguments if key exists (useful for expensive values).

```cpp
#include <map>
#include <string>
#include <utility>

struct Expensive {
    explicit Expensive(std::string) {}
};

int main() {
    std::map<int, Expensive> m;
    m.try_emplace(1, std::string("a"));     // constructs
    m.try_emplace(1, std::string("ignored")); // no construction of Expensive

    std::map<std::string, std::string> cfg;
    std::string value = "payload";
    cfg.try_emplace("key", std::move(value));  // [OK] value not moved from if "key" already exists
}
```

**`insert_or_assign`:** Updates mapped value if key exists (uses move/assignment), inserts otherwise.

```cpp
#include <map>
#include <string>

int main() {
    std::map<std::string, int> cfg;
    std::pair<std::map<std::string, int>::iterator, bool> ir =
        cfg.insert_or_assign("timeout", 30);   // returns std::pair<iterator, bool>
    (void)ir;
    cfg.insert_or_assign("timeout", 60);  // overwrites; bool indicates insert vs assign semantics per standard
}
```

**Difference from `operator[]`:** `insert_or_assign` works with **non-default-constructible** mapped types and can avoid accidental default insert side effects.

---

## 26.3 Container Support for Incomplete Types (p. 316)

`std::vector`, `std::list`, and `std::forward_list` may be instantiated on **incomplete** `T` until the first use that requires completeness.

Enables recursive tree structures:

```cpp
#include <vector>
#include <string>

struct Node {
    std::string name;
    std::vector<Node> children;  // C++17: OK with incomplete Node at field declaration
};
```

**Caveats:**

- Completeness required before operations that allocate or destroy elements.
- Not all containers gained this---e.g. `std::deque` historically differs; follow the standard for your type.

---

## 26.4 String Improvements (p. 318)

`std::basic_string::data()` now returns **`CharT*`** (non-const) in addition to `const` overload---mutable access to underlying sequence (contiguous).

```cpp
#include <string>
#include <cstring>

void fill(std::string& s) {
    char* p = s.data();  // non-const CharT* since C++17
    std::memset(p, 'A', s.size());
}

void mutate_first(std::string& s) {
    if (!s.empty()) {
        char* p = s.data();
        p[0] = 'X';  // [OK] mutable access via data()
    }
}
```

**Interaction with `std::as_bytes` / C++20:** in C++17, this closes a long-standing gap vs `vector::data`.

---

## Quick decision guide

| Task | Mechanism |
|------|-----------|
| Change map key | `extract` + mutate `nh.key()` + `insert` |
| Move map entry cheaply | `extract` + `insert` into other map |
| Bulk steal compatible nodes | `dst.merge(src)` |
| Insert if missing, skip cost otherwise | `try_emplace` |
| Upsert config value | `insert_or_assign` |
| Recursive child lists | `vector<Node>` with incomplete `Node` until used |

---

## Cross-reference

- Node handles also matter for exception safety when reinserting (ensure you handle failed insert paths).
- **Chapter 25:** `std::data` free function vs `string::data` (pp. 299-307).
