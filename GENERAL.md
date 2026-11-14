# Engineering Standards — General

*Language-independent principles. The language standards ([PYTHON.md](PYTHON.md),
[RUST.md](RUST.md)) refine these; they never contradict them silently. Where a
language standard deviates, it says so and says why.*

---

## 0. How to read this document

**Normative language.** **MUST** / **MUST NOT** are rules: breaking one needs a
written justification next to the code. **SHOULD** / **SHOULD NOT** are strong
defaults: deviate when the result is clearly better, and be ready to say why.
**MAY** marks an accepted option.

**Rule identifiers.** Rules carry short IDs (`N3`, `M1`, …) so reviews and
commit messages can cite them ("renamed per N3"). Language documents cite them as
`General N3`.

**Precedence.** Project-local documents (`DESIGN.md`, `CLAUDE.md`, `AGENTS.md`)
refine language standards, which refine this one. Upstream conventions of a code
base we are *modifying* (a fork, a disassembly, a third-party project) win over
all of these. Match the house style of the house you are in.

---

## 1. Principles

These are the reasons behind every rule that follows. When a rule and a
principle seem to disagree, the principle wins and the rule needs fixing.

**P1. Code is read far more often than it is written.** Optimize for the reader
who arrives cold, months later, without the author present. That reader may be
a human or an AI agent; both benefit from the same things.

**P2. Model the domain; let the code read like the problem.** The nouns of the
problem become types. The verbs become methods on those types. A reader who
understands the problem should recognize it in the code's structure before
reading a single function body.

**P3. Names are the first and most-read documentation.** A precise name removes
the need for a comment; a vague name makes every comment necessary.

**P4. Make invalid states unrepresentable.** Prefer types and constructors that
cannot produce a nonsensical value over checks scattered through the code that
hope to catch one.

**P5. One concept, one home. One fact, one source.** Each idea lives in exactly
one place, and every other place refers to it. A value written down twice will
eventually disagree with itself.

**P6. Concise, never cryptic.** Remove everything that does not carry meaning:
dead code, needless indirection, ceremony, redundant comments. Never remove
meaning to save characters. *Short* and *clear* are different goals; when they
conflict, choose clear.

**P7. Robust by default.** Fail loudly and early at boundaries, handle every case
explicitly, and leave nothing to coincidence. Silent failure is the most expensive
kind.

**P8. Consistency is a feature.** A consistent code base is navigable by
pattern. The *same* thing is always done the *same* way, so a *different* way
signals a *different* thing.

**P9. Let tools enforce what tools can enforce.** Formatting, import order, lint
rules, and type correctness are settled by tooling, never debated in review.
Human attention is reserved for design and meaning.

---

## 2. Naming

**N1. Use whole words.** Names are spelled out. A gamepad overlay selector
widget is `gamepad_overlay_selector_widget`, not `gosw`, `gp_sel`, or `w`.
The reader should never need to decode a name.

**N2. Avoid abbreviations as far as sensible.** This is a strong default, not
a mechanical ban. The test is the reader who is *not* an expert in this code: an
abbreviation is acceptable only if that reader could not reasonably fail to
understand it. In practice that means it is *more* recognizable than its
expansion, or it is the name of the thing itself: `url`, `http`, `json`, `csv`,
`xml`, `id`, `io`, `utf8`, `gpu`, `cpu`, `usb`, `pactl`. These are not
acceptable: `cfg`, `mgr`, `ctx`, `idx`, `cnt`, `buf`, `tmp`, `val`, `obj`,
`res`, `ret`, `str_`, `elem`, `p`, `reloc`, `venv_minor`.

Domain jargon sits between the two. `iv` is standard in cryptography, yet it is
also an example of why the default leans toward spelling things out: it is
easily misread as "initial value" when it means *initialization vector*. Where
a term is jargon to the domain but not to every reader of the code, prefer the
expansion (`initialization_vector`), or use the abbreviation consistently and
define it in the type's or module's documentation. Each project MAY keep a short
glossary of accepted domain abbreviations in its design document.

**N3. Single-letter names are reserved for conventions older than the code.**
Allowed: `x`, `y`, `z` for coordinates; `i`, `j` for a bare index in a short
arithmetic loop *only* when no better word exists (and one usually does:
`row_index`, `byte_offset`); generic type parameters where the language
convention expects them (`T` for "any element"). Everything else gets a word.
Lambdas and closures are not an exception: `lambda entry: entry.name`, not
`lambda e: e.name`.

