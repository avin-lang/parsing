# Kite

Kite is a small statically typed language run by a tree-walking interpreter. It exists to show
what the [parsing](../../README.md) library does with a real grammar: its parser,
[`kite/grammar.lavin`](kite/grammar.lavin), is the grammar below written as Avin values. A program
is a module of functions and types:

```kite
module geometry

type Shape =
    | Circle(r: f64)
    | Rect(w: f64, h: f64)

fn area(s: Shape) -> f64 {
    match s {
        Circle(r) => 3.14159 * r * r
        Rect(w, h) => w * h
    }
}

fn main() {
    let shapes = [Circle(r: 1.0), Rect(w: 2.0, h: 3.0)]
    for i in 0..2 {
        print("shape {i} has area {area(shapes[i])}")
    }
}
```

Blocks are braces, a line end ends a statement, and blocks, `if` and `match` are expressions whose
value is their last expression's. The types are `i64`, `f64`, `bool`, `string`, fixed arrays
`[T; n]`, records and sum types. There are no generics, traits, closures or overloading, and no
garbage collector to think about: values are copied.

## Grammar

White space, `// line comments` and `/* block comments */`, which nest, may stand between any two
tokens. `{ … }` repeats, `[ … ]` is optional, `|` separates alternatives, and `,?` is an optional
trailing comma.

```ebnf
file        = "module" dotted EOL { "import" dotted EOL } { decl EOL } ;
dotted      = name { "." name } ;
decl        = fnDecl | typeDecl ;
fnDecl      = "fn" name "(" [ param { "," param } ,? ] ")" [ "->" type ] block ;
param       = name ":" type ;
typeDecl    = "type" name "=" typeDef ;
typeDef     = "|" variant { "|" variant }                       (* a sum *)
            | "{" [ field { "," field } ,? ] "}"                (* a record *)
            | type ;                                            (* an alias *)
variant     = name [ "(" [ field { "," field } ,? ] ")" ] ;
field       = name ":" type ;
type        = name | "[" type ";" INT "]" ;

block       = "{" { stmt END } "}" ;                            (* END: a line end, or before "}" *)
stmt        = ( "let" | "var" ) name [ ":" type ] "=" expr
            | "while" expr block
            | "for" name "in" expr block
            | "return" [ expr ]                                 (* the value on the same line *)
            | "break" | "continue"
            | expr [ assignOp expr ] ;
assignOp    = "=" | "+=" | "-=" | "*=" | "/=" | "%=" ;

expr        = and { "||" and } ;
and         = cmp { "&&" cmp } ;
cmp         = range [ ( "==" | "!=" | "<" | "<=" | ">" | ">=" ) range ] ;
range       = sum [ ( ".." | "..=" ) sum ] ;
sum         = product { ( "+" | "-" ) product } ;
product     = prefix { ( "*" | "/" | "%" ) prefix } ;
prefix      = ( "-" | "!" ) prefix | postfix ;
postfix     = primary { "(" [ arg { "," arg } ,? ] ")" | "[" expr "]" | "." name } ;
arg         = [ name ":" ] expr ;
primary     = INT | FLOAT | STRING | "true" | "false" | name
            | "(" expr ")" | block | ifExpr | matchExpr
            | "[" [ expr ( ";" expr | { "," expr } ,? ) ] "]" ;
ifExpr      = "if" expr block [ "else" ( ifExpr | block ) ] ;
matchExpr   = "match" expr "{" { arm ( "," | END ) } "}" ;
arm         = pattern "=>" expr ;
pattern     = "_" | [ "-" ] INT | [ "-" ] FLOAT | PLAIN_STRING | "true" | "false"
            | name [ "(" [ pattern { "," pattern } ,? ] ")" ] ;
```

The tokens:

- `name`: a letter in any script or `_`, then letters, digits and `_`; not a keyword. The keywords
  are `module import fn type let var if else while for in match return break continue true false`.
- `INT`: ASCII digits, at most 9223372036854775807. `FLOAT`: digits, then `.` and digits, an
  exponent (`e` or `E`, an optional sign, digits), or both. A `.` not followed by a digit is not
  part of a number, so `0..10` is a range and `1.abs` a field.
- `STRING`: `"…"` on one line. `{expr}` interpolates any expression, strings included. The escapes
  are `\n \t \r \0 \\ \" \{ \}` and `\u{…}` with one to six hex digits naming a Unicode scalar
  value. A lone `}` is text. A `PLAIN_STRING`, in a pattern, cannot interpolate.

## Operators

Loosest first. Every binary operator is written between operands of the next tighter level.

| Level | Operators | Grouping |
| --- | --- | --- |
| 1 | `\|\|` | left |
| 2 | `&&` | left |
| 3 | `==` `!=` `<` `<=` `>` `>=` | none: `a < b < c` is an error |
| 4 | `..` `..=` | none: `a..b..c` is an error |
| 5 | `+` `-` | left |
| 6 | `*` `/` `%` | left |
| 7 | prefix `-` `!` | right: `- -a` |
| 8 | call `f(x)`, index `a[i]`, field `p.x` | left: `a.b(c)[d]` |

Assignment, `=` and the compound `+= -= *= /= %=`, is a statement, not an operator.

## Line ends

A line end ends a statement. It does not where the statement plainly is not over:

- **inside brackets**: inside `(…)`, `[…]`, a record type's `{…}` and an interpolation's `{…}`, a
  line end is white space like any other. A block's `{` holds statements, so inside it line ends
  count again, however deep in brackets the block is.
- **after an operator or a comma**: an expression that ends a line with `+`, `&&`, `..`, `=`,
  `=>`, `,` or any other operator continues on the next.

