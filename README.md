## Preface

There are not many resources for a deep dive into Epic Game's Unreal Engine architecture beyond stepping through the source code or reading the little information provided in the official documentation.
This should help fill in the gaps for those who are interested in learning more about the engine.

**Important for new readers:** This documentation goes from lower-layer to upper-layer parts of the engine. So if you want game-related information, you should start at the [bottom of the runtime engine architecture section](#game-layer-game-code).
Otherwise, start from the top, as understanding the lower layers will help you understand the upper layers.

## Two Parts

Unreal Engine can be broken into two important components: the Editor and the Runtime Engine. The Editor is the suite of tools used to create and edit content for the game. The Runtime Engine is the part that runs the game.

Unlike most other game engines, Unreal Engine (which took significant inspiration from the architecture of its competitor Quake Engine) and Quake Engine has the tool suite (`UnrealEd`) built directly into the runtime engine.
There are a lot of benefits for this, most importantly that the game can run via PIE (Play in Editor) in-editor without performance impacts, loading asset contents and seeing them in their full glory, in addition to other factors
such as reducing code duplication between the two. There are also drawbacks in developer productivity due to locking of files preventing simultaneous editing of assets. More on this later.

# Runtime Engine Architecture

Unreal Engine, like all software systems and game engines, is built in layers. In order to avoid circular dependencies which negatively impact testability, platform independence, and re-usability/modularity, the lower layers do not depend on upper layers.

The upper-most layers contain the well-known `GameFramework` classes containing `PlayerController` and
`GameModeBase`. The lower layers contain hardware-specific implementation such as `Runtime/Unix`.

From top to bottom, the layers are:
* Game-Specific Subsystems
* Gameplay Foundations, Rendering, Profiling & Debugging, Scene Graph / Culling, Visual Effects, Front End, Skeletal Animation Collision & Physics, Animation, AI, HID Audio, Input
* Resources (Resource Manager)
* Core Systems
* Platform Independence Layer (Networking, File System)
* 3rd Party SDKs (DirectX, OpenGL, PhysX)
* OS
* Drivers
* Hardware

To keep the project modular, many features within these layers (e.g. Replication Graph, Gameplay Ability System) are separated out into optional Plugins.

## Target Hardware Layer

This layer is platform-specific. Generally, Unreal Engine is platform-agnostic, but there are some platform-specific code and optimizations for different computer or console systems.
The Quake 2 engine, for example, had significant [optimizations made for the Intel's Pentium processor and its pre-fetching cache](https://fabiensanglard.net/quake2/quake2_software_renderer.php) due to their popularity at the time.

### Entry Point

The entry point for the engine depends on the platform. Every Windows program has an entry-point function called `WinMain`.
Unreal Engine's entry point for Windows, like all other game engines, is the `WinMain` function defined in `Windows/LaunchWindows.cpp`.
The [Quake 2 engine](https://github.com/id-Software/Quake-2/blob/master/win32/sys_win.c#L594), for example, also has the
identically named function.

Each supported platform has their respective entry point:
* MacOS: `INT32_MAIN_INT32_ARGC_TCHAR_ARGV` in `Mac/LaunchMac.cpp`
* Linux: `int main` in `Linux/LaunchLinux.cpp`
* IOS: `int main` in `IOS/LaunchIOS.cpp`

```c++
// Windows specific parameters: HINSTANCE is identification to prevent class name clashing
int32 WINAPI WinMain(_In_ HINSTANCE hInInstance, _In_opt_ HINSTANCE hPrevInstance, _In_ char* pCmdLine, _In_ int32 nCmdShow)
{
	int32 Result = LaunchWindowsStartup(hInInstance, hPrevInstance, pCmdLine, nCmdShow, nullptr); // Launch Unreal Engine
	LaunchWindowsShutdown(); 
	return Result; // 0 on success, error level otherwise
}
```

### Main Engine Loop

It is a very simple while loop.

```c++
while( !IsEngineExitRequested() )
{
    EngineTick();
}
```

### Apple

Apple-hardware specific code is under `Runtime/Core/Apple`.

```c++
// Core/Private/Apple/ApplePlatformMemory.cpp

// In this file, we keep track of the amount of memory we've allocated for an Unreal app running on an Apple device.

#include <stdlib.h>                          // c standard library
#include <objc/runtime.h>                    // for inspecting and manipulating Objective-C runtime data structures
#if PLATFORM_IOS && defined(__IPHONE_13_0)   // Include only for iOS 13+
#include <os/proc.h>                         // in order to call os_proc_available_memory which determines the amount of memory available to the current app (your game running on the iPhone)
#endif                                       // Only need to include one header specific to iOS 13+.
#include <CoreFoundation/CFBase.h>           // Core Foundation is an API for C used for its operating systems, provides primitive data types, etc.. Types used: CFIndex is a typedef for a signed integer type (SInt32) used to represent indices into a CFArray or CFString
                                             // CFOptionFlags is a typedef for an unsigned integer type (UInt32) used to represent bitfields for passing special allocations into CF funcs.
                                             // CFAllocatorContext is a struct containing callbacks for allocating, deallocating, and reallocating memory, and for retaining and releasing objects.

#include "HAL/LowLevelMemTracker.h"          // for FLowLevelMemTracker in order to track memory allocations
#include "Apple/AppleLLM.h"                  // Apple's Low-Level Memory Tracker which tracks all allocations from the OS

// Skip ~250 lines including functions for memory allocation

FMalloc* FApplePlatformMemory::BaseAllocator()                       // 
{
#if ENABLE_LOW_LEVEL_MEM_TRACKER
	FPlatformMemoryStats MemStats = FApplePlatformMemory::GetStats(); // FPlatformMemoryStats is the Apple implementation of FPlatformMemoryStats which contains memory numbers on available/used physical/virtual memory
	FLowLevelMemTracker::Get().SetProgramSize(MemStats.UsedPhysical);
#endif
```

### Xbox

### Playstation

### Nintendo

### Mobile

### Web

### VR

## Drivers Layer


## Operating System Layer

### Windows

### MacOS

### Linux

### IOS


## 3rd Party SDKs Layer

### Graphics

#### DirectX

#### Vulkan

#### OpenGL

### Physics & Collision

#### PhysX


## Platform Independence Layer

Unreal Engine's Platform Independence Layer is called the **Hardware Abstraction layer (HAL).** Everything under `Runtime/Core/Public/HAL` falls under this layer.

### Platform Detection

`Platform.h` defines multiple header guards for different platforms, such as `PLATFORM_CPU_X86_FAMILY` for x86 processors, `PLATFORM_CPU_ARM_FAMILY` for ARM processors, and `PLATFORM_APPLE` for Apple devices.
The `FPlatformAtomics` class contains platform-specific implementations of atomic operations.

### Primitive Data Types
### Collections & Iterators
### File System
### Networking
### Hi-Res Timer
### Threading Library
### Graphics Wrappers
### Physics & Collision Wrappers


## Core Systems Layer

### Data Structures & Algorithms

All software systems require containers that allow for the storage and manipulation of data and algorithms that use these data structures to solve problems. Many projects use standard and
third-party libraries that provide implementations of these data structure and algorithms.

However, Unreal Engine neither uses STL, the C++ Standard Template Library, nor third-parties like Boost, Folly, or Loki. It opts for a custom solution for performance benefits. Although it isn't used anywhere in the Engine, many of the custom implementations are inspired by Boost, a popular and powerful data
structures and algorithms C++ library that shares similar style to its predecessor STL. Many of the third party libraries included by UE, however, do use Boost.

Note: For an unknown reason there is an import for Boost (`Source/ThirdParty/Boost`).

#### Data Structures

##### Container Table

Unreal Engine defines most if not all its containers in `Runtime/Core/Public/Containers`.

<table>
<tr>
<th>Container</th>
<th>Description</th>
<th>STL Equivalent</th>
<th>Boost Equivalent</th>
</tr>
<tr>
<td>TAllocatorFixedSizeFreeList</td>
<td>Low-level container that allocates memory in fixed-size blocks. Its fixed-size property is more efficient compared to a heap allocator that allocates memory in variable sizes which leads to inefficiencies because otherwise the free list needs to be traversed to find the first 
block that is large enough to fit the requested size. Instead, the fixed-size allocator can simply return the next block in the free list. The fixed-size allocator is also more efficient than a fixed-size allocator because it does not need to keep track of the size of each block.
</td>
<td>-</td>
<td>boost::container::static_vector_allocator</td>
</tr>
<tr>
<td>TArray</td>
<td>Dynamic array</td>
<td>std::vector</td>
<td>boost::container::vector</td>
</tr>
<tr>
<td>TArrayView</td>
<td>
A statically sized (it does not have add or remove) view (it is a representation of a real array) of an array of typed elements. Designed for functions to take in as an argument where the function does not need to mutate the array.

```c++
// e.g.: SumAll takes in an array and returns the sum of all the elements, notice it does not mutate the array since accumulate is a const function
int32 SumAll(TArrayView<const int32> array) const
{ 
    return Algo::Accumulate(array);
}

int32 sum = SumAll({1, 2, 3, 4, 5}); // You can pass whatever array type you want, as long as it is a view of an array of int32
TArray<int32> MyTArray = {1, 2, 3, 4, 5}; // TArray is fine too
sum = SumAll(MyTArray); // The TArray is treated as a view
CArray<int32, 5> MyCArray = {1, 2, 3, 4, 5}; // CArray is fine too
sum = SumAll(MyCArray);
```
</td>
<td>-</td>
<td>boost::container::vector</td>
</tr>
<tr>
<td>TSet</td>
<td>Set</td>
<td>std::set</td>
<td>boost::container::set</td>
</tr>
<tr>
<td>TMap</td>
<td>Map</td>
<td>std::map</td>
<td>boost::container::map</td>
</tr>
<tr>
<td>TQueue</td>
<td>Queue</td>
<td>std::queue</td>
<td>boost::container::queue</td>
</tr>
<tr>
<td>TLinkedList</td>
<td>Linked list</td>
<td>std::list</td>
<td>boost::container::list</td>
</tr>
<tr>
<td>TDoubleLinkedList</td>
<td>Double linked list</td>
<td>std::list</td>
<td>boost::container::list</td>
</tr>
</table>

#### Algorithms

##### Algorithm Table

Unreal Engine defines most if not all its algorithms in `Runtime/Core/Public/Algo`.

| Algorithm | Description | STL Equivalent | Boost Equivalent |
|-----------|-------------|----------------|------------------|
| AllOf | Returns true if the given predicate returns true for all elements in the range. | std::all_of | boost::algorithm::all_of |
| AnyOf | Returns true if the given predicate returns true for any element in the range. | std::any_of | boost::algorithm::any_of |
| NoneOf | Returns true if the given predicate returns false for all elements in the range. | std::none_of | boost::algorithm::none_of |
| Find | Returns an iterator to the first element in the range that matches the given value. | std::find | boost::algorithm::find |

### Module Start-Up and Shut-Down

### Assertions

### Automation

#### Unit Testing

#### Gauntlet

### Memory Allocation

### Math Library

### Strings and Hashed String Ids

#### FName

### Debug Printing and Logging

### Localization Services

### Movie Player

### Parsers

### Profiling / Stats Gathering

### Engine Config

### Random Number Generator

### Curves & Surfaces Library

### RTTI / Reflection & Serialization

### Object Handles / Unique Ids

### Asynchronous File I/O


## Resources (Game Assets) Layer

### Resource Manager (UnrealEd)

Unreal Engine's highly centralized resource manager is a unified interface to access all types of game assets. This includes `umap` and
`uasset` files.

### 3D Model Resource

### Texture Resource

### Material Resource

### Font Resource

### Skeleton Resource

### Collision Resource

### Physics Parameters

### Game World/Map


## Collision & Physics Layer

This layer of the engine handles collision detection and rigid body dynamics (which is where it gets "physics" from). Calling it physics is a bit of a
misnomer, as the engine is primarily concerned about forces and torques acting on rigid bodies, and not much of anything else.

Typically, game engines do not implement their own physics engine. Instead, they use SDKs from a third-party physics & collision engine. Hence, Unreal Engine 
uses Nvidia's PhysX SDK, which is a free and open source. It does however have some of custom implementation such as `ARigidBodyBase`. It does not use Havok or Open Dynamics Engine.

Unreal Engine uses the `PxRigidActor` class from PhysX's `physx` namespace to represent dynamic and static rigid bodies.

Relevant folders are `Runtime/PhysicsCore` and `Runtime/Engine/PhysicsEngine`.

### Forces & Constraints

### Ray/Shape Casting (Queries)

### Rigid Bodies

### Phantoms

### Shapes / Collidables

### Physics/Collision World


## Human Interface Devices (HID) Layer

### Game-Specific Interface

### Physical Device I/O


## Profiling & Debugging Layer

### Recording & Playback

### Memory & Performance Stats

### In-Game Menus or Console


## Low-Level Renderer Layer

### Materials & Shaders

### Static & Dynamic Lighting

### Cameras

### Text & Fonts

### Primitive Submission

### Viewports & Virtual Screens

### Texture & Surface Management

### Debug Drawing (Lines etc.)


## Scene Graph / Culling Optimizations Layer

### Spatial Hash (BSP, Tree, kd-Tree, ...)

### Occlusion & PVS

### LOD System


## Visual Effects Layer

### Light Mapping & Dynamic Shadows

### HDR Lighting

### PRT Lighting, Subsurface Scatter

### Particle & Decal Systems

### Post Effects

### Environment Mapping


## Front End Layer

### Heads-Up Display (HUD)

### Full-Motion Video (FMV)

### In-Game Cinematics (IGC)

### In-Game GUI

### In-Game Menus

### Wrappers / Attract Mode


## Skeletal Animation layer

### Animation State Tree & Layers

### Inverse Kinematics (IK)

### Hierarchical Object Attachment (+Gameplay Foundations Layer)

### Game-Specific Post-Processing

### LERP and Additive Blending

### Animation Playback

### Sub-skeletal Animation

### Animation Decompression

### Skeletal Mesh Rendering (+Low-Level Renderer Layer)

### Ragdoll Physics (+Collision & Physics Layer)


## Audio Layer

### DSP/Effects

### 3D Audio Model

### Audio Playback / Management


## Online Multiplayer Layer

### Matchmaking & Game Management

### Object Authority Policy

### Game State Replication


## Gameplay Foundations Layer

### High-Level Game Flow System/FSM

### Scripting System

#### UnrealScript

#### Blueprint

All the graph editor tools are behind the scenes a `UEdGraph`. This includes Blueprint, Material, and Animation graphs.
The `UEdGraph` is a simple graph data structure that listeners on every node.

The all the graphs for a Blueprint, such as the Event Graph, are combined into an Ubergraph.

#### Verse

### Static World Elements

### Dynamic Game Object Model

#### UObject

All gameplay objects in the engine are derived from this class.
	
![uobjectdiagram](https://user-images.githubusercontent.com/11065634/229152810-f9c3423a-10be-4d60-93a1-1f1dda037fd3.jpg)

##### Garbage Collection

When `UObject`s are created, they are automatically added to a global array `GUObjectArray` which the GC uses for tracking any unreferenced objects for deletion, unless they have
flags to explicitly prevent garbage collection. One of these flags is the `EInternalObjectFlags::RootSet`.

```c++
// Runtime/CoreUObject/Private/UObject/UObjectHash.cpp

// Global UObject array instance
FUObjectArray GUObjectArray;                                   // To keep track of all UObjects
```

There are 3 ways to keep them referenced in the internal reference graph:

1. With a strong reference (`UPROPERTY()`) to them from objects that are also referenced

    In other words, if a `UObject` does not have a `UPROPERTY` reference, there is a good chance it will be garbage collected away.
    `UActor`s and `UActorComponent`s are an e exception this rule, since the Actor is referenced by the `UWorld` (which is in the root set),
    and the Actor Component is referenced by the Actor itself.
    
    Another practical implication of this is that once the owning `UActor` is destroyed, usually all of its `UActorComponent`s will be destroyed because
    the Actor was the only `UObject` with a strong reference to them.

2. By calling `UObject::AddReferencedObjects` from objects that are also referenced

3. By adding them to the root set with `UObject::AddToRoot`

    ```c++
    // Runtime/CoreUObject/Public/UObject/UObjectBaseUtility.h
    
    /**
     * Add an object to the root set. This prevents the object and all
     * its descendants from being deleted during garbage collection.
     */
    void UObjectBaseUtility::AddToRoot()                           // FORCEINLINE for performance benefits                                     
    {                                                              // GUObjectArray is the global array of all UObjects   
        GUObjectArray.IndexToObject(InternalIndex)->SetRootSet();  // Use the int32 InternalIndex belonging to UObjectBase to index into GUObjectArray
    }                                                              // Set RootSet flag for object
    ```
    
    ```c++
    // Runtime/CoreUObject/Public/UObject/UObjectArray.h
    
    void FUObjectItem::SetRootSet()                                 // FUObjectItem represents the UObject in the global array, it is a struct that contains a pointer to a UObject
    {
        ThisThreadAtomicallySetFlag(EInternalObjectFlags::RootSet); // Sets a flag which lets the garbage collector know not to garbage collect EVEN if unreferenced
    }
    ```


##### Automatic Property Initialization

`UObject`s properties are automatically initialized to zero before the constructor is called.

##### Factory Methods

* `NewObject()` is the simplest `UObject` factory method
* `NewNamedObject()` is the same as `NewObject()` but with an `FName`
* `ConstructObject()` is a more flexible factory method

##### Automatic Updating of References

`UObject`s marked with `UPROPERTY()` or stored in Unreal container classes are visible to the reflection system, and thus are automatically
nulled when the object is destroyed. This is because null-checking is more reliable than non-null pointers pointing at deleted memory.

##### Serialization



#### UActor

The word "Actor" is not a term unique to Unreal Engine, and can be found even in Nvidia's PhysX. It is the base class for all objects that can be placed in the world. Specifically, it is a
`UObject` with a transform.

#### Composition OOP Design Pattern - Components

Composition is a common object-oriented programming design pattern that defines reusable behavior and expresses has-a relationships instead
of is-a relationships. [Design Patterns: Elements of reusable Object-Oriented Software (1994) by Gang of Four](https://en.wikipedia.org/wiki/Design_Patterns) elaborates on this design pattern, I highly recommend
reading this book as Unreal Engine uses other patterns from this book. You will find that containing gameplay functionality within components rather than the larger gameplay classes prevents tarballing files into
a mess of tightly coupled code that takes longer to compile and harder to maintain due to everything depending on each other.

The base class for components is the `UActorComponent`.

##### Table of Components

| Component                     | Description                                                                                                                                                                                                                         |
|-------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| ActorComponent                | Every component inherits from this class.                                                                                                                                                                                           |
| ApplicationLifecycleComponent | For handling notifications received from the OS about the application state (low power mode, temperature changed, received startup, activated, suspended, termination, etc). Most of these notifications are sent via CoreDelegates |
| UArrowComponent               | A PrimitiveComponent (which means it renders unlike ActorComponent) that renders a simple arrow using lines. Can be used to indicate which way an object is facing.                                                                 |
| UAudioComponent               |
| UBillboardComponent
| UBoundsCopyComponent
| UBoxComponent
| UBoxReflectionCaptureComponent
| UBrushComponent
| UCapsuleComponent
| UChildActorComponent
| UDecalComponent
| UDirectionalLightComponent
| UDrawFrustumComponent
| UDrawSphereComponent
| UExponentialHeightFogComponent
| UForceFeedbackComponent
| UHierarchicalInstancedStaticMeshComponent
| UInputComponent
| UInstancedStaticMeshComponent
| UInterpToMovementComponent
| ULightComponent
| ULightComponentBase
| ULightmassPortalComponent
| ULineBatchComponent
| ULocalLightComponent
| ULODSyncComponent
| UMaterialBillboardComponent
| UMeshComponent
| UModelComponent
| UPawnNoiseEmitterComponent
| UPlanarReflectionComponent
| UPlaneReflectionCaptureComponent
| UPlatformEventsComponent
| UPointLightComponent
| UPoseableMeshComponent
| UPostProcessComponent
| UPrimitiveComponent           | An ActorComponent that can be rendered.                                                                                                                                                                                             |
| URectLightComponent
| UReflectionCaptureComponent
| URuntimeVirtualTextureComponent
| USceneCaptureComponent
| USceneCaptureComponent2D
| USceneCaptureComponentCube
| USceneComponent               | An ActorComponent that has a transform.                                                                                                                                                                                             |
| UShapeComponent
| USkeletalMeshComponent
| USkinnedMeshComponent
| USkyAtmosphereComponent
| USkyLightComponent
| USphereComponent
| USphereReflectionCaptureComponent
| USplineComponent
| USplineMeshComponent
| USpotLightComponent
| UStaticMeshComponent
| UStereoLayerComponent
| UTextRenderComponent
| UTimelineComponent
| UVectorFieldComponent
| UVolumetricCloudComponent
| UWindDirectionalSourceComponent
| UWorldPartitionStreamingSourceComponent

### Real-Time Agent-Based Simulation

### Event/Messaging System

### World Loading / Streaming


## Game-Specific Subsystems Layer

### Game Specific Rendering

#### Terrain Rendering

#### Water Simulation & Rendering

### Player Mechanics

#### State Machine & Animation

##### Gameplay Ability System

The Gameplay Ability System (GAS) is a framework to streamline the complex logic that goes into replication, canceling, casting, granting, and blocking of abilities. Without GAS, you would
have to use an increasingly unmaintainable spangle of conditional flag checking, timers, state machines, and RPC calls to implement abilities. The GAS pattern comes from World of Warcraft, an
obvious heavy user of abilities at scale.

###### Gameplay Effects

###### Gameplay Tags

Although Gameplay Tags is not exclusive to GAS, it handles the "conditional flag checking" of abilities. It is a simple system that allows you to tag objects with arbitrary hierarchical strings.
An object can have both the Damage.Fire and Damage.Fire.Fireball tags.

#### Camera-Relative Controls (HID) (+Game Cameras)

#### Collision Manifold

#### Movement

### Game Cameras

#### Fixed Cameras

#### Scripted/Animated Cameras

#### Player-Follow Camera

#### Debug Fly-Through Cam

### AI

#### Goals & Decision-Making

#### Actions (Engine Interface)

#### Sight Traces & Perception

#### Pathfinding (A*)

## Game Layer (Game Code)

**Note:** This is not part of the engine, but rather built on top of it. Including because it can be helpful and a good wrap up of the engine in practical usage

### RTS & TBS

#### Fog of War

### FPS, TPS, & RPG

### Weapons

### Abilities


# Editor Architecture - Unreal Editor

### Skeletons and Animation

#### Skeleton Editor

#### Skeletal Mesh Editor

#### Animation Editor

#### Animation Blueprint Editor

#### Physics editor
