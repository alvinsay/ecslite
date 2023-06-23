# LeoEcsLite - Lightweight C# Entity Component System framework
Performance, zero or minimal allocations, minimization of memory usage, no dependencies on any game engine - these are the main goals of this framework.

> **IMPORTANT!** Don't forget to use `DEBUG` builds for development and `RELEASE` builds for releases: all internal checks/exceptions will work only in `DEBUG` builds and removed for performance reasons in `RELEASE` builds.

> **IMPORTANT!** LeoEcsLite framework is **not thread-safe** and never will be! If you need multithreading - you should implement it yourself and integrate synchronization as an ecs-system.

# Table of Contents
* [Social Resources](#social-resources)
* [Installation](#installation)
    * [As a Unity module](#as-a-unity-module)
    * [As source code](#as-source-code)
    * [Other sources](#other-sources)
* [Main Types](#main-types)
    * [Entity](#entity)
    * [Component](#component)
    * [System](#system)
* [Shared Data Usage](#shared-data-usage)
* [Special Types](#special-types)
    * [EcsPool](#ecspool)
    * [EcsFilter](#ecsfilter)
    * [EcsWorld](#ecsworld)
    * [EcsSystems](#ecssystems)
* [Integration with Engines](#integration-with-engines)
    * [Unity](#unity)
    * [Custom Engine](#custom-engine)
* [Articles](#articles)
* [Projects using LeoECS Lite](#projects-using-leoecs-lite)
    * [With source code](#with-source-code)
* [Extensions](#extensions)
* [License](#license)
* [FAQ](#faq)

# Social Resources
[![discord](https://img.shields.io/discord/404358247621853185.svg?label=enter%20to%20discord%20server&style=for-the-badge&logo=discord)](https://discord.gg/UQjdcbcHSf)

# Installation

## As a Unity module
Installation as a Unity module is supported via git link in PackageManager or direct editing of `Packages/manifest.json`:
```
"com.leopotam.ecslite": "https://github.com/Leopotam/ecslite.git",
```
By default, the latest release version is used. If you need a "develop" version with the latest changes - switch to the `develop` branch:
```
"com.leopotam.ecslite": "https://github.com/Leopotam/ecslite.git#develop",
```

## As source code
You can download the source code from the [GitHub repository](https://github.com/Leopotam/ecslite) and add it to your project manually.

## Other sources
Other sources for downloading the framework can be found in the [GitHub repository](https://github.com/Leopotam/ecslite).

# Main Types
## Entity
## Component
## System

# Shared Data Usage

# Special Types
## EcsPool
## EcsFilter
## EcsWorld
## EcsSystems

# Integration with Engines
## Unity
## Custom Engine

# Articles

# Projects using LeoECS Lite
## With source code

# Extensions

# License

# FAQ

## As source code
The code can also be cloned or obtained as an archive from the releases page.

## Other sources
The official working version is located at [https://github.com/Leopotam/ecslite](https://github.com/Leopotam/ecslite), all other versions (including *nuget*, *npm* and other repositories) are unofficial clones or third-party code with unknown content.

> **IMPORTANT!** The use of these sources is not recommended, only at your own risk.

# Main types

## Entity
In itself, it does not mean anything and does not exist, it is exclusively a container for components. Implemented as `int`:
```c#
// Create a new entity in the world.
int entity = _world.NewEntity ();

// Any entity can be deleted, at the same time all components will be automatically deleted first and only then the entity will be considered destroyed.
world.DelEntity (entity);

// Components from any entity can be copied to another. If the source or target entity does not exist, an exception will be thrown in DEBUG mode.
world.CopyEntity (srcEntity, dstEntity);
```

> **IMPORTANT!** Entities cannot exist without components and will be automatically destroyed when the last component on them is removed.

## Component
It is a container for user data and should not contain logic (minimal helpers are allowed, but not pieces of main logic):
```c#
struct Component1 {
    public int Id;
    public string Name;
}
```
Components can be added, requested, or removed through [component pools](#ecspool).

## System
It is a container for the main logic to process filtered entities. It exists as a user class that implements at least one of the `IEcsInitSystem`, `IEcsDestroySystem`, `IEcsRunSystem` (and other supported) interfaces:
```c#
class UserSystem : IEcsPreInitSystem, IEcsInitSystem, IEcsRunSystem, IEcsPostRunSystem, IEcsDestroySystem, IEcsPostDestroySystem {
    public void PreInit (IEcsSystems systems) {
        // Will be called once during IEcsSystems.Init() and before IEcsInitSystem.Init() is called for all systems.
    }
    
    public void Init (IEcsSystems systems) {
        // Will be called once during IEcsSystems.Init() and after IEcsPreInitSystem.PreInit() is called for all systems.
    }
    
    public void Run (IEcsSystems systems) {
        // Will be called once during IEcsSystems.Run().
    }
    
    public void PostRun (IEcsSystems systems) {
        // Will be called once during IEcsSystems.Run() after IEcsRunSystem.Run() is called for all systems.
    }

    public void Destroy (IEcsSystems systems) {
        // Will be called once during IEcsSystems.Destroy() and before IEcsPostDestroySystem.PostDestroy() is called for all systems.
    }
    
    public void PostDestroy (IEcsSystems systems) {
        // Will be called once during IEcsSystems.Destroy() and after IEcsDestroySystem.Destroy() is called for all systems.
    }
}
```

# Shared data usage
An instance of any custom type (class) can be simultaneously connected to all systems:
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

# Special types

## EcsPool
It is a container for components, provides an API for adding / querying / removing components on entities:
```c#
int entity = world.NewEntity ();
EcsPool<Component1> pool = world.GetPool<Component1> (); 

// Add() adds a component to the entity. If the component already exists, an exception will be thrown in DEBUG mode.
ref Component1 c1 = ref pool.Add (entity);

// Has() checks if the component exists on the entity.
bool c1Exists = pool.Has (entity);

// Get() returns the existing component on the entity. If the component does not exist, an exception will be thrown in DEBUG mode.
ref Component1 c1 = ref pool.Get (entity);

```markdown
The Del() method removes a component from an entity. If the component did not exist, no errors will occur. If it was the last component, the entity will be automatically deleted.

```
pool.Del(entity);

```

The Copy() method copies all components from one entity to another. If the source or target entity does not exist, an exception will be thrown in DEBUG mode.

```
pool.Copy(srcEntity, dstEntity);

```

> **IMPORTANT!** After removal, the component will be placed in the pool for subsequent reuse. All component fields will be reset to default values automatically.

## EcsFilter
It is a container for storing filtered entities based on the presence or absence of certain components:

```c#
class WeaponSystem : IEcsInitSystem, IEcsRunSystem {
    EcsFilter _filter;
    EcsPool<Weapon> _weapons;
    
    public void Init (IEcsSystems systems) {
        // Get the default world instance.
        EcsWorld world = systems.GetWorld ();
        
        // We want to get all entities with the "Weapon" component and without the "Health" component.
        // The filter stores only entities, the data itself is in the "Weapon" component pool.
        // The filter can be dynamically assembled each time, but caching is recommended.
        _filter = world.Filter<Weapon> ().Exc<Health> ().End ();
        
        // Request and cache the "Weapon" component pool.
        _weapons = world.GetPool<Weapon> ();
        
        // Create a new entity for testing.
        int entity = world.NewEntity ();
        
        // And add the "Weapon" component to it - this entity should be included in the filter.
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

Additional requirements for filtered entities can be added using the `Inc<>()` / `Exc<>()` methods.

> **IMPORTANT!** Filters support any number of component requirements, but the same component cannot be in both the "include" and "exclude" lists.

## EcsWorld
It is a container for all entities, component pools, and filters, and the data of each instance is unique and isolated from other worlds.

> **IMPORTANT!** You must call `EcsWorld.Destroy()` on the world instance if it is no longer needed.

## EcsSystems
It is a container for systems that will process the `EcsWorld` instance of the world:
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
        // Execute all connected systems.
        _systems?.Run ();
    }

    void OnDestroy () {
        // Destroy connected systems.
        if (_systems != null) {
            _systems.Destroy ();
            _systems = null;
        }
        // Clear environment.
        if (_world != null) {
            _world.Destroy ();
            _world = null;
        }
    }
}
```

> **IMPORTANT!** You need to call `IEcsSystems.Destroy()` on the system group instance if it is no longer needed.

# Integration with engines

## Unity
> Tested on Unity 2020.3 (not dependent on it) and contains asmdef descriptions for compiling as separate assemblies and reducing the recompilation time of the main project.

[Unity editor integration](https://github.com/Leopotam/ecslite-unityeditor) contains code templates and provides monitoring of the world state.

## Custom engine
> C#7.3 or higher is required to use the framework.

Each part of the example below should be correctly integrated into the right place in the engine code execution:
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
            // Additional instances of worlds
            // should be registered here.
            // .AddWorld (customWorldInstance, "events")
            
            // Systems with main logic should
            // be registered here.
            // .Add (new TestSystem1 ())
            // .Add (new TestSystem2 ())
            
            .Init ();
    }

    // The method should be called from
    // the main update loop of the engine.
    void UpdateLoop () {
        _systems?.Run ();
    }

    // Clearing the environment.
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

# Projects using LeoECS Lite
## With sources

* ["SlimeHunter"](https://github.com/JimboA/SlimeHunter-LeoEcsLite)
  
  [![](https://media.githubusercontent.com/media/JimboA/SlimeHunter-LeoEcsLite/main/Screenshot_1.png)](https://github.com/JimboA/SlimeHunter-LeoEcsLite)


* ["3D Platformer"](https://github.com/supremestranger/3D-Platformer-Lite)

  [![](https://camo.githubusercontent.com/dcd2f525130d73f4688c1f1cfb12f6e37d166dae23a1c6fac70e5b7873c3ab21/68747470733a2f2f692e6962622e636f2f686d374c726d342f506c6174666f726d65722e706e67)](https://github.com/supremestranger/3D-Platformer-Lite)


* ["SharpPhysics2D"](https://github.com/7Bpencil/sharpPhysics)

  [![](https://github.com/7Bpencil/sharpPhysics/raw/master/pictures/preview.png)](https://github.com/7Bpencil/sharpPhysics)

## Without source code

* [Microbiome (WIP)](https://vk.com/microbiomegame)

  [![](https://img.youtube.com/vi/WTciasBN2eQ/0.jpg)](https://www.youtube.com/watch?v=WTciasBN2eQ)

# Extensions
* [Dependency Injection](https://github.com/Leopotam/ecslite-di)
* [Extended Systems](https://github.com/Leopotam/ecslite-extendedsystems)
* [Multithreading Support](https://github.com/Leopotam/ecslite-threads)
* [Unity Editor Integration](https://github.com/Leopotam/ecslite-unityeditor)
* [Unity uGui Support](https://github.com/Leopotam/ecslite-unity-ugui)
* [Unity Physx Events Support](https://github.com/supremestranger/leoecs-lite-physics)
* [Multiple Shared Injection](https://github.com/GoodCatGames/ecslite-multiple-shared)
* [EasyEvents](https://github.com/7Bpencil/ecslite-easyevents)
* [Entity Command Buffer](https://github.com/JimboA/EcsLiteEntityCommandBuffer)
* [Unity Editor Integration based on UIToolkit](https://github.com/Mitfart/LeoECSLite.UnityIntegration)
* [Unity Entity Converter](https://github.com/AndreyBirchenko/LeoEcsLiteEntityConverter)
* [Interval Systems](https://github.com/nenuacho/ecslite-interval-systems)
* [Quadtree Systems](https://github.com/nenuacho/ecslite-quadtree)

# License
The framework is released under two licenses, [details here](./LICENSE.md).

In cases of licensing under the MIT-Red terms, personal consultations or any guarantees should not be expected.

# FAQ

### What is the difference from the old version of LeoECS?

I prefer to call them `lite` (ecs-lite) and `classic` (leoecs). The main differences of `lite` are as follows:
* The framework's codebase has been reduced by half, making it easier to maintain and extend.
* `Lite` is not a cut-down version of `classic`, all functionality is preserved as core and external modules.
* No static data in the core.
* No component caches in filters, reducing memory consumption and increasing entity shifting speed through filters.
* Fast access to any component on any entity (not just filtered and through filter cache).
* No restrictions on the number of requirements/constraints on components for filters.
* Overall linear performance is close to `classic`, but access to components and entity shifting through filters has become disproportionately faster.
* Aimed at using multimaps - multiple instances of worlds simultaneously with data partitioning for memory consumption optimization.
* No reflection in the core, aggressive removal of unused code by the compiler is possible (code stripping, dead code elimination).
* Shared data between systems is implemented without reflection (if allowed, it is recommended to use the `ecslite-di` extension from the list of extensions).
* Entity implementation has returned to the usual `int` type, reducing memory consumption. If entities need to be saved somewhere, they still need to be packed into a special structure.
* Small core, all additional functionality is implemented through optional extensions.
* All new functionality will only be released to the `lite` version, `classic` has been switched to a bug-fixing support mode.

### I want to call one system in `MonoBehaviour.Update()` and another in `MonoBehaviour.FixedUpdate()`. How can I do this?

To separate systems based on different methods from `MonoBehaviour`, it is necessary to create a separate `IEcsSystems` group for each method:

```c#
IEcsSystems _update;
IEcsSystems _fixedUpdate;

void Start () {
    EcsWorld world = new EcsWorld ();
    _update = new EcsSystems (world);
    _update
        .Add (new UpdateSystem ())
        .Init ();
    _fixedUpdate = new EcsSystems (world);
    _fixedUpdate
        .Add (new FixedUpdateSystem ())
        .Init ();
}

void Update () {
    _update?.Run ();
}

void FixedUpdate () {
    _fixedUpdate?.Run ();
}
```

### I am not satisfied with the default values for component fields. How can I configure this?

Components support setting arbitrary values through the implementation of the `IEcsAutoReset<>` interface:

```c#
struct MyComponent : IEcsAutoReset<MyComponent> {
    public int Id;
    public object SomeExternalData;

    public void AutoReset (ref MyComponent c) {
        c.Id = 2;
        c.SomeExternalData = null;
    }
}
```

This method will be automatically called for all new components, as well as for all just removed components, before placing them in the pool.

> **IMPORTANT!** In case of using `IEcsAutoReset`, all additional field cleaning/checks of the component are disabled, which can lead to memory leaks. The responsibility lies with the user!

### I am not satisfied with the values for component fields when copying them through EcsWorld.CopyEntity() or Pool<>.Copy(). How can I configure this?

Components support setting arbitrary values when calling `EcsWorld.CopyEntity()` or `EcsPool<>.Copy()` through the implementation of the `IEcsAutoCopy<>` interface:

```c#
struct MyComponent : IEcsAutoCopy<MyComponent> {
    public int Id;

    public void AutoCopy (ref MyComponent src, ref MyComponent dst) {
        dst.Id = src.Id * 123;
    }
}
```

> **IMPORTANT!** In case of using `IEcsAutoCopy`, no default copying occurs. The responsibility for correctly filling in the data and for the integrity of the source lies with the user!

### I want to save a reference to an entity in a component. How can I do this?

To save a reference to an entity, it is necessary to pack it into one of the special containers (`EcsPackedEntity` or `EcsPackedEntityWithWorld`):

```c#
EcsWorld world = new EcsWorld ();
int entity = world.NewEntity ();
EcsPackedEntity packed = world.PackEntity (entity);
EcsPackedEntityWithWorld packedWithWorld = world.PackEntityWithWorld (entity);
...
// At the time of unpacking, we check if this entity is alive or not.
if (packed.Unpack (world, out int unpacked)) {
    // "unpacked" is a valid entity and we can use it.
}
```

```markdown
```csharp
// At the moment of unpacking, we check if this entity is alive or not.
if (packedWithWorld.Unpack(out EcsWorld unpackedWorld, out int unpackedWithWorld)) {
    // "unpackedWithWorld" is a valid entity and we can use it.
}
```

### I want to add reactivity and handle world change events myself. How can I do this?

> **IMPORTANT!** This is not recommended due to performance issues.

To activate this feature, you should add `LEOECSLITE_WORLD_EVENTS` to the list of compiler directives, and then add an event listener:

```csharp
class TestWorldEventListener : IEcsWorldEventListener {
    public void OnEntityCreated(int entity) {
        // Entity created - method will be called when world.NewEntity() is called.
    }

    public void OnEntityChanged(int entity) {
        // Entity changed - method will be called when pool.Add() / pool.Del() is called.
    }

    public void OnEntityDestroyed(int entity) {
        // Entity destroyed - method will be called when world.DelEntity() is called or when the last component is removed.
    }

    public void OnFilterCreated(EcsFilter filter) {
        // Filter created - method will be called when world.Filter().End() is called if the filter did not exist before.
    }

    public void OnWorldResized(int newSize) {
        // World resized - method will be called when the size of entity caches changes when world.NewEntity() is called.
    }

    public void OnWorldDestroyed(EcsWorld world) {
        // World destroyed - method will be called when world.Destroy() is called.
    }
}
...
var world = new EcsWorld();
var listener = new TestWorldEventListener();
world.AddEventListener(listener);
``` 

### I want to add reactivity and handle filter change events myself. How can I do this?

> **IMPORTANT!** This is not recommended due to performance issues.

To activate this feature, you should add `LEOECSLITE_FILTER_EVENTS` to the list of compiler directives, and then add an event listener:

```csharp
class TestFilterEventListener : IEcsFilterEventListener {
    public void OnEntityAdded(int entity) {
        // Entity added to filter.
    }

    public void OnEntityRemoved(int entity) {
        // Entity removed from filter.
    }
}
...
var world = new EcsWorld();
var filter = world.Filter<C1>().End();
var listener = new TestFilterEventListener();
filter.AddEventListener(listener);
```