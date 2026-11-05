# parsing

Parser combinators for [Avin](https://github.com/avin-lang/avin). A grammar is a set of Avin values
joined by operators that read like the grammar itself: `p ~ q` is `p` then `q`, `p | q` is `p` or
else `q`, and `p ^^ f` turns what `p` read into a value.

The library is written as literate Avin. Its three source files are Markdown documents that explain
the design as they go, and are worth reading in this order:

1. [`input.lavin`](dev/avinlang/parsing/input.lavin): the text being parsed, positions, white space,
   and what a parse answers.
2. [`parser.lavin`](dev/avinlang/parsing/parser.lavin): the parser, every combinator, and running a
   grammar.
3. [`leaves.lavin`](dev/avinlang/parsing/leaves.lavin): the parsers that read text directly:
   characters, literals, names, numbers and strings.

## Using it

Name it in your package's `avin.hocon`:

```hocon
dependencies {
  parsing { git = "github.com/avin-lang/parsing", version = "0.5.1" }
}
```

and import from the module `dev.avinlang.parsing`.

## An example

Arithmetic expressions, evaluated as they are parsed: `+` and `-` below `*` and `/`, below a
right-grouping `^`, with unary minus and parentheses. White space between tokens is skipped.

```avin
import dev.avinlang.parsing.{Error, Failure, Parser, Success, chainl1, chainr1, literal, parseAll, wholeNumber}

power(base: long, exponent: long) -> long
    var result: long = 1
    for _ in 0..<exponent
        result *= base
    result

// An operator: the literal, yielding the function it applies.
op(text: string, f: (long, long) -> long) -> Parser[(long, long) -> long] = literal(text) ^^^ f

lazy val number: Parser[long] = wholeNumber ^^ (s -> s.toLongOption().getOrElse(0))

lazy val atom: Parser[long] = number
    | literal("(") ~> expr <~ literal(")")
    | literal("-") ~> atom ^^ (n -> -n)

lazy val factor: Parser[long] = chainr1(atom, op("^", power), power, 1)

lazy val term: Parser[long] = chainl1(factor, op("*", (a, b) -> a * b) | op("/", (a, b) -> a / b))

lazy val expr: Parser[long] = chainl1(term, op("+", (a, b) -> a + b) | op("-", (a, b) -> a - b))

main()
    for text in ["1 + 2 * 3", "2 ^ 3 ^ 2", " -(2 + 3) * -2 ", "2 * (3 + 4", "1 + * 2"]
        parseAll(expr, text) match
            Success(n, _) -> println(s"$text = $n")
            Failure(msg, next) -> println(s"$text: ${next.pos()}: $msg")
            Error(msg, next) -> println(s"$text: ${next.pos()}: $msg")
```

```text
1 + 2 * 3 = 7
2 ^ 3 ^ 2 = 512
 -(2 + 3) * -2  = 10
2 * (3 + 4: 1:11: expected `^`, `*`, `/`, `+`, `-` or `)`, found end of input
1 + * 2: 1:5: expected whole number, `(` or `-`, found '*'
```

## Combinators

The methods of a parser:

| Form | Parses | Value |
| --- | --- | --- |
| `p ~ q` | `p` then `q` | `(a, b)` |
| `p ~> q` | `p` then `q` | `q`'s |
| `p <~ q` | `p` then `q` | `p`'s |
| `p ~! q` | `p` then `q`, committed: once `p` has succeeded, `q`'s failure fails the parse | `(a, b)` |
| `p \| q` | `p`, or else `q` from where `p` started | either's |
| `p \|\|\| q` | `p` and `q` from the same place, whichever reads further; `p` when they tie | either's |
| `p ^^ f`, `p.map(f)` | `p` | `f(a)` |
| `p ^^^ v` | `p` | `v` |
| `p ^? f` | `p`, failing where the `PartialFunction` `f` is not defined at its value | `f(a)` |
| `p.^?(f, error)` | the same, the failure expecting `error(a)` | `f(a)` |
| `p.flatMap(f)`, `p into f`, `p >> f` | `p`, then the parser `f(a)` | that parser's |
| `p.filter(pred)`, `p.withFilter(pred)` | `p`, failing where `pred` refuses its value | `p`'s |
| `p.named(n)` | `p`, which shows as `Parser (n)` | `p`'s |
| `p.withFailureMessage(msg)` | `p`, reported as expecting `msg` when it fails at its first character | `p`'s |
| `p.withErrorMessage(msg)` | `p`, reported as expecting `msg` when it fails past a commit point | `p`'s |

The functions that make a parser of others:

| Form | Parses | Value |
| --- | --- | --- |
| `rep(p)` | `p` as many times as it succeeds, none included | `List[T]` |
| `rep1(p)` | the same, at least once | `List[T]` |
| `rep1(first, p)` | `first`, then `p` as many times as it succeeds | `List[T]` |
| `repN(n, p)` | `p` exactly `n` times | `List[T]` |
| `repsep(p, q)` | `p` separated by `q`, none included | `List[T]` |
| `rep1sep(p, q)` | the same, at least one | `List[T]` |
| `opt(p)` | `p` or nothing | `Option[T]` |
| `chainl1(p, q)` | `p` separated by `q`, a `Parser[(T, T) -> T]`, combined from the left | `T` |
| `chainl1(first, p, q)` | `first`, then `p` after each `q`, a `Parser[(T, U) -> T]`, combined from the left | `T` |
| `chainr1(p, q, combine, first)` | `x₀ q₁ x₁ … qₙ xₙ`, folded from the right: `combine(x₀, f₁(x₁, … fₙ(xₙ, first)))` | `U` |
| `commit(p)` | `p`, whose failure fails the parse | `p`'s |
| `positioned(p)` | `p`, after white space, its value (a `Positional`) positioned where `p` began unless it already has a position | `p`'s |
| `guard(p)` | `p`, reading nothing | `p`'s |
| `not(p)` | nothing, where `p` fails; what `p` expected is no part of a failure's message | `()` |
| `phrase(p)` | `p`, then white space, then the end of the text | `p`'s |
| `log(p, name)` | `p`, printing where it was tried and what it answered | `p`'s |
| `success(v)` | nothing | `v` |
| `failure(msg)` | nothing: fails, expecting `msg` | |
| `err(msg)` | nothing: fails past a commit point, expecting `msg` | |

Every name with a `~` in it binds tightest, then `^^`, `^^^` and `^?`, then `|` and `|||`, so a
rule reads as written: `a ~ b ^^ f | c ~ d` is `((a ~ b) ^^ f) | (c ~ d)`. A rule of several
alternatives spans lines, each line after the first indented and starting with its `|`.

## Leaf parsers

The character parsers read exactly what they are told to, white space included:

| Parser | Reads | Value |
| --- | --- | --- |
| `elem(kind, pred)` | one character `pred` accepts, described as `kind` | `char` |
| `elem(c)`, `accept(c)` | exactly the character `c` | `char` |
| `acceptIf(pred, err)` | one character `pred` accepts, failing as expecting `err(c)` | `char` |
| `acceptMatch(expected, f)`, `accept(expected, f)` | one character the `PartialFunction` `f` is defined at | `f(c)` |
| `acceptSeq(cs)`, `accept(list)` | exactly the characters given, in order | `List[char]` |

The token parsers skip white space first:

| Parser | Reads | Value |
| --- | --- | --- |
| `literal("text")` | exactly `text` | `string` |
| `ident` | a letter or `_`, then letters, digits and `_`; letters in any script | `string` |
| `wholeNumber` | an optional `-` and decimal digits | `string` |
| `decimalNumber` | digits with an optional `.` and digits, or `.` and digits | `string` |
| `floatingPointNumber` | an optional `-`, a decimal number, an optional exponent, an optional `f`, `F`, `d` or `D` | `string` |
| `stringLiteral` | a double-quoted string with `\"`, `\'`, `\\`, `\b`, `\f`, `\n`, `\r`, `\t`, `\uXXXX` | `string`, as written |

The numbers and strings answer their text, for the grammar to convert to the type it wants.
The character tests for `elem` and `acceptIf` are the core `char` methods: `c.isDigit()`, `c.isLetter()`, `c.isLetterOrDigit()`, `c.isWhitespace()`. A parser the library
does not offer is made by giving the function: `Parser(input -> …)`, using `input.atEnd()`,
`input.first()`, `input.offset`, `input.handleWhiteSpace()` and `input.fail(what)`.

## Running a grammar, and white space

`parse(p, text)` reads as much of `text` as `p` takes. `parseAll(p, text)` is `parse(phrase(p),
text)`: it must read all of it, white space at the end included. Both answer a `ParseResult`:

| Case | Meaning |
| --- | --- |
| `Success(result, next)` | the value, and the input where the parser stopped |
| `Failure(msg, next)` | the message, and the input at the furthest place any alternative reached |
| `Error(msg, next)` | the same, past a commit point, so no other alternative was tried |

with `r.successful()`, `r.isEmpty()`, `r.get()` (which traps on a failure), `r.getOrElse(v)`,
`r.map(f)` and `r.next()`, whose `pos()` is the line and column. A result shows as
`[1:3] parsed: 42`, ``[2:5] failure: expected `,` or `]`, found '4'`` or `[1:2] error: …`.

The message is the furthest place any alternative reached, every distinct thing expected there in
the order the grammar tried them, and what was found. `withFailureMessage("value")` reports a
failure that got no further than a rule's first character as expecting `value`. A commit point,
`p ~! q`, `commit(p)` or `err(msg)`, stops backtracking where the grammar knows which rule it is in,
so the error names what that rule expected.

White space belongs to the grammar. The module's `parse` and `parseAll` skip white space in any
script; a grammar that wants otherwise is an object that is a `RegexParsers`, overrides
`whiteSpace()` or `skipWhitespace()`, and is run with its own `parse` and `parseAll`:

```avin
object Config: RegexParsers
    // Comments run from `#` to the end of the line, and count as white space.
    override whiteSpace() -> Parser[unit] = rep(elem("white space", c -> c.isWhitespace()) ^^^ () | literal("#") ~ rep(elem("comment", c -> c != '\n')) ^^^ ()) ^^^ ()

    lazy val entry: Parser[(string, string)] = ident ~ (literal("=") ~> ident)

main()
    println(Config.parseAll(rep(Config.entry), "a = b  # one\nc = d"))
```

An `override skipWhitespace() -> bool = false` makes a grammar read every character itself.

## Recursive grammars

Module-level rules are `lazy val`s, and they may refer to each other and to themselves in any
order: every combinator takes the parsers it is given by name, and reads one the first time the
parser it made runs, when every rule is complete. The receiver of a method is the exception, made
when the method is called, so a rule may not begin with itself, which would loop in any case.

Each level of nesting is a few calls deep, so the depth a text can nest is bounded by the stack of
the thread parsing it: a few thousand levels of parentheses on the main thread, more on a thread
spawned with a larger stack.

## Kite: a demo language

[`examples/kite`](examples/kite) is a small interpreted language, Kite, whose parser is written with
this library: [`SPEC.md`](examples/kite/SPEC.md) is its grammar, operator table and line-end rule,
and [`kite/grammar.lavin`](examples/kite/kite/grammar.lavin) is that grammar as a literate Avin
file. It shows a grammar owning its white space (nested comments, and line ends that end
statements except inside brackets or after an operator), a string's interpolation as a switch
between character parsers and token parsers, two modes of one set of expression rules, and errors
that name the bracket left open: ``expected `)` to close `(` at 3:13``. It is a package of its own,
so nothing that depends on this library compiles it:

```
cd examples/kite
avin test .
avin run . -- samples/geometry.kite
```

## Changes in 0.5.1

`Positional` holds the position itself, in a field, so a node class conforms to it and writes
nothing else:

```avin
class Name(val text: string): Positional
```

| 0.5.0 | 0.5.1 |
| --- | --- |
| each class kept a `var` and overrode `pos()` and `setPos(p)` | the trait's field `var pos: Option[Position]`, read as `n.pos` and assignable |
| `setPos(p)` set the position as each class wrote it | the trait's `setPos(p)` sets it only while there is none |

A trait holding a field is conformed to by a class or an object, so a positioned node type is a
class.

## Changes in 0.5.0

| 0.4.0 | 0.5.0 |
| --- | --- |
| `notFollowedBy(p)` | `not(p)` |
| `p into f` only | `p >> f` as well, at the precedence of a shift |
| `rep`, `rep1`, `repN`, `repsep`, `rep1sep` answer a `Vector[T]` | they answer a `List[T]`; the pairs of `~` are as before |
| `positioned(p)` answers `(T, Position)` | `positioned(p)` needs `T: Positional` and answers `T`, with its position set unless already set |
| `floatingPointNumber` ends at the exponent | it also takes a trailing `f`, `F`, `d` or `D` |
| `isDigit`, `isLetter`, `isSpace` exported | the core `char` methods `isDigit()`, `isLetter()`, `isLetterOrDigit()`, `isWhitespace()`; `isDigit` accepts a digit of any script |
| `print` in the examples | `println`, since `print` no longer ends the line |

## Changes in 0.4.0

The library's names now follow the standard vocabulary of parser combinators. A grammar written for
0.3.0 changes as follows.

| 0.3.0 | 0.4.0 |
| --- | --- |
| `p.parse(text)` → `Result[T, ParseError]` | `parseAll(p, text)` → `ParseResult[T]` |
| `p.parsePrefix(text)` → `Result[(T, int), ParseError]` | `parse(p, text)`, whose `next().offset` is where it stopped |
| `ParseError(position, expected, found)` | `Failure(msg, next)` and `Error(msg, next)`, `next.pos()` the position |
| `chainLeft(p, op)` | `chainl1(p, op)` |
| `chainRight(p, op)` | `chainr1(p, op, combine, first)`, folding into `first` |
| `repSep`, `rep1Sep` | `repsep`, `rep1sep` |
| `p.label(what)` | `p.withFailureMessage(what)` |
| `p.filter(what, pred)` | `p.filter(pred).withFailureMessage(what)` |
| `p.opt()`, `p.commit()`, `p.positioned()` | `opt(p)`, `commit(p)`, `positioned(p)` only |
| `lit("text")` | `literal("text")`, which skips white space first |
| `lit('c')` | `elem('c')` or `accept('c')` |
| `satisfy(what, pred)` | `elem(what, pred)` |
| `takeWhile(pred)`, `takeWhile1(what, pred)`, `digits` | `rep(elem(what, pred))`, `rep1(elem(what, pred))` |
| `integer`, `decimal` | `wholeNumber`, `floatingPointNumber`, answering the text |
| `stringLiteral`, answering the decoded text | `stringLiteral`, answering the text as written |
| `token(p)`, `symbol("text")`, `whitespace` | white space skipped by the token parsers, set by a `RegexParsers` |
| `eof` | `phrase(p)` |
| `input.text`, `input.current()` | `input.source`, `input.first()` |

New: `p ||| q`, `p ^? f`, `p into f`, `p.withFilter(f)`, `p.named(n)`, `p.withErrorMessage(msg)`,
`rep1(first, p)`, `repN(n, p)`, `chainl1(first, p, q)`, `phrase(p)`, `log(p, name)`, `err(msg)`,
`elem`, `accept`, `acceptIf`, `acceptMatch`, `acceptSeq`, `decimalNumber`,
`RegexParsers` with `whiteSpace()` and `skipWhitespace()`, and `ParseResult` with `Success`,
`Failure` and `Error`.

## Tests

```
avin test .
```
