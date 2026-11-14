# Engineering Standards — Rust

*Refines [GENERAL.md](GENERAL.md). Rule IDs such as `General M2` refer to that
document. Written for an engineer fluent in C++, C#, and Python and new to Rust,
so it explains the Rust idiom for each familiar idea as well as stating the rule.*

---

## 0. Sources and stance

This standard is built on four external references, in decreasing order of
authority:

1. **[Rust API Guidelines](https://rust-lang.github.io/api-guidelines/checklist.html)**
   (Rust library team). These are the canonical conventions; checklist IDs such
   as `C-METHOD` are cited below.
2. **[Rust Style Guide](https://doc.rust-lang.org/style-guide/)** as implemented
   by **rustfmt**. Formatting is not a matter of opinion (General V1).
3. **[Clippy](https://rust-lang.github.io/rust-clippy/stable/)**, the official
   linter, with about 800 lints in groups `correctness`, `suspicious`, `style`,
   `complexity`, `perf`, `pedantic`, `nursery`, `restriction`, and `cargo`.
4. **[Microsoft Pragmatic Rust Guidelines](https://microsoft.github.io/rust-guidelines/)**
   (version 2026.6). This is the most complete public industrial guide, and IDs
   such as `M-PANIC-ON-BUG` are cited below. We adopt most of it.
   **Deliberate deviations** are listed here, each with its reason.

| Microsoft guideline | Our position | Reason |
|---|---|---|
| **M-SHORT-NAMES** ("abbreviations are preferred", "no more than 2 short words") | **Rejected** as phrased; its *no-stutter* clause is kept | Conflicts with General N1. A name is as long as precision requires. We do keep the part that avoids repeating context (General N11). |
| **M-REGULAR-FN** ("prefer regular over associated functions") | **Adopted in its narrow sense**, with the API Guidelines' **C-METHOD** ("functions with a clear receiver are methods") taking precedence | Its real target is the C#-style *static utility class*, which we also reject (§4.8). It must not be read as "prefer free functions," which would contradict General M1–M2. |
| **M-ERRORS-CANONICAL-STRUCTS** (opaque error structs with backtraces and `is_xxx()` helpers) | **Adopted for published libraries only**. Applications and internal crates use `thiserror` enums (§7) | Opaque structs earn their cost when third parties depend on your semantic-versioning stability. Inside our own programs, exhaustive enums are simpler and let the compiler check our handling (General E7). |
| **M-MIMALLOC-APPS**, **M-TARGET-CPU** | **Not adopted by default** | Performance tuning, applied when profiling justifies it. A raised `target-cpu` breaks binaries distributed to other machines. |

Everything else in Microsoft's guide (`M-PANIC-ON-BUG`, `M-CANONICAL-DOCS`,
`M-STRONG-TYPES-GUARD`, `M-LINT-OVERRIDE-EXPECT`, `M-SINGLE-ITEM-PATH`,
`M-WEASEL-WORDS`, `M-DOCUMENTED-MAGIC`, `M-MOCKABLE-SYSCALLS`, `M-DI-HIERARCHY`,
`M-BALANCED-MODULES`, and others) is consistent with GENERAL.md and applies.

---

## 1. Rust for the C++ / C# / Python engineer

### 1.1 Concept map

| You know | In Rust | Note |
|---|---|---|
| class (fields + methods) | `struct` + `impl` block | Data and behavior are declared separately but live in the same module |
| constructor | Associated function: `new`, `from_*`, `with_*`, `Default`, builder | No overloading, so every alternative constructor *must* be named (General M7 comes for free) |
| destructor, `IDisposable`, `with` | `Drop` (RAII) | Deterministic, exactly like C++. There is no garbage collector. |
| interface, abstract class, `Protocol` | `trait` | Can carry default method bodies |
| virtual call | `dyn Trait`, or generics, or an `enum` | See the escalation ladder in §4.6 |
| inheritance | **None.** Composition, traits with default methods, supertraits | General M10 is enforced by the language |
| `enum class`, Python `Enum` | `enum`, and each variant **can carry its own data** | Sum types replace most small class hierarchies (§4.4) |
| `std::variant` | `enum` | With exhaustive `match` |
| `null`, `None` | `Option<T>` | No null references exist |
| exceptions | `Result<T, E>` with the `?` operator | `panic!` is for bugs only (§7) |
| templates, C# generics | Generics with trait bounds | Monomorphized like C++ templates, checked up front like C# constraints |
| namespace, package | `mod` (module); crate = package or assembly | |
| `public` / `private` / `internal` | `pub` / private (default) / `pub(crate)` | **Privacy is per module, not per type** (§4.2) |
| property `get` / `set` | `fn name(&self) -> &T` / `fn set_name(&mut self, …)` | No `get_` prefix (C-GETTER) |
| `const`, `readonly`, `frozen=True` | Everything is immutable unless declared `mut` | |
| operator overloading | Traits in `std::ops` (`Add`, `Index`, …) and `std::cmp` | |
| `ToString`, `__str__` | `impl Display` | You get `.to_string()` free |
| `__repr__` | `#[derive(Debug)]` | Every public type has it (C-DEBUG) |
| `==`, `GetHashCode`, `__lt__` | `#[derive(PartialEq, Eq, Hash, PartialOrd, Ord)]` | Field-wise, in declaration order |
| `IEnumerable`, generators | `Iterator`, `IntoIterator` | Adapter chains (`filter`, `map`, `collect`) are idiomatic, not clever |
| `unique_ptr` / `shared_ptr` | `Box<T>` / `Rc<T>`, `Arc<T>` | Rarely needed; plain ownership covers most cases |
| `std::mutex` + data | `Mutex<T>`, which **owns** the data it guards | Data cannot be touched without the lock |
| singleton, mutable static | Avoided (M-AVOID-STATICS); `LazyLock` / `OnceLock` for lazy immutable values in applications | |
| pyproject + uv | `Cargo.toml` + cargo | |
| black / clang-format | rustfmt | |
| ruff / clang-tidy | clippy | |
| mypy | The compiler | Strict, and not optional |
| pytest / xUnit | `#[test]`, `cargo test` / `cargo nextest` | |
| Sphinx / Doxygen / XML docs | rustdoc (`///`, `//!`), docs.rs, mdBook | §8 |

### 1.2 The four shifts that matter

1. **Ownership is a design tool, not a hurdle.** Every value has one owner.
   Borrowing (`&T` shared, `&mut T` exclusive) is checked at compile time. Model
   ownership the way the domain does: a `Screen` *owns* its `Output`s, and a
   `ProfileSelector` *borrows* the `Settings`. When the borrow checker objects,
   it is usually pointing at a real aliasing question your design had not
   answered.
2. **Enums with data replace class hierarchies.** Where C# would declare an
   abstract `Step` with eight subclasses, Rust usually writes one
   `enum Step { RunTo { target: String }, Equip { item: Item, slot: Slot }, … }`
   and one `match`. Adding a variant makes the compiler list every `match` that
   must handle it.
3. **Errors are values in the type signature.** A function that can fail says
   so in its return type, and `?` propagates the failure. You cannot forget to
   handle it, and you can see every failure point by reading the code.
4. **Traits are capabilities, not ancestors.** A type *implements* `Display`,
   `Ord`, and `ServerBackend`; it does not *descend* from them. Small traits
   compose, and big trait hierarchies fight the language.

---

## 2. Toolchain and project skeleton

**R-T0. New projects start from the `sevaht-rust-project` cookiecutter
template** (`~/scm/personal/production/project-templates/rust-project`), the
sibling of the Python template (PY-T1). Its configuration files and `checks`
script are copied from this document. When a convention here changes, update
the template in the same change and regenerate a project to confirm it still
passes `./checks` and `./checks --docs`.

**R-T1. Edition and toolchain.** New crates use the latest edition (**2024**)
(M-LATEST-EDITION). Each repository pins its toolchain channel in
`rust-toolchain.toml` and declares its minimum supported Rust version (MSRV)
with `rust-version` in `Cargo.toml`. Raise the MSRV deliberately and record the
change in release notes (M-MSRV).

```toml
# rust-toolchain.toml
[toolchain]
channel = "stable"
components = ["rustfmt", "clippy", "llvm-tools"]   # llvm-tools: ./checks --coverage
```

**R-T2. Every application is a library plus a thin binary.** Logic lives in
`src/lib.rs` and its modules. `src/main.rs` is only the entry point: parse
arguments, install logging, call into the library, and map errors to an exit
code (General O4). This is the Rust equivalent of `application.py` /
`__main__.py`. It also matters because **doctests and integration tests can only
exercise a library target**, not a binary.

```
project-name/
├── Cargo.toml
├── Cargo.lock           # committed for applications
├── rust-toolchain.toml
├── rustfmt.toml
├── clippy.toml
├── deny.toml
├── checks               # the single verification entry point (General V6)
├── README.md            # user-facing (General D10)
├── DESIGN.md            # maintainer-facing
├── src/
│   ├── main.rs          # entry point only
│   ├── lib.rs           # crate docs (//!), module declarations, top-level exports
│   ├── screen.rs        # module `screen`
│   ├── screen/          # submodules of `screen`, when it grows
│   │   └── output.rs
│   └── wlr_randr.rs
├── tests/               # integration tests: one crate per file, public API only
└── examples/            # runnable usage examples (libraries)
```

**R-T3. Module files use the `name.rs` + `name/` layout, never `mod.rs`.**
`mod.rs` files make every editor tab read "mod.rs". The convention is enforced
with `clippy::mod_module_files`.

**R-T4. Workspaces for more than one crate.** When a project needs several
crates (for example a reusable library plus a tool), use a Cargo workspace with
crates side by side under `crates/`. Shared settings (`edition`, `rust-version`,
`[workspace.lints]`, dependency versions) are inherited from the workspace root
(M-CARGO-WORKSPACE, M-CRATES-FLAT-FOLDER).

**R-T5. Complete package metadata** (C-METADATA): `description`, `license`,
`repository`, `readme`, `keywords`, `categories`, and `rust-version`.

**R-T6. The `checks` script** mirrors the Python template's. It fixes what can
be fixed and fails on anything else. **Documentation never gates the regular
run** (General V7): missing documentation is reported as a warning there, and
only `./checks --docs` treats it as a failure. `./checks --coverage` reports
test coverage (R-TEST7) and never fails on the result.

```bash
#!/usr/bin/env bash
# Runs every check this repository gates on, stopping at the first failure.
# With --docs, runs only the documentation checks, and those gate instead.
# With --coverage, runs the tests and reports how much code they exercise.
set -euo pipefail
cd "$(dirname "${BASH_SOURCE[0]}")"

# Lints about documentation. They warn in ordinary builds and in the regular
# checks, and fail only under --docs.
documentation_lints=(
    missing_docs
    clippy::missing_docs_in_private_items
    clippy::missing_errors_doc
    clippy::missing_panics_doc
    clippy::doc_markdown
)

run_check() {
    local description=${1}
    shift
    echo "Running ${1} ${2}... (${description})"
    "${@}"
    echo
}

# Builds "--<level> lint" argument pairs for every documentation lint.
documentation_lint_arguments() {
    local level=${1}
    local lint
    for lint in "${documentation_lints[@]}"; do
        echo "--${level}" "${lint}"
    done
}

check_code() {
    # --force-warn keeps the documentation lints from failing this run.
    run_check "formatting code" cargo fmt --all
    run_check "linting" cargo clippy --workspace --all-targets --all-features \
        -- --deny warnings $(documentation_lint_arguments force-warn)
    run_check "executing tests" cargo nextest run --workspace --all-features \
        --no-tests=warn
    run_check "executing doctests" cargo test --workspace --doc
    run_check "auditing dependencies" cargo deny check
    run_check "finding unused dependencies" cargo machete
}

check_documentation() {
    run_check "linting documentation" cargo clippy --workspace --all-targets \
        --all-features -- $(documentation_lint_arguments deny)
    RUSTDOCFLAGS="--deny warnings" run_check "building documentation" \
        cargo doc --workspace --no-deps --document-private-items
}

check_coverage() {
    # llvm-cov fails to report when no test ran; say so plainly instead.
    local tests
    tests=$(cargo nextest list --workspace --all-features \
        --message-format oneline)
    if [[ -z ${tests} ]]; then
        echo "No tests to measure coverage of."
        return
    fi
    run_check "measuring test coverage" cargo llvm-cov nextest --workspace \
        --all-features --no-tests=warn --no-report
    run_check "summarizing coverage" cargo llvm-cov report
    run_check "writing the coverage report" cargo llvm-cov report --html
    echo "Line-by-line report: target/llvm-cov/html/index.html"
}

case ${1:-} in
    "") check_code ;;
    --docs) check_documentation ;;
    --coverage) check_coverage ;;
    *)
        echo "usage: ${0} [--docs | --coverage]" >&2
        exit 2
        ;;
esac
echo "All checks completed successfully!"
```

The required tools are `cargo-nextest` (a faster, clearer test runner that does
not run doctests, hence the separate step), `cargo-deny` (license, advisory,
and duplicate-version policy), `cargo-machete` (unused dependencies: the role
vulture plays for Python), and, for `--coverage`, `cargo-llvm-cov`. CI runs the same script with `cargo fmt --check`.
Libraries also run `cargo semver-checks` before release. Crates containing
`unsafe` also run `cargo miri test` (M-UNSAFE).

---

## 3. Naming

General §2 applies in full. Rust's casing ([RFC 430](https://rust-lang.github.io/api-guidelines/naming.html), C-CASE):

| Item | Convention | Example |
|---|---|---|
| Crates (`Cargo.toml`) | `kebab-case` | `av-output-switcher` |
| Crates (in code), modules | `snake_case` | `av_output_switcher`, `wlr_randr` |
| Types, traits, enum variants | `UpperCamelCase` | `ProfileSelector`, `NameStyle::SnakeCase` |
| Functions, methods, locals, fields | `snake_case` | `apply_profile`, `column_count` |
| Constants and statics | `SCREAMING_SNAKE_CASE` | `CONFIG_FILE_NAME` |
| Generic type parameters | `UpperCamelCase`, usually one letter by convention | `T`, `K`, `V`, `E`, `R: Read`, `W: Write`, `F: Fn…` |
| Lifetimes | lowercase, short | `'a`, or descriptive (`'input`) when several are in play |

**R-N1. Acronyms are one word in camel case.** Write `CliTool`, `HttpServer`,
`SniRegistrationError`, `JsonValue`. This is unlike the Python projects'
`CLITool`, but it is the universal Rust convention (C-CASE) and rustfmt and
clippy (`upper_case_acronyms`) assume it.

**R-N2. Conversion prefixes carry cost semantics** (C-CONV):

| Prefix | Cost | Ownership | Example |
|---|---|---|---|
| `as_` | Free | Borrowed → borrowed | `str::as_bytes`, `Path::as_os_str` |
| `to_` | Expensive (allocates or computes) | Borrowed → owned | `str::to_lowercase`, `Position::to_cli_string` |
| `into_` | Variable | Owned → owned (consumes `self`) | `String::into_bytes` |
| `from_` | Variable | Constructor | `Mode::from_wlr_randr_line` |
| `try_` | Variable | Fallible version of the above | `try_from_millihertz` |

**R-N3. Getters have no `get_` prefix** (C-GETTER). A field `name` is read
through `fn name(&self) -> &str`. The bare name `get` is reserved for keyed
lookups on collections (`map.get(&key)`) and for unwrapping a newtype's single
value (`NonZeroU32::get`, `ChargePercent::get`). **A getter is cheap and infallible.** An
operation that performs I/O, runs a process, or can fail is not a getter. It
gets a verb (`read_status`, `query_sinks`, `list_batteries`) and returns
`Result`.

**R-N4. Iterator methods** are `iter`, `iter_mut`, and `into_iter` for
collections (C-ITER). Iterator-returning queries are plural nouns
(`active_outputs()`), and the iterator types are named after the method that
creates them (C-ITER-TY).

**R-N5. Predicates** are `is_*`, `has_*`, or `can_*` and return `bool`.
`with_*` returns a modified copy (or consumes and returns `self`). `set_*`
mutates in place.

**R-N6. Do not stutter, but stay unambiguous unqualified.** In Rust, *types* are
almost always imported and used unqualified (`use crate::config::ConfigFile;`),
so a type's name must stand on its own. That makes `ConfigFile` and
`ConfigError` correct even inside module `config`. A bare `File` would collide
with `std::fs::File` at every use site. *Functions* are idiomatically called
through their module (`naming::convert_name`, `fs::read_to_string`), so they do
not repeat the module name: `convert_name`, not `naming_convert_name`.
`clippy::module_name_repetitions` therefore stays **off** (General N11).

Error types follow the same split, and both halves are standard-library
practice:

- **The error of a crate (or a module) whose whole subject is one failure
  domain** is named `Error`, next to a `Result<T>` alias, and is always written
  qualified: `io::Error`, `fmt::Error`, `serde_json::Error`, `reqwest::Error`.
  Import the module, not the type (`use std::io;`, then `io::Error`).
- **An error belonging to one operation or concept inside a larger crate** is
  named for that situation: `ParseIntError`, `Utf8Error`, `TryFromIntError`
  (all in `std`), and here `ConfigError`. Microsoft's guidelines name errors
  the same way (`DownloadError`, M-ERRORS-CANONICAL-STRUCTS).

The test: if the name would read as "an error" with no qualifier, the module
name is doing the work and the type is `Error`. Otherwise the type names the
situation. A small application crate is the second case almost every time.

**R-N7. Whole words, enforced.** `clippy::min_ident_chars` flags
single-character identifiers except the conventional ones listed in
`clippy.toml` (§12): `x`, `y`, `z`, `w` for coordinates, `i`, `j` for indices,
`n` for a count. That list is also clippy's own default, so the lint targets the
meaningless single letters (`|o|`, `let s = …`), not the conventional ones. Closure
parameters are words: `.filter(|output| output.is_active())`, never `|o|`
(General N3).

---

## 4. Modeling

This section is the Rust vocabulary for General §3.

### 4.1 Structs and `impl` blocks: the class equivalent

A Rust "class" is a `struct` (its state) and an `impl` block (its behavior) in
the same module. Trait implementations (`impl Display for Screen`) are
additional, separately declared blocks of behavior.

**R-M1. One primary type per module** is the default organization unit. The
module's name is the concept, the primary type is its embodiment, and closely
related small types (its errors, its value parts, its iterator) live beside it.
`screen.rs` holds `Screen`, `Output`, and `Configuration`. `config.rs` holds
`ConfigFile` and `ConfigError`.

**R-M2. A single inherent `impl` block per type**, ordered as follows. Split
blocks only for different generic bounds or `cfg` conditions.

```rust
impl Screen {
    // 1. Associated constants
    // 2. Constructors: new, from_*, try_from_*, with_capacity…
    // 3. Accessors: cheap, infallible field reads (R-N3)
    // 4. Queries: &self methods that compute answers
    // 5. Commands: &mut self methods that change state
    // 6. Consuming transformations: self → something (with_*, into_*)
    // 7. Private helpers
}

// Then trait implementations, in this order: formatting (Display, manual Debug),
// conversions (From, TryFrom, FromStr, AsRef), operators and comparison
// (Add, PartialOrd…), iteration (IntoIterator, FromIterator, Extend),
// then everything else.
```

### 4.2 Encapsulation: privacy is per module

**R-M3. Fields are private when the type has an invariant**, and always private
in published libraries (C-STRUCT-PRIVATE). Outside code reads them through
accessors and changes them through methods that maintain the invariant.
**Passive value types without invariants** (`Position { x, y }`,
`Resolution { width, height }`) MAY have public fields inside an application.
That is the honest model of "a pair of numbers."

**R-M4. Know that privacy is module-wide.** Every item in a module can see every
private field of every type in that module. This is the reason for R-M1. A type
whose invariants matter should not share its module with unrelated code that
could bypass them.

**R-M5. Visibility is as narrow as works.** Use private, then `pub(crate)`, then
`pub`. `pub(crate)` is the Rust equivalent of C#'s `internal`, and is the
default for things shared across modules of an application.
`unreachable_pub` (§12) catches `pub` items that are not actually exported.

### 4.3 Construction and validity

**R-M6. `new` is the primary constructor.** Alternatives are named for their
source (`from_millihertz`, `from_wlr_randr_json`) (C-CTOR, General M7). A type
with an obvious empty or neutral value implements `Default`.

**R-M7. Newtypes guard their invariants** (M-STRONG-TYPES-GUARD, General M6, M9).
A type that exists to encode a constraint (a percentage, a port, a non-empty
name) has a private field and a **fallible** constructor. Conversions from raw
values go through `TryFrom` or `FromStr`, never through an infallible `From`.

```rust
/// A battery charge, as a whole percentage between 0 and 100 inclusive.
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct ChargePercent(u8);

impl ChargePercent {
    /// The largest valid charge.
    pub const FULL: Self = Self(100);

    /// Creates a charge, or returns `None` if `percent` exceeds 100.
    #[must_use]
    pub const fn new(percent: u8) -> Option<Self> {
        if percent <= Self::FULL.0 {
            Some(Self(percent))
        } else {
            None
        }
    }

    /// Returns the charge as a number between 0 and 100.
    #[must_use]
    pub const fn get(self) -> u8 {
        self.0
    }
}
```

**R-M8. Builders for many optional parameters** (C-BUILDER, M-BUILD-RESULT).
When a constructor would need more than about four arguments, most of them
optional, provide `Type::builder()` returning a `TypeBuilder` with one method
per option. A final `.build()` validates and returns `Result<Type, _>`. A small
options struct implementing `Default` (`CsvLoadOptions { delimiter: b';',
..CsvLoadOptions::default() }`) is the lighter alternative and is often enough.

**R-M9. Typestate for protocols with required ordering.** When operations are
only legal in certain states (unauthenticated, then authenticated; draft, then
published), encode the state as a type parameter. Calling an operation in the
wrong state then fails to compile.

```rust
/// The state of a post that is still being written.
#[derive(Debug)]
pub struct Draft;

/// The state of a post that readers can see.
#[derive(Debug)]
pub struct Published;

/// A blog post whose lifecycle stage is part of its type.
#[derive(Debug)]
pub struct Post<State> {
    /// The text of the post.
    body: String,
    /// Marks the stage; occupies no memory.
    state: PhantomData<State>,
}

impl Post<Draft> {
    /// Starts a draft with the given text.
    #[must_use]
    pub fn new(body: impl Into<String>) -> Self {
        Self {
            body: body.into(),
            state: PhantomData,
        }
    }

    /// Publishes the post; drafts cannot be read until this is called.
    #[must_use]
    pub fn publish(self) -> Post<Published> {
        Post {
            body: self.body,
            state: PhantomData,
        }
    }
}

impl Post<Published> {
    /// Returns the published text.
    #[must_use]
    pub fn body(&self) -> &str {
        &self.body
    }
}
```

Use this where the protection is worth the ceremony, not everywhere.

### 4.4 Enums: closed sets with behavior

**R-M10. A closed set of alternatives is an `enum`** (General M8). This covers
plain sets (`NameStyle`), and alternatives carrying different data (`Step`,
`Command`). Behavior that varies by variant is a method that `match`es on
`self`. Do not keep a parallel lookup table elsewhere.

```rust
/// A supported identifier casing convention.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum NameStyle {
    /// `words_joined_like_this`.
    SnakeCase,
    /// `words-joined-like-this`.
    KebabCase,
    /// `wordsJoinedLikeThis`.
    CamelCase,
    /// `WordsJoinedLikeThis`.
    PascalCase,
}

impl NameStyle {
    /// Returns the text placed between words.
    #[must_use]
    pub const fn separator(self) -> &'static str {
        match self {
            Self::SnakeCase => "_",
            Self::KebabCase => "-",
            Self::CamelCase | Self::PascalCase => "",
        }
    }

    /// Returns whether the first word starts with a capital letter.
    #[must_use]
    pub const fn capitalizes_first_word(self) -> bool {
        matches!(self, Self::PascalCase)
    }
}
```

Compare this with the Python `NameStyle`, which needs `__new__` to attach a
config tuple. In Rust the data lives in the method, and adding a fifth style
makes the compiler point at every `match` that must learn about it.

**R-M11. Matches over our own enums are exhaustive, with no wildcard arm**
(General E7, enforced by `clippy::wildcard_enum_match_arm`). A `_ =>` arm
silently swallows future variants.

**R-M12. Public enums that may grow are `#[non_exhaustive]`** in published
libraries. Downstream code must then include a wildcard arm, and adding a
variant is not a breaking change.

### 4.5 Value types: derive the vocabulary

**R-M13. Types eagerly implement the common traits** (C-COMMON-TRAITS). For
value objects (General M5), the default derive set is:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash, Default)]
```

Drop `Copy` when the type owns heap data (`String`, `Vec`). Drop
`PartialOrd`/`Ord` when there is no meaningful order. Drop `Default` when there
is no meaningful neutral value. Drop `Eq`/`Ord`/`Hash` when a field is a float.
(Floats are not totally ordered. Prefer integer units such as millihertz or
basis points for quantities that need comparison, General M9.) Every public
type is at least `Debug` (C-DEBUG). Types users read are also `Display`
(M-PUBLIC-DISPLAY).

**R-M14. Modified copies use `with_*` and consume `self`.** This avoids a clone
when the caller no longer needs the original:

```rust
#[must_use]
pub fn with_primary(mut self, name: &str) -> Self { /* … */ self }
```

### 4.6 Traits and polymorphism: the escalation ladder

Follow this order (M-DI-HIERARCHY), and stop at the first rung that works:

1. **A concrete type.** Most code has one implementation. Write the struct.
2. **An enum of implementations** when the set is closed and known: platform
   backends, or a real implementation plus a test double (M-MOCKABLE-SYSCALLS).
3. **A trait plus generics** (`fn apply(runner: &impl CommandRunner)`,
   `struct ProfileSelector<Runner: CommandRunner>`) when the set is open, for
   example users supplying implementations. This is statically dispatched and
   zero-cost.
4. **`dyn Trait`** only when generics would spread through too many signatures,
   or when heterogeneous collections are needed. Wrap it in a named type
   (`struct DynamicBackend(Box<dyn Backend>)`) instead of exposing `Box<dyn …>`
   in APIs (M-AVOID-WRAPPERS).

**R-M15. Traits name capabilities and stay narrow.** Prefer several small traits
(`LoadObject`, `StoreObject`) combined by a supertrait when needed. Avoid one
broad `IDatabase`-style interface, which is the C# habit that does not translate
(M-RUST-SHAPED). Do not create a trait with one implementation "for
testability" when an enum (rung 2) or a fake at the I/O boundary would do.

### 4.7 Resources: `Drop` is the context manager

**R-M16. Cleanup belongs in `Drop`** (General E8). A type that owns a resource
(a temporary directory, a child process, a socket, a lock) releases it in
`Drop`, so it is released on every path, including `?` early returns and
panics. Destructors must not fail (C-DTOR-FAIL). If release can fail
meaningfully, also provide an explicit `close(self) -> Result<…>` method and let
`Drop` be the best-effort fallback.

### 4.8 Methods, associated functions, and free functions

This is the rule our standards care about most, stated in Rust terms.

**R-M17. If a function has a clear receiver, it is a method** (C-METHOD,
General M1). If its main subject is a `Screen`, it takes `&self`, `&mut self`, or
`self` on `Screen`. The shared-first-argument test (General M2) applies
verbatim: three free functions taking `&Path` first are a struct holding a
`PathBuf`.

**R-M18. Associated functions (no `self`) are for construction**, plus a few
operations that are intrinsically "of the type" (`Ord::max`-like combinators,
`T::parse_all`). A function that does not concern the type does not move into
its `impl` merely for namespacing. That is the static-utility-class pattern
M-REGULAR-FN rightly rejects.

**R-M19. True free functions live in a module named for their concept**
(General M15). When there is no receiver (the operation is symmetric, purely
mathematical, or an entry point), write a free function in a module whose name
supplies the noun, and call it qualified: `naming::split_into_words(name)`. A
module containing only loosely related free functions is a junk drawer
(General O3).

**R-M20. Operations on someone else's type use an extension trait** only when
the operation reads naturally as a method of that type and is used widely. For
example, a `PathExt` trait adding `.read_trimmed()` to `Path`. Otherwise, write
a free function in a concept module. Do not reach for extension traits by
reflex.

---

## 5. Ownership in signatures

**R-O1. Borrow what you only read; take what you keep.**

| The function needs to… | Parameter type |
|---|---|
| Read text | `&str` |
| Read a path | `&Path`, or `impl AsRef<Path>` at public boundaries (M-IMPL-ASREF) |
| Read a sequence | `&[T]`, or `impl IntoIterator<Item = T>` when any iterable will do |
| Store text it is given | `String`, or `impl Into<String>` for caller convenience |
| Modify the caller's value | `&mut T` |
| Consume the value | `T` |
| Read or write bytes or streams | `impl Read` / `impl Write` taken by value (C-RW-VALUE) |

**R-O2. Return owned values from computations and borrowed views from
accessors.** `fn name(&self) -> &str`, but `fn display_label(&self) -> String`.

**R-O3. Structs own their data.** A struct holding references (and therefore a
lifetime parameter) is reserved for short-lived *views*, such as a parser over
an input buffer or an iterator over a collection. Long-lived domain objects own
their fields. When in doubt, own and `clone()`. A clone is cheap compared with a
lifetime parameter threaded through twenty signatures, and it can be optimized
later if profiling shows it matters.

**R-O4. Shared mutable state is a design decision, not a workaround.**
`Rc<RefCell<T>>` or `Arc<Mutex<T>>` reaching for every object is the sign of a
graph the design has not untangled. When genuinely needed (shared state across
threads), wrap it inside a named type with methods. Never expose
`Arc<Mutex<…>>` in a public signature (M-AVOID-WRAPPERS).

**R-O5. Generic over `impl Trait` in argument position** keeps signatures
readable. Write `fn load(source: impl Read)`, not
`fn load<R: Read>(source: R)`, unless the type parameter is needed elsewhere in
the signature.

---

## 6. File anatomy

```rust
//! One-line summary of the concept this module models.
//!
//! What the module's types are, how they relate, when to use them, and what
//! side effects they have (General D4, M-MODULE-DOCS).

use std::fmt;                          // 1. std, core, alloc
use std::path::{Path, PathBuf};

use serde::Deserialize;                // 2. external crates

use crate::cli_tool::CliTool;          // 3. this crate (crate::, super::)

/// Documented constant, with why this value (General K1, M-DOCUMENTED-MAGIC).
pub const CONFIG_FILE_NAME: &str = "config.json";

/// The primary type of this module.
pub struct Primary { /* … */ }

impl Primary { /* … ordered per R-M2 … */ }

impl fmt::Display for Primary { /* … */ }

// Secondary public types, then private items.

#[cfg(test)]
mod tests {
    use super::*;

    // …
}
```

**R-F1. Imports are grouped std / external / crate**, separated by blank lines,
and each group is sorted. rustfmt sorts within a group, but cannot yet group on
stable, so we group by hand. In `main.rs` and in `tests/`, the package's own
library is imported by its crate name; it still gets a group of its own, after
the external crates. Otherwise its position would depend on how the project
name sorts against them. Glob imports are banned outside `use super::*` in
test modules (`clippy::wildcard_imports`).

**R-F2. Unit tests sit at the bottom of the file they test**, in
`#[cfg(test)] mod tests`. This is the Rust convention, and it lets tests reach
private items.

---

## 7. Errors and panics

**R-E1. Recoverable failures return `Result`; bugs panic** (General E1,
M-PANIC-ON-BUG). A malformed config file is a `Result::Err`. An index that our
own logic guarantees is in range, but is not, is a bug and panics.

**R-E2. Library and module errors are `thiserror` enums**, with one variant per
failure a caller might handle differently, and named fields for context
(General E2):

```rust
/// Why a configuration file could not be read or written.
#[derive(Debug, thiserror::Error)]
pub enum ConfigError {
    /// The file exists but could not be read.
    #[error("could not read configuration file {}", .path.display())]
    Read {
        /// The file's location.
        path: PathBuf,
        /// What the operating system reported.
        source: io::Error,
    },
    // …
}
```

Each named field gets its own doc comment: `missing_docs` treats the fields of
a public enum's variants as public items.

Each error type is scoped to a module or operation family. Do not create one
`GlobalError` enum for the whole crate (M-ERRORS-CANONICAL-STRUCTS: "Error kinds
… should not be used to avoid creating separate public error types when there is
otherwise no error overlap").

**R-E3. Applications use `anyhow` at the top.** `main` and the top-level
coordinators return `anyhow::Result<T>` and add human context as errors cross
layers (M-APP-ERROR):

```rust
let settings = Settings::from_file(&path)
    .with_context(|| format!("loading profiles from {}", path.display()))?;
```

**R-E4. Error message style** follows the standard library: **lowercase, no
trailing punctuation, and without the underlying cause's text.** The cause is
reported through `source()`, and printing the chain (`{error:#}` with `anyhow`)
assembles the full story. For example: `could not read configuration file
/home/…/config.json: permission denied (os error 13)`.

**R-E5. No `unwrap()` outside tests** (`clippy::unwrap_used`). Where failure
would be a bug, use `expect`, phrased as **the reason it cannot fail**, per the
standard library's recommendation:

```rust
let document = serde_json::to_string_pretty(&map)
    .expect("a JSON object with string keys always serializes");
```

**R-E6. Every public function that returns `Result` has an `# Errors` doc
section, and every function that can panic has a `# Panics` section**
(C-FAILURE; `clippy::missing_errors_doc`, `missing_panics_doc`).

**R-E7. Never discard errors silently.** Converting `Result` to `Option` with
`.ok()` discards information. Do it only when absence is the correct reading, and
comment why. `let _ = fallible();` requires a comment. `map_err(|_| …)` that
drops the cause is flagged by `clippy::map_err_ignore`.

---

## 8. Documentation generated from the code

Rust has the best documentation tooling of any mainstream language. Every item
documents itself in `///` comments, which render to a searchable, cross-linked
HTML site with `cargo doc`, appear in every IDE on hover, and contain examples
that are compiled and run as tests.

**R-D1. Every module has `//!` documentation; every public item has `///`
documentation** (General D1, D4; lints `missing_docs` and
`clippy::missing_docs_in_private_items`, §12). Applications document private
items too. These lints warn; they fail only under `./checks --docs` (R-T6). The generated site (`cargo doc --document-private-items`) *is* the
architecture reference for maintainers.

**R-D2. The canonical shape** (M-CANONICAL-DOCS). Summary lines use the third
person, matching `std` (*"Returns the primary output…"*). This differs from
Python's imperative PEP 257 style. Follow each ecosystem's convention.

```rust
/// Returns the primary output, falling back to the first active output.
///
/// Wayland has no notion of a primary output; "primary" is this tool's own
/// choice, made with [`Screen::with_primary`] and persisted by
/// [`PrimaryStore`](crate::primary::PrimaryStore).
///
/// Returns `None` only when no output is active.
///
/// # Examples
///
/// ```
/// # use av_output_switcher::screen::Screen;
/// let screen = Screen::default();
/// assert!(screen.primary_output().is_none());
/// ```
#[must_use]
pub fn primary_output(&self) -> Option<&Output> { /* … */ }
```

Parameters are described in prose, not in a parameter table (M-CANONICAL-DOCS).
Rust signatures already carry the types. The sections, in order when they apply,
are `# Examples`, `# Errors`, `# Panics`, and `# Safety`.

**R-D3. Link everything.** Intra-doc links (``[`Screen`]``,
``[`Screen::with_primary`]``, ``[`crate::config`]``) are resolved and checked by
rustdoc (C-LINK). Broken links fail the build (§12).

**R-D4. Examples are doctests.** They use `?` rather than `unwrap` (C-QUESTION-MARK)
and hide setup lines with `# `. Doctests compile as an *external* crate, so they
can only use the public API. Examples for private items belong in unit tests.

**R-D5. The crate root documents the crate.** `lib.rs` opens with a thorough
`//!` overview: what the crate is, its core types, and a worked example
(C-CRATE-DOC). For libraries, the README can be included verbatim so the two
never drift: `#![doc = include_str!("../README.md")]`.

**R-D6. Narrative documentation uses mdBook.** Tutorials, the hello-world guide,
and design explanations that span modules are written in Markdown and built with
[mdBook](https://rust-lang.github.io/mdBook/), the tool used for the Rust book
itself and the Rust analogue of Sphinx + MyST. Published libraries get API docs
on docs.rs automatically. Applications publish `cargo doc` output and the mdBook
to GitHub Pages from CI. Diagrams in rustdoc MAY use the `aquamarine` crate
(Mermaid in doc comments). *(Optional; evaluate before adopting.)*

**R-D7. Mark re-exports `#[doc(inline)]`** so they render as first-class items
(M-DOC-INLINE). Each public item is reachable by exactly one public path
(M-SINGLE-ITEM-PATH).

---

## 9. Testing

**R-TEST1. Three kinds of tests, three places.** Unit tests go in
`#[cfg(test)] mod tests` beside the code, and may test private behavior.
Integration tests go in `tests/`, use the public API only, and are each compiled
as a separate crate (M-INTEGRATION-TESTS). Doctests go in documentation (§8).

**R-TEST2. Parametrized tables use `rstest`**, the pytest equivalent, including
fixtures and named cases (General T5):

```rust
#[rstest]
#[case::active_output_is_kept("DP-1", Some("DP-1"))]
#[case::inactive_output_falls_back("HDMI-A-1", Some("eDP-1"))]
fn with_primary_selects_expected_output(
    #[case] requested_name: &str,
    #[case] expected_name: Option<&str>,
) {
    let screen = sample_screen().with_primary(requested_name);
    let primary_name = screen.primary_output().map(|output| output.name.as_str());
    assert_eq!(primary_name, expected_name);
}
```

**R-TEST3. Use property tests for invariants** with `proptest`. For example,
"converting a name twice equals converting it once" or "parsing what we
formatted yields the original value." These satisfy General T2 without
hand-computing every case.

**R-TEST4. Use snapshot tests for rich output** with `insta`. Rendered tables,
`--state` output, and generated files are compared against reviewed snapshots
instead of hand-written strings.

**R-TEST5. CLI integration tests** run the built binary with `assert_cmd` and
`predicates`, using `tempfile` for isolated file systems.

**R-TEST6. I/O is replaceable** (General T4, M-MOCKABLE-SYSCALLS). Gateways that
run processes or touch `/sys` are reached through a narrow trait or a
real-or-fake enum (§4.6 rungs 2–3), so coordinators are tested against fakes.

**R-TEST8. Hardware backends split into a pure planner and a thin driver**
(General T7, T8). A backend over devices (DRM, libinput, udev, a bus) keeps its
decisions in plain-value types: what the devices reported goes in as data
(connectors, modes, events), what to do comes out as data (a plan, a list of
changes), and unit and property tests drive the planner with every case,
including hotplug sequences, missing devices and failures. The driver that calls
the kernel only reads the world into those values and carries the plan out, and
is kept small enough to review line by line. A simulated backend (headless)
runs the whole program end to end in tests.

**R-TEST7. Coverage is a guide, not a gate.** `./checks --coverage` runs the
unit and integration tests under `cargo llvm-cov` and prints per-file line,
function, and region coverage, with a line-by-line HTML report under
`target/llvm-cov/html/`. Use it to find untested behavior, not to chase a
number: there is no required percentage. Doctests are not measured; that needs
a nightly toolchain.

---

## 10. Logging and command-line applications

**R-A1. Diagnostics go through `tracing`** with structured fields, not
interpolated strings (M-LOG-STRUCTURED):
`tracing::info!(profile = %name, "applying profile")`. Libraries never install a
subscriber. Applications install one in `main` (`tracing-subscriber`) and map
`-v`/`-q`/`--debug` to filter levels, as `add_log_arguments` does in Python.
Use `#[tracing::instrument]` on a function when its arguments are useful
context for every message inside it, and `skip(...)` arguments that are large
or secret. Records from dependencies that use the `log` crate reach the same
outputs, through `tracing-subscriber`'s default `tracing-log` feature.
`println!` is for a CLI's actual output only (`clippy::print_stdout` is enabled
and expected at the output layer).

**R-A2. Arguments use `clap` with derive.** The argument structure is a typed
model of the command line, documented by its doc comments (which become
`--help`):

```rust
/// Switches audio and video outputs between saved profiles.
#[derive(Debug, clap::Parser)]
#[command(version, about)]
pub struct Arguments {
    /// How much diagnostic output to print.
    #[command(flatten)]
    pub logging: LoggingArguments,

    /// The action to perform.
    #[command(subcommand)]
    pub command: Command,
}

/// What the tool should do.
#[derive(Debug, clap::Subcommand)]
pub enum Command {
    /// Prints the screen state as known by the tool.
    State,
    /// Applies the named profile.
    Apply {
        /// The profile to apply, as named in the settings file.
        profile: String,
    },
    /// Cycles to the next profile that fits the connected outputs.
    CycleProfile,
}
```

Dispatching on `Command` is an exhaustive `match`, so a new command cannot be
forgotten.

**R-A3. Shape the command line the way the ecosystem does** (cargo, git,
rustup, and every clap tool follow the same grammar):

| Element | Use it for | Example |
|---|---|---|
| **Subcommand** | Each distinct *action* the tool performs | `tool apply`, `tool cycle-profile` |
| **Positional argument** | The thing the action acts on | `tool apply living-room` |
| **Option (`--name value`)** | A setting that modifies the action | `--settings PATH`, `--timeout-seconds 5` |
| **Flag (`--name`)** | A switch on the action, worded as what it turns on | `--dry-run`, `--english-only` |
| **Global option** (`global = true`) | Settings shared by every action | `--verbose`, `--quiet` |

A tool that only does one thing has no subcommands. Mutually exclusive *flags*
that pick an action (`--apply X` vs `--cycle`) are the argparse workaround that
subcommands replace. Keep mutual exclusion (`#[group(multiple = false)]`,
`conflicts_with`) for alternative *modifiers* of one action.

Clap's derive model does what argparse makes awkward:

- **Each subcommand has its own typed arguments.** `Apply { profile }` holds
  exactly what `apply` needs, so there is no flat namespace of options where
  most are irrelevant to the chosen action, and no `if args.command == "apply"`
  checks.
- **Enumerated values are enums.** A `#[derive(clap::ValueEnum)]` type lists its
  allowed values in `--help`, rejects anything else, and arrives as the enum:
  no string-to-enum mapping in the tool.
- **A default action** is `command: Option<Command>`, where `None` means the
  default action. That is clearer than argparse's `set_defaults` tricks.
- **Values can come from the environment** (`#[arg(env = "AV_SETTINGS")]`), and
  the help text says so.
- **Shell completions and man pages are generated from the same definition**
  (`clap_complete`, `clap_mangen`), like `--help` itself (General D1).

Option names are whole words in kebab case (`--cycle-profile`), derived from the
field name. Short forms (`-v`) are given only to options used often enough to
earn them.

**R-A4. `main` is small and returns `ExitCode`:**

```rust
fn main() -> ExitCode {
    let arguments = Arguments::parse();
    arguments.logging.install();
    match av_output_switcher::run(arguments.command) {
        Ok(()) => ExitCode::SUCCESS,
        Err(error) => {
            tracing::error!("{error:#}");
            ExitCode::FAILURE
        }
    }
}
```

---

## 11. Ecosystem defaults

Use these instead of reinventing them (General X1). Add a crate only when it
earns its place (General X2).

| Need | Crate | Python analogue |
|---|---|---|
| Argument parsing | `clap` (derive) | argparse |
| Serialization | `serde` + `serde_json` / `toml` | json, tomllib, dataclasses |
| Library errors | `thiserror` | custom exception classes |
| Application errors | `anyhow` | — |
| Logging | `tracing`, `tracing-subscriber` | logging, `log_utility` |
| Platform directories | `directories` | platformdirs (same caveat: the organization name is used only on Windows and macOS) |
| Regular expressions | `regex` | re |
| Process and system info | `sysinfo` | psutil |
| Lazy statics | `std::sync::LazyLock` / `OnceLock` | `@functools.cache` |
| Terminal styling | `anstream` / `owo-colors` | rich |
| Parametrized tests | `rstest` | pytest parametrize and fixtures |
| Property tests | `proptest` | hypothesis |
| Snapshot tests | `insta` | — |
| CLI tests | `assert_cmd`, `predicates`, `tempfile` | subprocess plus `tmp_path` |
| Async runtime, only when needed | `tokio` | asyncio |

**R-X1. Do not go async unless the program is I/O-concurrent** (servers, many
simultaneous connections). Desktop utilities and CLI tools are synchronous.

**R-X2. `unsafe` is forbidden by default** (`unsafe_code = "forbid"`). A crate
that genuinely needs it (FFI, memory-mapped save files) isolates it in one
module, documents each block with a `// SAFETY:` comment proving soundness, and
runs Miri (M-UNSAFE, M-UNSOUND).

**R-X3. Prefer the safe design, even when a library shows an unsafe one.**
Calling a library's `unsafe fn` is writing `unsafe` code, so R-X2 applies to it
in full. When a dependency's examples reach for `unsafe` (a `get_mut` on a
shared handle, an `fd` taken by raw value), look for the safe arrangement
first. Restructure ownership, keep the object outside the shared state, or use
a safe sibling API. A few more lines of safe code beat a shorter `unsafe`
block whose soundness rests on a comment. Reach for `unsafe` only when no safe
design exists, and record in the module documentation why none does.

The reverse also holds: **never give up functionality to avoid `unsafe`.**
Look hard for the safe design first (a safe library API, a different
ownership arrangement, a round trip through a component that already does
the unsafe work soundly) and use it when one exists. When none does, write the
smallest `unsafe` block that makes the feature possible, with its
`// SAFETY:` proof, rather than leaving the feature out.

---

## 12. Lint, format, and policy configuration

### `Cargo.toml`

Applications also optimize their release builds as a whole program. This makes
`cargo build --release` slower and the binary smaller and faster. Cargo ignores
a library's profiles when it is someone else's dependency, so libraries omit
this table.

```toml
[profile.release]
lto = true          # optimize across crate boundaries
codegen-units = 1   # optimize the crate as one unit
```

```toml
[lints.rust]
missing_docs = "warn"
unsafe_code = "forbid"
unreachable_pub = "warn"
unused_qualifications = "warn"
missing_debug_implementations = "warn"   # M-STATIC-VERIFICATION set:
ambiguous_negative_literals = "warn"
redundant_imports = "warn"
redundant_lifetimes = "warn"
trivial_numeric_casts = "warn"
unsafe_op_in_unsafe_fn = "warn"
unused_lifetimes = "warn"

[lints.rustdoc]
broken_intra_doc_links = "deny"
private_intra_doc_links = "warn"
missing_crate_level_docs = "warn"

[lints.clippy]
# Every major group, at warn; individual lints below override (priority).
cargo = { level = "warn", priority = -1 }
complexity = { level = "warn", priority = -1 }
correctness = { level = "warn", priority = -1 }
pedantic = { level = "warn", priority = -1 }
perf = { level = "warn", priority = -1 }
style = { level = "warn", priority = -1 }
suspicious = { level = "warn", priority = -1 }

# Selected `restriction` lints: each one enforces a rule in these standards.
allow_attributes = "warn"                  # use #[expect], never #[allow] (V4)
allow_attributes_without_reason = "warn"   # every override states why (V4)
min_ident_chars = "warn"                   # whole-word names (N1, N3)
missing_docs_in_private_items = "warn"     # docs for everything (R-D1); gates only under --docs
mod_module_files = "warn"                  # name.rs + name/, never mod.rs (R-T3)
unwrap_used = "warn"                       # expect-with-reason or ? (R-E5)
wildcard_enum_match_arm = "warn"           # exhaustive matches (E7, R-M11)
map_err_ignore = "warn"                    # preserve causes (E5)
unused_result_ok = "warn"                  # no silent discards (R-E7)
undocumented_unsafe_blocks = "warn"        # SAFETY comments (R-X2)
print_stdout = "warn"                      # expected only at the CLI output layer
print_stderr = "warn"
dbg_macro = "warn"
todo = "warn"
unimplemented = "warn"
clone_on_ref_ptr = "warn"                  # Arc::clone(&x) makes sharing visible
if_then_some_else_none = "warn"
renamed_function_params = "warn"
empty_structs_with_brackets = "warn"

# Opt-outs, each with its reason.
multiple_crate_versions = "allow"          # transitive duplicates are not ours to fix
```

Groups have `priority = -1`, so the individual lines after them win. Overrides
inside code use `#[expect(lint, reason = "…")]`, which warns when the
suppression stops being needed (M-LINT-OVERRIDE-EXPECT, General V4).

### `clippy.toml`

```toml
# Single-character names allowed by convention (General N3). This is clippy's
# default list, written out so it is visible; setting the key replaces the
# default rather than extending it. If clippy flags generic parameters, add the
# conventional ones here ("T", "K", "V", …).
allowed-idents-below-min-chars = ["i", "j", "n", "x", "y", "z", "w"]
too-many-arguments-threshold = 5           # more means a missing type (F3)
max-fn-params-bools = 1                    # boolean traps (F2, C-CUSTOM-TYPE)
allow-unwrap-in-tests = true
allow-expect-in-tests = true
allow-print-in-tests = true
doc-valid-idents = ["..", "PulseAudio", "XEmbed"]   # extend per project
# Trait methods whose standard parameter names are single letters (`f` in
# `fmt`), so implementations may use whole words (N3) without tripping
# `renamed_function_params`. ".." keeps clippy's own defaults.
allow-renamed-params-for = ["..", "core::fmt::Display", "core::fmt::Debug"]
```

### `rustfmt.toml`

rustfmt is Rust's black: the official formatter, installed with the toolchain,
run as `cargo fmt`, and deliberately short on options. Nearly every Rust project
uses it unconfigured, so the file stays small and only states choices worth
seeing. Every option below is available on stable Rust.

```toml
edition = "2024"
style_edition = "2024"
max_width = 100              # rustfmt default
use_field_init_shorthand = true
use_try_shorthand = true
newline_style = "Unix"
```

### `deny.toml` (minimal)

```toml
[licenses]
allow = ["MIT", "Apache-2.0", "BSD-2-Clause", "BSD-3-Clause", "ISC",
         "Unicode-3.0", "Unlicense", "Zlib"]
unused-allowed-license = "allow"   # the list is policy, not an inventory

[advisories]
yanked = "deny"

[bans]
wildcards = "deny"           # no `version = "*"` dependencies
```

---

## 13. Worked example: `av-output-switcher`'s model, in Rust

This example ports `av_output_switcher/model.py` and `gamepad-overlay`'s
`config.py` to show the standards working together. Notice where Rust lets the
design improve on the Python, not just transliterate it (M-RUST-SHAPED).

### `src/screen.rs`

```rust
//! The display layout as the rest of the tool sees it.
//!
//! Backend-neutral value types: what a monitor can do ([`Monitor`]), how an
//! output is configured ([`Configuration`]), and the whole layout
//! ([`Screen`]). Backends such as [`crate::wlr_randr`] produce these values;
//! profiles and the selector only ever read them.

use std::collections::BTreeSet;
use std::fmt;

/// A point in the compositor's global coordinate space, in pixels.
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash, Default)]
pub struct Position {
    /// Distance from the left edge of the layout.
    pub x: i32,
    /// Distance from the top edge of the layout.
    pub y: i32,
}

impl fmt::Display for Position {
    fn fmt(&self, formatter: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(formatter, "{:+}{:+}", self.x, self.y)
    }
}

/// The size of a display mode, in pixels.
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct Resolution {
    /// Horizontal size.
    pub width: u32,
    /// Vertical size.
    pub height: u32,
}

impl fmt::Display for Resolution {
    fn fmt(&self, formatter: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(formatter, "{}x{}", self.width, self.height)
    }
}

/// A refresh rate, stored exactly in thousandths of a hertz.
///
/// Compositors report rates such as 59.951 Hz. Storing millihertz keeps them
/// exact, totally ordered, and hashable, which a float would not be; the
/// Python version needed `Decimal` for the same reason.
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct RefreshRate {
    /// The rate in thousandths of a hertz.
    millihertz: u32,
}

impl RefreshRate {
    /// Thousandths of a hertz in one hertz.
    const MILLIHERTZ_PER_HERTZ: u32 = 1_000;
    /// Thousandths of a hertz in one hundredth of a hertz.
    const MILLIHERTZ_PER_HUNDREDTH: u32 = 10;

    /// Creates a refresh rate from thousandths of a hertz.
    #[must_use]
    pub const fn from_millihertz(millihertz: u32) -> Self {
        Self { millihertz }
    }

    /// Returns the rate in thousandths of a hertz.
    #[must_use]
    pub const fn millihertz(self) -> u32 {
        self.millihertz
    }
}

impl fmt::Display for RefreshRate {
    /// Formats as hertz truncated to hundredths, such as `59.95hz`.
    fn fmt(&self, formatter: &mut fmt::Formatter<'_>) -> fmt::Result {
        let whole_hertz = self.millihertz / Self::MILLIHERTZ_PER_HERTZ;
        let hundredths =
            self.millihertz % Self::MILLIHERTZ_PER_HERTZ / Self::MILLIHERTZ_PER_HUNDREDTH;
        write!(formatter, "{whole_hertz}.{hundredths:02}hz")
    }
}

/// A resolution paired with its refresh rate, when the backend reports one.
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct Mode {
    /// The mode's size.
    pub resolution: Resolution,
    /// The mode's refresh rate, if known.
    pub refresh_rate: Option<RefreshRate>,
}

impl fmt::Display for Mode {
    fn fmt(&self, formatter: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self.refresh_rate {
            Some(refresh_rate) => write!(formatter, "{}@{refresh_rate}", self.resolution),
            None => write!(formatter, "{}", self.resolution),
        }
    }
}

/// What an attached monitor reports it can display.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Monitor {
    /// Every mode the monitor reports; ordered, so the best mode is last.
    pub reported_modes: BTreeSet<Mode>,
    /// The mode the monitor asks to be driven at, if it states one.
    pub preferred_mode: Option<Mode>,
    /// A stable description of the physical monitor (make, model, serial).
    pub identifier: String,
}

impl Monitor {
    /// Returns the preferred mode, or else the largest reported mode.
    ///
    /// Returns `None` only for a monitor that reports no modes at all.
    #[must_use]
    pub fn default_mode(&self) -> Option<Mode> {
        self.preferred_mode
            .or_else(|| self.reported_modes.last().copied())
    }
}

/// How an active output is currently driven.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct Configuration {
    /// The mode it is driven at.
    pub mode: Mode,
    /// Where its top-left corner sits in the layout.
    pub position: Position,
}

/// One connector of the session, and what is attached and configured on it.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Output {
    /// The connector name, such as `DP-1`.
    pub name: String,
    /// The attached monitor, or `None` when nothing is connected.
    pub monitor: Option<Monitor>,
    /// The current configuration, or `None` when the output is disabled.
    pub configuration: Option<Configuration>,
}

impl Output {
    /// Returns whether the output is currently showing part of the desktop.
    #[must_use]
    pub fn is_active(&self) -> bool {
        self.configuration.is_some()
    }
}

/// The outputs of the session and which one is primary.
///
/// The Python model stored a `primary` flag on every output and had to raise
/// at runtime when two were set. Here the primary output is a single index,
/// so "two primaries" cannot be represented at all (General P4).
#[derive(Debug, Clone, PartialEq, Eq, Default)]
pub struct Screen {
    /// Every output, in the order the backend reported them.
    outputs: Vec<Output>,
    /// Which of `outputs` is primary, if one was chosen.
    primary_index: Option<usize>,
}

impl Screen {
    /// Creates a screen from the outputs a backend reported, in its order.
    #[must_use]
    pub fn new(outputs: Vec<Output>) -> Self {
        Self {
            outputs,
            primary_index: None,
        }
    }

    /// Returns every output, connected or not, in backend order.
    #[must_use]
    pub fn outputs(&self) -> &[Output] {
        &self.outputs
    }

    /// Returns the outputs currently showing part of the desktop.
    pub fn active_outputs(&self) -> impl Iterator<Item = &Output> {
        self.outputs.iter().filter(|output| output.is_active())
    }

    /// Returns the primary output, falling back to the first active output.
    ///
    /// Returns `None` only when no output is active.
    #[must_use]
    pub fn primary_output(&self) -> Option<&Output> {
        self.primary_index
            .and_then(|index| self.outputs.get(index))
            .or_else(|| self.active_outputs().next())
    }

    /// Returns this screen with the output named `name` as primary.
    ///
    /// If that output is not active, the first active output becomes primary
    /// instead; when no output is active, none is.
    #[must_use]
    pub fn with_primary(mut self, name: &str) -> Self {
        self.primary_index = self
            .outputs
            .iter()
            .position(|output| output.is_active() && output.name == name)
            .or_else(|| self.outputs.iter().position(Output::is_active));
        self
    }
}
```

### `src/config.rs`

This is the shared-first-argument refactoring (General M2) in Rust:

```rust
//! The application's configuration file: one JSON object of named sections.
//!
//! Each top-level key is an independent section (`selection`, `server`,
//! `overlay`) owned by a different part of the program. [`ConfigFile`] reads
//! and replaces one section at a time and preserves the others, so no part of
//! the program needs to know the shape of the rest of the file. A missing file
//! behaves like an empty one; a corrupt file is an error, not silently
//! discarded.

use std::fs;
use std::io;
use std::path::{Path, PathBuf};

use serde::Serialize;
use serde::de::DeserializeOwned;
use serde_json::{Map, Value};

/// The configuration file's name inside the configuration directory.
pub const CONFIG_FILE_NAME: &str = "config.json";

/// A JSON configuration file made of independent, named sections.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct ConfigFile {
    /// Where the file lives; it need not exist yet.
    path: PathBuf,
}

impl ConfigFile {
    /// Creates a handle for the file at `path`; nothing is read until asked.
    #[must_use]
    pub fn new(path: impl Into<PathBuf>) -> Self {
        Self { path: path.into() }
    }

    /// Returns the location of the file.
    #[must_use]
    pub fn path(&self) -> &Path {
        &self.path
    }

    /// Reads the section `name`, or `None` if the file or section is absent.
    ///
    /// # Errors
    ///
    /// Returns an error if the file cannot be read, is not a JSON object, or
    /// the section does not have the shape of `T`.
    pub fn section<T: DeserializeOwned>(&self, name: &str) -> Result<Option<T>, ConfigError> {
        let mut document = self.read_document()?;
        document
            .remove(name)
            .map(|value| {
                serde_json::from_value(value).map_err(|source| ConfigError::InvalidSection {
                    path: self.path.clone(),
                    section_name: name.to_owned(),
                    source,
                })
            })
            .transpose()
    }

    /// Replaces the section `name`, preserving every other section.
    ///
    /// # Errors
    ///
    /// Returns an error if the existing file cannot be read, `section` cannot
    /// be represented as JSON, or the file cannot be written.
    pub fn replace_section<T: Serialize>(
        &self,
        name: &str,
        section: &T,
    ) -> Result<(), ConfigError> {
        let value = serde_json::to_value(section).map_err(|source| ConfigError::Encode {
            section_name: name.to_owned(),
            source,
        })?;
        let mut document = self.read_document()?;
        document.insert(name.to_owned(), value);
        self.write_document(&document)
    }

    /// Removes the section `name` if present, preserving every other section.
    ///
    /// # Errors
    ///
    /// Returns an error if the file cannot be read or written.
    pub fn remove_section(&self, name: &str) -> Result<(), ConfigError> {
        let mut document = self.read_document()?;
        if document.remove(name).is_some() {
            self.write_document(&document)?;
        }
        Ok(())
    }

    /// Reads the whole file as a JSON object; a missing file is an empty one.
    fn read_document(&self) -> Result<Map<String, Value>, ConfigError> {
        let text = match fs::read_to_string(&self.path) {
            Ok(text) => text,
            Err(error) if error.kind() == io::ErrorKind::NotFound => return Ok(Map::new()),
            Err(source) => {
                return Err(ConfigError::Read {
                    path: self.path.clone(),
                    source,
                });
            }
        };
        match serde_json::from_str::<Value>(&text) {
            Ok(Value::Object(document)) => Ok(document),
            Ok(_) => Err(ConfigError::NotAnObject {
                path: self.path.clone(),
            }),
            Err(source) => Err(ConfigError::Parse {
                path: self.path.clone(),
                source,
            }),
        }
    }

    /// Writes `document` as the whole file, creating its directory if needed.
    fn write_document(&self, document: &Map<String, Value>) -> Result<(), ConfigError> {
        let write_error = |source| ConfigError::Write {
            path: self.path.clone(),
            source,
        };
        if let Some(directory) = self.path.parent() {
            fs::create_dir_all(directory).map_err(write_error)?;
        }
        let mut text = serde_json::to_string_pretty(document)
            .expect("a JSON object with string keys always serializes");
        text.push('\n');
        fs::write(&self.path, text).map_err(write_error)
    }
}

/// Why a configuration file could not be read or written.
#[derive(Debug, thiserror::Error)]
pub enum ConfigError {
    /// The file exists but could not be read.
    #[error("could not read configuration file {}", .path.display())]
    Read {
        /// The file's location.
        path: PathBuf,
        /// What the operating system reported.
        source: io::Error,
    },

    /// The file, or its directory, could not be written.
    #[error("could not write configuration file {}", .path.display())]
    Write {
        /// The file's location.
        path: PathBuf,
        /// What the operating system reported.
        source: io::Error,
    },

    /// The file is not valid JSON.
    #[error("configuration file {} is not valid JSON", .path.display())]
    Parse {
        /// The file's location.
        path: PathBuf,
        /// Where and why parsing failed.
        source: serde_json::Error,
    },

    /// The file is valid JSON, but its top level is not an object.
    #[error("configuration file {} does not contain a JSON object", .path.display())]
    NotAnObject {
        /// The file's location.
        path: PathBuf,
    },

    /// A section exists but does not have the expected shape.
    #[error("section `{section_name}` of {} has an unexpected shape", .path.display())]
    InvalidSection {
        /// The file's location.
        path: PathBuf,
        /// The section that could not be read.
        section_name: String,
        /// How the section differs from the expected shape.
        source: serde_json::Error,
    },

    /// A section's new value could not be represented as JSON.
    #[error("section `{section_name}` could not be encoded as JSON")]
    Encode {
        /// The section being replaced.
        section_name: String,
        /// Why the value could not be encoded.
        source: serde_json::Error,
    },
}
```

Callers now read as the domain does:

```rust
let config_file = ConfigFile::new(project_directories().config_dir().join(CONFIG_FILE_NAME));
let selection: Option<GamepadSelection> = config_file.section("selection")?;
config_file.replace_section("server", &server_settings)?;
```

---

## Appendix: Rust review checklist (in addition to General Appendix A)

- [ ] Does every function with a clear receiver live in that type's `impl`?
      (R-M17)
- [ ] Are fields private wherever the type has an invariant? (R-M3)
- [ ] Do newtypes have fallible constructors? (R-M7)
- [ ] Are matches exhaustive, with no `_ =>` over our own enums? (R-M11)
- [ ] Does each signature borrow what it reads and own what it keeps? (R-O1)
- [ ] Are there no `unwrap()` calls outside tests, and does every `expect`
      state why it cannot fail? (R-E5)
- [ ] Do public functions document `# Errors` and `# Panics`? (R-E6)
- [ ] Does `cargo doc --document-private-items` build cleanly, with every link
      resolving? (R-D1, R-D3)
- [ ] Does every suppression use `#[expect(…, reason = "…")]`? (§12)
