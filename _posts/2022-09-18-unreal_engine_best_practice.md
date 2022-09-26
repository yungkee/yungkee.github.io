---
layout: post
title: UE4 Best Practices
modified:
categories: 
excerpt: UnrealEngine
tags: [Notes]
image:
  feature: note.jpg
  thumb: thumb1.jpg
date: 2022-08-08T02:54:48+05:30
---

## UE4 Best Practices

 For the past couple of years working in unreal engine i've been collecting what i consider to be the best practices of how to work in it uh and i made a document of this which i figure might be good to share

TODO fix links:

Derived Data Cache (DDC)
This cache allows sharing of the cost to compile shaders and other operations. It’s a must when working in a team to prevent hours of waiting when working locally. It doesn’t make sense to use when working remotely because it would take longer to download than to generate, however you can specify the location of a local and shared DDC to prevent the warning messages from popping up.

Here’s Epic’s documentation:
Derived Data Cache
Team Setup (while in office)
The project should be set up to use a shared cache. An example setup used in Slingshot (DefaultEngine.ini):

[DerivedDataBackendGraph]
MinimumDaysToKeepFile=7
Root=(Type=KeyLength, Length=120, Inner=AsyncPut)
AsyncPut=(Type=AsyncPut, Inner=Hierarchy)
Hierarchy=(Type=Hierarchical, Inner=Boot, Inner=Pak, Inner=EnginePak, Inner=Local, Inner=Shared)
Boot=(Type=Boot, Filename="%GAMEDIR%DerivedDataCache/Boot.ddc", MaxCacheSize=512)
Local=(Type=FileSystem, ReadOnly=false, Clean=false, Flush=false, PurgeTransient=true, DeleteUnused=true, UnusedFileAge=34, FoldersToClean=-1, Path=%ENGINEDIR%DerivedDataCache, EnvPathOverride=UE-LocalDataCachePath, EditorOverrideSetting=LocalDerivedDataCache)
Shared=(Type=FileSystem, ReadOnly=false, Clean=false, Flush=false, DeleteUnused=true, UnusedFileAge=10, FoldersToClean=10, MaxFileChecksPerSec=1, Path=\\Slingshot\ddc, EnvPathOverride=UE-SharedDataCachePath, EditorOverrideSetting=SharedDerivedDataCache)
AltShared=(Type=FileSystem, ReadOnly=true, Clean=false, Flush=false, DeleteUnused=true, UnusedFileAge=23, FoldersToClean=10, MaxFileChecksPerSec=1, Path=\\Slingshot\ddc2, EnvPathOverride=UE-SharedDataCachePath2)
Pak=(Type=ReadPak, Filename="%GAMEDIR%DerivedDataCache/DDC.ddp")
EnginePak=(Type=ReadPak, Filename=%ENGINEDIR%DerivedDataCache/DDC.ddp)

Set up a machine on the network so that it has shared folders with everyone read/write access, and set the path in the DefaultEngine.ini as shown.
Locally (remote work)
When working remotely, it doesn’t make sense to try and use a shared DDC, because downloading it through the VPN would be slower than generating it yourself locally. We will disable the shader DDC to prevent duplicating data in the following steps.
By default, the engine will set up a local DDC in Users\username\AppData\Local\UnrealEngine\Common. I prefer having it somewhere easy to find (and delete), but you can leave it there if you don’t mind that location.
Working from home, I suggest using the following environment variables, which specifies the location of the local DDC and has no shared DDC for all projects. You can leave out the local DDC one if you are fine with where it’s currently located.

Disable the warnings in the editor by going into editor preferences, under General - Performance. Disable the option to “enable shared data cache performance notifications”.
Perforce
Perforce
Rider
There is a new IDE that can be used to work in Unreal Engine, Rider!

Rider for Unreal Engine

It has built-in Perforce support, and already has the ReSharper C++ functionality. In my experience it is much faster than Visual Studio and has many Unreal Engine features. Highly recommended and you can get a free trial!
Visual Studio
If you use Visual Studio, changing a few settings is imperative:

Setting Up Visual Studio for Unreal Engine

Using the Perforce plugin for Visual Studio makes it much faster to check out files:

Download Helix Plugin for Visual Studio (P4VS)

Resharper C++ is a plugin for Visual Studio which has Unreal Engine 4 functionality built into it. It makes it much easier to follow Epic’s coding standard, as it highlights many things that do not fit it. Additionally it speeds creating function prototypes, describes macros (UFUNCTION, UPROPERTY), and replaces intellisense.

ReSharper C++ : The Visual Studio Extension for C++ Development
Engineers
Reflection System and C++ Coding Standard
Unreal Engine 4 extends C++ by using a tool called the Unreal Header tool, along with a reflection system. Read this first:
Unreal Property System (Reflection)
Follow Epic’s coding standard, which is found here:
Coding Standard | Unreal Engine Documentation
Resharper C++, which is a Visual Studio plugin, has Unreal Engine 4 features that help you to follow the standard.
Features - ReSharper C++
Perforce makes a Visual Studio plugin. This makes it much easier to check out files automatically while you are working, or use Rider's perforce integration.
Download Helix Plugin for Visual Studio (P4VS) | Perforce
Use braces on separate lines for everything. This allows using breakpoints anywhere.
Where a class is large, use #pragma region and #pragma endregion to separate out functionality as necessary. For each region be sure to use public/private specifiers in case the region is moved.
Group UFUNCTIONS first and UPROPERTIES second after the private/public/protected specifiers.
In the main header file for a class, there should be a minimum of included header files to speed compilation when there are changes. Use ‘class’ to forward define class types that aren’t truly needed in the header file.
Built In Data Types
TArray
TArray: Arrays in Unreal Engine

A TArray is an array that is templated so that it can contain most any type, including UOBJECT based types.

Use a TArray when you care about the ordering of objects.
You can Add/Remove from the array using one function call, where remove will search the array with O(N) to find all elements that are that object and remove it.
You can sort the array using the sort function, and use a predicate if necessary.
You can access elements using [x] where x is the element number.

// Sort descending.
Scores.Sort([](const int& First, const int& Second) {return First > Second; });
TSet
TSet | Unreal Engine 4.27 Documentation

A TSet is a number of homogeneous objects that are referenced by the hash produced by the object itself. By default it only allows ONE of each object.

Use TSet when you don't care about order, and instead care about the speed of finding and removing objects.
Add/Remove uses a hash lookup and is much faster than TArray.
You can create your own hash using GetTypeHash() and HashCombine() if necessary.
You can't access objects using [x], you must use an iterator. The point of the TSet is not to care about order.
If you want to get an element from the TSet, you have to use an iterator; you could conceivably bail after getting the "first" element, which only means that it happens to be the first in the set. This order can change any time.
TMap
TMap | Unreal Engine 4.27 Documentation

A TMap is similar to a TSet, except that the key value, that is to say the value that you use to look up the object, can be specified.

