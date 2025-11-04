---
tags:
  - Unreal
  - Programming
title: Live Coding is Dangerous by Default
---
> [!warning] This article is up-to-date as of Unreal Engine 5.6.1
> Live Coding with Reinstancing enabled is dangerous to use and will be for the foreseeable future. Failure to exercise caution can lead to permanently corrupted Blueprint assets. **Please read the rest of the article.**

In the Unreal education community, it's common practice to introduce newcomers to the engine with how to compile their code exclusively using Live Coding with the Editor running. **This is extremely unsafe when presented as the default option for adding code to an Unreal Engine project** and completely subverts the standard workflow of adding code to an application in C++.

If you're coming from another game e:w
ngine such as **Unity** or **Godot**, the workflow of keeping the editor open while compiling runtime script changes is likely familiar. However, in Unreal Engine this workflow is only possible through hacky workarounds known as **Live Coding**, **Reinstancing**, and **Hot Reload**. 

This article will provide a comprehensive summary of the safe workflow when working with C++ in an Unreal project as well as context and explanations as to why Live Coding is not the same as Unity or Godot hot reloading assemblies for managed languages C#/GDScript.

# Unreal's editor is different than other engines' editors
In other popular game engines available to the general public, the editor is an unchanging binary that hosts your assets. When you want to work on your project, you launch the editor and keep it open for the duration of your work session while you add, remove, and modify scripts in a managed language that runs in a virtual machine like C# where hot reloading is well-supported.

**This is not the case with Unreal Engine when you're working with C++**. The editor is a binary application that **you compile** (just your game/plugin modules on an Epic Games Launcher build or the whole thing if building from source).

Without going into too much depth in this article, adding C++ code to Unreal is like modifying the source code of Unity or Godot's editor, **not** adding a script to a separate, sandboxed environment that the editor is providing a view into. 

This relationship is important to keep in mind going forward.

# Live Coding can permanently corrupt your blueprint assets if you're not careful
**Reinstancing** is what makes new reflected types and properties exposed via `UPROPERTY`, `UCLASS`, `USTRUCT`, and `UINTERFACE` show up in a blueprint asset when you compile using Live Coding with the editor open. 

This works through a complex system of ferrying reflected data into a new, temporary type. Hence why you'll often see `LIVECODING` prefixes/suffixes in your variables and types when using this feature.

The problem with Reinstancing is that—to put it bluntly—it's unreliable. Not just due to the complexity of the system itself, but also due to the fact the uassets are serialized to binary. This makes their data very hard to diff, reconcile, and manipulate. 

The result is that assets with types and properties that have been affected by reinstancing have a high propensity to have invalid or temporary data serialized into them if Live Coding with Reinstancing has been involved at any point before saving.

What makes this corruption insidious and hard to track down is that the symptoms are almost never immediately obvious. Common symptoms include:
* Actor components that are impossible to remove, display a blank details panel, and are internally `nullptr`, often causing exceptions in gameplay code that reasonably expects the component to exist
* Properties that either cannot be updated or do not save their updates when the blueprint is compiled/saved
* Transforms and attachment hierarchy not working properly in the case of scene components

Live Coding will succeed but often silently leave affected blueprint assets in a corrupted state with zero warnings nor errors. 

Due to how difficult it is to notice this corruption immediately, it's unfortunately very common that blueprint designers may submit corrupted assets to source control, making it difficult to impossible to find a clean revision to revert to. The only solution in many cases is to simply create a brand new blueprint which can be stressful and time-consuming for blueprint designers that lean heavily on visual scripting and write expansive core logic in blueprints.

# Making Live Coding safe
Live Coding can be made safe to use by disabling its asset-corrupting functionality entirely: **Reinstancing**.

Simply disable **Enable Reinstancing** in *Edit* > *Editor Preferences* > *Live Coding*. This will completely disable Live Coding's ability to add or remove reflected types and members.

> [!danger] DO NOT DISABLE LIVE CODING COMPLETELY
> In the Live Coding settings, there is another setting called *Enable Live Coding*. Do not uncheck this under any circumstance. Disabling Live Coding silently enables **Hot Reload** which is the previous implementation from 4.x that has all the corruption problems of Live Coding with Reinstancing enabled while being even less reliable.

## What Live Coding should be used for
* Editing existing functions, including their parameters as long as they're **not reflected** (no `UFUNCTION` specifier)
* Adding new **non-reflected**, **non-virtual** functions (no `UFUNCTION` specifier or `virtual` keyword)
* Adding new static variables
* Adding new **non-reflected** classes and structs (no `UCLASS`/`USTRUCT` specifier)
In any case, Live Coding should not be used for the following purposes:

## What Live Coding should never be used for
* Adding new **virtual** or **reflected** functions
* Editing constructors
* Adding or removing **reflected** classes and structs (`UCLASS`/`USTRUCT`)
* Adding new members

You will no longer be able to hotpatch `UCLASS`/`USTRUCT`/`UINTERFACE`/`UFUNCTION`/`UPROPERTY` members in and out. Please read below for the recommended workflow when working with C++ in Unreal Engine.
# Recommended C++ Workflow
When adding and removing reflected classes, structs, interfaces, functions, and properties to your project it is imperative that you build the editor from your IDE **without the editor running**. Remember, **you're building an editor binary**, not adding code to a sandbox as in other engines. **Adding code to a running application is not normal for C++**. 

One way to visualize the way adding C++ code to an Unreal project is that the editor is like a game itself and you're *modding the game*, not just adding extra data.
## Building and launching the editor
For C++ developers, launching the editor should be done through your IDE, not through the .uproject file nor through the Epic Games Launcher. **You are developing the editor itself**, not just dumping scripts into it.
### Rider 
1. Right click your .uproject and open with Rider or use *File* > *Open...* 
2. Click the green bug button in the toolbar (*Alt* + *F5* by default)

### Visual Studio
1. Right click your .uproject and click *Generate Visual Studio Project Files*
2. Open the generated .sln file with Visual Studio
3. Click the large "Local Windows Debugger" button in the toolbar (*F5* by default)

## Adding types, functions, and properties
When adding reflected types and properties, **you must close the editor and build it from your IDE**. Both Rider and Visual Studio will handle the process of closing, building, and reopening automatically when you click the relevant debug icon or hotkey (listed above)

**It is not recommended to use the editor's built-in *New C++ Class* wizard** to make the editor add code to itself. By default, this menu attempts to Live Code new reflected types, which is not recommended as discussed earlier.

**Rider** provides Unreal new file templates in *Right Click* > *Add* > *Unreal Engine Class*.

**Visual Studio** does not have Unreal file templates built-in, but a [plugin](https://marketplace.visualstudio.com/items?itemName=Fiquegnima-productions.UnrealWiz) exists that provides similar functionality to Rider.

If you still want to use the in-editor *New C++ Class* wizard, it is recommended to turn *Automatically Compile Newly Added C++ Classes* off in *Edit* > *Editor Preferences* > *Miscellaneous*, as without Reinstancing this will always fail and require you to build from your IDE anyways.
## Debugging
Launching with the editor with a debugger attached will allow your IDE to catch breakpoints, including exception breakpoints. This way, when you hit a `nullptr` exception or fail a `check`/`ensure`/`verify` expression you'll be brought straight to the code where the problem happened with a full callstack and the ability to inspect the value of the relevant variables in-memory, allowing you to diagnose the problem immediately in most cases instead of logging and guessing.

By contrast launching without a debugger will leave you with a vague gesture to what the problem might be via the Crash Reporter, which is intended for your designers and artists to hand to you when their immutable binary build of the editor crashes.

> [!info]
> Debugging will only work reliably if you are building your editor in the  `DebugGame Editor` configuration with the Debug Symbols downloaded in the install options of your engine version in the Epic Games Launcher.
> 
> If you're on a source build, you simply need to build in `Debug Editor` or `DebugGame Editor`, no extra symbols download required. 
> 
> `Debug Editor` allows you to place breakpoints and debug reliably inside engine code, while `DebugGame Editor` only allows you to reliably debug your project's code. `Debug Editor` is only available on a source build as Epic Games Launcher (EGL) builds cannot build the engine.

# Conclusion
Unreal’s C++ build process is not the same as Unity or Godot’s managed scripting environments and trying to treat it that way leads to corrupted assets as a result of Epic's attempt to contort a statically compiled language like C++ into a scripting language. The editor isn’t just an immutable application showing you a view into a sandbox with scripts. It’s the actual application you’re incrementally rebuilding every time you compile. In Unreal, **our job as C++ developers is to create a modded editor that designers can use to build a video game**.

When used correctly, Live Coding is great for quick iteration on non-reflected logic and small function edits—especially gameplay math and Slate UI code—without waiting for a full build. However, once you start modifying reflected members, constructors, or adding and removing types, you’ve crossed into code that cannot be safely hotpatched and requires a new rebuild of the editor from your IDE.