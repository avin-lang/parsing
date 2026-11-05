# Kite

Kite is a small statically typed language run by a tree-walking interpreter. It exists to show
what the [parsing](../../README.md) library does with a real grammar: its parser,
[`kite/grammar.lavin`](kite/grammar.lavin), is the grammar below written as Avin values. A program
is a module of functions and types:

```kite
module geometry

import io

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
        io.println("shape {i} has area {area(shapes[i])}")
    }
}
```

Blocks are braces, a line end ends a statement, and blocks, `if` and `match` are expressions whose
value is their last expression's. The types are `i64`, `f64`, `bool`, `string`, fixed arrays
`[T; n]`, records and sum types. There are no generics, traits, closures or overloading, and no
garbage collector to think about: values are copied. `kite check` reports every mistake in a file
and `kite run` checks a program and runs it; [Semantics](#semantics) says what each construct
means and what the checker refuses.

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

Every syntax error in a file is reported, in the order of the text, one to a line, by `kite run`,
`kite check` and `kite parse` alike; a file with a syntax error is not checked, and `kite` ends
with status 1. After an error the parser resynchronises and reads on:

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

## Semantics

[`kite/check.avin`](kite/check.avin) is the checker and [`kite/eval.avin`](kite/eval.avin) the
evaluator. The checker reports every mistake it finds, in the order of the text, as
`file:line:column: message`; a program with any is not run.

### Programs, modules and names

- **A file is a program.** `kite run` calls its `main`, which takes no parameters and returns
  nothing; a file without one is an error at 1:1. The module's name is free.
- **`import` names a built-in module**: `io`, with `print(text: string)` and
  `println(text: string)`, and `math`, with `abs(x: f64) -> f64` and `sqrt(x: f64) -> f64`. A
  module's functions are called through its name, `io.println("…")`, and only once it is imported;
  a module is not a value.
- **A module has two namespaces.** Types: aliases, records and sums. Values: functions, records
  (whose name builds one), variants and imported modules. A name is declared once in each; a
  record's name is in both. Variants are not qualified by their sum, so two sums cannot have a
  variant of one name. `i64`, `f64`, `bool` and `string` cannot be declared. A field or a
  parameter is declared once in its record, variant or function.
- **Locals** are parameters, `let` and `var` names, a `for`'s variable and a pattern's names. A
  local is in scope from the statement after its own to the end of its block; a `for`'s variable
  in the loop's body, a pattern's names in their arm. **A local never shadows a local**: it may not
  have the name of a parameter or a local in scope. Blocks side by side may each have their own
  `x`, and a local may hide a function, a record, a variant or a module. A name is looked up among
  the locals first.

### Types

- `i64`, `f64`, `bool`, `string`, `[T; n]` with `n` at most 2^24, records and sums. An alias stands
  for its type and may not reach itself; a record or a sum may hold itself, so `type L = | End |
  Link(v: i64, next: L)` is a list.
- An integer literal is an `i64` and a float literal an `f64`. Nothing converts implicitly:
  `f64(n)` makes an `f64` of an `i64` and `i64(x)` an `i64` of an `f64`, truncating toward zero.
- **`()`** is the type of a function without `-> T`, of a call of one, and of an `if` without
  `else`. It has no name in a program. `a..b` and `a..=b` are of a type with no name, a range of
  `i64`s, which a `let` may hold and a `for` walk. `[]` fits an array of any type of length 0.
- `[x; n]` takes an integer literal for `n`, since the length is part of the type.

### Expressions

| Operators | Operands | Result |
| --- | --- | --- |
| `+` | two `i64`s, two `f64`s or two `string`s, which it joins | the same |
| `-` `*` `/` `%` | two `i64`s or two `f64`s | the same |
| `<` `<=` `>` `>=` | two `i64`s, two `f64`s or two `string`s, by code point | `bool` |
| `==` `!=` | two values of any one type | `bool` |
| `&&` `\|\|` | two `bool`s; the right one is evaluated only when it decides | `bool` |
| prefix `-`, `!` | an `i64` or an `f64`; a `bool` | the same |

- **Equality is by value**: arrays, records and variants part by part, reals as IEEE 754 compares
  them (`0.0 == -0.0`, and NaN equals nothing), and two ranges when they hold the same integers
  (`0..3 == 0..=2`).
- **A call** names a function, a record or a variant, `Point(x: 1.0, y: 2.0)`. Its arguments are
  positional, then named; each parameter or field is given once, and every one is given. They are
  evaluated in the order written.
- `p.x` reads a record's field. A sum's fields are reached only through `match`.
- **A block's value** is its last statement's, when that is an expression, and `()` otherwise; one
  that ends in `return`, `break` or `continue` never finishes, and fits any type. An `if` with
  `else` has its branches' type, which must be one; a `match` its arms'.
- **Where a value is discarded**, as a statement, as the body of a function without `-> T` or of a
  loop, an `if`'s or a `match`'s branches may have any types.
- **An interpolation** takes any value but `()`, and writes an `i64` in decimal; an `f64` in the
  fewest digits that read back as it, with `.0` where they would read as an integer, and as `inf`,
  `-inf` or `NaN`; a `bool` as `true` or `false`; a string as its text, and inside an array, a
  record or a variant as a literal, quoted; an array as `[1, 2]`; a record or a variant as
  `Point(x: 1.0, y: 2.0)`, a variant without fields as its name; a range as `0..3` or `0..=3`.

### Statements

- `let` binds a name for good and `var` one that may be assigned. **An assignment's target is a
  place**: a `var`, or an element or a field of one, to any depth, `g[i].ends[0].x = 1.0`. A
  parameter, a `for`'s variable and a pattern's name cannot be assigned. A compound assignment
  takes the operands its operator does, the target's type being the left one's.
- An assignment evaluates the target's indices left to right, then the value, then, for a compound
  one, the operator, and stores the result; an index outside the array is found at the store.
- `while` takes a `bool`. `for` walks a range, its variable an `i64`, or an array, its variable each
  element in turn; the range's ends or the array are evaluated once, before the first pass, and a
  range to the largest `i64` ends there. `break` and `continue` act on the innermost loop, and
  only inside one.
- `return` takes a value of the function's result type, and none in a function without one. A
  function with `-> T` has a body whose value fits `T`; a function without one discards its body's
  value.

### match

- A pattern is `_`; a literal of the matched value's type; a name, which is a variant without fields
  where a variant of that name is declared and otherwise binds the value; or `V(p, …)`, a variant
  or a record with one pattern for each of its fields, in order.
- **Every value must be covered.** The checker names one that is not, as a pattern:
  `` this `match` does not cover every `Shape` value, such as `Rect(_, _)` ``. An `i64`, an `f64`
  or a `string` is covered only by an arm that matches anything.
- **An arm no value can reach is an error**: `this arm can never match`.
- A real pattern compares with `==`: `0.0` matches `-0.0`, and nothing matches NaN.

### Values

Every value is copied. Assigning an array or a record, passing it or putting it in another makes a
copy, and changing one never changes another: `var b = a` and then `b[0] = 9` leaves `a` as it was.

### Runtime errors

These stop the program with `file:line:column: runtime error: message` and status 1, after the
output it had already written:

- an `i64` result that does not fit, from `+`, `-`, `*`, `/` (the one quotient, of the smallest
  `i64` by `-1`), prefix `-` or a compound assignment: `` integer overflow in `+` ``. `%` cannot
  overflow: the smallest `i64` `% -1` is 0;
- an `i64` divided by zero, by `/` or `%`: `division by zero`. An `f64` follows IEEE 754, so
  `1.0 / 0.0` is `inf`;
- an index outside the array, reported at the index: `index 3 is out of bounds for an array of
  length 3`;
- `i64(x)` of an `x` an `i64` cannot hold: `` `i64` cannot hold NaN ``;
- calls nested more than 100,000 deep: `stack overflow: calls nested more than 100000 deep`.

A `match` that no arm matches cannot happen in a checked program.

### The command

`kite run file.kite` checks a program and runs it, `kite check file.kite` only checks it, and
`kite parse file.kite` prints its syntax tree. Each prints its mistakes and ends with status 1
where there are any, and with status 2 when it is not given a command and one file.

## Decisions

The first seventeen were made for the first stage, the syntax; the rest for the checker and the
evaluator. The reason follows each.

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
    next line is read as a statement and may be a second error. A file with a syntax error is never
    checked: the checker would otherwise report mistakes in a tree the parser read past one.
18. **One file is one program, and `import` names a built-in module** (`io`, `math`). Importing
    another Kite file would need a type from it to be named, and a type is a plain name in the
    grammar, with no `module.Type`; growing the grammar for it is a later stage's choice. Review
    with the user.
19. **Output is `io.print` and `io.println`, taking a `string`**; interpolation turns any value into
    one. A prelude `print` taking any value was the alternative; an import keeps the program's
    dependencies visible, and one parameter type needs no overloading. The samples, which called
    `print`, now import `io`.
20. **A local never shadows a local**, but may hide a top-level name. Shadowing within a function,
    `let x = x + 1` as a new `x`, was the alternative; refusing it makes every name in a function
    mean one thing, and the error names the first declaration. Review with the user.
21. **Variants are not qualified by their sum**, since a pattern's lone name is a variant
    (decision 4) and `Shape.Circle` would need a grammar for qualified names. So one module's
    variants have distinct names. An alias names a type only, and is not called: `Point(…)` builds
    a record through its own name.
22. **A discarded value is checked as wanting `()`**, and anything fits that, since no type a
    program can write is `()`: so an `if` or a `match` used as a statement may have branches of
    different types, and a function without `-> T` may end in any expression. Requiring `()`
    there was the alternative; it would make `if c { x += 1 } else { log(x) }` an error whenever
    `log` answers something.
23. **Every value is immutable inside the evaluator**: an assignment to an element or a field
    builds a new array or record, sharing the rest, and stores it in the variable. That is copying
    every value, as the language says, without a copy at every `let`, call and return.
24. **An assignment evaluates its target's indices, then its value**, left to right as written,
    and checks the indices at the store; a compound assignment evaluates them once. Evaluating the
    value first was the alternative; target first is the order the text reads in.
25. **Two ranges are equal when they hold the same integers**, so every empty range equals every
    other; equal bounds and kind was the alternative, under which `0..3 != 0..=2`.
26. **An unreachable arm is an error**, as Kite has no warnings; an arm that can never match is a
    mistake in the program as surely as a missing one.
27. **Calls nest at most 100,000 deep**, the evaluator running on a thread of its own with a 4 GiB
    stack (reserved, and used only as deep as the program goes), so the limit is Kite's runtime
    error with a position rather than the host's stack overflow, which names no place. No limit
    was the alternative. Review with the user if a deeper limit is wanted.
28. **An array has at most 2^24 items**, checked where its length is written, so `[0; n]` cannot
    ask the evaluator for more memory than a demo should take.
29. **`i64(x)` of a real it cannot hold is a runtime error**, not a saturated or wrapped result:
    NaN, the infinities and anything outside the `i64` range have no `i64` to be.
30. **The checker's messages come in the order of the text**, sorted by position, whatever order
    the checker found them in (it reads declarations before bodies).
31. **Interpolating `()` is an error**: it has nothing to show, and an `if` without `else` inside
    `{…}` is more likely a missing `else` than a wish to print `()`.