Use TMap when you don't care about order, care about speed in look up, and need to map some kind of key to a value.
For example, you might want to map FName to UDataAsset, looking up data assets by their name.
You could also map a uint32 Hash to an object in a map, and generate the hash manually by what is in the object that uniquely identifies it.
Class Creation
ALWAYS CREATE A BASE CLASS IN C++.
Mark the class as final if it does not have any classes which inherit from it, this will speed up function calls of virtual functions, as the class can be "de-virtualized". final specifier (since C++11)
Constructors can contain NO RUNTIME CODE and should only be used for constructing components and setting default values in the class. Do NOT assign dynamic delegates in the constructor, these are unpredictable.
Disable tick and start with tick enable in the constructor of the class UNLESS YOU ABSOLUTELY HAVE TO USE TICK. Setting start with tick enabled to false will cause the BP to have “Start with Tick Enabled” off by default, which is a great way to prevent having a ton of BP’s ticking that aren’t doing anything.
For actors:
PrimaryActorTick.bCanEverTick = false;
PrimaryActorTick.bStartWithTickEnabled = false;
For components:
PrimaryComponentTick.bCanEverTick =  false;
PrimaryComponentTick.bStartWithTickEnabled = false;
If you have to use tick:
Set the tick interval to the maximum value you can get away with. Unfortunately this is often per frame for smoothly moving things.
PrimaryActorTick.TickInterval = 0.2f;
PrimaryComponentTick.TickInterval = 0.2f;
Enable/disable tick to only tick when required.
SetActorTickEnabled()
SetComponentTickEnabled()
Components and variables created in the C++ base class:
ALWAYS create components in C++. 
ALWAYS SetupAttachment() to connect them in the constructor the way you desire to other components, assuming they are based on a scene component (ActorComponents have no transform and cannot be attached of course). If you want to connect them to a socket in a skeletal mesh, setup attachment allows that as well if you put the name of the socket in the call. Note that this won’t happen until the object is registered (the socketing), so you can’t set relative location or rotation to that socket until BeginPlay() on the socketed object.
If you don't want the object to be attached to anything else during gameplay, it is still a good idea to attach it in the constructor and detach it in BeginPlay() or at some other time. Alternatively you can set it to use absolute location, rotation, and scale.
If you don't attach it to a parent object, you will get weird results occasionally (it will auto attach it to the root for you, etc), so it's best to create predictable behavior by attaching it and detaching it specifically.
Should be protected, components should not be publicly accessible unless there’s some real reason; consider using friend classes.
Give absolutely minimum access necessary to variables and functions from BP to make it clear how they are used. Don’t just make everything BlueprintReadWrite, it will be hard to know what’s being used in BPs if you want to refactor. See the UFUNCTION and UPROPERTY specifiers section.
For components, use:
UPROPERTY(VisibleAnywhere)
    USomeComponent* SomeComponent;
Pointers to other objects or variables that need to be set will be
UPROPERTY(EditDefaultsOnly)
float SomeFloat = 0.3f;
Variables that are set up in the class ALWAYS NEED A DEFAULT VALUE UNLESS TRANSIENT.
float SomeFloat = 0.0f;
int SomeInt = 0;
bool bSomeBool = false;
Keep in mind that variables that are set in an inherited BP will not be accessible in the constructor. They will not be set until BeginPlay(). 
Variables that point to UOBJECTS such as an actor reference, should use UPROPERTY() to prevent garbage collection. Otherwise, the engine can garbage collect the pointer at any time, invalidating the reference. 

Unreal Object Handling | Unreal Engine Documentation

UPROPERTY(Transient)
UMaterialInstanceDynamic* DynamicMaterial;

Don’t spawn components conditionally or delete components in the constructor if you want the editor to visualize them properly. UE4 runs the constructor once to create a class default object and doesn’t run the constructor on actors that have been saved into levels once it has done this. Objects that are selected in the editor, if you have deleted components, will flicker in their description pane.
Polymorphic Components
If you want to create a component in a base class, and then have child classes override that class, it can be done using the object initializer.

Create your new base class.
Change the constructor that was automatically generated to include a FOBjectInitializer argument in the header and C++ file.

