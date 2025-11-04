---
tags:
  - Unreal
  - Programming
  - Advanced
title: How to permanently remove Hot Reload and Reinstancing from an Unreal source build
---
> [!DANGER]
> The methods involved in this article involve modifying engine source to achieve complete removal of Hot Reload and permanent disablement of Reinstancing. It assumes significant experience with Unreal, C++, and modifying the engine. I am not responsible for breakages or your teammates complaining that they have to properly build from their IDE instead of letting Reinstancing spin the wheel of corruption.

> [!DANGER]
> If you're not on a Source Build (i.e. your Unreal Engine installation came from the Epic Games Launcher or from the Linux Binaries webpage), do not attempt this method. Without a full souce build, modifying Engine files is not supported and there's no guarantee that you will be able to compile these changes, leaving your Engine install in a broken state where you'll have to Verify and potentially redownload.

> [!WARNING]
> These modifications will trigger a full Engine rebuild of ~4700 actions. Be prepared to be patient and don't drop these changes onto your team members unexpectedly.

> [!Important]
> You will still be able to use Live Coding with this method. Live Coding without Reinstancing can be used for the following:
> 
> * Editing function bodies
> * Adding new non-virtual non-reflected functions
> * Editing non-reflected function signatures
> * Adding static variables
> * Adding new non-reflected classes and structs
>   
>  Users will be able to turn Live Coding on/off through their Editor Preferences without the risk of enabling Hot Reload through a checkbox, but the Reinstancing option will be eliminated entirely.

This is the most comprehensive method for preventing users from accessing Hot Reload and Reinstancing. It involves modifying UHT-injected macro definitions to prevent Hot Reload from being compiled in at all and as non-invasively-as-possible disabling Reinstancing while leaving Live Coding intact. 

This will completely prevent Live Coding from permanently corrupting assets on all of your team members' machines, but also prevent them from compiling reflected changes with the editor open (as it should be). Please remember to inform any members of your team of these modifications, as they break the convention of what most developers expect when working on an Unreal project.

If you're here, you likely already know why you would want to perform this modification. If not, [this article](Live%20Coding) is probably what you're looking for.

