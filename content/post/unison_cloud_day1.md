---
title: "Unison Cloud - Day 1"
date: "2023-12-14"
toc: false
#author: "Kevin Hoffman"
type: post
description: "Wherein I finally get access to Unison cloud"
subtitle: "Unison cloud has chosen me!"
categories:     
    - "unison"
    - "distsys"
---

_My first exposure to building with Unison and deploying to Unison cloud._

<!--more-->

It feels like years ago that I originally submitted my email address indicating my interest in gaining access to Unison cloud. Yesterday, I was greeted with the best Hanukkah present I've had in a long time -- _access to Unison cloud_.

If you've read any of my blog posts on the subject, you know that Unison isn't like other programming languages. It doesn't build or compile an executable. The code simply _is_, and your code invokes other code through hashes. You manipulate your codebase through the `ucm` command and you can navigate your namespace hierarchy as though it were directories, complete with commands like `ls` and `cd`.

```
responsible-badger/main> ls

  1. README       (Doc)
  2. ReleaseNotes (Doc)
  3. examples/    (33 terms, 1 type)
  4. lib/         (280069 terms, 7244 types)
  5. metadata/    (3 terms)
  6. patch        (patch)
```

Unison cloud's deployment mechanism also doesn't work like some of the more familiar cloud execution platforms like fly, Azure, AWS, Cloudflare, etc. Instead of using a command line tool to "push" a compiled asset to the cloud, we actually run Unison code that deploys a _Unison function_.

When I followed the getting started directions, like any good "hello world", I got a sample function that handles an HTTP request. As you can see in the following code, the `logic` function takes an `HttpRequest`, requires the `Exception` and `Log` abilities, and returns an `HttpResponse`:

```haskell
  examples.helloWorld.logic : HttpRequest ->{Exception, cloud_6_0_5.Log} HttpResponse
  examples.helloWorld.logic = Route.run do
    use Text ++
    name = route GET Parser.text
    info "request for greeting" [("name", name)]
    ok.text ("👋 hello " ++ name ++ "\n")
```

The `route GET Parser.text` will basically turn everything after the base URL into a "name" for the purposes of this code. The `info` function is part of the `Log` ability, and lets me log some text as well as a set of key-value pairs.

To deploy this to Unison cloud, I execute the following function (which came with the new project template):

```haskell
examples.helloWorld.deploy : '{IO, Exception} ()
  examples.helloWorld.deploy = Cloud.main do
    name = ServiceName.create "hello-world"
    serviceHash = deployHttp !Environment.default helloWorld.logic
    ignore (ServiceName.assign name serviceHash)
    printLine "Logs available at:\n  https://app.unison.cloud"
```

At the `ucm` prompt, typing the following executes the deployment script:

```
ucm> run examples.helloWorld.deploy
```

Unison cloud even keeps a log history for me (that call to `info` in the `logic` function):

![unison cloud screenshot](/images/ucloud_log.png)

My favorite thing about the way Unison cloud works is that I was able to go from nothing to a running service in the cloud in about 10 seconds, which you can play with at [https://autodidaddict.unison-services.cloud/s/hello-world](https://autodidaddict.unison-services.cloud/s/hello-world/bloguser).

## Wrap-up
As usual, I find myself loving the Unison language, loving the concepts and architectural decisions that make it unique in an endless sea of programming languages. However, I'm also still sitting on the sidelines waiting to find a way to actually put Unison to use in a real project supporting real functionality.

For now I'll have to settle with Unison being an inspiration rather than a practical tool.