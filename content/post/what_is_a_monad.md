---
title: "What is a Monad?"
date: "2024-07-12"
toc: false
#author: "Kevin Hoffman"
type: post
description: "What the heck is a Monad"
subtitle: "And why should we care?"
categories: 
    - "fp"
    - "haskell"
    - "monads"
---
_An attempt to explain Monads from the perspective of a pragmatist with no time for Category Theory_.

<!--more-->

I run into this question a lot. Over the years, I've attempted to use Haskell "in anger"{{< sidenote "sn-anger" >}}When we say that we've used something "in anger", we mean that we've used it to do hard and/or practical things that go beyond tutorials or "hello world" samples.{{< /sidenote >}} a number of times. I'm sad to report that most of those attempts were abysmal failures. Some of these failures were because the community's (at least the portion I was exposed to) elitist and condescending attitude drove me away. Other times, the failures were just because I had no idea what I was doing.{{< sidenote "sn-doing" >}}Narrator: **he still doesn't**{{< /sidenote >}}

I'm not going to start with a definition. Instead, I'm going to build up to an explanation by showing you some fairly common _imperative_ programming patterns using Go{{< sidenote "sn-go" >}}Don't try and compile this. This is basically psuedocode{{< /sidenote >}}.

I'm building an application and I know ahead of time that I'm likely going to need to pass a bunch of contextual data to my functions. Let's assume that I'm not using Go structs to store that context, and I'm just writing functions.

The easiest thing to do is use `context.Context`, an idiom that shows up in just about every Go code base.

```go
// This function makes me a pile of cash
func MakeItRain(ctx context.Context) (int, error) {
    return nil
}
```

I quickly exceed the limited capabilities of Go's `Context`, and I want strongly typed contextual information, not just an arbitrary map. That's fine, I'll add some more parameters to my function to help me with this:

```go
func MakeItRain(ctx context.Context, 
    myBank Bank, 
    cash CashSource, 
    stocks StockSource) (int, error) {
    balance := myBank.Balance + cash.Withdraw(100)
    balance += stocks.Withdraw(100)
    
    return balance, nil
}
```

This is fine, but I also need the user identity:

```go
func MakeItRain(ctx context.Context,
    myBank Bank,
    cash CashSource,
    stocks StockSource,
    user UserIdentity,
) (int, error) { ... }
```

Carrying around all these parameters is a huge pain in the butt, even for one function. But what if I have this entire chain of functions that I want to execute, and they all need the same set of contextual information? 

In Go, we can take advantage of [closures](https://gobyexample.com/closures) to capture variables that are in scope. We can use this to create a _wrapper function_ that allows my inner function to _capture_ the `UserIdentity` context:

```go
type MakeItRainFunc func(Env) (int, error)

type Env struct {
    user UserIdentity
}

func WithIdentity(user UserIdentity, f MakeItRainFunc) MakeItRainFunc{
    env := Env{user: user}

    return f(env)
}

... 
func makeItRain(env Env) (int, error) {
    ...
}
...
// withIdentity returns a function, not a result, so we
// need to execute it...
munny, err := WithIdentity(user, makeItRain)()
```

There's a very important change (one might call it an inversion) in perspective here. Now, rather than `user` being an explicit parameter to my function, it's available in a single context.

We can now add more wrappers that will continue to build up the necessary context needed for my function:

```go
munny, err := WithBank(bank,
                WithCash(cash,
                    WithStocks(stocks,
                        WithUser(user, makeItRain))))()
```
Now we have a function that captures context and calls another, and that one captures context and calls another, etc until we finally get to the function that needs all that context: `makeItRain`.

We're stretching things a bit here because in Go we don't normally do this to build up context, we just run a bunch of imperative functions in sequence, but bear with me, I might actually make a point soon.

This is where things will get messy in Go. What If I now have a series of functions I want to run, in sequence, that all perform some actions on the various portfolio contexts available? If any one of these actions in the sequence fails, we need to abort. For Go devs, this means a pile of `if err != nil { ... }` boilerplate, and for Rust devs it's a bunch of `.and_then(...)` chains.

In Haskell, I might be able to define this process as follows{{< sidenote "sn-haskell" >}}This is also psuedocode that I don't expect to compile{{< /sidenote >}}:

```haskell
balancePortfolio :: MoneyM m => ... (hidden for clarity)
balancePortfolio = do
    _ <- withdrawCash 100
    _ <- withdrawStocks 5
    return balance
```
What we're looking at is a function that requires that it be run "inside" the `Money` monad. You might invoke `balancePortfolio` like this:

```haskell
ghci> runBanking kevinsAccount $ do balancePortfolio
```

The `runBanking` function here creates the monad, accepts `kevinsAccount` as the context, and then invokes `balancePortfolio` as an inner function that captured its outer context via lambda.

The `MoneyM` type defines the `withdrawCash` and `withdrawStocks` functions, which are kinda sorta like captured variables from a closure, but these variables are functions, which have also been able to capture variables from their own surroundings.

The Haskell `do` notation is syntactic sugar for stuff that still looks too spaghetti tangled for me to enjoy reading, but it also gives us built-in primitives to allow the sequence of calls that occur within this context to be able to short-circuit and abort early.

_"But everything in Haskell is immutable",_ you might be thinking, _"how does withdraw actually change data?"_

There's a couple of ways this happens. One way is through an `IORef` which is kind of like a pointer, and you can mutate things under/referenced-by the IO ref with functions (like an `Agent` in Elixir or an "inner mutability" pattern implemented in Rust).

The other way is by (this is a _gross_ oversimplification) way of recursive functions. To change "state" here, you simply call the same function with a different value, e.g. `withdrawCash b n = withdrawCash (b-n) n`.

When we're using Monads, we're basically saying "run my function, which _requires_ a particular shape of context, inside this monad function, which _provides_ a suitably shaped context".

Monads are powerful on their own (you Rustaceans might like to know that `Option` and `Result` would be "monadic" types in a functional language). Where things get truly insane is when you combine monads. At the simplest level, you might combine a bunch of monads in a chain that can abort early without you having to use ugly constructs like `break` or `continue`:

```haskell
thing = do
    runThis
    <*> andThat
    <$> andThis
    <$> thenThat
```

The strange symbols here are shortcuts for the monad{{< sidenote "sn-applicative" >}}Note: I'm not going to dwell on the difference between Applicative and Functor and Monad here, those things usually cause people to flee in droves during first exposure{{< /sidenote >}} combination functions.

Combining monads lets you do things like `map` a function like `(+)` to `Just 3` and `Just 5` to produce `Just 8`. If you think about the fact that you can make any of your own types combinable and monadic, the possibilities get really amazing (and the code you read that does this gets _really confusing_). Similarly, you could `map` the `withdraw` function to `Just kevinsAccount` and `Just bobsAccount`.

The other thing you'll see _a ton_ in Haskell code using monads is the concept of `lift`. This is basically what you use to say "run this non-monadic function as though it were a monad". Again, once you see codebases littered with `lift`, things get significantly harder to read.

To recap, monads are a way of providing context to functions by wrapping them with other functions and making clever use of closures and variable capture where variables are also functions. If you want to _provide_ some service, capability, or context to some logically related group of function calls, monads are the tool to do it.

**Disclaimer**: this blog post represents the way I think about monads in order to not get overwhelmed by the massive amount of academic content on the web for Haskell and category theory. My definitions are not likely to satisfy a purist or even a weak-principled dabbler. However, this perspective does help me make sense of things as I read (sometimes), experiment, and learn.