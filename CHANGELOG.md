# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](http://keepachangelog.com/en/1.0.0/)
and this project adheres to [Semantic Versioning](http://semver.org/spec/v2.0.0.html).

## \[Unreleased\]

### Added

- Python 3.14 is now tested and declared as supported. No source changes were needed; the full
  suite passes on 3.14 as-is.
- `BlockView.start_line` and `BlockView.end_line`, so a block's span can be read from the query API without serializing it. `with_meta` puts the numbers in the output dict, which meant reaching them through the label nesting, or through the rule's private `_meta`. Both are `None` for a tree built by the deserializer, which carries no positions. `hq` picks them up through its property accessors: `hq 'resource[*] | .start_line' main.tf`. Thanks, @livingstaccato ([#333](https://github.com/amplify-education/python-hcl2/pull/333))
- `SerializationOptions.metadata_sidecar`, which carries `__is_block__`, `__comments__` and `__inline_comments__` beside the mapping rather than among its keys. HCL reserves none of those names, so a document may declare an attribute called any of them -- and in-band one of the two has to lose: on read the marker overwrites the attribute, on write the deserializer drops it, and by then the dict holds a single value with no way to tell which happened. With the option set, `loads` returns an `HclDict`, a `dict` subclass whose `hcl_meta` holds the three, so the mapping contains attributes and nothing else. `dumps` accepts either form, including a hand-built dict using the old keys. Off by default: the keys are a documented part of the output shape, and JSON cannot carry a sidecar. `HclDict`, `HclMeta` and `meta_of` are exported from `hcl2`. The query views follow the option too: `to_dict` on a block or an attribute view returns an `HclDict`, with any adjacent comments in its metadata. Copying, merging with `|` and pickling carry the metadata; `dict(d)` and `{**d}` deliberately do not, since asking for a `dict` gives the mapping and nothing else. ([#331](https://github.com/amplify-education/python-hcl2/issues/331))

### Changed

- **Breaking for direct `cli.*` imports.** The CLI modules moved from a top-level `cli` package
  to `hcl2.cli`, so installing python-hcl2 no longer claims the generic top-level `cli` name.
  Since 8.0 the distribution installed `cli/` into `site-packages`, where it collided with
  projects that have a top-level `cli` package of their own and made their `cli.*` imports
  resolve to the wrong module. The `hcl2tojson`, `jsontohcl2`, and `hq` commands and
  `python -m hcl2` are unaffected; only code importing `cli.hcl_to_json`, `cli.json_to_hcl`,
  `cli.hq`, or `cli.helpers` needs to add the `hcl2.` prefix. No compatibility shim ships,
  because a shim would still occupy the colliding name.
- The redundant `cli/py.typed` marker is gone; `hcl2/py.typed` already covers `hcl2.cli`.
- `blocks()` and `attributes()` on the query views are annotated as returning `List[BlockView]` and `List[AttributeView]` rather than `List[NodeView]`, and `BlockView.body` as `BodyView`. Each only ever returns the concrete class; the wider annotation put `block_type`, `labels`, `name_labels` and `AttributeView.name` behind an `isinstance` narrowing or a cast for callers under a strict type checker. The view classes are now imported at module level rather than inside each method, so `typing.get_type_hints` can resolve the annotations the way any consumer reads them. Thanks, @livingstaccato ([#332](https://github.com/amplify-education/python-hcl2/pull/332))
  - Runtime behaviour and the values returned are unchanged, but this is a static break for one shape of caller: `list` is invariant, so `views: List[NodeView] = document.blocks()` no longer type-checks even though `BlockView` is a `NodeView`. Narrow the annotation, drop it, or use `Sequence[NodeView]`.

### Fixed

- `with_meta` emits `__start_line__` and `__end_line__` again. The option, the `hcl2tojson --with-meta` flag and the migration guide's promise that the v7 keys are "still available" all survived the v8 rewrite; the code that produced the keys did not, leaving the option read nowhere in the package. Blocks carry the same spans 7.3.1 produced for the same input; attributes carry none, as in v7. Thanks, @livingstaccato ([#333](https://github.com/amplify-education/python-hcl2/pull/333))
  - The deserializer reads `__start_line__` and `__end_line__` as metadata only where `with_meta` writes them: together, as integers, on a block's body. An attribute of either name anywhere else still survives `dumps(loads(...))`, as it did before. A block that declares both with integer values cannot be told apart from the metadata and loses them; [#331](https://github.com/amplify-education/python-hcl2/issues/331) tracks moving the keys out of band.
- Concurrent calls to `load`/`loads` no longer corrupt each other's values. `serialize()` declared `context=SerializationContext()` as a default argument, which Python evaluates once at import, so every rule in the process shared one mutable context — and the expression, function and indexing rules mutated it in place via `context.modify(inside_dollar_string=True)`. A thread serializing a function call therefore set that flag for every other thread, and a tuple or object being serialized elsewhere came back as its inline HCL source (`'[1, 2, 3]'`) rather than a list, with no exception raised. Measured before the fix: 400 of 400 interleaved parses corrupted. Every other rule's `serialize` carried the same default and is changed with them: the public API always enters at `StartRule`, so those were unreachable with a shared context, but a caller serializing a rule directly still had one. Both parameters are annotated `Optional[...]`, so a rule that takes `None` and then dereferences it without constructing one is a mypy error rather than an `AttributeError` for the direct caller this protects. `options` loses its shared default too. Nothing in the package assigns to a `SerializationOptions`, so that one was not a live defect, but it is the same construct in the same position -- one mutable object handed to every caller that omits the argument, reachable by any subclass or hook a consumer writes.
- `SerializationContext` is immutable, so a context a caller builds and hands to concurrent calls is safe to share. Removing the shared default fixed only the contexts the package creates for itself; `modify` was a context manager that set fields on whatever context it was given and restored them on exit, so a consumer passing one object to several threads still had several writers and the same silent corruption, with nothing to warn them. A traversal now descends by building a child with `replace`, the dataclass is `frozen=True`, and `modify` is gone -- an assignment that used to be a temporary mutation is a `FrozenInstanceError` where it is written. Racing two threads does not demonstrate the old defect, since `modify` restored the field within one call; the regression test reads the caller's own context from inside the scope that used to mutate it, and sees `True` there before this change. ([#344](https://github.com/amplify-education/python-hcl2/issues/344))
- A heredoc whose interpolation spans lines is not flattened. The quoted form cannot hold one: the newlines inside `${...}` are expression source, where OpenTofu rejects an escaped newline and a raw one makes the string span lines, which it also rejects. It used to emit the raw version -- output neither Terraform nor this library could read, written with no error -- and now hands the heredoc back in the form `preserve_heredocs=True` produces, which reads back as that heredoc. Declining is the only answer that does not change what the document means. The value form, which can carry such a body, measures a `<<-` margin on the lines that start with literal text: a line that begins inside `${...}` is expression source, and OpenTofu does not let its indent lower the margin of the rest. The same holds for a heredoc whose template uses a `~` strip marker: a heredoc body is lexed one line at a time, so the marker strips no further than its own line, while in the quoted string the same whitespace runs on into the next line's indent -- OpenTofu evaluates the loop `%{ for s in ["a", "b"] ~}\n  - ${s}\n%{ endfor ~}` to `  - a\n  - b\n` as a heredoc and to `- a\n- b\n` flattened. A declined heredoc written as a function argument goes back as the heredoc itself rather than as a quoted copy of its source. ([#347](https://github.com/amplify-education/python-hcl2/issues/347))
- `$${` and `%%{` resolve to `${` and `%{` in the value form, in both quoted strings and heredocs. They are HCL's escapes for a literal sigil, exactly as `\"` is for a quote, and OpenTofu evaluates `"$${esc}"` to the six characters `${esc}`; returning them doubled made the value differ from the one Terraform reads, in the one mode that promises the value. The escapes written after one resolve as well, since the whole run is literal text: `"$${a\tb}"` is `${a<TAB>b}`, and so is the literal text between two directives. ([#336](https://github.com/amplify-education/python-hcl2/issues/336))
- `strings_to_heredocs` resolves every escape the reader does. It knew `\n`, `\r`, `\"` and `\\`, so `"a\tb\n"` was written into the body as a backslash and a `t` -- two characters where Terraform reads one tab -- and `\uNNNN` fared the same. It now uses `process_escape_sequences`, the package's one implementation of that alphabet. ([#329](https://github.com/amplify-education/python-hcl2/issues/329))
- Escapes are no longer added or resolved inside `${...}`. The text there is expression source, where a nested `"..."` is a string literal of its own: OpenTofu reads `"${upper("a")}"` as `A` and rejects `"${upper(\"a\")}"` outright, so escaping through an interpolation produced source the reference implementation will not parse, and resolving through one closed a nested literal early. Both directions now work on the spans the text is actually made of, and the scan knows the things inside an expression that can carry a non-structural brace: a string literal, HCL's `#`, `//` and `/* */` comments, and the nested expressions a string literal may itself contain -- OpenTofu evaluates `${1 /* } */ + 2}` to 3 and `"a ${upper("v${ "{" }w")} b"` to `a V{W b`, so counting either brace closed the expression inside itself. A heredoc body interprets no escape, so there a backslash is a character and the `${` after it still opens an expression: OpenTofu evaluates `<<EOF\nC:\${upper("a")}\nEOF` to `C:\A`. ([#339](https://github.com/amplify-education/python-hcl2/issues/339))
- `dumps` writes a quoted string back split where the reader splits it. The regex it used knew nothing of string literals and then stripped a `"` from the edge of every piece rather than from the string once, so `"a \"${"b"}\" c"` came back as `"a \${"b"}\" c"`, which OpenTofu rejects with "Invalid escape sequence"; a flattened heredoc holding a JSON policy did the same, and a brace inside a nested literal raised `UnexpectedToken`. It now uses the same span scanner as the rest of this change.
- Flattened heredoc bodies match the values Terraform and OpenTofu evaluate the same source to. Three things differed, all checked against OpenTofu v1.12.5 rather than read off the spec: the newline terminating the last content line was dropped (`<<EOT\nline\nEOT` returned `'line'`, not `'line\n'`); `<<-` measured its indent in spaces alone, so a tab-indented body was not dedented at all; and a whitespace-only line was excluded from the measurement but trimmed anyway. This is not a regression — 7.2.1 returned the same values — so it changes long-standing behaviour rather than restoring anything.
- A carriage return in a flattened heredoc body is written as `\r` rather than left raw. `preserve_heredocs=False` returns quoted-string *source*, and a quoted string cannot hold a literal carriage return: OpenTofu rejects one with "No closing marker was found for the string". A heredoc read out of a CRLF file therefore flattened to source that would not parse again. The value form (`strip_string_quotes=True`) is unchanged and still hands back real carriage returns. `strings_to_heredocs` resolves `\r` when it writes a body, so the two halves stay each other's inverse: a heredoc interprets no escape, so a body carrying a backslash and an `r` would be those two characters rather than the carriage return the value held.
- A heredoc written inside a list or an object ends its own line, `<<-` as well as `<<`. It ends at its closing marker, on a line of its own, so what follows has to start the next one -- `EOF,` closes nothing, and the file this library had just written did not parse, here or in Terraform. A top-level attribute survived only because the newline after it comes from the document. A heredoc built from a dict is now the same token a parsed one is: the grammar's terminal name, running through the newline after the marker, so it ends its line wherever it is written and reconstructing a parsed document is still byte for byte what it was. In an object that line break is the separator: OpenTofu rejects a comma at the start of the next line, so none is written there, whether the object is written out or inline inside an expression. ([#338](https://github.com/amplify-education/python-hcl2/issues/338))
- A heredoc written as a function argument or an operand loads as the heredoc it is. It came back quoted, markers and all: `upper(<<EOF\nfoo\nEOF\n)` loaded as `${upper("<<EOF\nfoo\nEOF")}`, which `dumps` wrote as a multi-line quoted string that OpenTofu rejects with "Invalid multi-line string" and that was a different value besides. It now loads as `${upper(<<EOF\nfoo\nEOF\n)}`, keeping the newline after the closing marker so the `)` starts the next line, and `dumps` writes the source back. **This changes the default output** for any document with a heredoc inside an expression; a heredoc that is a whole attribute value is unchanged.
- A `<<-` heredoc whose closing marker is indented with something other than spaces or tabs no longer appends that indentation to the value. The dedent already measured whitespace rather than spaces, matching OpenTofu, but the marker's own indent was stripped as `[ \t]*`, so a body indented with a non-breaking space, a vertical tab, a form feed or an ideographic space came back with one of those characters on the end. Whitespace here is Go's `unicode.IsSpace`, which OpenTofu measures with: the information separators U+001C to U+001F, which Python counts as whitespace, are content, so a line led by one sets the margin to zero rather than being dedented and losing them. Four such cases are now in the table that `bin/heredoc_ground_truth` re-derives from OpenTofu.
- `strings_to_heredocs` leaves a value carrying a lone carriage return quoted. A heredoc body is read literally, so it can hold a `\r` only where one ends a line: OpenTofu rejects `<<EOF\nx\ry\nEOF` with "No closing marker was found for the string", while the quoted `"x\ry\n"` it came from is valid. Such a value stays quoted, for the same reason one that does not end in a newline does.
- `strings_to_heredocs` picks a delimiter the body cannot close. It wrote `<<EOF` over every value, so a string holding a line reading `EOF` -- a log excerpt, a shell script, an embedded config, the payloads heredocs are for -- ended its own heredoc early and produced a file that no longer parsed. A numbered variant is used when the body occupies `EOF`, and ordinary values are written exactly as before. The lines that count as markers are Terraform's: OpenTofu ends a heredoc on the word padded with any whitespace at all, `EOF  `, a non-breaking space or a form feed included. A CRLF body counts too -- it is split on `\n`, so its lines carry their own `\r`, and OpenTofu ends a heredoc on `EOF\r` as readily as on `EOF `. ([#330](https://github.com/amplify-education/python-hcl2/issues/330))
- `strings_to_heredocs` no longer adds a line to the body it writes. The value's own trailing newline is the one that precedes the closing marker, so a heredoc was being emitted one line longer than the string it came from. A value that does not end in a newline is now left as a quoted string, since no heredoc can express it. Flattening a document and restoring it now yields HCL that OpenTofu evaluates identically to the original; five of the eleven values in the round-trip fixture did not survive it before.
- `$${` and `%%{` resolve to `${` and `%{` in the value form, in both quoted strings and heredocs. They are HCL's escapes for a literal sigil, exactly as `\"` is for a quote, and OpenTofu evaluates `"$${esc}"` to the six characters `${esc}`; returning them doubled made the value differ from the one Terraform reads, in the one mode that promises the value. The escapes written after one resolve as well, since the whole run is literal text: `"$${a\tb}"` is `${a<TAB>b}`, and so is the literal text between two directives. ([#336](https://github.com/amplify-education/python-hcl2/issues/336))
- `strings_to_heredocs` resolves every escape the reader does. It knew `\n`, `\r`, `\"` and `\\`, so `"a\tb\n"` was written into the body as a backslash and a `t` -- two characters where Terraform reads one tab -- and `\uNNNN` fared the same. It now uses `process_escape_sequences`, the package's one implementation of that alphabet. ([#329](https://github.com/amplify-education/python-hcl2/issues/329))
- Escapes are no longer added or resolved inside `${...}`. The text there is expression source, where a nested `"..."` is a string literal of its own: OpenTofu reads `"${upper("a")}"` as `A` and rejects `"${upper(\"a\")}"` outright, so escaping through an interpolation produced source the reference implementation will not parse, and resolving through one closed a nested literal early. Both directions now work on the spans the text is actually made of, and the scan knows the things inside an expression that can carry a non-structural brace: a string literal, HCL's `#`, `//` and `/* */` comments, and the nested expressions a string literal may itself contain -- OpenTofu evaluates `${1 /* } */ + 2}` to 3 and `"a ${upper("v${ "{" }w")} b"` to `a V{W b`, so counting either brace closed the expression inside itself. A heredoc body interprets no escape, so there a backslash is a character and the `${` after it still opens an expression: OpenTofu evaluates `<<EOF\nC:\${upper("a")}\nEOF` to `C:\A`. ([#339](https://github.com/amplify-education/python-hcl2/issues/339))
- `force_operation_parentheses` now adds parentheses inside an expression that is already parenthesised. The operation rules handed `inside_parentheses` — which means "my container already wrapped me", and stops the option doubling parentheses — down to their operands, which nothing wraps, so a single pair anywhere above an operation silenced the option for everything below it: `(b + c * d)` came back unchanged. It now reaches through parentheses, function calls, indexes and for-expressions alike. The option-less path is unaffected. Thanks, @livingstaccato ([#348](https://github.com/amplify-education/python-hcl2/pull/348))
- Parse heredocs whose closing marker carries trailing spaces or tabs, such as `EOF  `. Both heredoc terminals required the newline to follow the delimiter immediately, so the marker went unrecognised, the heredoc ran on to a later one, and the parse failed pointing at an unrelated line. Terraform ends a heredoc at any line holding the delimiter and nothing else that matters, so such a file parses everywhere else. One input changes meaning: a body line consisting of the delimiter plus trailing whitespace now closes the heredoc rather than being content, as it does in Terraform. Thanks, @livingstaccato ([#349](https://github.com/amplify-education/python-hcl2/pull/349))
- `strip_string_quotes` now writes a heredoc inside an expression as a string instead of splicing its body in bare. `StringRule` checks `inside_dollar_string` to keep its quotes for exactly this reason; the heredoc rules did not, so `upper(<<E\nx\nE\n)` came back as `${upper(x)}` — a reference to a variable nobody declared — and a multi-line body put raw newlines into source that would not parse. Both `<<` and `<<-` are fixed, in every expression context. Thanks, @livingstaccato ([#350](https://github.com/amplify-education/python-hcl2/pull/350))
- `strip_string_quotes` now keeps the delimiters of a string literal inside a template directive. `TemplateStringRule` only ever appears inside `%{ ... }`, where the text is expression source and the quotes belong to a literal written in it, so dropping them turned `%{ if x == "y" }` into `%{ if x == y }`: a comparison against a variable rather than against a string. Thanks, @livingstaccato ([#350](https://github.com/amplify-education/python-hcl2/pull/350))
- A parenthesised expression writes what it wraps as HCL rather than as a Python value, on the default options as well: `(true)` used to come back as `${(True)}` and `(null)` as `${(None)}`, which `dumps` wrote back as references to variables nobody declared (OpenTofu rejects both with "Invalid reference"); a tuple or object inside the parentheses came back as a Python repr that `dumps` could not parse; and with `strip_string_quotes`, `("s")` lost its quotes and became `${(s)}`. They now come back as `${(true)}`, `${(null)}`, `${([1, "a"])}` and `${("s")}`, and each round trip evaluates to the value the source does. Thanks, @livingstaccato ([#350](https://github.com/amplify-education/python-hcl2/pull/350))
- `heredocs_to_strings` writes the heredoc's value rather than its own text. It was quoting the source -- markers and all -- so `<<EOT\nhello\nEOT` became `"<<EOT\nhello\nEOT"`, a quoted string spanning three physical lines. A quoted template cannot span lines, so OpenTofu rejects that with "Invalid multi-line string", and reading it back here gave the marker text rather than the value: neither a valid file nor the right content. It now reuses the flattening the reader already performs, so the two cannot drift. A heredoc whose `${...}` runs across lines has no quoted spelling -- the newlines inside the expression can be neither escaped nor left raw -- so it stays a heredoc; flattening one raised `UnexpectedToken` out of `dumps`. ([#337](https://github.com/amplify-education/python-hcl2/issues/337))
- A heredoc written inside a list or an object ends its own line, `<<-` as well as `<<`. It ends at its closing marker, on a line of its own, so the separator that follows has to start the next one -- `EOF,` closes nothing, and the file this library had just written did not parse, here or in Terraform. A top-level attribute survived only because the newline after it comes from the document. Only a heredoc built by the deserializer needs this: `HEREDOC_TEMPLATE` matches through the newline after the marker, so one that came from the parser already ends the line, and reconstructing a parsed document is byte for byte what it was. ([#338](https://github.com/amplify-education/python-hcl2/issues/338))
- A heredoc written as a function argument or an operand loads as the heredoc it is. It came back quoted, markers and all: `upper(<<EOF\nfoo\nEOF\n)` loaded as `${upper("<<EOF\nfoo\nEOF")}`, which `dumps` wrote as a multi-line quoted string that OpenTofu rejects with "Invalid multi-line string" and that was a different value besides. It now loads as `${upper(<<EOF\nfoo\nEOF\n)}`, keeping the newline after the closing marker so the `)` starts the next line, and `dumps` writes the source back. **This changes the default output** for any document with a heredoc inside an expression; a heredoc that is a whole attribute value is unchanged.
- A heredoc whose interpolation spans lines is not flattened. The quoted form cannot hold one: the newlines inside `${...}` are expression source, where OpenTofu rejects an escaped newline and a raw one makes the string span lines, which it also rejects. It used to emit the raw version -- output neither Terraform nor this library could read, written with no error -- and now hands the heredoc back in the form `preserve_heredocs=True` produces, which reads back as that heredoc. Declining is the only answer that does not change what the document means. The value form, which can carry such a body, measures a `<<-` margin on the lines that start with literal text: a line that begins inside `${...}` is expression source, and OpenTofu does not let its indent lower the margin of the rest. ([#347](https://github.com/amplify-education/python-hcl2/issues/347))

## \[8.1.4\] - 2026-09-08

### Fixed

- Parse blocks whose type or unquoted label is an HCL keyword, such as the `in` block in Snowflake's `snowflake_schemas` data source. HCL reserves no keywords, so all are now accepted as block labels. Diagnosed independently in [#355](https://github.com/amplify-education/python-hcl2/pull/355). ([#357](https://github.com/amplify-education/python-hcl2/pull/357))
- Parse keyword-named *object* keys reliably, fixing a regression of [#148](https://github.com/amplify-education/python-hcl2/issues/148). A key such as `in` parsed only where the lexer fell back to `NAME`, so its separator and position decided whether the file parsed. ([#357](https://github.com/amplify-education/python-hcl2/pull/357))

## \[8.1.3\] - 2026-08-26

Several fixes below change the values `loads()` returns for input that already
parsed without error in 8.1.x — negative integer literals, both
`strip_string_quotes` behaviours, and the two heredoc body fixes. The previous
result was a bug in each case, so this stays a patch release; re-check your
expectations if you built around the old values.

### Fixed

- Restore `py.typed` marker so type checkers recognize `hcl2` (and `cli`) as typed packages. ([#299](https://github.com/amplify-education/python-hcl2/pull/299))
- Parse heredocs with an empty body again. A marker immediately followed by its closing delimiter failed to match, and the lexer then ran on to a later delimiter, silently absorbing the attributes in between. Thanks, @livingstaccato ([#312](https://github.com/amplify-education/python-hcl2/pull/312))
- Negative integer literals load as numbers again instead of `${-N}` expression strings, matching negative floats and the pre-8.x behaviour. Thanks, @livingstaccato ([#311](https://github.com/amplify-education/python-hcl2/pull/311))
- `strip_string_quotes` no longer unquotes string literals nested inside expressions, which produced invalid HCL such as `${upper(x)}` from `upper("x")`. Thanks, @livingstaccato ([#313](https://github.com/amplify-education/python-hcl2/pull/313))
- `strip_string_quotes` now resolves escape sequences, so the values it yields match what the option documents. Escapes naming a codepoint outside the Unicode range, or a lone surrogate, are preserved verbatim rather than raising. Thanks, @livingstaccato ([#313](https://github.com/amplify-education/python-hcl2/pull/313))
- Parse files with CRLF (`\r\n`) line endings, including heredocs. A `\r` acting as part of a line ending is ignored, so a CRLF file reconstructs with LF endings; a `\r` that is content — inside a quoted string or a heredoc body — is preserved. Thanks, @agu2347 ([#317](https://github.com/amplify-education/python-hcl2/pull/317))
- Flattened heredoc bodies keep their trailing blank lines and trailing spaces instead of being right-stripped away, for both `<<MARKER` and `<<-MARKER`. The closing marker line's own indentation is still removed, and a blank line no longer cancels the `<<-` dedent. Thanks, @agu2347 ([#318](https://github.com/amplify-education/python-hcl2/pull/318))
- Parse heredocs whose delimiter is a single character, such as `<<E`. The spec defines the delimiter as an Identifier, which permits one character. ([#323](https://github.com/amplify-education/python-hcl2/pull/323))
- `preserve_heredocs=False` combined with `strip_string_quotes` now returns the heredoc body as a plain multi-line string instead of escaping every newline to a literal `\n`. The escaping is still applied to the quoted source form produced without `strip_string_quotes`. ([#324](https://github.com/amplify-education/python-hcl2/pull/324))

## \[8.1.2\] - 2026-04-10

### Fixed

- `true`, `false`, and `null` now serialize to native JSON types instead of strings. ([#293](https://github.com/amplify-education/python-hcl2/issues/293))

## \[8.1.1\] - 2026-04-07

### Added

- v7-to-v8 migration guide and absolute GitHub links in README docs table. ([#287](https://github.com/amplify-education/python-hcl2/pull/287))

## \[8.1.0\] - 2026-04-07

### Added

- Full architecture overhaul: bidirectional HCL2 ↔ JSON pipeline with typed rule classes. ([#203](https://github.com/amplify-education/python-hcl2/pull/203))
- `hq` read-only query CLI for HCL2 files ([#277](https://github.com/amplify-education/python-hcl2/pull/277))
- Agent-friendly conversion CLIs: `hcl2tojson` and `jsontohcl2` ([#274](https://github.com/amplify-education/python-hcl2/pull/274))
- Add template directives support (`%{if}`, `%{for}`) in quoted strings ([#276](https://github.com/amplify-education/python-hcl2/pull/276))
- Support loading comments ([#134](https://github.com/amplify-education/python-hcl2/issues/134))
- CLAUDE.md ([#260](https://github.com/amplify-education/python-hcl2/pull/260))

### Fixed

- Ternary with strings parse error ([#55](https://github.com/amplify-education/python-hcl2/issues/55))
- "No terminal matches '|' in the current parser context" when parsing multi-line conditional ([#142](https://github.com/amplify-education/python-hcl2/issues/142))
- reverse_transform not working with object-type variables ([#231](https://github.com/amplify-education/python-hcl2/issues/231))
- reverse_transform not handling nested functions ([#235](https://github.com/amplify-education/python-hcl2/issues/235))
- `writes` omits quotes around map keys with `/` ([#236](https://github.com/amplify-education/python-hcl2/issues/236))
- Operator precedence bug ([#248](https://github.com/amplify-education/python-hcl2/issues/248))
- Empty string dictionary keys can't be parsed twice ([#249](https://github.com/amplify-education/python-hcl2/issues/249))
- jsonencode not deserialized correctly ([#250](https://github.com/amplify-education/python-hcl2/issues/250))
- Literal string "string" incorrectly quoted ([#251](https://github.com/amplify-education/python-hcl2/issues/251))
- Interpolation literals added to locals/variables in maps ([#252](https://github.com/amplify-education/python-hcl2/issues/252))
- Object literal expression can't be serialized ([#253](https://github.com/amplify-education/python-hcl2/issues/253))
- Heredocs should interpret backslash literally ([#262](https://github.com/amplify-education/python-hcl2/issues/262))
- Parsing a multi-line multi-conditional expression causes exception — Unexpected token Token('QMARK', '?') ([#269](https://github.com/amplify-education/python-hcl2/issues/269))
- Parsing error for multiline binary operators ([#246](https://github.com/amplify-education/python-hcl2/pull/246))

### Changed

- Updated package metadata: development status, dropped Python 3.7 support. ([#263](https://github.com/amplify-education/python-hcl2/pull/263))

## \[7.3.1\] - 2025-07-24

### Fixed

- Updated pyproject.toml dependencies. Thanks, @kkorlyak ([#244](https://github.com/amplify-education/python-hcl2/pull/244))

## \[7.3.0\] - 2025-07-23

### Fixed

- Issue parsing interpolations and escaped interpolations in a single string. ([#239](https://github.com/amplify-education/python-hcl2/pull/239))

## \[7.2.1\] - 2025-05-16

### Fixed

- More robust escaping for special characters. Thanks, @eranor ([#224](https://github.com/amplify-education/python-hcl2/pull/224))
- Issue parsing interpolation string as an object key ([#232](https://github.com/amplify-education/python-hcl2/pull/232))

## \[7.2.0\] - 2025-04-24

### Added

- Possibility to parse deeply nested interpolations (formerly a Limitation), Thanks again, @weaversam8 ([#223](https://github.com/amplify-education/python-hcl2/pull/223))

### Fixed

- Issue parsing ellipsis in a separate line within `for` expression ([#221](https://github.com/amplify-education/python-hcl2/pull/221))
- Issue parsing inline expression as an object key; **see Limitations in README.md** ([#222](https://github.com/amplify-education/python-hcl2/pull/222))
- Preserve literals of e-notation floats in parsing and reconstruction. Thanks, @eranor ([#226](https://github.com/amplify-education/python-hcl2/pull/226))

## \[7.1.0\] - 2025-04-10

### Added

- `hcl2.builder.Builder` - nested blocks support ([#214](https://github.com/amplify-education/python-hcl2/pull/214))

### Fixed

- Issue parsing parenthesesed identifier (reference) as an object key ([#212](https://github.com/amplify-education/python-hcl2/pull/212))
- Issue discarding empty lists when transforming python dictionary into Lark Tree ([#216](https://github.com/amplify-education/python-hcl2/pull/216))

## \[7.0.1\] - 2025-03-31

### Fixed

- Issue parsing dot-accessed attribute as an object key ([#209](https://github.com/amplify-education/python-hcl2/pull/209))

## \[7.0.0\] - 2025-03-27

### Added

- `Limitations` section to README.md ([#200](https://github.com/amplify-education/python-hcl2/pull/200))

### Fixed

- Issue handling heredoc with delimiter within text itself ([#194](https://github.com/amplify-education/python-hcl2/pull/194))
- Various issues with parsing object elements ([#197](https://github.com/amplify-education/python-hcl2/pull/197))
- Dictionary -> hcl2 reconstruction of `null` values ([#198](https://github.com/amplify-education/python-hcl2/pull/198))
- Inaccurate parsing of `null` values in some cases ([#206](https://github.com/amplify-education/python-hcl2/pull/206))
- Missing parenthesis in arithemetic expressions ([#194](https://github.com/amplify-education/python-hcl2/pull/199))
- Noticeable overhead when loading hcl2.reconstructor module ([#202](https://github.com/amplify-education/python-hcl2/pull/202))
- Escaped string interpolation (e.g. `"$${aws:username}"`) parsing ([#200](https://github.com/amplify-education/python-hcl2/pull/200))

### Removed

- Support for parsing interpolations nested more than 2 times (known-issue) ([#200](https://github.com/amplify-education/python-hcl2/pull/200))

## \[6.1.1\] - 2025-02-13

### Fixed

- `DictTransformer.to_tf_inline` - handle float type. ([#188](https://github.com/amplify-education/python-hcl2/pull/188))

## \[6.1.0\] - 2025-01-24

### Fixed

- fix e-notation and negative numbers literals. ([#182](https://github.com/amplify-education/python-hcl2/pull/182))
- fix parsing of `null`.  ([#184](https://github.com/amplify-education/python-hcl2/pull/184))
- DictTransformer - do not wrap type literals into `${` and `}`. ([#186](https://github.com/amplify-education/python-hcl2/pull/186))

## \[6.0.0\] - 2025-01-15

### Added

- Support full reconstruction of HCL from Python structures. Thanks, @weaversam8, @Nfsaavedra ([#177](https://github.com/amplify-education/python-hcl2/pull/177))

## \[5.1.1\] - 2024-10-15

### Added

- fix `tree-to-hcl2-reconstruction.md` URL in README.md ([#175](https://github.com/amplify-education/python-hcl2/pull/175))

## \[5.1.0\] - 2024-10-15

### Added

- support python 3.13 ([#170](https://github.com/amplify-education/python-hcl2/pull/170))
- add section about Tree->HCL2 reconstruction to the README.md ([#174](https://github.com/amplify-education/python-hcl2/pull/174))

## \[5.0.0\] - 2024-10-07

### Added

- Support full reconstruction of HCL from parse tree. Thanks, @weaversam8 ([#169](https://github.com/amplify-education/python-hcl2/pull/169))

## \[4.3.5\] - 2024-08-06

### Added

- additional test coverage ([#165](https://github.com/amplify-education/python-hcl2/pull/165))
- fix: Add support for attributes named "in". Thanks, @elisiariocouto ([#164](https://github.com/amplify-education/python-hcl2/pull/164))
- fix: add "for" attribute identifier. Thanks, @zhcli ([#167](https://github.com/amplify-education/python-hcl2/pull/167))
- allow `if` and `for_each` keywords to be used as identifiers ([#168](https://github.com/amplify-education/python-hcl2/pull/168))

### Added

## \[4.3.4\] - 2024-06-12

### Added

- fix codacy badge ([#157](https://github.com/amplify-education/python-hcl2/pull/157))
- Fix MANIFEST.in and/or Python dependency filename(s) ([#161](https://github.com/amplify-education/python-hcl2/pull/161))
- adds support for provider functions. Thanks, @lkwg82 ([#162](https://github.com/amplify-education/python-hcl2/pull/162))

## \[4.3.3\] - 2024-03-27

### Added

- Support for Python 3.12 ([#153](https://github.com/amplify-education/python-hcl2/pull/153))

## \[4.3.2\] - 2023-05-24

### Added

- Support for the conditional inside the nested locals without parentheses ([#138](https://github.com/amplify-education/python-hcl2/pull/129))

## \[4.3.1\] - 2023-05-02

### Added

- Support for the braces in the next line. Thanks @rout39574 ([#129](https://github.com/amplify-education/python-hcl2/pull/129))
- Support for the ternary multi-line expression. Thanks @seksham ([#128](https://github.com/amplify-education/python-hcl2/pull/128))

## \[4.3.0\] - 2022-01-16

### Added

- Add tests for multiline comments inside a tuple ([#118](https://github.com/amplify-education/python-hcl2/pull/118))
- Add `__begin_line__` and `__end_line__` meta parameters ([#120](https://github.com/amplify-education/python-hcl2/pull/120))
- Add feature to parse comments in function args and list elems ([#119](https://github.com/amplify-education/python-hcl2/pull/119))

### Fixed

- Support empty heredoc and fix catastrophic backtracking issue ([#117](https://github.com/amplify-education/python-hcl2/pull/117))

### Changed

- Use Lark with its cache feature, instead of creating a standalone parser by @erezsh ([#53](https://github.com/amplify-education/python-hcl2/pull/53))
- Refactor tests ([#114](https://github.com/amplify-education/python-hcl2/pull/114))
- Remove pycodestyle, add black, add numerous pre-commit checks ([#115](https://github.com/amplify-education/python-hcl2/pull/115))

## \[4.2.0\] - 2022-12-28

### Added

- Added support of the `lark ≥1.0,<2`. Thanks @KOLANICH ([#100](https://github.com/amplify-education/python-hcl2/pull/100))

### Changed

- Dropped support of the `lark <1.0`.
- Added code improvements

## \[4.1.0\] - 2022-12-27

### Added

- Added support of python 3.11

### Changed

- Moved from setup.py to pyproject.toml. Thanks @KOLANICH ([#98](https://github.com/amplify-education/python-hcl2/pull/98))
- Updated the tox version in github actions to >=4.0.9,\<5.
- Dropped completely python 3.6.

## \[4.0.0\] - 2022-12-14

### Added

- Added PEP improvements
- Added support of python 3.10

### Changed

- Dropped support of python 3.6
- Setup tox-gh-actions
- Migrated from nose to nose2

## \[3.0.5\] - 2022-03-21

### Fixed

- Fixed parsing of for expressions when there is a new line before the colon

## \[3.0.4\] - 2022-02-22

### Added

- Handle nested interpolations. Thanks @arielkru and @matt-land ([#61](https://github.com/amplify-education/python-hcl2/pull/61))

## \[3.0.3\] - 2022-02-20

### Fixed

- Fixed nested splat statements. Thanks @josh-barker ([#80](https://github.com/amplify-education/python-hcl2/pull/80))

## \[3.0.2\] - 2022-02-20

### Fixed

- Fixed an issue of whitespace around for expressions. Thanks @ryanking and @matchaxnb ([#87](https://github.com/amplify-education/python-hcl2/pull/87))

## \[3.0.1\] - 2021-07-15

### Changed

- Included the generated parser in the distribution.

## \[3.0.0\] - 2021-07-14

### Changed

- BREAKING CHANGES: Attributes in blocks are no longer transformed into Python lists. Thanks @raymondbutcher ([#73](https://github.com/amplify-education/python-hcl2/pull/73))

## \[2.0.3\] - 2021-03-04

### Changed

- Skipped more exceptions for un-parsable files. Thanks @tanasegabriel ([#60](https://github.com/amplify-education/python-hcl2/pull/60))

## \[2.0.2\] - 2021-03-04

### Changed

- Allowed empty objects. Thanks @santoshankr ([#59](https://github.com/amplify-education/python-hcl2/pull/59))

## \[2.0.1\] - 2020-12-24

### Changed

- Allowed multiline conditional statements. Thanks @stpierre ([#51](https://github.com/amplify-education/python-hcl2/pull/51))

## \[2.0.0\] - 2020-11-02

### Changed

- Added support for Python 3.9
- Upgraded to Lark parser 0.10

### Fixed

- Fixed errors caused by identifiers named "true", "false", or "null"

## \[1.0.0\] - 2020-09-30

### Changed

- Treat one line blocks the same as multi line blocks.
  This is a breaking change so bumping to 1.0.0 to make sure no one accidentally upgrades to this version
  without being aware of the breaking change.
  Thank you @arielkru ([#35](https://github.com/amplify-education/python-hcl2/pull/35))

## \[0.3.2\] - 2020-09-29

### Changed

- Added support for colon separators in object definitions as specified in the [spec](https://github.com/hashicorp/hcl/blob/hcl2/hclsyntax/spec.md#collection-values)

## \[0.3.1\] - 2020-09-27

### Changed

- Added support for legacy array index notation using dot. Thank you @arielkru ([#36](https://github.com/amplify-education/python-hcl2/pull/36))
