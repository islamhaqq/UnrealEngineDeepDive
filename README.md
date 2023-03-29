## Preface

While asking for resources for a deep dive into Epic Game's Unreal Engine architecture beyond the little information
provided in the official documentation, to my dismay someone responded there aren't many resources like that around. So
begins my journey to understand the engine, write notes on my findings, and share it.

Although there may not be many resources on unraveling the engine code, there are a plethora of resources that help in
understanding the
engine. [Game Engine Architecture by Jason Gregory](https://www.goodreads.com/book/show/6709076-game-engine-architecture)
is a great resource on understanding Unreal Engine, and game engines in general.

## Entry Point

The entry point to the engine, like most other game engines, is the `WinMain` function, particularly for Windows. Unreal
Engine has it defined in `Windows/LaunchWindows.cpp`.
The [Quake 2 engine](https://github.com/id-Software/Quake-2/blob/master/win32/sys_win.c#L594), for example, also has the
identically named function. It returns a 0 on success or error levels otherwise.

![image](https://user-images.githubusercontent.com/11065634/228419404-c8debb73-0864-4f5e-a8c9-96507bd03d24.png)

Each supported OS has their respective entry point: MacOS' version for example is `INT32_MAIN_INT32_ARGC_TCHAR_ARGV`
in `Mac/LaunchMac.cpp` and for Linux it's `int main`.

## Main Engine Loop

## Runtime Components

Unreal Engine, like all software systems and game engines, is built in layers. In order to
maintain modularity and avoid circular dependencies, the lower layers do not depend on
upper layers.

The upper-most layers contain game runtime classes including `PlayerController` and
`GameModeBase`.

## Blueprint

All the graph editor tools are behind the scenes a `UEdGraph`. This includes Blueprint, Material, and Animation graphs.
The `UEdGraph` is a simple graph data structure that listeners on every node.

The all the graphs for a Blueprint, such as the Event Graph, are combined into an Ubergraph.