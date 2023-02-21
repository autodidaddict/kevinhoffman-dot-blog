---
title: "wasmCloud and Unison Cloud"
date: "2023-02-15"
toc: false
#author: "Kevin Hoffman"
type: post
description: "Contemplating the similar inspirations behind wasmCloud and Unison Cloud"
subtitle: "Birds of a Feather?"
categories: 
    - "wasmcloud"
    - "unison"
    - "webassembly"
---

_A look at the types of developer scenarios enabled by wasmCloud and Unison Cloud to see where there are areas of overlap and similar inspiration._

<!--more-->

I've done quite a bit of posting on the topic of [unison](/categories/unison/) on this blog. One aspect of unison that I haven't really covered here is the notion of the _unison cloud_. [Unison cloud](https://unison.cloud), at least in its current incarnation, is an opaque vessel into which you deploy code and then remotely execute it. You write a function, push your namespace into your cloud, then execute that function. Unison will take care of figuring out the most appropriate node (or nodes) on which to execute your function and get you the result. It feels very much like on-demand distributed compute, like an ad hoc map/reduce or even a lambda/function-as-a-service.

[wasmCloud](https://wasmcloud.com) allows you to build an opaque vessel into which you can deploy compute (WebAssembly modules or "actors") by stitching together any number of disparate infrastructures together over a flattened [NATS](https://nats.io) topology that wasmCloud calls a **lattice**. You can think of wasmCloud actors as little WebAssembly lambdas, or you can think of them like something smaller than a microservice, e.g. a nanoservice.

Both of these tools have been created with the explicit purpose of making distributed computing fun and easy, alleviating the burden of becoming an expert in low-level distsys from developers and letting the platform take care of the ugly bits like resiliency, scale, logging, metrics, observability, execution, consistency, and more.

## The Capability Model
wasmCloud embraces the capability model by making actors communicate with non-functional requirements (aka system services, side effects, I/O) by _contract_ rather than explicitly. This means that a wasmCloud actor, compiled as a WebAssembly module, makes function calls against abstractions like a key-value store, message broker request, HTTP client request, HTTP server request handler, and much more. This forced abstraction gives developers the flexibility to satisfy their capabilities at runtime with whatever they choose, giving them the flexibility to use in-memory/test/mock capability providers when testing or executing their inner development loop and more robust products when deployed to production--all without recompiling or refactoring.

Here's an example of what it looks like for a wasmCloud actor to make a request of the key-value capability provider, without ever knowing which vendor is being used, the connection string, or any other secrets or configuration data:

```rust
let kv = KeyValueSender::new();
let new_value = kv.increment("mykey", 1)?;
```

As I mention in my blog post exploring [Unison's abilities](../exploring_unison_abilities), unison has a very similar concept. The main difference is that Unison abilities are granted to individual functions in-process, while wasmCloud's ability mechanism is managed out of process because of the external host runtime.

In unison, I might have an ability that looks like this:

```haskell
unique ability KeyValue where
  get : String ->{Keyvalue} String
  increment : String -> {KeyValue} Nat
```

If we lived in the magical alternate universe where unison compiled to WebAssembly, I could write a function like the following and expect the host runtime to plug in the appropriate ability the same way that wasmCloud does:

```haskell
WasmCloud.keyvalue 'let
    increment "mykey" 1
```

Here the `WasmCloud.keyvalue` function creates a unison _handler_ that, instead of providing the functionality directly, would instead provide the functionality by communicating with the host WebAssembly runtime (in this case, wasmCloud).

If I wanted multiple capabilities available to my actor function:

```haskell
WasmCloud.keyvalue 'let
    WasmCloud.messaging 'let
        newvalue = increment "mykey" 1
        publish newvalue "sample.data"
```

The unison syntax is even flexible enough that I could create both of these providers with alternate [link names](https://wasmcloud.com/docs/reference/host-runtime/links):

```haskell
WasmCloud.keyvalue "cache" 'let
    WasmCloud.messaging "internal" 'let
        newvalue = increment "mykey" 1
        publish newvalue "sample.data"
```

💡 Unison abilities and wasmCloud capabilities give developers a way to reason about this kind of behavior without needing to invoke the word `monad`, which some people fear as much as speaking aloud the name of he-who-shall-not-be-named.

## Unison Cloud as a Capability Provider
Another thought occurred to me as I was writing this blog post. What if wasmCloud actors could gain client access to a unison cloud by virtue of a capability provider? The contract might be something fairly generic like `wasmcloud:remotecode` that accepts a function name and parameters. With a capability like this, I could submit unison code to execute a function in my unison cloud and obtain the results in my actor, all without knowing how many nodes had to do work to reduce my call to an answer.

## Final Thoughts

Given how nascent unison cloud is, a provider granting wasmCloud actors to a unison cloud wouldn't be very practical right now. However, I would gladly sever a limb to be able to write Unison code that compiles to a WebAssembly module usable as an actor that can use wasmCloud capability providers as though they were native unison abilities.
