---
title: "Easily Add Custom 'Project Settings' to Unreal Engine (.INI)"
date: 2021-11-11
categories: 
  - "cpp"
tags: 
  - "cpp"
  - "quality-of-life"
  - "tips-tricks"
  - "editor"
coverImage: "blog_header_developersettings_4.jpg"
---

You might be placing all your settings and tweakable options in Blueprints or even hard-coded in C++. Unreal Engine does have the option to easily add more configuration settings in the INI config file system using the Developer Settings class. You are probably familiar with the existence of these configuration INI files already. DefaultGame.ini, DefaultEngine.ini, etc. are built using this class and the Unreal Editor's Project Settings and Editor Settings use this system.

New in **Unreal Engine 5** is **_DeveloperSettingsBackedByCVars_** which adds easy binding with Console Variables (CVars) and project/editor settings. I'm explaining this new feature at the bottom of this post.

This system requires some (basic) [C++](https://www.tomlooman.com/unreal-engine-cpp-tutorials/) to define the variables, so even without programming experience, it's relatively easy to use.

## Setting up Developer Settings & Configuration Files

```
// Example of configuration file content. These files are located in MyProject/Config/*.ini

[/Script/ActionRoguelike.SSaveGameSettings]
SaveSlotName=SaveSlot03
DummyTablePath=/Game/ActionRoguelike/Monsters/DT_Monsters.DT_Monsters
```

By deriving a new C++ class from _UDeveloperSettings_ you can easily add your own. The \[CategoryName\] will be your Project + ClassName: **\[/Script/ActionRoguelike.SaveGameSettings\]** in the case of my open-source [Action Roguelike GitHub](https://github.com/tomlooman/ActionRoguelike) project.

[Configuration Files](https://dev.epicgames.com/documentation/en-us/unreal-engine/configuration-files-in-unreal-engine) use key-value pairs **Key=Value** and support file paths and even arrays. We'll be filling an FString and asset path to assign a DataTable via the INI file.

_DeveloperSettings_ is a Module. Creating a UDeveloperSettings derived class will add this module to your .uproject automatically. If it doesn't or you want all your modules in the _.Build.cs_ file then you should add DeveloperSettings manually.

## Defining Developer Settings in C++

Below you'll find an example from the sample project, make sure you expose it to the editor as well (_EditAnywhere_) so it shows up in your Project Settings automatically (see screenshot below).

```
UCLASS(Config=Game, defaultconfig, meta = (DisplayName="Save Game Settings")) // Give it a better looking name in UI
class ACTIONROGUELIKE_API USSaveGameSettings : public UDeveloperSettings
{
	GENERATED_BODY()

public:
	/* Default slot name if UI doesn't specify any */ 
	UPROPERTY(Config, EditAnywhere, BlueprintReadOnly, Category = "General")
	FString SaveSlotName;
	
	/* Soft path will be converted to content reference before use */
	UPROPERTY(Config, EditAnywhere, BlueprintReadOnly, Category = "General", AdvancedDisplay)
	TSoftObjectPtr<UDataTable> DummyTablePath;

	USSaveGameSettings();
};
```

- **Config** \- Exposes the variable to the INI file specified in the UCLASS in the top (Game = DefaultGame.ini)

- **EditAnywhere** \- Exposes it to the Project Settings window.

- **BlueprintReadOnly** \- Exposes variables to be accessed in Blueprint Graph via the "GetClassDefaults" node.

- **defaultconfig** \- _"Save object config only to Default INIs, never to local INIs." (local INIs are in your MyProject/Saved/Config/... folder)_

- **Config=Game** - Store in DefaultGame.ini, other examples include Engine, Input.

![Unreal Editor Project Settings with the new custom settings.](images/ue_projectsettings_customconfig-900x312.jpg)

To access the developer settings in C++ we use the CDO ([Class Default Object](https://dev.epicgames.com/documentation/en-us/unreal-engine/objects-in-unreal-engine)) as that is already automatically instanced for us and accessed using GetDefault<T>();

```
void USSaveGameSubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
	Super::Initialize(Collection);

	const USSaveGameSettings* SGSettings = GetDefault<USSaveGameSettings>(); // Access via CDO
	// Access defaults from DefaultGame.ini
	SlotName = SGSettings->SaveSlotName;

	// Make sure it's loaded into memory .Get() only resolves if already loaded previously elsewhere in code
	UDataTable* DummyTable = SGSettings->DummyTablePath.LoadSynchronous();
}
```

We can't store direct pointers to content files, but we can use soft asset paths and resolve them in code once we need them. Make sure you actually load in the asset manually as otherwise it may or may not sit in memory yet (eg. when you already opened the asset once in your current editor session).

The neat thing about configuration files is that the values are stored as plain text and not compiled to binary, unlike C++ and Blueprint. They can be changed easily even after you already packaged your game.

## Developer Settings Blueprint Access

Getting read-only access to the configuration settings is very easy using the GetClassDefaults node. Make sure you mark your variables BlueprintReadOnly for them to show up.

![blueprint node Get Class Defaults to access Developer Settings.](images/ue_developersettings_blueprintaccess.jpg)

## Game User Settings

If you want to store player configurable settings there is a different class available: [GameUserSettings](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/Engine/GameFramework/UGameUserSettings?application_version=5.3). This class already includes a bunch of settings including graphical options. Here you might want to add mouse sensitivity, FOV, etc.

## DeveloperSettingsBackedByCVars

New in **Unreal Engine 5.0** is **_DeveloperSettingsBackedByCVars_** which adds easy binding with Console Variables (CVars) and project/editor settings.

This new class lets us bind _console variables_ to project settings and easily change and store defaults either per developer or project-wide. In practice this means we can define default values in the INI files and at runtime change them using console variables.

An example of this can be found in the [Lyra Starter Game](https://www.unrealengine.com/marketplace/en-US/product/lyra) which was released with UE5.0. The _LyraWeaponsDebugSettings_ has several properties for debugging trace hits. By using _DeveloperSettingsBackedByCVars_ and the ConsoleVariable meta-specifier you can bind the variables together.

```
// Should we do debug drawing for bullet traces (if above zero, sets how long (in seconds)
UPROPERTY(config, EditAnywhere, Category=General, meta=(ConsoleVariable="lyra.Weapon.DrawBulletTraceDuration")
float DrawBulletTraceDuration;
```

The Console Variable is still defined elsewhere. In the example, you can find the CVar inside _LyraGameplayAbility\_RangedWeapon.cpp_:

```
namespace LyraConsoleVariables
{
	static float DrawBulletTracesDuration = 0.0f;
	static FAutoConsoleVariableRef CVarDrawBulletTraceDuraton(
		TEXT("lyra.Weapon.DrawBulletTraceDuration"),
		DrawBulletTracesDuration,
		TEXT("Should we do debug drawing for bullet traces (if above zero, sets how long (in seconds))"),
		ECVF_Default);
}
```

Then access the CVar by using the Namespace and the static float, not the variable in the Settings file.

```
if (LyraConsoleVariables::DrawBulletTracesDuration > 0.0f)
{
	static float DebugThickness = 1.0f;
	DrawDebugLine(GetWorld(), StartTrace, EndTrace, FColor::Red, false, LyraConsoleVariables::DrawBulletTracesDuration, 0, DebugThickness);
}
```

From testing this out in the Action Roguelike project, it's important to **manually apply the CVARs from INI** during `PostInitProperties()` with the example below or check out the [RogueDeveloperSettings](https://github.com/tomlooman/ActionRoguelike/blob/master/Source/ActionRoguelike/Development/RogueDeveloperSettings.cpp).

```
void URogueDeveloperSettings::PostInitProperties()
{
#if WITH_EDITOR
	if (IsTemplate())
	{
		// We want the .ini file to have precedence over the CVar constructor, so we apply the ini to the CVar before following the regular UDeveloperSettingsBackedByCVars flow
		UE::ConfigUtilities::ApplyCVarSettingsFromIni(TEXT("/Script/ActionRoguelike.RogueDeveloperSettings"), *GEditorPerProjectIni, ECVF_SetByProjectSetting);
	}
#endif // WITH_EDITOR

	Super::PostInitProperties();
}
```

This new addition can be quite nice to easily set defaults in your project settings that are either per-user or per project. The shown example specifies `UCLASS(config=EditorPerProjectUserSettings)` to have this stored per developer.

## Closing

For more details and examples we're using Developer Settings to configure a [SaveGame system](https://www.tomlooman.com/unreal-engine-cpp-save-system/) in my [**Unreal Engine C++ Pro Course**](https://courses.tomlooman.com/p/unrealengine-cpp?coupon_code=COMMUNITY15). The project source is available to browse on [GitHub](https://github.com/tomlooman/ActionRoguelike). Make sure you check it out!

[Follow me on Twitter](https://www.tomlooman.com/unreal-engine-optimal-graphics-settings/) for more Unreal-related content or have a look at my other [C++ Tutorials](https://www.tomlooman.com/unreal-engine-cpp-tutorials/) and tricks such as an easy way to [Auto-detect optimal graphics settings.](https://www.tomlooman.com/auto-detect-graphics-settings-ue4/)
