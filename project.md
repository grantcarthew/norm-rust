# norm-codec Project Plan

## Overview

Rust crate implementing a parser and encoder for the NORM data format (v0.1).
Spec: https://github.com/norm-format/norm-spec (spec.md is the authoritative source)

Crate name: `norm-codec`
CLI binary name: `norm`
License: MPL-2.0
Rust edition: 2021

## Crate Shape

Single crate with a library target and a binary target.

The library exposes the public API. The binary provides the CLI. They share the same `Cargo.toml`.

## Public API

```rust
use serde_json::Value;

/// Parse a NORM document string into a JSON value.
pub fn parse(input: &str) -> Result<Value, NormError>

/// Encode a JSON value into a NORM document string.
pub fn encode(value: &Value) -> Result<String, NormError>

/// Validate a NORM document string without producing JSON output.
/// Returns all errors found, not just the first.
pub fn validate(input: &str) -> Result<(), Vec<NormError>>
```

No file I/O in the library. Callers handle reading/writing files themselves.

## Error Type

```rust
#[derive(Debug, thiserror::Error)]
pub enum NormError {
    #[error("UTF-8 BOM detected; NORM files must not contain a BOM")]
    BomDetected,

    #[error("null byte at line {line}")]
    NullByte { line: usize },

    #[error("missing root declaration; first non-comment line must be :root or :root[]")]
    MissingRootDeclaration,

    #[error("invalid root declaration at line {line}")]
    InvalidRootDeclaration { line: usize },

    #[error("invalid section name {name:?} at line {line}")]
    InvalidSectionName { line: usize, name: String },

    #[error("duplicate section name {name:?} at line {line}")]
    DuplicateSectionName { line: usize, name: String },

    #[error("section {name:?} is unreachable from the root")]
    UnreachableSection { name: String },

    #[error("duplicate pk {pk} at line {line}")]
    DuplicatePk { line: usize, pk: String },

    #[error("invalid pk value {value:?} at line {line} (leading zeros not permitted)")]
    InvalidPk { line: usize, value: String },

    #[error("unresolved reference {reference:?} at line {line}")]
    UnresolvedReference { line: usize, reference: String },

    #[error("circular reference detected involving {reference:?}")]
    CircularReference { reference: String },

    #[error(":root document has multiple rows in its root section (line {line})")]
    MultipleRootRows { line: usize },

    #[error(":root declaration but root section is an array section at line {line}")]
    ArraySectionAsRoot { line: usize },

    #[error("empty row in array section at line {line}")]
    EmptyArrayRow { line: usize },

    #[error("encoder: root JSON value must be an object or array, not a scalar")]
    ScalarRoot,
}
```

`parse` and `encode` return the first error encountered. `validate` collects all errors and returns them together.

## Directory Structure

```
norm-codec/
  Cargo.toml
  src/
    lib.rs          public API: re-exports parse, encode, validate and NormError
    error.rs        NormError enum
    lexer.rs        line-level tokeniser: produces a stream of Token
    document.rs     internal Document, Section, Row types (first-pass output)
    parser.rs       NORM string → Document (pass 1) → serde_json::Value (pass 2)
    encoder.rs      serde_json::Value → NORM string
    bin/
      main.rs       CLI
  tests/
    parse.rs        integration tests: valid documents
    encode.rs       integration tests: valid JSON input
    roundtrip.rs    round-trip tests: encode→parse and parse→encode
    errors.rs       one test per MUST-reject rule
    fixtures/       .norm and .json files used by integration tests
```

## Internal Architecture

### Lexer (lexer.rs)

Detects BOM and null bytes upfront. Then processes the input as a stream of tokens.

Structural lines (root declarations, section headers, blanks, comments) are identified by inspecting the first non-whitespace characters of each line. CRLF is stripped. Inline comments on structural lines are stripped before classification.

Data rows use `csv-core` for cell splitting. The csv-core state machine handles quoted fields with embedded newlines, doubled-quote escaping, and commas inside quotes. This means a single data row may span multiple lines in the source text — the lexer feeds input to csv-core and accumulates cells until a record boundary is reached.

`#` on a data row is not treated as a comment — it is literal cell content.

