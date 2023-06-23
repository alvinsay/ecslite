# LeoEcsLite - Lightweight C# Entity Component System Framework
The main goals of this framework are performance, zero or minimal allocations, minimization of memory usage, and no dependencies on any game engine.

> **IMPORTANT!** Don't forget to use `DEBUG` builds for development and `RELEASE` builds for releases: all internal checks/exceptions will work only in `DEBUG` builds and are removed for performance reasons in `RELEASE` builds.

> **IMPORTANT!** LeoEcsLite framework is **not thread-safe** and will never be! If you need multithreading, you should implement it yourself and integrate synchronization as an ecs system.

# Table of Contents
* [Social resources](#social-resources)
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
* [Engine Integration](#engine-integration)
    * [Unity](#unity)
    * [Custom Engine](#custom-engine)
* [Articles](#articles)
* [Projects using LeoECS Lite](#projects-using-leoecs-lite)
    * [With Source Code](#with-source-code)
* [Extensions](#extensions)
* [License](#license)
* [FAQ](#faq)

# Social Resources
[![discord](https://img.shields.io/discord/404358247621853185.svg?label=enter%20to%20discord%20server&style=for-the-badge&logo=discord)](https://discord.gg/UQjdcbcHSf)

# Installation

## As a Unity module
You can install it as a Unity module by adding the git link to the PackageManager or editing directly the `Packages/manifest.json`:
```
"com.leopotam.ecslite": "https://github.com/Leopotam/ecslite.git",
```
By default, the latest release version is used. If you need the "develop" version with the latest changes, switch to the `develop` branch:
```
"com.leopotam.ecslite": "https://github.com/Leopotam/ecslite.git#develop",
```

## As source code
You can also clone the code or obtain it as an archive from the release page.

## Other Sources
The official working version is located at [https://github.com/Leopotam/ecslite](https://github.com/Leopotam/ecslite), all other versions (including *nuget*, *npm*, and other repositories) are unofficial clones or third-party code with unknown contents.

> **IMPORTANT!** Using these sources is not recommended, only at your own risk.

# Main Types

## Entity
It doesn't mean anything by itself, and exists solely as a container for components. Implemented as an `int`:
```c#
// Create a new entity in the world.
int entity = _world.NewEntity ();

// Any entity can be removed, all its components will be automatically removed first, and only then the entity will be considered destroyed. 
world.DelEntity (entity);

// Components from any entity can be copied to another. If the source or target entity does not exist, an exception will be thrown in DEBUG builds.
world.CopyEntity (srcEntity, dstEntity);
```

> **IMPORTANT!** Entities cannot exist without components and will be automatically destroyed when their last component is removed.

## Component
It is a container for user data and should not contain any logic (minimal helpers are allowed, but not chunks of the main logic):
```c#
struct Component1 {
    public int Id;
    public string Name;
}
```
The components can be added, requested or removed through [component pools](#ecspool).

## System
It is a container for the main logic to process filtered entities. It exists as a user class implementing at least one of `IEcsInitSystem`, `IEcsDestroySystem`, `IEcsRunSystem` (and other supported) interfaces:
```c#
class UserSystem: IEcsPreInitSystem, IEcsInitSystem, IEcsRunSystem, IEcsPostRunSystem, IEcsDestroySystem, IEcsPostDestroySystem {
    public void PreInit (IEcsSystems systems) {
        // Will be called once during IEcsSystems.Init() and before IEcsInitSystem.Init() is triggered for all systems.
    }

    public void Init (IEcsSystems systems) {
        // Will be called once during IEcsSystems.Init() after IEcsPreInitSystem.PreInit() is triggered for all systems.
    }

    public void Run (IEcsSystems systems) {
        // Will be called once during IEcsSystems.Run().
    }

    public void PostRun (IEcsSystems systems) {
        // Will be called once during IEcsSystems.Run() after IEcsRunSystem.Run() is triggered for all systems.
    }

    public void Destroy (IEcsSystems systems) {
        // Will be called once during IEcsSystems.Destroy() and before IEcsPostDestroySystem.PostDestroy() is triggered for all systems.
    }
    
    public void PostDestroy (IEcsSystems systems) {
        // Will be called once during IEcsSystems.Destroy() after IEcsDestroySystem.Destroy() is triggered for all systems.
    }
}
```

# Data sharing
An instance of any custom type (class) can be connected to all systems simultaneously:
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
class TestSystem1: IEcsInitSystem {
    public void Init (IEcsSystems systems) {
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

// Add() adds a component to the entity. If the component already exists, an exception will be thrown in DEBUG version.
ref Component1 c1 = ref pool.Add (entity);

// Has() checks if a component exists on the entity.
bool c1Exists = pool.Has (entity);

// Get() returns an existing component on the entity. If the component does not exist, an exception will be thrown in DEBUG version.
ref Component1 c1 = ref pool.Get (entity);

// Del() removes a component from the entity. If there was no component, there will be no errors. If it was the last component - the entity will be removed automatically.
pool.Del (entity);

// Copy() performs a copy of all components from one entity to another. If the source or target entity does not exist, an exception will be thrown in DEBUG version.
pool.Copy (srcEntity, dstEntity);
```

> **IMPORTANT!** After deletion, the component will be placed in the pool for subsequent reuse. All component fields will be reset to default values automatically.

## EcsFilter
It is a container for storing filtered entities based on the presence or absence of certain components:
```c#
class WeaponSystem: IEcsInitSystem, IEcsRunSystem {
    EcsFilter _filter;
    EcsPool<Weapon> _weapons;

    public void Init (IEcsSystems systems) {
        // Get the default world instance.
        EcsWorld world = systems.GetWorld ();

        // We want to get all entities with the "Weapon" component and without the "Health" component.
        // The filter only stores entities, the data itself is stored in the "Weapon" component pool.
        // The filter can be dynamically assembled each time, but caching is recommended.
        _filter = world.Filter<Weapon> ().Exc<Health> ().End ();

        // Request and cache the "Weapon" component pool.
        _weapons = world.GetPool<Weapon> ();
        
        // Create a new entity for testing.
        int entity = systems.GetWorld ().NewEntity ();
        
        // Add a "Weapon" component to this entity - it must be included in the filter.
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

Additional requirements for filtered entities can be added through the `Inc<>()` / `Exc<>()` methods.

> **IMPORTANT!** Filters support any number of component requirements, but the same component cannot be included in both the "included" and "excluded" lists.

## EcsWorld
It is a container for all entities, component pools and filters. The data of each instance of the world is unique and isolated from other worlds.

> **IMPORTANT!** You should call `EcsWorld.Destroy()` on an instance of the world if you no longer need it.

## EcsSystems
It is a container for the systems that will operate on the `EcsWorld` instance:
```c#
class Startup : MonoBehaviour {
    EcsWorld _world;
    IEcsSystems _systems;

    void Start () {
        // Create an environment, connect the systems.
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
        // Clean up the environment.
        if (_world != null) {
            _world.Destroy ();
            _world = null;
        }
    }
}
```

> **IMPORTANT!** You should call `IEcsSystems.Destroy()` on a group of systems if you no longer need them.

# Integrating with Engines

## Unity
> Tested on Unity 2020.3 (is not dependent on it) and contains asmdef descriptions for compiling into separate assemblies and reducing main project recompilation time.

[Unity Editor Integration](https://github.com/Leopotam/ecslite-unityeditor) contains code templates and monitoring of world status.

## Custom Engines
> Requires C# 7.3 or higher to use the framework.

Each part of the example below should be correctly integrated into the correct place in the engine code execution:
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
            // Additional instances of the world
            // should be registered here.
            // .AddWorld (customWorldInstance, "events")
            
            // Systems with the main logic should
            // be registered here.
            // .Add (new TestSystem1 ())
            // .Add (new TestSystem2 ())
            
            .Init ();
    }

    // The method must be called from
    // the main update loop of the engine.
    void UpdateLoop () {
        _systems?.Run ();
    }

    // Clear the environment.
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

# Projects Using LeoECS Lite
## With Source

* ["SlimeHunter"](https://github.com/JimboA/SlimeHunter-LeoEcsLite)
  
  [![](https://media.githubusercontent.com/media/JimboA/SlimeHunter-LeoEcsLite/main/Screenshot_1.png)](https://github.com/JimboA/SlimeHunter-LeoEcsLite)


* ["3D Platformer"](https://github.com/supremestranger/3D-Platformer-Lite)

  [![](https://camo.githubusercontent.com/dcd2f525130d73f4688c1f1cfb12f6e37d166dae23a1c6fac70e5b7873c3ab21/68747470733a2f2f692e6962622e636f2f686d374c726d342f506c6174666f726d65722e706e67)](https://github.com/supremestranger/3D-Platformer-Lite)


* ["SharpPhysics2D"](https://github.com/7Bpencil/sharpPhysics)

  [![](https://github.com/7Bpencil/sharpPhysics/raw/master/pictures/preview.png)](https://github.com/7Bpencil/sharpPhysics)

## Without Source

* [Microbiome (WIP)](https://vk.com/microbiomegame)

  [![](https://img.youtube.com/vi/WTciasBN2eQ/0.jpg)](https://www.youtube.com/watch?v=WTciasBN2eQ)

# Extensions
* [Dependency Injection](https://github.com/Leopotam/ecslite-di)
* [Extended Systems](https://github.com/Leopotam/ecslite-extendedsystems)
* [Multithreading support](https://github.com/Leopotam/ecslite-threads)
* [Integration into Unity Editor](https://github.com/Leopotam/ecslite-unityeditor)
* [Support for Unity uGui](https://github.com/Leopotam/ecslite-unity-ugui)
* [Unity Physx events support](https://github.com/supremestranger/leoecs-lite-physics)
* [Multiple Shared injection](https://github.com/GoodCatGames/ecslite-multiple-shared)
* [EasyEvents](https://github.com/7Bpencil/ecslite-easyevents)
* [Entity command buffer](https://github.com/JimboA/EcsLiteEntityCommandBuffer)
* [Integration into Unity Editor based on UIToolkit](https://github.com/Mitfart/LeoECSLite.UnityIntegration)
* [Unity Entity Converter](https://github.com/AndreyBirchenko/LeoEcsLiteEntityConverter)
* [Interval Systems](https://github.com/nenuacho/ecslite-interval-systems)
* [Quadtree Systems](https://github.com/nenuacho/ecslite-quadtree)

# License
The framework is released under two licenses, [details here](./LICENSE.md).

In cases of licensing under the MIT-Red terms, do not expect
personal consultations or any warranties.

# FAQ

### What is the difference from the old version of LeoECS?

I prefer to call them `lite` (ecs-lite) and `classic` (leoecs). The main differences of `lite` are as follows:
* The framework's code base has been reduced by a factor of 2, making it easier to maintain and expand.
* `Lite` is not a cut-down version of `Classic`, all functionality is preserved as the core and external modules.
* No static data in the core.
* No component caches in filters, reducing memory consumption and increasing transfer speed of entities across filters.
* Fast access to any component on any entity (not just filtered and through a filter cache).
* No restrictions on the number of component requirements or constraints for filters.
* Overall linear performance is close to `Classic`, but access to components, transfer of entities across filters has become incomparably faster.
* Focus on using multiple worlds - multiple instances of worlds with data partitioning for memory consumption optimization.
* No reflection in the core, aggressive removal of unused code by the compiler (code stripping, dead code elimination) is possible.
* Joint use of shared data between systems occurs without reflection (if allowed, it is recommended to use the extension `ecslite-di` from the extensions list).
* Entity implementation returned to a regular `int` type, reducing memory consumption. If entities need to be saved somewhere, they still need to be packed into a special structure.
* Small core, all additional functionality is implemented through optional extensions.
* All new functionality will only be released to the `Lite` version, `Classic` is moved to bug fixing support mode.

### I want to call one system in `MonoBehaviour.Update()` and another in `MonoBehaviour.FixedUpdate()`. How can I do that?

To separate systems based on different methods from `MonoBehaviour`, you need to create a separate `IEcsSystems` group for each method:
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

### I don't like the default values for component fields. How can I configure this?

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
"This method will be automatically called for all new components as well as for all just deleted ones before placing them in the pool. IMPORTANT! When using `IEcsAutoReset`, all additional cleaning/checks of component fields are disabled, which can lead to memory leaks. Responsibility lies with the user!


### I do not like the values for the component fields when they are copied through EcsWorld.CopyEntity() or Pool<>.Copy(). How can I configure this?

Components support setting arbitrary values when calling `EcsWorld.CopyEntity()` or` EcsPool<>.Copy()` through the implementation of the `IEcsAutoCopy <>` interface:
```c#
struct MyComponent : IEcsAutoCopy<MyComponent> {
    public int Id;

    public void AutoCopy (ref MyComponent src, ref MyComponent dst) {
        dst.Id = src.Id * 123;
    }
}
```
> IMPORTANT! In case of applying `IEcsAutoCopy`, no default copying is performed. Responsibility for filling in the data correctly and for the integrity of the source lies with the user!

I want to save a reference to the entity in the component. How can I do that?

To save a reference to the entity, you need to pack it into one of the special containers (`EcsPackedEntity` or` EcsPackedEntityWithWorld`):
```c#
EcsWorld world = new EcsWorld ();
int entity = world.NewEntity ();
EcsPackedEntity packed = world.PackEntity (entity);
EcsPackedEntityWithWorld packedWithWorld = world.PackEntityWithWorld (entity);
...
// When unpacking, we check if this entity is alive or not.
if (packed.Unpack (world, out int unpacked)) {
    // "unpacked" is a valid entity and we can use it.
}

// When unpacking, we check if this entity is alive or not.
if (packedWithWorld.Unpack (out EcsWorld unpackedWorld, out int unpackedWithWorld)) {
    // "unpackedWithWorld" is a valid entity and we can use it.
}
```

I want to add reactivity and process changes in the world myself. How can I do that?

> IMPORTANT! This is not recommended due to performance degradation.

To activate this functionality, you should add `LEOECSLITE_WORLD_EVENTS` to the list of compiler directives, and then add an event listener:

```c#
class TestWorldEventListener : IEcsWorldEventListener {
    public void OnEntityCreated (int entity) {
        // Entity created - the method will be called at the moment of calling world.NewEntity ().
    }

    public void OnEntityChanged (int entity) {
        // Entity changed - method will be called at the time of pool.Add () / pool.Del () call.
    }

    public void OnEntityDestroyed (int entity) {
        // Entity destroyed - method will be called when world.DelEntity () is called or when the last component is deleted.
    }

    public void OnFilterCreated (EcsFilter filter) {
        // Filter created - the method will be called at the time of world.Filter ().End () call if the filter did not exist before.
    }

    public void OnWorldResized (int newSize) {
        // The world has changed - the method will be called when the size of caches for entities changes at the time of world.NewEntity () call.
    }

    public void OnWorldDestroyed (EcsWorld world) {
        // The world is destroyed - the method will be called when world.Destroy () is called.
    }
}
...
var world = new EcsWorld ();
var listener = new TestWorldEventListener ();
world.AddEventListener (listener);
``` 

I want to add reactivity and process filter change events. How can I do that?

> IMPORTANT! This is not recommended due to performance degradation.

To activate this functionality, you should add `LEOECSLITE_FILTER_EVENTS` to the list of compiler directives, and then add an event listener:

```c#
class TestFilterEventListener : IEcsFilterEventListener {
    public void OnEntityAdded (int entity) {
        // The entity has been added to the filter.
    }

    public void OnEntityRemoved (int entity) {
        // The entity has been removed from the filter.
    }
}
...
var world = new EcsWorld ();
var filter = world.Filter<C1> ().End ();
var listener = new TestFilterEventListener ();
filter.AddEventListener (listener);
```