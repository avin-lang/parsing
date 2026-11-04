# parsing

Parser combinators for [Avin](https://github.com/avin-lang/avin). A grammar is a set of Avin values
joined by operators that read like the grammar itself: `p ~ q` is `p` then `q`, `p | q` is `p` or
else `q`, and `p ^^ f` turns what `p` read into a value.

The library is written as literate Avin. Its three source files are Markdown documents that explain
the design as they go, and are worth reading in this order:

1. [`input.lavin`](dev/avinlang/parsing/input.lavin): the text being parsed, positions, and how
   errors are gathered.
2. [`parser.lavin`](dev/avinlang/parsing/parser.lavin): the parser and every combinator.
3. [`leaves.lavin`](dev/avinlang/parsing/leaves.lavin): the parsers that read text directly:
   literals, names, numbers and JSON strings.

## Using it

Name it in your package's `avin.hocon`:

```hocon
dependencies {
  parsing { git = "github.com/avin-lang/parsing", version = "0.2.0" }
}
```

and import from the module `dev.avinlang.parsing`.

## An example

Arithmetic expressions, evaluated as they are parsed: `+` and `-` below `*` and `/`, below a
right-grouping `^`, with unary minus and parentheses.

```avin
import dev.avinlang.parsing.{Parser, chainLeft, chainRight, defer, integer, symbol, token, whitespace}

power(base: long, exponent: long) -> long
    var result: long = 1
    for _ in 0..<exponent
        result *= base
    result

// An operator: the symbol, yielding the function it applies.
op(text: string, f: (long, long) -> long) -> Parser[(long, long) -> long] = symbol(text) ^^^ f

lazy val number: Parser[long] = token(integer)

// The grammar refers to `expr` and `atom` before they are complete, so it reads them through `defer`.
lazy val atom: Parser[long] = number
    | symbol("(") ~> defer(expr) <~ symbol(")")
    | symbol("-") ~> defer(atom) ^^ (n -> -n)

lazy val factor: Parser[long] = chainRight(atom, op("^", power))

lazy val term: Parser[long] = chainLeft(factor, op("*", (a, b) -> a * b) | op("/", (a, b) -> a / b))

lazy val expr: Parser[long] = chainLeft(term, op("+", (a, b) -> a + b) | op("-", (a, b) -> a - b))

lazy val calculation: Parser[long] = whitespace ~> expr

main()
    for text in ["1 + 2 * 3", "2 ^ 3 ^ 2", " -(2 + 3) * -2 ", "2 * (3 + 4", "1 + * 2"]
        calculation.parse(text) match
            Ok(n) -> print(s"$text = $n")
            Err(e) -> print(s"$text: $e")
```

```text
1 + 2 * 3 = 7
2 ^ 3 ^ 2 = 512
 -(2 + 3) * -2  = 10
2 * (3 + 4: 1:11: expected `^`, `*`, `/`, `+`, `-` or `)`, found end of input
1 + * 2: 1:5: expected integer, `(` or `-`, found '*'
```

## Combinators

| Form | Parses | Value |
| --- | --- | --- |
| `p ~ q` | `p` then `q` | `(a, b)` |
| `p ~> q` | `p` then `q` | `q`'s |
| `p <~ q` | `p` then `q` | `p`'s |
| `p ~! q` | `p` then `q`, committed: once `p` has succeeded, `q`'s failure fails the parse | `(a, b)` |
| `p \| q` | `p`, or else `q` from where `p` started | either's |
| `p ^^ f`, `p.map(f)` | `p` | `f(a)` |
| `p ^^^ v` | `p` | `v` |
| `p.flatMap(f)` | `p`, then the parser `f(a)` | that parser's |
| `p.filter(what, pred)` | `p`, failing where `pred` refuses its value | `p`'s |
| `p.label(what)` | `p`, reported as expecting `what` | `p`'s |
| `rep(p)` | `p` as many times as it succeeds, none included | `Vector[T]` |
| `rep1(p)` | the same, at least once | `Vector[T]` |
| `opt(p)`, `p.opt()` | `p` or nothing | `Option[T]` |
| `repSep(p, sep)` | `p` separated by `sep`, none included | `Vector[T]` |
| `rep1Sep(p, sep)` | the same, at least one | `Vector[T]` |
| `chainLeft(p, op)` | `p` separated by `op`, a `Parser[(T, T) -> T]`, combined from the left | `T` |
| `chainRight(p, op)` | the same, combined from the right | `T` |
| `commit(p)`, `p.commit()` | `p`, whose failure fails the parse | `p`'s |
| `positioned(p)`, `p.positioned()` | `p` | `(a, Position)` of where it began |
| `guard(p)` | `p`, reading nothing | `p`'s |
| `notFollowedBy(p)` | nothing, where `p` fails | `()` |
| `defer(p)` | the parser `p` names, found when it first runs | `p`'s |
| `success(v)`, `failure(what)` | nothing; `failure` fails expecting `what` | `v` |

Every name with a `~` in it binds tightest, then `^^` and `^^^`, then `|`, so a rule reads as
written: `a ~ b ^^ f | c ~ d` is `((a ~ b) ^^ f) | (c ~ d)`. A rule of several alternatives spans
lines, each line after the first indented and starting with its `|`.

## Leaf parsers

| Parser | Reads | Value |
| --- | --- | --- |
| `lit("text")` | exactly `text` | `string` |
| `lit('c')` | exactly the character `c` | `char` |
| `satisfy(what, pred)` | one character `pred` accepts | `char` |
| `takeWhile(pred)` | the characters `pred` accepts, none included | `string` |
| `takeWhile1(what, pred)` | the same, at least one | `string` |
| `digits` | ASCII decimal digits, at least one | `string` |
| `ident` | a letter or `_`, then letters, digits and `_`; letters in any script | `string` |
| `whitespace` | white space, none included | `()` |
| `eof` | nothing, at the end of the text | `()` |
| `integer` | an optional `-` and decimal digits, fitting a `long` | `long` |
| `decimal` | a number as JSON writes it: `-`, digits, a fraction, an exponent | `real` |
| `stringLiteral` | a double-quoted string with JSON's escapes, `\u` pairs included | `string` |
| `token(p)` | `p`, then white space | `p`'s |
| `symbol("text")` | `token(lit("text"))` | `string` |

`isDigit`, `isLetter` and `isSpace` are the character tests they use. A parser the library does not
offer is made by giving the function: `Parser(input -> …)`, using `input.atEnd()`,
`input.current()`, `input.offset`, and `input.fail(what)`.

## Running and errors

`p.parse(text)` parses the whole text and answers a `Result[T, ParseError]`; a value that stops
short of the end is an error expecting `end of input`. `p.parsePrefix(text)` parses as much as `p`
takes and answers the byte offset it stopped at.

A `ParseError` is the furthest place any alternative reached, every distinct thing expected there
in the order the grammar tried them, and what was found:

```text
2:5: expected `,` or `]`, found '4'
```

`p.label("value")` reports a failure that got no further than `p`'s first character as expecting
`value`. A commit point, `p ~! q` or `commit(p)`, stops backtracking where the grammar knows which
rule it is in, so the error names what that rule expected.

## Recursive grammars

Module-level rules are `lazy val`s, and a rule that refers to one not yet complete reads it through
`defer`, whose parameter is evaluated the first time the deferred parser runs. Every cycle of rules
needs a `defer` somewhere on it. Each level of nesting is a few calls deep, so the depth a text can
nest is bounded by the stack of the thread parsing it: a few thousand levels of parentheses on the
main thread, more on a thread spawned with a larger stack.

## Tests

```
avin test .
```