This modification is even more important if you have team members that use Linux or MacOS, as these platforms do not support Live Coding (it's Windows-only) and thus force Hot Reload on.
# Step 1: Removing the `WITH_HOT_RELOAD` preprocessor define to compile without Hot Reload

In `Engine/Source/Runtime/Core/Public/Misc/Build.h`, find the following section:
```c++
/**
* Whether we support hot-reload. Currently requires a non-monolithic build and non-shipping configuration.
*/
#ifndef WITH_HOT_RELOAD
	#define WITH_HOT_RELOAD (!IS_MONOLITHIC && !UE_BUILD_SHIPPING && !UE_BUILD_TEST && !UE_GAME && !UE_SERVER)
#endif
```

Comment out or delete the existing line and replace with `#define WITH_HOT_RELOAD 0`
```c++
/**
* Whether we support hot-reload. Currently requires a non-monolithic build and non-shipping configuration.
*/
#ifndef WITH_HOT_RELOAD
	// #define WITH_HOT_RELOAD (!IS_MONOLITHIC && !UE_BUILD_SHIPPING && !UE_BUILD_TEST && !UE_GAME && !UE_SERVER)
#define WITH_HOT_RELOAD 0
#endif
```

This is not enough to disable compiling support for Hot Reload. UHT's codegen has a flag in `UhtCompilerDirective` to ignore preprocessor directives when parsing header files. We need to remove the exemption for `WithHotReload` in `Engine/Source/Programs/Shared/EpicGames.UHT/Parsers/UhtHeaderFileParser.cs`:
```c#
		/// <summary>
		/// The following flags are always ignored when keywords test for allowed conditional blocks
		/// </summary>
		AllowedCheckIgnoredFlags = CPPBlock | NotCPPBlock | ZeroBlock | OneBlock | WithHotReload,
```

Remove `WithHotReload` from the flag:
```c#
		/// <summary>
		/// The following flags are always ignored when keywords test for allowed conditional blocks
		/// </summary>
		AllowedCheckIgnoredFlags = CPPBlock | NotCPPBlock | ZeroBlock | OneBlock,
```

Hot reload will now be completely omitted from your engine builds.
# Permanently disable Reinstancing

In `Engine/Source/Developer/Windows/LiveCoding/LiveCodingSettings.h`, find the following `UPROPERTY`s
```c++
	UPROPERTY(config, EditAnywhere, Category=General, Meta=(EditCondition="bEnabled"))
	bool bEnableReinstancing;
	
	UPROPERTY(config, EditAnywhere, Category=General, Meta=(EditCondition="bEnabled", DisplayName="Automatically Compile Newly Added C++ Classes"))
	bool bAutomaticallyCompileNewClasses;
```

Comment them out or delete them. I recommend commenting them out and leaving a note as to why they were removed:
```c++
	// Reinstancing can cause asset corruption and unexpected behavior. Do not use this.
	// UPROPERTY(config, EditAnywhere, Category=General, Meta=(EditCondition="bEnabled"))
	// bool bEnableReinstancing;

	// Without Reinstancing we cannot compile new classes.
	// UPROPERTY(config, EditAnywhere, Category=General, Meta=(EditCondition="bEnabled", DisplayName="Automatically Compile Newly Added C++ Classes"))
	// bool bAutomaticallyCompileNewClasses;
```

`bAutomaticallyCompileNewClasses` is used in the `Tools > New C++ Class...` New Class Wizard. Without Reinstancing it's always going to fail, so remove the option here. Later,  we'll add a dialogue box to warn users who are used to this workflow about these changes.

In `Engine/Source/Developer/Windows/LiveCoding/Private/LiveCodingModule.cpp`, find the following lines:
```c++
bool FLiveCodingModule::IsReinstancingEnabled() const
{
#if WITH_EDITOR
	return Settings->bEnableReinstancing;
#else
	return false;
#endif
}

bool FLiveCodingModule::AutomaticallyCompileNewClasses() const
{
	return Settings->bAutomaticallyCompileNewClasses;
}
```

Now they should always return `false`:
```c++
bool FLiveCodingModule::IsReinstancingEnabled() const
{
	return false;
}

bool FLiveCodingModule::AutomaticallyCompileNewClasses() const
{
	return false;
}
```

## Adding a Warning for Developers that use the New Class Wizard

Some developers prefer to use the New Class wizard to create class files as it includes file templates for common Unreal classes where vanilla Visual Studio provides none (Rider does provide templates in *Add* > *Unreal Engine Class...* and the [Unreal Wizard plugin](https://marketplace.visualstudio.com/items?itemName=Fiquegnima-productions.UnrealWiz) provides them for VS). 

The New Class wizard still works with these changes and is able to add the files, however without Reinstancing the it'll never be able to hotpatch the classes that it adds.

To add a dialogue to warn them and offer to stop or continue, add the following in `Engine/Source/Editor/GameProjectGeneration/Private/GameProjectUtils.cpp` at the top of `GameProjectUtils::OpenAddToProjectDialog`:
```c++
void GameProjectUtils::OpenAddToProjectDialog(const FAddToProjectConfig& Config, EClassDomain InDomain)
{
	EAppReturnType::Type NoCompileMsgResult = FMessageDialog::Open(EAppMsgCategory::Error, EAppMsgType::YesNo, LOCTEXT("NewClassDialogCannotCompileForYou", "Reinstancing is disabled in this version of the engine. New source files can be added to the filesystem from the New Class menu however new classes will not be compiled while the editor is running.\n\nIf you still want to use the New Class menu, close the Editor once the classes are added to the filesystem and compile from your IDE.\n\nWould you like to continue?"));
	if (NoCompileMsgResult == EAppReturnType::No)
		return;
	
	//...
}
```

# The Result
After compiling, to confirm that Hot Reload is completely gone you can go into Editor Settings and untick `Enable Live Coding`. When you click the hotpatch button in the bottom right of the Editor we should see the following warning in the log:
```
Warning: RebindPackages not possible (hot reload not supported)
```

As for the New Classes menu, before the New Classes dialogue appears users will be greeted with an informational dialogue box that explains the changes and offers to stop or continue.