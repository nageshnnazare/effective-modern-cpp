# Chapter 20: The Filesystem Library (The Complete Guide, pp. 201-248)

This chapter follows *C++17 - The Complete Guide* by Nicolai M. Josuttis. Page references use **p.**

C++17 integrates the **Filesystem library** as defined in ISO C++17. The facilities live in `<filesystem>` under namespace `std::filesystem`.

**Markers used in this note:**

- **[OK]** Correct usage or expected outcome.
- **[ERROR]** Typical failure mode, wrong assumption, or exception path.
- **[OOPS]** Subtle trap or portability footgun.

---

## 20.1 `checkpath`: full example (p. 201–204)

A complete motivating program: for each path given on the command line, print the path, then whether it exists; if it exists, whether it is a regular file (with size) or a directory; if it is a directory, list immediate children with `directory_iterator`. Josuttis also shows switching on **filesystem** file type using `status(p).type()` (after confirming existence).

### Complete program (checkpath style)

```cpp
#include <filesystem>
#include <fstream>
#include <iostream>

namespace fs = std::filesystem;

// Print one path's nature using status().type() (p. 203-205)
void checkWithSwitch(const fs::path& p) {
    if (!fs::exists(p)) {
        std::cout << p << " does not exist\n";
        return;
    }

    const fs::file_status st = fs::status(p);
    std::cout << p << " exists as ";
    switch (st.type()) {
        case fs::file_type::none:
            std::cout << "none\n";
            break;
        case fs::file_type::not_found:
            std::cout << "not_found\n";
            break;
        case fs::file_type::regular:
            std::cout << "a regular file (" << fs::file_size(p) << " bytes)\n";
            break;
        case fs::file_type::directory:
            std::cout << "a directory containing:\n";
            try {
                for (const fs::directory_entry& e : fs::directory_iterator{p}) {
                    std::cout << "  " << e.path().filename() << '\n';
                }
            } catch (const fs::filesystem_error& e) {
                std::cerr << "[ERROR] iteration: " << e.what() << '\n';
            }
            break;
        case fs::file_type::symlink:
            std::cout << "a symlink\n";
            break;
        case fs::file_type::block:
            std::cout << "a block device\n";
            break;
        case fs::file_type::character:
            std::cout << "a character device\n";
            break;
        case fs::file_type::fifo:
            std::cout << "a fifo\n";
            break;
        case fs::file_type::socket:
            std::cout << "a socket\n";
            break;
        case fs::file_type::unknown:
        default:
            std::cout << "something unknown\n";
            break;
    }
}

int main(int argc, char* argv[]) {
    if (argc < 2) {
        // Default: current directory for demonstration
        checkWithSwitch(fs::current_path());
        return 0;
    }
    for (int i = 1; i < argc; ++i)
        checkWithSwitch(fs::path{argv[i]});
    return 0;
}
```

### Windows path handling: stream vs `.string()` (p. 202–204)

**[OK]** Inserting a `fs::path` into a stream (`std::cout << p`) uses **quoted** representation so spaces and special characters are visible and unambiguous.

**[OK]** Printing the native string (`std::cout << p.string()` or `p.u8string()` / `wstring` as appropriate) shows the path **without** extra quotes—what you would pass to many APIs.

**[OOPS]** On Windows, quoted stream output may show **doubled backslashes** in the presentation; that is presentation only, not a change to the path value.

Example distinction:

```cpp
fs::path p = "C:\\Program Files\\app\\test.txt";
std::cout << p << '\n';           // often: "C:\\Program Files\\app\\test.txt"
std::cout << p.string() << '\n';  // native:  C:\Program Files\app\test.txt
```

`status(p)` vs `symlink_status(p)`: **`status`** follows symlinks to classify the **target**; **`symlink_status`** classifies the **link node itself**. For a `switch` on “what is this name,” pick the one that matches your intent (p. 205).

---

## 20.1 `createfiles`: full example (p. 206–210)