**N4. Name what a thing *is*, not how it happens to be used.** A window that
selects a gamepad is a `GamepadSelectorWindow`, even if the tray icon is what
opens it; calling it `TrayWindow` describes one caller, not the thing. A hook
that fires on a general event is named for the event (`on_idle`), not for the
first job someone gave it (`on_tray_ready`).

**N5. Name with the precision the reader needs.** Choose the word that rules
out the wrong interpretation. `english_only` (exclude everything non-English) is
better than `require_english` (which could mean "must include English").
`--filter-metadata` says nothing. Name the effect.

**N6. Speak the domain's language, and only one dialect of it.** Pick one term
per concept and use it everywhere: code, CLI flags, log messages, README, and
tests. If the project says *gamepad*, the word *controller* does not appear.
Record the chosen terms in the project's design document.

**N7. Grammatical roles are fixed.**

| Kind of item | Form | Examples |
|---|---|---|
| Types, classes, traits | Noun phrase | `ProfileSelector`, `BatteryStatus` |
| Functions/methods with effects | Imperative verb phrase | `apply_profile`, `save_primary` |
| Queries without effects | Noun phrase, or `is_`/`has_`/`can_` predicate | `default_mode`, `is_connected` |
| Booleans | Predicate, positive form | `is_active`, `has_monitor`, `allow_column_subset` |
| Collections | Plural noun | `outputs`, `candidate_sinks` |
| Mappings | `key_to_value` or `value_by_key` | `name_to_data`, `LANGUAGE_BY_VALUE` |

Both mapping forms are chosen for being unmistakable, not clever: a reader who
has never seen `name_to_data` still knows what it holds and which way it maps.
Use whichever reads best where the mapping is used. `key_to_value` reads well
when the mapping is described or passed around (`field_to_column_index`), and
`value_by_key` reads well at a lookup (`price_by_item[item]`) and for grouped
values (`sinks_by_card`).
| Counts / indices / sizes | `_count`, `_index`, `_size` suffix | `column_count`, `row_index` |
| Quantities with units | Unit suffix (or a unit type, see M9) | `timeout_seconds`, `max_kb` |
| Conversions | `to_`, `as_`, `from_`, `into_` | `to_cli_string`, `from_stem` |

**N8. Avoid negative booleans.** `is_enabled`, not `is_not_disabled`;
`if not is_enabled` reads naturally, while `if not is_not_disabled` does not.

**N9. Use symmetric vocabulary for symmetric operations.** Use `open`/`close`,
`begin`/`end`, `load`/`save`, `push`/`pop`, `acquire`/`release`, and
`block`/`delete_block`. Do not mix `load`/`write`, and do not mix `start`/`finish`.

**N10. No weasel words.** `Manager`, `Helper`, `Util`, `Handler`, `Processor`,
`Data`, `Info`, `Object`, `Thing`, `Stuff`, and `Service` rarely add
information. If a type handles bookings, call it `Bookings`. If it dispatches
them, call it `BookingDispatcher`. The test: delete the weasel word, and if
nothing is lost, it should not have been there. These words are acceptable when
they are the precise term: an *event handler* is a handler, and `archlinux-manager`
is a tool that manages Arch Linux.

**N11. Context is part of the name; do not repeat it, do not truncate it.**
A field `name` on `Sink` is already the sink's name: `sink.name`, not
`sink.sink_name`. But context only counts if it is *visible at the point of use*.
A free-floating `name` local in a 200-line function is not "obviously" the sink's
name. Name for the place where the name is *read*.

**N12. Project and product names are literal.** A gamepad overlay is
`gamepad-overlay`. No witty names. When a literal name collides, qualify it with
the organization (`sevaht`) rather than inventing a pun.

**N13. Private means "implementation detail," not "I haven't thought about
reuse."** Mark an item private only when it exists solely to serve its module's
public items. Something designed to be reused is public and lives where reusers
can find it, usually in a library module, not hidden inside the first consumer.

---

## 3. Modeling and object-oriented design

This is the heart of the standard. The goal is code organized around
**concepts that own their data and behavior**, not a flat field of free
functions that pass the same arguments around.

