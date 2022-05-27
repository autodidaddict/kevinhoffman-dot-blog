---
title: "Exploring Unison Abilities"
date: "2022-05-26"
toc: false
#author: "Kevin Hoffman"
type: post
description: "Taking a deeper dive into the Unison ability system"
subtitle: "Taking a deeper dive into the Unison ability system"
categories: 
    - "unison"
---

_I attempt to learn and explore the Unison ability system without turning and running away in fear of all things monadic_.{{< marginnote "mn-example" >}}I've had a number of deeply painful experiences from condescending and gatekeeping developers when it came to monads. This has contributed greatly to my aversion to Haskell.{{< /marginnote >}}

<!--more-->

In my [previous post](../unison_cards) on Unison, I modeled a deck of cards with some very basic functionality. One of the functions I wrote that dealt a hand from a deck required access to a random number generator.

Doing something like accessing a random number generator, accessing the system clock, reading or writing files or to the console, are all things that can be labeled algebraic effects{{< sidenote "sn-effects" >}}If you're curious about how effects are handled in [wasmCloud](https://wasmcloud.dev), check out [this blog post](https://wasmcloud.com/blog/caps_are_effects/){{< /sidenote >}}. Effects typically take the purity of a function and smear mud all over it.

Unison's abilities keep the messy side effects managed separately from your pure functions. As an example, the signature of the `deal` function from the previous blog post indicates all of the ability requirements of that function:

```haskell
Deck.deal : Nat -> ([Card], Deck) -> {Random} ([Card], Deck)
```
After a bit of getting used to, Unison type signatures become pretty easy to read. This one indicates that, given a natural number and a tuple, it will make use of the `Random` ability to return a new tuple.

On the `Random` ability is the `natIn` function. This means that anywhere inside a function that requires `Random`, I can call the `natIn` function and be certain that _some implementation provided by the consumer_ will satisfy my requirements for a random number.

What if we want to _emit_ cards to a stream as they are dealt? This stream could be anything. It could end up being gathered into a list, but it could also be a message broker where the cards are published on a given topic, perhaps even for an online multiplayer game. The great part about it is that the `deal` function can make use of the `Stream` ability (part of Unison's base library), and let consumers of the function choose what kind of stream they're going to use.

We can modify the `deal` function so that it requires both a `Random` ability and a `Stream` ability (which takes a type parameter). Here's the new version of that function:

```haskell {linenos=table, hl_lines=["6", "8"]}
Deck.deal : Nat -> ([Card], Deck) -> {Stream Card, Random} ([Card], Deck)
Deck.deal n tup =
  (hand, (Deck deck)) = tup
  if (n == 0 || (deck === List.empty)) then tup else
    max = List.size deck
    index = natIn 0 max
    card = List.unsafeAt index deck
    emit card
    remainingDeck = Deck (List.deleteAt index deck)
    Deck.deal (n - 1) ((card +: hand), remainingDeck)
```
On line 6 we're using the `natIn` function from the `Random` ability like we did before, but on line 8, we're using the `emit` function from the `Stream` ability. We don't care _how_ the value is emitted, only that it _is_.

This gives us a little bit more trouble when we try and call this function, because now we need to ensure that the calling scope has both of our required abilities.

Let's take a look at the new `testMain` function that supplies both abilities to the `Deck.draw` function:

```haskell {linenos=table, hl_lines=["8"]}
testMain : ' {IO, Exception} ()
testMain _ =
  dealt_hand : '([Card], Deck)
  dealt_hand = 'let
    (EpochTime t) = !systemTime
    dealt = '(Deck.draw 4 fullDeck)
    Random.lcg t dealt
  (cards, deck) = (Stream.trace dealt_hand)
  printLine (List.map Card.render cards |> Text.join ",")
```
There are a couple of subtle changes here that required me to enlist the help of some people in the Unison slack. In this new code, `dealt_hand` is a delayed computation that returns another delayed computation. This delayed computation still requires both the `Random` and the `Stream` abilities.

What's important to notice is that we pass the `dealt_hand` delayed computation as an argument to the `Stream.trace` function (which I'll show shortly). I go back and forth between understanding how this works and being completely befuddled. It can be a little confusing figuring out that calling `Stream.trace` with the delayed execution actually returns the result of the delayed computation.

The code for `Stream.trace` shows how to implement an ability _handler_:

```haskell {linenos=table, hl_lines=["4","8"]}
Stream.trace : '{g, Stream a} r -> {g} r
Stream.trace s = 
  h = cases
    { Stream.emit a -> k } -> 
      Debug.trace "Emitted " a
      handle !k with h
    { r } -> r
  handle !s with h
```
The type signature `Stream.trace : '{g, Stream a} r -> {g} r` means that this function takes a delayed function that returns type `r` (which can also utilize a generic ability `g`) and returns the executed result of type `r`. This is how `trace` can wrap an algebraic effect around a function and still preserve that function's return value.

Line 4 illustrates a _request constructor_. This is a pattern match for when the function to which we're providing the ability requests a function from the ability--in this case the `emit` function. When the function requests an `emit`, we invoke `Debug.trace`. If this were something like a message broker handler, we might do something like `Broker.publish` on a given topic.