AMeleeWeaponActor(const class FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

Use that constructor to create the base version of the component.

AMeleeWeaponActor::AMeleeWeaponActor(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{

    AttackColliderComponent = CreateDefaultSubobject<UShapeComponent>(TEXT("AttackCollider"));
}

Now create your subclass of the actor:

UCLASS()
class WHISPER_API AShovelActor final : public AMeleeWeaponActor 
{
    
    GENERATED_BODY()

public:

    explicit AShovelActor(const FObjectInitializer& ObjectInitializer);
    
};

In the constructor of this child actor, use the initializer to specify the class for the component.

AShovelActor::AShovelActor(const FObjectInitializer& ObjectInitializer) :
    Super(ObjectInitializer.SetDefaultSubobjectClass<UBoxComponent>(TEXT("AttackCollider")))
{
    UBoxComponent* Box = CastChecked<UBoxComponent>(AttackColliderComponent);

    Box->SetupAttachment(GetRootComponent());
    Box->SetCollisionProfileName(Weapon_ProfileName);
    Box->SetGenerateOverlapEvents(false);
    Box->CanCharacterStepUpOn = ECB_No;
}

At this point, the original class will have the shape component, and since a collision box is a subclass of shape, the new class will have a box component. This way you can make multiple subclasses, and each could have a different collision shape (box, capsule, sphere) if you desired.
Good places to look at this in the engine code is ACharacter and ATriggerBase if you want to see examples.
UCLASS
Class Specifiers | Unreal Engine Documentation
HideCategories
UCLASS(HideCategories=(Actor, Physics, Collision, Lighting, Rendering, Replication, Input, LOD, Cooking))

It’s possible to hide categories from the blueprint editor / details pane of a class by specifying the category of the class to hide, or the class name for properties with no category.
UFUNCTION
BlueprintCallable
Allows a function to be called from BP in general. If the function is setting/getting a variable or doesn’t change the underlying class, use the more specific specifiers below.

UFUNCTION(BlueprintCallable)
void TriggerSomething();
BlueprintPure (Implies BlueprintCallable)
Allows a function to be called from BP that doesn’t change the underlying class. If the function is just setting a variable in the underlying class, use BlueprintSetter.

UFUNCTION(BlueprintPure, Category = "Point Cloud")
TArray<FVector> GetPointsInBox(const FBox& WorldSpaceBox) const;
BlueprintGetter (Implies BlueprintPure and BlueprintCallable) and BlueprintSetter (Implies BlueprintCallable)
Example UPROPERTY:

UPROPERTY(EditAnywhere, BlueprintGetter=GetDisablePostProcessBlueprint, BlueprintSetter=SetDisablePostProcessBlueprint, Category = Animation)
uint8 bDisablePostProcessBlueprint:1;

BlueprintGetter is intended for getting a variable from the underlying class:

UFUNCTION(BlueprintGetter)
bool GetDisablePostProcessBlueprint() const { return bDisablePostProcessBlueprint; }

BlueprintSetter is intended for setting a variable in the underlying class:

UFUNCTION(BlueprintSetter)
void SetDisablePostProcessBlueprint(bool bInDisablePostProcess)
{
    if(!bInDisablePostProcess && bDisablePostProcessBlueprint && PostProcessAnimInstance)
    {
        PostProcessAnimInstance->InitializeAnimation();
    }

    bDisablePostProcessBlueprint = bInDisablePostProcess;
}
BlueprintImplementableEvent
Defines a function that is overridden in BP, and has no native implementation. See BlueprintNativeEvent to add a native implementation.

UFUNCTION(BlueprintImplementableEvent)
void PlayerIsHealthy(float Health);

In your class Blueprint, right click and type the name of your function to create an event that is the function override.
BlueprintNativeEvent
Define a function that is intended to be overridden in BP, but also has a native implementation specified with the _Implementation label.

UFUNCTION(BlueprintNativeEvent)
UEnum* GetAnimationStateEnum();
virtual UEnum* GetAnimationStateEnum_Implementation()
{
    return GetAnimationStateEnum_Internal();
}
Passing Structs
In order for these to compile and to show up in Blueprints properly, you must pass the structure as a const reference, as seen below.

// Calls the SetHandPose() function on the animation blueprint.
UFUNCTION(BlueprintNativeEvent)
void SetHandPose(const FPoseSet& Poses, bool bImmediate);
void SetHandPose_Implementation(const FPoseSet& Poses, bool bImmediate);

USTRUCT(BlueprintType)
struct FPoseSet
{
  GENERATED_BODY()

  UPROPERTY(EditAnywhere, BlueprintReadWrite)
  UAnimSequence* FemaleLeftPose;

  UPROPERTY(EditAnywhere, BlueprintReadWrite)
  UAnimSequence* FemaleRightPose;

  UPROPERTY(EditAnywhere, BlueprintReadWrite)
  UAnimSequence* MaleLeftPose;

  UPROPERTY(EditAnywhere, BlueprintReadWrite)
  UAnimSequence* MaleRightPose;
};

Structs as Function Inputs
Use UPARAM(ref):

UFUNCTION(BlueprintCallable, Category = "Example Nodes")
static void HandleTargets(UPARAM(ref) TArray<FVector>& InputLocations, TArray<FVector>& OutputLocations);
CallInEditor
Allows the function to be called in the editor when selecting the object in the level. This can be done while running the game or not. Very useful for testing functionality; a button appears that is the name of the function in the actor’s details pane.
Exec
Allows executing the function from the console.

UPROPERTY
Garbage Collection
Garbage collection traverses through all objects with UPROPERTY’s; those that cannot be reached can be garbage collected at any time. Therefore you should use UPROPERTY() on variables that are references to UOBJECTS.

// Make sure my actor reference isn’t garbage collected
UPROPERTY(Transient)
AActor* ReferenceActor;
Config
Specifies that the variable is loaded from and possibly saved to a config file. The file to save to is defined at the top of the class. Keep in mind that classes that inherit to Blueprints can create a confusing situation, as the config variable is tied to the underlying class name. It’s best to use the Blueprint name if you do this to avoid weird behavior.

UCLASS(Config=Game)
class SLINGSHOT_API ADisplayService : public AActor, public IServiceInterface
{

    UPROPERTY(Config, VisibleAnywhere)
    TSoftClassPtr<AScreenCollider> ScreenClass;
}

In DefaultGame.ini:

[/Script/Slingshot.DisplayService]
ScreenClass="/Script/Slingshot.ScreenCollider"
Transient
Property should not be saved and is zero-filled at load time. This is good for references to actors or things like that which are created during gameplay to prevent garbage collection.

UPROPERTY(Transient)
UAnimInstance* Anim;
Edit Specifiers
EditAnywhere
Set a variable to be edited anywhere; in the BP, in an object in the world, etc. Good for non-pointer values that you want to be accessible from BP. Use this only if necessary, use the more restrictive specifiers below if possible.

UPROPERTY(EditAnywhere, BlueprintReadWrite)
float Something = 0.0f;
EditDefaultsOnly
Set a variable to be edited only from an archetype. This mainly means that you can only set it in the BP editor, not in an instance of an object.
    
UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
int32 QtyToSpawn = 12;
EditFixedSize
This specifier prevents changing the size of an array. It can be used in combination with other edit or visibility specifiers; for example:

UPROPERTY(VisibleDefaultsOnly, EditFixedSize)
TArray<FTransform> RightHandRelativeTransforms;

Once you have set up a variable in this manner, initialize the size of the array in the constructor if the size is known at that point (an arbitrary constant), or initialize in PostLoad() if it's driven by Blueprint. Post load can overwrite the values in it that have been serialized if not guarded.

void ANavTableActor::PostLoad()
{
Super::PostLoad();

    if(RighHandRelativeTransforms.Num() != 4)
{
  RightHandRelativeTransforms.Init(FTransform::Identity, 4);
}
}
EditInstanceOnly
Inverse from EditDefaultsOnly, this variable can only be edited on an object in the world. Useful for properties that rely on other objects in the world, for example, a pointer to an actor in the world.

UPROPERTY(EditInstanceOnly)
AActor* InteractionActor;
Visibility Specifiers
VisibleAnywhere
Indicates that the variable can be seen anywhere, but not edited. Mainly used for components that are spawned in native code but accessible from blueprints. Useful to show spawned components as well in the editor during gameplay.

UPROPERTY(VisibleAnywhere)
UStaticMeshComponent* StaticMeshComponent;
VisibleDefaultsOnly
The property can only be seen in an archetype (Blueprint editor).
VisibleInstanceOnly
The property can only be seen when selecting an instance of the object in the world outliner.
BlueprintReadOnly
Property can be read by blueprints, but not changed. This does not apply to member variables, their members will have unique specifiers. For example:

UPROPERTY(Transient, BlueprintReadOnly)
AActor* Test;

Blueprints will be able to call functions on Test and change member variables within test based on their respective UPROPERTY specifiers, but won’t be able to set Test equal to something.
BlueprintReadWrite
Property can be read and written by blueprints. This allows a Blueprint to set the property directly, for example setting an actor pointer to a new actor.
Meta Properties
These values can be placed within a UPROPERTY(..., meta=(Property=”Something”)) macro. There are many of these, but these are useful.

You can find these in the engine source:
\UnrealEngine\Engine\Source\Runtime\CoreUObject\Public\UObject\ObjectMacros.h
BindWidget
Allows access to widgets created in a widget blueprint in C++. Create a UUserWidget C++ base class, set the parent of your Widget Blueprint to this C++ class, and then bind to the widgets using their name and this meta property.

UPROPERTY(EditAnywhere, meta = (BindWidget))
UTextBlock* ScoreText;
ClampMin, ClampMax
Use to force a range of values in the editor.

UPROPERTY(Category=Setup, EditAnywhere, BlueprintReadOnly, meta=(ClampMin=1, ClampMax=1024))
int32 MapWidth;
DisplayName
Sets a pretty name that is displayed in the editor. I don’t like using this as it makes it hard to find in code.

UPROPERTY(EditAnywhere, meta=(DisplayName="Additional Textures"))
TArray<UTexture*> AdditionTextures;
EditCondition
Defines a boolean that allows editing of the property. This is useful if you want to be able to edit this only if a different property is set to a particular value.

UPROPERTY(BlueprintReadOnly, EditDefaultsOnly, Category = "Destructible", meta = (EditCondition = "bUseVelocityForFractureVariation"))
FFractureData DefaultBreakVariation;
EditConditionHides
If you use this in conjunction with EditCondition above, it will hide the UPROPERTY completely if the edit condition is not met.
MakeEditWidget
Adds a 3d widget to the object when viewed in the scene that appears as a purple diamond. For example, if you place this property on an array of transforms:

UPROPERTY(EditAnywhere, meta = (MakeEditWidget = true))
TArray<FTransform> NavigationPoints;

You can create transforms in the array in the details pane of an object, and see them in the scene. You can’t see them in the Blueprint editor though, so be aware!
Multiline=true
For text and string fields, this allows the property to accept line breaks, which you enter with SHIFT+ENTER.

UPROPERTY(EditAnywhere, BlueprintReadOnly, meta = (MultiLine=true))
FText Description;
TitleProperty

If you have an array of objects, and want to be able to see some property inside the array as the row title in the details panel, you can use this specifier.

UPROPERTY(EditAnywhere, meta = (TitleProperty = "Type"))
TArray<FTimeChefCookType> Types;

USTRUCT(BlueprintType)
struct FTimeChefCookType
{

  GENERATED_BODY()
 
  UPROPERTY(EditAnywhere)
  float Amount = 0.0f;

  UPROPERTY(EditAnywhere)
  EHeatSourceType Type = EHeatSourceType::CoalFire;
 
};
FDirectoryPath - RelativeToGameContentDir, LongPackageName
If you want to store a directory path (useful for editor utilities), you can use this data type and meta properties.
RelativeToGameContentDir - the picker will output a path relative to the game content directory.
Components
Editor Only Components
Components which have transforms in the world (objects based on the scene component, for example an arrow) require their transforms to be updated every frame while moving.
This cost can be removed by making the component editor only and caching it's transform for use outside the editor. For example, if an arrow in the editor represents a fixed relative transform related to a base component, you can cache that relative transform and convert that relative transform to world transform when using it.
To make a component editor only, wrap the component definition #if WITH_EDITORONLY_DATA in the header file.

#if WITH_EDITORONLY_DATA
    UPROPERTY(VisibleAnywhere)
    UArrowComponent* ArrowComponent;
#endif

Wrap where the component is constructed or accessed as well, and use the CreateEditorOnlyDefaultSubobject. Be sure to check if the object exists before referencing it, as in non-editor builds, it will be null.

#if WITH_EDITORONLY_DATA
  ArrowComponent = CreateEditorOnlyDefaultSubobject<UArrowComponent>(TEXT("ArrowComponent"));
if(ArrowComponent)
{
ArrowComponent->SetupAttachment(....
...
#endif

Cache the relative transform of the component; this can be done in many ways, but using PostEditChangeProperty() to set the cache value based on the component value when things are changed in the editor is an easy way to do it.
Then you can get the world transform of the component by multiplying the cached transform with the transform of the component it was attached to. For example:

FTransform WorldTransform = CachedRelativeTransform * GetActorTransform();
Components that Can't Be Changed In Editor
If you create a component with an empty UPROPERTY() specifier, it will exist in the editor, but if you select it in the Blueprint, no properties will be visible or be able to be changed. This is nice if you want to set up the component C++ side and not allow anything to change in the editor.

    UPROPERTY()
    UArrowComponent* ArrowComponent;
Renaming Components
If you want to rename a component, there are actually two names at play:
The name in the Blueprint, which you choose when creating the default sub object. Below it is the "RightPointerMesh". This is the name you can refer to in order to set properties and so on inside a Blueprint. If you rename this, you will need to fix all of the references to it, if any exist, in BP.

RightPointerMeshComponent = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("RightPointerMesh"));


The name of the component in C++, the "RightPointerMeshComponent" as seen below. If you rename this, you need to rename all references in C++ obviously.

UPROPERTY(VisibleAnywhere)
    UStaticMeshComponent* RightPointerMeshComponent;

If you rename a component's Blueprint name, it will LOSE ALL settings that you have set in the Blueprint, and go back to defaults.

It is also possible, if you move a component from a subclass to a base class, and the sub classes had Blueprint names that don't agree, that you will get a bugged component, which is to say when you select it in the Blueprint nothing will appear in the details panel.

If you have a problem with a component showing nothing in the details panel - rename the C++ component to something temporary:

UPROPERTY(VisibleAnywhere)
UStaticMeshComponent* RightPointerMeshComponent2;

Then compile, open the Blueprint and confirm that you can now see the details panel, save the Blueprint, then name it as you desire. The compiling/saving seems to clear something out.
AIController
This component ticks, but is only required to do so if you want it to update the rotation of the controlled character. Feel free to disable the tick if you don’t need this functionality. Below is the AIController tick.

void AAIController::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);
    UpdateControlRotation(DeltaTime);
}
Arrow and Scene Components
If an arrow or scene component is mobile, its transform will have to be updated to world every frame.
It’s best to make these components editor only where possible, caching their relative location, and using that for offsets.
Particle Systems
Use ‘Auto Manage Attachment’ to make them only move while active. (UE4.24+). You will need to set the AutoAttachParent so the system knows what to connect to, and you will need to set the auto attach rules if you intend them to snap to the target when activated.

DustMotesComponent = CreateDefaultSubobject<UNiagaraComponent>(TEXT("DustMotesComponent"));
DustMotesComponent->SetupAttachment(GetRootComponent());
DustMotesComponent->SetAutoActivate(false);
DustMotesComponent->AutoAttachParent = GetRootComponent();
DustMotesComponent->AutoAttachLocationRule = EAttachmentRule::SnapToTarget;
DustMotesComponent->AutoAttachRotationRule = EAttachmentRule::SnapToTarget;
DustMotesComponent->AutoAttachScaleRule = EAttachmentRule::KeepWorld;
DustMotesComponent->SetUseAutoManageAttachment(true);

Use pooling to reduce spawn times, by specifying the pooling method when spawning the systems.

UNiagaraFunctionLibrary::SpawnSystemAttached(
       ShootSystem, MotionController, NAME_None, FVector::ZeroVector, FRotator::ZeroRotator, EAttachLocation::SnapToTarget,
       true, true, ENCPoolMethod::AutoRelease, true);
Pawns
IN GENERAL, USE A BASIC SHAPE COLLIDER AS ROOT.
See how Epic made the character class - they used a collision capsule as the root component.
You will not be able to use the ‘Try to Adjust Location’ spawn collision handling method if you have a skeletal mesh or static mesh as the root component. The engine will print warning messages every time this is tried and it will fail.
If you don’t use a base shape for collision that is cheap, like a capsule, sphere, or box component, collision checking can be expensive.
Radial Force Components
On tick by default these do a multi-overlap which is very expensive!
If you are not using the force strength over time, and are using impulse, be sure to disable auto activate and tick.
RadialForceComponent->SetAutoActivate(false);
RadialForceComponent->SetComponentTickEnabled(false);
Skeletal Mesh Components
By default, these are set in the Optimization section to “Always Tick Pose and Refresh Bones”. Consider setting this to “Only Tick Pose When Rendered” (which doesn’t support root motion).
Skeletal mesh components must tick to update their transforms during animation. However, if your skeletal mesh does not animate for a period of time, disable the tick and enable it before animating. You can also set “Render Static” in optimization, however it will still tick if you don’t disable that manually.
The variable PredictedLodLevel in the skeletal mesh component can tell you the current LOD level. This is useful if you want to change spawning destruction, effects, particles, based on the LOD for performance.
If motion blur is on, consider disabling motion blur per bone. This is an expensive process if not needed.
Static Mesh Components
Generate overlap events and have collision by default. Consider disabling generate overlaps in the C++ base class.
StaticMeshComponent->SetGenerateOverlapEvents(false);
Collision should only be on for what is required. Turn off if not required, an easy way to do this is to use collision profile names. If it is required, ignore all collision and object responses that you can, and set the collision enabled type to just query, or just physics if possible.
Timeline Components
Timeline components tick when they are playing. Consider setting the timeline component tick interval to something larger than the frame time.
If the timeline component is doing something linearly, it’s cheaper to write a function that operates on tick in C++ and manipulates the value rather than running a separate tick and curve manipulation in the timeline.
Interfaces
C++
Interfaces | Unreal Engine Documentation

Interfaces are a great way to add generic functionality that can be a part of any class, rather than part of a base class.
Create them like you create any new C++ class, and select the Unreal Interface as the parent class.
It creates two classes, a U and and I prefixed version of the same class. The U version is used for Blueprint support and doesn’t need any modification.
Create virtual functions that are a part of the I class which you want to override.
If your class that you want to support the interface, just add it as a second class:

class SUNRISEPROD_API AToolActor : public AActor, public IBirdTutorialInterface

Override these functions in your class.
Check if a class supports the interface from C++ by casting to the I version of the class.

IReactToTriggerInterface* ReactingObjectA = Cast<IReactToTriggerInterface>(OriginalObject); // ReactingObject will be non-null if OriginalObject implements UReactToTriggerInterface.

Just call the function on that interface cast object if it’s not equal to nullptr.
Blueprint
To make your interface work both in Blueprints and C++, a few modifications are needed.
First, make the UINTERFACE be Blueprintable. This will allow a Blueprint to add the interface using the class settings.

UINTERFACE(MinimalAPI, Blueprintable)
class UBirdTutorialInterface : public UInterface

The functions that you create in the I prefix class will now need UFUNCTION(BlueprintNativeEvent) for functions with C++ and BP functionality.
The function will now require two definitions; one normal one for BP, and one that contains the _Implementation suffix that can be overridden in C++.

UFUNCTION(BlueprintNativeEvent)
void OnBirdArrival(ATutorialBirdNavigationActor* NewTutorialManager);
virtual void OnBirdArrival_Implementation(ATutorialBirdNavigationActor* NewTutorialManager) {}

On all C++ classes that support this, you must now override the _Implementation function.
In order to determine if an object supports the interface in C++ now, you can no longer cast it to the interface. Instead you must use Implements<>() to check. This handles objects that are BP or C++. Note that you must use the U version of the class for this check.

if(InterestActors[Index]->Implements<UBirdTutorialInterface>())

Executing the function on the object is done differently as well. You must use a static call of the interface to execute the function, with the object itself as the first argument, and subsequent arguments to pass in the function arguments. Note that you must use the I version of the class for this call.

IBirdTutorialInterface::Execute_OnBirdArrival(InterestActors[Index], this);
Collision
Rather than modifying collisions on different objects in the game manually, it's best to use Collision Profiles. This way you can set the collision profile on a particular class of objects, and later on be able to modify it all at once. It makes it easier to reduce the cost of collision and keep track of what interacts.

To make a new collision preset:

Go to the Project Settings -> Engine -> Collision.
Create a new Object Channel that is the object you want to use; for example, Projectile. Set the default response as desired; ignore is always a more performant choice, so that you have to specifically enable a response to this object in any other preset.
Create a new Collision Preset for the object.
It's critical to set CollisionEnabled properly. If you are only using overlap collision, set it to Query Only; if you are using physics or testing for hits, use Collision Enabled.
Set the object type to the newly created object channel name.
Create a good description.
Set the collision response on all channels. The fewer enabled channels, the better performance. So only turn on things that are needed.
In the DefaultEngine.ini, it will create the settings that appear in this collision settings under /Script/Engine.CollisionProfile. You can see what channel it assigns to your custom object, of which in C++ you will want to match up with a define so you can reference it.

[/Script/Engine.CollisionProfile]
+Profiles=(Name="Projectile",CollisionEnabled=QueryOnly,ObjectTypeName="Projectile",CustomResponses=,HelpMessage="Preset for projectiles",bCanModify=True)
+DefaultChannelResponses=(Channel=ECC_GameTraceChannel1,Name="Projectile",DefaultResponse=ECR_Block,bTraceType=False,bStaticObject=False)

In C++, create a statics file where you create FNames for these profiles:

#define PROJECTILE_CHANNEL ECC_GameTraceChannel1
static const FName ProjectileCollisionPreset = FName(TEXT("Projectile"));

Use this collision profile name for all appropriate colliders.

CollisionComp->SetCollisionProfileName(ProjectileCollisionPreset);
Pointers
Be careful when you use GetWorld()->Overlap... or ->Sweep when it comes to the query parameters.
FCollisionQueryParams::DefaultQueryParam is what is used by default, which traces COMPLEX. I suggest specifying parameters that don't use complex for performance, the normal constructor defaults this to false.
Nativization
If you find that you want to replace a blueprint only class with a C++ one, there’s a process that makes this a little easier to swap out all those in the game.
Create a new temporary blueprint based on your C++ class.
Setup all the variables, test, and get it to work identically to the original BP class.
Open the original BP class and delete everything from the event graph and any components that were added.
Reparent it to your temporary BP.
Reparent it to your C++ class. This will get the values from your temporary BP and set them, and then still be parented properly to just the C++ class.
Delete your temporary BP class.
Class inheritance can be found in Developer->Class Viewer. This is super helpful to know what BP’s inherit so that you can not break them when you try and reparent.
All properties that are in a blueprint that are changed from defaults are marked with a yellow arrow. This makes it easier to know what to set up in C++ as default values that differ from the norm.
If a BP has components that are socketed, just put the name in SetupAttachment() in the constructor.
Setting Up a Debug System
When setting up debug systems, be sure to design them so that:
They are event driven; the functionality is turned on/off via an event, and is not polling.
Use variables in BP’s so that someone can quickly tweak values without having to recompile.
Use boolean variables in BP’s to enable/disable debug functionality.
Have no or very little performance cost when disabled. You don’t want to end up with a development build that has horrible performance because these are stripped out only in shipping build.
Avoid console and config variables for debugging and tweaking if possible. Console variables must be polled, config variables must be reloaded. These are not fast processes.
Suggested is a bool, bShowDebug, on the class, that prints debug messages and draws debug using the debug helpers.
Arrays / Scope Locks
Scope locks are used to prevent multiple threads from accessing the same variables at the same time. This can be a problem when a variable is accessed during tick and possibly in overlap events, in the main class, etc. This is particularly problematic with arrays.

To set up a scope lock, first add a FCriticalSection variable:

FCriticalSection SocketsCriticalSection;

UPROPERTY(Transient)
TArray<AActor*> Sockets;

Next, where you are accessing the variable you want to protect, implement a scope lock.

void AddSocket(AActor* NewSocket)
{
    // The area in the braces will be locked
FScopeLock Lock(&SocketsCriticalSection);

Sockets.Add(NewSocket);
}

void RemoveSocket(AActor* OldSocket)
{
    // The area in the braces will be locked
FScopeLock Lock(&SocketsCriticalSection);

Sockets.Remove(OldSocket);
}

This will cause the area within the last braces {} to be locked out. So if another thread arrives at a scope lock and it’s locked, that thread will have to wait for the scope lock to go out of scope. For example:

void Tick(float DeltaSeconds)
{
    FScopeLock Lock(&SocketsCriticalSection);
for(AActor* Actor : Sockets)
    {
    // Do something with the actors and don’t fear they will 
// change while you are accessing them. Otherwise, if 
// remove gets called while ticking, you will crash.

// Always check validity.
if(!IsValid(Actor))
{
    continue;
}

Actor->SetActorLocation(...);
...
}
}
Multiple Tick Functions for a Single Actor
Sometimes you need to have separate tick functions on a single actor so that it’s easy to stop and start functionality without affecting other things.

To do this, go to the class you want to have multiple ticks, and create a new structure for your new tick that overrides FTickFunction. You must also use the template function to prevent the compiler from trying to use the = operation on the tick function, which is impossible. Override the diagnostic message and execute tick functions.

USTRUCT()
struct FPickupAttachedToContainerTickFunction : public FTickFunction
{

  GENERATED_BODY()

  UPROPERTY(Transient)
  class APickupActor* Target;

  virtual FString DiagnosticMessage() override;
 
  virtual void ExecuteTick(const float DeltaTime, const ELevelTick TickType, ENamedThreads::Type CurrentThread, const FGraphEventRef& MyCompletionGraphEvent) override;
};

template<>
struct TStructOpsTypeTraits<FPickupAttachedToContainerTickFunction> : public TStructOpsTypeTraitsBase2<FPickupAttachedToContainerTickFunction>
{
  enum
  {
     WithCopy = false
   };
};

Add the structure to your class:

UPROPERTY(EditDefaultsOnly)
struct FPickupAttachedToContainerTickFunction ContainerTick;

Override your RegisterActorTickFunctions():

void APickupActor::RegisterActorTickFunctions(const bool bRegister)
{
  Super::RegisterActorTickFunctions(bRegister);

  if(bRegister)
  {
     if(ContainerTick.bCanEverTick)
     {
        ContainerTick.Target = this;
        ContainerTick.SetTickFunctionEnable(ContainerTick.bStartWithTickEnabled || ContainerTick.IsTickFunctionEnabled());
        ContainerTick.RegisterTickFunction(GetLevel());
     }
  }
  else
  {
     if(ContainerTick.IsTickFunctionRegistered())
     {
        ContainerTick.UnRegisterTickFunction();         
     }
  }
}

And set it up in your constructor:

// Set whether it can ever tick; set to false to make it never work.
ContainerTick.bCanEverTick = true;
// Set whether it starts with tick enabled or not.
ContainerTick.bStartWithTickEnabled = false;
// Set the tick group; you can tick pre or post physics, etc.
ContainerTick.TickGroup = TG_PrePhysics;
// Make sure to set the tick function disabled here, or it will start by itself even if bStartWithTickEnabled is false.
ContainerTick.SetTickFunctionEnable(false);

The overridden functions:

FString FPickupAttachedToContainerTickFunction::DiagnosticMessage()
{
  return Target->GetFullName() + TEXT("[TickActor]");
}

void FPickupAttachedToContainerTickFunction::ExecuteTick(const float DeltaTime, const ELevelTick TickType, ENamedThreads::Type CurrentThread, const FGraphEventRef& MyCompletionGraphEvent)
{
  if(Target && !Target->IsPendingKillOrUnreachable())
  {
     if(TickType != LEVELTICK_ViewportsOnly || Target->ShouldTickIfViewportsOnly())
     {
        FScopeCycleCounterUObject ActorScope(Target);
        if(Target->GetWorld())
        {
           Target->AttachedToContainerTick(DeltaTime * Target->CustomTimeDilation);
        }
     }
  }
}

Now you should be set up to enable/disable your custom tick as you see fit.
Android
Platform Settings
Package game data inside.apk?
If you set this setting, all files go in the APK. Makes it easier to install and upload, but has a file size limit.
If you don't set it, you get additional .OBB files that need to be dealt with.
UseExternalFilesDir for UE4Game files?
If you set this to true, you won't need to read/write external files (and thus won't need the permissions) to save a game.
Turning this on will also make you not be able to browse to the game on the Android device and see the log files.
Removing Undesired Permissions
The best way to do this is to use UPL to remove the permissions as it's being built.
Unreal Plugin Language | Unreal Engine Documentation

