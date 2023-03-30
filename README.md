## Preface

There are not many resources for a deep dive into Epic Game's Unreal Engine architecture beyond stepping through the source code or reading the little information provided in the official documentation.
This should help fill in the gaps for those who are interested in learning more about the engine.

Although there may not be many resources on unraveling the engine code, there are however a plethora of resources that help in
understanding game engines in general. [Game Engine Architecture by Jason Gregory](https://www.goodreads.com/book/show/6709076-game-engine-architecture)
is a great resource that will help you understand Unreal Engine.

**Important for new readers:** This documentation goes from lower-layer to upper-layer parts of the engine. So if you want game-related information, you should start at the bottom of the document.
Otherwise, start from the top, as understanding the lower layers will help you understand the upper layers.

## Two Parts

Unreal Engine can be broken into two important components: the Editor and the Runtime Engine. The Editor is the suite of tools used to create and edit content for the game. The Runtime Engine is the part that runs the game.

Note: Unreal Engine (taken from Quake Engine architecture) is different from most other engines in that the tool suite (UnrealEd) is built into the runtime engine. 

## Runtime Engine Architecture

Unreal Engine, like all software systems and game engines, is built in layers. In order to
maintain platform independence, modularity, and avoid circular dependencies, the lower layers do not depend on
upper layers.

The upper-most layers contain the well-known `GameFramework` classes containing `PlayerController` and
`GameModeBase`. The lower layers contain hardware-specific implementation such as `Runtime/Unix`.

From top to bottom, the layers are:
* GameFramework, Rendering, Collision & Physics, Animation, AI, HID Audio, Input
* Resources (Resource Manager)
* Core Systems
* Platform Independence Layer (Networking, File System)
* 3rd Party SDKs (DirectX, OpenGL, PhysX)
* OS
* Drivers
* Hardware

To keep the project modular, many features such as the Gameplay Ability System are separated out into optional Plugins, while core components are under `Source`.

## Hardware Layer

This layer is platform-specific, such as optimizations made for different computer or console systems.

### Entry Point

The entry point for the engine depends on the platform. Every Windows program has an entry-point function called `WinMain`.
Unreal Engine's entry point for Windows, like all other game engines, is the `WinMain` function defined in `Windows/LaunchWindows.cpp`.
The [Quake 2 engine](https://github.com/id-Software/Quake-2/blob/master/win32/sys_win.c#L594), for example, also has the
identically named function.

Each supported platform has their respective entry point:
* MacOS: `INT32_MAIN_INT32_ARGC_TCHAR_ARGV` in `Mac/LaunchMac.cpp`
* Linux: `int main` in `Linux/LaunchLinux.cpp`
* IOS: `int main` in `IOS/LaunchIOS.cpp`

```cpp
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

```cpp
while( !IsEngineExitRequested() )
{
    EngineTick();
}
```

### Xbox

### Playstation

### Nintendo

### Mobile

### Web

### VR

## Drivers Layer

## 3rd Party SDKs Layer

### Graphics

#### DirectX

#### Vulkan

#### OpenGL

## Platform Independence Layer

### Networking

### File System

### Threading

### Graphics Wrappers

### Physics & Collision Wrappers

### Hi-Res Timer

### Collections & Iterators

### Primitive Data Types

### Platform Detection

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

```cpp
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

## Blueprint

All the graph editor tools are behind the scenes a `UEdGraph`. This includes Blueprint, Material, and Animation graphs.
The `UEdGraph` is a simple graph data structure that listeners on every node.

The all the graphs for a Blueprint, such as the Event Graph, are combined into an Ubergraph.

## Visual Effects Layer

## Resources (Game Assets) Layer

### UnrealEd

### Resource Manager

Unreal Engine's highly centralized resource manager is a unified interface to access all types of game assets. This includes `umap` and
`uasset` files.

## Collision & Physics

This layer of the engine handles collision detection and rigid body dynamics (which is where it gets "physics" from). Calling it physics is a bit of a
misnomer, as the engine is primarily concerned about forces and torques acting on rigid bodies, and not much of anything else.

Typically, game engines do not implement their own physics engine. Instead, they use SDKs from a third-party physics & collision engine. Hence, Unreal Engine 
uses Nvidia's PhysX SDK, which is a free and open source. It does however have some of custom implementation such as `ARigidBodyBase`. It does not use Havok or Open Dynamics Engine.

Unreal Engine uses the `PxRigidActor` class from PhysX's `physx` namespace to represent dynamic and static rigid bodies.

Relevant folders are `Runtime/PhysicsCore` and `Runtime/Engine/PhysicsEngine`.

## Gameplay Foundations Layer

### Gameplay Ability System

The Gameplay Ability System (GAS) is a framework to streamline the complex logic that goes into replication, canceling, casting, granting, and blocking of abilities. Without GAS, you would
have to use an increasingly unmaintainable spangle of conditional flag checking, timers, state machines, and RPC calls to implement abilities. The GAS pattern comes from World of Warcraft, an
obvious heavy user of abilities at scale.

#### Gameplay Tags

Although Gameplay Tags is not exclusive to GAS, it handles the "conditional flag checking" of abilities. It is a simple system that allows you to tag objects with arbitrary hierarchical strings.
An object can have both the Damage.Fire and Damage.Fire.Fireball tags.

#### Gameplay Effects

## Tool Suite - Unreal Editor

### Skeletons and Animation

#### Skeleton Editor

#### Skeletal Mesh Editor

#### Animation Editor

#### Animation BLueprint Editor

#### Physics editor

### Gameplay Foundation Layer

#### UObject

All objects in the engine are derived from this class.

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