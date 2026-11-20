# EditorConfig recipes

Use these as a starting point, then match the repository's actual formatters.

## Contents

- [Polyglot baseline](#polyglot-baseline)
- [JavaScript and TypeScript](#javascript-and-typescript)
- [Python](#python)
- [Rust](#rust)
- [Go](#go)
- [Java, Kotlin, and C#](#java-kotlin-and-c)
- [C and C++](#c-and-c)
- [Ruby and PHP](#ruby-and-php)
- [YAML, JSON, and TOML](#yaml-json-and-toml)
- [Markdown](#markdown)
- [Makefile](#makefile)
- [Shell and Dockerfile](#shell-and-dockerfile)
- [Terraform and HCL](#terraform-and-hcl)
- [Windows scripts](#windows-scripts)
- [Monorepos and nested configs](#monorepos-and-nested-configs)

## Polyglot baseline

```ini
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

[Makefile]
indent_style = tab

[*.md]
trim_trailing_whitespace = false
```

## JavaScript and TypeScript

```ini
[*.{js,jsx,mjs,cjs,ts,tsx,mts,cts}]
indent_style = space
indent_size = 2

[*.{json,jsonc}]
indent_style = space
indent_size = 2
```

- Prettier's `tabWidth` and `useTabs` must match. If Prettier uses 2 spaces, do not set 4 here.
- TypeScript/ESLint's `indent` rule should agree; many projects disable the ESLint indent rule and let Prettier own formatting.
- `end_of_line = lf` here should match Prettier's `endOfLine` (`lf`, `crlf`, or `auto`).

## Python

```ini
[*.{py,pyi}]
indent_style = space
indent_size = 4
```

- PEP 8 and Black use 4 spaces.
- Black's line length is separate; EditorConfig's core spec has no `max_line_length`.
- Do not set tabs for Python; mixed tabs/spaces are a syntax hazard.

## Rust

```ini
[*.rs]
indent_style = space
indent_size = 4
```

- `rustfmt` defaults to `tab_spaces = 4` and `hard_tabs = false`.
- If the project sets `hard_tabs = true` in `rustfmt.toml`, match it with `indent_style = tab`.

## Go

```ini
[*.go]
indent_style = tab
```

- `gofmt` uses tabs for indentation. Do not set spaces.
- `indent_size` is not meaningful for gofmt; leave it at the default or set a team tab width if an editor needs one.
- `goimports`/`gofumpt` inherit the same tab convention.

## Java, Kotlin, and C#

```ini
[*.{java,kt,kts,cs}]
indent_style = space
indent_size = 4
```

- Match the project's formatter (google-java-format, ktlint, dotnet format). C# conventions commonly use 4 spaces; some teams use tabs.
- If a `.editorconfig` already exists for a .NET repository, preserve its `dotnet_*` keys even though they are not part of the EditorConfig core spec; they are used by the .NET tooling.

## C and C++

```ini
[*.{c,h,cc,cpp,cxx,hpp,hxx}]
indent_style = space
indent_size = 2
```

- Google/LLVM styles commonly use 2 spaces; many projects use 4. Match `.clang-format` (`IndentWidth`, `UseTab`).
- Do not guess; run the formatter on a sample file and align.

## Ruby and PHP

```ini
[*.rb]
indent_style = space
indent_size = 2

[*.php]
indent_style = space
indent_size = 4
```

- Ruby convention is 2 spaces; RuboCop can enforce it.
- PHP PSR-12 uses 4 spaces. Match the project's PHP-CS-Fixer/PHP_CodeSniffer config.

## YAML, JSON, and TOML

```ini
[*.{yml,yaml}]
indent_style = space
indent_size = 2

[*.{json,jsonc}]
indent_style = space
indent_size = 2

[*.toml]
indent_style = space
indent_size = 2
```

- YAML forbids tab indentation; never set tabs for YAML.
- Some JSON formatters use 4 spaces; match the project. Most JavaScript ecosystems use 2.
- TOML has no universal width; 2 spaces is common. Match `taplo`/formatter settings.

## Markdown

```ini
[*.md]
trim_trailing_whitespace = false
```

- Two trailing spaces create a hard line break in Markdown; trimming them changes rendering.
- Optionally set `indent_size = 2` for list indentation; some styles use 4.
- Do not enforce `insert_final_newline = false` for Markdown; a final newline is fine.
- MDX may need the same exception; add `[*.mdx]` if the project uses it.

## Makefile

```ini
[Makefile]
indent_style = tab

[*.mk]
indent_style = tab
```

- Recipe lines require a literal tab. An editor that converts tabs to spaces breaks the Makefile.
- Do not apply `trim_trailing_whitespace` in a way that removes meaningful trailing tabs; the core property is line-end whitespace, and Makefile recipes do not normally rely on trailing whitespace.

## Shell and Dockerfile

```ini
[*.{sh,bash,zsh}]
indent_style = space
indent_size = 2

[Dockerfile]
indent_style = space
indent_size = 2

[*.dockerfile]
indent_style = space
indent_size = 2
```

- Shell style varies; Google's shell style uses 2 spaces, many projects use 4. Match `shfmt` (`-i`).
- Dockerfile leading whitespace is generally ignored, but consistent indentation helps readability.

## Terraform and HCL

```ini
[*.{tf,tfvars,hcl}]
indent_style = space
indent_size = 2
```

- `terraform fmt` uses 2 spaces.
- Do not set tabs; `terraform fmt` will rewrite them.

## Windows scripts

```ini
[*.{bat,cmd}]
end_of_line = crlf

[*.{ps1,psm1,psd1}]
end_of_line = crlf
```

- `.bat`/`.cmd` are commonly CRLF; some modern Windows tooling tolerates LF, but CRLF is the safer default for cmd.exe.
- PowerShell supports LF, but Windows PowerShell 5.1 and some tools expect CRLF. Choose based on the project's CI and signing/packaging requirements.
- Pair with `.gitattributes` if Git should normalize these files.

## Monorepos and nested configs

Root `.editorconfig`:

```ini
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true
indent_style = space
indent_size = 2
```

`services/api/.editorconfig` for a Go service:

```ini
[*.go]
indent_style = tab
```

`packages/python-lib/.editorconfig` for a Python package:

```ini
[*.{py,pyi}]
indent_style = space
indent_size = 4
```

- The nested file inherits the root and overrides only what differs.
- Do not repeat `root = true` in every package; that stops inheritance and creates drift.
- A nested config applies by path, regardless of package boundaries. Place it at the directory that should own the override.