**M1. Behavior lives with the data it governs.** If an operation's main
subject is a value of some type, it is a method on that type. A reader looking
at `Screen` should find everything a `Screen` can do in one place.

**M2. The shared-subject test.** When two or more functions operate on the
same thing (the same value, idea, or concept), that thing is an object you have
not named yet. Name it, make the value its state, and make the functions its
methods. Argument position does not matter: `read_config(path)`,
`merge_into(defaults, path)`, and `backup(path, destination)` share a subject
even though `path` moves around. Neither does argument identity: functions that
take a `path` in one place and an already-open handle to the same file in
another still share a subject. The quickest symptom is a shared *first*
argument, but the real question is "what are these functions about?"

> *Before:* `read_config(path)`, `write_config(path, data)`,
> `read_section(path, name)`, `update_section(path, name, data)`,
> `remove_section(path, name)`.
>
> *After:* `ConfigFile(path)` with `.read()`, `.write(data)`,
> `.section(name)`, `.replace_section(name, data)`, `.remove_section(name)`.

The after version is shorter at every call site. It is discoverable, because one
type lists everything you can do. It is also extensible: caching, locking, or a
different format changes one class, not five signatures.

**M3. The travelling-parameter test.** When the same group of values is passed
from function to function (`attached, client_count`, or `x, y, width, height`),
the group is a type. Load it once, validate it once, and pass the object.
Configuration is the canonical case. Parse the file into a typed configuration
object at the boundary, validate it there, and pass that object inward. Never
copy fields one at a time into a dozen members.

**M4. Classify every type by role.** Most well-designed types fall into one of a
few roles. Knowing the role tells you how the type should behave.

| Role | Purpose | Character |
|---|---|---|
| **Value object** | Describes something: `Resolution`, `Mode`, `BatteryStatus` | Immutable, compared by value, hashable, printable, cheap to copy |
| **Entity** | Something with identity that changes over time: `State`, `Route` | Mutable through methods that preserve invariants |
| **Gateway / adapter** | Wraps an external system: `PACtl`, `WlrRandr`, `CLITool` | Owns all knowledge of the external protocol; exposes domain types |
| **Coordinator** | Runs a use case across other objects: `ProfileSelector` | Holds collaborators; contains *policy*, delegates *mechanism* |
| **Interface** | A capability several types provide: `TrayIcon` | Small, named for what it enables |

A type that fits no role, or fits three, is usually a design smell.

**M5. Prefer value objects.** Most domain types should be immutable values. They
cannot be corrupted by aliasing, are safe to share, are trivially testable, and
make "changing" something explicit (`screen.with_primary(name)` returns a new
`Screen`). Mutation belongs in a small number of entities and coordinators.

**M6. Construct valid, or do not construct.** Validation happens once, at
construction. After that, a value of the type *is* proof of validity
("parse, don't validate"). Raw input becomes a domain type at the boundary,
and the interior never sees raw strings, dictionaries, or JSON.

**M7. Named constructors express where values come from.** Use
`from_stem(...)`, `from_json(...)`, `from_identifiers(...)`, and `for_capacity(...)`.
A constructor named for its source documents the conversion and lets one type
have many origins without overloaded, ambiguous signatures.

**M8. Closed sets are enumerations, and enumerations carry behavior.** When a
value is one of a fixed set, it is an enum, never a bare string or integer.
Behavior that varies by member lives on the enum. For example,
`BatteryLevel.for_capacity(85)` and `NameStyle.SNAKE_CASE.config` keep the
mapping in one place instead of an `if`/`elif` ladder repeated at each use.

**M9. Give primitives meaning with types.** Ports, identifiers, percentages,
durations, offsets, and file names are not interchangeable integers and strings.
When confusing two of them would be a bug, give each its own type. Use the
standard library's types where they exist: durations, paths, dates.

**M10. Composition over inheritance.** Build objects out of other objects.
Inheritance is reserved for genuine *is-a* relationships with shared
implementation, kept shallow (one level is typical, two is rare). Never inherit
merely to reuse code.

**M11. Interfaces describe capabilities, and stay small.** An interface exists
when there are (or will imminently be) several implementations: platform
backends, test doubles, plug-ins. It names a capability (`TrayIcon`,
`ServerBackend`) and contains the fewest operations that capability needs.
Choose the implementation behind one factory function or constructor so the rest
of the program never knows which backend it holds.

