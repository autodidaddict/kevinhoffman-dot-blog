---
title: "Microservices with wasmCloud and Unison Cloud"
date: "2023-04-22"
toc: false
#author: "Kevin Hoffman"
type: post
description: "Both wasmCloud and Unison cloud strive for a better future for microservices"
subtitle: "The future of microservices"
categories: 
    - "wasmcloud"
    - "unison"
    - "webassembly"
    - "microservices"
---

_Unison is working on a new way of thinking about and building microservices for Unison Cloud. In this post, I'll take a look at that approach and how it lines up with what wasmCloud and Cosmonic are building_.

<!--more-->

In their recent post, [reimagining the microservice](https://www.unison-lang.org/whats-new/unison-services-preview/), the folks at Unison give us a sneak preview of what they're working on to let people build microservices on Unison cloud. Unsurprisingly, this looks _an awful lot_ like what wasmCloud lets you do. Let's walk through some of the code to see the similarities in both vision and implementation.

First, let's take a look at their basic hello world service (I've removed the deployer code to make the comparison easier):

```haskell
hello : HttpRequest -> HttpResponse
hello req = HttpResponse.ok (Body (Text.toUtf8 "Hello, world!"))
```

As it should be, this is just a function that takes in an HTTP request and returns an HTTP response. It cares about the _purpose_ of the abstraction and not about the _execution_ of that abstraction. There's no references to port numbers, TLS, SSL, threading models, proxies, or any other stuff that this code shouldn't have to care about.

Let's look at the same as a wasmCloud component in Rust:

```rust
async fn handle_request(&self, _ctx: &Context, req: &HttpRequest) -> 
    Result<HttpResponse, RpcError> {       
        Ok(HttpResponse::ok(b"Hello, world!"))
}
```

Aside from the presence of a `Context` in the Rust implementation, they're very nearly language-to-language ports of each other.

Next in Unison's post, they illustrate how a deployed microservice function can make use of other abilities. As I've mentioned before, Unison abilities are spiritual counterparts to wasmCloud's capability providers. Here's an example of a Unison service using the `Log` ability:

```haskell
logic : HttpRequest ->{Log} HttpResponse
logic req = 
  log "Waving"
  HttpResponse.ok (Body (Text.toUtf8 "👋, world!"))
```

What I love about Unison's ability system is that it's so explicit. The type signature of the function is explicit that it utilizes the `Log` ability, and it's right there at the top so when you're skimming through code at the end of a long day of debugging, it's still obvious which abilities a function uses. Compare this to the Rust implementation of the same service in wasmCloud:

```rust
async fn handle_request(&self, _ctx: &Context, req: &HttpRequest) -> 
    Result<HttpResponse, RpcError> {
        info!("Waving");
        Ok(HttpResponse::ok(b"👋, world!"))
}
```

Here we get to leverage Rust's macro system, but under the hood there's still just a call to `logger.log(Level::Info, ...)`. Because Rust doesn't have an innate ability concept, we don't gain the benefit of seeing which capability providers any function uses. However, wasmCloud [signs](https://wasmcloud.com/docs/reference/host-runtime/security#actor-identity) actors cryptographically, authorizing them to use a set of capabilities so we can still easily look that up.

Next let's take a look at the canonical wasmCloud data service, the key value counter. This service takes a request which contains the name of the counter to increment, and it returns an HTTP response containing JSON with that new counter.

```rust
async fn handle_request(&self, ctx: &Context, req: &HttpRequest) -> RpcResult<HttpResponse> {
    let trimmed_path: Vec<&str> = req.path.trim_matches('/').split('/').collect();
     match (req.method.as_ref(), trimmed_path.as_slice()) {
        ("GET", ["api", "counter"]) => increment_counter(ctx, "default", 1).await,
        ("GET", ["api", "counter", counter]) => increment_counter(ctx, counter, 1).await,
        (_, _) => Ok(HttpResponse::not_found()),
    }
}
async fn increment_counter(ctx: &Context, counter: &str, value: i32) -> RpcResult<HttpResponse> {
    let key = format!("counter:{}", counter.replace('/', ":"));

    let (body, status_code) = match KeyValueSender::new()
        .increment(ctx, &IncrementRequest { key, value })
        .await
    {
        Ok(v) => (json!({ "counter": v }).to_string(), 200),        
        Err(e) => (json!({ "error": e.to_string() }).to_string(), 500),
    };

    Ok(HttpResponse::json(body))
}
```

What strikes me here is how, again, the use of the capability blends (IMHO) too much into the background and you have to explicitly look for the use of the `KeyValueSender` struct to know that this code is using that ability instead of a comparable SQL query or something else.

I'll try and translate this into Unison, but don't try and compile it because a) my blog is a poor compiler and b) this stuff isn't publicly accessible yet.

```haskell
kvcounter : HttpRequest -> {Log, KeyValue} HttpResponse
kvcounter req = 
    getDefaultCount = do
      Routes.get (root / "api" / "counter")      
      n = fetchAndIncrement "default"
      HttpResponse.ok (Body (n |> Nat.toText |> toUtf8))
    getNamedCount = do
      Routes.get (root / "api" / "counter" / counterName)       
      n = fetchAndIncrement counterName
      HttpResponse.ok (Body (n |> Nat.toText |> toUtf8))
    Http.handler (getDefaultCount <|> getNamedCount <|> 'notFound)
```
Here `fetchAndIncrement` is assumed to be part of the key value ability, but it could also be a separate function that only requires the `KeyValue` ability. Again, as with the wasmCloud code: there is no indication of how fetch and increment will be _executed_: there's no connection string, no security boilerplate, and no reference to which vendor (e.g. Redis, Cassandra, home-made) the key value store comes from.

To keep the code easier to read, my Unison example above doesn't emit JSON it just dumps the number, but that's an implementation detail I didn't think was useful for the comparison. I still can't get over how much this type signature makes my brain happy:

```haskell
kvcounter : HttpRequest -> {Log, KeyValue} HttpResponse
```

Just like with wasmCloud, it's impossible to execute this code without supplying an implementation for these abilities at runtime.

In my ideal world, I would be able to take the Unison code above and compile it into a WebAssembly [component](https://github.com/WebAssembly/component-model), deploy it to wasmCloud, and launch it. 

Or I could use the code as-is and run a `ucm` command to deploy it to Unison cloud. Or, in the distant future, there might be some way to federate Unison cloud and a wasmCloud lattice to allow the `Log` and `KeyValue` abilities to be satisfied by wasmCloud capability providers. I don't really know... all I _**do**_ know, is that I just want my peanut butter and chocolate: _wasmCloud and Unison_. 🤔