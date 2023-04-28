---
title: "Unison Exercism Track - Day 1"
date: "2023-04-28"
toc: false
#author: "Kevin Hoffman"
type: post
description: "My first jump into the deep end of Unison's exercism track"
subtitle: "Being humbled by Unison and Exercism"
categories:     
    - "unison"
    - "exercism"
---

_My first exercise with Unison and its Exercism track was both humbling and rewarding._

<!--more-->

I'm not including the "Hello World" exercise here because its goal is more to get you set up with the exercism pipeline and not to actually teach you anything or test real knowledge. The next problem I attempted was called **Acronym**. The following is the description of the exercise:

> _Convert a phrase to its acronym. Techies love their TLA (Three Letter Acronyms)! Help generate some jargon by writing a program that converts a long name like Portable Network Graphics to its acronym (PNG). Punctuation is handled as follows: hyphens are word separators (like whitespace); all other punctuation can be removed from the input._

The sample conversions showing what will pass the tests are (I added the last one to reinforce the idea that extraneous punctuation cannot have an effect on the acronym):

| Input | Output |
| -- | -- |
| As Soon As Possible | ASAP |
| Liquid-crystal display | LCD |
| Thank George It's Friday | TGIF |
| Portable \_network\_ graphics | PNG |

At first blush, I think that I'll want to strip all non-splitting punctuation from the input. Then, uppercase it all. Then, split on either whitespace or a hyphen, take the first character of each word and then finally rejoin into a string.

My single biggest difficulty was in finding the various functions on the `Text` type. Using unison share was confusing for me and I had to ask for help in their slack a number of times to work my way through it. 

Here is what I came up with after multiple iterations and discussions with folks on the Unison slack. I think this was my fourth or fifth revision:

```haskell
abbreviate : Text -> Text
abbreviate input = let    
  input    
    |> dropWhile (is (not (anyOf [?\s, ?-] + alphanumeric)))    
    |> segmentBy ((c -> (c Char.== ?\s) || (c Char.== ?-)) >> not)     
    |> List.map (elem -> Text.toUppercase (Text.take 1 elem))    
    |> Text.join "" 
```

The functional pipeline basically goes:
1. `drop(not space, -, or alphanumeric)`
1. `split(on space or -)`
1. `take 1st char`
1. `uppercase`
1. `join`.

The function call `anyOf [?\s, ?-] + alphanumeric` uses some DSL-like functionality (`anyOf`) in the `Char` type.

The final version that I got working on Exercism looks like this:

```haskell
abbreviate : Text -> Text
abbreviate input =
  (+) = or
  input
    |> Text.segmentBy (is (in "'" + not (whitespace + punctuation)))
    |> map (Text.take 1)
    |> Text.join ""
    |> Text.toUppercase
```

The `(+) = or` is required because that particular alias isn't in the current version of the base library. That can be removed shortly after the next update happens (I'm assuming).

What I think helps with readability here is that I'm not using any of the special "monad-ish" functions like `<<` or `<|` that I had in a few intermediate solutions. I think this code is readable enough for someone who doesn't sling Unison all day to understand.

⚠️ **Fun fact**: _both `punctuation` and `isPunct` consider the apostrophe punctuation, but we don't want to split on apostrophes, so it has to be explicitly ignored_

## Wrap-Up
It's been a long time since I've stumbled and failed so miserably to build out such a simple solution. I think there are a couple of factors, the largest of which is that I just don't get to play with Unison frequently enough to build up the same kind of muscle memory that I have with Rust and Go and Elixir.

I greatly enjoyed learning and "failing with purpose" and am eagerly looking forward to the next exercise that I can find time to work through.