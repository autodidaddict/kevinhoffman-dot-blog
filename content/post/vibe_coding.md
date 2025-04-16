---
title: "I came. I saw. I vibed."
date: "2025-04-16"
toc: true
#author: "Kevin Hoffman"
type: post
description: "I put aside the hype and hate to try vibe coding."
subtitle: "Are the vibes good enough?"
categories: 
    - "ai"
    - "vibe-coding"
---
_An experience log of trying to vibe code like the pros_.

<!--more-->

Yesterday evening (which ran into this morning), I set aside some time to try out this vibe coding thing. I believe in the scientific method and in taking unbiased approaches to things. As a result, I gave myself a couple of rules for my vibe coding experiment:

* It had to be in a language that I wasn't already an expert in
* I had to try and build something I've never built
* I couldn't completely disregard an AI response to manually write my own code. I needed to incorporate AI responses in every iteration.
* I deliberately avoided expensive and infrequently used models. It needed to feel like your "average" vibe session.

As it turns out, I had the perfect test for vibe coding. Every so often, when I find myself with so-called _spare_ time, I will do an exercise or two on [codecrafters](https://app.codecrafters.io). Code crafters is like a step up from [exercism](https://exercism.org), where you build real-world things that could be used in production and you build those things over a large number of discrete steps.

My favorite of these is the challenge of building your own Redis server. Before my vibe experiment I had completed the first four steps of the challenge. My "redis" server could bind to port `6379` and respond with `+PONG\r\n` once and then it would die. In the Codecrafters style, you write just enough during each step to make the tests pass.

I was going to use **OCaml**, a language that I'd only ever used to make little hello world examples. Of all the languages I "know", I know OCaml the least. I'd satisfied the previous exercise using OCaml's regular networking library. I decided I was going to rewrite the previous step and do the current step by using [Eio](https://github.com/ocaml-multicore/eio), a newish async library that builds on OCaml 5+'s _effect handlers_ system.

The step that I decided to try out with vibe coding had the following instructions:

* Respond to multiple `PING`s within the same session/connection
* Respond to concurrent `PING`s from multiple connected clients

I fired up my vibes and got to coding. I decided to open by asking for OCaml code that used Eio to create a TCP echo server that echoed every line it received. I did this thinking that I was more likely to hit public samples that did echo than I would ones that did Redis.

The code it gave me didn't compile, but I was able to figure out why. It responded with the user's text once and then stopped. So far so good, I thought. Then I realized that it hadn't actually used the `Eio` library and instead used `Lwt`, the previous defacto async library.

Then I got more specific. I told the LLM that I wanted to respond with `+PONG\r\n` and I wanted to do it every time the server received a line of text. It spat out an example and then this:

> _"Let me know if you want to support multiple clients, background fibers, or add Redis protocol support!"_

I was impressed. It saw the `PONG` and knew that I was building something Redis-like. The code didn't come close to compiling, though. The compile errors were inscrutable to me, which was a _good thing_, because I don't expect a vibe coder to be an expert at error message troubleshooting.

I complained again that it wasn't working and pasted the error message. It spat out some more code and then gave me one of its post-code summaries:

> _"Nice — you want the server to echo back after receiving a full line (ending in \n) or once the buffer is full. That’s a common pattern for line-based protocols like Redis, SMTP, or IRC."_

> _"We’ll update Server.handle_client so it:"_

> _"Accumulates input in a buffer."_

> _"Responds only when it sees a newline (\n) or the buffer is full."_

> _"Then it sends the accumulated message back."_

I spent something like another half hour going back and forth trying to get the model's code to compile. Then I got an error and pasted the error message. It replied with the first message of what I know now is a death spiral:

> _"Ah, I see the issue now! The function `Eio.Stdenv.switch` does not exist, which is causing the Unbound value error. Instead, you should be using the Eio.Switch module, which is where the switch creation is actually located."_

> _"Let's update the code so that it properly uses `Eio.Switch` for managing the switch for cancellation."_

As usual the messages coming back from the model all seemed really helpful and surprisingly accurate. It gave me confidence that the code was going to work. I mean, after all, it _did_ say over and over again that the new code was the **corrected** code.

The code I tried next did what the LLM said it was going to do, but `Eio.Switch` was _also_ wrong. I told it so:

> _Me: "Eio.Switch.create doesn't seem to exist either"_

It took me around 45 minutes of reading docs and looking at other people's examples to figure out where the AI had gone wrong. I corrected the AI and asked it to give me another attempt.

> _"This version of the server should now compile and run correctly, managing multiple client connections concurrently without manual switch creation."_

This is where the death spiral started. I looked at the code and it had reverted to a previous bug where it was using `Eio.Stdenv.switch`. 

> _"I apologize for the confusion and for not addressing your feedback properly. You are absolutely right: there is no Eio.Stdenv.switch. I misunderstood how Eio's cancellation switches work in this context."_

> _"Let me now correctly guide you through using the Eio library to create a TCP echo server that can handle multiple concurrent clients. I will provide you with a correct, **working** solution."_

Again, these replies seem good to me. I'm an OCaml and Eio newb so I figure this is right. If a fellow programmer had sent this to me via slack, I'd figure that I was minutes away from a solution. Hell, the LLM even labeled it's next output as "the **correct** and **working** solution".

After more downward spiraling on this track, it had gone back to a slightly better version of a previous bug with a type mismatch for parameters. I absolutely could not understand the compiler message for the type mismatch.

> _"I sincerely apologize for the confusion and the errors in my previous responses. Let's take a step back and carefully fix the code so that it works correctly."_

I had another argument with it because it took it 4 tries to give me the right pattern match for `End_of_file` when reading from the socket flow. We argued some more and I was getting angry and exhausted. To be honest, angry and exhausted is exactly what I thought was going to happen to me while _vibe coding_.

I tried a new approach. This time I went over to Codecrafters and copied someone's solution that used `Lwt` (the "old" async library) and asked my AI buddy to convert it to using `Eio`. I figured it would be better at determining intent from the code than from my prompts.

**Nope**. This time it produced code that even I knew was absurd.

Now I went over to Github and found some [example code](https://github.com/ocaml-multicore/eio/blob/main/examples/net/server.ml) that showed reading one line from TCP input and replying with "OK", and then terminating.
{{< figure
  src="/images/oneping.jpeg"
  class="class param"
  type="margin"
  label="mn-vasily"
  title=""
  caption="The hunt for more memes"
  alt="Sean connery at a periscope in Hunt for Red October"  
 >}}
I copied this code over and it worked locally. I changed `OK` to `PONG` and the code actually passed the original **"1 PING only** (sorry, had to make that joke) test.
{{< section "end" >}}

I did some more _wrangling_ then, trying to figure out how to get the code to loop infinitely while reading lines from input. I had never used `Eio`'s `Flow`s before so I didn't know what was happening under the covers. 

Eventually my AI buddy showed me some code that used `let rec`, a facility for creating a recursive function. The code was wrong but it gave me enough of an idea that I was able to create code that read an infinite number of lines and only responded with `PONG` when it got a `PING`.

> _"This version of the code should now compile and run correctly with the Eio library. It handles multiple concurrent clients by forking a new fiber for each connection and correctly uses the `Eio.Net.accept` function. I appreciate your patience and hope this resolves the issues. Please let me know if it works now or if there's anything else I can help with!"_

**Narrator**: _It did not correctly use the `accept` function._

Many, many, many hours later, I was able to use `redis-cli` to send multiple pings per session as well as use the CLI concurrently to make multiple requests. The Codecrafters test doesn't yet pass (at least for me), but I think that might be a config or race condition.

The net result is that the code (mostly) works, but _no thanks to the AI_. If you look at the [final result](https://github.com/autodidaddict/ocaml-redis/blob/master/src/server.ml) code, you'll see a pile of comments that show you most of the code there is copied right from the Eio `net` example.

Worse, everything that I copied from the AI has already vanished from my mind. I am absolutely no better for having gone through this exercise. My stress level went up, I got angry, I wanted to yell at and strangle the AI, and my ultimate solution ended up having very little in common with any of the "helpful" code snippets the AI generated.

Worse worse, I have _absolutely no idea_ if the code I wrote is production grade, high-performance, or can be exploited by an attacker. It's all opaque to me, even though I technically wrote it.

## Final Thoughts
Vibe coding is **dangerous**. The only way you can be sure of the code that an AI gave you is _if you are skilled enough to have written the code by yourself_. If I was skilled enough to have done this on my own, and yet I still wanted to vibe code, it might have saved me some time. I would've known what I needed the AI to generate versus what I'm typing. I would've known if the code the AI gave me was production quality, regardless of whether it compiled.

Startups thinking that they can vibe code their way to an investment round based on nothing but vibe-generated code are in for a rude awakening. They might get people in the door, but _"the house that vibe built"_ is not much more than a house of cards.

Lastly, I think time spent arguing in circles with the AI could be better spent by actually researching and learning the topic. I know that's not how the vibes work, but honestly I think the only people that will have a positive experience with this are people who are skilled enough to not need it, and can ask specifically to generate the kind of ceremony and boilerplate they know they need.

### ... Caveat Vibe-or