**M12. Separate the domain from the machinery.** Domain models (what a screen
*is*) know nothing about how the information was obtained (`wlr-randr` output,
`pactl` JSON, `/sys` files). Gateways translate external formats into domain
types. Coordinators combine the two. The command-line interface is a thin layer
that parses arguments, calls coordinators, and reports results.

**M13. Dependencies point inward and downward.** The entry point
(`application`, `main`) imports everything and is imported by nothing. Domain
modules import nothing from gateways or the user interface. Shared, generic
facilities move *down* into a library that knows nothing about its consumers.
If two modules need something from each other, the shared part belongs in a
third module both can depend on.

**M14. Generic belongs in the library; specific belongs in the program.**
When a facility is not specific to this program (a tk utility widget, a hook
mechanism, a typed CSV loader), it goes into a reusable library or a clearly
generic module. The program's own code then reads as *only* what this program
does. This is how a driver script becomes clear: the generic machinery is
somewhere else.

**M15. When free functions are right.** A function that has no natural subject
is a free function. This covers a pure transformation between equals, a
mathematical operation, or the program's entry point. Such functions MUST live
in a module whose name supplies the missing noun (`naming.convert_name`,
`units.parse_duration`), so the call site still reads as *concept.operation*.
A module of loosely related free functions named `utils`, `helpers`, or `common`
is not a concept; see O3.

**M16. One responsibility per type, but not one method per type.** Split a type
when it has two independent reasons to change, not because it is long. Keep two
distinct features separable: if a style and a label can be used independently,
model them independently, even when they are usually used together.

---

## 4. Functions and interfaces

**F1. A function does one thing at one level of abstraction.** A function that
both decides policy and performs byte-level mechanics should be two functions.
Length is a symptom, not the disease. A long function at one level of
abstraction is often fine.

**F2. Behavior-changing choices are explicit at the call site.** Avoid bare
boolean and `None`-as-flag arguments: `render(True, False)` is unreadable. Use
an enumeration, a separate well-named function, or, where the language has
them, a keyword-only argument. When an argument changes *what the function
does*, it MUST NOT have a default. Make every caller state its choice.

**F3. Few parameters.** More than about four parameters usually means a missing
type (M3) or a function doing two things (F1). Options that travel together
become an options object (`CsvLoadOptions`).

**F4. Separate commands from queries.** A function either changes state or
answers a question. Queries do not have effects. Commands MAY return what they
did (the profile that was applied), but a query never quietly mutates.

**F5. Return rich types, not codes.** Return the domain value, a result or
optional type, or a small record. Do not return `-1`, magic empty strings, or
out-parameters that signal "nothing found." The language's "absent" construct
(`None`, `Option`) exists for this.

**F6. Accept the most general input, return the most specific output.** Accept
an iterable, not specifically a list. Accept a path-like value, not specifically
a string. Return the concrete type so callers keep the full capability.

**F7. Callable from code, not only from a command line.** Every operation a CLI
exposes is an ordinary function or method with ordinary defaults, usable from
other code and tests. CLI frameworks that make functions behave differently
when called programmatically, such as hidden metadata defaults, are avoided.

**F8. Every interactive feature has an equivalent non-interactive path.**
Whatever the GUI or TUI can do, the command line can do too, with matching
terminology and output.

**F9. Wait for events; do not poll.** To learn that something changed (a
file, a process, a device, a service, the clock), subscribe to the
notification the system offers: inotify, a D-Bus signal, a process or
pidfd descriptor, a netlink or udev event, a timer armed for the moment
itself. Polling (checking on a timer whether anything changed) wastes
power, keeps the machine awake, and still sees changes late. It is
acceptable only when there is no conceivable way to be told, and then the
code says so where it polls, naming the event source it looked for and did
not find. A timer that fires because a moment has come (a clock's next
minute, a timeout, key repeat) is not polling; a timer that fires to look
at something is.

---

## 5. Errors and robustness

**E1. Distinguish failures from bugs.** A *failure* is something the world did:
a missing file, a malformed input, an absent dependency. It is reported through
the language's error mechanism and handled where someone can act on it. A *bug*
is a broken invariant inside the program. It stops the program loudly, with a
message that says which invariant broke.

