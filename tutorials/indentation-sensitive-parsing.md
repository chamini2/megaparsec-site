---
title: Indentation-sensitive parsing
subtitle: Native, composable solution
published: January 10, 2016
---

Megaparsec 4.3.0 introduces new combinators that should be of some use when
you want to parse indentation-sensitive input. The combinators do not yet
cover all possible use-cases, but certain class of problems should be
trivial to solve now. This tutorial shows how these new tools work, compose,
and hopefully, *feel natural* — something we cannot say about ad-hoc
solutions to this problem that exist as separate packages to work on top of
Parsec, for example.

1. [Combinator overview](#combinator-overview)
2. [Parsing simple indented list](#parsing-simple-indented-list)
3. [Nested indented list](#nested-indented-list)
4. [Parsing of trivial `do` block](#parsing-of-trivial-do-block)
5. [Conclusion](#conclusion)

## Combinator overview

From the first release of Megaparsec, there has been `indentGuard` thing,
which is a great shortcut, but kind of pain to use for complex tasks. So, we
won't cover it here, instead we will talk about new combinators built upon
it and available beginning from Megaparsec 4.3.0.

First, we have `indentLevel`, which is defined just as:

```haskell
indentLevel :: MonadParsec s m t => m Int
indentLevel = sourceColumn <$> getPosition
```

That's right, it's just a shortcut, but I found myself using this idiom so
often, so I defined this.

Second, we have `nonIndented`. This allows to make sure that some input is
not indented, just wrap parser for that input in `nonIndented` and you're
done.

`nonIndented` is trivial to write as well:

```haskell
nonIndented :: MonadParsec s m Char
  => m ()              -- ^ How to consume indentation (white space)
  -> m a               -- ^ How to parse actual data
  -> m a
nonIndented sc p = indentGuard sc (== 1) *> p
```

However, it's a part of implementation of logical model behind high-level
parsing of indentation-sensitive input. What is this model? We state that
there are top-level items that are not indented (`nonIndented` helps to
define parsers for them), and all indented tokens are directly or indirectly
are “children” to those top-level definitions. In Megaparsec, we don't need
any additional state to express this. Since all indentation is always
relative, our idea is to explicitly tie parsers for “reference” tokens and
indented tokens, thus defining indentation-sensitive grammar via pure
combination of parsers, just like all the other tools in Megaparsec. This is
different from old solutions built on top of Parsec, where you had to deal
with ad-hoc state. It's also more robust and safer, because the less state
you have, the better.

So, how do you define indented block? `indentBlock` should handle this
easily. Let's take a look at its signature:

```haskell
indentBlock :: MonadParsec s m Char
  => m ()              -- ^ How to consume indentation (white space)
  -> m (IndentOpt m a b) -- ^ How to parse “reference” token
  -> m a
```

Here we specify how to consume indentation. Important thing to note here is
that this space-consuming parser *must* consume newlines as well, while
tokens (“reference” token and indented tokens) should not normally consume
newlines after them.

As you can see, the second argument allows us to parse “reference” token and
return a data structure that tells `indentBlock` what to do next. There are
several options:

```haskell
data IndentOpt m a b
  = IndentNone a
    -- ^ Parse no indented tokens, just return the value
  | IndentMany (Maybe Int) ([b] -> m a) (m b)
    -- ^ Parse many indented tokens (possibly zero), use given indentation
    -- level (if 'Nothing', use level of the first indented token); the
    -- second argument tells how to get final result, and third argument
    -- describes how to parse indented token
  | IndentSome (Maybe Int) ([b] -> m a) (m b)
    -- ^ Just like 'ManyIndent', but requires at least one indented token to
    -- be present
```

We can change our mind and parse no indented tokens, we can parse *many*
(that is, possibly zero) indented tokens or require at least one such
token. We can either allow `indentBlock` detect indentation level of first
indented token and use that, or manually specify indentation level. This
should be flexible enough.

Experienced reader will notice that one topic is not covered here: line
folding. This is not yet implemented functionality, but in future it will
be. If you care, you can propose your solution and open a pull request.

## Parsing simple indented list

Now it's time to put our new tools into practice. In this section, we will
parse simple indented list of some items. Let's begin with import section:

```haskell
{-# LANGUAGE TupleSections #-}

module Main where

import Control.Applicative (empty)
import Control.Monad (void)
import Text.Megaparsec
import Text.Megaparsec.String
import qualified Text.Megaparsec.Lexer as L
```

We will need two kinds of space-consumers, one that consumes new lines `sc`
and one that doesn't `sc'` (actually it only parses spaces and tabs here):

```haskell
lineComment :: Parser ()
lineComment = L.skipLineComment "#"

sc :: Parser ()
sc = L.space (void spaceChar) lineComment empty

sc' :: Parser ()
sc' = L.space (void $ oneOf " \t") lineComment empty

lexeme :: Parser a -> Parser a
lexeme = L.lexeme sc'
```

Just for fun, we will allow comments that start with `#` as well.

Assuming `pItemList` parsers entire list, we can define high-level parser
as:

```haskell
parser :: Parser (String, [String])
parser = pItemList <* eof
```

This will make it consume all input.

`pItemList` is a top-level form that itself is a combination of “reference”
token (header of list) and indented tokens (list items), so:

```haskell
pItemList :: Parser (String, [String]) -- header and list items
pItemList = L.nonIndented sc (L.indentBlock sc p)
  where p = do
          header <- pItem
          return (L.IndentMany Nothing (return . (header, )) pItem)

pItem :: Parser String
pItem = lexeme $ some (alphaNumChar <|> char '-')
```

For our purposes, an item is a sequence of alpha-numeric characters and
dashes.

Now, load the code into GHCi and try it with help of `parseTest` built-in:

```haskell
λ> parseTest parser ""
1:1:
unexpected end of input
expecting '-' or alphanumeric character
λ> parseTest parser "something"
("something",[])
λ> parseTest parser "something\none\ntwo\nthree\n"
2:1:
unexpected 'o'
expecting end of input
```

Remember that we're using `IndentMany` option, so empty lists are OK, on the
other hand built-in combinator `space` has hidden space from error messages,
so this error message is perfectly reasonable. Let's continue:

```haskell
λ> parseTest parser "something\n  one\n    two\n  three\n"
3:5:
incorrect indentation
λ> parseTest parser "something\n  one\n  two\n three\n"
4:2:
incorrect indentation
λ> parseTest parser "something\n  one\n  two\n  three\n"
("something",["one","two","three"])
```

This definitely seems to work. Let's replace `IndentMany` with `IndentSome`
and `Nothing` with `Just 5` (indentation levels are counted from 1, so it
will require 4 spaces before indented items):

```haskell
pItemList :: Parser (String, [String])
pItemList = L.nonIndented sc (L.indentBlock sc p)
  where p = do
          header <- pItem
          return (L.IndentSome (Just 5) (return . (header, )) pItem)
```

Now:

```haskell
λ> parseTest parser "something\n"
2:1:
incorrect indentation
λ> parseTest parser "something\n  one\n"
2:3:
incorrect indentation
λ> parseTest parser "something\n    one\n"
("something",["one"])
```

First case may be a bit surprising, but Megaparsec knows that there must be
at least one item in the list, so it checks indentation level and it's 1,
which is incorrect, so it reports it. I find current behavior satisfactory,
but it may be changed in the future.

## Nested indented list

What I like about `indentBlock` is that another `indentBlock` can be put
inside of it and the whole thing will work smoothly, parsing more complex
input with several levels of indentation. No additional effort is required.

Let's allow list items to have sub-items.

## Parsing of trivial `do` block

## Conclusion
