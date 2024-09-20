---
title: "Complex Systems Arise from Simple Rules"
date: "2024-09-20"
toc: true
#author: "Kevin Hoffman"
type: post
description: "Complex Systems Arise from Simple Rules"
subtitle: "Complicated Systems Arise from Complex Rules"
categories:     
    - "architecture"
    - "design"
    - "control-theory"
    - "feedback-control"
---

_**Complex** systems arise from simple rules, **complicated** systems arise from complex rules._

<!--more-->

## Introduction
Lately I have found myself repeating this mantra in my head more and more. Over the years I've built and developed pretty much every kind of software you can imagine, but it's only been recently that I've been adamant about never violating this rule. The trick is to try and explain it without violating it (and so not be unnecessarily complicated).

Firstly, I want to define _complex_ and _complicated_, because I think my definitions have a subtle connotation that you might not pick up from the dictionary. _Complex_ means _consisting of many different and connected parts_, while _complicated_ can technically mean the same thing, I like to define it as _something that has been made more difficult through the addition of complexity_. Complex just describes the intricate workings of a machine, and isn't inherently bad, while _complicated_ is bad: _difficult_. 

## Complex Distributed Systems
Remember that there's nothing inherently bad about complex systems. Distributed systems are full of interconnected parts, it's what makes them distributed in the first place. You could build a distributed system where you scale the same process out to `n` instances, and each one is [queue subscribed](https://docs.nats.io/using-nats/developer/receiving/queues) to the same work request subject. Such a system is complex, but is composed of a single simple thing (the queue subscribed service).

Now let's suppose things are more messy. We've got a dozen discrete (micro)services. They're all part of the same distributed system, but they adhere to 12 factor design, follow [Sam Newman](https://samnewman.io/books/building_microservices_2nd_edition/)'s advice, and strictly follow the single responsibility principal (SRP). This system is complex, but still not complicated.

Take NATS, for example. It starts with a core messaging layer made of up incredibly simple rules (you can even use `telnet` to interact with the server!). JetStream is a persistent streaming layer built on top of core NATS. You communicate with streams and their metadata via simple usage of core NATS components. On top of JetStream are more simple building blocks like the key-value store and the object store.

Think of a system managing all of the controllable components in a car. You simply _can't_ build a complex component that exerts control over the whole of the system at once. The only realistic way to build such systems is with with tons of small, simple components that all surface information to higher layers, which in turn surface to higher, etc. Decisions are made based on [feedback control](https://batch.libretexts.org/print/url=https://eng.libretexts.org/Bookshelves/Industrial_and_Systems_Engineering/Chemical_Process_Dynamics_and_Controls_(Woolf)/11%3A_Control_Architectures/11.01%3A_Feedback_control-_What_is_it_When_useful_When_not_Common_usage..pdf)-type process, which are also _small_ decisions. Think of a thermostat in your house. Rather than trying to exert total control through a complex process, it makes a number of small decisions based on feedback, which eventually results in smooth temperature control.{{< sidenote "sn-effects" >}}I still remember when my grandfather told me about how derivatives and small feedback data could be used to smooth transitions, like game systems steering through curves in the road. This conversation was life-altering.{{< /sidenote >}}

Finally, (if you've seen the movie), think back to the microbots from [Big Hero 6](https://bighero6.fandom.com/wiki/Microbots). Rather than having built a single robot that has one big complicated purpose, the protagonist built millions of little ones that can collaborate to do amazing things all by following simple rules.

Complex systems arise from composing **_simple_** rules and building blocks, and are easier to operate, maintain, and enhance than their evil twins: _complicated_ systems.

## Complicated Distributed Systems
The hallmarks of complicated systems are complex rules or foundational building blocks. Rather than progressively layer simple rules on top of each other, complicated systems try to manage "everything" with one set of logic to rule it all. Complicated systems feel the need to _tackle complexity_, and ironically they do so by increasing how complicated they are.

You can see this in systems like Kubernetes. If you look at its origins, and peek deep down into some of the original code, it looks like the goal was to build a complex system from simple rules. But as the number of things Kubernetes could do grew, rather than creating more primitive Lego blocks, Kubernetes ended up being more of a shelf for pre-made Lego sets like the Millenium Falcon and the Death Star. Kubernetes is so _complicated_, in fact, that there is a cottage industry of software vendors making their money entirely on the premise that they will reduce or tackle Kubernetes' complexity. This is where things get ugly, because we're now in a realm where we're laying complex components (the "manager of managers" software) on top of already complex components, creating an incredibly complicated system.

Kubernetes, SalesForce, Oracle, SAP and their massive ecosystems are all prime examples of complication over complexity. Complex systems allow you to zoom in on a specific piece, work on that piece, and then zoom back out again. Complicated systems, on the other hand, require you to view, constrain, manage, and understand the whole of the universe all at once.

## Build on a Foundation of Simple Rules
In keeping with my own advice, I'll keep this blog post short and simple. The one thing I want you to take away from this post is that we need not _add layers_ of complexity on top of pre-existing complexity, as that will just simply compound the problem. Instead, we should put our collective feet down and demand that we build from simple building blocks or we strip existing code down so we can refactor it to use simple building blocks. Complex building blocks create complicated (aka "bad") systems.

A few tips and guidelines I've picked up as I learn to demand simpler rules:

* Application of simple rules in loosely coupled layers can allow powerful, complex systems, including those we might not have expected from our customers. (e.g. "emergent" behavior)
* Use small, _simple_, composable building blocks
* Embrace the inconsistency of systems rather than waste effort trying to control them (As Bruce Lee said, the bamboo survives by _bending with the wind_)
* Favor **choreography** over **orchestration**.
* Be inspired by the [12 Factors](https://12factor.net/)