Shows `create_directories`, creating files with `std::ofstream`, **`operator/`** on `path`, **`create_directory_symlink`** (with the relative-target rule), **`recursive_directory_iterator`** with **`directory_options::follow_directory_symlink`**, **`lexically_normal()`** for readable output, and **`filesystem_error`** (`what()`, `path1()`, `path2()`).

### Relative target rule for `create_directory_symlink` (p. 208)

**[OK]** The **first argument** (target) is resolved **relative to the directory containing the new symlink**, not relative to the process’s current directory (unless they coincide).

**[OOPS]** If you pass an absolute target when you meant a sibling-relative name, behavior differs from shell `ln -s` mental model; verify with a small experiment on your OS.

### Complete program (createfiles style)

```cpp
#include <filesystem>
#include <fstream>
#include <iostream>

namespace fs = std::filesystem;

void printRecursive(const fs::path& root) {
    namespace dopts = fs::directory_options;
    try {
        fs::recursive_directory_iterator it{
            root,
            dopts::follow_directory_symlink | dopts::skip_permission_denied
        };
        for (const fs::directory_entry& e : it) {
            // lexically_normal cleans ., .., redundant separators in the printed form (p. 209-210)
            fs::path shown = e.path().lexically_normal();
            std::cout << shown << '\n';
        }
    } catch (const fs::filesystem_error& ex) {
        std::cerr << "[ERROR] " << ex.what() << '\n';
        if (!ex.path1().empty())
            std::cerr << "  path1: " << ex.path1() << '\n';
        if (!ex.path2().empty())
            std::cerr << "  path2: " << ex.path2() << '\n';
        std::cerr << "  code:  " << ex.code().message() << '\n';
    }
}

int main() {
    const fs::path base = fs::path{"tmpdir"} / "createfiles_demo";
    try {
        fs::create_directories(base / "sub" / "other");
        // Regular file via ofstream; path built with /  (p. 207)
        {
            std::ofstream{base / "sub" / "other" / "hello.txt"} << "hello\n";
        }
        // Directory symlink: target string is relative to link’s parent directory (p. 208)
        fs::create_directory_symlink("other", base / "sub" / "new_other");
    } catch (const fs::filesystem_error& ex) {
        std::cerr << "[ERROR] setup: " << ex.what() << "\n";
        return 1;
    }

    printRecursive(base);
    return 0;
}
```

**[OK]** `create_directories` creates all missing intermediate directories; returns `true` if any directory was created.

**[ERROR]** If the leaf exists as a file, directory creation fails (`filesystem_error` or non-throwing overload sets `ec`).

---

## 20.2 Parallel algorithms remark (p. 209)

Josuttis notes combining directory iteration with parallel algorithms while remembering filesystems are **concurrent**: entries can appear or disappear between calls (TOCTOU). Any parallel walk must tolerate races or serialize sensitive operations.

---

## 20.2 Portability: paths, roots, case (p. 210–212)

### Case sensitivity

**[OOPS]** POSIX filesystems are often **case-sensitive** (`File` ≠ `file`). Windows paths are usually **case-insensitive** but **case-preserving**. Lexical comparisons in C++ do **not** automatically fold case.

### Absolute vs relative: `/bin`, `C:/bin` (p. 211)

| Path   | POSIX typical interpretation | Windows typical interpretation |
|--------|------------------------------|--------------------------------|
| `/bin` | **Absolute** (root `/`)      | **Relative** (current drive)   |
| `C:/bin` / `C:\\bin` | **Relative** (no root—you have a root-name `C:` without absolute root directory in the generic grammar sense) see book nuance | **Absolute** on `C:` |

**[OOPS]** A path can look “familiar” from another platform but parse differently. Prefer **`fs::path`** and filesystem queries (`absolute`, `canonical`, `weakly_canonical`) over hand-parsing strings.

### Generic path format (p. 212)

Josuttis describes the logical layout:

```
┌─────────────────────────────────────────────────────────────┐
│  [ root-name ] [ root-directory ] [ relative-path ]         │
│       optional      optional           optional              │
│                                                             │
│  Example (Windows):  C:  \   usr\bin\gcc                    │
│  Example (POSIX):    (empty) / usr/bin/gcc                   │
└─────────────────────────────────────────────────────────────┘
```