So an operator, a call's `(`, an index's `[` or a field's `.` after a line end does **not** continue
the line above. `a` then `- b` on the next line are two statements; `+ b` on the next line is an
error that says to end the line above with the `+`. A line end anywhere in the white space between
two tokens counts, inside a block comment as well.

Only expressions are bound by this. The other parts of a statement may stand on lines of their own:
`else`, a function's `-> T` and `{`, a loop's `{`, a sum type's `|`s.

## Errors

Every syntax error has a line and a column, both counted from 1, the column in characters. Where the
parser knows which rule it is in, after a keyword or inside a bracket, the error is final and names
what that rule needed:

```text
bad.kite:5:1: expected operator or `)` to close `(` at 3:13, found '}'
```

- A missing closing bracket is reported against its opening one, with that one's position. If
  something inside went wrong further on, that is reported instead.
- A string left open at its line end is reported against its opening quote.
- `a < b < c` asks for `&&`; `+ b` at the start of a line says to end the line above with it; a
  keyword where a name belongs is named; an integer too large for `i64`, a bad escape, a `\u{…}`
  that is not a Unicode scalar value and a `/*` never closed are each reported where they begin.

Every syntax error in a file is reported, in the order of the text, one to a line; `kite` then
ends with status 1. After an error the parser resynchronises and reads on:

- a **statement**, a `match` **arm** or an **import** resumes at the end of its line, or before the
  `}` that closes its block. One whose error is at the first token of a later line, `let x =` with
  the value forgotten, resumes at that token, since that line was not the mistake.
- a **declaration** resumes at the next line that begins with `fn` or `type`.
- a line that begins with `fn` or `type` inside a block ends it: the block was left open, which is
  the error, and the declaration is read.
- the skipped text is read as characters, with comments and strings skipped whole and a `{` skipped
  with everything to its `}`.

An error at the end of the file, and an error in the `module` line, end the parse.

```text
bad.kite:3:7: expected operator, end of line or `}`, found 'b'
bad.kite:4:7: expected operator, end of line or `}`, found 'd'
```

## Decisions

Each was made for this first stage, the syntax; the reason follows it.

1. **Kite is interpreted** by a tree-walking interpreter: a checker, then an evaluator, then a
   `kite run`. The parser keeps every node's position for the checker's messages.
2. **Records and variants are built by calls with named arguments**, `Point(x: 1.0, y: 2.0)`. A
   record literal in braces, `Point { x: 1.0 }`, would make `if p { … }` ambiguous, since a
   condition could be a record. Arguments may be named or not; the checker matches them to fields.
3. **A sum type begins with `|`**, every variant after one. `type M = f64` is then an alias and
   `type S = | One` a sum of one variant, with no guessing.
4. **A pattern's lone name is a binding or a variant without fields**, which the checker decides by
   whether a variant of that name is in scope. The parser keeps it as a name.
5. **The left side of an assignment is any expression to the parser**; whether it is a place (a
   name, an element, a field) is the checker's question, reported at the target.
6. **A line end in a block comment ends the line**, as one outside would: the rule is "a line end
   anywhere between two tokens", with no exceptions to learn.
7. **`else` may begin a line.** No statement begins with `else`, so there is nothing for the line
   end to have ended; the same holds for a function's `->`, a loop's `{` and a sum's `|`.
8. **No semicolons.** One statement to a line; two on one line are an error at the second.
9. **A trailing comma is allowed** in every comma list, and a `match` arm may end with a comma
   instead of a line end, so a short `match` fits on one line.
10. **Comparisons and ranges do not chain.** `a < b < c` does not mean what it reads as, and
    `a..b..c` means nothing; both are errors at the second operator.
11. **Ranges bind tighter than comparisons and looser than `+`**: `0..n + 1` is `0..(n + 1)`, and
    `a..b == c..d` compares two ranges.
12. **The smallest `i64` has no literal**: a literal is a non-negative integer and `-` is an
    operator, so `-9223372036854775808` is out of range; it is written `-9223372036854775807 - 1`.
13. **A string holds no raw line end**, and `{` always interpolates; `\{` writes a brace. A
    lone `}` is text. An interpolation's braces are a bracket, so a long expression inside one may
    span lines.
14. **Every file begins with `module`**; an empty file is an error at 1:1.
15. **A block may stand where a condition does**: `while { … } { … }` parses, since a block is an
    expression; whether it is a `bool` is the checker's question.
16. **Kite is its own package** under `examples/kite`, depending on the library by path, so a
    package that depends on the library never compiles the demo. Its tests are run from that
    directory: `avin test .`.
17. **Every syntax error is reported, resynchronising at `fn`, `type`, `}` and line ends.** Settled on
    the user's delegation 2026-11-05. The parser is built with the library's `recover`, which keeps
    a failure as an error, clears the record of what was expected and the commit, and skips with a
    resynchronisation. A statement, an arm and an import resume at the end of their line or before
    their block's `}`; a declaration at the next line beginning `fn` or `type`; and a statement's
    skip refuses to pass a line that begins a declaration, so a block left open is reported once, as
    its missing `}`, rather than as every line after it. An error at the first token of a later line
    skips nothing, because the mistake was the line before. Alternatives weighed: resynchronising at
    tokens rather than lines, which needs a lexer Kite does not have; skipping from where the failed
    statement began rather than from the error, which goes back over text that read correctly and
    stops early in a statement spread over lines; reporting only the first error, as before. Known
    cost: a mistake inside a bracket that spans lines resumes at that line's end, so the bracket's
    next line is read as a statement and may be a second error.
