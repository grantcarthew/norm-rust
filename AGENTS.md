# norm-codec: Rust NORM Format Parser and Encoder

Rust crate implementing a parser and encoder for the NORM data format (v0.1).
Spec: `../norm-spec/spec.md` — read it before working on parsing or encoding logic.

Crate: `norm-codec` | Binary: `norm` | License: MPL-2.0 | Edition: 2021

## Commands

```sh
cargo build              # build lib + binary
cargo build --release    # release build
cargo test               # run all tests
cargo test --test parse  # run one integration test file
cargo clippy             # lint
cargo fmt                # format
```

## Project Layout

```
norm-codec/
  Cargo.toml
  src/
    lib.rs       public API re-exports
    error.rs     NormError enum
    lexer.rs     line-level tokeniser
    document.rs  internal Document, Section, Row types
    parser.rs    NORM string → Document → serde_json::Value
    encoder.rs   serde_json::Value → NORM string
    bin/
      main.rs    CLI (clap)
  tests/
    parse.rs     valid NORM → expected JSON
    encode.rs    valid JSON → round-trips back
    roundtrip.rs encode→parse and parse→encode
    errors.rs    one test per MUST-reject rule
    fixtures/    .norm and .json files
```

## Public API

```rust
pub fn parse(input: &str) -> Result<Value, NormError>
pub fn encode(value: &Value) -> Result<String, NormError>
pub fn validate(input: &str) -> Result<(), Vec<NormError>>
```

No file I/O in the library. `parse` and `encode` return on first error. `validate` collects all errors.

## Dependencies

```toml
[dependencies]
serde_json = "1"
thiserror = "2"
clap = { version = "4", features = ["derive"] }
```

`clap` is used only in `src/bin/main.rs`. Do not import it in library code.

## CLI

```
norm parse [--compact] [FILE]    # NORM → JSON (pretty by default)
norm encode [FILE]               # JSON → NORM
norm validate [FILE]             # validate, print all errors to stderr
```

All subcommands read from `FILE` or stdin if omitted. Output to stdout. Errors to stderr.
Exit code 0 = success, 1 = error.

## Architecture

### Lexer (lexer.rs)

Processes input line by line. Detects BOM and null bytes. Strips CRLF.
Classifies lines into `Token`:

```rust
enum Token<'a> {
    RootDeclaration { array: bool },
    SectionHeader { name: &'a str, array: bool },
    DataRow { cells: Vec<&'a str> },
    Blank,
    Comment,
}
```

Inline comments stripped on structural lines only. `#` in data rows is literal.

### Parser (parser.rs) — two passes

Pass 1: lex input, collect all sections into `Document`. Validate:
- Root declaration present and valid
- Section names valid (`[a-zA-Z_][a-zA-Z0-9_]*`) and unique
- pk values globally unique, no leading zeros
- Array sections contain no empty rows
- `:root` with array section as first section → error
- `:root` with multiple rows → error

Pass 2: resolve references starting from root section. Detect unresolved refs, circular refs, unreachable sections.

### Encoder (encoder.rs)

Traverses `serde_json::Value` depth-first. Rejects scalar root. Assigns pk values starting at 1. Sanitises section names derived from JSON keys (replace invalid chars with `_`, prepend `_` if first char is a digit).

## Key Spec Behaviours

- `pk` column is structural: exclude it from reconstructed JSON objects
- A JSON object with a key literally named `pk` produces two `pk` columns; only the first is structural
- Empty cell = absent key (not null); `""` = JSON empty string — these are distinct
- `#` in a data row is a literal character, not a comment
- Forward references are valid; collect all sections before resolving
- Circular references must be detected and rejected
- Parsers must reject UTF-8 BOM, null bytes, invalid pk (leading zeros), duplicate pk across all sections

## Error Variants

| Variant | Trigger |
|---------|---------|
| `BomDetected` | UTF-8 BOM at start of file |
| `NullByte { line }` | null byte anywhere in file |
| `MissingRootDeclaration` | first non-comment line is not `:root` or `:root[]` |
| `InvalidRootDeclaration { line }` | malformed root declaration |
| `InvalidSectionName { line, name }` | name not matching `[a-zA-Z_][a-zA-Z0-9_]*` |
| `DuplicateSectionName { line, name }` | name reused |
| `UnreachableSection { name }` | section not visited during pass 2 |
| `DuplicatePk { line, pk }` | same pk value in two table sections |
| `InvalidPk { line, value }` | leading zero in pk |
| `UnresolvedReference { line, reference }` | `@N` or `@name` with no match |
| `CircularReference { reference }` | resolution chain revisits a row or section |
| `MultipleRootRows { line }` | `:root` with more than one data row |
| `ArraySectionAsRoot { line }` | `:root` but first section is `name[]` |
| `EmptyArrayRow { line }` | blank row inside an array section |
| `ScalarRoot` | encoder receives a scalar JSON value |

## Testing

Integration test files in `tests/` use fixtures from `tests/fixtures/`. Each normative spec example maps to a fixture. `errors.rs` has one test per MUST-reject rule — do not merge or combine them.

When adding a new MUST-reject rule, add a corresponding fixture `.norm` file and a dedicated `#[test]` in `errors.rs`.

Use `cargo test <test_name>` to run a single test by name filter.
