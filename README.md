# hocon

[HOCON](https://github.com/lightbend/config/blob/main/HOCON.md), read into a tree that remembers
where every value was written, for sysl.

A program that reads a configuration file to *check* it — a manifest, a board description — has to
point at the line that is wrong. Every value and every key in the tree this package answers carries
its `Span`, and a refusal is a `sh.sysl.parsing.Diagnostic`, so both are reported the way the sysl
compiler reports its own errors:

```
error: expected ',' or a new line after the value of `pin`
 --> board.hocon:4:23
  |
4 |     pin = GP16 colour = red
  |                       ^ found `=`
```

## Using it

```hocon
dependencies {
  hocon { git = "github.com/sysl-lang/hocon", version = "0.2.0" }
}
```

The module is `sh.sysl.hocon`. Naming the coordinate is enough to import `sh.sysl.parsing` too, which
a caller needs for `Source` and `Diagnostic`.

```sysl
import sh.sysl.hocon.*
import sh.sysl.parsing.source_of

main()
    val text = "led { pin = 16, color = green }\nd1 = ${led} { label = \"D1\" }"
    val src = source_of("board.hocon", text)

    parse(src) match
        Ok(doc) ->
            val d1 = doc.root.get("d1").unwrap()

            print(d1.get("color").unwrap().as_text())        // Some(green)
            print(src.locate(d1.get("pin").unwrap().span.start))  // 1:13, where 16 was written
        Err(d) -> print(d.render(src))
```

## The tree

- `Value { span, kind }`, where `kind` is `Obj(fields)`, `Arr(items)`, `Str(s)` (quoted), `Word(s)`
  (unquoted), `Int(n, written)`, `Real(x, written)`, `Bool(b)` or `Null`. A bare word that is a JSON
  number is a number; `1.0.0` and `0x10` are words. **A number keeps the text it was written as**:
  `0.10` is the real 0.1 and still `0.10`, and an integer too wide for a `long` keeps every digit.
- `Field { key, key_span, value, implied }`. `implied` marks an object a dotted key made (`a` in
  `a.b = 1`).
- `get`, `field`, `at("a.b.c")`, `fields`, `keys`, `items`, `is_obj`, `is_arr`, `is_null`,
  `kind_name` (`"an object"`, for a message).
- Scalars, each `None` where the value is not one: `as_text` (a string, quoted or not), `as_quoted`
  (a quoted string only), `as_word` (an unquoted value as written: a word, a number, `null`),
  `as_scalar` (any scalar as written, `true` and `false` included), `scalar_at(key)`, `as_long`,
  `as_real`, `as_bool`.
- `as_text_list` (an array of strings and words, as their texts), `flags` (an object's boolean fields,
  in order), and `object_of(pairs)` to build an object.
- `Document { root, repeats }`. HOCON lets a name be written twice — objects merge, a later plain
  value replaces — so that is not an error here. Every name the document wrote out twice is
  recorded as a `Repeat { path, first, again }`, for a format that wants it to be one.
- `render(v)` writes a value back as HOCON that reads to the same tree, and `same(a, b)` compares two
  trees ignoring spans.

## What is read

Objects with or without `=`/`:` before the brace, a root with or without braces (or an array),
dotted and quoted keys, `#` and `//` comments, separators by comma or new line, JSON's escapes
(surrogate pairs included), triple-quoted strings, unquoted strings and their concatenation,
substitutions `${a.b}` and `${?a.b}` (resolved against the root, forward references included),
concatenation of strings, objects (`${base} { pin = 16 }`) and arrays, merging of repeated objects,
CRLF line endings and a byte-order mark.

## Where a refusal points

At the first mistake, with a span. An object, array or string the input ends inside is reported
where the input ends — that is where the reader stopped and where the missing close belongs — with a
note naming where it was opened:

```
error: unclosed object — expected '}'
 --> sysl.hocon:3:1
  |
3 |
  | ^ the input ends here
  |
  = note: the '{' at line 1, column 9 is never closed
```

for `package {\n  name = "a"\n`.

A binding with nothing after it but the end of the input, inside braces, is that unclosed object
(`targets { default =` is a block the writer is still typing). The messages are the ones the sysl
compiler's manifest reader has always given, since that is this package's first reader.

## What is refused, by name

- **`include`** — a document is read from one text this package was handed; it opens nothing.
- **`+=`** and **a value that refers to its own earlier setting** (`path = ${path}":/bin"`), which is
  reported as a substitution cycle. Both need the history of a key, which a tree does not keep.
- **Environment variables** are not consulted for a substitution that names nothing: that is an error
  unless it was written `${?path}`.
- `+` is allowed in an unquoted string except as the start of `+=`, so `1.0.0+build` needs no quotes.

## Licence

ISC — see `LICENSE`.
