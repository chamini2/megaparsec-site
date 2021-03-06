---
title: Fun with the recovery feature
subtitle: Megaparsec has grown even more powerful in version 4.4.0
published: May 14, 2016
---

Megaparsec 4.4.0 is a major improvement of the library. Among other things,
it provides new primitive combinator `withRecovery` that allows to recover
from parse errors “on-the-fly” and report several errors after parsing is
finished or ignore them altogether. In this tutorial, we will learn how to
use this incredible tool.

1. [Language that we will parse](#language-that-we-will-parse)
2. [Parser without recovery](#parser-without-recovery)
3. [Making use of the recovery feature](#making-use-of-the-recovery-feature)
4. [Conclusion](#conclusion)

## Language that we will parse

For the purposes of this tutorial, we will write parser for a simplistic
functional language that consists only of equations with symbol on the left
hand side and arithmetic expression on the right hand side:

```
y = 10
x = 3 * (1 + y)

result = x - 1 # answer is 32
```

Something like that. Here, it can only calculate arithmetic expressions, but
if we were to design something more powerful, we could introduce more
interesting operators to grab input from console, etc., but since our aim is
to explore new parsing feature, this language will do.

First, we will write parser that can parse entire program in this language
as list of ASTs representing equations. Then we will make it
failure-tolerant in a way, so when it cannot parse particular equation, it
does not stop, but continues its work until all input is analyzed.

## Parser without recovery

The parser is very easy to write. We will need the following imports:

```haskell
{-# LANGUAGE FlexibleContexts #-}
{-# LANGUAGE TypeFamilies     #-}

module Main where

import Control.Applicative (empty)
import Control.Monad (void)
import Data.Scientific (toRealFloat)
import Text.Megaparsec
import Text.Megaparsec.String
import Text.Megaparsec.Expr
import qualified Text.Megaparsec.Lexer as L
```

To represent AST of our language we will use these definitions:

```haskell
type Program = [Equation]

data Equation = Equation String Expr deriving (Eq, Show)

data Expr
  = Value          Double
  | Reference      String
  | Negation       Expr
  | Sum            Expr Expr
  | Subtraction    Expr Expr
  | Multiplication Expr Expr
  | Division       Expr Expr
  deriving (Eq, Show)
```

↑ Here, it's now obvious that a program in our language is collection of
equations, where every equation gives name to expression which in turn can
be simply a number, reference to other equation, or some math involving
those concepts.

As usual, first thing that we need to handle when starting a parser is white
space. We will have two space-consuming parsers:

* `scn` — consumes newlines and white space in general. We will use it for
  white space between equations, which will start with a newline (since
  equations are newline-delimited).

* `sc` — this does not consume newlines and is used to define lexemes,
  i.e. things that automatically eat white space after them.

Here is what I've got:

```haskell
lineComment :: Parser ()
lineComment = L.skipLineComment "#"

scn :: Parser ()
scn = L.space (void spaceChar) lineComment empty

sc :: Parser ()
sc = L.space (void $ oneOf " \t") lineComment empty

lexeme :: Parser a -> Parser a
lexeme = L.lexeme sc

symbol :: String -> Parser String
symbol = L.symbol sc
```

Consult Haddocks for description of `L.space`, `L.lexeme`, and
`L.symbol`. In short, `L.space` is a helper to quickly put together
general-purpose space-consuming parser. We will follow this strategy:
*assume no white space before lexemes and consume all white space after
lexemes*. There is a case with white space that can be found before any
lexeme, but that will be dealt with specially, see below.

We also need a parser for equation names (`x`, `y`, and `result` in the
first example). Like in many other programming languages, we will accept
alpha-numeric sequences that do not start with a number:

```haskell
name :: Parser String
name = lexeme ((:) <$> letterChar <*> many alphaNumChar) <?> "name"
```

All too easy. Parsing of expressions could slow us down, but there is a
solution out-of-box in `Text.Megaparsec.Expr` module:

```haskell
expr :: Parser Expr
expr = makeExprParser term table <?> "expression"

term :: Parser Expr
term = parens expr
  <|> (Reference <$> name)
  <|> (Value     <$> number)

table :: [[Operator Parser Expr]]
table =
  [ [Prefix (Negation <$ symbol "-") ]
  , [ InfixL (Multiplication <$ symbol "*")
    , InfixL (Subtraction    <$ symbol "/") ]
  , [ InfixL (Sum            <$ symbol "+")
    , InfixL (Division       <$ symbol "-") ]
  ]

number :: Parser Double
number = toRealFloat <$> lexeme L.number

parens :: Parser a -> Parser a
parens = between (symbol "(") (symbol ")")
```

We just wrote fairly complete parser for expressions in our language! If
you're new to all this stuff I suggest you load the code into GHCi and play
with it a bit. Use `parseTest` function to feed input into the parser:

```haskell
λ> parseTest expr "5"
Value 5.0
λ> parseTest expr "5 + foo"
Sum (Value 5.0) (Reference "foo")
λ> parseTest expr "(x + y) * 5 + 7 * z"
Sum
  (Multiplication (Sum (Reference "x") (Reference "y")) (Value 5.0))
  (Multiplication (Value 7.0) (Reference "z"))
```

Power! The only thing that remains is parser for entire equations and parser
for entire program:

```haskell
equation :: Parser Equation
equation = Equation <$> (name <* symbol "=") <*> expr

prog :: Parser Program
prog = between scn eof (sepEndBy equation scn)
```

Note that we need to consume leading white-space in `prog` manually, as
described above. Try the `prog` parser — it's a complete solution that can
parse language we described in the beginning. Parsing “end of file” `eof`
explicitly makes the parser consume all input and fail loudly if it cannot
do it, otherwise it would just stop on the first problematic token and
return what it has parsed so far.

## Making use of the recovery feature

Our parser is really dandy, it has nice error messages and does its job
really well. However, every expression is clearly separated from the others
by a newline. This separation makes it possible to analyze many expressions
independently, even if one of them is malformed, we have no reason to stop
and not to check the others. In fact, that's how some “serious” parsers work
(parser of C++ language, although it depends on compiler I guess). Reporting
multiple parse errors at once may be more efficient method of communication
with programmer that needs to fix them than when he has to recompile the
program every time to get to the next error. In this section we will make
our parser failure-tolerant and able to report multiple error messages at
once.

Let's add one more type synonym — `RawData`:

```haskell
type RawData t e = [Either (ParseError t e) Equation]
```

This will represent collection of equations, just like `Program`, but every
one of them may be malformed: in that case we get original error message in
`Left`, otherwise we have properly parsed equation in `Right`.

You will be amazed just how easy it is to add recovering to existing parser:

```haskell
rawData :: Parser (RawData Char Dec)
rawData = between scn eof (sepEndBy e scn)
  where e = withRecovery recover (Right <$> equation)
        recover err = Left err <$ manyTill anyChar eol
```

Let try it, here is the input:

```
foo = (x $ y) * 5 + 7.2 * z
bar = 15
```

Result:

```
[ Left
   (ParseError
     { errorPos = SourcePos
       { sourceName = "", sourceLine = Pos 1
       , sourceColumn = Pos 10} :| []
       , errorUnexpected = fromList [Tokens ('$' :| "")]
       , errorExpected = fromList
         [ Tokens (')' :| "")
         , Label ('o' :| "perator")
         , Label ('r' :| "est of expression") ]
       , errorCustom = fromList [] })
, Right (Equation "bar" (Value 15.0)) ]
```

How does it work? `withRecovery r p` primitive runs parser `p` as usual, but
if it fails, it just takes its `ParseError` and provides it as argument of
`r`. In `r` you start right were `p` failed — no backtracking happens,
because it would make it harder to find position from where to start normal
parsing again. Here you have a chance to consume some input to advance
parser's textual position. In our case it's as simple as eating all input up
to the next newline, but it might be trickier.

You probably want to know now what happens when recovering parser `r` fails
as well. The answer is: your parser fails as usual, as if no `withRecovery`
primitive was used. It's by design that recovering parser cannot influence
error messages in any way, or it would lead to quite confusing error
messages in some cases, depending on logic of recovering parser.

Now it's up to you what to do with `RawData`. You can either take all error
messages and print them one by one, or ignore altogether and filter only
valid equations to work with.

## Conclusion

When you want to use `withRecovery`, the main thing to remember that parts
of text that you want to allow to fail should be clearly separated from each
other, so recovering parser can reliably skip to the next part if current
part cannot be parsed. In a language like Python, you could use indentation
levels to tell apart high-level definitions, for example. In every case you
should use your judgment and creativity to decide how to make use of
`withRecovery`. In some cases it may be not worth it, but more often than
not you will be able to improve experience of people who work with your
product by using this new Megaparsec's feature.