In your Source directory, next to your .Build.cs file, create a new XML file. Name the file AndroidSanitizePermissions_UPL.xml.

<?xml version="1.0" encoding="utf-8"?>
<root xmlns:android="http://schemas.android.com/apk/res/android">
    <androidManifestUpdates>
        <removePermission android:name="android.permission.READ_EXTERNAL_STORAGE" />
        <removePermission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
    </androidManifestUpdates>
</root>

This file is used to specify which permissions to strip out of the AndroidManifest.xml. Once you've created this, edit your Build.cs file.

Add this to the top:
using System.IO;

And add this after the private dependency module names:

if(Target.Platform == UnrealTargetPlatform.Android)
        {
            var manifestFile = Path.Combine(ModuleDirectory, "AndroidSanitizePermissions_UPL.xml");
            AdditionalPropertiesForReceipt.Add("AndroidPlugin", manifestFile);  
        }

Now when you package your Android project, check intermediates to see the AndroidManifest.xml, and you can see what permissions are there. Then you can add these to the list that you want removed as necessary. Note that some permissions may be added by plugins, so make sure to remove all plugins that aren't in use.
Multiplayer
UE4 Multiplayer
Getting Values from the Class Default Object
If you have a class, that has a Blueprint in which you set the value of a variable:

UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Ability")
EDeathChuteAbilityInputId AbilityInputId = EDeathChuteAbilityInputId::None;

