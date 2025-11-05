---
title: Unreal Pointer Type Reference
tags:
  - Unreal
  - Programming
---
This guide covers Unreal’s wrappers for managing lifetimes and references.
# Terminology
* **Strong Reference**: Reference that keeps what it's pointing to alive.
* **Weak Reference**: Reference that does not keep what it's pointing to alive and may become invalid at any time.
* **Hard Reference**: Reference to an object stored on disk that is a dependency for the type in which this member resides. Side effects explained below.
* **Soft Reference**: Reference to an object stored on disk that may or may not be currently loaded.

# UObject GC Pointers
The `UObject` pointers are used to manage the lifetime of a `UObject`.

`UObject`s are garbage collector (GC) managed objects that will be destroyed when the GC runs if there isn't an unbroken path from the "root" to the object through a chain of **strong references** through other `UObject`s and `USTRUCT`s. 

`UObject`s are always allocated on the heap and cannot be created on the stack nor created without the use of `NewObject`. 

In most code, checking for `nullptr` against a `UObject` isn't enough to confirm that it's in a valid state. `bool IsValid(const UObject* Test)` is a global function designed to test for both `nullptr` and any special `UObject` invalid flags such as being marked for GC. In most code, **prefer `IsValid` over a simple nullptr check** to ensure you aren't operating on an object that is—for all intents and purposes—dead.   

In reflection, `UObject`s are always passed around via pointers. However, `UObject&` is perfectly acceptable to use in C++, provided `IsValid` is checked before dereferencing. I personally strongly prefer using `UObject&` as it represents a promise that it was checked for `IsValid` before dereferencing and if we run into undesired behavior regarding lifetimes, we only have one source to blame—the single dereferencer.

## TObjectPtr
`TObjectPtr<T>` is the most common type of `UObject` pointer wrapper. It's a **strong reference** and a **hard reference** when marked with `UPROPERTY()`—and this is the only time it should be used. It serves as the replacment to raw `T*` pointers in 5.0+ to `UObject`s in `UPROPERTY()` members to enable [Incremental GC](https://dev.epicgames.com/documentation/en-us/unreal-engine/incremental-garbage-collection-in-unreal-engine) in 5.4+. 

Per Epic's documentation, `TObjectPtr` should only ever be used as a `UPROPERTY`'d member of a `UCLASS` or `USTRUCT`, never as a function parameter nor as a local variable. `TObjectPtr` has operators to implicitly convert into `T*`, so anywhere `T*` is accepted, `TObjectPtr<T>` can fit right in without any boilerplate. 

It purpose is to assert ownership over a `UObject` by a `UCLASS` or `USTRUCT` and tie its lifecycle to its owner's for clean memory management. 

If you need to hold on to a reference to an object in two places, it's more likely you want a [TWeakObjectPtr](#tweakobjectptr) in the non-owning location.

It's recommended to migrate legacy code using raw `T*` pointers to `TObjectPtr<T>` for compatiblility with future engine versions. On 5.4+ UHT will emit warnings for `UPROPERTY`'d raw `T*` pointers.

