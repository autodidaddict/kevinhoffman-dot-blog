---
title: "Constraining Types with DataKinds in Haskell"
date: "2025-07-27"
#author: "Kevin Hoffman"
type: post
description: "Exploring polymorphism and type constraints in Haskell with DataKinds"
subtitle: "Exploring polymorphism and type constraints in Haskell with DataKinds"
categories: 
    - "haskell"
    - "fp"
---
_Late to the party as always, I discover why people love type systems._

<!--more-->

I'm no stranger to [type systems](https://en.wikipedia.org/wiki/Type_system) or functional programming. I still consider myself a worthless newb when it comes to Haskell, however. I have been trying to learn more and not let AI sap my inspiration and desire to tinker.

When I think about type systems, my mind usually wanders to Rust structs or even C++ class hierarchies. I'm used to defining constraints like this. For example, if I'm building a `Player` type, it's easy to define an `experience_points` field that has a value between 0 and the largest signed 64-bit whole number.

As always, I think about game models when trying to learn new concepts. In classical OOP, I might define a `GameObject` type, and then I might derive/inherit from that an `Item` (which can have inventory and be in an environment) and a `Player`. The core object has shared properties like `ID`, `name`, `environment`, etc. Players have information like a name and experience points.

This is all pretty much review for me. My next step in modeling might be to add some code that validates these objects at runtime. The main constraint I want to add is that an object can only ever have its environment set to a reference to a room. You can only ever put a thing inside a room (small nit: I know _inventory_ is a thing, but I have a different idea for that).

## Constraining Types
Here is where my lack of experience with _higher kinded types_ shows. I basically never consider the idea that I might be able to model a _compile-time_ set of types that enforces my rules. This changes the game--everything goes from hoping that I catch all the edge cases at runtime to encoding those edge cases directly in types, making it _physically impossible_ to use a type in a way that's not allowed.

In the "old school" days, I would have a field like `objectType` which might be an enum. Then I would write a bunch of tedious runtime logic to enforce rules. Let's create a sum type for this instead:

```haskell
data ObjectKind = RoomK | PlayerK | ItemK
```
Next I want to create references to these objects. Rooms are singletons, and so they lack an _instance ID_. Whereas players and items are clones of a prototype, and so include both the prototype name (e.g. `std.player` or `mordor.ring`) and an _instance ID_, which could be a short hex string like `#f3ef7e`.

I could enforce this rule about references at runtime, and try and throw errors when the constructor is used to create an instance reference to a room. Or, now that my world has been expanded, I can create a data structure that takes a _kind_ as a parameter and I can use one of the `ObjectKind` variants as a _tag_ of sorts.

```haskell
-- Rooms have no instance, only a prototype
data ObjectRef (k :: ObjectKind) where
  RoomRef :: Text -> ObjectRef 'RoomK
  InstRef :: Text -> Text -> ObjectRef

showRef :: ObjectRef k -> Text
showRef (RoomRef proto)    = proto
showRef (InstRef proto objId) = proto <> "#" <> objId
```
The `'RoomK` tag here is made possible through a language extension (`DataKinds`) that creates kind constructors from regular data constructors. In `ObjectRef`, it's _physically impossible_ to create a room reference with an instance ID because that function is of type `Text -> ObjectRef 'RoomK`.

Room references print out like `mordor.room12` while item and player references print like `std.sword#f734e`. I could use this opportunity to make a dig at "generics" and how these type system constraints are better, but I won't. Nope, I'll take the high ground.

Let's take a look at the `ObjectData` type now. Remember this is a data structure that is enforcing compile-time constraints that I might otherwise have to code (and pray it works). DataKinds with tags makes it obvious what's going on here:

```haskell
-- All objects have the same core object data
data ObjectData (k :: ObjectKind) = ObjectData
  { objRef        :: ObjectRef k
  , objName       :: Text
  , objEnv        :: ObjectRef 'RoomK
  , objInventory  :: [SomeInstRef]
  , objVisible    :: Visibility
  , objPersistent :: Bool
  }
```
`objRef` here is something akin to `self`, it's what the object looks like as a reference. Note that it's passing the kind parameter through to the `ObjectRef` data type.

Next we see `objEnv`. This field isn't just a regular `ObjectRef`, it's a compile-time verified type that can _only ever be_ a reference to a room via the `ObjectRef 'RoomK` type.

