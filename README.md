## Preface

There are not many resources for a deep dive into Epic Game's Unreal Engine architecture beyond stepping through the
source code or reading the little information provided in the official documentation.
This should help fill in the gaps for those who are interested in learning more about the engine.

## History

It is important to start off with the context and history of Unreal Engine. Tim Sweeney took significant inspiration from Quake 1 (1996) and Quake 2 (1997) before he released Unreal (1998). Thus, most of the foundational architecture for Unreal Engine is very similar to Quake. In fact, this quote sums it up

> "This is probably going to come out sounding demeaning, but Epic wants Unreal to be Quake. Everything they did with Unreal, they did because they wanted it to be like what Quake turned out to be." - [John Carmack, creator of Quake, id Software](http://advsys.net/ken/carmken.htm) 

**Unreal shipped as a first person shooter game.** This is important. All the networking, rendering, and the rest of the architecture was developed with an FPS game in mind. Although it did a much better job than Quake in adding in flexibility for other genres (e.g. Quake rendering favored dark-lit corridor gameplay), and although Epic Games will market Unreal Engine as supporting all genres, it is a fact that Unreal Engine was originally and currently still is optimized for first person shooters and genres similar to it. Take client-side prediction for example, or its decision for UDP networking, these are optimizations that greatly improve the experience of FPS and TPS games, but not nearly as much for RTS or TBS games.

Thus, a good resource for understanding the Unreal Engine architecture, is in fact the [Quake source code and architecture](https://github.com/id-Software/Quake-2).

## Two Parts

Unreal Engine can be broken into two important components: the Editor and the Runtime Engine. The Editor is the suite of tools used to create and edit content for the game. The Runtime Engine is the part that runs the game.

Unlike most other game engines, Unreal Engine and Quake Engine has the tool suite (`UnrealEd`) built directly into the runtime engine. There are a lot of benefits for this architectural decision, most importantly that the game can run via PIE (Play in Editor) without performance impacts. This also allows loading asset contents and seeing them in their full glory. Furthermore, this reduces code duplication between the two since the Editor is directly using the runtime code. There are drawbacks however, such as in developer productivity due to the locking of files preventing simultaneous editing of assets. More on this later.

# Runtime Engine Architecture

Unreal Engine, like all software systems and game engines, is built in layers. Generally, the lower layers do not depend on the upper layers. This prevents circular dependencies and provides modularity that allows for cross-platform support. One of the biggest benefits is that this makes the code more testable.

The upper-most layers contain the well-known `GameFramework` classes containing `PlayerController` and
`GameModeBase`. The lower layers contain platform-specific implementations such as `Runtime/Unix`.

From top to bottom, the layers are:

* Game-Specific Subsystems
* Gameplay Foundations, Rendering, Profiling & Debugging, Scene Graph / Culling, Visual Effects, Front End, Skeletal
  Animation Collision & Physics, Animation, AI, HID Audio, Input
* Resources (Resource Manager)
* Core Systems
* Platform Independence Layer (Networking, File System)
* 3rd Party SDKs (DirectX, OpenGL, PhysX)
* OS
* Drivers
* Hardware

```mermaid
flowchart TD
    A[Game-Specific Subsystems]
    subgraph B[Gameplay Foundations]
        B1[Rendering]
        B2[Profiling & Debugging]
        B3[Scene Graph / Culling]
        B4[Visual Effects]
        B5[Front End]
        B6[Skeletal Animation]
        B7[Collision & Physics]
        B8[Animation]
        B9[AI]
        B10[HID Audio]
        B11[Input]
    end
    C[Resources: Resource Manager]
    D[Core Systems]
    E[Platform Independence Layer: Networking, File System]
    F[3rd Party SDKs: DirectX, OpenGL, PhysX]
    G[OS]
    H[Drivers]
    I[Hardware]
    A --> B
    B --> C
    C --> D
    D --> E
    E --> F
    F --> G
    G --> H
    H --> I
```
    


To keep the project modular, many features within these layers (e.g. Replication Graph, Gameplay Ability System) are
separated out into optional Plugins.

## Target Hardware Layer

This layer is platform-specific. Generally, Unreal Engine is platform-agnostic, but there are some platform-specific
code and optimizations for different computer or console systems.
The Quake 2 engine, for example, had
significant [optimizations made for the Intel's Pentium processor and its pre-fetching cache](https://fabiensanglard.net/quake2/quake2_software_renderer.php)
due to their popularity at the time.

### Apple

### Xbox

### Playstation

### Nintendo

### Mobile

### Web

### VR

## Drivers Layer

Drivers manage hardware resources and provide an interface (abstraction) for the operating system to interact with the
myriad variants of hardware devices.

## Operating System Layer

This part of Unreal Engine handles the various operating systems which share hardware resources between multiple
applications, one being your game. Unlike consoles of old
where a game could "own" the entire device and assume full control of memory
and compute resources, modern consoles and modern operating systems employe preemptive multitasking and can have multiple applications
running alongside your game (e.g. Xbox Live, Netflix, Voice Chat, store downloads) that take over certain system resources or
pause the game entirely (Xbox Dashboard). At that time, a layer like this was nonexistent or at most limited to a
library that directly access hardware resources.

Some reasons why this layer exists:

* Implement memory access and tracking for each platform.
* Obtain platform properties regarding features that are supported (e.g. Texture Streaming, High Quality Light Maps,
  Audio Streaming)
* Access (and wrap functions for) platform native APIs (e.g. Atomics, File I/O, Time)
* Execute general platform commands (e.g. get orientation of screen, get network type)
* Provide platform-specific implementations of OS functions (e.g. `FPlatformProcess::Sleep`
  , `FPlatformProcess::LaunchURL`)

`FGenericPlatformMisc` and `FGenericPlatform` are examples of OS layer classes.

### Windows

#### The Window

The engine starts with a window. The operating system provides the fundamental functionality, such as the APIs (e.g. Windows API) and the graphical subsystem
for creating and managing a windows. The graphical subsystem refers to a stack of software and hardware components responsible for rendering the graphics on
your screen. The window manager handles the drawing, positioning, resizing, and user interactions with the window. The Graphics Device Interface (GDI) is what creates
the graphical content by communicating with the graphics drivers by sending commands. The graphics drivers translate the commands from the GDI into instructions
the graphics hardware can understand. And finally, the the graphics hardware is the physical component that renders the graphics on the screen by creating a
frame buffer, applying algorithms to each pixel on the frame buffer, and display the final image.

#### Entry Point

The entry point for the engine depends on the platform. Every Windows program has an entry-point function
called `WinMain`.
Unreal Engine's entry point for Windows, like all other game engines, is the `WinMain` function defined
in `Windows/LaunchWindows.cpp`.
The [Quake 2 engine](https://github.com/id-Software/Quake-2/blob/master/win32/sys_win.c#L594), for example, also has the
identically named function.

Each supported platform has their respective entry point:

* MacOS: `INT32_MAIN_INT32_ARGC_TCHAR_ARGV` in `Mac/LaunchMac.cpp`
* Linux: `int main` in `Linux/LaunchLinux.cpp`
* IOS: `int main` in `IOS/LaunchIOS.cpp`

```c++
// Launch/Private/Windows/LaunchWindows.cpp

// Windows specific parameters: HINSTANCE is identification to prevent class name clashing
int32 WINAPI WinMain(_In_ HINSTANCE hInInstance, _In_opt_ HINSTANCE hPrevInstance, _In_ char* pCmdLine, _In_ int32 nCmdShow)
{
	int32 Result = LaunchWindowsStartup(hInInstance, hPrevInstance, pCmdLine, nCmdShow, nullptr); // Launch Unreal Engine
	LaunchWindowsShutdown(); 
	return Result; // 0 on success, error level otherwise
}
```

#### Main Engine Loop

It is a very simple while loop.

```c++
// Runtime/Launch/Private/Launch.cpp

while( !IsEngineExitRequested() )
{
    EngineTick();
}
```

#### Table of Files


| File                    | Description                                                                                                                                                                                          |
|-------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| WindowsPlatform.h       | Abstracts away Windows platform-specific details (compiler, sdk versions, etc.) and provides a consistent interface for the rest of the code base. Specifically defines types, settings, and macros. |


### MacOS & iOS

Unreal Engine interfaces with Apple platforms in `Runtime/Core/Apple` with the help of Apple's Core Foundation (CF) SDK.
Core
Foundation is an API for C used for its operating systems, providing primitive data types and wrapper functions (file
I/O, network I/O).

#### Table of Files

| File                            | Description                         |
|---------------------------------|-------------------------------------|
| AppleLLM.h                      | Low-Level Memory Tracker            |
| ApplePlatformAffinity.h         | -                                   |
| ApplePlatformAtomics.h          | Wrap Apple atomic implementations   |
| ApplePlatformCompilerPreSetup.h | -                                   |
| ApplePlatformCrashContext.h     | Crash context                       |
| ApplePlatformDebugEvents.h      | -                                   |
| ApplePlatformFile.h             | Wrap Apple File I/O implementations |
| ApplePlatformMemory.h           | Memory allocation and tracking      |
| ApplePlatformMisc.h             | -                                   |
| ApplePlatformRunnableThread.h   | -                                   |
| ApplePlatformStackWalk.h   | -                                   |
| ApplePlatformString.h   | -                                   |
| ApplePlatformTime.h   | Wrap Apple Time implementations     |
| ApplePlatformTLS.h   | -                                   |
| CFRef.h   | -                                  |
| PostAppleSystemHeaders.h   | Preserve macros                     |
| PreAppleSystemHeaders.h   | Preserve macros                     |

```c++
// Core/Private/Apple/ApplePlatformMemory.cpp

// In this file, we keep track of the amount of memory we've allocated for an Unreal app running on an Apple device.

#include <stdlib.h>                          // c standard library
#include <objc/runtime.h>                    // for inspecting and manipulating Objective-C runtime data structures
#if PLATFORM_IOS && defined(__IPHONE_13_0)   // Include only for iPhone 13+
#include <os/proc.h>                         // in order to call os_proc_available_memory which determines the amount of memory available to the current app (your game running on the iPhone)
#endif                                       // Only need to include one header specific to iOS 13+.
#include <CoreFoundation/CFBase.h>           // Types used from Core Foundation: CFIndex is a typedef for a signed integer type (SInt32) used to represent indices into a CFArray or CFString
                                             // CFOptionFlags is a typedef for an unsigned integer type (UInt32) used to represent bitfields for passing special allocations into CF funcs.
                                             // CFAllocatorContext is a struct containing callbacks for allocating, deallocating, and reallocating memory, and for retaining and releasing objects.

#include "HAL/LowLevelMemTracker.h"          // for FLowLevelMemTracker in order to track memory allocations
#include "Apple/AppleLLM.h"                  // Apple's Low-Level Memory Tracker which tracks all allocations from the OS

// Skip ~250 lines including functions for memory allocation

FMalloc* FApplePlatformMemory::BaseAllocator()                        
{
#if ENABLE_LOW_LEVEL_MEM_TRACKER
	FPlatformMemoryStats MemStats = FApplePlatformMemory::GetStats(); // FPlatformMemoryStats is the Apple implementation of FPlatformMemoryStats which contains memory numbers on available/used physical/virtual memory
	FLowLevelMemTracker::Get().SetProgramSize(MemStats.UsedPhysical);
#endif
```

### Linux

## 3rd Party SDKs Layer

Unreal Engine leverages a number of third-party software development kits (SDKs) including:

* Nvidia SDKs
    * CUDA (Compute Unified Device Architecture) - API for using GPUs for general purpose
      computing `ThirdParty/NVIDIA/CUDA`
    * GeForce NOW - Cloud gaming service `Plugins/Runtime/Nvidia/GeForceNOWWrapper`
    * GPUDirect - Direct data exchange with Nvidia GPUs `ThirdPartyNVIDIA/GPUDirect`
* Python - For enabling developers to create editor widgets `ThirdParty/Python3`
* Steamworks - For enabling Steam online services
* Oculus - Oculus VR support
* WebRTC - Standing for Web Real-Time Communication, it is a technology that uses web sockets to enable real-time
  communication
  between web browsers and mobile applications, without the need for a plugin or external app. This enables seamless
  streaming of
  video, audio, and data which enables video conferences, augmented reality, and online gaming. In the case for Unreal
  Engine,
  it is used extensively for Pixel Streaming. Using a client-server model instead of peer-to-peer, Pixel Streaming video
  encodes rendered Unreal Engine content and audio running on a server
  and streams it to connected web browsers or mobile applications for decoding without powerful hardware
  client-side. `ThirdParty/WebRTC`
* SpeedTree - For generating and rendering trees

Their respective source code and pre-built `.lib` (`.a` for Linux) files are located in their corresponding folders. However, they are
not viewable in your solution explorer until you generate the project files adding the `-THIRDPARTY` flag to
the `GenerateProjectFiles.bat` The `.lib` files are intermediate libraries of object files (intermediate sdk source code
files) produced as part of the compilation process that are later used to link to the final executable.

### Graphics

As Unreal Engine tries to be platform-agnostic, it lets graphics APIs like DirectX and OpenGL handle the abstraction of low-level
GPU hardware communication.

#### DirectX

Microsoft's 3D graphics API. SDKs for DirectX 9, 11, and 12 are found under `ThirdParty/Windows/DX9`
, `ThirdParty/Windows/DX11`, and `ThirdParty/Windows/DX12`.

These SDKs are primarily used for DirectX RHI implementations, some others include shader compilation. Compared to OpenGL,
it provides a higher level of abstraction and leverages Microsoft hardware optimizations.

#### Vulkan

Khronos Group's Vulkan provides a lower-level API compared to DirectX and OpenGL, enabling more efficient use of the GPU. In addition, it allows sending GPGPU (General Purpose GPU) commands to the GPU.

#### OpenGL

Portable 3D graphics SDK.

### Physics & Collision

#### Nvidia PhysX

## Platform Independence Layer

Unreal Engine's Platform Independence Layer is called the **Hardware Abstraction layer (HAL).** Everything
under `Runtime/Core/Public/HAL` falls under this layer.

### Platform Detection

`Platform.h` defines multiple header guards for different platforms, such as `PLATFORM_CPU_X86_FAMILY` for x86
processors, `PLATFORM_CPU_ARM_FAMILY` for ARM processors, and `PLATFORM_APPLE` for Apple devices.
The `FPlatformAtomics` class contains platform-specific implementations of atomic operations.

### Primitive Data Types

### Collections & Iterators

### File System

### Networking

The original Unreal game shipped with its multiplayer networking layer built on top of the User Datagram Protocol (UDP) as the chosen transport layer, taking inspiration from its competitor Quake. This is due to UDP's lower latency and the fine-grain control it provides to the developer compared to TCP. UDP packets contain less headers and do not require an established connection. Furthermore, UDP does not guarantee packet delivery, nor the ordering of packets. While this reduces reliability of UDP, this also reduces the overhead in sending packets.

Whereas for TCP, dropped packets are retransmitted and packets contain additional headers to maintain ordering. The additional overhead this creates is unacceptable for games like FPS games with low tolerance for delay. One of the biggest issues with using TCP for FPS games for example, is that TCP will retransmit dropped low-priority packets even if it means delaying high-priority packets. For example, a packet for shooting a sniper rifle might be delayed because of a dropped VOIP packet once TCP retransmits it and waits for successful acknowledgement.

Unreal Engine implements its own custom networking protocol called Unreal Datagram Protocol (UDPG) built on top of User Datagram Protocol (UDP) that provides reliability to the unreliable data you get with UDP due to dropped and out-of-order packets.

The Tribes model is very similar to UE networking model, reading this paper should give you a good idea. https://archive.org/details/tribes-networking-model.

Relevant files:
* UdpMessaging/Private/Transport/UdpMessageProcessor.h

#### Client Side Prediction

It is good to provide context and some history here. The original Quake 1 released with the client-server network topology and "Dumb Terminal" networking model where essentially all the simulation was taken care of server-side. There was no client-side prediction, dead reckoning, nor any other kind of lag compensations made to improve the player experience. Although this reduced vectors for cheating, this creates a poor player experience. This meant that when a player moves his avatar, the following sequence of events occur:

1. His local game sends a packet to the server containing the move input
2. The packet takes 50ms (hypothetical) to get to the server
3. The server receives the client packet and updates the avatar's position
4. The server sends a packet to the client with the updated avatar position
5. The packet takes 50ms (hypothetical, both ends usually are not identical) to get to the client
6. The client receives the packet with the latest avatar position and updates the avatar state with the new location

By then, it has been a 100ms round-time trip (RTT) before the client can even update its position in the game. To make matters worse, the client is now 50ms (1/2 RTT) behind the server. This creates a noticeable delay for the player, who might be running at 60fps (16.7ms per frame). By the time the player sees his avatar move, he would have rendered 6 frames already!

For a Quake 1 multiplayer update called Quake World (1996), [John Carmack made significant improvements to the player experience](https://fabiensanglard.net/quakeSource/johnc-log.aug.htm) that would change multiplayer gaming forever and influence the networking model of games to this day (including Unreal Engine). He introduced client-side prediction:

Instead of waiting for the full RTT (e.g. 100ms) to update the local avatar, the client will run a local simulation of the movement action. Then, the client makes appropriate adjustments when receiving the authoritative packet from the server by interpolating between the two positions. Since the client side code for movement is usually identical to that of the server-side, most of the time, this client side prediction is pretty accurate and does not stray too far off from the server.

Come Unreal in May 1998, employed very similar architecture to Quake.

#### Pixel Streaming

Pixel Streaming uses WebRTC to stream rendered Unreal Engine content over the internet to connected remote clients
in-real time via a server-client model. A server on the cloud (e.g. AWS Tesla T4 gpu instance) can run a headless Unreal
Engine,
and use Nvidia's NVENC to encode the rendered frames and audio into H.264. Then stream it to clients that will then
decode the frames and audio to display them to the client's screen without the need for a plugin, external app,
or powerful hardware client-side.


### Hi-Res Timer

### Threading Library

Unreal Engine improves performance of its applications by taking advantage of the multi-core architecture of modern CPUs
with the use of a threading library.

#### Thread

A thread is a theoretical concept representing a component of a process. A process is a unit of resources, while a
thread is a unit of scheduling and execution. You can analogize it to a cotton thread, a long thin strand of cotton
fibers used for sewing, except the
cotton fibers are sequential instructions. This thread includes the scheduling and execution of these instructions by
the CPU. You can consider the execution of the functions in your game as a thread.

The API for system threads is located in `Runtime/Core/Public/HAL/Thread.h`.

##### Usage

* Create threads using the constructor

```c++
// Runtime/Core/Public/HAL/Thread.h

/**
* Creates and immediately starts a new system thread that will execute `ThreadFunction` argument.
* Can return before the thread is actually started or when it already finished execution.
* @param ThreadName Name of the thread
* @param ThreadFunction The function that will be executed by the newly created thread
* @param StackSize The size of the stack to create. 0 means use the current thread's stack size
* @param ThreadPriority Tells the thread whether it needs to adjust its priority or not. Defaults to normal priority
* @param ThreadAffinity Tells the thread whether it needs to adjust its affinity or not. Defaults to no affinity
* @param IsForkable Tells the thread whether it can be forked. Defaults to NonForkable
*/
FThread(
    TCHAR const* ThreadName,
    TUniqueFunction<void()>&& ThreadFunction,
    uint32 StackSize = 0,
    EThreadPriority ThreadPriority = TPri_Normal,
    FThreadAffinity ThreadAffinity = FThreadAffinity(),
    EForkable IsForkable = NonForkable
);
```

```c++
FThread Thread = FThread(
    TEXT("MyThreadWithSingleton),                        // Give any name
    []()                                                 // Since a thread is a sequence of instructions, we pass a function for the new thread to execute
    {                                                    // One-time use anonymous lambda function
        DoWork();
    });
Thread.Join();
```

##### Program Stack

The `FThread`'s `StackSize` parameter for the constructor refers to the size of a key component of a thread, the
thread's _Program Stack_.

The stack data structure is a container of contiguous blocks of memory (analogous to a stack of plates), where only the
top of the stack is accessible and needs to be removed (popped) before the block below it
can be accessed. In other words, the first block pushed onto the stack is the last block to be popped off, and the last
block pushed onto the stack is the first block to be popped off (LIFO - Last In First Out).

Stack Diagram (_javabycode.com - Stack Data Structure in Java, easy in 5 minutes_):

![image](https://user-images.githubusercontent.com/11065634/229366965-473ed9d6-610f-4c17-9963-1c2a08d5327a.png)

This behavior of a stack is convenient for representing function calls because the nature of functions is that a
function may call (push onto the stack) other (nested) functions that it depends on, and as a result, a
particular function cannot complete execution until all its nested functions are completed (popped) first.

This stack for function calls is called the _Program Stack_, and each item on the stack (a block of memory) is called
a _stack frame_. Whenever a function is called (by another function), the operating system stores all local variables
declared in the
function and the contents of CPU registers for the function to utilize in this stack frame. The
return address for the called function is also stored in the stack frame because once the stack frame is popped
after the called function is returned, the caller needs to continue execution from where it left off.

Before a program is loaded onto memory and executed, the operating system needs to first reserve an area of memory for
the _program stack_. The _stack pointer_, the value of a single CPU register, is used to push and pop stack frames.

### Graphics Wrappers

### Physics & Collision Wrappers

## Core Systems Layer

### Data Structures & Algorithms

All software systems require containers that allow for the storage and manipulation of data and algorithms that use
these data structures to solve problems. Many projects use standard and
third-party libraries that provide implementations of these data structure and algorithms.

However, Unreal Engine neither uses STL, the C++ Standard Template Library, nor third-parties like Boost, Folly, or
Loki. It opts for a custom solution for performance benefits and cross-platform support. Although it isn't used anywhere in the Engine, many of the
custom implementations are inspired by Boost, a popular and powerful data
structures and algorithms C++ library that shares similar style to its predecessor STL. Some of the third party
libraries included by UE, such as SDKs made by Pixar, however do use Boost.

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

Unreal Engine's highly centralized resource manager is a unified interface to access all types of game assets. This
includes `umap` and
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

This layer of the engine handles collision detection and rigid body dynamics (which is where it gets "physics" from).
Calling it physics is a bit of a
misnomer, as the engine is primarily concerned about forces and torques acting on rigid bodies, and not much of anything
else.

Typically, game engines do not implement their own physics engine. Instead, they use SDKs from a third-party physics &
collision engine. Hence, Unreal Engine
uses Nvidia's PhysX SDK, which is a free and open source. It does however have some custom implementations such
as `ARigidBodyBase`. It does not use Havok or Open Dynamics Engine.

Unreal Engine uses the `PxRigidActor` class from PhysX's `physx` namespace to represent dynamic and static rigid bodies.

The Physics Engine implementation is in `Runtime/Engine/Private/PhysicsEngine`, `Runtime/Engine/Public/Physics`,
and `Runtime/Engine/PhysicsEngine`.

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

### Graphics Device Interface

Unreal Engine calls this the RHI (Render Hardware Interface). Utilizes graphics SDKs (DirectX, OpenGL, Vulkan) to
enumerate
available graphics devices, initialize them, and set up render surfaces (back buffer, stencil buffer, depth buffer,
etc.).

Found in `Runtime/RHI` and `Runtime/RHICore`.

#### Table of Files

| File | Description |
|------|-------------|
| BoundShaderStateCache.h | |
| DynamicRHI.h | |
| GPUDefragAllocator.h | |
| GPUProfiler.h | |
| GpuProfilerTrace.h | |
| MultiGPU.h | |
| PipelineFileCache.h | |
| PipelineStateCache.h | |
| PsoLruCache.h | |
| RHI.h | |
| RHIBreadcrumbs.h | |
| RHICommandList.h | |
| RHICommandList.inl | |
| RHICommandListCommandExecutes.inl | |
| RHIContext.h | |
| RHIDefinitions.h | |
| RHIGPUReadback.h | |
| RHIResources.h | |
| RHIShaderFormatDefinitions.inl | |
| RHIStaticStates.h | |
| RHISurfaceDataConversion.h | |
| RHITransientResourceAllocator.h | |
| RHIUtilities.h | |
| RHIValidation.h | |
| RHIValidationCommon.h | |
| RHIValidationContext.h | |
| RHIValidationTransientResourceAllocator.h | |
| RHIValidationUtils.h | |
| TextureProfiler.h | |

#### DirectX RHI

#### Vulkan RHI

#### OpenGL RHI

### Materials & Shaders

### Static & Dynamic Lighting

### Cameras

### Text & Fonts

### Primitive Submission

### Viewports & Virtual Screens

### Texture & Surface Management

### Debug Drawing (Lines etc.)

## Scene Graph / Culling Optimizations Layer

Generally, any sort of Spatial Subdivision is called a "Scene Graph". Common algorithms for spatial subdivision includes Octrees, Quadtrees, Binary Space Partitioning trees, k-d trees, and sphere hierarchical. Unreal Engine uses a Scene Outliner, a hierarchical scene representation using a tree structure, to perform optimizations such as frustum culling.

Culling can involve a simple algorithm such as frustum (the trapezoidal prism that represents your view into the world) culling or can leverage spatial partitioning algorithms to reduce geometry in which to do culling upon.

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

When `UObject`s are created, they are automatically added to a global array `GUObjectArray` which the GC uses for
tracking and deleting at regular intervals any unreferenced objects or
objects explicitly marked for destruction, unless they have flags to explicitly prevent garbage collection. One of these
flags is the `EInternalObjectFlags::RootSet`.

```c++
// Runtime/CoreUObject/Private/UObject/UObjectHash.cpp

// Global UObject array instance
FUObjectArray GUObjectArray;                                   // To keep track of all UObjects
```

There are 3 ways to keep them referenced in the internal reference graph:

1. With a strong reference (`UPROPERTY()`) to them from objects that are also referenced

   In other words, if a `UObject` does not have a `UPROPERTY` reference, there is a good chance it will be garbage
   collected away.
   `UActor`s and `UActorComponent`s are an e exception this rule, since the Actor is referenced by the `UWorld` (which
   is in the root set),
   and the Actor Component is referenced by the Actor itself.

   Another practical implication of this is that once the owning `UActor` is destroyed, usually all of
   its `UActorComponent`s will be destroyed because
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

`UObject`s marked with `UPROPERTY()` or stored in Unreal container classes are visible to the reflection system, and
thus are automatically
nulled when the object is destroyed. This is because null-checking is more reliable than non-null pointers pointing at
deleted memory.

##### Serialization

#### UActor

The word "Actor" is not a term unique to Unreal Engine, and can be found even in Nvidia's PhysX. It is the base class
for all objects that can be placed in the world. Specifically, it is a
`UObject` with a transform.

#### Composition OOP Design Pattern - Components

Composition is a common object-oriented programming design pattern that defines reusable behavior and expresses has-a
relationships instead
of is-a
relationships. [Design Patterns: Elements of reusable Object-Oriented Software (1994) by Gang of Four](https://en.wikipedia.org/wiki/Design_Patterns)
elaborates on this design pattern, I highly recommend
reading this book as Unreal Engine uses other patterns from this book. You will find that containing gameplay
functionality within components rather than the larger gameplay classes prevents tarballing files into
a mess of tightly coupled code that takes longer to compile and harder to maintain due to everything depending on each
other.

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

The Gameplay Ability System (GAS) is a framework to streamline the complex logic that goes into replication, canceling,
casting, granting, and blocking of abilities. Without GAS, you would
have to use an increasingly unmaintainable spangle of conditional flag checking, timers, state machines, and RPC calls
to implement abilities. The GAS pattern comes from World of Warcraft, an
obvious heavy user of abilities at scale.

###### Gameplay Abilities

Blueprint-able functionality that can be given or removed, triggered manually or by input. Can be paired with an animation montage. Replication is built into the
system.

###### Gameplay Effects

###### Gameplay Tags

Although Gameplay Tags is not exclusive to GAS, it handles the "conditional flag checking" of abilities. It is a simple
system that allows you to tag objects with arbitrary hierarchical strings.
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

**Note:** This is not part of the engine, but rather built on top of it. Including because it can be helpful and a good
wrap up of the engine in practical usage

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
