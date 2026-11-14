# Engineering Standards — Python

*Refines [GENERAL.md](GENERAL.md). Rule IDs such as `General M2` refer to that
document. This standard is a codification of the practices already established
across the production projects (`sevaht-utility`, `av-output-switcher`,
`dat-file-filter`, `gamepad-overlay`, the `python-project` cookiecutter
template). Where it adds something new, it says so.*

**Baselines:** [PEP 8](https://peps.python.org/pep-0008/) (style),
[PEP 257](https://peps.python.org/pep-0257/) (docstrings),
[PEP 484](https://peps.python.org/pep-0484/) /
[PEP 695](https://peps.python.org/pep-0695/) (typing), and the
[Google docstring format](https://google.github.io/styleguide/pyguide.html#38-comments-and-docstrings).
Where this document is silent, those apply.

---

## 1. Toolchain

| Concern | Tool | Configuration |
|---|---|---|
| Environment and running | **uv** | `uv run …`; `uv.lock` committed for applications |
| Build backend | **hatchling** + **hatch-vcs** | Version from git tags; `fallback-version` for fresh trees |
| Formatting | **black**, moving to **`ruff format`** (PY-V4) | `line-length = 79`, `skip-magic-trailing-comma = true` |
| Import sorting | **ruff** (`I`) | Run as a separate step before linting |
| Linting | **ruff** | Broad `select` (see §11); opt-outs are explicit |
| Type checking | **mypy** | `strict = true`, plus the extra error codes in §11 |
| Dead code | **vulture** | `min_confidence = 70` for libraries |
| Tests | **pytest** | `testpaths = ["tests"]` |
| API docs | **Sphinx** + **furo** + **myst-parser** + **napoleon** | Google-style docstrings, autodoc |
| Single entry point | **`./checks`** | Runs all of the above in order (General V6) |

**PY-T1.** New projects start from the `sevaht-python-project` cookiecutter
template and stay structurally in sync with it. When a convention changes, the
template changes first.

**PY-T2.** The minimum supported Python version is declared in
`requires-python` and is currently **3.12**. Use language features available at
that floor: PEP 695 `type` aliases, `typing.override`, generic class syntax, and
`match`.

---

## 2. Project layout

```
project-name/
├── pyproject.toml
├── README.md            # user-facing guide (General D10)
├── DESIGN.md            # maintainer/agent-facing rationale (optional, recommended)
├── checks               # the single verification entry point
├── src/project_name/
│   ├── __init__.py      # package docstring; version; minimal re-exports
│   ├── __main__.py      # `raise SystemExit(log_exceptions()(main)())`, nothing else
│   ├── application.py   # CLI: parse, configure, dispatch, map errors to exit codes
│   ├── py.typed
│   └── <concept>.py     # one module per concept (General O1)
├── tests/
│   └── test_<concept>.py
└── docs/                # Sphinx sources (libraries)
```

**PY-L1.** Use the `src/` layout. The import package name is the distribution
name with hyphens turned into underscores.

**PY-L2.** `application.py` is the top of the dependency graph (General M13). It
contains `main()` and its private argument-parsing and dispatch helpers. It holds
no domain types, and no other module imports from it. Shared identity (the
application name, the organization name, platform directories) lives in the
package `__init__.py` or a dedicated module that both `application.py` and the
rest of the package import from.

**PY-L3.** Underscore-prefixed modules (`_tray_sni.py`) are backend
implementations hidden behind a public interface. A module that is meant to be
imported by users is not underscore-prefixed (General N13).

---

## 3. Module anatomy

Every module follows this order (General O5):

```python
"""One-line summary of the concept this module models.

A paragraph describing the types it defines, how they relate, and when to use
them. Refer to items with Sphinx roles: :class:`Screen`, :func:`convert_name`.
"""

from __future__ import annotations

import logging                                   # 1. standard library
from dataclasses import dataclass
from typing import TYPE_CHECKING

from sevaht_utility.parsing import json5_load    # 2. third-party

from .cli_tool import CLITool                    # 3. this package (relative)

if TYPE_CHECKING:                                # 4. type-only imports
    from collections.abc import Iterator, Sequence

    from .model import Screen

logger = logging.getLogger(__name__)             # 5. module logger

CONFIG_FILE_NAME = "config.json"                 # 6. public constants
_POLL_INTERVAL_SECONDS = 5                       #    private constants

type ColumnIndex = int                           # 7. type aliases


class ...                                        # 8. public types, most important first
def ...                                          # 9. public functions (rare; General M15)
def _...                                         # 10. private helpers
```

**PY-M1.** Every module starts with a docstring (General D4).

**PY-M2.** Every module starts with `from __future__ import annotations` while
`requires-python` is below 3.14. Imports used only in annotations go under
`if TYPE_CHECKING:` (ruff `TC`). *Revisit when the floor reaches 3.14. PEP 649
deferred evaluation makes the `__future__` import unnecessary from that version
on.*

**PY-M3.** Use relative imports within a package (`from .model import Screen`)
and absolute imports for everything else. Never use wildcard imports.

**PY-M4.** Import modules or names, but be consistent within a file. Use the
`from x import Name` form for types used throughout a module. Use the
`import x` form when the qualifier aids clarity (`subprocess.run`, `json.loads`).

---

## 4. Naming

General §2 applies in full. Python-specific casing:

| Item | Convention | Example |
|---|---|---|
| Packages, modules | `snake_case`, short but whole words | `wlr_randr`, `log_utility` |
| Classes, type aliases, type variables | `PascalCase` | `ProfileSelector`, `TextProvider` |
| Functions, methods, variables, parameters | `snake_case` | `apply_profile`, `column_count` |
| Constants (module or class level) | `UPPER_SNAKE_CASE` | `CONFIG_FILE_NAME`, `_STRUCT` |
| Enum members | `UPPER_SNAKE_CASE` | `NameStyle.SNAKE_CASE` |
| Exceptions | `PascalCase`, `Error` suffix | `ShortRowError` |
| Internal (module- or class-private) | Leading `_` | `_resolve_profile`, `_heap` |

**PY-N1.** An acronym in a `PascalCase` name is written in all capitals:
`CLITool`, `PACtl`, `SNIRegistrationError`. This is PEP 8's rule ("When using
acronyms in CapWords, capitalize all the letters of the acronym. Thus
HTTPServerError is better than HttpServerError."), and the standard library
follows it (`HTTPServer`, `JSONDecodeError`). Rust's opposite convention
(`CliTool`) is Rust's own; each language follows its own naming rules. In `snake_case`, acronyms are
lowercase (`cli_tool`, `sni_watcher_present`).

**PY-N2.** A single leading underscore marks module- or class-internal items. Do
not use double-underscore name mangling. It obscures debugging and offers no real
protection. The exception is a deliberate collision-avoidance case, which must be
commented.

**PY-N3.** Mappings are named `key_to_value` (`name_to_data`,
`fmg_to_entry_to_text`) or `value_by_key` (`LANGUAGE_BY_VALUE`,
`sinks_by_card`), whichever reads best in context (General N7). Nested mappings
chain the pattern.

---

## 5. Modeling toolkit

This section is the Python vocabulary for General §3.

### 5.1 Dataclasses are the default type

**PY-D1.** Domain types are `@dataclass`es unless they need custom `__init__`
logic that cannot be expressed with `__post_init__` or a named constructor.

| Role (General M4) | Decorator |
|---|---|
| Value object | `@dataclass(frozen=True, slots=True)`, plus `order=True` when ordering is meaningful |
| Value object using `cached_property` | `@dataclass(frozen=True)`. **`slots=True` is incompatible with `cached_property`**, which needs `__dict__` |
| Entity, coordinator, gateway | `@dataclass` (mutable) with `field(init=False)` for derived state |
| Options or parameter object | `@dataclass(frozen=True, kw_only=True)` |

**PY-D2.** Use `kw_only=True` (or a `_: KW_ONLY` marker) whenever a dataclass has
several fields of the same type, or more than three fields. Positional
construction of `BatteryStatus("BAT0", 85, True, "Charging", delta)` is a
reading hazard.

**PY-D3.** Mutable defaults always use `field(default_factory=…)`. Collaborators
use `field(default_factory=Collaborator)`, so tests can inject replacements:

```python
@dataclass
class ProfileSelector:
    settings: Settings
    pactl: PACtl = field(default_factory=PACtl)
    randr: WlrRandr = field(default_factory=WlrRandr)
```

**PY-D4.** Class-level constants are `ClassVar`s on the class that uses them
(`_STRUCT: ClassVar[struct.Struct]`). Immutable lookup tables are wrapped in
`types.MappingProxyType` or held in a `frozenset`/`tuple` (General K4).

### 5.2 Construction

**PY-C1.** Alternative constructors are `@classmethod`s named `from_<source>`
(`from_json`, `from_stem`, `from_data`) or `for_<purpose>`
(`BatteryLevel.for_capacity`) (General M7). They return `Self` (or the class
name when `from __future__ import annotations` is active).

**PY-C2.** Validate in `__post_init__` or in the named constructor, and raise a
specific exception (General M6). After construction, the object is valid.

**PY-C3.** A shared, lazily-created resource (such as the `PlatformDirs` object
for this application) is a public, `@functools.cache`-decorated function that
returns it. Do not use a module-level global that is assigned later, and do not
use a hand-rolled singleton class.

```python
@cache
def platform_directories() -> PlatformDirs:
    """The per-user directories for this application, on every platform."""
    return PlatformDirs(appname=APPLICATION_NAME, appauthor=ORGANIZATION_NAME)
```

### 5.3 Behavior on the type

**PY-B1.** Use `@property` for cheap, side-effect-free derived values
(`BatteryStatus.level`). Use `@cached_property` for expensive derived values of
immutable objects (`Monitor.sorted_modes`). Never use a property that performs
I/O or can fail in ordinary use. That is a method.

**PY-B2.** Value objects implement the protocol methods that make them natural.
`__str__` gives a human-readable form. `__bool__` answers "has any information,"
when that is a meaningful question. Arithmetic or comparison operators are added
when the domain has that algebra: `Metrics.__add__`, `Solution.__lt__`, and
`@total_ordering`. Operators must behave unsurprisingly. If you would have to
explain what `+` means, use a named method instead.

**PY-B3.** Enumerations carry their behavior (General M8). Use `StrEnum` when the
member's value is its external text, and `@unique` always. When each member
needs associated data, attach it through `__new__` (the pattern used in
`NameStyle`) or a property that consults a `MappingProxyType` keyed by member.
Do not maintain a parallel dictionary in another module.

**PY-B4.** Methods that produce modified copies of immutable values are named
`with_<change>` and use `dataclasses.replace` (`Screen.with_primary(name)`).

### 5.4 Interfaces and polymorphism

**PY-I1.** An interface is a `typing.Protocol` (structural typing). Implementations
do not inherit from it. Static type checking verifies conformance at the point
where an implementation is returned as the protocol type (General M11).

**PY-I2.** Use an `abc.ABC` only when implementations share real code (a
template method). Keep hierarchies shallow (General M10).

**PY-I3.** Overriding methods are decorated with `@typing.override`. *(New.)*
This turns a misspelled override into a type error.

**PY-I4.** The backend is chosen in one factory function (`create_tray_icon`)
that returns the protocol type. Callers never import a concrete backend.

### 5.5 Applying the free-function tests in Python

Two refactorings illustrate General M2 and M3 using code from these projects.

**Shared subject → class** (from `gamepad-overlay/config.py`):

```python
# Before: five functions, all taking `config_path` first.
def read_config(config_path: Path) -> dict[str, object]: ...
def write_config(config_path: Path, config: dict[str, object]) -> None: ...
def read_section(config_path: Path, section_name: str) -> dict[str, object]: ...
def update_section(config_path: Path, section_name: str, ...) -> None: ...
def remove_section(config_path: Path, section_name: str) -> None: ...
```

```python
# After: the path is the object's identity; the operations are its methods.
@dataclass(frozen=True)
class ConfigFile:
    """A JSON configuration file made of independent, named sections.

    A missing or unreadable file behaves like an empty configuration, so the
    first write creates it.
    """

    path: Path

    def read(self) -> dict[str, object]: ...
    def write(self, document: Mapping[str, object]) -> None: ...
    def section(self, section_name: str) -> dict[str, object]: ...
    def replace_section(
        self, section_name: str, section: Mapping[str, object]
    ) -> None: ...
    def remove_section(self, section_name: str) -> None: ...
```

**Travelling parameters → value object** (from `gamepad_selector.py`):

```python
# Before: three helpers, each taking the same (attached, client_count) pair.
def _status_detail(*, attached: bool, client_count: int) -> str: ...
def _status_text(*, attached: bool, client_count: int) -> str: ...
def _status_tooltip(*, attached: bool, client_count: int) -> str: ...
```

```python
# After: the pair is a concept, with three presentations.
@dataclass(frozen=True, slots=True, kw_only=True)
class ConnectionStatus:
    """Whether a gamepad is attached and how many overlay clients listen."""

    is_gamepad_attached: bool
    client_count: int

    @property
    def detail(self) -> str: ...
    @property
    def text(self) -> str: ...
    @property
    def tooltip(self) -> str: ...
```

The same reasoning applies to `battery.py`. `_read_text(path)`,
`_read_int(path)`, `_time_remaining(base)`, and `read_battery(name, root=)` all
revolve around one power-supply directory. They want to be a `PowerSupply`
gateway with a `status()` method returning `BatteryStatus`, and `status_lines`
wants to be `BatteryStatus.lines()`.

---

## 6. Functions

**PY-F1.** Parameters after the first one or two are keyword-only (`*`).
Boolean and optional "mode" parameters are *always* keyword-only (General F2;
ruff `FBT`):

```python
def invoke(
    self, arguments: list[str], *, capture_output: bool = True
) -> str: ...
```

**PY-F2.** When a keyword argument changes behavior (as opposed to tuning it), it
has no default (General F2). For example, `extract(..., comments=...)` requires
callers to choose.

**PY-F3.** Annotate parameters with abstract collection types from
`collections.abc` (`Sequence`, `Mapping`, `Iterable`) and annotate return values
with concrete ones (`list`, `dict`, `tuple`) (General F6). Use `Path` for paths,
never `str`. Accept `Path | str` only at public library boundaries, and convert
immediately.

**PY-F4.** Prefer returning `None` for "absent" over sentinel values such as
`""` or `-1` (General F5). *(Existing code often returns `""` for "no profile."
New code should return `str | None`, and old code should migrate when touched.)*

**PY-F5.** Use generators (`Iterator[T]`) for streams. Return materialized
collections when callers need `len()`, indexing, or repeated iteration.

---

## 7. Typing

**PY-Y1.** `mypy --strict` passes on the package, the tests, and the `checks`
script (`--scripts-are-modules`). Every function is fully annotated, including
`-> None`.

**PY-Y2.** Use `Any` only where data is genuinely untyped (parsed JSON or
reflection) and only until it is validated into a domain type at the boundary
(General M6). Use `object` instead of `Any` when you only need "something."

**PY-Y3.** Never write a bare `# type: ignore`. It always names the error code
(`# type: ignore[no-any-return]`) and is a last resort. For an untyped
dependency, prefer a small local stub covering the subset you actually use,
under `typings/`, over ignoring the module (General V4).

**PY-Y4.** Name compound types with `type` aliases (PEP 695):
`type TextProvider = str | Path | TextIO | list[str]`. Use `typing.NewType` for
distinct identifiers that share a representation (General M9). Use `Literal`
for small sets of literal values in signatures, but prefer an `Enum` when the
set is a domain concept.

**PY-Y5.** Keep `cast()` rare and local, with a comment when the reason is not
obvious.

---

## 8. Errors and logging

**PY-E1.** Each failure mode that callers might distinguish is its own exception
class, deriving from the most fitting built-in (`ValueError`, `LookupError`,
`RuntimeError`, `TypeError`). The constructor takes structured, keyword-only
arguments, stores them as attributes, and composes the message (General E2):

```python
class ColumnIndexOutOfRangeError(ValueError):
    """A mapped column index lies outside the row's columns."""

    def __init__(
        self, *, field_name: str, column_index: int, column_count: int
    ) -> None:
        super().__init__(
            f"Column index out of range for field {field_name!r}: "
            f"{column_index} (column count: {column_count})"
        )
        self.field_name = field_name
        self.column_index = column_index
        self.column_count = column_count
```

**PY-E2.** For one-off raises of built-in exceptions, assign the message first:
`msg = f"…"; raise ValueError(msg)` (ruff `EM`).

**PY-E3.** When translating an exception, always chain it: `raise DomainError(…)
from exc` (General E5). Use `except Exception` only with a `# noqa: BLE001` and a
comment explaining why the callee is unknowable (General E4).

**PY-E4.** Use `assert` only in tests. A broken invariant in production code
raises `AssertionError` (or a specific error) explicitly, with a message, because
`assert` disappears under `-O`.

**PY-E5.** Every module that logs defines `logger = logging.getLogger(__name__)`.
Libraries never configure logging. Applications configure it once in `main()`
through `sevaht_utility.log_utility`. Use `print` only for a CLI's actual output
(the data the user asked for), never for diagnostics.

**PY-E6.** The entry point maps expected failures to a terse error log and a
non-zero exit code, with no traceback (General E3, E6). Unexpected exceptions
propagate to `log_exceptions`.

**PY-E7. Log messages pass their values as arguments, not f-strings:**
`logger.info("applying profile: %s", name)`, never
`logger.info(f"applying profile: {name}")`. Ruff's `G` rules enforce this
(`G004` forbids the f-string). The reasons:

1. **Disabled levels cost nothing.** `logging` formats the message only if a
   handler will emit it; an f-string is formatted before the call, always.
2. **The message template stays constant.** Every record keeps the fixed
   template (`record.msg`) separate from its values (`record.args`), so filters,
   handlers, and any future log aggregator can group "the same message" across
   calls.
3. **A failure to format does not crash the caller.** If a value's `__str__`
   raises, `logging` reports the error and carries on; inside an f-string, the
   exception escapes into the code that was only trying to log.

Use `%s` for display and `%r` where the exact value matters (`%r` shows quotes
around strings, making empty or whitespace values visible). This is the one
place `%` formatting is used; everywhere else, f-strings remain the standard.
For work more expensive than formatting, guard it with
`if logger.isEnabledFor(logging.DEBUG):`.

---

## 9. Command-line applications

**PY-A1.** `main(argv: Sequence[str] | None = None) -> int` is the entry point.
`__main__.py` contains only `raise SystemExit(log_exceptions()(main)())`.
`[project.scripts]` points at `package.__main__:main`.

**PY-A2.** Use `argparse`. The parser is built in a private `_build_parser()`.
The description comes from package metadata
(`importlib.metadata.metadata(__package__)["summary"]`). Logging flags come from
`add_log_arguments`.

**PY-A3.** Arguments that select a mode go in mutually exclusive groups.
Configuration-changing flags (select a gamepad, set a poll interval) update the
saved configuration and return without running the application. Their names
make that clear (General F8).

**PY-A4.** `main` dispatches to small private functions or to coordinator
methods. It does not contain the logic itself (General O4).

---

## 10. Documentation

**PY-DOC1.** Docstrings use the Google format, rendered by Sphinx with
`napoleon`. Summary lines use the imperative mood (PEP 257: *"Return the
current status."*, not *"Returns…"*), fit on one line, and end with a period.

```python
def convert_name(name: str, *, style: NameStyle) -> str:
    """Convert a name from any supported style into ``style``.

    This is :func:`split_into_words` followed by :func:`join_words`. Because
    splitting normalizes to lowercase words, conversion is idempotent.

    Args:
        name: The identifier to convert. May use any supported style.
        style: The target :class:`NameStyle`.

    Returns:
        ``name`` rendered in ``style``.

    Raises:
        ValueError: If ``name`` contains characters that cannot appear in an
            identifier.

    Example:
        >>> convert_name("someSampleName", style=NameStyle.SNAKE_CASE)
        'some_sample_name'
    """
```

**PY-DOC2.** Public modules, classes, functions, and methods have docstrings.
`Args:` and `Returns:` are required when the signature alone does not make
meaning obvious. `Raises:` is required whenever the function raises
intentionally. Dataclass fields are documented in an `Attributes:` section of the
class docstring.

**PY-DOC3.** Cross-reference with Sphinx roles (`:class:`, `:func:`, `:meth:`,
`:data:`) so the generated documentation links correctly and IDE tooltips stay
readable.

**PY-DOC4.** `Example:` sections are doctests and are executed by
`pytest --doctest-modules` (General D5). *(New: add `--doctest-modules` to the
pytest options in libraries.)*

**PY-DOC5.** Libraries publish Sphinx HTML (furo theme) to GitHub Pages through
the template's `docs.yml` workflow. Narrative pages, such as a quick-start or a
hello-world tutorial, are Markdown via MyST in `docs/`.

**PY-DOC6.** Comment style: complete sentences, the *why* not the *what*
(General D6), and the same 79-column limit.

---

## 11. Lint and type-check configuration

The template's configuration stands. The additions marked **new** close gaps that
the standards above rely on.

```toml
[tool.black]                    # until the move to `ruff format` (PY-V4)
line-length = 79
skip-magic-trailing-comma = true

[tool.ruff]
line-length = 79

[tool.ruff.lint]
select = [
    "A", "ANN", "ARG", "ASYNC", "B", "BLE", "C4", "C90", "COM",
    "DTZ", "E", "EM", "ERA", "F", "FA",
    "FBT",  # new: boolean-trap parameters (General F2)
    "FLY", "FURB",
    "G",    # new: logging arguments, not f-strings (PY-E7)
    "I",
    "ISC", "LOG", "N", "PERF", "PGH", "PIE", "PLR", "PLW",
    "PT",   # new: pytest style
    "PTH", "Q", "RET", "RSE", "RUF", "S", "SIM",
    "SLF",  # new: no reaching into other objects' private members
    "TC", "TD", "TID", "TRY", "UP",
]
extend-ignore = ["COM812", "COM819", "E203", "TD003"]  # new: E501 enforced

[tool.ruff.lint.pydocstyle]
convention = "google"           # new: used by `./checks --docs` (PY-V3)

[tool.ruff.lint.per-file-ignores]
"checks" = ["S603"]
"tests/**/*" = ["PLR2004", "S101", "D", "SLF001"]

[tool.mypy]
strict = true
warn_unreachable = true         # new
enable_error_code = [           # new
    "ignore-without-code",
    "redundant-expr",
    "truthy-bool",
    "possibly-undefined",
]

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = ["--strict-markers", "--doctest-modules"]  # new
xfail_strict = true                                  # new
```

**PY-V1. `E501` (pycodestyle's "line too long") is enforced everywhere**
(General V2). The formatter already wraps code, but it never splits string
literals or comments, so without `E501` a long message or comment can run past
79 columns unnoticed. Split long strings with implicit concatenation inside
parentheses. A URL that cannot be split takes a `# noqa: E501`. *(New: the
template currently ignores `E501`; remove it from `extend-ignore` there.)*

**PY-V3. Docstring checks run only under `./checks --docs`** (General V7).
That mode runs `ruff check --extend-select D` (docstring presence and Google
format) and builds the Sphinx site with `-W` (warnings as errors). The regular
`./checks` does neither, so a missing docstring never blocks work.

**PY-V2.** A `# noqa` names the rule code and, unless the code makes the reason
self-evident, carries a short reason (`# noqa: S603 — arguments are fixed, not
user input`).

**PY-V4. `ruff format` replaces black** when a project next touches its
tooling. It is designed as a drop-in replacement for black (its documentation
reports over 99.9% of lines formatted identically on black-formatted projects),
and it removes
one tool from the chain. The equivalent configuration:

```toml
[tool.ruff]
line-length = 79                    # shared by the formatter and E501

[tool.ruff.format]
skip-magic-trailing-comma = true    # black's option of the same name
# Defaults already match black: double quotes, spaces, magic trailing commas
# otherwise respected, docstring code left alone.
```

The `checks` script then runs `ruff check --select I --fix` (import sorting),
`ruff format`, and `ruff check`, in that order. Delete `[tool.black]` and the
black dependency in the same change. `ruff format` warns if a selected lint rule
conflicts with the formatter (`COM812` and `ISC001` are the usual ones). Follow
the warning, and ignore the rule it names.

---

## 12. Testing

**PY-TEST1.** Use pytest functions, not `unittest.TestCase`. Test modules are
named `test_<module>.py`, mirroring the package.

**PY-TEST2.** Use fixtures for setup, `tmp_path` for file systems, and
`monkeypatch` or injected collaborators (PY-D3) for gateways. Tests never invoke
real external tools, networks, or hardware (General T4).

**PY-TEST3.** Use `@pytest.mark.parametrize` with `ids=` for case tables
(General T5).

**PY-TEST4.** Test names describe behavior:
`test_split_into_words_separates_medial_acronym`.

**PY-TEST5.** Every package keeps `test_package_version.py` (the template's
smoke test). It proves the package imports and the build metadata is intact.

---

## 13. Shell scripts accompanying Python projects

Short operational scripts (`publish-test.sh`, `build.sh`) follow these rules.

- Use `#!/usr/bin/env bash` and `set -euo pipefail`.
- Reference every variable as `${name}`. Quote expansions where word-splitting
  matters, and nowhere else. Do not quote plain literals needlessly.
- Use full-word variable names (General N1). `venv_minor` that actually
  holds major and minor versions is wrong twice.
- Use 80 columns, with a short header comment rather than a manifesto
  (General D7).
- Prefer `[[ … ]]` with `&&` chains over `continue` gymnastics, and prefer one
  clear compound condition over several partial ones.