Take a look at the `objInventory` field. Here we have a list of objects that can be contained _within_ the object. All objects (including rooms) can _contain_ inventory, but rooms cannot _be_ inventory. So let's take a look at the `SomeInstRef` type, which, as the name implies, only allows references to instances (which automatically excludes rooms).

```haskell
-- Instanced Ref (no RoomRef allowed)
data InstancedRef (k :: ObjectKind) where
  IsInst :: ObjectRef k -> InstancedRef k

mkInstancedRef :: ObjectRef k -> Maybe (InstancedRef k)
mkInstancedRef ref@(InstRef _ _) = Just (IsInst ref)
mkInstancedRef (RoomRef _)       = Nothing

data SomeInstRef = forall k. SomeInstRef (InstancedRef k)

-- A wrapper for heterogeneous inventories
data SomeObjectRef = forall k. SomeRef (ObjectRef k)
deriving instance Show SomeObjectRef
```
Here a "smart constructor" is being used (`mkInstancedRef`) which prevents the creation of an instance ref to a room object. Because the pattern match in `mkInstancedRef` is just a regular pattern match, it is invoked at runtime and will give me a `Nothing` if I try and create a reference to a room.

This isn't necessarily a bad thing, but look back at the inventory type (`[SomeInstRef]`). While eventually you'll run into runtime problems using `mkInstancedRef` the type system isn't actually enforcing this at compile time. Compile time enforcement was kind of the point of this blog post.

So I created a type class called `IsInstantiable`, which will only have members of `'ItemK` and `'PlayerK`. It might seem odd that players are instantiable, but think of every logged in player as some instance of the `std.player` prototype, e.g. `std.player#bob` or `std.player#eff7de`.

```haskell
class IsInstantiable (k :: ObjectKind)a
instance IsInstantiable 'PlayerK
instance IsInstantiable 'ItemK
-- Note: No instance for 'RoomK


```
It took me a while to realize that you can create instances of a type class that takes a _kind_ as a parameter. It's still kind of blowing my mind.

And now I can modify the `SomeInstRef` type used to store heterogenous inventory to use this new constraint:

```haskell
data SomeInstRef = forall k. IsInstantiable k => 
    SomeInstRef (InstancedRef k)
```

So now the `SomeInstRef` type (note the GADT syntax again) is simply _not defined_ when the `kind` type is `'RoomK`.

You haven't yet seen the function that will add an object to an object's inventory. While we know (indirectly) that we can't create a data type unsuitable for being in inventory, we can also change the signature for `addToInventory` so that only an `IsInstantiable` type can be added to an inventory:

```haskell
addToInventory :: IsInstantiable j => ObjectData k 
    -> InstancedRef j -> ObjectData k
addToInventory obj inst =
  obj { objInventory = SomeInstRef inst : objInventory obj }
```

The `addToInventory` function can now work like this: 

```haskell
addToInventory darkDungeon creepySnake
```
Another tiny thing to note is that we don't pass a `SomeInstRef` into the function, we pass an `InstancedRef`, which is a data record. We only need the `SomeInstRef` to store heterogenous inventory items in a single list of a single type.

## Wrap-up
It might not be obvious at first. It might take a while to sink in. I know it didn't quite click for me until I had to try out `DataKind`s in anger when building something "real" rather than hello world (or worse, a _math_-related sample).

Think about building just the inventory system the "old" way. You would have to enforce the instantiability requirement at _every single call point_ that attempted to add an item to inventory. That's not bad, you might be thinking. Just make sure to route all of that through a single function that enforces the constraints. 

That sounds good on paper, but I know from painful experience that such solutions only last a short time in production. At some point in the very near future, someone's going to push an enhancement that circumvents the runtime check. 

But if we are using data kinds, type classes, and other fundamental aspects of Haskell's type system, we _literally cannot write code that violates the rules_. This is so much better in a thousand ways than runtime checking. Think about it, if your types are strict, then nobody will ever be able to push a pull request that sneaks a violation of the rules into the codebase.

I have a **ton** of other stuff that I've learned while tinkering with this, but I'll save it for another post, or maybe even another video using my disembodied avatar. Hope you geek out on this stuff as much as I do.
