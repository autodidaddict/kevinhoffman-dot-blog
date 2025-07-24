---
title: "Prompt Engineering - Round Two"
date: "2025-07-24"
#author: "Kevin Hoffman"
type: post
description: "I use prompts in anger. Lots of anger."
subtitle: "I use prompts in anger. Lots of anger."
categories: 
    - "ai"
    - "prompts"
    - "muds"
---
_Wherein I try to add more agentic functionality to my MUD_

<!--more-->

I had everything planned out. I was going to create a second video as a follow-up to the first one I made. I even had a title: **_How I learned to stop worrying and love the prompt_**. This was a bad idea.

In the first video, I talked about using an LLM (in my case, `llama3.5`) to convert natural language descriptions into strict data structures describing game object behavior.

<iframe width="560" height="315" src="https://www.youtube.com/embed/ncahHw1F8SI?si=8ue7s7lCmGPWficR" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

In the video, I show how I was able to create an agent that can convert instructions like the one below into JSON data:

```
When someone wears the cloak, send them the message 
"You feel heavier, as though burdened by something." 
and send everyone else in the room the message,
"{player} vanishes."
```

I eventually get this all working, but it required a _really_ long prompt. The second phase of using LLMs to assist with the game is to use a model to map intent from natural language sentences to commands that are split into a verb, a direct object, and an indirect object.

In other words, I want to change

```
Give the red ring to the large elf
```

into some JSON identifying the parts of speech and the object IDs of the specific items in question. The ultimate goal is to identify the _exact_ ring and the _exact_ elf being referenced. This should be done by comparing these references to a list of nearby objects eligible for commands.

So I want to see `mordor.ring#3f7e9` for the ring to be given, and I want `player.dave` (or whatever) for the indirect object; the object that is the receiving the direct object.

After several hours of trying to get something working, I finally had a prompt that worked for the "happy path" - when everything lines up perfectly. But look at the size of this
thing:

```
You are a game assistant responsible for interpreting a natural language 
description of a command.

When a player enters a command, the interpreter will attempt 
to identify the verb and the direct
and indirect objects of the command based on available 
context and the command itself.
                    
If a command cannot be interpreted from the user's input, 
you should return an error message
with the response. You will not make up answers and you will not return commands 
that are not included in the valid command list.
                    
If a command can accept a direct or indirect object as a parameter, 
you will use the supplied list of eligible nearby objects as candidates. 
This list will be supplied as a JSON array of objects with the following
                    
schema:
[
   {
      "objectId": "string",
      "name": "string",
      "adjectives": ["string", "string"],
   }
]
                    
The adjectives for objects can be used to disambiguate between
objects or as aliases in place of the name.
                    
You will return the list of potential candidates according 
to the following schema:

{
    "objects":
    [
        {
          "verb" : "string",
          "directObjectId": "string" | null,
          "indirectObjectId": "string" | null
        }
    ]
}
                    
If you cannot find any suitable objects to pair with the command verb, \return an empty objects array.
Do NOT return any additional information.
Do not return any messages or text, only valid
JSON according to the schema described above.
                    
If the command contains a valid verb and direct object, 
but requires an indirect object and that indirect object is not 
in the list of eligible objects, return an empty objects array.
                    
Possible commands are to be returned in the same order as they 
appear in the object list, so if there are multiple matches for 
a player's input, the first matching object will be the first command in the response.
                    
The following are a few examples of user input and the 
expected response, given this set of sample eligible objects:

... etc ...

... etc ... 

Eligible objects:
[
  ... etc ...
]   
```

This version was probably iteration 20+ after trying to get `llama3.5` to not choke on edge cases. This version would sometimes return two commands, both of which were incorrect. Sometimes it would fabricate nouns or adjectives that weren't in the user's input. Other times, despite my best attempts, it would match on things that shouldn't have matched. It even returned command verbs that weren't in the authorized list (I had `wield` and `give` as the only valid commands).

After this was still failing, I actually opened up 3 new browser tabs. I told each of these other AIs (including GPT) my problem, the failing prompt, and the bad output. Each of them had some suggestions for how better to ground the model and fix the problem. 

One promising lead was to change the prompt from one that returned results to one that used tool calling. The notion there was that some models are better able to map intent to tool calls than they are to the seemingly strict demands of a JSON schema.

I switched to tool calling, which gave me a much smaller system prompt, but in the end I traded trash results for trash function calls. The model just kept calling my functions with worse and worse data.

After hours of banging my head against the desk I (partially) gave up. I wondered if I would have better results if I asked the model to do fewer things. Instead of trying to identify parts of speech, match to items in the eligible object list, and correlate object IDs and ajectives, I just tried parts of speech.

The assumption was that the model could do this easily, and then I could do post-processing to match the nouns with the nearby objects.

The results? **NOPE**.

Supplying `Give red ring to the large elf` actually returned `the` as an indirect object (when it's actually a definite particle). Ready to stab my eyes with rusty spoons, I refactored the "simple" prompt more and more and eventually had to force a `temperature` of `0.0`.

After all of this, in the wee hours of the morning, I was finally able to convert the user's input into something like this:

```json
{
    "verb": "give",
    "directObject": {
	"noun": "ring",
	"adjectives": ["red"]
    },
    "indirectObject": {
	"noun": "elf",
	"adjectives": ["large"]
    }
}
```
Was any of this actually worth it? Would it be worth it when actually playing the game?

## Takeaways
Here's a list of key takeaways from this experiment. Some of these I knew intuitively already, but this exercise kicked me in the face with the items on this list.

* Prompts are tightly coupled to models. My `llama3.5` prompt returned worse, better, or even incoherent results when I tried different models and providers (e.g. `OpenAI`)
* More words isn't always better. 
* Each model has a favorite way of receiving constraints
* Providing `n-shot` examples in the user message as opposed to the system prompt can dramatically increase success. In my case, I provided 3 sample input sentences and their corresponding output JSON.
* If you're using an LLM to give you specific, actionable, code-usable results... _you need to plan for frequent failure_. Even after you've tweaked everything so it looks like you always get good results, at some point, you will get trash results. This is inevitable.
* LLMs _are not algorithms_. They don't know what algorithms are. Asking an LLM to perform algorithmic processing is a recipe for bad results. LLMs are autocomplete prediction engines.
* Building agents (and their prompts) that need strict results feels like trying to mold a sand castle out of dry, loose, desert sand. _Internal cohesion feels impossible_.
* I don't ever want my day job to consist solely of shaping and crafting prompts.

Once I've calmed down a bit more, I might decide to make the second video in the series where I show the final prompt/agent in action.

The goal of this experiment wasn't to assess the practicality of this option, but more to force me to do a number of agentic activities _in anger_. 

If my goal was to just find a practical way to parse these types of commands, I could probably have solved the whole thing in 20 minutes by creating a regex for each of the possible commands and used captures to grab the important words.
