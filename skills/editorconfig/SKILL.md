---
name: editorconfig
description: >-
  Use this skill to create or fix `.editorconfig` files from a repository's actual languages and tooling: indentation, line endings, charset, trailing whitespace, final newlines, glob sections, monorepo overrides, exceptions for Markdown/Makefile/Go, and alignment with Prettier, rustfmt, gofmt, Black, and other formatters. Trigger it when the user mentions EditorConfig, `.editorconfig`, consistent indentation, line endings, tabs vs spaces, or editor settings that disagree across IDEs. Chinese cues — EditorConfig, 编辑器配置, 缩进, 换行符, 制表符, 空格, 跨编辑器一致. Skip formatter configuration itself, ESLint/Prettier rule design, and `.gitattributes`-only work.
---

# EditorConfig

EditorConfig is a small file format that tells editors and IDEs how to handle indentation, line endings, charset, trailing whitespace, and final newlines for matching files. It is not a formatter and it is not enforced by Git: a plugin must be installed, and a formatter/linter may still disagree.

The goal is a baseline every editor agrees on, with narrow exceptions where a language or tool requires them.

## Workflow

1. **Inspect the repository** — file types, package manifests, formatter configs, existing `.editorconfig`, CI, and platform-specific files.
2. **Set a root baseline** — `root = true`, UTF-8, LF, final newline, trim trailing whitespace, and a default indentation.
3. **Add language sections** — override indentation for Python, Go, Rust, Makefile, Markdown, YAML, and other file types.
4. **Respect formatters** — align EditorConfig values with Prettier/rustfmt/gofmt/Black; do not fight them.
5. **Handle nested configs** — use a root file plus package-level overrides only where genuinely different.
6. **Validate** — parse the file, run `editorconfig-checker` or an equivalent check, and test in at least two editors.

## Baseline

```ini
# .editorconfig
root = true

[*]
charset = utf-8
end_of_line = lf
indent_style = space
indent_size = 2
insert_final_newline = true
trim_trailing_whitespace = true

[*.{js,jsx,ts,tsx,json,jsonc,css,scss,html,vue,svelte}]
indent_size = 2

[*.{py,pyi}]
indent_size = 4

[*.{rs,java,kt,kts,cs}]
indent_size = 4

[*.go]
indent_style = tab
indent_size = 4

[Makefile]
indent_style = tab

[*.md]
trim_trailing_whitespace = false
```

Rules:

- The filename is `.editorconfig` in lowercase. The file is UTF-8, with LF or CRLF line endings.
- `root = true` stops the search at this directory. Use it at the repository root; nested configs should not repeat it unless they are independent roots.
- Comments start with `;` or `#` **at the beginning of a line**. Inline comments are not part of the 0.15+ spec; `value # text` is the value.
- Section names are file globs. Later matching sections override earlier ones; closer `.editorconfig` files override parent files.
- `*` does not cross `/`; `**` does. A section without `/` matches at any depth below the file.
- `indent_size = tab` uses `tab_width` (or the editor's tab size) for the width of a tab.
- `unset` removes a property inherited from a parent section/file.

## Language defaults

| Language/file | Indentation | Notes |
| --- | --- | --- |
| JavaScript/TypeScript/JSON/CSS/HTML | 2 spaces | Match Prettier `tabWidth`/`useTabs`. |
| YAML | 2 spaces | YAML forbids tabs. |
| Python | 4 spaces | PEP 8; `black` uses 4. |
| Rust | 4 spaces | `rustfmt` default `tab_spaces = 4`. |
| Java/Kotlin/C# | 4 spaces | Match the project formatter. |
| Go | tabs | `gofmt` uses tabs; set `indent_style = tab`. |
| Makefile | tabs | Recipe lines require tabs; never convert them to spaces. |
| Markdown | 2 spaces (optional) | `trim_trailing_whitespace = false` because trailing spaces can be hard breaks. |
| Shell | 2 spaces (common) | Check the project's shfmt/style. |
| Dockerfile | 2 spaces (common) | Docker ignores leading whitespace; match the project. |
| Terraform/HCL | 2 spaces | `terraform fmt` uses 2. |

These are defaults, not laws. If the repository's formatter uses a different width, match the formatter.

## Endings and charset

- Prefer `end_of_line = lf` for portable source. Windows-only `.bat`/`.cmd` files often need `crlf`:

  ```ini
  [*.{bat,cmd}]
  end_of_line = crlf
  ```

- `charset = utf-8` is the safe default. `utf-8-bom` is discouraged; use it only when a Windows tool requires a BOM.
- EditorConfig does not normalize existing files or Git history. Pair line-ending rules with `.gitattributes` (`* text=auto eol=lf`, plus explicit `eol=crlf` for Windows scripts) if the repository needs Git-level normalization.
- `insert_final_newline = true` avoids missing-newline diffs; empty files are not forced to gain a newline by conforming editors.
- `trim_trailing_whitespace = true` is a good default, with exceptions for Markdown and whitespace-sensitive fixtures.

## Monorepos

- Put `root = true` in the repository root.
- Add package-level `.editorconfig` files only when a package genuinely uses a different style (for example, a Go service in a mostly TypeScript monorepo).
- Nested files inherit the parent and only override what differs.
- Do not repeat every property in every nested file; it makes drift harder to see.
- EditorConfig is path-based, not workspace-aware; a nested config applies to files under its directory, regardless of package boundaries.

## Formatters and linters

EditorConfig is a baseline, not a replacement:

| Tool | Align with |
| --- | --- |
| Prettier | `tabWidth`, `useTabs`, `endOfLine` in `.prettierrc` |
| ESLint | `indent` rule (if used) and parser options |
| rustfmt | `tab_spaces`, `hard_tabs` in `rustfmt.toml` |
| gofmt | tabs; no width option |
| Black | 4 spaces; `line-length` is separate |
| shfmt | `-i`, `-ci` flags/config |
| terraform fmt | 2 spaces; no width option |

If EditorConfig says 2 spaces and Prettier says 4, editors and CI will disagree. Pick one source of truth for formatting and make EditorConfig agree.

`max_line_length` is **not** part of the EditorConfig 0.17.2 core property list. Some editors/plugins support it, but do not rely on it for enforcement; use the formatter/linter's line-length setting.

## Reference map

| Task | Read |
| --- | --- |
| Spec details: globs, precedence, properties, comments, `unset` | [`references/syntax.md`](references/syntax.md) |
| Language and project recipes, Markdown/Makefile/Go exceptions, monorepos | [`references/recipes.md`](references/recipes.md) |
| `editorconfig-checker`, CI/pre-commit, formatter alignment, troubleshooting | [`references/validation.md`](references/validation.md) |

## Common failures

| Symptom | Cause | Fix |
| --- | --- | --- |
| Settings ignored | No editor plugin, or the file is not named `.editorconfig` | Install the plugin; use the lowercase filename. |
| Parent settings override child | `root = true` in the wrong place, or section order | Put `root = true` at the repo root; later sections win. |
| Makefile breaks | Indentation converted to spaces | `[Makefile] indent_style = tab`. |
| Markdown hard breaks lost | Trailing whitespace trimmed | `[*.md] trim_trailing_whitespace = false`. |
| Go formatting fights the editor | `indent_style = space` | Use `indent_style = tab` for `.go`. |
| Editor and CI disagree | EditorConfig conflicts with Prettier/rustfmt/Black | Align the formatter settings; one source of truth. |
| Inline comment becomes part of the value | EditorConfig 0.15+ has no inline comments | Put comments on their own line. |
| `max_line_length` does nothing | Not a core property in 0.17.2 | Use the formatter/linter setting. |
| Git shows line-ending churn | EditorConfig does not normalize Git | Add `.gitattributes` rules. |

## Official links

- EditorConfig project: <https://editorconfig.org/>
- Specification 0.17.2: <https://spec.editorconfig.org/>
- EditorConfig checker: <https://github.com/editorconfig-checker/editorconfig-checker>
- Prettier end-of-line: <https://prettier.io/docs/options#end-of-line>
