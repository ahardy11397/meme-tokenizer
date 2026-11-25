# meme-tokenizer

A complete `.meme` source tokenizer CLI written entirely in Memelang (self-hosting step 1: the first real compiler component written in Memelang, compiled to a dependency-free native binary via `meme build`). Demonstrates self-tokenization of its own source code.

## 1. CLI Usage

Build the native executable using `meme build`:
```bash
uv run --project /mnt/ssd/projects/meme meme build main.meme -o tokenizer_bin
```

Run subcommands via `./tokenizer_bin`:
```bash
# Token stream as JSON to file (fails exit 1 + one-line error if file is missing/unreadable)
./tokenizer_bin tokenize <file.meme> -o <out.json>

# Human-readable table: LINE COL KIND VALUE (one token per line)
./tokenizer_bin tokens <file.meme>

# Summary: total tokens, lines, per-kind counts
./tokenizer_bin count <file.meme>
```

Unknown subcommands or incorrect argument counts print usage instructions and terminate with exit code 1.

## 2. Token Model and Kinds

The tokenizer uses the record definition:
```meme
record Token:
    kind: String
    value: String
    line: Int
    col: Int
```
- `line`: 1-based line number.
- `col`: 0-based byte offset within the line (reset to 0 after `\n`; tabs advance byte offset by 1).

Supported token kinds:
- `comment`: `#` to end of line (token value includes the `#` prefix).
- `string`: double-quoted string literals with escapes (`\"`, `\\`, `\n`). Token value includes the surrounding quotes.
- `int`: consecutive ASCII digits (`0`–`9`).
- `float`: consecutive digits containing a `.` followed by digits (e.g. `3.14`).
- `identifier`: ASCII letter or `_` followed by letters, digits, or `_`.
- `keyword`: an identifier present in the curated Memelang keyword set.
- `operator`: multi-character operators (`==`, `!=`, `>=`, `<=`, `->`) scanned greedily, and single-character operators/punctuation (`+ - * / % < > = ( ) [ ] { } , : ; .`).
- `error`: diagnostic token generated when an unexpected character or unterminated string is encountered.

## 3. Keywords List

Keywords are maintained centrally in `keywords.meme`. The curated keyword set includes 36 Memelang core keywords:
- `add`
- `and`
- `assert`
- `call`
- `define`
- `each`
- `else`
- `empty`
- `error`
- `exit`
- `for`
- `from`
- `function`
- `if`
- `in`
- `is`
- `length`
- `not`
- `nothing`
- `of`
- `or`
- `print`
- `print error`
- `read`
- `record`
- `remove`
- `return`
- `set`
- `sort`
- `test`
- `the`
- `to`
- `use`
- `value`
- `while`
- `with`
- `write`

Identifiers matching any entry in this set are classified as `keyword`; all other identifiers are classified as `identifier`.

## 4. Error Mechanism

The tokenizer uses diagnostic `error` tokens (`kind: "error"`) for lexer failures:
- When an unexpected character (e.g. `@`, `$`) or an unterminated string literal is encountered, `tokenize()` appends a `Token(kind="error", value=..., line=..., col=...)` and immediately returns the token list.
- Callers (such as `main.meme`) check for error tokens using `token_has_error()`. When present, `main` writes the exact error message to standard error (`print error`) and halts with `exit with code 1`.
- For file I/O errors, `tokenize_file(path)` returns `Optional<List<Token>>` (`nothing` when the file cannot be read). `main` reports `Error: cannot read file: <path>` to `stderr` and exits with code 1.
- In `count`, line count is calculated from split lines, and if the file cannot be read, an error is printed to stderr and the program exits with code 1.

## 5. The Int/Float Dot Rule

Float literals require digits **both** before and after the decimal point:
- `42` -> token of kind `int` with value `"42"`
- `3.14` -> token of kind `float` with value `"3.14"`
- `1.` -> token of kind `int` with value `"1"`, followed by an `operator` token `"."` (because no digits immediately follow the `.`).
- `1.foo` -> `int` `"1"`, `operator` `"."`, `identifier` `"foo"`.
- `.42` -> `operator` `"."` followed by `int` `"42"`.
- `3.14.15` -> `float` `"3.14"`, `operator` `"."`, `int` `"15"`.

This rule ensures that property accesses or method calls following numeric expressions are never misinterpreted as malformed floats.

## 6. Test Instructions

Individual test suites reside in `tests/meme/` and are executed using `meme test`:

```bash
# Run individual test files under python
uv run --project /mnt/ssd/projects/meme meme test tests/meme/01_keywords_test.meme --target python
uv run --project /mnt/ssd/projects/meme meme test tests/meme/02_numbers_test.meme --target python
uv run --project /mnt/ssd/projects/meme meme test tests/meme/03_strings_test.meme --target python
uv run --project /mnt/ssd/projects/meme meme test tests/meme/04_comments_test.meme --target python
uv run --project /mnt/ssd/projects/meme meme test tests/meme/05_operators_test.meme --target python
uv run --project /mnt/ssd/projects/meme meme test tests/meme/06_position_test.meme --target python
uv run --project /mnt/ssd/projects/meme meme test tests/meme/07_io_test.meme --target python
uv run --project /mnt/ssd/projects/meme meme test tests/meme/08_fixture_test.meme --target python

# Run on other backends (rust, c, cpp, go, ts)
uv run --project /mnt/ssd/projects/meme meme test tests/meme/01_keywords_test.meme --target c
uv run --project /mnt/ssd/projects/meme meme test tests/meme/08_fixture_test.meme --target rust
```
