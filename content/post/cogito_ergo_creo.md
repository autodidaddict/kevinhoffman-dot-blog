---
title: "Cogito Ergo Creo"
date: "2023-10-11"
toc: false
#author: "Kevin Hoffman"
type: post
description: "Exploring the Ergo Framework in Go"
subtitle: "Exploring Go's Ergo Framework"
categories:     
    - "erlang"
    - "otp"
    - "golang"
    - "ergo" 
---

_First impressions are regularly wrong. My exposure to the [Ergo](https://cloud.ergo.services/) framework for Go is no exception_.

<!--more-->

I am not ashamed to admit that I spent more time trying to come up with a catchy title for this blog post than writing the actual content. It isn't every day that you get to make a latin pun, so I took my chance. For the curious, the title is a different spin on "I think therefore I am" and means _"I think therefore I create"_.

A while ago I stumbled upon the [Ergo](https://cloud.ergo.services/) framework. My incorrect first impression was that people were just building something Erlang/OTP inspired in Go. I didn't think it was much more than a thin veneer over go routines and channels. Obviously, I was wrong.

The goal of the Ergo project is to leverage the _decades_ of experience and community revolving around building resilient, performant, distributed systems with Erlang/OTP and combine that with the performance of Go.

Once I started digging a little deeper, I realized there was more to this project than I thought. They didn't just build a Go library that feels like Erlang, they built one that is _network compatible_ with OTP (via EPMD, DIST, etc). This means that you can write an application in Go using Ergo and have it join a cluster with your Elixir application and it will "just work".

Having built many Elixir systems running in production, both Phoenix web applications and highly scalable backends, the idea that I could have cluster access to Go opens up a bunch of opportunities. While I can use a NIF to spawn out to Rust or even Zig, there's no easy way to do the same with Go. If we want to play in the cloud space, having access to the ecosystem of Go libraries is a huge bonus. Being able to build those Go applications as a declarative, well-defined supervision tree is the icing on the cake.

So, what does it look like to build with Ergo?

```go
node, err := ergo.StartNode("demo@127.0.0.1", "cookienomnom", node.Options{})
if err != nil {
    panic(err)
}
```

Starting a named node with a cookie is second nature to Elixir folks, but there's something about seeing it in Go that feels good... like I now have my cake and can eat it too--Elixir and Go, like peanut butter and chocolate?

Now let's spawn a `GenServer`:

```go
_, err = node.Spawn("example", gen.ProcessOptions{}, &demo{})
if err != nil {
    panic(err)
}
```

Here `demo` is a Go struct that implements the `GenServer` interface (_behavior_ for you alchemists). Here's what the implementation looks like (this is right out of one of Ergo's GitHub examples):

```go
type demo struct {
	gen.Server
}

func (d *demo) HandleCast(process *gen.ServerProcess, message etf.Term) 
    gen.ServerStatus {
        fmt.Printf("[%s] HandleCast: %#v\n", process.Name(), message)
        switch message {
        case etf.Atom("stop"):
            return gen.ServerStatusStopWithReason("stop they said")
        }
        return gen.ServerStatusOK
}

func (d *demo) HandleCall(process *gen.ServerProcess, from gen.ServerFrom, 
    message etf.Term) (etf.Term, gen.ServerStatus) {
	fmt.Printf("[%s] HandleCall: %#v, From: %s\n", process.Name(), 
                message, from.Pid)

	switch message.(type) {
	case etf.Atom:
		return "hello", gen.ServerStatusOK

	default:
		return message, gen.ServerStatusOK
	}
}
```

With the server spawned onto a node and the node configured with a cookie, we can use `node.Wait()` to leave the process up while all the OTP machinery does what it does best.

Now for the amazing part. Fire up `iex` as a named cluster node:

```shell
iex --name ergo-demo@127.0.0.1 --cookie 123
```

To show that there's nothing up my sleeve:

```shell
iex(ergo-demo@127.0.0.1)1> Node.list
[]
```

And this is where my head exploded:

```shell
iex(ergo-demo@127.0.0.1)2> Node.connect(:"demo@127.0.0.1")
true
iex(ergo-demo@127.0.0.1)3> Node.list
[:"demo@127.0.0.1"]
```

If I had spare heads, they would also have exploded when I did the following:

```shell
iex(ergo-demo@127.0.0.1)4> GenServer.call({:example, :"demo@127.0.0.1"}, :hi)
~c"hello"
iex(ergo-demo@127.0.0.1)5> GenServer.call({:example, :"demo@127.0.0.1"}, {:echo, 1, 2, 3})
{:echo, 1, 2, 3}
iex(ergo-demo@127.0.0.1)6> GenServer.cast({:example, :"demo@127.0.0.1"}, {:cast, 1, 2, 3})
:ok
```

When it comes to distributed systems, I am easily amused. I get the giggles any time one device talks to another. So seeing this Go application--which is an actor hierarchy with servers that behave like `GenXxx` servers--talk to an Elixir (and by extension, Erlang) node over native `epmd` with no shenanigans is fantastic.

Even in the simplest of cases, I can now write Go applications and services that are better organized and more resilient than if I was manually managing the channel spaghetti. Then if I want to extend the capabilities of my Elixir cluster by joining Go nodes, that's also super easy.

The folks behind Ergo also have a cloud beta that you can sign up for, which extends a "cloud overlay network" to your Ergo nodes, automating the cluster joining and discovery activities.

I'm looking forward to being able to use this soon. Now all I need is to run into a problem that can be solved by an Elixir-Go hybrid cluster!