Note: csv-core never returns errors; it always produces some output. The lexer must layer its own validation (e.g. detecting malformed quoting) where strict spec compliance requires it.

```rust
enum Token {
    RootDeclaration { array: bool },
    SectionHeader { name: String, array: bool },
    DataRow { cells: Vec<String> },
    Blank,
    Comment,
}
```

Expose as a `pub(crate)` function so parser tests can verify tokenisation independently:

```rust
pub(crate) fn lex(input: &str) -> Result<Vec<(usize, Token)>, NormError>
```

Note: tokens own their data (`String` instead of `&'a str`) because csv-core writes unescaped field content into a caller-provided buffer, so cells cannot borrow from the original input.

### Document (document.rs)

Internal representation produced by pass 1 of the parser. All types are `pub(crate)` so unit tests within the crate can construct them directly for pass 2 testing, while integration tests in `tests/` go through the public API only.

```rust
pub(crate) struct Document {
    pub(crate) root_array: bool,
    pub(crate) sections: Vec<Section>,
}

pub(crate) struct Section {
    pub(crate) name: String,
    pub(crate) array: bool,
    pub(crate) header: Vec<String>,  // empty for array sections
    pub(crate) rows: Vec<Row>,
}

pub(crate) struct Row {
    pub(crate) line: usize,
    pub(crate) cells: Vec<String>,
}
```

### Parser (parser.rs)

Expose each pass as a separate `pub(crate)` function for independent testing:

```rust
pub(crate) fn collect_document(input: &str) -> Result<Document, NormError>  // pass 1
pub(crate) fn resolve(doc: &Document) -> Result<Value, NormError>           // pass 2
```

Pass 1 can be tested by verifying the `Document` structure without checking JSON output. Pass 2 can be tested by constructing `Document` values directly without parsing NORM text.

Pass 1: lex the input, collect all sections into a Document. Validate structure during collection:
- Root declaration present and valid
- Section names valid and unique
- pk values present where required, globally unique, no leading zeros
- Array sections contain no empty rows
- :root with array section as first section → error
- :root with multiple rows → error

Pass 2: starting from the root section, resolve references and reconstruct serde_json::Value.
- Detect unresolved references
- Detect circular references (track the resolution chain; error if a row or section is visited twice)
- Detect unreachable sections (any section not visited during pass 2)

Value mapping — extract as a standalone `pub(crate)` function for focused unit testing of quoting edge cases (`"42"` vs `42`, `"@tags"` vs `@tags`, `""` vs empty cell):

```rust
pub(crate) fn map_cell(cell: &str) -> CellValue
```

Mapping rules:
- Quoted string → JSON string (strip outer quotes, unescape doubled quotes)
- Bare number (JSON number syntax) → JSON number
- `true` / `false` → JSON boolean
- `null` → JSON null
- Empty cell → key omitted from object
- `@N` → resolve row by pk, reconstruct as JSON object
- `@name` → resolve section, reconstruct as JSON array (table) or JSON array (array section)
- `@[]` → empty JSON array

### Encoder (encoder.rs)

Traverses a serde_json::Value tree depth-first.

Steps follow the JSON to NORM conversion rules in the spec:
1. Reject scalar root
2. Emit :root or :root[] declaration
3. Traverse the tree; identify nested objects and arrays
4. Assign globally unique pk values (incrementing integer starting at 1) to rows referenced by @N
5. Derive section names from JSON keys; if a key is not a valid section name, sanitise it
   by replacing invalid characters with underscores and prepending an underscore if the
   first character is a digit
6. Emit root section, then nested table sections, then array sections
7. Quote string values that would be ambiguous unquoted (numbers, booleans, null, @-prefixed)

## CLI

Binary name: `norm`

Subcommands:

```
norm parse [FILE]
norm encode [FILE]
norm validate [FILE]
```

All subcommands read from FILE if provided, or from stdin if omitted.
All output goes to stdout. Errors go to stderr.

### norm parse

Reads a NORM document, outputs JSON to stdout.

Options:
  --compact    Output compact JSON (default is pretty-printed)

Exit codes:
  0  success
  1  parse error (error message on stderr)

### norm encode

Reads a JSON document, outputs a NORM document to stdout.