You can read the value of the default value of the variable set in the BP from the class itself without instantiating it using GetDefaultObject():

for(const TSubclassOf<UDeathChuteGameplayAbility>& Ability : DefaultAbilities)
{
  AbilitySystemComponent->GiveAbility(FGameplayAbilitySpec(Ability, 0.0f,
     static_cast<int32>(Ability.GetDefaultObject()->AbilityInputId), this));
}
Redirecting Class Names
If you want to rename the underlying class of a Blueprint, you can use an active class redirect in the DefaultEngine.ini.

[/Script/Engine.Engine]
+ActiveClassRedirects=(OldClassName=”MyAwesomeClass”,NewClassName=”MyNewClass”)

If you redirect the class, then save all Blueprints based on it, you can then delete the class redirect. Be careful if you do this, because if you remove the redirect without resaving all BPs that are based on it, those BPs will fail and can't be fixed without putting back the redirector.
Designers
Naming Convention
Follow this guide, or create your own variation:
Allar/ue4-style-guide: An attempt to make Unreal Engine 4 projects more consistent

Some examples:
Static Mesh: S_
Skeletal Mesh: SK_
Level (Persistent): _P
Blueprint: BP_
Texture: T_, then _? at the end, _D for diffuse, _R for rough, etc.
Animation Blueprint: ABP_
Skeleton: SKEL_
Blueprints
Please read this first. Blueprint Best Practices
ALWAYS create a Blueprint from a base C++ class that an engineer has created in the project.
ALWAYS have the components created for you in C++.
Creating them in BP is much slower for the engine when spawning. https://medium.com/@rgbguy/how-epic-games-optimized-unreal-engine-for-fortnite-battle-royale-part-1-8190f10bf940
BP components must be looked up with Actor->FindComponentsOfClass in C++, which is much slower than using a direct accessor.
Replacing components in BP with a C++ version later can be painful to do. It messes up level serialization and any level with the object in question will need to be loaded/saved again.
If you replace a BP component with a C++ one, data saved in inherited components will be lost.
DON'T USE MORE THAN 50 NODES. If you notice that your blueprint is becoming larger than this, have an engineer nativize parts of the Blueprint to reduce what is being done to something more manageable. 
USE TICK SPARINGLY!
If you think you have to tick, check with an engineer. It may be possible to do it without ticking or tick can be used in C++ which will be much cheaper.
DON'T USE TIMELINES. These appear to be a way to get around tick, but that is not true. They tick on their own, and typical usage can be very expensive if you are updating a material parameter per frame for example. They should only be used if you must interpolate a designer specified curve, and can be done C++ side.
DON’T USE GET ALL ACTORS OF CLASS.
If you find that you need to do this, have an engineer create a place where these actors will report themselves to. That way, all of those actors can be found and acted upon without having to search through every actor in a scene.
Alternatively, create a GetThisActor() function that uses GetActorOfClass if a variable you set up is null, and just returns the variable if not. This way you only call GetActorOfClass once.
Try not to use DELAY.
Delay notes use latent actions, which tick the blueprint to count time, and are slow.
These are fine for testing timing of calling functions but once you have everything the way you like it, but should be replaced when optimizing.
Only small amounts of logic should be in the BP. If you find yourself adding more than 50 nodes, ask an engineer to create the functionality you need in C++ and give you variables to tweak in a data only BP.
BE CAREFUL OF MACROS. Think of a macro as a rubber stamp that copies code and pastes it into the BP. If the macro has anonymous variables, these can be duplicated each time, and memory consumption can get huge. It’s better to use functions instead.
Artists and Animators
Collision
It’s important to make a simple collision for static mesh objects that fit the geometry.
If you don’t make a simple collision, it will use complex (per triangle) collision for the object, which is very expensive!
Use player collision and visibility collision view modes to check on your collision complexity.
You can use the PhysX Visual Debugger and the command pvd connect to visualize the collision that UE4 is sending to PhysX. Note that it will send both complex and simple collision if you don’t have this disabled in the project settings, which allows trace responses to use complex collision for higher accuracy if needed.
If the object will never collide, just use the engine to create a sphere collider anyway that surrounds the object, and set the collision to disabled by default in the static mesh itself.
Collision shape in order of performance cost:
Sphere
Capsule
Box
Convex Element
Complex
IF AN OBJECT IS SET TO VISIBLE, BUT HIDDEN IN GAME, IT WILL STILL COLLIDE. Be sure to disable collision on these objects!
Objects that don’t need collision, such as reference color or size actors, should have their collision set to no collision in the world.
Collision can be made and imported from whatever program the asset was created in if you name it appropriately.
FBX Static Mesh Pipeline | Unreal Engine Documentation
Be sure to keep in mind the cost of the performance for the collision primitives; use USP, UCP, UBX if you can, and UCX when you have to.
You should only enable collision types and channels that are used.
It’s best to use collision profiles, and to set the profile properly, instead of setting up custom collision on everything. Look in the project settings -> Collision.
Lightmaps
If you plan on using baked lighting, light map resolution per object will directly affect the quality of the shadows, size of the light bake, and amount of time to bake.
First look under LOD 0 Build Settings for a mesh you want to bake lighting and open the UV channel that is being used for the lightmap.

