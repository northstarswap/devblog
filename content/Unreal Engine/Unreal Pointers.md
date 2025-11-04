---
title: Unreal UObject and Smart Pointer Guide
tags:
  - Unreal
  - Programming
draft: true
---
# Terminology
* **Strong Reference**: Reference that keeps what it's pointing to alive.
* **Weak Reference**: Reference that does not keep what it's pointing to alive and may become invalid at any time.
* **Hard Reference**: Reference to an object stored on disk that is a dependency for the type in which this member resides. Side effects explained below.
* **Soft Reference**: Reference to an object stored on disk that may or may not be currently loaded.

# Unreal UObject GC Pointers
The `UObject` pointers are, unsurprisingly, used to manage the lifetime of a `UObject`.

`UObject`s are garbage collector (GC) managed objects that will be destroyed when the GC runs if there isn't an unbroken path from the "root" to the object through a chain of **strong references** through other `UObject`s and `USTRUCT`s. 

`UObject`s are always allocated on the heap and cannot be created on the stack nor created without the use of `NewObject`. 

In most code, checking for `nullptr` against a `UObject` isn't enough to confirm that it's in a valid state. `bool IsValid(const UObject* Test)` is a global function designed to test for both `nullptr` and any special `UObject` invalid flags such as being marked for GC. In most code, **prefer `IsValid` over a simple nullptr check** to ensure you aren't operating on an object that is—for all intents and purposes—dead.   

In reflection, `UObject`s are always passed around via pointers. However, `UObject&` is perfectly acceptable to use in C++, provided `IsValid` is checked before dereferencing. I strongly prefer using `UObject&` as it represents a promise that it was checked for `IsValid` before dereferencing and if we run into undesired behavior regarding lifetimes, we only have one source to blame—the single dereferencer.

## TObjectPtr
`TObjectPtr<T>` is the most common type of `UObject` pointer wrapper. It's a **strong reference** when marked with `UPROPERTY()`—and this is the only time it should be used. It serves as the replacment to raw `T*` pointers in 5.0+ to `UObject`s in `UPROPERTY()` members to enable [Incremental GC](https://dev.epicgames.com/documentation/en-us/unreal-engine/incremental-garbage-collection-in-unreal-engine) in 5.4+. 

Per Epic's documentation, `TObjectPtr` should only ever be used as a `UPROPERTY`'d member of a `UCLASS` or `USTRUCT`, never as a function parameter nor as a local variable. `TObjectPtr` has operators to implicitly convert into `T*`, so anywhere `T*` is accepted, `TObjectPtr<T>` can fit right in without any boilerplate. 

It purpose is to assert ownership over a `UObject` by a `UCLASS` or `USTRUCT` and tie its lifecycle to its owner's for clean memory management. 

If you need to hold on to a reference to an object in two places, it's more likely you want a [TWeakObjectPtr](#TWeakObjectPtr) in the non-owning location.

It's recommended to migrate legacy code using raw `T*` pointers to `TObjectPtr<T>` for compatiblility with future engine versions. Currently on 5.4+ UHT will emit warnings for `UPROPERTY`'d raw `T*` pointers.

## TWeakObjectPtr
`TWeakObjectPtr<T>` is a **weak reference** to a `UObject`. It does not keep the object alive and may become invalid at any time.

It's mainly useful when you want a `UObject` to be owned by one object, but observed by another. For example, a UI widget may want to observe a `UObject` that holds score information without taking on the responsibility of owning it and keeping it alive. Failure to use a `TWeakObjectPtr` in this situation would lead to the widget keeping the score object alive longer than necessary, potentially leading to memory leaks and unpredictable behavior.

`TWeakObjectPtr` is compatible with `UPROPERTY` and appears in reflection (including Blueprint) just like a regular `TObjectPtr`.

## TStrongObjectPtr
`TStrongObjectPtr<T>` is an advanced usecase **strong reference** to a `UObject`. Its main use is to keep a `UObject` alive when outside of the normal `UObject` property chain, such as in systems using many non-reflected types.

Use of `TStrongObjectPtr` should be kept to a minimum, as it breaks expectations for how the GC works. You should only use it if you have a firm grasp on what you're doing and have a strong justification for not staying inside the normal realm of `TObjectPtr` and `TWeakObjectPtr`. Runaway `TStrongObjectPtr` references can be hard to track down and cause memory leaks and strange behavior where `UObject`s aren't freed when they should be.

### TStrongObjectPtr on <5.5
Prior to 5.5, `TStrongObjectPtr` was a wrapper for `FGCObject`, making it extremely heavy compared to the other Unreal GC pointers. On 5.5+, the implementation has changed internally to use `AddReferencedObjects` with a proper refcount implementation. `TStrongObjectPtr` should still be avoided if possible, but on 5.5+ it's no longer the performance tanker it used to be.

# Unreal Smart Pointers
**Unreal's Smart Pointers** are a feature reimplemented from the C++ standard library with a handful of extra features added on. 