Translated with [Claude AI](https://claude.ai/)


# LeoEcsLite - Lightweight C# Entity Component System framework
Performance, zero or minimal allocations, memory usage minimization, no dependencies on any game engine - these are the main goals of this framework.

> **IMPORTANT!** Do not forget to use `DEBUG` builds for development and `RELEASE` builds for releases: all internal checks/exceptions will work only in `DEBUG` builds and removed to increase performance in `RELEASE` builds.

> **IMPORTANT!** The LeoEcsLite framework is **not thread safe** and will never be! If you need multithreading - you should implement it yourself and integrate synchronization as an ecs system.

# Table of Contents
* [Social Resources](#social-resources)
* [Installation](#installation)
    * [As Unity Module](#as-unity-module)
    * [As Source Code](#as-source-code) 
    * [Other Sources](#other-sources)
* [Core Types](#core-types)
    * [Entity](#entity)
    * [Component](#component) 
    * [System](#system)
* [Shared Data](#shared-data)
* [Special Types](#special-types)
    * [EcsPool](#ecspool)
    * [EcsFilter](#ecsfilter)
    * [EcsWorld](#ecsworld)
    * [EcsSystems](#ecssystems)
* [Engine Integration](#engine-integration)
    * [Unity](#unity)
    * [Custom Engine](#custom-engine) 
* [Articles](#articles)
* [LeoECS Lite Based Projects](#leoecs-lite-based-projects)
    * [With Source Code](#with-source-code)
    * [Without Source Code](#without-source-code)
* [Extensions](#extensions) 
* [License](#license)
* [FAQ](#faq)

# Social Resources
[![discord](https://img.shields.io/discord/404358247621853185.svg?label=enter%20to%20discord%20server&style=for-the-badge&logo=discord)](https://discord.gg/UQjdcbcHSf)

# Installation

## As Unity Module 
Installation as Unity module via git url in PackageManager or directly editing `Packages/manifest.json` is supported:
```
"com.leopotam.ecslite": "https://github.com/Leopotam/ecslite.git",
```
By default the latest release version is used. If you need "in development" version with latest changes - you should switch to `develop` branch:
```
"com.leopotam.ecslite": "https://github.com/Leopotam/ecslite.git#develop",
```

## As Source Code
The code can also be cloned or downloaded as archive from releases page.

## Other Sources
The official working version is hosted on [https://github.com/Leopotam/ecslite](https://github.com/Leopotam/ecslite), any other versions (including *nuget*, *npm* and other repositories) are unofficial clones or 3rd party code with unknown contents. 

> **IMPORTANT!** Using these sources is not recommended, only at your own risk.

# Core Types

## Entity 
Does not have any meaning by itself and does not exist, is exclusively a container for components. Implemented as `int`:
```c#
// Create new entity in world.
int entity = _world.NewEntity ();

// Any entity can be destroyed, all components will be automatically removed first and only after that entity will be considered as destroyed.
world.DelEntity (entity); 

// Components from any entity can be copied to another one. If source or destination entities do not exist - exception will be thrown in DEBUG mode.  
world.CopyEntity (srcEntity, dstEntity);
```

> **IMPORTANT!** Entities can not exist without components and will be automatically destroyed on removal of the last component.

## Component
Is a container for user data and should not contain logic (minimal helpers are allowed, but not big pieces of core logic):
```c# 
struct Component1 {
    public int Id;
    public string Name;
}
```
Components can be added, requested or removed through [component pools](#ecspool). 

## System
Is a container for core processing logic for filtered entities. Exists as a custom class, implementing one or more of `IEcsInitSystem`, `IEcsDestroySystem`, `IEcsRunSystem` (and other supported) interfaces:
```c#
class UserSystem : IEcsPreInitSystem, IEcsInitSystem, IEcsRunSystem, IEcsPostRunSystem, IEcsDestroySystem, IEcsPostDestroySystem {
    public void PreInit (IEcsSystems systems) {
        // Will be called once on IEcsSystems.Init() before IEcsInitSystem.Init() triggered on any system.
    }
    
    public void Init (IEcsSystems systems) {
        // Will be called once on IEcsSystems.Init() after IEcsPreInitSystem.PreInit() triggered on all systems.
    }
    
    public void Run (IEcsSystems systems) {
        // Will be called once on IEcsSystems.Run().
    }
    
    public void PostRun (IEcsSystems systems) {
        // Will be called once on IEcsSystems.Run() after IEcsRunSystem.Run() triggered on all systems.
    }

    public void Destroy (IEcsSystems systems) {
        // Will be called once on IEcsSystems.Destroy() before IEcsPostDestroySystem.PostDestroy() triggered on any system.
    }
    
    public void PostDestroy (IEcsSystems systems) {
        // Will be called once on IEcsSystems.Destroy() after IEcsDestroySystem.Destroy() triggered on all systems.
    }
}
```

# Shared Data
Any custom type (class) instance can be injected into all systems simultaneously:
```c#
class SharedData {
    public string PrefabsPath;
}
...
SharedData sharedData = new SharedData { PrefabsPath = "Items/{0}" };
IEcsSystems systems = new EcsSystems (world, sharedData);
systems
    .Add (new TestSystem1 ())
    .Init ();
...
class TestSystem1 : IEcsInitSystem {
    public void Init(IEcsSystems systems) {
        SharedData shared = systems.GetShared<SharedData> ();
        string prefabPath = string.Format (shared.PrefabsPath, 123);
        // prefabPath = "Items/123" at this point. 
    }
}
```

# Special Types

## EcsPool
Is a container for components, provides api for adding / requesting / removing components on entities: 
```c#
int entity = world.NewEntity ();
EcsPool<Component1> pool = world.GetPool<Component1> ();

// Add() adds component to entity. If component already exists - exception will be thrown in DEBUG mode. 
ref Component1 c1 = ref pool.Add (entity);

// Has() checks if entity has component.
bool c1Exists = pool.Has (entity);

// Get() returns existing component on entity. If component does not exist - exception will be thrown in DEBUG mode.
ref Component1 c1 = ref pool.Get (entity); 

// Del() removes component from entity. No errors if component did not exist. If it was last component - entity will be removed automatically.
pool.Del (entity);

// Copy() copies all components from one entity to another. If source or destination entities do not exist - exception will be thrown in DEBUG mode.
pool.Copy (srcEntity, dstEntity);

```

> **IMPORTANT!** After removal, component will be put to pool for reuse. All component fields will be reset to default values automatically.

## EcsFilter 
Is container for storing filtered entities by required or excluded components:
```c#
class WeaponSystem : IEcsInitSystem, IEcsRunSystem {
    EcsFilter _filter;
    EcsPool<Weapon> _weapons;
    
    public void Init (IEcsSystems systems) {
        // Get default world instance.
        EcsWorld world = systems.GetWorld ();
        
        // We want all entities with "Weapon" component and without "Health" component.
        // Filter stores only entities, data itself resides in "Weapon" component pool.
        // Filter can be rebuilt dynamically each time, but caching is recommended. 
        _filter = world.Filter<Weapon> ().Exc<Health> ().End ();
        
        // Request and cache weapons pool.
        _weapons = world.GetPool<Weapon> ();
        
        // Create new entity for test.
        int entity = world.NewEntity ();
        
        // And add "Weapon" component to it - this entity should appear in filter.
        _weapons.Add (entity);
    }

    public void Run (IEcsSystems systems) {
        foreach (int entity in filter) {
            ref Weapon weapon = ref _weapons.Get (entity);
            weapon.Ammo = System.Math.Max (0, weapon.Ammo - 1); 
        }
    }
}
```

> **IMPORTANT!** It's enough to build filter once and cache it, rebuild to update entity list is not required.

Additional include/exclude constraints can be added via `Inc<>()` / `Exc<>()` methods.

> **IMPORTANT!** Filters support any number of component constraints, but same component can not be in both include and exclude lists.

## EcsWorld
Is a container for all entities, component pools and filters, each instance data is unique and isolated from other worlds. 

> **IMPORTANT!** `EcsWorld.Destroy()` should be called on world instance if it's no more needed.

## EcsSystems
Is a container for systems that will process `EcsWorld` instance:
```c# 
class Startup : MonoBehaviour {
    EcsWorld _world;
    IEcsSystems _systems;

    void Start () {
        // Create environment, connect systems.
        _world = new EcsWorld ();
        _systems = new EcsSystems (_world);
        _systems
            .Add (new WeaponSystem ())
            .Init ();
    }
    
    void Update () {
        // Run all connected systems.
        _systems?.Run (); 
    }

    void OnDestroy () {
        // Destroy connected systems.
        if (_systems != null) {
            _systems.Destroy ();
            _systems = null;
        }
        // Cleanup environment.
        if (_world != null) {
            _world.Destroy ();
            _world = null;
        }
    }
}
```

> **IMPORTANT!** `IEcsSystems.Destroy()` should be called on system group instance if it's no more needed.

# Engine Integration

## Unity
> Tested on Unity 2020.3 (not dependent on it) and contains asmdef setup for compilation as separate asm to reduce main project recompilation time.

[Unity editor integration](https://github.com/Leopotam/ecslite-unityeditor) provides code templates and world state monitoring. 

## Custom Engine
> C#7.3 or later required for framework usage.

Each part of example below should be correctly integrated in proper execution place of engine code:
```c#
using Leopotam.EcsLite;

class EcsStartup {
    EcsWorld _world;
    IEcsSystems _systems;

    // Environment initialization.
    void Init () {        
        _world = new EcsWorld ();
        _systems = new EcsSystems (_world);
        _systems
            // Additional world instances 
            // should be registered here.
            // .AddWorld (customWorldInstance, "events")
            
            // Systems with core logic should
            // be registered here.
            // .Add (new TestSystem1 ()) 
            // .Add (new TestSystem2 ())
            
            .Init ();
    }

    // Method should be called from
    // main update loop of engine. 
    void UpdateLoop () {
        _systems?.Run ();
    }

    // Environment cleanup.
    void Destroy () {
        if (_systems != null) {
            _systems.Destroy ();
            _systems = null;
        }
        if (_world != null) {
            _world.Destroy ();
            _world = null;
        }
    }
}
```

# Articles

* ["Creating a dungeon crawler with LeoECS Lite. Part 1"](https://habr.com/en/post/661085/)

  [![](https://habrastorage.org/r/w1560/getpro/habr/upload_files/372/b1c/ad3/372b1cad308788dac56f8db1ea16b9c9.png)](https://habr.com/en/post/661085/)

* ["Creating a dungeon crawler with LeoECS Lite. Part 2"](https://habr.com/en/post/673926/)

  [![](https://habrastorage.org/r/w1560/getpro/habr/upload_files/63f/3ef/c47/63f3efc473664fdaaf1a249f258e2486.png)](https://habr.com/en/post/673926/)

# LeoECS Lite Based Projects
## With Source Code

* ["SlimeHunter"](https://github.com/JimboA/SlimeHunter-LeoEcsLite)
  
  [![](https://media.githubusercontent.com/media/JimboA/SlimeHunter-LeoEcsLite/main/Screenshot_1.png)](https://github.com/JimboA/SlimeHunter-LeoEcsLite)


* ["3D Platformer"](https://github.com/supremestranger/3D-Platformer-Lite)

  [![](https://camo.githubusercontent.com/dcd2f525130d73f4688c1f1cfb12f6e37d166dae23a1c6fac70e5b7873c3ab21/68747470733a2f2f692e6962622e636f2f686d374c726d342f506c6174666f726d65722e706e67)](https://github.com/supremestranger/3D-Platformer-Lite)


* ["SharpPhysics2D"](https://github.com/7Bpencil/sharpPhysics)

  [![](https://github.com/7Bpencil/sharpPhysics/raw/master/pictures/preview.png)](https://github.com/7Bpencil/sharpPhysics)

## Without Source Code
* ["Super BioMan" (STEAM)](https://store.steampowered.com/app/2144580/Super_BioMan/)

  [![](https://cdn.akamai.steamstatic.com/steam/apps/2144580/header.jpg)](https://cdn.akamai.steamstatic.com/steam/apps/2144580/header.jpg)


* ["Microbiome" (WIP)](https://vk.com/microbiomegame)

  [![](https://img.youtube.com/vi/WTciasBN2eQ/0.jpg)](https://www.youtube.com/watch?v=WTciasBN2eQ)

# Extensions
* [Dependency Injection](https://github.com/Leopotam/ecslite-di)
* [Extended Systems](https://github.com/Leopotam/ecslite-extendedsystems)
* [Multithreading Support](https://github.com/Leopotam/ecslite-threads)  
* [Unity Editor Integration](https://github.com/Leopotam/ecslite-unityeditor)
* [Unity uGui Support](https://github.com/Leopotam/ecslite-unity-ugui)
* [Unity Physx events support](https://github.com/supremestranger/leoecs-lite-physics)
* [Multiple Shared injection](https://github.com/GoodCatGames/ecslite-multiple-shared)
* [EasyEvents](https://github.com/7Bpencil/ecslite-easyevents)
* [Entity command buffer](https://github.com/JimboA/EcsLiteEntityCommandBuffer)
* [Unity Editor Integration based on UIToolkit](https://github.com/Mitfart/LeoECSLite.UnityIntegration)
* [Unity Entity Converter](https://github.com/AndreyBirchenko/LeoEcsLiteEntityConverter)
* [Interval Systems](https://github.com/nenuacho/ecslite-interval-systems) 
* [Quadtree Systems](https://github.com/nenuacho/ecslite-quadtree)
* [LeoECS Lite Unity Zoo](https://github.com/aleverdes/leoecslite-zoo)
* [Adding/removing components debugger for LeoECS Lite](https://github.com/supremestranger/LiteEzDebuggerModule)

# License
Framework released under two licenses, [details here](./LICENSE.md).

In case of MIT-Red licensing do not expect any personal support or guarantees.

# FAQ

### What is the difference from old LeoECS version?

I prefer calling them `lite` (ecs-lite) and `classic` (leoecs). Main `lite` differences:
* Framework codebase reduced by 2 times, it became easier to support and extend.
* `Lite` is not a stripped down version of `classic`, all functionality preserved in form of core and external modules.  
* No any static data inside core.
* No component caches inside filters, it reduces memory usage and increases entity rearrange speed between filters.
* Fast access to any component on any entity (not only filtered one through filter cache).
* No limits on component requirements/constraints for filters.
* Common linear performance close to `classic`, but component access, entity rearrange between filters became extremely faster.
* Aiming on multi-world usage - running multiple world instances simultaneously to optimize memory usage by splitting data between them.
* No reflection inside core, aggressive code stripping by compiler (code stripping, dead code elimination) possible.
* Shared data injection between systems works without reflection (if allowed, it's recommended to use `ecsl