In terms of asset management, `TObjectPtr` being a hard reference means that if a `UObject` stored as an asset is pointed to by `TObjectPtr`, loading the owner of this `TObjectPtr` will hang the game thread while it forces a synchronous load. This is usually considered bad practice and a common culprit for large stutters in games. Asynchronously loading the owner of a `TObjectPtr` will also asynchronously load the asset pointed to by the `TObjectPtr`. See [this section](#) for types that avoid these hard references and allow you to perform async loading operations.  

## TWeakObjectPtr
`TWeakObjectPtr<T>` is a **weak reference** to a `UObject`. It does not keep the object alive and may become invalid at any time.

`UObject`s referenced by `TWeakObjectPtr` may be remotely GC'd at any time. Before dereferencing or using `TWeakObjectPtr`, always check for `IsValid()` on the `TWeakObjectPtr`.

> [!info]
> If you call `Get()` on a `TWeakObjectPtr` there's no need to call `IsValid()` before. `Get()` internally performs an `IsValid()` check and returns `nullptr` if it isn't valid. You only need to check for `nullptr`.

It's mainly useful when you want a `UObject` to be owned by one object, but observed by another. For example, a UI widget may want to observe a `UObject` that holds score information without taking on the responsibility of owning it and keeping it alive. Failure to use a `TWeakObjectPtr` in this situation would lead to the widget keeping the score object alive longer than necessary, potentially leading to memory leaks and unpredictable behavior.

`TWeakObjectPtr` is compatible with `UPROPERTY` and appears in reflection (including Blueprint) just like a regular `TObjectPtr`.

## TStrongObjectPtr
`TStrongObjectPtr<T>` is an advanced usecase **strong reference** to a `UObject`. Its main use is to keep a `UObject` alive when outside of the normal `UObject` property chain, such as in systems using many non-reflected types.

Use of `TStrongObjectPtr` should be kept to a minimum, as it breaks expectations for how the GC works. You should only use it if you have a firm grasp on what you're doing and have a strong justification for not staying inside the normal realm of `TObjectPtr` and `TWeakObjectPtr`. Runaway `TStrongObjectPtr` references can be hard to track down and cause memory leaks and strange behavior where `UObject`s aren't freed when they should be.

### TStrongObjectPtr on <5.5
Prior to 5.5, `TStrongObjectPtr` was a wrapper for `FGCObject`, making it extremely heavy compared to the other Unreal GC pointers. On 5.5+, the implementation has changed internally to use `AddReferencedObjects` with a proper refcount implementation. `TStrongObjectPtr` should still be avoided if possible, but on 5.5+ it's no longer the performance tanker it used to be.

# Unreal Smart Pointers
**Unreal's Smart Pointers** are Epic's reimplementation of the C++ standard library's smart pointers with a handful of extra features added on. 

Smart Pointers are reference-counted automatic memory types that are the generally-recommended way to handle heap-allocated memory in modern C++ as opposed to manually managing it via `new` and `delete`. When there are no more **strong references** to a smart pointer-allocated object, the referenced object will be **immediately** destroyed (no waiting around for GC as with the `UObject` GC system)

Unlike `UObject` GC pointer types, Smart Pointers can be used for any type except those with special lifetime requirements like `UObject`, regardless of whether they're reflected or not (This means you can use them with both `USTRUCT`s and non-reflected classes and structs)

For a more comprehensive summary of Smart Pointers in general, please see [this article](https://www.geeksforgeeks.org/cpp/smart-pointers-cpp/).

## TSharedPtr
`TSharedPtr<T>` is the Unreal equivalent of `std::shared_ptr`. It's a **strong reference** that keeps its referenced object alive by adding to its refcount.

`TSharedPtr` is often used as a member variable that may be shared externally later, as an element type of an array (since `TSharedRef`'s non-nullability doesn't play nice with arrays, as a nullable return value, and occasionally as a nullable function parameter.
## TSharedRef
`TSharedRef<T>` is an Unreal-exclusive smart pointer type with no `std` equivalent. It is in effect simply a `TSharedPtr` that promises to never be null/invalid.

This is the preferred type to use in function parameters and "make" functions return values as allowing a passed-in type to be nullable is more often than not undesirable. 

`TSharedPtr` can be converted to `TSharedRef` via `ToSharedRef()`, but be aware that this will hit an assert if the `TSharedPtr` is null since you're going from a nullable type to a non-nullable type. Always make sure to check for `TSharedPtr<T>::IsValid()` before "dereferencing" with `ToSharedRef()` . 

`TSharedRef` is not recommended to be used as a class or struct member as they must be externally initialized with *something* as they can never be null. `TShardPtr` or `TUniquePtr` are often more suitable for this usecase.

## TWeakPtr
`TWeakPtr` is the Unreal equivalent of `std::weak_ptr`. It's a **weak reference** that does not keep its referenced object alive and does not contribute to the refcount.

Objects referenced by a `TWeakPtr` may be remotely nulled at any time. Before using a `TWeakPtr`, it is necessary to call `Pin()` to convert it into a `TSharedPtr` that may or may not be valid. Check `IsValid()` on the returned `TSharedPtr` before dereferencing and using a pinned `TWeakPtr`.

Like `TWeakObjectPtr`, the purpose of this type is to keep a non-owning reference to an object that you want to observe without taking on the responsibility of managing its lifetime.
## TUniquePtr
`TUniquePtr` is Unreal's equivalent of `std::unique_ptr`. It's a **strong reference** that keeps its referenced object alive.

Only one reference made to the referenced object allocated inside a `TUniquePtr` may ever exist. This means no `TWeakPtr`s, `TSharedPtr`s, nor `TSharedRef`s may be created to remotely reference or keep the referenced object alive.

The primary purpose of `TUniquePtr` is to be used as a member variable that isn't meant to be shred externally. This allows Smart Pointer automatic memory management to clean up the `TUniquePtr`'s object when the owner is destroyed while preventing rogue external references from being made. 

## Creating Smart Pointers
`MakeShared` and `MakeShareable` are Unreal's equivalent to `std::make_shared`. They are the preferred way to create `TSharedRef` instances (that are easily convertible to `TSharedPtr` and `TWeakPtr`).

### MakeShared
`MakeShared` is generally the preferred function to use if possible, as it allocates the Smart Pointer control block and the object memory in a single memory allocation, making it more performant. `MakeShared` takes a template argument as the referenced type and takes **variadic params** that it tries to fit into one of the template type's constructors. For example:

```cpp
struct FFoo
{
public:
	FFoo() = default;
	
	FFoo(int32 InMyInt, InMyString) : 
		MyInt(InMyInt),
		MyString(InMyString)
		{}
private:
	int32 MyInt = 0;
	FString MyString;
}

//...
// This works because the default ctor isn't deleted
TSharedRef<FFoo> NewFoo = MakeShared<FFoo>();

// This also works because we have a ctor that fits the variadic arguments
TSharedRef<FFoo> NewFooWithArgs = MakeShared<FFoo>(5, "Bar");
```

`MakeShared` also supports both a threadsafe and non-threadsafe mode in its template args. By default, Smart Pointers created by `MakeShared` are threadsafe.
### MakeShareable
`MakeShared` uses different syntax as it takes an existing raw heap-allocated pointer and converts it to a Smart Pointer. 

`MakeShared` also has an overload that supports custom deleters. Simply pass a deleter functor as the second argument and the template should pick it up. If you implement a custom deleter, remember to call `FMemory::Free` to actually free the memory, otherwise you'll have a memory leak.

It's generally recommended to avoid `MakeShareable` if possible as it will cause 2 allocations to occur as opposed to `MakeShared` where the referenced object is allocated alongside its control block in one go.

`MakeShareable`'s syntax is as follows:

```cpp
struct FFoo
{
public:
	FFoo() = default;
	
	FFoo(int32 InMyInt, InMyString) : 
		MyInt(InMyInt),
		MyString(InMyString)
		{}
private:
	int32 MyInt = 0;
	FString MyString;
}

//...
TSharedRef<FFoo> NewFoo = MakeShareable();

TSharedRef<FFoo> NewFooWithArgs = MakeShared<FFoo>(new FFoo(5, "Bar"));
```

### MakeUnique and MakeUniqueForOverwrite
`MakeUnique` and `MakeUniqueForOverwrite` are Unreal's equivalent of `std::make_unique` and `std::make_unique_for_overwrite` respectively, both used for creating `TUniquePtr`s.

`MakeUnique` takes variadic arguments in the same was as `MakeShared` while `MakeUniqueForOverwrite` takes no arguments. Additionally, `MakeUnique` zeroes the memory of the newly-allocated referenced object while `MakeUniqueForOverwrite` leaves it uninitialized. 

It's generally recommended to use `MakeUnique` unless you have a specific need for `MakeUniqueForOverwrite`.

## TSharedFromThis
When creating a class or struct that's intended to be allocated inside a Smart Pointer, it's often a good idea to inherit `TSharedFromThis<T>`:

```cpp
struct FFoo: TSharedFromThis<FFoo>
{}
```

This enables the use of `SharedThis(this)` from inside the type, allowing it to pass a reference to itself as a shared pointer. This is particularly useful when creating delegate bindings with `CreateSP`.

# Soft Asset Pointers
Unreal's asset pointers are an abstraction for assets located on disk that makes them behave like a normal `UObject` pointer, however they may or may not be loaded at any given time. Internally, they are just string paths to the asset in the application's virtual filesystem. 

The two main types of soft asset pointer are `TSoftObjectPtr` and `TSoftClassPtr`, which are recommended to be used in most usecases over the non-typesafe `FSoftObjectPtr`/`FSoftClassPtr`/`FSoftObjectPath`/`FSoftClassPath`. The only difference is that `TSoftObjectPtr` is for any `UObject` asset and `TSoftClassPtr` is for `UClass` assets specifically.

When the owner of a soft pointer is loaded, any soft pointers on that object may or may not be valid at any given time. You should check for `IsValid()` to see if the object is **currently** loaded or `IsNull()` to check if it could ever be valid if it were loaded (if it has a valid path at all).

Soft pointers **do not** keep the assets that they reference alive after being loaded. You must either store a **strong ref** somewhere or store the handle returned by `FStreamableManager::RequestAsyncLoad` to keep it alive and loaded.

Calling `Release()` on `FStreamableHandle` will cause *that* handle's reference to drop, allowing an object to *potentially* be unloaded. The normal Unreal `UObject` GC rules apply—there must be zero unbroken paths from the root to the object for it to be garbage collected (and "returned" to being only on the disk so-to-speak)

Loading a soft pointer that's already loaded will not load an asset twice. Loaded assets are globally unique and the operation will complete instantaneously.

It's expected in non-editor code that you load soft pointers asynchronously (on another thread) so that the load operation doesn't hang the game thread:
```cpp
// Assuming we're in a UObject-derived class

UPROPERTY(EditAnywhere)
TSoftObjectPtr<UStaticMesh> SoftMesh;

// Keeps the mesh alive on its own by being stored
TSharedPtr<FStreamableHandle> MeshHandle;

void OnSoftMeshLoaded()
{
	// SoftMesh is now valid and we can do something with it.
	// We can call Release() on MeshHandle once we're done with SoftMesh to release our strong ref. Other strong refs may exist, so this does not guarantee freeing it.
}

void LoadSoftMesh()
{
	if (SoftMesh.IsNull())
	{
		return;
	}
	
	FStreamableManager& SM = UAssetManager::Get().GetStreamableManager();
	
	MeshHandle = SM.RequestAsyncLoad(SoftMesh, FStreamableDelegate::CreateUObject(this, &ThisClass::OnSoftMeshLoaded));
}
```

>[!info]
>5.6 introduced the `LoadAsync` function on `TSoftObjectPtr`/`TSoftClassPtr` that takes an FLoadSoftObjectPathAsyncDelegate, which eliminates the need to call `RequestAsyncLoad` through the `StreamableManager` directly. It exposes the loaded `UObject` through its delegate parameters and you are expected to store it in a strong ref to keep it alive if you need to do so.