**E2. Errors are specific types that carry context.** An error says what
happened, to what, and why, as structured data and not only prose. For example,
`ShortRowError(line_number=, field_name=, column_index=, column_count=)` can be
handled, logged, and tested. A generic error with an interpolated string can
only be read.

**E3. Error messages are actionable.** State what failed, with which input, and,
when known, what to do: "required command not found on PATH: 'wlr-randr' —
please install it." The user should not need a stack trace for an expected
failure.

**E4. Never swallow an error silently.** Catching broadly is permitted only at a
boundary that genuinely cannot know what the callee raises, such as user-supplied
converters or plug-ins. It MUST be commented with the reason, and it MUST log
what was caught.

**E5. Preserve causes.** When translating a low-level error into a domain error,
chain the original as its cause.

**E6. Report at the top, decide in the middle, detect at the bottom.** Low-level
code detects and raises. Mid-level code adds context or recovers when it has a
real alternative. The entry point reports to the user and chooses the exit
status.

**E7. Handle every case.** Pattern matches and enum dispatches are exhaustive.
Adding a new member must force every dispatcher to be revisited, and the
language's exhaustiveness checking is used to make that happen.

**E8. Cleanup is structural.** Resources are released by scope-bound
constructs: context managers, destructors, `Drop`, and `using`. Never rely on
manual cleanup calls that an early return or an error can skip.

---

## 6. Constants and magic values

**K1. No unexplained literals.** A number or string with meaning beyond itself
gets a name. If its value is not self-evident, it also gets a comment saying
*why this value*, what changes if it changes, and where it comes from.

**K2. Derive rather than hard-code.** If a value can be computed or looked up
from the authoritative source, such as a symbol's address in a disassembly, a
size from a structure definition, or a version from the package metadata, compute
or look it up. Hard-coded copies of derivable facts rot.

**K3. Say it once.** Identity facts, such as the organization name, the
application name, or a configuration file name, are defined in exactly one place.
Everything else derives from that definition. Prefer one shared accessor
(for example, a cached function returning the platform-directories object) over
several narrow helpers that each re-state part of the fact.

**K4. Constants live with their owner.** A constant used by one type is a class-
or type-level constant of that type. A constant shared by a module is defined
near the top of that module. Global constant dumping grounds are avoided.

---

## 7. Comments and documentation

The aim is documentation that is **generated from the code**, so it cannot drift
from it, and supplemented by a small amount of prose that explains *why*.

**D1. Every public item is documented in the code.** Use the language's native
documentation syntax (docstrings, `///` doc comments) so IDEs show it on hover
and tooling renders it to a browsable site. Documentation that lives only in a
wiki is documentation that will be wrong.

**D2. The first sentence is a summary.** It is one line, about 15 words or
fewer, and makes sense on its own in an index listing. Further paragraphs
elaborate.

**D3. Document behavior and contract, not implementation.** Describe what the
item does, what it requires, what it returns, what it raises or panics on, and
any side effects. Explain *how* only where the *how* is the point.

**D4. Every module opens with a description of its concept.** The module
documentation says what the module models, how its types relate, and when to use
it. A reader should understand the module's design from this paragraph before
reading any code. For example: "Backend-neutral value types: what a monitor can
do (`Monitor`), how an output is configured (`Configuration`), and the whole
layout (`Screen`)."

**D5. Examples are executable.** Usage examples in documentation are compiled or
run as tests (doctests), so they are proven correct on every build.

**D6. Comments explain *why*.** The code says *what*. A comment earns its place
by recording intent, a constraint, a non-obvious consequence, or the reason the
obvious alternative was rejected. A comment that restates the code is noise, and
it becomes a lie the first time the code changes.

**D7. Comments are concise.** One accurate sentence beats a paragraph. A large
banner comment at the top of a short script means the script needs a README or
a better structure.

**D8. No commented-out code.** Version control remembers it. Code kept
deliberately but not used, such as dead data retained for fidelity in a ROM
modification, is marked clearly and succinctly as intentionally unused, with the
reason.

**D9. TODOs are actionable.** Write `TODO(topic): what needs doing and why`.
A TODO without enough context for a stranger to act on it gets deleted.

