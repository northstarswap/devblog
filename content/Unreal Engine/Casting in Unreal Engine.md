---
tags:
  - unreal
  - programming
draft: false
---
This article provides an overview of the various cast types including the Blueprint Cast nodes as well as dispelling myths and misconceptions about them.

# Unreal Cast
Unreal's `Cast<T>(UObject*)` is an Unreal-specific global function that **dynamically casts** `UObject`s between types. It attempts to cast the given `UObject` to the template type (`T`) using Unreal's reflection system, returning `nullptr` if the cast fails.

`Cast` can only be used with `UObject`s and is the most common type of cast used when working within a game project.

> [!important]
> `Cast<T>`'s performance cost is almost entirely negligible, even when performing thousands in a single tick.
> 
> Storing a weak reference to a casted type is wholly unnecessary in almost every scenario. Instead, I recommend writing a simple typed getter that performs the cast (`AMyActor* GetMyActor() const { return Cast<MyActor>(GetOwner()); }`)  
> 
> Casting to interfaces instead of classes does not improve performance in any way (and it doesn't in BP either, this is a myth that will be dispelled below)  
## CastChecked
`CastChecked<T>(UObject*, ECastCheckedType::Type)` is a variant of `Cast` that offers asserts when certain conditions aren't met. It should be used when you don't want to handle the cast failing.

One example of where I would use `CastChecked` is if I have a component that is only ever meant to go on one type of actor, say `AGameState`. I would create a getter like so:

```cpp
AGameState& GetGameStateOwner() const { return *CastChecked<AGameState>(GetOwner(), ECastCheckedType::NullChecked); }
```

This way, I can call `GetGameStateOwner()` and never have to think about handling the case of the owner not being `AGameState` or `nullptr`. Handling every possibility and creating multiple points of failure in an API is less robust compared to having one point of entry and being able to say "After this point in the API, the owner must be a valid `AGameState`."

The 2nd argument `ECastCheckedType` allows you to specify if you want to assert on `nullptr` in addition to a type mismatch via `NullAllowed`/`NullChecked`.
# Blueprint Cast Node
In Blueprint, one Cast node is generated per `UCLASS` type in the form of "Cast to x" nodes. This is often simply called "BP Cast."

One common misnomer that many Unreal tutorials teach is that BP Cast has significantly worse performance than `Cast<T>` and that using interfaces is a way to mitigate this performance loss. This comes from a seed of truth about using a cast node in a BP graph, but it's not telling the whole story about *why* using an interface *might* be a good idea (it's often not even the correct solution depending on the problem anyways)

Unlike C++ classes which are loaded and instantiated on engine start, BP classes exist as assets that are loaded on-demand. When you place a Cast node in a BP graph, **the BP that contains it now has that class as a hard reference/dependency**. 

What this means in practice is that you've declared that in order for the class whose graph contains the Cast node to function, **it needs the entire class referenced by the Cast node to also be loaded, including *that* class's hard dependencies**. 

Unreal does not async load assets for you, so if you neglect proper async asset management and synchronous/hard load the object with the Cast node, the class referenced in the Cast node will also be hard loaded. During this entire process, **the main application thread (game thread) will be blocked and the entire application will stall**. 

Not only that, but **the async loading queue will be flushed**, causing all in-progress async loads happening on other threads to stop and restart as synchronous loads on the game thread causing—at best—a giant hitch/stutter and at worst several seconds of the application freezing possibly leading to the OS prompting the user to close the application or the application just closing itself on consoles.

**None of this explicitly has to do with the BP Cast node/Casting**. These are the same considerations you'd have to contend with if you were to include a heavy `UPROPERTY() TObjectPtr<T>` or `TSubclassOf<T>` directly on an actor.

Interfaces or the new "soft cast" node are often presented as a solution to this, but they're a half-measure that doesn't address the root issue: **the object with the Cast node should have been fully loaded asynchronously on another thread by the time it's being used during gameplay**. This requires implementing a proper [asset management scheme](Asset%20Management) using [soft references](Unreal%20Pointers).
# Standard Library Casts
Besides `dynamic_cast` which isn't usable in an Unreal codebase, standard casts can be thought of as different levels of bypassing compile-time checks and saying "I know what I'm doing here."

**`static` and `reinterpret` casts do not check for type and return nullptr if the check fails**. There is zero feedbaclk at runtime whether a compile-time cast succeeded or failed because **they don't exist at runtime, nor do classes**. C++ is a compiled language and everything turns into a raw memory slurry after compilation. 

Unreal's reflection system only applies to the `U`-types which is why you need to add all the strange macros everywhere, some of which are fake and only act as markers for UHT to generate special reflection code. It's not applicable here.
## static_cast
`static_cast<T>` is a **compile-time** cast that allows you to "forcibly" use one type as another, as long as the incoming object could *potentially* be the type you're casting to according to the class/struct inheritance tree. If it's not potentially compatible based on this information, the compiler will throw an error on compile.

To reiterate, **it will not return nullptr if the cast fails** or otherwise notify you about this. It doesn't exist at runtime. It's a soft compiler type check bypass. 
## reinterpret_cast
`reinterpret_cast<T>` is a **compile-time** cast that's similar to `static_cast` except it can be considered as a "hard bypass" that skips any compile-time checks. You can think of it as the "I *really* know what I'm doing" cast.

Its main use is to reinterpret between `void*` and an actual, usable type. It can also be used to cast between types that you know have the same memory layout but the compiler would have no way of understanding that. 

An example of this scenario is `FInstancedStruct` and `TInstancedStruct<T>`. `TInstancedStruct` has a single `FInstancedStruct` member and no virtuals, so its layout is identical in memory. These two types can be `reinterpret_cast` between each other.
## dynamic_cast (not used in Unreal)
In a non-Unreal codebase, `dynamic_cast` acts as an equivalent of `Cast<T>`, using C++'s "RTTI" reflection. RTTI is often disabled in vanilla C++ codebases anyways as most applications that need dynamic casting like Unreal also need other reflection features which the barebones standard RTTI doesn't offer.