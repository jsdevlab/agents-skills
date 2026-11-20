# EditorConfig syntax and precedence

This reference follows EditorConfig specification 0.17.2: <https://spec.editorconfig.org/>.

## Contents

- [File format](#file-format)
- [Glob expressions](#glob-expressions)
- [Search and precedence](#search-and-precedence)
- [Supported properties](#supported-properties)
- [unset and unknown keys](#unset-and-unknown-keys)
- [Examples](#examples)

## File format

- The filename is `.editorconfig` (all lowercase).
- The file is UTF-8 encoded, with LF or CRLF line separators.
- It is an INI-like format. Each line is one of:
  - blank (ignored);
  - comment starting with `;` or `#`;
  - section header `[glob]`;
  - key-value pair `key = value`.
- Leading/trailing whitespace on a line is trimmed. Keys and values are trimmed; internal whitespace is preserved.
- A key without `=` has an empty value.
- Comments must be on their own line. In 0.15+, `;` or `#` after the start of a line is part of the value, not a comment.
- There is no escaping mechanism for `;` or `#`; avoid values that need them.
- The preamble is everything before the first section. `root` belongs there; all other keys belong under a section.

## Glob expressions

Section names are filepath globs, case-sensitive, using `/` as the path separator (never `\`).

| Pattern | Matches |
| --- | --- |
| `*` | Any string except `/`. |
| `**` | Any string, including `/`. |
| `?` | Any single character except `/`. |
| `[seq]` | Any single character in `seq` (literal characters). |
| `[!seq]` | Any single character not in `seq`. |
| `{s1,s2,s3}` | Any of the strings; can be nested. A single item `{s1}` is literal. |
| `{num1..num2}` | Any integer in the inclusive range; `num1 < num2`. |

- Backslash escapes a special character.
- If a glob contains `/`, it is relative to the directory containing the `.editorconfig` file. `/subdir/*.c` and `subdir/*.c` are equivalent.
- If a glob has no `/`, it matches at any depth below the `.editorconfig`.
- A section name ending in `/` matches no file.
- Section names can contain `[` and `]` inside the outer brackets; the parser uses the first `[` and last `]` on the line.
- Cores must accept section names up to 1024 characters.

Examples:

```ini
[*.js]                 # any .js file at any depth
[lib/**.js]            # .js under lib/ at any depth
[*.{js,jsx,ts,tsx}]    # brace expansion
[{package.json,.npmrc}] # exact filenames
[test{1..3}.py]        # test1.py, test2.py, test3.py
```

## Search and precedence

When an editor opens a file:

1. It searches the file's directory and each parent directory for `.editorconfig`.
2. It stops at the filesystem root or at a file with `root = true`.
3. It applies matching sections from each file.
4. Within one file, later matching sections override earlier ones.
5. Across files, the closer `.editorconfig` (deeper in the tree) takes precedence.

Therefore:

```ini
[*]
indent_size = 2

[*.py]
indent_size = 4
```

A `.py` file gets 4. If a parent says 2 and a child says 4, the child wins.

Use `root = true` at the repository root to avoid inheriting settings from a parent directory outside the project (for example, a home-directory `.editorconfig`).

## Supported properties

| Property | Values | Notes |
| --- | --- | --- |
| `indent_style` | `tab`, `space` | Case-insensitive. |
| `indent_size` | whole number, or `tab` | Width of one indent level / soft tab. `tab` uses `tab_width`. |
| `tab_width` | whole number | Columns per tab; defaults to `indent_size`. |
| `end_of_line` | `lf`, `cr`, `crlf` | Case-insensitive. |
| `charset` | `latin1`, `utf-8`, `utf-8-bom`, `utf-16be`, `utf-16le` | `utf-8-bom` is discouraged. |
| `spelling_language` | `ss` or `ss-TT` | ISO 639/3166; optional spell-check language. |
| `trim_trailing_whitespace` | `true`, `false` | Remove whitespace before newlines. |
| `insert_final_newline` | `true`, `false` | Ensure a final newline; empty files are not forced. |
| `root` | `true` (preamble only) | Stop searching parent directories. |

`max_line_length` is not in the 0.17.2 core list. Some plugins support it, but it is not part of the specification and should not be relied on for enforcement.

## unset and unknown keys

- Any property can be set to `unset` to remove the effect of a value inherited from a parent section or file:
  ```ini
  [vendor/**]
  indent_size = unset
  ```
- Cores must accept and report syntactically valid keys even if the spec does not define them.
- Plugins ignore unrecognized keys and unsupported values; do not assume a non-standard property works in every editor.
- Keys are case-insensitive; all keys are lowercased after parsing.

## Examples

```ini
root = true

[*]
charset = utf-8
end_of_line = lf
indent_style = space
indent_size = 2
insert_final_newline = true
trim_trailing_whitespace = true

[*.{js,jsx,ts,tsx,json,jsonc,css,scss,html}]
indent_size = 2

[*.{py,pyi}]
indent_size = 4

[*.{rs,java,kt,kts,cs}]
indent_size = 4

[*.go]
indent_style = tab

[Makefile]
indent_style = tab

[*.md]
trim_trailing_whitespace = false

[*.{bat,cmd}]
end_of_line = crlf
```

Check the file with a conforming core/checker. Parsing valid INI is not enough; section precedence and glob behavior matter.

```sh
editorconfig-checker
```
