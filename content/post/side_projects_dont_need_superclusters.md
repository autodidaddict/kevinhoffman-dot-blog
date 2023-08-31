---
title: "My Side Project Doesn't Need a Global Supercluster"
date: "2023-08-31"
math: true
#author: "Kevin Hoffman"
type: post
description: "My Side Project Doesn't Need a Global Supercluster"
subtitle: "Eating, sleeping, and breathing distributed systems is not always a good thing."
categories: 
    - "distsys"
---
_There's a trend in the industry where we feel that every project, no matter the size, needs the most infinitely scalable architecture possible_.

<!--more-->

The things we build and the way we build them color our view of the world. When I only build distributed systems that can handle 100,000 requests per second, and that's all I've built for the past 10 years, I start to think that everything needs to be built that way. I start to think that every problem is a distributed systems problem. I start to think that every problem needs to be solved with a distributed systems solution.

This problem has also been called the "Netflix problem". Netflix will have a brilliant engineer show up at a conference and walk through their event sourced architecture that can handle a million requests concurrently. They'll outline the thirty different internal microservices and 10 applications (5 of which they've open sourced) that make up the core framework, on top of which they've built 3,500 services. A select group of people in the audience will then go back to their project building a blogging engine and try and apply that architecture to it.

Something has happened over time that has made the act of putting everything you need in a single process anathema to building "real world" software. Need to put a service in front of a write-only bank ledger? Surely a single process won't suffice. What you'll need is to run 3 hot instances of it on every node in your 500 node cluster, and maintain a cool backup of that 500 node cluster in another country, while also making sure that you have a global anycast ingress into this application so that people in Asia have the same latency as people in the US.

Oh, and you're also going to need to put a service discovery system in there so consumers of the service can find it. But if you've got hundreds of consumers of this service, then you're going to need client-side load balancing, as well as server-side load balancing and the ability to do A/B testing on demand. Don't forget real-time tracing, observability, log distribution and shipping, a data lake, and the mandatory ML model training.

While this may sound like a snarky rant, there's some unfortunate truth here. As an industry, I think we have a bad habit of over-engineering things. When we see something that's complex, rather than scraping that complexity away, we build a new, more complicated layer on top. While there are _definitely_ valid use cases for all of this technology, I think we've lost sight of the fact that not every project needs to be built this way.

I'm as guilty of this as anyone. As some of you may know, whenever I encounter a new language or technology, I try and learn it by building a MUD (multiplayer text adventure). Recently I was looking into a new technology and how to build this MUD.

Obviously, loading all of the game objects into a single process was an absurd notion. So, I needed to build a way to instantiate, address, and communicate with game objects across an indeterminately large cluster. Once I had that, I needed a way to have realtime communication between players and game objects to the tune of 10s of thousands of messages per second. I was going to need a message broker and a distributed, eventually consistent database and deal with distributed locks to avoid concurrent mutation ... all the "good" stuff from building massive distributed systems.

Here's the important history lesson. **30 years ago** (holy crap I'm old), these text-based adventure games ran on (typically university) outdated servers and they consumed _dozens_ of _**megs**_ of RAM and supported thousands of concurrent online players... all in a single process. A Raspberry Pi Zero is easily _**40x**_ (assuming a 1Ghz PiZero) more powerful than the machines that ran these games. One server I used to play and code on would do a periodic reboot to get memory back after continuous play for a week or so. This was, I think, at the **16MB** point.

The Apollo Guidance Computer got humanity to the moon, and yet today's commodity virtual machines spun up in the cloud are a thousand times faster and more powerful and, quite frankly, I think many of us are squandering those resources with bloated, over-engineered solutions. Don't even get me started on burning CPU cycles and heat to produce artificial scarcity for the tulip economy that is crypto currency.

So what would it look like if I built this MUD as a single process? Well, I'd probably start with a socket accepter (or websocket if I want to forego telnet). I'd probably spin up a coroutine/green thread for each player that runs in a loop of accepting input and executing the appropriate code against a list of nearby game objects. Players could communicate with each other as easily and simply as many of the 10-line "chat room" samples that you see in nearly every language these days. Hell, I could even use JSON or BSON on disk to store the object and game state and still be able to handle many thousands of concurrent users on the smallest free tier of any cloud provider.

Now that I've decided not to build a distributed system, my complexity-loving brain seeks out the most difficult way to build the innards of this monolith. Why? Because we've all been brainwashed to think that _simple isn't good enough_. For whatever reason, if the code is clean, easy to read, simple to maintain, simple to use, and doesn't require the use of $500/month worth of cloud resources, then it's worthless.

It's easy for those of us who have been building applications that actually were justified in their need of really large scale (like deploying biometrics clients across the country that needed to handle sustained and peak loads) to start with the most complex thing possible and then try and strip away complexity when the nightmare realization of "day 2" problems hits us.

But what if we just built the simplest possible thing to start with? What if we abandoned our prejudices against the "old way" of doing things, and just got something working in a simple, clean, reliable way? Wouldn't it be amazing if we could start with something simple and only add the complexity of distributed systems as our needs actually grew to require it?

If anybody needs me, I'll be off building a single-process game server that can handle a thousand concurrent users on my PiZero.
