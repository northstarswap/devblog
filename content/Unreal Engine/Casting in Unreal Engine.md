---
tags:
  - unreal
  - programming
draft: true
---
Casting is a common but often-misunderstood part of developing an Unreal Engine application. This article provides an overview of the various cast types including the Blueprint Cast nodes as well as dispelling myths and misconceptions about them.

# Unreal Cast
Unreal's `Cast` function with a capital "C" is an Unreal-specific global function that **dynamically casts** `UObject`s between types.

## CastChecked
`CastChecked` is a variant of `Cast` that offers asserts when certain conditions are not met.

# Blueprint Cast Node
In Blueprint, one Cast node is generated per `UCLASS` type in the form of "Cast to x" nodes.
# C++ Standard Library Casts

## static_cast

## reinterpret_cast

## dynamic_cast (disabled in Unreal)