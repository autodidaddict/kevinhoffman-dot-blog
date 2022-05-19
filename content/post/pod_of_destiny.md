---
title: "The Pod of Destiny"
date: "2022-05-18"
toc: false
#author: "Kevin Hoffman"
type: post
description: "How Dave and Buster's Fueled my Distributed Systems Addiction"
subtitle: "How Dave and Buster's Fueled my Distributed Systems Addiction"
categories: 
    - "gaming"
    - "distsys"    
---

_A long time ago, a single visit to [Dave & Buster's](https://www.daveandbusters.com/) forever changed my career and fueled my passion._

<!--more-->

Get cozy, because Uncle Kevin's going to give all the kids in the front a history lesson while all the old codgers here can sit back and enjoy the nostalgia.

In the summer of an undisclosed year (early 2000s), I visited a Dave & Buster's in Houston. Off in a dark corner of the arcade sat a few **pods**. These heavily decorated behemoths were fully immersive--you sat in the pod and the door gave a firm click as it slid shut, nestling you into its cocoon. At D&B you don't pay dollars, you swipe a card, but I'm pretty sure each play of this game cost around **$5 USD** at the time, and didn't last nearly long enough. The attendant I spoke with at the time said barely any customers ever thought they got their money's worth. _They didn't know what they had on their hands_.


{{< figure
  src="/images/angled_closed_pod_on_white.jpg"
  class="class param"
  type="margin"
  label="mn-tesla"
  title="Pod of Destiny"  
  caption="Generation 4 VWE Battletech (Tesla) Pod"    
  alt="Battletech Pod"  
 >}}
Once in the pod, you could pilot a Battletech robot in an arena-style match against other players (I think it also had NPCs). If I remember correctly, we could choose between four different mech types. The controls were surprisingly involved, with a number of actual physical buttons and switches as well as different toggleable targeting views.
{{< section "end" >}}

One of my favorite shows when I was younger, **Beyond 2000**, did a story{{< sidenote "sn-by2000" >}}[1992 Episode](https://www.youtube.com/watch?v=jdJA5C_Po4U){{< /sidenote >}} on the origins of the _Virtual World Entertainment_ Battletech system. Rather than starting out inside Dave & Buster's, these things originally appeared in massive, themed centers with dozens of pods, merchandise, and all the bling. If you're curious, [Games Radar](https://www.gamesradar.com/battletech-arcades-were-decades-ahead-of-their-time-holding-global-3d-matches-before-wed-even-played-a-snes-heres-their-story/){{< sidenote "sn-warning" >}}Caution: lots of ads{{< /sidenote >}}
did a story on the history of these pods, covering the post-1992 era that included Dave & Buster's attempted revival of the technology. Today there are a number of groups trying to collect and restore{{<sidenote "sn-restore" >}}[Restoring Battletech Pods](https://www.youtube.com/watch?v=1FPaV44FW3k){{</sidenote>}} various generations of these machines.

I feel like I need to set some context here. In 1992, most people did not have access to what we now know as the Internet. When D&B had these pods wasting away in the back corners of their arcades, most people were using **56kbps** dial-up modems, or, if they were lucky, they had DSL. Average Internet speeds wouldn't be measured in _megabits/s_ until the latter half of the 2000s. Massively multiplayer global arena combat? _Unheard of, and likely viewed as black magic_.

{{< figure
  src="/images/battletech_arcade.jpg"
  class="class param"
  title="Immersive Nerdity"
  caption="Fully Immersive Mech Experience"
  label="mn-inpod"  
  alt="Person operating game controls in pod"  
 >}}
{{< section "end" >}}

I stepped into the arcade and immediately got a little hit of dopamine as I could hear the gunshots from the pistol games, punches and screams from the fighters, and I was accosted by lights from all directions. None of that mattered. As soon as I saw those pods in the back of the arcade, I _needed_ to get inside one.

I slipped inside one of those pods and I was 8 years old again, mouth agape, in awe of the gleaming _newness_ of the experience. I played a couple of times until I got the hang of the controls, then I played again and again until my game card ran out. 

Then I refilled my card.

Back then, we'd all played the racing games where multiple racers could compete in the same race. We'd seen that concept done and re-done with formula 1, motorcycles, downhill skiing, ad nauseum. But _nothing_ could compare to this experience. Multiple pods all fought in the same virtual world and, if you were in the right place at the right time, the pods in Houston could fight against those in New York or even _Tokyo_.

I'm not exaggerating when I say these games were _decades_ ahead of their time. There was a match management system operated by an attendant. They could set controls like weather and other environmental parameters. You could watch the battle on an overhead monitor, and your callsign even appeared in gleaming orange LED on the outside of the pod. It was a magical experience.

In a time before ubiquitous, high-speed Internet, before people could play MMOs on their phones, before graphics cards came in our toasters, you could slip into a Battletech robot and compete against players across the world, with people in nearby pods on your team.

When I was finally done for the night (I'd run out of money), I talked the infinitely patient attendant into letting me take a look around. The match management computer was mostly off-the-shelf, while the pods seemed to be made up of a hybrid of custom pieces and commodity Pentium computers. It and all the other pods were linked on a switched network, and, hidden below a panel in the back, was another server and an ISDN line.

The attendant couldn't answer for sure, but given how the match management system seemed to work, I figured that the arcades/game environments could _peer_ with one another on demand. In other words, nobody needed to operate central infrastructure to facilitate the games. Today, we'd likely be running game servers in "the cloud."

{{< figure
  src="/images/pod_network.png"
  class="class param"
  title="Pod Networking"
  caption="First thing I imagined after playing. I can't count the number of production apps I've made that look like this."
  label="mn-inpod"  
  alt="Diagram of Game Network"  
 >}}
{{< section "end" >}}

As I walked toward the D&B exit in a haze, all I could think about was what the network topology might look like for these systems. I imagined a local peer network where the game blasted state updates locally via UDP and an on-prem server dealt with the relay between venues.

A stream-of-consciousness conversation I had with myself went something like this:


{{< blockquote cite="kevinhoffman.blog" footer="Kevin's Brain" >}}
They probably don't need an anti-cheat arbiter because it's a closed system, so the software can trust all the packets. The bottleneck is probably the relay between venues. 

What happens if two venues try and start a match at the same time? Which one wins? Is it eventually consistent and just synchronizes and merges the overlapping matches? How did they manage to do all this with apps that run in less than 64MB of RAM?

Do the match managers at each venue present a global view of the match, or are they limited to just what's in the venue? If it's the latter, is there a "super" match view somewhere that shows the true global state of the arena?
{{< /blockquote >}}

I could ramble on and on about this game and how it made me feel, but I want to get the real point of this blog post. Throughout my life and career, there have been distinct moments burned into my memory--moments that changed and broadened my perspective.

They say a muse creates a fire in your mind, ignites a piece of your soul that would otherwise be dark and dormant. Experiences like this were my muse.

Not only did it remind me that I've always wanted to _make_ games, but I discovered that I was drawn to distributed systems problems like a moth to the flame. I could see myself solving problems like this for the rest of my life without ever getting bored.

So go out and jump head first into things you've never experienced before. Explore that underused dark corner, check underneath that random rock. Who knows, it could change everything.

_p.s. If you have a VWE Tesla pod for sale, let me know_