- **root-name**: e.g. volume or UNC host share prefix on Windows.
- **root-directory**: directory separator that anchors an **absolute** path (when present with root-name rules as per standard).
- **relative-path**: sequence of *filename* elements.

---

## Normalization table: `lexically_normal()` (p. 212–213)

`lexically_normal()` performs **purely lexical** cleanup (no filesystem calls): collapses `.`, resolves `..` where defined, folds redundant separators. **Exact outcomes differ** between POSIX-style and Windows-style paths as in the book.

| Input path (example) | After `lexically_normal()` POSIX-style | After `lexically_normal()` Windows-style |
|----------------------|----------------------------------------|------------------------------------------|
| `foo/.///bar/../`    | `foo/`                                 | `foo\`                                   |
| `//host/../foo.txt`  | `//host/foo.txt`                       | `\\host\foo.txt`                         |
| `C:bar/../`          | `.`                                    | `C:`                                     |
| `C:/bar/..`          | `C:/`                                  | `C:\`                                    |
| `C:\bar\..`          | `C:\bar\..` (lexical rules: mixed grammar) | `C:\`                                |

**[OOPS]** Rows such as `C:\bar\..` illustrate that **mixing** separators and roots can leave paths that normalize oddly on a “wrong” platform abstraction—always construct paths with **`fs::path`**, not ad-hoc strings.

**[OK]** For **existing** paths, **`canonical()`** / **`weakly_canonical()`** consult the OS and symlinks; they can differ from `lexically_normal()`.

---

## Member vs free-standing operations (p. 213)

```
┌──────────────────────────────┬────────────────────────────────────┐
│         Category             │              Cost / semantics         │
├──────────────────────────────┼───────────────────────────────────────┤
│ path members (p.filename(),  │  Cheap: lexical only, no OS calls     │
│   .parent_path(), +=, /=)   │                                       │
├──────────────────────────────┼───────────────────────────────────────┤
│ Free-standing (exists,       │  Potentially expensive: may call OS │
│   file_size, create_*)       │                                       │
└──────────────────────────────┴───────────────────────────────────────┘
```

**[OK]** Argument-dependent lookup applies: **`remove(p)`** may find **`std::filesystem::remove`** when `p` is a `path`.

**[OOPS]** If you write **`remove("tmpdir")`** with a string literal, **ADL does not** bring in `std::filesystem::remove` the same way; overload resolution may pick the C library **`::remove`** from `<cstdio>` (remove a file by `char*` name)—a classic **name clash**.

**[OK]** Prefer **`fs::remove`** / **`std::filesystem::remove`** explicitly, or pass a **`fs::path`** so the filesystem overload is unambiguous.

---

## Error handling (p. 214–216)

### `filesystem_error` members

| Member / idiom | Role |
|----------------|------|
| `what()` | Human-readable message (often includes paths). |
| `path1()`, `path2()` | Paths involved (second used for two-path ops, e.g. `rename`). |
| `code()` | `std::error_code` for programmatic handling. |

### Non-throwing overloads

```cpp
std::error_code ec;
bool ok = fs::create_directory(p, ec);
if (ec) {
    // [ERROR] failed — inspect ec and optionally ec.category()
}
```

### Comparing to specific conditions (p. 215–216)

```cpp
if (ec == std::errc::read_only_file_system) {
    // [ERROR] known cause: read-only volume
}
```

**[OK]** Use `error_code` overloads in libraries, servers, or embedded contexts where exceptions are discouraged.

**[OK]** Use throwing overloads in application code where “fail fast” is acceptable; **`catch (const fs::filesystem_error& e)`** to log **`e.what()`** and paths.

---

## `file_type` enumeration (p. 216)

Values you may see from **`status` / `symlink_status`** (`.type()`):

| Enumerator | Meaning (typical) |
|------------|-------------------|
| `file_type::none` | Type not available / not evaluated. |
| `file_type::not_found` | Path does not exist or not discoverable. |
| `file_type::regular` | Regular file. |
| `file_type::directory` | Directory. |
| `file_type::symlink` | Symbolic link. |
| `file_type::block` | Block special file (POSIX). |
| `file_type::character` | Character special file (POSIX). |
| `file_type::fifo` | Pipe / FIFO. |
| `file_type::socket` | Socket file (POSIX). |
| `file_type::unknown` | Exists but type unknown to implementation. |

---

## 20.3 Path creation (p. 217)

| Expression / function | Purpose |
|-----------------------|---------|
| `fs::path{charseq}` | Construct from `string`, `string_view`, C string, `iterator` range, etc. |
| `fs::path{beg, end}` | Range constructor (character sequence). |
| `fs::u8path(u8string)` | Build path from UTF-8 (`char` sequence labeled UTF-8). |
| `fs::current_path()` | Current working directory as `path`. |
| `fs::temp_directory_path()` | Temporary directory suitable for scratch files. |

**[OOPS]** Encoding on Windows often wants wide or UTF-8-aware APIs; **`u8path`** bridges UTF-8 source to `path` where supported.

---

## 20.3 Path inspection (p. 218–220)

### Predicates: `has_...`

| Function | [OK] `true` when… |
|----------|-------------------|
| `p.empty()` | No path components. |
| `p.has_root_path()` | `root_path()` is non-empty. |
| `p.has_root_name()` | Volume / device name part present (e.g. `C:` on Windows). |
| `p.has_root_directory()` | Root directory present (e.g. `\` after volume). |
| `p.has_relative_path()` | Relative portion exists beyond root. |
| `p.has_parent_path()` | A parent path exists (`parent_path().native().size() < native().size()` in standard terms). |
| `p.has_filename()` | Filename component exists (includes `.` or `..` as filenames). |
| `p.has_stem()` | Stem (filename without last extension) non-empty. |
| `p.has_extension()` | Extension non-empty (note: leading dot rules for “pure extension” filenames). |

### Getters

| Function | Typical role |
|----------|--------------|
| `root_name()` | e.g. `C:` or UNC prefix |
| `root_directory()` | e.g. `/` or `\` |
| `root_path()` | `root_name` + `root_directory` |
| `relative_path()` | Part after `root_path` |
| `parent_path()` | Prefix before final `filename` |
| `filename()` | Last component (may be `..`) |
| `stem()` | `filename` without final *extension* segment |
| `extension()` | Final extension including dot, or empty |

### Platform illustration: `C:/hello.txt` (p. 219)

**[OOPS]** On **POSIX**, `C:` is just a filename-like component; `/` starts an absolute path from root—so **`root_path()`** might be **`/`** and **`relative_path()`** begins with **`C:`**… Josuttis contrasts this with **Windows**, where `C:` + `\` forms a familiar root.

Always test mental models with **`fs::path{"C:/hello.txt"}`** on your target OS or by reading decomposition in a debugger.

---

## Path iteration: component sequence (p. 219–220)

Range-for over **`path`** yields each **filename** piece (including root directory as appropriate per platform).

```cpp
#include <filesystem>
#include <iostream>