As you can see above, the min lightmap resolution is 64, and it is not set to generate lightmap UVs. Note the spacing around the edges and between UV islands. This spacing is set up to try to prevent bleed when baking.
In this case, I enabled Generate Lightmap UVs, set the source lightmap index to 1, and the destination to 1. Then I set the minimum resolution to 512.


Note how much less space is around the islands. Since the minimum is 512, there's no need for that extra space to prevent bleed. Next set the lightmap resolution to 512 in the general settings.
Now it's ready to be used in the world and baked! If you decide to adjust the lightmap size, you can go back and set the build setting lower, and reduce the lightmap resolution under general settings.
You can also look under View->OptimizationViewModes->LightmapDensity to visualize this in a level.
If a part of a mesh will never be seen, it may be worth it to edit the lightmap UVs manually to make the ones that are seen as large as possible and those that are very small.
You can look under World Settings -> Lightmaps to see what is being generated and their sizes.
LOD
Static Mesh
The engine can create LODs for you using Simplygon.
Open your static mesh. Set the LOD Group to auto generate LODs.
They can be visualized using the Level Of Detail Coloration -> Mesh LODs mode from the view mode picker.

Skeletal Mesh
Similar to static mesh, skeletal mesh can have their LODs generated for you.
Open the skeletal mesh. Navigate to LOD settings, and set the number of LODs you desire. It will generate them.
Use the LOD picker to specify the settings for each LOD if you don’t like the defaults.