**D10. Three documents, three audiences.**
- **README**: for users. It says what the tool does, how to install it, and how
  to accomplish real tasks, written as a guide. It assumes CLI literacy, not
  knowledge of the code.
- **API documentation (generated)**: for programmers using the code. It is
  produced from doc comments and published (GitHub Pages or docs.rs).
- **DESIGN.md**: for maintainers and AI agents. It covers architecture, data flow,
  invariants that are easy to break, rejected alternatives, and the domain
  glossary (N6). It focuses on *why*, and points into the code rather than
  duplicating it.

---

## 8. Organization

**O1. Group by concept, not by kind.** Modules are named for what they model
(`battery`, `profiles`, `screen`, `wlr_randr`), not for the syntactic category
of their contents (`classes`, `types`, `enums`, `errors`, `constants`). An error
type lives beside the code that raises it.

**O2. File names describe contents.** If a file's name does not tell a reader
whether the thing they seek is inside, rename it. A module that grows a second,
unrelated concept splits.

**O3. No junk drawers.** `utils`, `helpers`, `misc`, and `common` modules inside
an application are where design goes to die: everything fits, so nothing belongs.
Find the concept each function serves and move it there. A dedicated, cohesive
*library* of general utilities (such as `sevaht-utility`) is different. It is
itself a product with a documented scope.

**O4. Entry points are thin.** The program's entry module parses arguments,
configures logging, builds the top-level objects, dispatches, and maps errors to
exit codes. Classes and domain functions do not live there.

**O5. Order within a file reads top-down.** Module documentation comes first,
then imports, constants, public types, and finally private helpers. Within a
type, constructors come first, then queries, then commands, then conversions and
representations. Readers meet the public story first and the machinery last.

**O6. Balanced modules.** Avoid both extremes: one 3,000-line module holding
everything, and a forest of one-function modules. A module is the right size
when its documentation (D4) can describe it in a short paragraph.

---

## 9. Formatting and static verification

**V1. Formatters are authoritative.** Code is formatted by the language's
standard formatter, with project configuration committed. Hand-formatting
against the tool is not allowed.

**V2. Line length is enforced.** The limit is set per language (see the language
standards). It applies to code, comments, and documentation alike. Long string
literals are split at natural boundaries, not crammed.

**V3. Linters run at their strictest practical setting.** Enable the broadest
rule sets the ecosystem offers, then opt out of individual rules with a recorded
reason. Do not start from the defaults and opt in.

**V4. Suppressions are rare, local, and justified.** A suppression applies to the
narrowest scope possible (one line or one item), names the specific rule, and
states why. Before suppressing, fix the cause. For example, write a small type
stub instead of ignoring a whole typed module, and restructure instead of
silencing a complexity warning. Suppressions that become unnecessary must be
detected and removed.

**V5. Strict type checking.** Use the strictest mode of the type checker. Every
function signature is fully typed. Escape hatches such as `Any`, unchecked casts,
and `unsafe` are confined to boundaries and justified.

**V6. One command runs every check.** Each repository has a single entry point
(`./checks`) that formats, sorts imports, type-checks, lints, finds dead code,
and runs tests, in that order, stopping at the first failure. CI runs the same
command.

**V7. Documentation completeness is reported, not gated.** Missing or
malformed documentation shows up as a warning in ordinary builds and in
`./checks`, but never fails them. `./checks --docs` runs the documentation
checks alone (presence, format, and a warning-free documentation build) and
fails on any finding. The rules in §7 still apply; this rule only decides when
a gap stops the work.

---

## 10. Testing

**T1. Test behavior, not implementation.** Tests exercise public behavior
through public interfaces, so the internals can be refactored freely.

**T2. Tests do not merely restate the code.** A test that recomputes the expected
value with the same logic as the implementation proves nothing. Expected values
come from independent knowledge: hand-computed cases, known references, and
invariants.

**T3. Tests are documentation.** Test names state the behavior they verify:
`test_cycle_primary_wraps_to_first_candidate`. A reader should be able to learn
the feature from the test list.

**T4. Isolate the world.** File systems, clocks, subprocesses, networks, and
hardware are reached through gateways (M4) that tests can replace. Tests do not
depend on the machine they run on.

**T5. Prefer fixtures and data over repetition.** Shared setup goes in fixtures
or helpers. Many cases of one behavior go in a parametrized table, not copied
test bodies.