Exit codes:
  0  success
  1  encode error (error message on stderr)

### norm validate

Validates a NORM document. Prints all errors to stderr, one per line, with line numbers.

Exit codes:
  0  valid
  1  one or more validation errors

## Dependencies

```toml
[features]
default = []
cli = ["dep:clap"]

[dependencies]
serde_json = "1"
thiserror = "2"
csv-core = "0.1"
clap = { version = "4", features = ["derive"], optional = true }

[dev-dependencies]
# none required; use std test harness

[[bin]]
name = "norm"
path = "src/bin/main.rs"
required-features = ["cli"]
```

`csv-core` is used by the lexer for CSV cell splitting (handles quoted fields, embedded newlines, escaped quotes). `clap` is feature-gated behind `cli`. Library consumers never pull in clap. The binary requires `--features cli` to build or install. Library code must not import `clap`.

## Testing Strategy

### Unit tests

Each module has `#[cfg(test)]` blocks covering its internal logic:
- lexer: token classification for each line type, BOM detection, CRLF stripping
- parser pass 1: each structural validation rule
- parser pass 2: each reference resolution case and error case
- encoder: value quoting logic, section name sanitisation

### Integration tests (tests/)

`parse.rs` — valid NORM documents produce the expected JSON value.
Use the normative examples from spec.md as fixtures. Each spec example becomes one test.

`encode.rs` — valid JSON values produce a NORM document that round-trips back to the same JSON.

`roundtrip.rs` — encode(parse(norm_text)) produces a document that parses back to the same Value.
Use solar_system and other spec fixtures as round-trip inputs.

`errors.rs` — one test per MUST-reject rule in the spec:
- BOM detected
- Null byte present
- Invalid root declaration
- Invalid section name
- Duplicate section name
- Unreachable section
- pk with leading zero
- Duplicate pk across sections
- Unresolved @N reference
- Unresolved @name reference
- Circular reference
- :root with multiple rows
- :root with array section as first section
- Empty row in array section
- Encoder rejects scalar root

### Fixtures (tests/fixtures/)

Copy fixture pairs from ../norm-spec/fixtures/ (11 .norm/.json pairs).
Add a .norm file for each error test case.

Caution: the spec fixtures are hand-written and have not been validated by a known-good implementation. If a test fails against a fixture, verify the fixture itself against the spec before assuming a bug in our code.

## Spec Compliance Notes

The following behaviours are specified by the spec but worth calling out for implementors:

- `pk` column is structural: exclude it from reconstructed JSON objects
- Duplicate `pk` column name is permitted when a JSON object has a key literally named `pk`;
  the first `pk` column is structural, the second is data
- Empty cell and empty string `""` are distinct: empty cell omits the key, `""` is a JSON empty string
- `#` in a data row is a literal character, not a comment
- Section names match `[a-zA-Z_][a-zA-Z0-9_]*`; names matching `r` followed only by digits
  are technically valid but reserved-looking — no special treatment required
- Forward references are valid; parsers must collect all sections before resolving references
- Circular references must be detected and rejected

## Current State

Status: planning complete, implementation not started.

Reviewed: 2026-04-08

Toolchain: Rust 1.94.1 (Arch Linux pacman package).

Dependencies verified (all latest majors match the plan):
- serde_json 1.0.149 (plan: "1")
- thiserror 2.0.18 (plan: "2")
- clap 4.6.0 (plan: "4")
- csv-core 0.1.13 (plan: "0.1")

No source code, tests, Cargo.toml, or fixture files exist yet. The repository contains only project documentation (project.md, AGENTS.md, LICENSE, .gitignore).

The norm-spec repo at ../norm-spec/ provides:
- spec.md (v0.1) — authoritative specification, 259 lines
- fixtures/ — 11 paired .norm/.json fixture files ready to copy into tests/fixtures/:
  array_root, object_root, comments, csv_escaping, empty_structures, heterogeneous, nested_arrays, pk_collision, quoting, references, solar_system

The solar_system fixture pair (324-line NORM, 21 KB JSON) serves as the comprehensive round-trip test.

Spec and fixtures have been read and cross-checked. The project plan aligns with the spec. No discrepancies found between the planned architecture, error variants, public API, and the spec's requirements.