Root Motion
UE4 Root Motion
Materials
CREATE MASTER MATERIALS THAT ALL OTHER MATERIALS INHERIT FROM (MATERIAL INSTANCES).
The reason for this is to prevent the graphics card from having to load different materials as much as possible.
You can see the cost of the shaders in the stats window.
You can visualize how many material swaps are being done in a frame using RenderDoc.
For each different type of shader, you will need one master material. Below are some examples.
MM_Blockout - Vector parameters for BaseColor, Metallic, and Roughness. Leave the material domain to surface, blend mode to opaque, and shading model to default lit.
MM_Substance - TextureSampleParameter2D for BaseColor, and Normal, and one for the combine occlusion, roughness, and metallic produced by Substance Painter. You will need to make sure that the sampler type on each parameter is color for base color, normal 
MM_Masked - Set the blend mode to masked. This is for textures that are cut out, like foliage.
MM_Translucent - Set the blend mode to translucent.
Texture Size
ALL TEXTURE SIZES MUST BE A POWER OF 2! 
If this is not done, the engine cannot mipmap the texture, lowering the resolution automatically when it is far away. 
It makes it expensive to load the texture into the graphics card memory.
It makes it expensive to look up in the graphics card memory. If two pixels that are in a quad are far from each other, it will have to get the large version of the texture to compare them.
The texture size required for assets can be visualized using the Required Texture Resolution view mode. Select an actor, then select the texture from the top right to see coloration.

