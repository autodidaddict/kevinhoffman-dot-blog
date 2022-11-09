---
title: "Can we Game in the Fediverse?"
date: "2022-11-08"
toc: false
#author: "Kevin Hoffman"
type: post
description: "Thinking about Twitter, the Fediverse, and Gaming"
subtitle: "Thinking about Twitter, the Fediverse, and Gaming"
categories: 
    - "fediverse"
    - "gaming"
---
_Wondering if we can take advantage of all of the renewed interest in the Fediverse to get our game on._

<!--more-->

Recently Twitter has been all over the news, and not for good reasons. People are being laid off, Elon's taking over, rules are changing, free speech is potentially at risk--dogs and cats living together--mass hysteria!

One side effect of this is that a lot of attention has fallen upon [Mastodon](https://joinmastodon.org/) as an alternative. Whether I think Mastodon is a direct competitor to Twitter is a topic for another time. However, thinking about Mastodon and how it works in collaboration with [the Fediverse](https://fediverse.party/en/fediverse/) (_federated universe_) inspired some new thoughts.

The federated universe is basically a decentralized web of securely connected systems that can interoperate based on a standard set of protocols. If this description sounds familiar, it's basically what the original vision of the web aspired to be (and it's definitely _not_ web3 or related to crypto).

For a fun exercise, you can check out [fediverse.space](https://fediverse.space/) to view some cool visualizations of how all of these systems interconnect.

## Federating Worlds
There are a number of protocols involved in the Fediverse. It's a difficult course to navigate, as each product that claims to run in the fediverse can support a different subset of protocols. However, there are some that are core to how this all works:

* Activity Streams
* Activity Pub (based on ActivityStreams)
* Web Finger

### Activity Streams
Activity Streams 2.0 is a [W3C Standard](https://www.w3.org/TR/activitystreams-core/#introduction). As the abstract says, activity streams is a way of representing _potential_ and _completed_ activities using JSON. What you can define as an activity is practically limitless. 

Activities are semantic descriptions of actions. They're verbs, performed by _actors_ on _objects_. I won't copy and paste the spec here as you can follow the link and read it on your own time. The super short version is that the activity streams protocol is made up of **object**s, **actor**s, **activities**, and collections of those things.

For a detailed list of what you can describe using activity streams, check out the [vocabulary](https://www.w3.org/TR/activitystreams-vocabulary/) reference.

### ActivityPub
The activity pub protocol sits on top of activity streams. Using the activity stream primitives, this protocol defines interactions between clients and servers and between peer servers. This forms the core of how Mastodon and many similar applications work.

These interactions can involve the standard _Create_, _Update_, and _Delete_ activities, but also things like Add, Remove, _Like_, _Follow_, _Block_, etc.

By examining `inbox`es and `outbox`es and collections of activities and objects, robust applications can be created where feeds of information are sent out to interested parties through a combination of federated interest graphs, standard protocols, and secure communication. Queries of resources indicated by well-formed and standards-compliant URIs can give applications access to all the data they need to operate.

This is how one Mastodon server can send a direct message to someone on another server, how they can receive and accept (or reject) follow requests, view the latest news feed, and much more.

### Web Finger
Web finger is another discovery protocol that, as defined by the spec, defines **a method for resolving links to resources**. Web finger queries are sent to a well-known endpoint at `/.well-known/webfinger`. If you're as old as I am, you remember discovering peers and colleagues at universities using the `finger` command on shared machines.

The modernized version of this allows you to make queries of the webfinger endpoint to get a payload of information which can contain additional URLs to be used for pulling even more metadata. For example, if you open a browser to [https://mastodon.world/.well-known/webfinger?resource=acct:autodidaddict@mastodon.world](https://mastodon.world/.well-known/webfinger?resource=acct:autodidaddict@mastodon.world) (this is me), you'll receive the following JSON output:

```json
{
  "subject": "acct:autodidaddict@mastodon.world",
  "aliases": [
    "https://mastodon.world/@autodidaddict",
    "https://mastodon.world/users/autodidaddict"
  ],
  "links": [
    {
      "rel": "http://webfinger.net/rel/profile-page",
      "type": "text/html",
      "href": "https://mastodon.world/@autodidaddict"
    },
    {
      "rel": "self",
      "type": "application/activity+json",
      "href": "https://mastodon.world/users/autodidaddict"
    },
    {
      "rel": "http://ostatus.org/schema/1.0/subscribe",
      "template": "https://mastodon.world/authorize_interaction?uri={uri}"
    }
  ]
}
```

Inside this response, you'll find additional metadata including a list of links that can be followed for more information and possibly a list of aliases that correspond to this account. As you can see in the second link (type `application/activity+json`), there's a link to this user's activity feed. Having this activity feed URL allows us to post activities (like messages) to this user, even if we're not on that user's server.

The third link contains a template for producing a URL to authorize interactions using yet another protocol called [OStatus](https://en.wikipedia.org/wiki/OStatus). Activity Pub, from what I have read, supercedes **OStatus** though services like Mastodon still use it.

## Will it Game?
My first exposure to distributed/multiplayer computing was through **[MUD](https://en.wikipedia.org/wiki/MUD)** s (as well as MUCK, MUSH, MOO, etc). As a _wizard_, I was able to log into the game and I could write code in **LPC**. I could use commands like `load`, `unload`, `clone` to interact with the code I'd written. So I could have my character standing in a room and I could type `clone sword.c` and it would create an instance of the sword and put it in my inventory. This development loop absolutely peaked the dopamine drip rate and I have never, ever experienced anything as enjoyable since then.

{{< figure
  src="/images/ac_nh_multiplayer.jpeg"
  class="class param"
  type="margin"
  label="mn-ac"
  title="Animal Crossing Multiplayer"  
  caption="4 player couch/8 player online"    
  alt="Image showing multiple players in animal crossing"  
 >}}
Fast forward a few years and I find myself playing **Animal Crossing**, and it started with very limited ability to _visit_ other people's villages that ultimately ended up in New Horizons with local and online co-op. The user experience for this was fantastic - you could get on a train and then you'd arrive at your friend's village and you could sell them stuff that's cheap in your village but rare in theirs. I heard there were nefarious things you could do, but I never tried it.
{{< section "end" >}}



I see this pattern occur over and over again with lots of incredibly fun games. Someone can set up a Minecraft server and, using a Minecraft client, I can connect to it and interact with the amazing stuff there. In **No Man's Sky**, I can build extensive bases and settlements and then I can have visitors stop by and hang out and even contribute and make enhancements. In **Stardew Valley** there is support for 4-player co-op within the same farm.

{{< figure
  src="/images/stardew_multiplayer.jpeg"
  class="class param"
  type="margin"
  label="mn-stardew"
  title="Stardew Valley Multiplayer"  
  caption="Example multiplayer environment"    
  alt="Stardew valley with multiple houses and players"  
 >}}
What's happening with these games is what I (however inaccurately) call **sandbox federation**. I can play in my sandbox and have a pile of fun, but if I want you to be able to enjoy it as well, I can give you something that gives you access, or it can be mutual and I can hang out in your sandbox while you mess mine up, and so on.
{{< section "end" >}}

What separates this from something like an MMO is that in an MMO, your client and my client would both connect to some endpoint exposed by a massive, single shared universe that neither of us host. With federation, you and I can both have our own easily self-hosted worlds and share them at will, without requiring a AAA game publisher to host our worlds.

The promise of federation means that I could start a docker image or just run `./mygame` and I'm up and running and playing locally. Then, through federation, my game could interact with the larger universe of other games, all without the need for a central server or authority.

## A Hypothetical Fediverse Game
Let's say we're playing a space game where you and I are building our own little space colonies on our own little moons. In a universe as vast as ours, it would be impossible to accidentally run into each other, so I send a code to your colony computer (how I discover your colony is up to the game, but could even be done entirely offline). Next time you log in, you see the code and you _accept_ the request to exchange and now our colonies know about each other and can share information.

Under the hood, this could be coded as a simple "follow and accept" activity flow. Once we're mutuals, my game can send relevant external activities to your inbox and yours to mine. Just off the top of my head, a number of interactions seem possible using **Activity Pub** actions:

* Trade/sell items with prices valued by local availability
* Post microblog messages to all of your "followers" (colonies w/trade agreements)
  * Everyone gets a feed from colonies in the group
* Notification of important external events
  * colony defended against attack / colony attacked outward
  * new technology levels reached / R&D projects succeed
* Change in market conditions
* Structures created / destroyed
* Alliances could be formed so that when one colony is attacked, other colony's resources can automatically come to aid
* ... others limited only by imagination


With the ability to read collections that contain items that can also be collections, games have the ability to obtain all the information needed to allow one player to visit another (provided there's an agreement in place).

I think an MVP in this space would be a point where anyone can start the game locally, make it securely accessible to anyone else running the game, and establish a mutual relationship (thematically could be a _trade agreement_ or something which can be upgraded to an alliance later). Then, I should be able to see a "traditional" style microblog feed (similar to that bird site) from all the other colonies I have a relationship with.

Once that foundation is in place, we could add gameplay elements incrementally and build on that core foundation. Now I _really_ want to build a game like this. 

### Shameless Plug
What might make this game even cooler is that instead of (or in addition to) players manually moving stuff around locally with clicks and keys, they could build the logic for in-game elements as [wasmCloud](https://github.com/wasmcloud) actor components. These autonomous agents could go about their business in the game in real-time while the player goes to work and takes care of important business like consuming coffee.

This unlocks all sorts of new gameplay design ideas, including allowing the "brain code" for elements to flow and be shared among members of the same group.

## Wrap-Up
In short, I think Activity Pub / Activity Streams could very definitely be used for a new, fun kind of federated sandbox gaming. I think all the renewed interest in the Fediverse and Mastodon may just get people thinking about new ways to use federation protocols, like gaming. Now all I need to do is find the time to make a game.