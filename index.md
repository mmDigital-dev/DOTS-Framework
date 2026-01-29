# DOTS Framework – Entity Template & LinkedObject System

> ⚠️ **Work in Progress**
> This framework is actively under development. Some features are experimental, limited, or subject to change.
> Performance characteristics and APIs may evolve.

---

## Table of Contents

* [Overview](#overview)
* [Core Concepts](#core-concepts)
  * [Entity Templates](#entity-templates)
  * [Entity Instances](#entity-instances)
  * [Batched Execution](#batched-execution)

* [Editor Tooling & Code Generation](#editor-tooling--code-generation)
* [Entity Creation & Registration](#entity-creation--registration)
* [Runtime EntityBuilder (Experimental)](#runtime-entitybuilder-experimental)
* [Static Framework API](#static-framework-api)
* [LinkedObjects](#linkedobjects)
* [Template Attributes](#template-attributes)
* [Queries](#queries)
* [Runtime Execution Lifecycle](#runtime-execution-lifecycle)
* [Supported Data Types](#supported-data-types)
* [Data Access (Getter / Setter API)](#data-access-getter--setter-api)
* [Performance Guidelines](#performance-guidelines)
* [Roadmap](#roadmap)

---

## Overview

This framework provides a **template-based abstraction layer** on top of **Unity DOTS (Entities)**.

Key goals:

* Define **Entity Archetypes (Templates)** once
* Instantiate entities cheaply and consistently
* Control execution through **attribute-driven queries**
* Share data safely between **GameObject world and DOTS** using **LinkedObjects**
* Automatically batch structural changes at the end of the frame
* handles all scheduling for maximal parallelism

---

## Core Concepts
The framework simplifies the complexity of Unity DOTS into an intuitive workflow based on three pillars:

1. **Templates (Blueprints)**: Instead of manually assembling entities with an `EntityManager`, you register templates. These define the component layout (archetype) and allow you to manage instantiation consistently.


2. **LinkedObjects (The Bridge)**: This solves the primary DOTS challenge: communication with standard MonoBehaviours. You define static data structures that both GameObjects and DOTS jobs can access safely and performantly.


3. **Queries (Logic)**: You don't need to write complex `SystemBase` classes. Instead, you use static methods with attributes like `[UpdateTemplate]`. The framework automatically handles scheduling and parallelism as jobs.

### Entity Templates

An **Entity Template** represents an **archetype**, not an instance.

* Templates are registered once
* Components can be added before instantiation
* Instantiation clones the template archetype

Templates are identified by:

* `int ID`
* or `string name` (internally hashed)

---

### Entity Instances

When a template is instantiated:

* A new entity is created
* A **SubID (ulong)** is assigned
* The SubID is used to destroy the instance later

---

### Batched Execution

All framework operations are **queued** and executed **at the end of the frame**:

* Entity registration
* Component additions
* Instantiation
* Destruction

✅ This avoids structural changes during query execution  
⚠️ This means **results are not immediate** (changes at end of frame)

---

## Editor Tooling & Code Generation

The framework automates the boilerplate for Templates and LinkedObjects through background code generation. Manage these tools via the **DotsFramework** top-level menu.

### Menu Commands

* **Framework Compilation** The master switch. Enables or disables the entire framework.
* **Auto Compilation** Automatically triggers a recompile when changes are detected in your scripts.
* **Recompile** Force-cleans the cache and triggers a fresh compilation.
* **Clear** Wipes all generated files and resets the internal framework state.

---

## Entity Creation & Registration

### Creating Templates

```csharp
Framework.RegisterEntity("Player", playerTemplate);
Framework.RegisterEntity(2, enemyTemplate);
```
> ⚠️ Strings will be hashed internally. When using mixed id types, it may **lead to duplicates**
---

### Adding Components to Templates

```csharp
Framework.AddComponent(1, new Health { Value = 100 });
Framework.AddComponent(2, new Speed  { Value = 5f });
```

> ⚠️ Adding components **changes the archetype** of the registered template  
> ⚠️ Already instantiated entities **won't change**

---

## Runtime EntityBuilder (Experimental)

```csharp
new EntityBuilder().BuildEntity(gameObject, entityID);
```

### Status

* ⚠️ **Experimental**
* ⚠️ Physics components do **not work correctly**
* ⚠️ **High performance cost**

### Recommendation

> ❌ **Do NOT use in performance-critical code**  
> ✅ Prefer **Bakers**

### Authoring & Baking
The `RegisterAsEntity` Authoring Component allows you to integrate directly with your Baker. By adding this component to your GameObject, the resulting baked entity is registered automatically, removing the need for manual registration in code.

---

## Static Framework API

All interactions go through the static `Framework` class.

---

### RegisterEntity

```csharp
Framework.RegisterEntity(int id, Entity entity);
Framework.RegisterEntity(string name, Entity entity);
```

Registers an entity archetype as a template.

---

### AddComponent

```csharp
Framework.AddComponent<T>(int id, T component);
Framework.AddComponent<T>(string name, T component);
```

* Adds a component to a **registered template**
* Changes its archetype
* `T` must be `unmanaged` and `IComponentData`

---

### Instantiate

```csharp
Framework.Instantiate(int id);
Framework.Instantiate(string name);

Framework.Instantiate(int id, LocalTransform transform);
Framework.Instantiate(string name, LocalTransform transform);
```

* Creates an instance of the template
* Optional transform initialization
* returns unique identifier of instance (**subID**)

⚠️SubID is only unique in same session. Resets when the framework resets.

---

### Destroy

```csharp
Framework.Destroy(ulong subID);
```

Destroys a previously instantiated entity **by subID**.

---

## LinkedObjects

LinkedObjects allow **data sharing between GameObject world and DOTS**.

They are:

* Declared as public static fields
* Accessed in Burst-compatible DOTS queries
* Automatically synchronized every frame

---

### Declaration (GameObject World)

```csharp
[LinkedObject(1)]
public static PlayerDataStruct PlayerData;
```

```csharp
[LinkedObject("EnemiesInRange")]
public static List<ulong> EnemiesInRange;
```

> ⚠️ LinkedObjects **must be defined before the first Update**  
> ⚠️ Strings will be hashed internally. When using mixed id types, it may **comes to duplicates**


---

## Template Attributes

These attributes control **when to run functionality for which templates**.
* Runs only for the specified template IDs
* If **no IDs are provided**, runs for **all entities matching the query**
* ✅Fewer IDs → less exclusion → better performance
---

### `[UpdateTemplate]`

```csharp
[UpdateTemplate(1, 2)]
```

* Called **every frame**
* **First** query template that runs

---

### `[StartTemplate]`

```csharp
[StartTemplate("Player")]
```

* Called **once** when an entity instance is created
* Calls in the frame after the instance was created
* **Second** query template that runs

---

### `[EndTemplate]`

```csharp
[EndTemplate(2)]
```

* Called when an entity instance is destroyed
* Calls in the same frame, the instance is destroyed
* **Last** query template that runs

---

## Queries

Queries are the core logic units of your application. They define which code should run for specific entities and provide access to both DOTS component data and global `LinkedObjects`.

### Query Structure

Every query is defined as a `static void` method. The framework automatically detects these methods and schedules them within the Unity Job System.

```csharp
[BurstCompile]
[UpdateTemplate(PlayerID)] // Filters execution to this specific template
public static void PlayerMovement(
    ref EntityID entityID,                                              // Metadata (contains SubID and TemplateID)
    ref LocalTransform transform,                                       // Standard DOTS component (Read/Write)
    [LinkedObject("PlayerData")] ref LinkedObject playerData,       // Read-only global data (Burst-compatible)
    [LinkedObject("DeltaTime", true)] ref LinkedObject deltaTime    // Read-only deltaTime
)
{
    // Access DOTS components directly

    // Access LinkedObject data

    // Perform your logic here...
}

```



### Attributes:
### `[BurstCompile]`
* Enables the Unity Burst Compiler for high-performance machine code execution.
### `[UpdateTemplate(ID)]`
* Restricts the query to entities belonging to specific templates. If no ID is provided, the query runs on all entities that possess the required components.
* **`[StartTemplate]` / `[EndTemplate]`**: Use these instead of `UpdateTemplate` to run logic only when an entity is created or destroyed.

### Parameter Types (all optional):

### Components (`T`)
* Direct access to standard DOTS `IComponentData`. You can include as many components as needed.
### `EntityID`
* Provides the unique `SubID` of the instance. Available on every instance.
### `[LinkedObject]`
* Bridges the gap to the GameObject world. It allows you to pull in global data or shared structures.

### Parallelism:

The framework automatically handles job scheduling and dependency management:

* **Thread Safety**: If multiple queries try to write to the same `LinkedObject`, they are scheduled sequentially to prevent data corruption.
* **ReadOnly Optimization**: By setting the second parameter of `[LinkedObject]` to `true`, you mark the data as read-only. This allows the framework to run this query in parallel.

> ⚠️ **Parallel Execution**: A query is only scheduled in parallel if **all** LinkedObjects in its parameters are marked as **ReadOnly**


---

### Runtime Execution Lifecycle


<img src="https://github.com/mmDigital-dev/DOTS-Framework.Documentation/raw/main/runtime_sequence_diagram.png" width="1080" alt="Runtime Sequence Diagram">


The execution flow strictly separates **logic** from **structural changes** to guarantee thread safety.

1. **Sync In**
   * Pushes `LinkedObject` data (GO World → ECS)
2. **Run Queries** (Worker Threads)
   * `[StartTemplate]` (New entities)
   * `[UpdateTemplate]` (Active entities)
   * `[EndTemplate]` (Destroyed entities)
3. **Sync Out**
   * Pulls data back to `LinkedObjects`
4. **Apply Batched Operations**
   * Executes queued structural changes (Creation, Destruction)


---

## Supported Data Types

### Primitive Types

* `float`, `double`
* `short`, `int`, `long`
* `ushort`, `uint`, `ulong`
* `byte`

---

### Math Types

* `Vector2` / `float2`
* `Vector3` / `float3`
* `Vector4` / `float4`

---

### Strings

* `string`
* `FixedStringXXBytes`

⚠️Strings are all turned to size of 128 bytes internally

---

### Containers

* `Array`
* `List`

⚠️Containers can be nested, but depending on datatype it may introduce to some extra overhead


---

### Unsupported Types

Any unsupported type is:

* Automatically **flattened**
* Only **public fields** are considered

⚠️ Nested reference types are discouraged

---

## Data Access (Getter / Setter API)

All LinkedObject data is accessed via generic functions.

---

### Get

```csharp
playerData.Get(out int value, key, index);
```

---

### Set

```csharp
playerData.Set(value, key, index);
```

---

### Add (Lists)

```csharp
playerData.Add(value, key, index);
```

---

### Remove

```csharp
playerData.Remove(at, key, index);
```

---

### Length

```csharp
playerData.Length(key, index);
```

---

### Parameters

* `key` → field name (`FixedString32Bytes`)
* `index` → nested access path (`FixedList64Bytes<int>`)

```csharp
public struct PlayerData
{
    public int Health;
    public Skill[] Skills
}
public struct Skill
{
    public string Name;
}

playerSpeed.Get(out float speed); // no need for key if access is clearly defined (all but struct/class)

playerData.Get(out int health, "Health");

playerData.Get(out FixedString128Bytes skillName, "Skills.Name", new FixedList64Bytes<int>() {0}); // Gets first Index of Skills
```
---

## Performance Guidelines

### LinkedObjects

* Complex LinkedObjects → more overhead
* Data is synced **twice per frame** (read/write)
* LinkedObject sync cost scales with data size
---

### Lists

❌ Avoid:

```csharp
List<Class>
```

✅ Prefer:

```csharp
List<int>
List<float3>
```

---

### Queries & Parallelism

* Queries with **no overlapping write access** run in parallel
* ReadOnly LinkedObjects are ignored for conflict checks
* Fewer query parameters → higher chance of parallel execution

---

### Query Template Count

* Each query template has scheduling overhead
* Merge logic where possible

---

### Advanced Optimization

You may bypass the Getter/Setter API by directly accessing generated fields.

* ✅ **Significantly Faster**: Direct access avoids the large overhead of the generic API.
* ✅ **Stable**: However, the Getter/Setter API works **independently** of whether code has been generated or not.
* ⚠️ **Workflow Hazard**: Direct access **requires** generated code. If you use it without up to date generated code (e.g. change in fields), you will get **syntax errors** which block compilation. Since the generator needs compilation to run, you can get stuck.
* 💡 **Recommendation**: Use the Getter/Setter API during active development for stability. Switch to direct access **at the end of development** for performance.

```csharp
playerData.Get(out FixedString128Bytes skillName, "Skills.Name", new FixedList64Bytes<int>() {0}); 

playerData.Data.Skills[0].Name; //⚠️Only valid with generated code
```

You also can look into the generated files yourself to see what happens.

---

## Roadmap

The framework is under active development. Future updates will focus on performance overhead reduction, developer experience (DX), and extending the type system.

### 🚀 Performance & Core Engine

* **ReadOnly Optimization**: Refine the internal scheduling dependencies to better leverage `ReadOnly` parameters for even higher query concurrency.
* **Custom Struct Registration**: Allow users to register custom unmanaged structs (like float3) to be treated as compatible types within LinkedObjects, bypassing structure flattening.
* **Entity Type Support**: Add support for the `Entity` type as a query parameter to allow direct referencing outside the framework.
* **Refined Execution Ordering**: Implement a priority system (e.g., `[UpdateTemplate(Order = 10)]`) to allow manual control over the scheduling sequence when logic depends on specific execution flow.

### 🛠️ Developer Experience (DX)
* **Configurable String Sizes**: Add a project-wide setting to customize the default size of FixedString types (e.g., switching from 128 to 64 or 512 bytes) via the Editor menu.
* **Framework Generation Config**: A configuration menu to toggle specific code generation features, such as Getter/Setter APIs and other framework-specific options.
* **Framework Info Menu**: A new Editor window to visualize which fields are currently visible to the generator and how you can access them in the LinkedObject.
* **Code Analyzer**: Integrated analyzers to provide feedback (warnings/suggestions) on framework related code.
* **Automatic Access Generator**: A tool to automatically generates your code between the safe **Getter/Setter API** and the high-performance **Direct Access** mode for generated fields.
* **Enhanced Documentation**: Detailed technical deep-dives into the code-generation logic.

### 🧪 Stability & Compatibility

* **EntityBuilder V2**: Rewrite the experimental `EntityBuilder` to fully support Physics components and improve performance and compability.
* **Common Query Parameters**: Add built-in support for common global variables (like `DeltaTime`) as native query parameters without requiring manual LinkedObject setup.
* **Unit Testing Suite**: Implement a comprehensive test runner to validate generated code integrity.

