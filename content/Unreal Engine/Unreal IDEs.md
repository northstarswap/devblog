---
title: IDEs for Unreal Development
tags:
  - unreal
  - programming
  - tools
---
When it comes to IDEs for Unreal Engine, there are two practical choices; JetBrains Rider and Microsoft Visual Studio 2022. 

Technically, you can use any text editor to edit code files such as Visual Studio Code or Notepad++, but this article is covering IDEs that provide a suite of features to make development more convenient—of which there are currently two that are generally considered "fully supported."

Regardless of which IDE you choose, please read [this article](Live%20Coding.md) for guidance as to how to compile your project properly (generally not Live Coding with the editor open)
# Visual Studio 2022/2026
Visual Studio is the IDE that most beginners will start out using, as it is popular among Unreal educators for demonstration purposes and many people confuse it for being free (if you have more than 5 people in your team, [it isn't](https://visualstudio.microsoft.com/vs/pricing/?tab=paid-subscriptions))
## Intellisense and code completion
Visual Studio **does not fully understand Unreal projects**, despite many promoting its use in Unreal game development. Unreal's build system does a lot of code generation and substituting in includes at paths that are not known to the IDE. This causes Visual Studio to hallucinate errors in the error list and red squiggly lines that **do not prevent you from building**. 

Epic's own documentation instructs you to [turn off the Error List window](https://dev.epicgames.com/documentation/en-us/unreal-engine/setting-up-visual-studio-development-environment-for-cplusplus-projects-in-unreal-engine#turningofftheerrorlistwindow) if you're using vanilla Visual Studio, and you should disable red squigglies as well as they are completely unreliable and more often than not errors caused by Visual Studio's own lack of support for parsing Unreal's includes and macros. Code completion for engine types will often not work for the same reason.

The workaround to having no reliable predicted errors is simple—just build the project. Only rely on the build output to diagnose issues and ignore the more-often-then-note hallucinated output of the VS error list and squigglies.
### Fixing Visual Studio's code completion with ReSharper
[ReSharper](https://www.jetbrains.com/resharper/) is a commercial product made by JetBrains that replaces Intellisense and Search with a generally superior alternative. ReSharper *does* support and understand Unreal projects and fixes the issues with hallucinated errors and squigglies as well as providing reliable code completion.
## Creating new Unreal project files
Visual Studio has an issue with Unreal's generated .sln where when viewing your project from the solution view (aka via filters) instead of the directory view, your newly-added files will be added to `Intermediate/` instead of `Source/`. This will silently cause them to be missing from your builds. The solution to this is to either ensure you're using the directory view or create new files within your filesystem manually via your file explorer or terminal.

After creating new files, you must manually run *Generate Visual Studio Project Files* by right clicking your .uproject so that they can be seen by Visual Studio.

# JetBrains Rider
Rider is an IDE that integrates ReSharper directly and thus fully supports Unreal's project structure. Rider is able to open .uproject files directly without running *Generate Visual Studio Project files*. 

**You do not have to regenerate project files when adding or modifying new files, modules, or plugins**, Rider will quietly update in the background whenever you perform these operations.

**Rider is free for students and noncommercial use**. The following is not legal advice and I am not a lawyer; but in my opinion if you're a solo dev that hasn't made a penny on your project yet, then you can decide to make your project commercial or noncommercial at any point just by thinking it. It's up to you whether you believe you qualify for the noncommercial license and I am not involved in that decision.
## Important note about compilers
If you're compiling on windows, that usually means you're using MSVC (for the time being—until Verse comes around and mandates the use of Gamer Clang). MSVC is part of Visual Studio meaning you need to have Visual Studio installed in order to use Rider for Unreal projects. 

To be clear, neither Visual Studio nor Rider actually handle compilation. They are practically just fancy text editors. Both of them invoke Unreal Build Tool which runs UHT and MSVC (or Clang on Linux/MacOS) to compile your project. MSVC just happens to be bundled with Visual Studio.

What this means in terms of licensing is that **if your team has more than 5 people on it, you need to buy a Visual Studio license to use MSVC to compile your project**, regardless of your JetBrains license. If you're at the point where there's more than 5 people on your team however, you probably aren't that concerned about a handful of IDE licenses/subscriptions.

# Conclusion
**Visual Studio** is familiar to those coming off of online Unreal courses and YouTube tutorials, but requires disabling error reporting or buying ReSharper in order to achieve reliable code completion and error detection. It also requires manual project file regeneration for adding new files.

**Rider** provides superior out-of-the-box support for Unreal's project structure, eliminates the need for manual project file regeneration and offers more reliable code completion and error detection without additional plugins. It's free for students and noncommercial use, though you still need Visual Studio installed on Windows to access MSVC in order to compile your project.

Regardless of IDE choice, an IDE is ultimately just a text editor that invokes MSVC via Unreal Build Tool. Everything that applies when working on an Unreal project to one applies equally to all the others.

In my opinion, **Rider is generally the superior IDE for working with Unreal projects** due to its better support and quality-of-life features.