namespace fs = std::filesystem;

void printPath(const fs::path& p) {
    for (const fs::path& part : p) {
        std::cout << '"' << part.string() << "\" ";
    }
    std::cout << '\n';
}
```

**[OK]** `printPath("../sub/file.txt")` → components such as **`".." "sub" "file.txt"`**.

**[OK]** `printPath("/usr/tmp/test/dir/")` on POSIX-like paths → **`"/" "usr" "tmp" "test" "dir" ""`** (final empty component reflects trailing separator decomposition in the standard model—verify on your library).

**[OK]** Windows: `"C:\\usr\\..."` may iterate as **`"C:" "\\" "usr" ...`** (root name, root directory, then names).

### Trailing separator check (p. 220)

**[OK]** Detect “ends with directory separator” style paths using iterator to last element:

```cpp
bool hasTrailingDirSep(const fs::path& p) {
    if (p.empty()) return false;
    auto it = p.end();
    --it;
    return it->empty(); // last iterator element is empty for trailing separator case
}
```

**[OOPS]** Rely on this idiom rather than string hacks so behavior stays aligned with **`path`** grammar.

---

## Path I/O (p. 221–223)

| Operation | Effect |
|-----------|--------|
| `std::cout << p` | **Quoted** output by default; implementation presents escaped form. |
| `std::cout << p.string()` | **Unquoted** native string. |
| `generic_string()`, `u8string()`, etc. | Alternate encodings / generic `/` form. |

**[OK]** On Windows, quoted stream output often shows **doubled backslashes**.

### `lexically_relative` and `lexically_proximate` (p. 222–223)

Both are **lexical** (no filesystem).

- **`lexically_relative(base, p)`**: If `p` shares a root with `base`, returns relative path from `base` to `p`; otherwise empty path.
- **`lexically_proximate(base, p)`**: Like relative, but if roots differ, may return **`p`** (so you always get *some* navigational hint).

```cpp
fs::path base = "/a/b";
fs::path p    = "/a/b/c/d";
fs::path rel  = p.lexically_relative(base);   // often: c/d
```

**[OOPS]** After symlinks or mount points, lexical relative path may not match a valid relative **filesystem** walk—use **`relative`** (C++17 overloaded free function using `weakly_canonical`) when you need filesystem-aware semantics (where both paths exist).

---

## Path modification details (p. 226–228)

### `+=` vs `/=`

| Op | Meaning |
|----|--------|
| **`p /= "sub"`** | Append as **path element**: adds preferred separator between existing non-empty root-relative tail and new piece. |
| **`p += ".git"`** | **Literal character append** to native representation—**no** automatic separator. |

**[OOPS]** `p += "/sub"` stitches characters; you may get odd double slashes or ambiguous roots.

### Appending an **absolute** subpath replaces (p. 227)

**[OK]** **`fs::path("/usr") / "/tmp"`** → **`"/tmp"`** (absolute right-hand side wins).

**[OOPS]** On Windows, appending **`"` / "`"`** across **`root_name`** boundaries follows standard **root-truncation** rules (e.g. **`D:\`** wins over a prior `C:`-relative path)—consult table in Josuttis for **`path::operator/`** edge cases.

### `replace_extension` (p. 228)

**[OK]** `path{"file.txt"}.replace_extension(".tmp")` → **`file.tmp`**.

**[OOPS]** `path{".git"}.replace_extension(".tmp")` → **`.git.tmp`** — a lone dot filename is **not** treated as “no stem + `.git` extension” the way you might expect; **pure extension-looking names** bite stem/extension rules.

---

## Comparisons (p. 228–229)

**[OK]** `==`, `<`, etc. compare paths **lexically** in the path’s **element** sequence—not necessarily the same as resolving onto the same inode.

**[OOPS]** **`tmp1/f`** and **`./tmp1/f`** can compare **unequal** until you apply **`lexically_normal()`** (or equivalents).

**[OK]** **`equivalent(p1, p2)`** resolves via the filesystem: **[OK]** both paths must refer to **existing** entities that the implementation can confirm as the same file (e.g. hard links). **[ERROR]** if one side is missing, typically `false` or error per overload.

---

## `last_write_time` (p. 232–234)

**[OK]** Return type is **`file_time_type`**, a **`chrono::time_point`** using the **filesystem’s clock**, **not** `std::chrono::system_clock`.

**[OOPS]** You cannot `time_t`-cast `file_time_type` directly portably across all implementations without bridging clocks.

### Josuttis-style bridge to `time_t` (workaround sketch)

```cpp
#include <chrono>
#include <ctime>
#include <filesystem>

template <typename TimePoint>
std::time_t toTimeT(TimePoint tp) {
    using namespace std::chrono;
    // Offset between file clock and system clock via "now" on both (p. 233-234)
    auto sctp = system_clock::now() + (tp - TimePoint::clock::now());
    return system_clock::to_time_t(sctp);
}
```

**[OOPS]** Small races between the two `now()` calls can cause a one-second glitch near boundaries; for display-only use this is usually acceptable.

**[OK]** Format with **`std::ctime`** (note: not thread-safe on some platforms) or **`strftime`** after converting to `tm` in UTC/local as appropriate.

---

## Permissions (p. 235–237)

### `perms` bitmask (typical octal; p. 236)

POSIX permission bits map to `fs::perms` (exact values from standard; octal as commonly documented):

| `fs::perms` | Value (octal) | Meaning |
|-------------|---------------|---------|
| `owner_read` | `0400` | Owner read |
| `owner_write` | `0200` | Owner write |
| `owner_exec` | `0100` | Owner execute |
| `group_read` | `0040` | Group read |
| `group_write` | `0020` | Group write |
| `group_exec` | `0010` | Group execute |
| `others_read` | `0004` | Others read |
| `others_write` | `0002` | Others write |
| `others_exec` | `0001` | Others execute |
| `set_gid` | `02000` | Set-group-ID |
| `set_uid` | `04000` | Set-user-ID |
| `sticky_bit` | `01000` | Sticky bit |
| `mask` | `07777` | Mask of permission + special bits |
| `unknown` | (implementation-defined) | Unknown bits |

Also **`none`** (no bits set) and **`all`** / **`group_all`** / **`owner_all`** / **`others_all`** convenience composites appear in the library.

### Permission string (conceptual `rwxr-xr-x`)

**[OK]** POSIX: map owner/group/other triples to `r`, `w`, `x`, or `-` for each bit; optional leading character for file type is **not** part of `perms` itself.

### Windows ACL mapping (p. 237)

**[OOPS]** Windows uses **ACLs**; the standard maps to a **reduced** POSIX-like view—Josuttis effectively describes **two coarse modes** (readable vs read/write style simplification) compared to full Unix `chmod` semantics.

### `perm_options` with `permissions()` (p. 236)

```cpp
fs::permissions(p, fs::perms::owner_write | fs::perms::group_read,
                fs::perm_options::add);
fs::permissions(p, fs::perms::others_all, fs::perm_options::remove);
```

| Option | Effect |
|--------|--------|
| `perm_options::replace` | Replace entire permission set (subject to OS). |
| `perm_options::add` | OR in bits. |
| `perm_options::remove` | Clear bits. |

---

## `copy_options` bitmask (p. 238–239)

Used primarily with **`copy`** and **`copy_file`** (overloads that accept `copy_options`; not every filesystem function takes this bitmask).

| Option | Meaning |
|--------|---------|
| `copy_options::none` | Default behavior. |
| `copy_options::skip_existing` | Don’t overwrite destination if it exists. |
| `copy_options::overwrite_existing` | Overwrite. |
| `copy_options::update_existing` | Overwrite only if newer (or per implementation rules). |
| `copy_options::recursive` | Copy directory trees. |
| `copy_options::copy_symlinks` | Copy symlinks as symlinks. |
| `copy_options::skip_symlinks` | Ignore symlinks (don’t follow as files). |
| `copy_options::directories_only` | Only create directory structure. |
| `copy_options::create_symlinks` | Create symlinks instead of copying file data. |
| `copy_options::create_hard_links` | Create hard links instead of copying bytes. |

**[OOPS]** Conflicting combinations yield **[ERROR]** (fail / `filesystem_error` / `ec`).

---

## Directory iterator options (p. 245)

`directory_options` applies to **`directory_iterator`** and **`recursive_directory_iterator`**:

| Option | Meaning |
|--------|---------|
| `directory_options::none` | Default. |
| `directory_options::follow_directory_symlink` | Traverse into directory symlinks during recursion. |
| `directory_options::skip_permission_denied` | Skip entries not readable instead of failing entire iteration. |

**[OK]** `directory_iterator` is a range: **`begin(dir_it)`** / **`end(dir_it)`** work with range-for without manual end object in many patterns.

---

## `directory_entry` operations and caching (p. 246–247)

`directory_entry` can cache metadata from the listing syscall to speed repeated queries.

### Typical operations / status (table, p. 246–247)

| Query (member / free function style) | Notes |
|-------------------------------------|-------|
| `entry.path()` | Path recorded for entry. |
| `exists(entry)` / `entry.exists()` | May use cache. |
| `is_regular_file` / `is_directory` / `is_symlink` | Overloads taking `directory_entry`. |
| `file_size(entry)` | Cached when possible for regular files. |
| `last_write_time(entry)` | Cached when available. |
| `status(entry)`, `symlink_status(entry)` | File status objects. |
| Hard link count, etc. | As provided overloads / members. |

**[OK]** Call **`entry.refresh()`** or **`refresh(entry, ec)`** when the file may have changed on disk since the directory was opened and you need up-to-date attributes.

**[OOPS]** Assuming freshness without **`refresh`** after external mutations can yield **[ERROR]** logic (stale size or type).

---

## Space, links, rename/remove (p. 240–243)

```cpp
fs::space_info si = fs::space(p);  // overloads with std::error_code ec also exist
// si.capacity  — total size of the filesystem
// si.free      — free space on filesystem
// si.available — free space available to unprivileged process (may be less than free)
```

**[OK]** **`rename(old, new)`** moves within the same filesystem when possible; **[ERROR]** can occur across devices depending on OS (may need copy+remove).

**[OK]** **`remove(p)`** deletes a **file** or **empty** directory; **`remove_all(p)`** recursively deletes a tree (dangerous).

**[OK]** **`read_symlink(p)`** returns the target string of a symlink.

**[OK]** **`hard_link_count(p)`** counts names for a regular file’s inode (where applicable).

**[OK]** **`create_hard_link(target, link)`** adds another directory entry for the same file data.

---

## Additional reference: `canonical` vs `weakly_canonical` (p. 213–215)

- **`canonical(p)`**: **[OK]** absolute path with symlinks resolved; **[ERROR]** if path does not exist where required by implementation.
- **`weakly_canonical(p)`**: **[OK]** more tolerant—missing tail can remain; good when constructing paths before all components exist.

---

## ASCII: path component breakdown (example)

```
/rooted/usr/local/bin/app.tar.gz

 root_path (POSIX): /
 parts:       usr ─ local ─ bin ─ app.tar.gz
                                    │
                                    ├── stem:      app.tar
                                    └── extension: .gz
```

---

## ASCII: directory iteration

```
 Flat directory_iterator on "root"
 root/
   ├─ a.txt        <- visited
   ├─ sub/         <- visited as entry (not descended)
   └─ b.txt

 Recursive recursive_directory_iterator
 root/
   ├─ a.txt
   ├─ sub/
   │    └─ c.txt   <- visited after descending
   └─ b.txt
```

---

## ASCII: error handling decision tree

```
 Need filesystem call
        │
        ├─ library / server code, central error handling?
        │      └─> [OK] error_code overloads
        │          propagate ec to response / logs
        │
        └─ app code, exceptions allowed?
               └─> [OK] default overloads
                   catch fs::filesystem_error
                   -> e.path1(), e.path2(), e.code(), what()
```

---

## Namespace idiom (p. 210)

```cpp
#include <filesystem>
namespace fs = std::filesystem;
```

---

## Portability summary (Josuttis highlights)

- Path encodings differ (UTF-8 vs wide paths on Windows APIs).
- Permission and symlink models differ between POSIX and Windows.
- Race conditions: TOCTOU patterns apply; check-then-act is fragile on shared filesystems.

---

## Where to read in the book

- **§20.1** (p. 201): motivating examples (`checkpath`, `createfiles`).
- **§20.2** (p. 210): concepts, normalization, error model, member vs free-standing.
- **§20.3** (p. 217): `path` mechanics, I/O, iteration, modification, comparison.
- **§20.4** (p. 230): mutating operations, metadata (`last_write_time`, permissions, `copy_options`).
- **§20.5** (p. 244): iterators, `directory_options`, `directory_entry` caching.

These notes are a chapter companion keyed to *C++17 - The Complete Guide*; printings may shift page numbers slightly.