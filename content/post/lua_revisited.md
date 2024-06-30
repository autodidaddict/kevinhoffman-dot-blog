---
title: "Revisited: Lua in Elixir for Fun and Games"
date: "2024-06-30"
toc: false
#author: "Kevin Hoffman"
type: post
description: "A fresh look at an old post about Elixir and Lua"
subtitle: "Re-evaluating my decisions and code from a blog post I wrote long ago"
categories: 
    - "lua"
    - "elixir"
    - "mud"
---
_Leveraging the benefit of hindsight and experience to take a look back at some code and opinions._

<!--more-->

Many, many years ago, when the earth was a cold and dark place (6 years ago), I wrote a blog post entitled
[Hosting a Lua Script inside an Elixir GenServer for Fun and Games](https://kevinhoffman.medium.com/hosting-a-lua-script-inside-an-elixir-genserver-for-fun-and-games-2c0662660007). {{< sidenote "sn-medium" >}}I haven't posted on Medium since they started paywalling regular user's content without paying regular users, but that's a rant for another day.{{< /sidenote >}}

Throughout my entire life, I've used the act of building a MUD (or something vaguely MUD-like) as a means to learn new languages and technologies. The joy and awe that I got from programming those in high school and college is something I know I'll never experience again, yet I try and recreate those moments even now.

The opinion that I drew in the original blog post was that I could use a `GenServer` to house the current state of a Lua script, which represented a _game object_. Every loaded game object would therefore be a process that clients could interact with via standard `GenServer` client APIs like `cast` and `call`.

As an example, I could interact with a game object as follows:

```terminal
iex> {:ok, pid} = Game.Object.start_link("/path/to/object.lua")
```
And this would then use `luerl` to initialize the script:

```elixir
def init(room_script) do           
    root = :luerl.init()
    {_result, state} = 
        :luerl.dofile(room_script |> String.to_charlist(), 
            state)
    {:ok, state}
end
```

After having used Elixir in anger for many production projects since I wrote that original blog, I've seen (and agreed with) the opinion that you shouldn't model your process hierarchy the way your mental model represents your domain. You don't want to create an OOP hierarchy out of processes because that will likely end up producing very inefficient code.

The so-called right way to do things is to create a _pure functional core_ and then layer on things like concurrency concerns later. So rather than having a `GenServer` for each game object, I just have a variable that represents the current state of a script.

Let's walk through what this kind of refactor might look like and see if I run into any problems (I'm a _discovery_ developer... I generally can't finish a design until I start typing).

Let's initialize a game object (which is just a struct now):

```elixir
{:ok, game_object} = Game.Object.load("paladins_room.lua")
```

What I get back here is opaque to the client. I shouldn't care what's in it, as I'm just going to use it as a _token_ when I do pipelined calls. It's just going to be a black box parameter that I continue to pass in to functions in the `Game.Object` module.

In a pure functional world, I'm not allowed to have side effects. Things like using I/O to send messages to players are side effects, so how do I run the following script as a pure function? (this is the same script from my original blog post):

```lua
add_item("lever",
    { "switch", "rusty lever" },
    "There is a suspicious looking lever here.")

add_action("pull", "lever", "pull_lever")

function short()
    return "A basic room"
end

function long()
    return "You find yourself in the most basic of rooms."
end

function pull_lever (ob)
    if ob.class == "paladin" then
        tell(ob, "The lever glows slightly in your grasp.")
        move(ob, "~/paladin_hq")
        return true
    end
    tell(ob, "The lever does not budge")
    say(ob, ob.name .. " tries to pull the lever, but fails.")
    return false    
end
```
The answer is another common refactoring pattern. Instead of _doing_ a set of actions, which would be a side effect, I can _return_ a list of actions to be performed. This not only lets me continue my refactor in a fully functional world, but it should make testing my scripts and code much easier, as I can just assert that a given input results in a given output of actions. First, so you don't have to keep switching over to the old post, let's take a look at how this works inside a `GenServer` with side effects:

```elixir
iex(5)> GenServer.cast(pid, 
    {:do_action, 
    %{name: "kevin", player_id: "12", class: "paladin"}, 
    ["pull", "lever"]})
telling
:ok
[
  [{"class", "paladin"}, 
   {"name", "kevin"}, 
   {"player_id", "12"}],
  "The lever glows slightly in your grasp."
]
moving
[
    [{"class", "paladin"}, {"name", "kevin"}, {"player_id", "12"}], 
    "~/paladin_hq"
]
```
The `telling` and `moving` output lines are debug prints from effectful functions that are (assuming the code is finished) going to send messages to a player (e.g. their socket) and then manipulate the state of one object by moving it into the inventory (room contents) of another.

This actually gives us a good hint at how we can make a functional refactor:

```elixir
{:ok, game_object, actions} = Game.Object.do_action(
    game_object, 
    "pull_lever",
    %{name: "kevin", player_id: "12", class: "paladin"})
```
And this would then produce an action list like the following:
```elixir
actions
[
    {:tell_player, {object_ref: "12"}, 
        "The lever glows slightly in your grasp."},
    {:move_object, 
        {from_ref: "paladins_room", 
         to_ref: "paladin_hq"} }
]
```
Now, the `actions` variable can then be sent to one or more `GenServer`s responsible for communication with players and state management. Outside of the functional core, something can send a message to a player based on the `:tell_player` command. The cool part is we now have freedom to send that message to the player's connected socket, to a NATS messaging subject for delivery somewhere, or published via `Phoenix.PubSub`.

This is all well and good, but there's a big gap between the "hello world" of refactoring to a functional core and really doing it. Let's say we've got code in our Paladin room that looks like this:

```lua
statue = get_object("paladin_foyer/statue.lua")
if ob.class == "paladin" && statue.get_property("triggered") then
    tell(ob, "The lever glows slightly in your grasp.")
    move(ob, "~/paladin_hq")
    return true
end
```
In this new script we only allow Paladins to be teleported to the HQ if their class is `"paladin"` _and_ the statue in a nearby foyer has been pulled back like a lever/trigger.

Now things are getting tricky. This is where, in the past, I would probably get off the pure functional train and go back to `GenServer` modeling. Where does this pure function get its list of objects from which to call `get_object`, and where does it get the state to retrieve the `triggered` property of the object? The answer, of course, is **Monads**.{{< sidenote "sn-medium" >}}I'm kidding. I just felt like I had to troll some readers. Elixir doesn't have Monads... _or does it_ ??{{< /sidenote >}}

When people talk about the "IO Monad", an analogy they like to use is that you _pass in the entire universe_ as a parameter to the function, which accesses the universe, performed effects on the universe, and returns the new universe. This neatly bundles all side effects in "universe mutations". While the academic analogies refer to a metaphorical universe, we can actually use this idea and a "real" universe to make `get_object` and `get_property` work.

```elixir
{:ok, game_object, actions, universe} = Game.Object.do_action(
    universe,
    game_object, 
    "pull_lever",
    %{name: "kevin", player_id: "12", class: "paladin"})
```
Here we pass in the universe _as it existed at the exact moment the function was called_, perform the action, and get a brand new universe (remember Elixir has no mutable variables) in return. It's worth pointing out that processing commands from the command list can also change the state of the universe (e.g. moving the player to the Paladin HQ).

Let's think about the impact this has on testing.

{{< blockquote footer="And there was much rejoicing" >}}
_For any given state of the universe, performing a given action with a given input will produce exactly one predictable output of commands and a new universe._ (assuming randomness can be predicted with a seed injection, but that's yet another topic)
{{< /blockquote >}}

The universe could be a simple Elixir map like this:

```elixir
%{
    objects: [...],
    players: [...],
    stuff: %{},
    more_stuff: %{}
}
```
While this is the simplest way to deal with it, something about the idea of passing a universe containing every piece of data about every single loaded object feels _heavyweight_. In some other language, like Rust, I can _move_ the entire universe into the function, mutate it internally, and then move it back out again. For the duration of that move, no other code can access that universe.

Elixir and other functional languages deal with this by not allowing any data mutation at all. If I pass the universe into a function, and then return a changed version of that universe, I've made a _copy_. But, have I copied every single value in the entire tree? _It depends_. When we pass a map (`%{foo: "bar"}`) to an Elixir function, it passes a pointer to that map into the function. If I then return a copy of that map with only one field changed, Elixir is going to return a _completely new value_, but yet somehow (I don't understand the internals) some internal pointer magic makes it so we don't actually have two fully populated structs, we have the original and then some magic. This is possible because the original is immutable, so, unlike Rust, we don't have to go to great lengths to ensure a value doesn't mutate underneath a shared pointer.

If, on the other hand, I decide to pass the universe _in a message_ sent to another process (e.g. a `GenServer.call` or `GenServer.cast`), then Elixir will actually make a true _deep_ copy of the universe prior to sending.

The universe can also be entirely opaque. For example, we could create a universe, create a game object in that universe, and then call a function in the object, all without knowing the implementation details of the universe. Note that I don't need to return a universe from `do_action`, since the list of commands should be the only way to make changes to a universe.

```elixir
{:ok, universe} = Game.Universe.new()
{:ok, game_object, universe} = Game.Object.init("paladin_room.lua")
{:ok, game_object, actions} = Game.Object.do_action(
    universe,
    game_object, 
    "pull_lever",
    current_player)

{:ok, universe} = Game.Dispatcher.dispatch(actions, universe)
```
At this point, we can hand `actions` off to some dispatcher (which _should_ be a `GenServer`), and we're content knowing that this universe is consistent with the changes the script wanted to make.

Here, every mutation we want to do to a universe can be done via `Game.Universe` functions. When we do this, we can have the universe be a map (or any other data structure). The function doesn't manipulate some globally shared universe. It can only return the universe contained within (or pointed to) by the parameter.

We're at another turning point where it would be easy to jump off this silly functional bandwagon. How do we make sure that other functions being called on other objects have the most up to date version of the universe? If each function can return a changed universe, how do we reconcile a single consistent universe? Ugh, this feels too much like _work_.

Lets say we have a central `GenServer` (or pool thereof) called `Game.CommandHandler`. This server is responsible for accepting commands from players, through whatever channels are available (telnet, NATS, websockets, etc). This command handler could have a handler function that looks like this (psuedo-code):

```elixir
def handle_input({player, input}, state) do
    {:ok, parsed} = parse_input()
    {:ok, gobj} = 
        find_target_object(player, state.universe, parsed)
    {:ok, commands} = 
        Game.Object.handle_input(gobj, input, state.universe)
    {:ok, universe} = 
        Game.Dispatcher.dispatch_commands(state.universe, commands)
    
    state.universe = universe
    {:ok, state}
end
```
Obviously there's some hand-waving between parsing the input and finding the target of the action, but that's kind of a concern of the parser grammar, and the universe is just used to locate the player, and then apply their input to their immediate surroundings.

If `handle_input` is done inside a `GenSever`'s  `handle_xxx` function, then we know that it is single-threaded because all messages are handled through that server's mailbox. We also know that calling dispatch commands will manipulate the universe in isolation because it happens in the contents of a mailbox receive.

Now we're in a bit of a pickle. We've refactored all the game logic and script handling so that it's purely functional, and running a function in a game script will return a list of commands for dispatch (to be used to mutate the universe). The problem is that now all player input is going through a single pipe.

At this point I'm faced with a question: Do I continue jumping over hoops in the name of functional purity, or is there some kind of compromise I'm willing to make? Remember my goal was to refactor my use of `GenServer`s to be more efficient, not to write the most pristine pure functional code ever.

What if the universe was an ETS table instead of a map that gets passed around ad hoc?{{< sidenote "sn-medium" >}}We could probably have the universe be an `Agent` as well, but that might result in the same kind of bottleneck as a single input handler.{{< /sidenote >}} 

In an ETS table, concurrent access is already taken care of for us. We could continue to use the `Game.Universe` module to broker access to the underlying data because our design is already around an opaque universe "handle". Now instead of returning the new universe, we can manipulate the universe indicated by the handle (which would in turn point to an ETS table):

```elixir
def handle_input({player, input}, state) do
    {:ok, parsed} = parse_input()
    {:ok, gobj} = 
        find_target_object(player, state.universe, parsed)
    {:ok, commands} = 
        Game.Object.handle_input(gobj, input, state.universe)
    :ok = 
        Game.Dispatcher.dispatch_commands(state.universe, commands)
        
    {:ok, state}
end
```

With this compromise, I think I might've gotten the right set of tradeoffs. All of the game script functions still operate on a _given_ universe, and the only thing that can mutate a universe is the command dispatcher. This means I can create a test universe as a fixture in all of my tests, or multiple universes in multiple fixtures. When running the game, I can have pools of both dispatchers and input handlers to support large number of concurrent players while not lining everyone up behind a single queue. The only time we'll run into "lock waits" is when multiple concurrent activities attempt to change the same piece of data. We can avoid conflicts here with good ETS table organization and possibly by having a single-threaded command dispatcher.

I could even utilize the concept of multiple isolated universes in the game itself. There could be "regions" in which code can't access the state of code from other regions. In this case, the chance for lock waiters goes down even more because we can have smaller, less populated "shards". It wouldn't take much to "move" a player from one universe to another as they cross a zone barrier.

I'm intrigued by the changes here and I can definitely see the inefficiencies in the design from my original blog post. I wonder what I'll think of these changes 6 years from now, when humans live in other galaxies far, far away.