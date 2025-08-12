# Streamable Flat Object Representation (SFOR)

**Version**: 0.1\
**Status**: Draft (Open for feedback)

SFOR is a lightweight, stream-friendly, and human-readable data serialization format that represents structured objects using a clear, line-based syntax. It is inspired by the readability of YAML, the predictability of JSON, and the modularity of protocol buffers.

---

## 📌 Key Features

* Line-based, whitespace-tolerant format
* Fully streamable with forward references
* Deterministic parsing with minimal grammar
* Supports comments, metadata, and references
* Optional type hinting and custom type definitions for stricter parsing and casting
* Ideal for configurations, logs, and modular data transport
* Easy to write by hand, parse by machine

---

## 🔤 Syntax Summary

Each line must begin with an optional amount of leading whitespace, followed by a **control character** and a space. The control character defines the type of line.

### Line Types

| Symbol | Meaning                             | Example             |
| ------ | ----------------------------------- | ------------------- |
| `=`    | Key-value pair (single line)        | `= version = 0.2`   |
| `@`    | Section/header start                | `@ headers`         |
| `:`    | Key-value inside section            | `: H1 = Exclude`    |
| `,`    | List item (multi-field)             | `, H1, keyboard`    |
| `-`    | List item (single-field)            | `- 1`               |
| `&`    | Reference placeholder               | `, & id1`           |
| `>`    | Begin downstream referenced content | `> id1`             |
| `<`    | End referenced content              | `< id1`             |
| `#`    | Comment (inline only, after data)   | `- 2 # second item` |

### Special Sections

| Symbol      | Description                      |
| ----------- | -------------------------------- |
| `> comment` | Begin a multi-line comment block |
| `< comment` | End a multi-line comment block   |

---

## 📄 Example

### JSON Equivalent

```json
{
  "version": "0.2",
  "headers": {
    "H1": "Exclude",
    "H2": "NewLine",
    "H3": "Knolbay:Zoo@1.0.0",
    "H4": {
      "name": "Knolbay:Animals@1.0.0",
      "type": "animals"
    }
  },
  "segments": [
    ["H3", "010010"],
    ["H1", "keyboard"],
    ["H3", "11"],
    ["H1", "mouse"]
  ],
  "codes": [1, 2, 3]
}
```

### SFOR Equivalent (without type hints)

```
> main
  = version = 0.2
  @ headers
    : H1 = Exclude
    : H2 = NewLine
    : H3 = LUT:Knolbay:Zoo@1.0.0
    : H4 = & id1
  @ segments
    , H3, 010010
    , H1, keyboard
    , H3, 11
    , & id2
  @ codes
    - 1 # this is the first code
    - 2
    - 3
< main

> comment
  the main object representation is paused here and sub-objects follow
  which will be interpreted as attached to the references from the main object
< comment

> id1
  @
    = name = Knolbay:Animals@1.0.0
    = type = animals
< id1

> id2
  @
    , H1, mouse
< id2

> main
  - 4
  - 5
< main
```