When we see `handle !k with h` inside the definition for `h`, we're using the handler recursively. The `k` here is a _continuation_. Think of it like a pointer to the rest of the code that occurs after the wrapped function made its ability request. If we continue handling, we move the code forward, but we can also do things like choose to stop execution, as is the case in the `Abort` ability.

Lastly, on line 8 above, the handler `h` is invoked for the first time on the executed result of the delayed computation that was passed as an argument, which is of type `'{g, Stream a} r`. You could also write this as `() -> {g, Stream a} r` if that helps with the mental model around delayed execution.

Now when I execute `run testMain` inside ucm, I get output that looks like the following (your output will vary because it's random!):

```
.> run testMain
trace: Emitted 
Card Ace Spades
trace: Emitted 
Card Two Hearts
trace: Emitted 
Card Jack Diamonds
trace: Emitted 
Card Six Clubs
♣6,♦J,♥2,♠A
```
This ability stuff is very dense, and I find that it's taking me a while to fully get my head around it all. The more I get exposed to it, the more I understand and the more it completely blows my mind ... and then the next day I feel like I've learned nothing.

I recently wrote a blog post on [algebraic effects](https://wasmcloud.com/blog/caps_are_effects/) in an open source project I created called wasmCloud. In that post, I talk about a sample domain problem of performing an international account withdrawal, and the difficulty in separating the pure business logic from the side effects. The psuedocode for that looked like this:

```
internationalWithdrawal account amount localCurrency =
    exchangeRate = Market.getRate(localCurrency)
    newAmount = amount * exchangeRate
    fee = Market.getFee(localCurrency)
    Ledger.withdraw(account, amount, newAmount, exchangeRate)
    Ledger.fee(account, fee)
    Ledger.balance(account)
```
It should be pretty straight forward to translate the above psuedocode into a function in Unison that requires two abilities: `Ledger`, which provides mutable access to a customer's account ledger, and `Market`, which provides access to the current state of the financial markets, including exchange rates and transaction fees:

```haskell
internationalWithdrawal : Text -> Nat -> Currency -> {Ledger, Market} Nat
internationalWithdrawal account amount localCurrency =
    exchangeRate = getRate localCurrency
    newAmount = exchangeRate * amount -- for brevity I didn't do a float here
    fee = getFee localCurrency
    withdraw account amount newAmount exchangeRate
    balance account
```
This gives us a chance to see the declaration of an ability, which is just the interface containing the list of functions and types of requests that can be made of the ability--not the actual implementation:

```haskell
unique ability Market where
  getRate : Currency ->{Market} Nat
  getFee : Currency -> {Market} Nat

unique ability Ledger where
  withdraw : Text -> Nat -> Nat -> Nat -> {Ledger} ()
  fee : Text -> {Ledger} Nat
  balance : Text -> {Ledger} Nat
```
It's important to remember that this isn't like invoking a function on an implementation of an interface like we would in an object-oriented language. When a function makes a _request_ of an ability, it _pauses_, the ability is given the request, and then the ability can choose to continue the supplied _thunk_ {{< sidenote "sn-thunk" >}}[Thunk](https://en.wikipedia.org/wiki/Thunk) is a name for a specific type of delayed computation{{< /sidenote >}}
 or terminate, or even select a different handler for the continuation.

While a real handler for `Market` might be involved and linked to real-time data sources and feeds, a mock one is fairly easy to write:

```haskell {linenos=table}
Market.mock : '{Market, g} r -> {g} r
Market.mock m =
    h = cases 
        { Market.getRate currency -> k } -> handle k 100 with h
        { Market.getFee currency -> k } -> handle k 12 with h
        { r } -> r
    handle !m with h
```
In this mock, the `getRate` function always returns `100` and the `getFee` function always returns `12`. Now let's mock up a `Ledger` ability:

```haskell {linenos=table}
Ledger.mock : '{Ledger, g} r -> {g} r
Ledger.mock m =
    h = cases
        { Ledger.withdraw account amount newAmount exchangeRate -> k } -> handle k () with h
        { Ledger.fee account -> k } -> handle k 5 with h
        { Ledger.balance account -> k } -> handle k 75 with h
        { r } -> r
    handle !m with h
```
So far I've found that the most difficult syntax to figure out is how to actually provide an ability to a deferred execution block. Note that in both of the mocks here, the ability requirements include a generic ability `g`. This makes it so that additional abilities can be supplied to nested blocks--which we need.

This is the syntax I used to supply the mocks to a call to `internationalWithdrawal`:

```haskell
> Ledger.mock '(
    Market.mock '(internationalWithdrawal "ABC123" 100 (Currency "USD")))
```
The `>` means it's a _watch expression_ that `ucm` will evaluate immediately. Because the balance is always `75`, the value of this watch expression is `75`.

I can also clean up the syntax a bit:
```haskell
Ledger.mock 'let
  Market.mock 'let
    internationalWithdrawal "ABC123" 100 (Currency "USD")
```
I could probably go on for many more pages about Unison's abilities but I'll wrap it up here for this post. Overall my impression of abilities is that they're more accessible than monads, and they're far better for reliable software than aspect-oriented programming or dependency injection. This could also be me looking at a new language through rose-colored glasses, so we'll see if I still feel like this when I finally get to the "using Unison in anger" stage.