## Preface

Wanting a fast way to learn the insides of Unreal Engine, I inquired the UE dev community regarding resources for a deep dive into Epic Game's Unreal Engine architecture beyond stepping through the source code or reading the little information provided in the official documentation. To my dismay, someone responded there aren't many resources like that around. So
begins my journey to understand the engine, write notes on my findings, and share it.

Although there may not be many resources on unraveling the engine code, there are a plethora of resources that help in
understanding the engine. [Game Engine Architecture by Jason Gregory](https://www.goodreads.com/book/show/6709076-game-engine-architecture)
is a great resource on understanding Unreal Engine, and game engines in general.

## Runtime Engine Architecture

Unreal Engine, like all software systems and game engines, is built in layers. In order to
maintain modularity and avoid circular dependencies, the lower layers do not depend on
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

To keep the project modular, many Engine features are separated out into optional Plugins, while core components are under `Source`.

## Entry Point

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

## Main Engine Loop

It is a very simple while loop.

```cpp
while( !IsEngineExitRequested() )
{
    EngineTick();
}
```

## Blueprint

All the graph editor tools are behind the scenes a `UEdGraph`. This includes Blueprint, Material, and Animation graphs.
The `UEdGraph` is a simple graph data structure that listeners on every node.

The all the graphs for a Blueprint, such as the Event Graph, are combined into an Ubergraph.

## Visual Effects

## Resource Manager

Unreal Engine's highly centralized resource manager is a unified interface to access all types of game assets. This includes `umap` and
`uasset` files.

## Target Hardware

This layer is platform-specific, such as optimizations made for different computer or console systems.

## Collision & Physics

This layer of the engine handles collision detection and rigid body dynamics (which is where it gets "physics" from). Calling it physics is a bit of a
misnomer, as the engine is primarily concerned about forces and torques acting on rigid bodies, and not much of anything else.

Typically, game engines do not implement their own physics engine. Instead, they use SDKs from a third-party physics & collision engine. Hence, Unreal Engine 
uses Nvidia's PhysX SDK, which is a free and open source. It does however have some of custom implementation such as `ARigidBodyBase`. It does not use Havok or Open Dynamics Engine.

Unreal Engine uses the `PxRigidActor` class from PhysX's `physx` namespace to represent dynamic and static rigid bodies.

Relevant folders are `Runtime/PhysicsCore` and `Runtime/Engine/PhysicsEngine`.

## UObject

All objects in the engine are derived from this class.

## UActor

The word "Actor" is not a term unique to Unreal Engine, and can be found even in Nvidia's PhysX. It is the base class for all objects that can be placed in the world. Specifically, it is a
`UObject` with a transform.

## Components

Composition is a common object-oriented programming design pattern that defines reusable behavior and expresses has-a relationships instead
of is-a relationships. You will find that containing gameplay functionality within components rather than the larger gameplay classes prevents tarballing into
a mess of tightly coupled code that takes longer to compile and harder to maintain.

The base class for components is the `UActorComponent`.

### Table of Components

| Component                                  | Description                                                                                                                                                                                                                         |
|--------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `UActorComponent`                          | Every component inherits from this class.                                                                                                                                                                                           |
| `UApplicationLifecycleComponent`           | For handling notifications received from the OS about the application state (low power mode, temperature changed, received startup, activated, suspended, termination, etc). Most of these notifications are sent via CoreDelegates |
| `UArrowComponent`                          | A PrimitiveComponent (which means it renders unlike ActorComponent) that renders a simple arrow using lines. Can be used to indicate which way an object is facing.                                                                 |
| `UAudioComponent`                          | 
| `UBillboardComponent`                       
| `UBoundsCopyComponent`                      
| `UBoxComponent`                             
| `UBoxReflectionCaptureComponent`            
| `UBrushComponent`                           
| `UCapsuleComponent`                         
| `UChildActorComponent`                      
| `UDecalComponent`                           
| `UDirectionalLightComponent`                
| `UDrawFrustumComponent`                     
| `UDrawSphereComponent`                      
| `UExponentialHeightFogComponent`            
| `UForceFeedbackComponent`                   
| `UHierarchicalInstancedStaticMeshComponent` 
| `UInputComponent`
| `UInstancedStaticMeshComponent`
| `UInterpToMovementComponent`
| `ULightComponent`
| `ULightComponentBase`
| `ULightmassPortalComponent`
| `ULineBatchComponent`
| `ULocalLightComponent`
| `ULODSyncComponent`
| `UMaterialBillboardComponent`
| `UMeshComponent`
| `UModelComponent`
| `UPawnNoiseEmitterComponent`
| `UPlanarReflectionComponent`
| `UPlaneReflectionCaptureComponent`
| `UPlatformEventsComponent`
| `UPointLightComponent`
| `UPoseableMeshComponent`
| `UPostProcessComponent`
| `UPrimitiveComponent`                      | An ActorComponent that can be rendered.                                                                                                                                                                                             |
| `URectLightComponent`
| `UReflectionCaptureComponent`
| `URuntimeVirtualTextureComponent`
| `USceneCaptureComponent`
| `USceneCaptureComponent2D`
| `USceneCaptureComponentCube`
| `USceneComponent`                           | An ActorComponent that has a transform.                                                                                                                                                                                             |
| `UShapeComponent`
| `USkeletalMeshComponent`
| `USkinnedMeshComponent`
| `USkyAtmosphereComponent`
| `USkyLightComponent`
| `USphereComponent`
| `USphereReflectionCaptureComponent`
| `USplineComponent`
| `USplineMeshComponent`
| `USpotLightComponent`
| `UStaticMeshComponent`
| `UStereoLayerComponent`
| `UTextRenderComponent`
| `UTimelineComponent`
| `UVectorFieldComponent`
| `UVolumetricCloudComponent`
| `UWindDirectionalSourceComponent`
| `UWorldPartitionStreamingSourceComponent`