**T6. Every bug fix adds a test that fails without it.**

**T7. Model for testability: decide in a pure model, act in a thin shell.**
Every decision code makes (which mode to set, which output is primary, what a
hotplug changed, where a window goes) is a function of plain values in a type
with no I/O, so a unit test can give it any situation. The code that talks to
hardware, the kernel, other processes, or the clock is a thin shell: it reads
the world into those values, asks the model, and carries out the answer. If a
behavior can only be checked on the real device, that is a design smell:
pull the decision out until the device-bound remainder is too simple to be
wrong. The question to ask while designing is not "how will I test this?"
after the fact, but "what plain values does this decision depend on?" first.

**T8. Every layer has its own tests, and the whole has a simulated run.**
Pure models get unit and property tests; each gateway gets a fake that tests
can drive (a fake backend, a fake bus, a fake clock); and the program as a
whole runs end to end against those fakes (a headless backend, private
directories) the way it runs for real. A feature is not done until each layer
it touches is covered, so that the first run on real hardware confirms rather
than discovers.

---

## 11. Dependencies and platforms

**X1. Use maintained libraries for solved problems.** Process inspection,
platform directories, argument parsing, serialization, and terminal styling are
solved problems. Use the established library (`psutil`, `platformdirs`,
`clap`, `serde`, `rich`) instead of reinventing it, unless there is a concrete
reason not to (size, licensing, a missing capability).

**X2. But keep the set small.** Every dependency is a liability that must be
justified by what it saves. Prefer one well-chosen library over three overlapping
ones.

**X3. Version constraints are explicit.** Specify compatible ranges and commit
the lockfile for applications.

**X4. Support only the platforms you target.** Name the supported platforms in
the README. Code paths and tests for untargeted platforms are dead weight.
Platform differences are isolated behind an interface (M11), never scattered as
`if platform == …` checks.

---

## 12. Notes for AI collaborators

These rules exist because of recurring problems. They are binding on any agent
working in these code bases.

1. **Design before typing.** Before writing a function, ask which type it
   belongs to (M1, M2). A change that adds several free functions that share
   arguments is incomplete until those functions are reorganized into a type.
2. **Match the existing style.** Read neighbouring code first. Follow its
   structure, naming, and idioms, even when you would have chosen differently,
   unless the task is to change them.
3. **Do not hide reusable code.** Do not underscore-prefix or nest a facility
   that is meant to be reused (N13). Put generic facilities in the library (M14).
4. **Do not silence tools to get green.** Fix the cause (V4). A new suppression
   in a diff needs a justification the user would accept.
5. **Do not leave compatibility debris.** When renaming or moving something,
   update every reference. Do not leave re-exports, aliases, or wrapper shims
   at the old location unless the user asks for them.
6. **Rename aggressively, consistently, and completely.** When a better name
   is found, apply it everywhere: code, CLI, logs, docs, tests (N6).
7. **Stay in scope, but report what you see.** Fix what was asked. Mention
   other problems you notice; do not silently fix unrelated things.
8. **Version control belongs to the user** unless explicitly delegated. Propose
   commit messages; do not commit, push, or tag.
9. **Concision applies to you too.** Do not produce enormous comments, files,
   or reports when a short one is enough (P6, D7).

---

## Appendix A. Review checklist

- [ ] Could a newcomer predict where each new piece of behavior lives? (M1)
- [ ] Do several functions operate on the same thing, or pass the same group
      of values around? (M2, M3)
- [ ] Is every name a whole word, precise, and in the project's vocabulary?
      (N1–N6)
- [ ] Are boundaries validated once, with domain types used in the interior?
      (M6)
- [ ] Are closed sets enums, and are their matches exhaustive? (M8, E7)
- [ ] Are errors specific, contextual, and actionable? (E2, E3)
- [ ] Is every change learned of by an event rather than by polling? (F9)
- [ ] Are literals named and explained, and are derivable facts derived?
      (K1, K2)
- [ ] Do public items and modules have documentation with a one-line summary?
      (D1, D2, D4)
- [ ] Does `./checks` pass with no new suppressions, or with justified ones?
      (V4, V6)
- [ ] Do tests cover the behavior, including the bug just fixed? (T1, T6)
- [ ] Are decisions in pure models, with the I/O shell thin enough to trust? (T7, T8)
