---
title: Everything you need to know about Actor Components
tags:
  - Unreal
  - Programming
---
# Declaring component pointer members and instantiating components
Epic's documentation makes a handful of mistakes when demonstrating how to correctly declare component pointer members and instantiate them in a constructor (often called a "ctor")

The following is an example of the correct way to add component pointer members to an actor in 5.0+ (for example's sake: a capsule component and a static mesh component):

```cpp
// NsActor.h
//...
UCLASS()
class ANsActor : public AActor
{
	GENERATED_BODY()

public:
	ANsActor();
	
public:
	UPROPERTY(VisibleAnywhere, BlueprintReadOnly)
	TObjectPtr<UCapsuleComponent> CapsuleComponent;
	
	UPROPERTY(VisibleAnywhere, BlueprintReadOnly)
	TObjectPtr<UStaticMeshComponent> StaticMeshComponent;
}
```

```cpp
// NsActor.cpp
//...
ANsActor::ANsActor()
{
	CapsuleComponent = CreateDefaultSubobject<UCapsuleComponent>("CapsuleComponent");
	SetRootComponent(CapsuleComponent);
	
	StaticMeshComponent = CreateDefaultSubobject<UStaticMeshComponent>("StaticMeshComponent");
	StaticMeshComponent->SetupAttachment(CapsuleComponent);
}
```

A couple of notes on why I set this up the way I did:
* Components should always be `VisibleAnywhere`, not `EditAnywhere`. This is because the `EditAnywhere` would make the **pointer to the component** editable, not the component itself, which is almost never desirable and can cause issues with serialization that can permanently corrupt an Actor blueprint. **You will still be able to edit the members of the component in the details panel in the editor**.
* Same idea with `BlueprintReadOnly`, we don't want blueprints attempting to swap out our component for another one. We just want them to access their members (which again—they can read/write. Just not the actual pointer to the component itself)
* I used `TObjectPtr<T>` instead of `T*` for the pointer as in 5.4+ [Incremental Garbage Collection](https://dev.epicgames.com/documentation/en-us/unreal-engine/incremental-garbage-collection-in-unreal-engine) requires the use of `TObjectPtr`. `TObjectPtr` is for the most part a drop-in replacement for raw ptr members for `UObject`s
* The `TEXT()` macro is no longer required in the `CreateDefaultSubobject` (often called CDSO) call.
# Components shouldn't own other components
Unfortunately, components' lifetimes are closely tied to being owned by an Actor, not another component. This means that **components shouldn't have authoritative "owning" strong references to other components** nor should they instantiate components via CDSO in their constructors.

Components owning other component will lead to transforms not working correctly and serialization often having issues. It's just half-baked and not supported as of writing.

You're still able to create a hierarchy of attached components, but an actor has to own them all, leading to clunky APIs. Sadly, there is no way around this.
# Components can exist in a world without an actor for some reason
This pattern is especially prevalent with `UAudioComponent` and `UNiagaraComponent` where they will just be spawned dangling, not owned by any actor in the world. I wouldn't recommend following this pattern as it completely breaks expectations as to what a component is in most programming theories, but I think it's important to know that it's sometimes used in Unreal.
# You can add components at runtime to an actor
The following function adds a component of a given class to an actor at runtime. 

We optionally flag it as transient to prevent it from being serialized (as we usually want to add runtime instance components only for runtime) and non-transactional to exclude it from the editor's undo system (saves some performance and undo on a programmatically-added runtime-only component is rare).

```cpp
/** Adds a transient ActorComponent at runtime to an Actor. */
UFUNCTION(BlueprintCallable, meta = (DeterminesOutputType=ComponentClass))
static UActorComponent* AddComponent(AActor* Actor, TSubclassOf<UActorComponent> ComponentClass, bool bTransient = true);
{
        if (!IsValid(Actor) || !IsValid(ComponentClass))
                return nullptr;

        EObjectFlags Flags = RF_NoFlags;
        if (bTransient)
        {
                Flags |= RF_Transient;
                Flags &= ~RF_Transactional;
        }

        UActorComponent* NewComponent = NewObject<UActorComponent>(Actor, ComponentClass, NAME_None, Flags);
        if (!IsValid(NewComponent))
                return nullptr;

        NewComponent->CreationMethod = EComponentCreationMethod::Instance;
        NewComponent->RegisterComponent();
        Actor->AddInstanceComponent(NewComponent);
        
        return NewComponent;
}
```

# Child Actor Component is a piece of shit
**Don't use it for anything that needs to be serialized** such as in an Actor blueprint. 

This is sadly the most obvious usecase for this component, but it has several bugs with serialization to the point where in one of the Unreal communities that I'm in being "burned by the CAC" is a rite of passage.

That being said, CAC is fine when used in a completely transient context, such as adding one at runtime to manage the lifetime of an actor you're attaching. Just make sure to mark the strong reference holding it as `UPROPERTY(Transient)` so it doesn't get serialized. 

But at this point, you could just use a normal actor without the component.