What you want is “Good” at the distance the character will be from the object.
Adjust the Maximum Texture Size OR the texture’s LOD bias in order to scale it down to an acceptable level in a non-destructive manner.
When importing textures, it’s important to set the texture settings properly for desired results.
If it’s a diffuse/albedo texture, sRGB should be checked and default settings will work.
If it’s a grayscale texture such as roughness, metallic, or a mask, sRGB should NOT be checked and the compression setting should be masked or grayscale. Grayscale is MUCH less compressed, so only use this when visual aberrations are noticed (gradients).
If the texture is a packed texture from Substance or some other program that has roughness, metallic and other channels separated into each color (red is metallic, etc), use masked..
Driving Material Parameters from an Animation
If you have a material parameter that you wish to change every frame from an animation:
Open the animation.
Create a new curve and name it the name of the scalar parameter in the material.



In the Window->AnimCurves, set the curve to be material type.



Modify the curve and see it change in real time.
Materials on Instanced Static Meshes
UE4 Materials on Instanced Static Mesh Component Instances
Audio Engineers
Follow this naming scheme for naming assets in the project:
Allar/ue4-style-guide: An attempt to make Unreal Engine 4 projects more consistent
How Built in Audio Works
The audio system in Unreal Engine is built on top of source wave files; these wav files have settings that can be changed without resaving the associated UASSET by using cue files. Cues allow you to mix, loop, randomly play, modulate, change volume, set classes, attenuations, and concurrencies and many other features, but are not always required.

Classes are used to specify large groups of audio types that you might want to operate on all at once. You should have one per concurrency.

Attenuations allow you to specify how audio is attenuated or panned.

Concurrency allows you to specify how to control the number of voices of a sound that can be present at once and how to deal with old sounds; for example, you may want to have only one voice for a particular sound and remove the older one if you have a new one happen.
Source Wave Files
Sound wave files (*.wav) should all be the same sampling rate (44.1kHz or 48kHz). It's a good practice to keep sampling rates consistent in the project.
Files are monaural if they are to be localized / spatialized in the world. For example, if you want to hear a gun shooting sound on the left side, this sound should be mono so that it can be panned.
Music or ambient sounds can be stereo, but should be used sparingly, especially if the target hardware is mobile.
All audio files are to be normalized before import. This is to keep a consistent volume across the project. DO NOT LOWER THE VOLUME OF SOUNDS BY EXPORTING THEM AT A LOWER VOLUME FOR INPUT INTO THE ENGINE.
The reason you don’t want to do this is that the audio is sampled into discrete values as a sampling rate and bit depth. If you look at the bit depth chart, you can see how this number is fixed, and if you squish the audio down very small, then the accuracy of the audio is lost.

All audio files should fade out and be as short as possible.
All audio files are to be named with the prefix A_. For example, A_FlowerGrow1.
Sound Cues
Sound cues can be used to specify properties of the wave files and how they are played in combination or with modulation.
Sound cues are to be named with the prefix A_ and postfix _Cue. For example, A_FlowerGrow1_Cue.
Classes
Make a sound class for all sounds, called default, and make child classes for other types of sound depending on the project (for example, Music, Effects). Sound classes have no pre or postfix. For example, Effects.
Sound class mixes can be used to switch audio of specific classes between different mixes; for example, you can make all the effects class sounds go through a low pass temporarily by using a default and low pass sound class mixer. They are named with MIX_ prefix; for example, MIX_Lowpass.
Attenuation
Make a default sound attenuation, which is required to pan sounds; they are named with a prefix ATT_, for example ATT_Default.
All cues that are mono and need to be panned should use this attenuation.
You can create additional ones if you want to attenuate the volume in specific ways depending on the sound.
Sound Concurrency
Sound concurrency can be set on a per class basis, which allows you to limit the number of voices for that class.
For example, if you only want to hear 4 explosions at once, that can be set here.
Name them by their class name with a postfix of _SC. For example, Explosion_SC.
Basic Performance
Good video(s) to watch to start:
UE4 Performance and Profiling | Unreal Dev Day Montreal 2017 | Unreal Engine
Profiling and Optimization in UE4 | Unreal Indie Dev Days 2019 | Unreal Engine
Who Is Responsible For Performance?
Everyone is. It’s important that all of us profile soon and often, pay attention to the output log to verify things we’ve created aren’t spamming warnings and errors, and do map checks to avoid problems with assets we have added.
There are many tools that are “easy” to use while creating to keep an eye on performance.
Collision Costs: Developer Tools -> Collision Analyzer
CPU Costs: Developer Tools -> Session Front End
GPU Costs: GPU Visualizer (CTRL-SHIFT-COMMA)
Unreal Insights: Unreal Insights Overview
In depth GPU costs: Renderdoc. This is a wonderful tool that’s easy to set up and shows you the cost of everything you are rendering, and visualizes it!
RenderDoc | Unreal Engine Documentation

Artists
UE4 Blender
Importing Models
Make sure you have no overlapping UV’s.
The engine will automatically generate a UV channel for baked shadows; if this is not desired you can import your own by adding a second UV channel and specifying it in the mesh settings.
Make sure you have a simple collision object to import or create one within the engine.
Make sure to create basic LODs in the engine. These are non-destructive and can be tweaked later or removed if necessary.
If you have vertex color, make sure to set up in import settings to include this and not wipe it out. By default it will ignore it.
Importing Textures
Make sure they are a power of 2 in size (512x512, 1024x1024, 2048x2048, etc).
Make sure the compression settings / sRGB settings are correct.
Diffuse - Default Compression, sRGB
Roughness, Metallic - Default Compression, NO sRGB.
Normal Map - Normal Map Compression, NO sRGB.
Packed Textures - Default Compression, NO sRGB
Masks - Masked Compression, NO sRGB.
Map Check
When adding assets, run BUILD->MAP CHECK.
This will catch overlapping UV’s, improper bound scales, changing the number of materials in an import, etc.
Engineers and Designers
Hard References in Blueprint
When blueprints reference another blueprint, it must load that blueprint. Then that new blueprint must load all blueprints it references, and so on. This can create situations where many things need to be loaded and is very slow. Casting while in a blueprint creates hard references.

Instead of doing this, use interfaces or soft references.
Blueprint Interface | Unreal Engine Documentation
Call an interface function on the actor that you would cast.
If the actor supports the interface, it will execute the function, otherwise it will silently fail.
Ticking
The command ‘dumpticks’ on the console will dump all ticking actors and components to the console.
Check that what you are creating only ticks where expected. Remember to run it during gameplay. Be sure to disable tick on components that you don’t need to tick, or don’t need to tick always, and set them up to enable as necessary.
Remember to tick at the minimum rate you can afford by setting tick interval on actors and components.
There are many ways to tick; in order of cheapest to most expensive:
Ticking in C++
Timer in C++
Ticking in BP
Delay in BP
Timer in BP
Output Log
It’s up to you to make sure that what you’ve created or worked on doesn’t spam the output log with warnings and errors. It makes it very difficult to find problems later if the output log is filled.
Optimizing Performance
CPU Optimization
UE4 CPU Optimization
GPU Optimization
UE4 GPU Optimization
UE4 PSO Caching
How to Use Render Doc on Mobile
UE4 Android (Phone, Quest)
Console Commands
View Modes
These can be useful to view things in a mode while playing the game, such as in VR. Some only work with one eye.

viewmode lit
viewmode unlit
viewmode wireframe
viewmode collisionpawn
viewmode collisionvis
Command Line Options
-FeatureLevelES31
Forces ES3.1 mode in a build.