> Note: please refer [Optional Typing System](#-optional-typing-system) section below for type hinted example.

---

## 📘 Parsing Rules

* All lines must begin with one of the defined control characters followed by a space.
* Leading whitespace before control characters is allowed and ignored.
* Inline comments must follow a `#` after the value.
* Multi-line comments can be embedded between `> comment` and `< comment`.

### Escaping

Escaping is needed only for specific cases to preserve parsing clarity:

* `\\n` → Literal backslash followed by the character 'n' inside value (as `\n` indicates literal newline)
* `\#` → Literal `#` inside value (to avoid triggering comment parsing)
* `\(` → When used at the beginning of a value, to avoid being misinterpreted as a type hint

Additional characters may require escaping depending on usage and parsing context — this is still under evaluation.

> Note: Escaping aims to minimize cognitive load and while remaining unambiguous (simplify parsing).

### References

* `& id` marks a forward reference
* `> id` begins a downstream block with id `id`
* `< id` ends the block
* Duplicate `< id` is invalid
* Missing `< id` or unresolved `& id` should be handled gracefully by the SDK

### Contextual Sections

* `@ key` defines a new section or substructure
* `@` (with no key) may be used in reference blocks to attach data directly to the reference — such as a list or dictionary — without worrying about naming
* Data under `@` is inserted into the containing structure; `key` is not preserved unless explicitly retained
* A reference block must contain **either** a dictionary or a list — not a mix of both

---

## 🧪 Implementation Notes

* Parsers must support forward-only reading for streaming
* References should be resolvable post-parse via in-memory mapping or lazy evaluation
* Data should be serializable back to JSON or other common formats via SDKs

---

## 🔠 Optional Typing System

SFOR supports **optional inline type hinting** and **user-defined structured types** to enable safer parsing, validation, and richer schema support.

### 🔹 Inline Type Hints

To enforce or hint a type at any data point, prefix the value with a type in parentheses:

```
@ numbers
  - (int) 1
  - (string) hello
  - (float) 3.14
  - (bool) true
```

Parsers may cast values accordingly or skip type enforcement, but must provide introspective capabilites.

| Type Hint    | Description                                                                 |
| ------------ | --------------------------------------------------------------------------- |
| `(int)`      | Integer (e.g. `42`)                                                         |
| `(float)`    | Floating-point number (e.g. `3.14`) — may lose precision                    |
| `(decimal)`  | Arbitrary-precision decimal (e.g. `3.141592653...`) — preferred for money   |
| `(bool)`     | Boolean (`true` or `false`)                                                 |
| `(string)`   | Textual data                                                                |
| `(null)`     | Represents `null` / `None` / `nil`                                          |
| `(date)`     | ISO 8601 date (e.g. `2025-08-05`)                                           |
| `(time)`     | Time only (e.g. `13:45:00`)                                                 |
| `(datetime)` | Full ISO timestamp (`2025-08-05T13:45:00Z`)                                 |
| `(uuid)`     | Universally unique identifier (e.g. `550e8400-e29b-41d4-a716-446655440000`) |
| `(bin)`      | Binary digits (0s and 1s)	                                                 |
| `(hex)`      | Hex-encoded binary data                                                     |
| `(b64)`      | Base64-encoded binary data                                                  |
| `(regex)`    | Regex pattern (for validators)                                              |
| `(enum)`     | Value must be from a predefined list (can reference enum block)             |
| `(any)`      | Explicitly accepts any type — bypasses validation                           |

> 📝 Note: The above type hints are exploratory and subject to change. They illustrate a potential direction for SFOR's optional type safety features, but final types, syntax, and semantics may evolve with further feedback.

### 🔹 Custom Structured Types

Custom types allow you to define structured, reusable value formats using simple grammar-like expressions. These types are defined at the top of the document using `> types` and `< types` blocks.

```
> types
  (address) = (int=3.) (string=2-10) (string=40)
< types
> main
  @ person
    - (int) age # 39
    - (string) BruceKaushik # nickname
    - (address) 10, A.S. Rao Nagar, TG
< main
```

The type definition:

`(address) = (int=0-99) (string=2-10) (string=2.)`

Defines `address` as a composed type consisting of:

  * an integer between `0` and `99`
  * a string of exactly 2 characters
  * a string between 2 and 10 characters long

Parsers can use these definitions to validate, cast, and extract structured data with confidence.

> 📝 Note: Just like inline type hints, custom structured types are exploratory and subject to change. They represent a possible direction for SFOR's type system, but final syntax, semantics, and validation rules may evolve based on community feedback and real-world usage.

### 🔹 Notes

* Type hints are **completely optional**.
* Custom types allow for reusable semantics, schema definitions, and stricter parsing.
* This system makes SFOR suitable for use cases like DSLs, templating, and interop with statically typed systems.

---

## 🧪 API: Access and Introspection

### File Objects

```python
import sfor

# Open in efficient read mode: enables streaming access with minimal memory usage
# The entire file is not loaded into memory — only byte offsets (.tell()) are stored
obj = sfor.open('data.sfor', 're')

# Mandatory scan step when re mode is used: parses the file once to build an internal index of offsets
# This explicit step makes it clear to developers that a full scan is performed before random access is enabled
obj.scan()
```

---

### 📍 Path-Based Access

SFOR supports intuitive path-based lookups:

```python
obj.get('main/headers/H1')
```

- Returns the value if loaded or decodable.
- Returns `None` (sentinel value) if the section is excluded or not yet available.

---

### 🔍 Introspection

You can query metadata about any path using the second argument to `get()`:

```python
obj.get('main/headers/H1', 'type')      # e.g., 'string', 'int', 'address'
obj.get('main/headers/H1', 'comment')   # Inline comment text if available
obj.get('main/headers/H1', 'span')      # Byte range as (start_byte, end_byte)
obj.get('main/headers/H1', 'raw')       # Original string value before parsing
```

> You can optionally use `.has(path)` to check existence before accessing.

---

### 🧾 Read Modes

- `'re'` — **read-efficient**: streaming, low memory, requires `.scan()`
- `'rl'` — **read-load**: loads entire SFOR file into memory

The default `'r'` mode is intentionally unsupported to force developers to be explicit.

---

### 🧰 Other Possible API Methods

```python
obj.require('main/data/H3/')  # Raises error if missing
obj.has('main/data/H3/')      # Returns True/False
obj.meta('main/data/H3/')     # Shortcut for all metadata
```

---

## 📁 Suggested File Extension

* `.sfor`

## 🧾 MIME Type

* `application/sfor+text`

---

## 🚧 TODO

* Don't assume that consumers will always load the entire structure into memory using dictionaries or hashmaps
  * SFOR must support streaming or partial access to data, especially for large files
  * consumers will need to build the structure as well as read it
  * the API must be updated reflect building phase

* Consider using a storage layer (again more important for building phase)
  * allows efficient, memory-safe access and indexing
  * backing the parser with SQLite/LMDB
  * enables scalable random access, filtering, and introspection without holding the full structure in memory

---

## Feature Comparision

| Feature                     | JSON   | YAML   | TOML   | Protobuf       | SFOR      |
|-----------------------------|--------|--------|--------|----------------|-----------|
| Streaming-Friendly          | No     | Partial| No     | No             | **Yes**   |
| Forward References/Detached | No     | Partial| No     | No             | **Yes**   |
| Human Readable/Edit-Friendly| Partial| Yes    | Yes    | No (binary)    | **Yes + regular** |
| Explicit Referencing        | No     | Partial*| No    | No             | **Yes**   |
| Simple Canonicalization     | Partial| No     | Partial| Yes (binary)   | **Partial**   |
| No Required Backtracking    | No     | No     | No     | Yes (binary)   | **Yes**   |
| Type Safety                 | No     | Minimal| Minimal| **Yes (schema)** | **Optional, inline & schema** |

\* YAML has anchors/aliases but not formal detached blocks or forward-only block parsing.

> **Note**: If your use case does not require streamability, explicit typing, or minimal memory usage, you may be better served by established formats such as JSON, YAML, TOML, or Protocol Buffers, which have broader ecosystem support and tooling.

---

## Ideal Use Cases

- **Any system that needs partial, modular, or lazy object construction:**
  _e.g., config overlays, plugin manifests, streamed logs, structured state snapshots._
- **Very large datasets** where full-in-memory loading isn’t feasible.
- **Advanced data pipelines:** ML, data ops, IoT, telemetry, scientific logging.
- **App ecosystems requiring** *editable, compositional, streamable* config.
- **Canonical, deterministic text serialization** for hashing, deduplication, audit, or migration — applicable to data types with unique textual representation and subject to normalization rules for types like floats.
- **Optional and embedded type safety,** reducing errors in data interchange or dynamic pipelines.

---

## 📜 License

SFOR is experimental and currently under active development. Licensed under MIT.

---

## ✍️ Author

Made by Nanduri Srinivas Koushik.

Ideas, suggestions, constructive criticism, and contributions are always welcome. Whether it’s proposing improvements, pointing out flaws, or sharing new use cases — your input helps make SFOR better.
