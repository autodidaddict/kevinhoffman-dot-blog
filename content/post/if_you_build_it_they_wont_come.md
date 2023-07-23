---
title: "If You Build It, They Might Not Come"
date: "2023-07-23"
toc: false
#author: "Kevin Hoffman"
type: post
description: "Just because you build something good doesn't mean people will use it"
subtitle: "You Can Lead a Customer to Water..."
categories:     
    - "development"
---

There's a movie quote that has become part of our lexicon (I think the kids call it a _meme_ these days). It comes from the movie [Field of Dreams](https://www.imdb.com/title/tt0097351/), and says, _"If you build it, they will come_". The quote from the movie refers to a bunch of ghostly baseball players eagerly awaiting a corn field to morph into a baseball diamond.

A sentiment of some debate in the community is whether what we do can be considered **art**. If we are artists, then it should be no surprise that we want our art to be viewed. Admired. Respected.

But what we do isn't purely art for art's sake, is it? If anything, what we do is _functional_ art. We actually have to solve real-world problems with what we make, regardless of whether we think it's art worthy of being mounted and displayed in a gallery.

Sometimes we let the desire to have our work be appreciated get in the way of actually making products that solve real problems. Sit back 🪑, grab a coffee ☕️, and let uncle Kevin share some bad memories 😢.

# Is a File Better than an App?
I once worked in a part of an enterprise made up of a dozen or so teams. Each of them had their own application and yet they all communicated with each other in real-time. Managing the data structures internal to each application as well as the structures used to exchange data between applications was, as you can imagine, a nightmare.

I was on the team making the messaging platform that each team would be using for _intra_-app communication. This platform generated code from schemas, so it seemed like a great idea to centralize that. It was a grand app--it managed data dictionaries, dependencies between dictionaries, ownership, and it even dispensed automatically generated JAR files or maven artifacts the app teams could use.

It made clever (this word should've been a warning) use of a graph database to prevent transitive dependency failures. It was an `Angular` app that sat atop `Play`, `Scala`, `Akka`, and `Neo4j`. A developer could log into the data dictionary manager and say, _"I need to send an upstream message `X` to application `Y`, give me the code I need."_

This application operated on the assumption that the inner development loop's main bottleneck was in communicating with the message bus. There were other things that could take _weeks_ of person-hours that needed to accompany schema changes. In the grand scheme of things, the data dictionary was a drop in the ocean.

The developers loved and it enjoyed the clever (there's that word again) use of a graph database. They liked the site and the functionality it provided. However, none of them used it. Not. One. Even worse, the developers all ultimately decided to stop using our messaging platform, opting instead to communicate directly with the messaging substrate and bypass any middleware abstractions. They just hand rolled the data structure files and the data marshallers.

Some hard lessons learned here:
* If you make an abstraction on top of something, it needs to make that thing easier to use. If it's easier to bypass your API than use it, something has gone awry.
* Just because you're solving _**a**_ problem doesn't mean you're solving the _**right**_ problem. If an app automates away a 2 hour task and leaves 3 weeks of manual process, it's pretty easy just to ignore the app, especially if the app represents a process change.

A wise former employer of mine used to say, "If you can do something easier with a pencil than a computer, use the pencil." While we can argue the finer points of the details of that statement, the idea behind it rings true: if it's easier for people to _not_ use your code, they won't use it. 

_App users are like lightning. They will find the shortest path to their goal, even if that means circumventing your marvelous art installation of an app._

# Cookie Cutters Cause Stagnation
After you stamp out some dough with a cookie cutter, you can then easily make changes to that thing. You can add a smileyface to your gingerbread man, or a frowny face, or demonic horns if you bake the way I do. The point is that even though you start every new cookie with the same shape, _none of them bake up the same way_.

At another company, I put a tremendous amount of effort into coming up with a template for building Go-based microservices that hundreds of teams could use when they kicked off a new microservice project. The starter kit had everything from logging to observation, metrics, HTTP server and router, secrets management, access to company resources, etc: all the delicious toppings.

What we all knew and never really said out loud was that the split second a new project started based on this template, it was _divergent_, and not in the [Hollywood moneymaking](https://en.wikipedia.org/wiki/Divergent_(film)) movie way. The more teams that used the same starter kit, the more outliers we had. We fielded countless calls from developers using this template looking for help with their apps... and those apps looked _nothing_ like the recommended shape of the cookie cutter. 

We now had dozens of snowflakes that only pretended to have something in common with the template. Worse, we had apps failing in production and those teams blaming us, when the problems were buried deep in the post-template code.

The lesson learned here? It's better to give people composable, reusable building blocks (software legos) than it is to let them stamp out an insta-legacy scaffold. The components can be kept up to date over time, while it's nearly impossible to track the sprawling, infinite diversions from a template.

# Miscellaneous Misery
In one instance, I had made an administration application. This app let people monitor and administer a running application in production. It gave them the ability to manage very difficult and complicated data structures with ease. Did anyone use it? Nope. After a few weeks the app was basically unused.

Turns out the target audience was a team of apologists. They'd become so used to manually administering the system with a combination of CLI tools, rubber bands, and duct tape that clicking through the web app slowed them down. It didn't seem to matter that the web app was designed to eliminate the super high rate of administrative user error from those complicated maintenance tasks. People didn't like change, and they didn't want us moving their cheese. It didn't matter to them that the app actually did solve one of their main problems.

In some cases, even if you are solving a real problem, and doing so in a way that saves time, reduces errors, cures cancer, and brings about world peace--some people are _still_ going to reject your app.

I've made real-time business-to-business integration services that nobody used because, and this is an actual quote, _"the nightly FTP batch job works just fine for us"_.

I've automated things people didn't want automated because they were absolutely terrified of the consequences of a human not having their finger on a button.

I've moved things to the cloud that people were perfectly happy to leave sitting in a data center wasting all kinds of power.

I've made a mobile application that ran on phones and tablets and the users made real phone calls (still don't know what those are) to actual humans to get their work done and never touched the app.

I made an app for a small newspaper that automated the editorial pipeline so staff writers could see a piece move from concept to publication. Nobody used the app as everyone preferred to print out paper copies and physically deliver them to editors' desks.

# Summary
Of course, for every one of these sob stories of unused applications, I have `0.25` stories of applications that are lovingly used by adoring fans to this day. It's not all bad, but building applications as art pieces rather than usable solutions is a very common mistake in our industry.

So, the next time you're in a meeting and someone decides to quote the movie and say, _"If you build it, they will come"_, make sure you put your critical thinking hat on and ask, **_"But will they, though?"_**