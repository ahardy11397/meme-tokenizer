# PLAN — meme-tokenizer: a Memelang graduate-level demo (self-hosting step 1)

Build a complete .meme source tokenizer CLI entirely in Memelang. This is
self-hosting step 1: the first real compiler component written in Memelang,
compiled to a dependency-free native binary via `meme build`. Demonstration
goal: it tokenizes its own source.

Language docs: /mnt/ssd/projects/meme/docs/WRITING_GUIDE.md and
docs/TUTORIAL.md (read both first). Compiler commands run via
`uv run --project /mnt/ssd/projects/meme meme ...` from this repo.

## Ground rules (non-negotiable)

- Work ONLY in this repo. NO git commits — leave changes for review.
- 100% Memelang for program logic; no other languages.
- All persistence as JSON in `data/` (created on demand) using the Optional
  record readers/writers; corrupt/missing file = clean Optional handling,
  never a traceback.
- Every construct must parse with `--provider heuristic`.
- Run all test suites in the FOREGROUND only.
- Check the memelang-dev skill's references/language-limitations.md before
  designing; design around known limitations, don't rediscover them.
- NO new language features. If the language genuinely lacks something this
  program needs, STOP that thread, record the gap precisely (what you tried,
  exact error), route around it if possible, and report it in the final
  summary. Do not hack around a gap silently.

## Domain and CLI (thin argv-router main + logic modules; unknown command
-> usage + exit 1; every error = one clear line + exit 1)

    tokenize <file.meme> -o <out.json>   # token stream as JSON to file (exit 1 + one-line error if file missing/unreadable)
    tokens <file.meme>                   # human-readable table: LINE COL KIND VALUE, one token per line
    count <file.meme>                    # summary: total tokens, lines, per-kind counts

Token model: `record Token: kind: String, value: String, line: Int, col: Int`
(1-based line, 0-based col as byte offset in the line).

Kinds and rules (all deterministic, ASCII):
- whitespace: skipped; newline advances line, resets col.
- `#` comment: from `#` to end of line -> kind `comment` (content includes #).
- String literal: double quotes with escapes `\"`, `\\`, `\n` -> kind
  `string`. Unterminated string = error: one clear line to stderr + exit 1.
- Int literal: digits -> kind `int`. Float: digits with a `.` between digits
  -> kind `float`.
- Identifier: letter or `_` then letters/digits/`_` -> kind `identifier`;
  if it is in the keyword set -> kind `keyword`.
- Keyword set: a fixed documented Set<String> of Memelang core keywords
  (function, return, if, else, for, each, in, define, set, to, print, print
  error, record, test, assert, use, nothing, is, not, and, or, call, exit,
  with, from, of, the, value, empty, add, remove, sort, read, write, length,
  while). Keep the exact list in ONE place (keywords module) and document it
  in the README; being a curated subset is fine and expected.
- Operators/punctuation: multi-char first (`==`, `!=`, `>=`, `<=`, `->`),
  then single char (`+ - * / % < > = ( ) [ ] { } , : ; .`) -> kind `operator`.
- Any other character: error, one clear line to stderr + exit 1
  (name the char and line).

Architecture: `main.meme` = thin argv router (argument count / argument 1);
`lexer.meme` = core scan: `tokenize(text: String) -> List<Token>` char loop
using `length of string`, `char at i of`, `number of <char>`, plus
`tokenize_file(path) -> Optional<List<Token>>` (Optional file readers;
missing file = nothing -> main prints one-line error + exit 1);
`keywords.meme` = the keyword Set + `classify(word) -> String`.
Errors inside the lexer: return Optional<List<Token>> (nothing = error
already printed) or build a Token with kind `error` — pick ONE mechanism,
document it in the README, and be consistent.

## Tests — tests/meme/*.meme, `meme test`

- Empty input -> empty token list.
- Keywords vs identifiers (e.g. `function` vs `functions` vs `_fn2`).
- Ints and floats (`42`, `3.14`, `1.` is NOT a valid float -> operator `.`
  after int `1`; document chosen rule in README and test it).
- Strings: plain, with `\"` and `\\` escapes, empty string; unterminated ->
  error path.
- Comments: full-line and trailing; `#` inside a string is NOT a comment.
- Operators: every multi-char operator recognized greedily (`->` is one
  token, not `-` + `>`); `>=` not `>` + `=`.
- Line/col tracking across multi-line input with tabs.
- `tokenize_file` on a missing path -> nothing (no traceback).
- Self-tokenize fixture: a small literal .meme snippet (fixed string) whose
  expected token list is asserted exactly.
- Fixed scratch files under tests/tmp/, unique per test, cleaned up.

## Acceptance ("done means")

1. `meme test` python target — 0 failed.
2. Same `--target rust cpp go c ts` — ALL SIX green.
3. `meme build main.meme -o tokenizer_bin` — end-to-end demo transcript in
   the final summary, including `./tokenizer_bin tokenize main.meme -o
   data/self.json` (the tokenizer tokenizing its own source) and
   `./tokenizer_bin count main.meme`.
4. README.md: CLI usage, token kinds + full keyword list, error mechanism,
   the int/float dot rule, test instructions.
5. Final message: files, test count per target, demo transcript,
   deviations (if any) with reasons.
