---
title: "Dealing Cards with Unison"
date: "2022-05-21"
toc: false
#author: "Kevin Hoffman"
type: post
description: "Exploring the Unison language by modeling a deck of cards"
subtitle: "Exploring the Unison language by modeling a deck of cards"
categories: 
    - "unison"
---

_I started with a relatively small domain model to experiment with some Unison fundamentals._

<!--more-->
In programming language years, [Unison](https://www.unison-lang.org/) is still just a baby. There are a dozen or more really important points about Unison, but by far the biggest thing you need to know about the language is that it is _content-addressed_.{{< sidenote "sn-unisonv" >}}[Scale by the Bay 2019](https://www.youtube.com/watch?v=IvENPX0MAZ4)<br/>[Houston FPUG](https://www.youtube.com/watch?v=tJR-MvPQhT8){{< /sidenote >}}

This means that code is not identified by name, it is identified by the _hash of its content_. Additionally, code isn't stored as _text files_, it's stored as a _serialized syntax tree_.

These points are kind of dense, so I'll unpack it a bit. One subtle but important aspect of content-addressable code is that the hash pins down the implementation _and its dependencies_, to something that _never changes_. Oh, and **there are no builds**.

When my function depends on another function, it depends on _just that function_. Dependencies in Unison are much more fine-grained than in other languages, and the concept of a library/bundle/package is much more flexible.

Even though Unison is "hashes all the way down", developers still need to type something, so it does have a syntax. I don't know if it's the Haskell-like style of the syntax that made me want to start with a type system, but here's the first set of definitions I entered:

```haskell {linenos=table, hl_lines=["4-5"]}
unique type Suit = Hearts | Clubs | Spades | Diamonds
unique type CardValue = Two | Three | Four | Five | Six | Seven |
            Eight | Nine | Ten | Jack | Queen | King | Ace
unique type Card = Card CardValue Suit
unique type Deck = Deck [Card]
```

For lines 1-3, if you've used TypeScript or any other language that supports union types{{< sidenote "sn-uniontypes" >}}[Wikipedia Reference](https://en.wikipedia.org/wiki/Union_type){{< /sidenote >}} (or enums), then this code should look pretty simple. I'm just identifying that I have a `Suit` type and a `CardValue` type that are the primitive building blocks for the rest of my card model, and the `Card` and `Deck` types build atop those.

Unison doesn't have any (that I could find) convenient `to_string()` equivalent to render a union type so I just defined some easy `render` functions for the types:

```haskell {linenos=table, hl_lines=["17"]}
Suit.render : Suit -> Text
Suit.render suit = 
    match suit with 
        Hearts -> "♥"
        Clubs -> "♣"
        Spades -> "♠"
        Diamonds -> "♦"

CardValue.render : CardValue -> Text
CardValue.render value =
    match value with
        Two -> "2"
        ...

Card.render : Card -> Text
Card.render card =
    (Card value suit) = card
    Suit.render suit ++ CardValue.render value
```

Line 17 is probably the most alien-looking piece of code here. The `(Card value suit)` syntax is a _data constructor_. This produces an instance of the `Card` type. Used on the left hand side of the equality operator here it becomes a _destructuring pattern match_, and I am actually extracting the `value` and `suit` values from the card. I then use the extracted values to generate a string which is the concatenation of both types' `render` function. So a two of spades will render as `♠2`.

In the code below you can see a function that also uses pattern matching to extract the list of cards from a `Deck` instance:

```haskell {linenos=table}
Deck.cards deck =
    match deck with 
        Deck c -> c
```

The first thing I want to be able to do in my model is generate a standard deck of 52 cards. If you're unfamiliar with this type of deck, there are 13 cards of each suit, and there are 4 suits.

The folks in the Unison slack were kind enough to help me refactor my newbsauce code into something more elegant. The following code really shows off some of the simple elegance you can achieve in this language (I originally used a `flatMap` and a nested `map` to produce the same effect):

```haskell {linenos=table}
Deck.standard: Deck
Deck.standard = 
    Deck (Each.toList 'let
        suit = each allSuits
        value = each allValues
        Card value suit)
```

Here, I'm calling `Each.toList` on the result of an explicit deferred execution{{< sidenote "sn-defex" >}}Think of this as a closure in other languages. It's code-as-data that will, upon request, be executed. Here, Each.toList makes such a request.{{< /sidenote >}} block indicated by the `'let` syntax. The `each` is producing or yielding each element in `allSuits` and `allValues` (both of which are lists). When `Card value suit` is called, it will be called, "for each suit for each value", effectively generating the 52-element list I need for a full standard deck. Finally, the `[Card]` result from `Each.toList` is used as the constructor argument for the `Deck`.

There is something semi-hidden happening here called an _ability_, which I'll talk about a bit later in this post.

Now that I had a full deck of cards, I wanted to be able to deal cards out of the deck. I want to be able to see the cards dealt and get the reduced deck back (rememember in the pure functional world, nothing is mutable in place).

I'm going to create a recursive function. It will start out with an empty "accumulator"{{< sidenote "sn-fold" >}}If you're thinking we could use something like a left fold here, you're right. Doing so, however, would likely confuse both me and the reader. I've chosen to favor readability over tersity.{{< /sidenote >}} and in each iteration remove a card from the deck and add it to the accumulator. When the function hits its exit condition, I'll have a tuple containing a list of cards I drew from the deck and the new deck.

```haskell {linenos=table, hl_lines=["6"]}
Deck.deal : Nat -> ([Card], Deck) -> {Random} ([Card], Deck)
Deck.deal n tup =  
  (hand, (Deck deck)) = tup  
  if (n == 0 || (deck === List.empty)) then tup else
    max = List.size deck  
    index = Random.natIn 0 max
    card = List.unsafeAt index deck
    remainingDeck = Deck (List.deleteAt index deck)
    Deck.deal (n - 1) ((card +: hand), remainingDeck)
```

On line one, we see `{Random}` up in the type signature. This is an _ability_. I'll likely spend an entire forthcoming blog post on them, but for now you can think of them as ways to accomplish side-effects while still  maintaining pure functional purity. They feel far more accessible to me than monads do in other languages.

{{< figure
  src="/images/is_this_a_monad.jpeg"
  class="class param"
  title="Butterfly Meme"
  caption="I couldn't resist"
  label="mn-meme"  
  alt="Butterfly meme"  
 >}}
{{< section "end" >}}

You can read that type signature as, "The Deck.deal function takes a natural number and a tuple of a list of cards and a deck, and, in order to return a tuple containing a list of cards and a deck, it requires the `Random` ability to access a random number generator."

Line 6 contains the actual use of the ability via `Random.natIn`.

This function exposes its recursive internals by requiring the tuple accumulator as a parameter. We can provide a slightly easier to use function that is the "starter" for our recursive function:

```haskell
Deck.draw : Nat -> Deck ->{Random} ([Card], Deck)
Deck.draw n deck =
    Deck.deal n ([], deck)
```

In order to call this function, or the inner `deal` function, we have to _provide_ a _handler_ for the `Random` ability. The following two lines show using the `lcg`{{< sidenote "sn-fold" >}}LCG here stands for [linear congruential generator](https://en.wikipedia.org/wiki/Linear_congruential_generator){{< /sidenote >}}
random number generator with a fixed seed to deal from the deck:

```haskell
Random.lcg 42 '(Deck.deal 5 ([], fullDeck))
Random.lcg 99 '(Deck.deal 5 ([(Card Ten Hearts)], fullDeck)
```
In the preceding code, anything in the deferred execution block passed as the final parameter to `Random.lcg` will have access to the lcg implementation of the `Random` ability. This means the body of `Deck.deal` will use fixed-seed LCG to get its random numbers.

This is great so far, but in the real world we can't use fixed seeds. We'd rather do something like use the system time as the seed for the generator. In Unison, we can use the `systemTime` function to get this. Let's take a look at the type signature for this:

```haskell
base.io.systemTime : '{IO, Exception} EpochTime
```

`EpochTime` is a unique data type that takes a single `Nat` as the data constructor. This means we'll have to use a pattern like `(EpochTime t)` to extract the `Nat`.

We've seen the `{Ability}` syntax already, but the `systemTime` function requires both the `IO` and the `Exception` abilities. What's interesting about this is that when you're using the `ucm` CLI, watch expressions don't have access to either of these abilities.

The solution here is to write a `main` funtion that we can then invoke via the ucm `run` command.

```haskell
testMain : ' {IO, Exception} ()
testMain _ =
    newdeck = 'let
        (EpochTime t) = !systemTime
        Random.lcg t '(Deck.draw 4 fullDeck))
    (cards, deck) = !newdeck
    printLine (List.map Card.render cards |> Text.join ",")
```

The `testMain` function has both the `IO` and `Exception` abilities, which is precisely what we need. Also note that this function does not require the `Random` ability, because it instead _provides_ this ability inside the deferred execution block assigned to the `newdeck` variable.

Finally, with all of this in place, I can use the following `ucm` command to execute `testMain`:

```
.> run testMain   
♥3,♠10,♥2,♠A
```

This was my first real foray into the language. I'd taken a look at the docs and played with it a bit once or twice, but this was the first "real" thing I made with it.

I definitely need to do some blog posts and explorations on Unison's ability system and its distributed system capabilities, but I'm having fun for now. The Unison creators share my desire to make the developer experience _delightful_ and I can definitely see the potential here.