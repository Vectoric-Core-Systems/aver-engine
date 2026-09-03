# Aver Engine — C# Scripting API

The gameplay you write for Aver Engine is **C# that runs in-process** against the engine's entity/component
world. You derive a base type (`AverActor`, `AverPawn`, `AverCharacter`, …), override lifecycle hooks
(`OnBeginPlay`, `OnTick`, …), and reach the running game through a small, static-first API (`Game`,
`Input`, and the `Entity` handle). There is **no `UObject`**: a "class" is a row in a native registry, and
spawning is a memcpy of that row's defaults — so scripts are thin, and the surface below is all you touch.

This document is the reference for that surface. It lives in `Aver.Framework` (namespace
`Aver.Framework`); vector maths (`Vec3`, `Quat`, `Rot`) lives in `Aver.Scene`; the log (`Log`) lives in
`Aver.Scripting`. Two further assemblies a game author writes against directly are documented here too:
**`Aver.UI`** (the game's HUD) and **`Aver.Materials`** (surfaces authored in C#). A new project
references all four whether or not it uses them (`sandbox/src/ProjectScaffold.cpp`) — an unused
reference costs nothing, and the alternative is a compile error naming an assembly the author has never
heard of, in a file the editor generated.

A fifth assembly, **`Aver.Physics`**, is *not* one of the four the scaffold references: it is a leaf
(no reference to `Aver.Scene` or `Aver.Framework`, so it speaks its own `Float3`/`Quaternion` rather
than `Vec3`/`Quat`) that a project adds a `ProjectReference` to only once it needs more than
`Aver.Framework.Physics`'s small compatibility subset — see [§14](#14-physics).

---

---

> Consolidated from the engine tree's `docs/SCRIPTING_API.md`.
> The website splits this across several pages; this file is the whole
> thing, for reading offline or searching in one place.

---

## 2. The class hierarchy

```
AverActor                     the root: a thing in the world with a transform and a lifecycle
├── AverPawn                  an actor that can be POSSESSED by a controller
│   └── AverCharacter         a first/third-person walking pawn (WASD + mouse)
├── AverPlayerController      the will that drives a pawn (possession control)
├── AverGameMode              the rules of a play session (names its pawn + controller)
└── AverGameInstance          process-lifetime state that spans levels
```

The base type you derive is how the **compiler** decides which capability flag the class gets
(`PAWN`, `CONTROLLER`, `GAME_MODE`, …) and which hooks you may override. `AverActor`, `AverPawn`,
`AverPlayerController`, `AverGameMode` and `AverGameInstance` are all abstract lineage anchors — none of
the five spawns on its own, so a `[AverGameMode]` that leaves `PlayerControllerClass` at its own default,
`"PlayerController"`, with no subclass of its own gets a controller that refuses to spawn, and possession
silently never happens. `AverCharacter` is the one exception: as of 0.3.0 it's concrete on its own,
registered as `"Character"` — a `[AverGameMode]` can set `DefaultPawnClass = "Character"` directly, with
no C# subclass required. Earlier releases needed a project's own concrete subclass for every base here,
this one included.



## 5. Actors, Pawns, Characters, Controllers, GameMode

### `AverActor`

The root type. Every actor has `public Entity Self { get; }` — its entity handle (you read it, never
assign it). See [Spawning](#9-spawning-and-destroying) for the `Spawn`/`Destroy` members.

### `AverPawn : AverActor`

Adds possession. `public Entity Controller`, `public T? ControllerAs<T>()`, `public bool IsPossessed`, and
the `OnPossessed`/`OnUnpossessed` hooks.

### `AverCharacter : AverPawn`

A first/third-person walking character. As of 0.3.0 it's **concrete** — before, it was abstract like
every other base on this page, and a project had to write its own subclass just to reach the capsule and
view machinery below. It now registers directly under the name `"Character"`, so that machinery is
reachable with no C# file at all.

| Member | Description |
|---|---|
| `public float MoveSpeed` | Ground move speed, cm/s (default 350). |
| `public float TurnSpeed` | Mouse-look sensitivity, degrees of yaw per pixel (default 0.2). |
| `public CameraView CameraViewMode` | `FirstPerson` or `ThirdPerson` (default third). |
| `public float EyeHeight` | First-person eye height, cm (default 160). |
| `public float BoomLength` | Third-person camera distance behind, cm (default 450). |
| `public float Yaw { get; }` | The character's facing yaw, degrees (read-only; turned by `DriveWithInput`, set by `SetYaw`). |
| `public float Height` / `public float Radius` | Capsule size, cm (default 180 / 34). Set before the first tick. |
| `public float JumpSpeed` | Upward speed a jump starts with, cm/s (default 465 — about 110 cm of height). |
| `public bool IsSimulated` | True when backed by the physics world rather than translating directly. |
| `public bool IsGrounded` | True while standing on ground shallow enough to hold. |
| `public Vec3 Velocity` | Current velocity, cm/s. The vertical component is the simulation's. |
| `public bool Jump()` | Jump if grounded; returns false when airborne rather than swallowing it. |
| `public void Teleport(Vec3 feet)` | Move the character *and* its capsule. Setting the transform alone leaves the capsule behind. |
| `protected void DriveWithInput(float dt)` | One frame of WASD-walk + mouse-turn control; call from `OnTick` while possessed. Reads the settled result of the last step, then writes this frame's intent. |
| `protected void SetYaw(float degrees)` | Snap the facing yaw without the mouse having moved it. |

The character's origin is its **feet**, not the capsule centre — which is what `EyeHeight` already
assumed, and what makes "put it on the ground" mean setting `z` to the ground height.

### `AverPlayerController : AverActor`

The will that drives a pawn. See [Possession](#10-possession).

### `AverGameMode : AverActor`

The rules for a play session. Names its pawn/controller *by string* on `[AverGameMode]`. Adds
`virtual void OnPostLogin(Entity controller)` — fires once the controller has entered.

### `AverGameInstance : AverActor`

Process-lifetime state that spans levels. Adds nothing over `AverActor`; the concept it contributes is
*lifetime*. Reach it with `Game.Instance`.

---

## 3. The play lifecycle

The world has two lives: **EDITOR** (authoring) and **PLAYING**. Pressing **Play** in the editor calls
`begin_play`, which:

1. moves the state to `Playing` **first** — so every `OnBeginPlay` and `OnPostLogin` fired below already
   observes `Game.State == Playing`, not the editor it is leaving;
2. spawns the **GameInstance** (if the project has one), then the **GameMode** (mandatory — if *it* fails
   to spawn, the state is restored to `Editor` and no session forms);
3. spawns the GameMode's named **PlayerController** and **Pawn**, and **possesses** the pawn (a mode may
   legally name neither — a menu mode, say);
4. fires the GameMode's `OnPostLogin`, after the possess, so it can already see the controller's pawn.

The ordering of step 1 is load-bearing, not incidental: a hook that gates on `Game.IsPlaying` runs during
begin-play and must see the session it is starting.

**Stop** calls `end_play`, which unpossesses, then ends the four session roots in the *reverse* of spawn
order (so `OnEndPlay` mirrors `OnBeginPlay`), then sweeps **every other actor the session spawned** — an
actor a GameMode spawned in a hook is not a root, and would otherwise get `OnBeginPlay` and never
`OnEndPlay`. Everything ends with `EndReason.Stop`, and the world returns to `EDITOR`. Read the current state and
singletons through [`Game`](#7-game--the-running-session).

The hook order within a frame: begin-play queue drains → `OnTick` in tick-group order → destroy queue
drains. Gameplay ticks only while `Playing` (a `Paused` session freezes it without tearing down).



## 4. Lifecycle hooks

Override these on your `AverActor` subclass. All are optional; the defaults do nothing.

| Hook | When |
|---|---|
| `void OnBeginPlay(BeginReason reason)` | Once when the actor begins playing. `reason` is `Spawn` (fresh), `Play` (an already-placed actor entering Play), or `Reload` (after a hot reload). |
| `void OnTick(float dt)` | Every frame the class is scheduled to tick, in its tick group. `dt` is clamped seconds. Requires `b.Ticks(...)` in `Configure`. |
| `void OnEndPlay(EndReason reason)` | Once when the actor stops playing. `Stop` = the world stopped; `Destroy` = it was destroyed; `Reload` = the entity stays, only the managed half is rebuilt. |
| `void OnRebound()` | **DECLARED BUT NEVER CALLED.** The hook exists in `framework_hooks.h` and nothing in the engine invokes it; the bridge's own comment records the rebind path as out of scope. Do not put logic here expecting it to run. |
| `void BuildModels(ActorBuilder builder)` | `protected`. Builds the actor's model tree; the editor overrides it in a generated `.Designer.cs`. Hand code rarely writes this. |

Pawn-only hooks (`AverPawn`): `OnPossessed(Entity controller)`, `OnUnpossessed()`.
GameMode-only hook (`AverGameMode`): `OnPostLogin(Entity controller)`.

> **Exceptions never cross back into the engine.** A throw out of any hook is caught, logged, and that one
> actor is disabled for the session; the process and every other actor survive.

---

## 6. `Entity` — the handle to a thing in the world

`Entity` is a `readonly struct` — a 32-bit key into the world's component arrays. `default` is invalid.
Transform **writes are methods** (not property setters) because it is returned by value.

**Identity**

| Member | Description |
|---|---|
| `int Handle` | The raw ABI handle (0 = invalid). |
| `static Entity None` | The invalid handle. |
| `bool IsValid` | `Handle != 0`. |
| `bool IsAlive` | The handle still addresses a live entity. |
| `bool IsActor` | A gameplay class owns this entity (vs a plain scene entity). |
| `ActorClass Class` | The class this entity is an instance of. |
| `string Name` / `void SetName(string)` | The display name. |
| `ulong ObjectId` | The persisted identity (fnv1a64 of the name), survives serialisation. |
| `AverActor? Actor` / `T? As<T>()` | The live managed instance bound to this entity, or null. |
| `void Destroy()` | Destroy this entity (full framework teardown if it's an actor). |

**Transform** (local, centimetres; rotation as a quaternion — compose in degrees via `Rot`)

| Member | Description |
|---|---|
| `Vec3 LocalPosition` / `void SetLocalPosition(Vec3)` / `void Translate(Vec3 delta)` | Local position. |
| `Quat LocalRotation` / `void SetLocalRotation(Quat)` | Local rotation. |
| `Vec3 LocalScale` / `void SetLocalScale(Vec3)` | Local scale. |
| `Vec3 Forward` / `Right` / `Up` | Local axes (+X / +Y / +Z), rotated by the local rotation. |
| `Vec3 WorldPosition` | World-space position, composed through parents. |
| `Vec3 WorldForward` / `WorldRight` / `WorldUp` | World-space axes, normalised. |

**Hierarchy**

| Member | Description |
|---|---|
| `Entity Parent` / `Entity FirstChild` / `Entity NextSibling` / `int ChildCount` | Links. |
| `IEnumerable<Entity> Children` | Immediate children, in order. |
| `bool SetParent(Entity parent)` / `bool Detach()` | Reparent (`Entity.None` → root) / detach. Refuses a cycle. |

**Components** (see the [`Component`](#12-enums) enum)

| Member | Description |
|---|---|
| `bool HasComponent(Component)` / `bool AddComponent(Component)` | Query / attach a **built-in** component (idempotent). |
| `bool HasComponent(string name)` / `bool AddComponent(string name)` | As above, for a component registered **at runtime** rather than built in — `"CControlRig"`, `"CSynapseAgent"` — which therefore has no `Component` enum value: its id depends on registration order. A misspelt name resolves to nothing rather than the wrong pool. New in 0.5.0 (`aver_scene_component`). A component attached this way arrives **zero-filled with no constructor run** — write every field it needs yourself; see [`Entity.SetControlRig`](#21-animation--on-entity) for why that matters in practice. |
| `bool Visible` / `bool SetVisible(bool)` | Mesh visibility (adds a mesh renderer if absent). |
| `bool SetMesh(string path)` / `bool SetMaterial(string name)` | Set the drawn mesh / material. |

**Tags** (a 32-bit mask the gameplay layer gives meaning to — define your own bit constants)

| Member | Description |
|---|---|
| `uint Tags` / `bool SetTags(uint)` | The whole mask. |
| `bool HasTag(uint)` / `bool HasAnyTag(uint)` | All bits set / any bit set. |
| `bool AddTag(uint)` / `bool RemoveTag(uint)` | Set / clear bits. |

**Generic component fields** (by qualified name, e.g. `"CLight.intensityLux"` — works for `[Editable]`
fields and any component field)

`float GetFloat(string)` · `bool SetFloat(string, float)` · `int GetInt(string)` ·
`bool SetInt(string, int)` · `long GetInt64(string)` · `bool SetInt64(string, long)` ·
`Vec3 GetVec3(string)` · `bool SetVec3(string, Vec3)` · `string GetString(string)` ·
`bool SetString(string, string)`

---

## 7. `Game` — the running session

Static. In EDITOR every handle is `Entity.None` and `State` is `Editor`. **Read-only**: starting/stopping
a session is the editor's job, not a script's.

**State**

| Member | Description |
|---|---|
| `PlayState State` | `Editor` / `Playing` / `Paused`. |
| `bool IsPlaying` / `bool IsPaused` / `bool HasSession` | Playing / paused / a session exists (playing OR paused). |

**Session singletons** (each `Entity`, with a typed `…As<T>()` counterpart)

| Member | Typed |
|---|---|
| `Entity Mode` | `T? ModeAs<T>()` where `T : AverGameMode` |
| `Entity Instance` | `T? InstanceAs<T>()` where `T : AverGameInstance` |
| `Entity LocalPlayerController` / `Entity GetPlayerController(int i = 0)` | `T? PlayerControllerAs<T>(int i = 0)` |
| `Entity GetPlayerPawn(int i = 0)` | `T? PlayerPawnAs<T>(int i = 0)` where `T : AverPawn` |

**Lookup & iteration** (each a linear scan taken this frame — don't spawn/destroy while enumerating)

| Member | Description |
|---|---|
| `Entity Find(string name)` / `T? FindActor<T>(string name)` | Resolve an actor by name. |
| `IEnumerable<Entity> AllActors()` | Every live actor. |
| `IEnumerable<T> ActorsOf<T>()` | Every live actor whose instance is a `T`. |
| `IEnumerable<Entity> WithTag(uint mask)` | Every entity carrying all of `mask`. |

`Actors.Get(Entity)` / `Actors.Get<T>(Entity)` are the lower-level resolvers `Entity.As<T>` is built on.



## 9. Spawning and destroying

`Spawn` is `protected static` on `AverActor` (call from within an actor). The typed forms return the **live
instance** (or null if the class isn't declared); the `ActorClass` forms return the `Entity`.

| Member | Description |
|---|---|
| `Entity Spawn(ActorClass c, Vec3 at)` | Spawn a class chosen at runtime. |
| `Entity Spawn(ActorClass c, Vec3 at, Rot rotation)` | …with orientation. |
| `Entity Spawn(ActorClass c, Vec3 at, Rot rotation, Vec3 scale)` | …with a full transform. |
| `T? Spawn<T>(Vec3 at)` | Spawn `T`, return the instance. |
| `T? Spawn<T>(Vec3 at, Rot rotation)` / `T? Spawn<T>(Vec3 at, Rot rotation, Vec3 scale)` | …with orientation / full transform. |
| `T? SpawnAttached<T>(Entity parent, Vec3 localPosition)` | Spawn as a child of `parent`; returns null if the attach is rejected (and cleans up). |
| `void Destroy()` | Destroy this actor now (fires `OnEndPlay`). |
| `static void Destroy(Entity other)` / `static void Destroy(AverActor? other)` | Destroy another. |

**`ActorClass`** — the `Spawn<T>` forms name the class by *type*; the `Spawn(ActorClass, …)` forms take a
class **handle**, for when the class is chosen at runtime. Get one with `ActorClass.Find("BP_Coin")` (an
invalid handle if no such class is declared — check `IsValid`); `Entity.Class` gives the class an entity is
an instance of. An `ActorClass` is a `readonly struct` with `int Handle`, `bool IsValid`, and `string Name`
(`0 == invalid`).

```csharp
ActorClass coin = ActorClass.Find("BP_Coin");
if (coin.IsValid)
    for (int i = 0; i < count; i++)
        Spawn(coin, new Vec3(i * 100f, 0f, 20f));   // class picked at runtime
```



## 10. Possession

On `AverPlayerController`:

| Member | Description |
|---|---|
| `Entity Possessed` / `T? PossessedAs<T>()` / `bool HasPawn` | The controlled pawn. |
| `bool Possess(Entity pawn)` / `bool Possess(AverPawn pawn)` | Take control. A pawn already driven by another controller is *stolen* (it gets `OnUnpossessed` then `OnPossessed`); re-possessing the pawn you already drive is a no-op that returns true. |
| `bool Unpossess()` | Release the current pawn. |

On the pawn side, `AverPawn.Controller` / `ControllerAs<T>()` / `IsPossessed` read the inverse, straight
from the world (never cached).

---

## 8. `Input`

Static, polled, Unity-style. The editor pushes the frame's device state each tick (suppressed while a text
field has focus). Read it from `OnTick`.

| Member | Description |
|---|---|
| `bool GetKey(Key k)` | Held. |
| `bool GetKeyDown(Key k)` / `bool GetKeyUp(Key k)` | Went down / up this frame. |
| `float MouseDeltaX` / `MouseDeltaY` / `MouseWheel` | This frame's mouse movement / wheel. |
| `Vec3 MoveAxis` | WASD/arrows as `(forward, right, 0)`; not normalised. |
| `bool GetVk(int vk)` / `GetVkDown(int vk)` / `GetVkUp(int vk)` | New in 0.5.0. The raw Win32 virtual-key code, additive to `Key` rather than replacing it: `Key` has 46 slots and a saved graph's `InputKey` node stores one as a literal int, so the enum can never be renumbered to reach the F-keys, numpad or OEM range — ask for `GetVk(0x74)` (F5) by value instead. |
| `bool GetGamepadButton(GamepadButton button, int pad = 0)` / `float GetGamepadAxis(GamepadAxis axis, int pad = 0)` | New in 0.5.0, and **the ABI shape only** — nothing in the shipped engine polls a physical pad yet (stated in `framework_abi.h`: zero consumers was the reason not to build a poller), so these read back whatever a future provider writes, or all-zero/false. `pad` is always 0 today. `GamepadAxis` values are unclamped, with no dead zone applied. |

`Key` values: `A`–`Z`, `D0`–`D9`, `Space`, `LeftShift`, `LeftCtrl`, `LeftAlt`, `Enter`, `Escape`, `Tab`,
`Left`, `Right`, `Up`, `Down`, `MouseLeft`, `MouseRight`, `MouseMiddle`. `GamepadButton` — `DPadUp/Down/
Left/Right`, `Start`, `Back`, `LeftThumb`, `RightThumb`, `LeftShoulder`, `RightShoulder`, `A`, `B`, `X`,
`Y`. `GamepadAxis` — `LeftX/Y`, `RightX/Y`, `LeftTrigger`, `RightTrigger`.

### Action mappings

`Input` is the device. `EnhancedInput` is the layer above it, for gameplay that should name *what the
player is doing* rather than which key they pressed — so a pawn asks whether `Fire` happened and never
mentions a key (`scripting/csharp/Aver.Framework/EnhancedInput.cs`). As of 0.5.0 this is no longer pure
C#: the algorithm (dead zone, accumulation, priority-stacked consumption) moved down onto the native
action ABI (`aver_fw_action_*`, framework minor 5), and this file is a thin wrapper over it — which is
what makes it reachable from an Aver Node graph (`InputAction`/`InputActionPressed`/`InputActionReleased`
nodes) and from C++, not only from a script that happens to be C#. Registration is idempotent **by
handle** now, not only by name: two call sites that each write `InputAction.Digital("Jump")` get the
same managed `InputAction` object.

| Type | Description |
|---|---|
| `InputAction` | A thing the player can do. `InputAction.Digital(name)` / `.Axis1D(name)` / `.Axis2D(name)`. Read `IsHeld`, `WasPressed`, `WasReleased`, `Value1D`, `Value2D`. Declare each **once**, usually as a `static readonly` field on a class shared between the pawn that reads it and the context that binds it. `InputAction.Find(name)` looks up one already registered **by this runtime** — the ABI itself has no getter for a handle's value type, so a handle registered elsewhere cannot become a strongly-typed `InputAction` here. |
| `InputMappingContext` | A set of key→action bindings pushed and popped as a whole. Derive it and bind in the constructor: `BindKey`, `BindAxis1D(action, positive, negative)`, `BindAxis2D(action, up, down, right, left)`, `BindMouseLook`, `BindMouseWheel`. |
| `EnhancedInput` | The router: `AddContext(context, priority = 0)`, `RemoveContext`, `ClearContexts`, `ContextCount`. |

A **higher-priority context consumes the keys it binds**, so a lower one never sees them — pushing a
menu or vehicle context suppresses the walking bindings without anything having to disable them.
Consumption is per *context*, not per binding, so one key can still feed two actions in the same
context. Actions are recomputed once per frame before the first tick group, so every actor in a frame
reads the same input; polling per actor would let two pawns disagree about a "was pressed" edge purely
because of tick order.

```csharp
static class GameActions
{
    public static readonly InputAction Move = InputAction.Axis2D("Move");
    public static readonly InputAction Jump = InputAction.Digital("Jump");
}

public sealed class OnFoot : InputMappingContext
{
    public OnFoot()
    {
        BindAxis2D(GameActions.Move, Key.W, Key.S, Key.D, Key.A);   // X forward, Y right
        BindKey(GameActions.Jump, Key.Space);
    }
}
```

---

## 20. `Aver.Settings` — durable per-player settings

New in 0.5.0: the first C# binding onto `Aver.Settings`' native store — nothing in C# had called it
before. Durable, per-player key/value storage that outlives any save file: a save is a world, and
settings belong to the player, not to a playthrough — deleting a save must not reset the render scale,
and loading one must not change a remapped key. Static and stateless from the caller's side, matching
`Input`'s own shape.

| Member | Description |
|---|---|
| `bool Open(string path)` / `bool OpenDefault()` | Opens (or creates) the store. Reading a file that does not exist is not an error — a first run has no settings. `OpenDefault()` opens `DefaultPath`; a project that wants its own file should call `Open` with its own path instead. Calling `Open` again with a different path closes whatever was open **without flushing it first**. |
| `string DefaultPath` | `<user data dir>/Aver/settings.ini`. Reading this opens nothing by itself. |
| `bool Flush()` | Writes to disk if anything changed since the last flush — true **including** when nothing needed writing; false only when a write was attempted and failed. Atomic on the native side (temp file, then rename). |
| `float GetFloat(key, fallback = 0)` / `int GetInt(key, fallback = 0)` / `bool GetBool(key, fallback = false)` / `string GetString(key, fallback = "")` | A missing or unparsable key returns `fallback` rather than a silent zero. |
| `bool SetFloat` / `SetInt` / `SetBool` / `SetString(key, value)` | False for an empty key. `SetString` is also false (and stores nothing) if `value` contains a newline — the store's format is one `key=value` line each, unescaped. |
| `bool Has(string key)` | True regardless of the key's stored type. |
| `bool Remove(string key)` | True if the key is gone afterwards — **including** when it never existed; a postcondition check, not a "did anything change" flag. |
| `int Count` | How many keys the store holds. |

This is what a per-player input rebind will persist through once a rebind UI exists: an
`InputMappingContext` is reconstructed from code every time its owning script spawns, so a remapped key
has to live somewhere that outlives the C# object describing the default — this store, not a save file,
is that somewhere. Nothing wires `EnhancedInput` to it automatically yet; a project that wants rebinding
today reads/writes `Settings` itself around `BindKey`.

---

## 14. `Physics`

Everything below is in the engine's contract — centimetres, +X forward, +Y right, +Z up, left-handed.
The backend (Jolt, MIT) is right-handed, +Y up and metric; that translation happens once behind the
native ABI, so no Jolt convention ever reaches a script. The world is stepped by the frame loop
**between the PrePhysics and PostPhysics tick groups**, at a **fixed step** regardless of frame rate —
set what you want in a `PrePhysics` tick and read what happened in a `PostPhysics` one.

There are two ways to reach it now, and which one you want depends on how much of the surface you need.

**`Aver.Framework.Physics`** (namespace `Aver.Framework`, one of the four assemblies every project
already references) is a small, `Vec3`-flavoured compatibility shim: `Ready`, `FixedStep`,
`SetGravity(Vec3)`, `BodyCount`, `AddStaticBox`/`AddDynamicBox`/`AddDynamicSphere`, `AddSensorBox`/
`AddSensorSphere`, `Raycast`/`RaycastAny`, `OverlapSphere`, `SphereCast`, and the `Contacts`/`Overlaps`
event queues, plus `Body`/`ContactEvent`/`OverlapEvent`/`RaycastHit` structs typed in `Vec3`/`Quat`. A
script that only ever called this subset before 0.5.0 compiles and behaves identically — nothing here
was removed or renumbered, and it converts to/from `Aver.Physics` underneath rather than P/Invoking a
second time.

**`Aver.Physics`** is where the rest of this release's physics work lives: joints, the extra rigid-body
shapes, collision layers, soft bodies, buoyancy, and the rest of the character controller — 125 of 125
ABI functions bound, up from 49. It is a **separate, leaf assembly** (no reference to `Aver.Scene` or
`Aver.Framework`, so it takes and returns its own `Float3`/`Quaternion` rather than `Vec3`/`Quat`) and,
unlike `Aver.Framework`, is **not** one of the four a new project references by default — add a
`ProjectReference` to `Aver.Physics.csproj` yourself, then `using Aver.Physics;`. `Float3`/`Quaternion`
exist only so a call site does not have to spell three or four floats each time; both are plain readonly
structs with `X`/`Y`/`Z`(`/W`) and no behaviour beyond `+`, `-`, `*` and `.ToString()`.

> The backend documents **broadphase queries as non-deterministic** (the broad phase is modified from
> several threads), and callback ordering likewise. Rely on *whether* something was hit and *where* —
> never on which of several equidistant bodies comes back. This applies to both surfaces above.

### The world (`Aver.Physics.Physics`, static)

| Member | Description |
|---|---|
| `bool Init()` / `void Shutdown()` | Starts the simulation (idempotent) / destroys every body, character and the world. |
| `bool Ready` | True while a world exists; false with physics compiled out. |
| `float FixedStep` / `bool SetFixedStep(float seconds)` | The fixed step, 1/60 unless changed. Set it before any bodies exist. |
| `void SetGravity(Float3)` | cm/s². Default `(0, 0, -980)` — one g, straight down. |
| `int Step(float dt)` | Advances by `dt` of real time in fixed steps (leftover time carries, a long stall is clamped); returns how many steps actually ran. |
| `int BodyCount` | Live bodies. |

### Creating bodies

Box, sphere, convex hull, mesh and heightfield live directly on `Physics`; capsule, cylinder, tapered
capsule and box-compound live on `Shapes` (below) — split only because the header they come from is
split the same way.

| Member | Description |
|---|---|
| `Body AddStaticBox(Float3 centre, Float3 halfExtents)` | Never moves — floors, walls, level geometry. |
| `Body AddDynamicBox(centre, halfExtents, float massKg = 0)` | Falls and collides; `massKg <= 0` derives mass from volume. |
| `Body AddDynamicSphere(centre, float radius, float massKg = 0)` | Likewise. |
| `Body AddSensorBox(centre, halfExtents)` / `AddSensorSphere(centre, radius)` | A trigger volume — reports through `Overlaps`, never pushes anything. |
| `Body AddConvexHull(Float3[] points, centre, bool dynamic, float massKg = 0)` | A hull wrapped around `points`. |
| `Body AddMesh(Float3[] vertices, int[] indices, centre)` | Static only. `indices` is 3 per triangle. |
| `Body AddHeightfield(float[] samples, int sampleCount, float spacingCm, centre)` | A row-major `sampleCount` × `sampleCount` grid of heights. |
| `Shapes.AddStaticCapsule` / `AddDynamicCapsule(centre, radius, height, massKg = 0)` | `height` is **total**, both hemispherical caps included. |
| `Shapes.AddStaticCylinder` / `AddDynamicCylinder(centre, radius, height, massKg = 0)` | Flat-ended; `height` is the full end-to-end length — no cap radius to add. |
| `Shapes.AddStaticTaperedCapsule` / `AddDynamicTaperedCapsule(centre, topRadius, bottomRadius, height, massKg = 0)` | Different cap radii along +Z — a forearm thicker at the elbow than the wrist. |
| `Shapes.AddStaticCompoundBoxes` / `AddDynamicCompoundBoxes(centre, Float3[] offsets, Float3[] halfExtents, massKg = 0)` | N boxes welded into **one** rigid body (a chair, a table); `offsets`/`halfExtents` are in the body's own local frame, not world space. |

### Soft bodies

A soft body **is** a `Body` — same handle counter, so it works with `Destroy`, `SetEntity` and every
query below unchanged; calling a soft-body accessor on a rigid body returns the ABI's own "not a soft
body" answer (`0`, `false`) rather than throwing.

| Member | Description |
|---|---|
| `Physics.AddSoftBody(Float3[] vertices, int[] indices, float[]? invMasses, centre, float compliance, float pressure, float damping = 0.1f, int iterations = 5)` | `invMasses` null = every particle at inverse mass 1 (a particle at 0 is pinned); `compliance` is inverse stiffness (0 = inextensible); `pressure` inflates a closed mesh from within (0 for cloth). |
| `Physics.AddSkinnedSoftBody(vertices, indices, invMasses, jointIndices, jointWeights, influences, jointCount, maxDistanceCm, backStopDistanceCm, centre, compliance)` | As above plus tethers to a bind-pose skinned position, free to move up to `maxDistanceCm`. Pass the bind pose as `vertices`; drive it each frame with `Body.SoftBodySkin`. |
| `Body.SoftBodyVertexCount` / `Body.SoftBodyVertices(float[] outXyz, int max)` | Particle count, and the deformed world-space positions. |
| `Body.SoftBodySkin(float[] jointMatrices, int jointCount, bool hardSkin)` | Call **before** `Physics.Step` each frame. `hardSkin` true snaps every particle instead of constraining toward it — pass true on the first frame and after a teleport. |
| `Body.SoftBodyApplyImpulse(Float3 centre, float radiusCm, Float3 velocity, float strength)` | Blends nearby particles' velocity a `strength` fraction toward `velocity` — a blend, not an added impulse, so calling it every step converges instead of running away. Returns how many particles were nudged. |

### Buoyancy

| Member | Description |
|---|---|
| `Physics.SetWaterPlane(heightCm, Float3 normal, buoyancy, linearDrag, angularDrag, Float3 fluidVelocity)` / `ClearWaterPlane()` / `WaterPlane()` | A global plane: every dynamic body below `heightCm` floats until the plane is cleared or the body leaves. `WaterPlane()` returns the height or `null`. |
| `Physics.BuoyantBodyCount` | How many bodies had buoyancy applied last step — tells "nothing floats" apart from "nothing is in the water". |
| `Body.SetWaterVolume(surfacePos, surfaceNormal, buoyancy, linearDrag, angularDrag, fluidVelocity)` / `Body.ClearWaterVolume()` | A per-body override (a puddle or tank at a different height) that takes precedence over the global plane for that body. |

### Collision layers

New in 0.5.0. Layer 0 is the default and collides with everything; a project that never touches this
behaves exactly as if it did not exist.

| Member | Description |
|---|---|
| `Physics.LayerCount` | `16`. |
| `Physics.LayerMaskAll` | Every layer — passing this to a filtered query makes it identical to the unfiltered one. |
| `Physics.LayerBit(int layer)` | The bit for one layer, for building a mask: `LayerBit(2) \| LayerBit(5)`. |
| `Physics.SetLayerCollision(a, b, bool enabled)` / `LayerCollision(a, b)` | Whether two layers collide — **symmetric**, and **subtractive**: every pair starts enabled, so a call says what does *not* collide. |
| `Physics.ResetLayerCollisions()` | Puts every pair back to colliding — what a level teardown wants, since the matrix belongs to the world. |
| `Body.Layer` / `Body.SetLayer(int)` | Which of the 16 layers (0..15) a body is on. Changing it never changes static/dynamic. |

### Contact and overlap events

Polled, not called back. Both queues are cleared at the start of each `Step`.

| Member | Description |
|---|---|
| `Physics.Contacts` | `IReadOnlyList<ContactEvent>` — bodies that began touching last step. `ContactEvent`: `A`, `B`, `Point`, `Normal`, `Other(Body one)`. |
| `Physics.Overlaps` | `IReadOnlyList<OverlapEvent>` — sensor entries/exits last step. `OverlapEvent`: `Sensor`, `Other`, `Entered`. |

### Queries

| Member | Description |
|---|---|
| `Physics.Raycast(origin, direction, maxDistanceCm)` / `RaycastAny(...)` | The first hit, unfiltered. `direction` need not be unit length. |
| `Physics.RaycastEx(origin, direction, maxDistanceCm, uint layerMask, Body ignoreBody)` | As `Raycast`, but only against bodies on a layer in `layerMask` and never `ignoreBody` — almost always the caster itself, so a ray fired from inside your own capsule does not hit you at distance 0. |
| `Physics.OverlapSphere(centre, radius, int maxResults = 64)` / `OverlapSphereEx(centre, radius, layerMask, ignoreBody, maxResults = 64)` | Every body overlapping a sphere; a count equal to `maxResults` means truncated. |
| `Physics.SphereCast(origin, direction, maxDistanceCm, radius)` / `SphereCastEx(..., layerMask, ignoreBody)` | Sweeps a sphere; `RaycastHit.Entity` is always `0` here — the native sweep does not resolve entities. |

`RaycastHit` — `Hit`, `Body`, `Entity`, `Point`, `Normal`. `Entity` is the scene entity that owns
whatever was hit, resolved via `Body.SetEntity`: a body from a level placement, a character or the
editor already carries this stamp; a body a script creates (`AddDynamicBox` and siblings) only carries
one once the script calls `SetEntity` itself. `Entity.None` (`0`) means the hit is real but
unclaimed — a landscape heightfield, today — which is **not** the same as a miss, so check `Hit` first.

### `Body` — a rigid or soft body handle (`0` invalid)

| Group | Members |
|---|---|
| Placement | `Position`, `Rotation` (`Quaternion`), `Velocity`, `AngularVelocity` (rad/s per axis); `SetPosition`/`SetRotation`/`SetVelocity`/`SetAngularVelocity` (all wake the body, ignore collision on the way, teleport rather than push); `AddVelocity(delta)` adds rather than replaces. |
| Motion & lifetime | `MotionTypeRaw` (`-1` for a dead handle, `0` = Static is a real answer), `SetMotionType(MotionType)`, `Destroy()`. |
| Forces & impulses | `AddForce(Float3)` / `AddForceAt(f, atPoint)` / `AddTorque(Float3)` — last **one step**, re-apply to push continuously. `AddImpulse(Float3)` / `AddImpulseAt(i, atPoint)` / `AddAngularImpulse(Float3)` — instantaneous, velocity changes by impulse/mass, do not accumulate. Units: force kg·cm/s², impulse kg·cm/s, torque/angular-impulse the cm² equivalents. |
| Material & mass | `Friction`/`SetFriction` (0 ice, ~1 rubber, Jolt default 0.2, geometric mean of the two touching bodies); `Restitution`/`SetRestitution` (0..1, Jolt default 0); `GravityFactor`/`SetGravityFactor` (1 normal, 0 floats, negative falls up); `Damping` (linear, angular)/`SetDamping`; `Mass`/`SetMass` — **dynamic bodies only** (static/kinematic report 0), and the setter **rescales** the shape's own inertia so it keeps rotating like itself at the new weight. |
| Sleeping | `IsActive`, `Activate()`, `Deactivate()`. Every force/impulse call above already wakes the body. |
| Layer, entity, water | `Layer`/`SetLayer` (above); `SetEntity(int)`; `SetWaterVolume`/`ClearWaterVolume` (above). |
| Soft body | `SoftBodyVertexCount`, `SoftBodyVertices`, `SoftBodySkin`, `SoftBodyApplyImpulse` (above). |

Axial quantities (angular velocity, torque, angular impulse) use the axis convention the engine's
coordinate-basis change requires — get the sign wrong and a top spins the opposite way it was told to;
this is what the 0.5.0 test suite added a deliberately-wrong-conversion check for.

### `Joint` — a constraint between two bodies (`0` invalid)

Twelve constraint types, all Jolt's own, previously compiled into every build with no entry point. Every
point and axis is **world-space at the moment the joint is created** — place both bodies where they
belong, then join them. `Joint.WorldBody` (`Body.None`) as `bodyB` joins `bodyA` to an immovable frame
instead of another body — how a door hangs on a wall that is not itself simulated. A factory returns
`Joint.None` if either body is dead, both are the world, or the world is not running.

| Factory | Shape |
|---|---|
| `Fixed(a, b, point, axisX, axisY)` | Welds rigidly — no relative movement at all. |
| `Point(a, b, point)` | A ball joint: shares a point, rotates freely about it. |
| `Distance(a, b, pointA, pointB, minCm, maxCm = -1)` | A rope (min 0) or a rigid strut (min == max); negative max means "however far apart they are now". |
| `Hinge(a, b, point, hingeAxis, normalAxis, minAngleRad, maxAngleRad)` | One axis of rotation — a door, a lever, a wheel (±π or wider = unlimited). |
| `Slider(a, b, point, sliderAxis, normalAxis, minCm, maxCm)` | One axis of translation — a piston, a drawer. |
| `Cone(a, b, point, twistAxis, halfConeAngleRad)` | Free rotation within a cone, twist unlimited; 0 locks to the axis. |
| `SwingTwist(a, b, point, twistAxis, planeAxis, normalHalfConeRad, planeHalfConeRad, twistMinRad, twistMaxRad)` | An elliptical cone plus separately-limited twist — the ragdoll joint (a shoulder). |
| `SixDof(a, b, point, axisX, axisY, float[6] limitMin, float[6] limitMax)` | Six independent axis limits (`SixDofAxis` order: 3 translations cm, 3 rotations rad) for the shape none of the named joints fits — a rotating drawer, a joystick, a suspension strut. `min > max` locks an axis; ±1e30 frees it. |
| `Gear(a, b, hingeAxisA, hingeAxisB, ratio)` | Couples two bodies' **rotation** at a fixed ratio — meshed cogs. Each body needs its own hinge already; `ratio` is teeth2/teeth1, negative reverses direction. |
| `RackAndPinion(a, b, hingeAxisA, sliderAxisB, ratioRadPerCm)` | Couples a hinge's rotation to a slider's translation — a pinion turning a rack. |
| `Pulley(a, b, bodyPointA, fixedPointA, bodyPointB, fixedPointB, ratio, minLengthCm, maxLengthCm = -1)` | A rope over two fixed points — as one body descends the other rises. |
| `Path(a, b, Float3[] points, bool closed, float maxSlideCm = -1)` | Slides a body along a polyline — a roller coaster car, a camera dolly. Jolt owns the path as a separate object; `Remove()` disposes both. |

| Instance member | Description |
|---|---|
| `Remove()` | Removes the joint; the bodies stay. |
| `Bodies()` | `(Body A, Body B)` — `B` is `Body.None` for a joint to the world. |
| `Enabled` | A disabled joint stops constraining immediately (bodies fall apart) and can be re-enabled — a breakable-but-repairable link. |
| `SetMotor(MotorState, float target)` / `SetMotor(SixDofAxis, MotorState, float target)` | **Hinge, slider and six-DOF only** — every other type returns `false`. `target` is radians/centimetres, or per-second for `Velocity`. |
| `SetMotorStrength(float)` / `SetMotorStrength(SixDofAxis, float)` | The max force/torque the motor may exert; left at Jolt's default the motor is effectively unlimited — setting this is what lets it **stall**. |
| `SetLimits(float min, float max)` / `SetLimits(SixDofAxis, min, max)` | Changes a hinge's (radians) or slider's (centimetres) limits after creation. |
| `Value()` | A hinge's current angle (radians) or a slider's offset (centimetres); `null` for every other joint type or a dead handle. |

`MotorState` — `Off` (still respects limits), `Velocity`, `Position`. `SixDofAxis` — `TranslationX/Y/Z`,
`RotationX/Y/Z`.

**Coverage, stated rather than hidden:** only the seven most-reached-for joint types carry test
coverage; the six-DOF joint's Z-axis rotation limit may read inverted on an asymmetric range.

### `CharacterBody` — the walking capsule (`0` invalid)

Swept and resolved, not simulated as a rigid body — named `CharacterBody` rather than `Character`
because `Aver.Framework` already has a `Character` concept and the two would be easy to confuse across
the assembly boundary. Drawn from the **same handle counter** as `Body`, so `character.AsBody` is a
valid `Body` for anything that does not care which family it is (`SetEntity`, a raycast hit,
`Joint.WorldBody`).

| Member | Description |
|---|---|
| `Create(radius, height, position)` / `Destroy()` | `height` is the total capsule height, caps included. |
| `Velocity`/`SetVelocity` | The velocity the character **wants**; the simulation manages the vertical component unless this sets it. |
| `Position`/`SetPosition` | Teleports. |
| `Grounded` | True on ground steep enough to hold. See `GroundState` for the four-way answer. |
| `SetEntity(int)` | Equivalent to `AsBody.SetEntity`. |
| `MaxSlopeAngle`/`SetMaxSlopeAngle(radians)` | Steeper ground reports `GroundState.OnSteepGround` rather than `OnGround`. |
| `StairStepping`/`SetStairStepping(stepUpCm, stepDownCm)` | The tallest stair a step-up may climb and how far a stick-to-floor pass may pull the character back onto lost ground. Un-set reports Jolt's own defaults (40 cm up, 50 cm down), not zero. Measured: a 40 cm step lets a character cross a 25 cm kerb it used to stop dead against. |
| `GroundStateRaw` / `GroundState` (`GroundState?`) | `OnGround` (0, walking freely) / `OnSteepGround` (touching ground too steep, slides if not held) / `NotSupported` (touching something but not standing on it) / `InAir`. `-1` raw means a dead handle. |
| `GroundNormal` / `GroundPosition` | Only meaningful while `OnGround`/`OnSteepGround` — Jolt still answers *something* otherwise, typically whatever was last touched. |
| `GroundBody` | The body/character stood on, resolved like a raycast hit — `Body.None` both for standing on an unclaimed heightfield and for standing on nothing; check `GroundState` to tell those apart. |
| `GroundVelocity` | **This is what a moving platform is.** Add it to the character's desired horizontal velocity before the next update, or standing on a lift means being left behind as it rises. |
| `SetShape(radius, height, maxPenetrationCm)` | Crouching: swaps the capsule; standing back up is the same call with the original numbers. **Can fail on purpose** — growing under a low ceiling would embed the character in solid geometry, so `false` means "you cannot stand up here" and the old shape is kept. |
| `Mass`/`SetMass(kg)` | Weighs down whatever the character stands on (Jolt default 70). Never affects how the character itself moves. |
| `MaxStrength`/`SetMaxStrength(kg·cm/s²)` | The most force the character can push other bodies with; a motor character shoving a crate stalls past this. Jolt's default (~100 N) lets it shove almost anything. |

```csharp
using Aver.Physics;

// A knockback: an instantaneous impulse at the point of impact.
hit.Body.AddImpulseAt(new Float3(0f, -4000f, 1500f), hit.Point);

// A door that swings open under load and stops at 100 degrees.
Joint door = Joint.Hinge(frame, panel, hingePoint,
    hingeAxis: Float3.Up, normalAxis: new Float3(1f, 0f, 0f),
    minAngleRad: 0f, maxAngleRad: 1.745f);
door.SetMotor(MotorState.Position, target: 1.745f);
door.SetMotorStrength(600f);   // stalls against anything heavier than this can push
```

---

## 21. Animation — on `Entity`

No new P/Invoke: every member here is a field write over the generic scene ABI, exactly as `SetMesh`
is (`scripting/csharp/Aver.Framework/Animation.cs`).

| Member | Description |
|---|---|
| `bool SetSkeleton(string skeletonAsset)` | Binds a skeleton, adding `SkeletalMesh` if absent. |
| `bool PlayAnimation(string clipAsset, bool loop = true)` | Plays from the start, adding an `Animator` if absent. |
| `bool PauseAnimation()` / `ResumeAnimation()` / `StopAnimation()` | Hold in place / continue / stop and rewind. |
| `bool IsAnimating` | True while a clip is advancing (not paused). |
| `float AnimationTime` / `AnimationSpeed` / `AnimationWeight` | Seconds into the clip (settable, so a script can scrub or sync) / playback rate (1 normal, negative reverses, 0 reads as 1) / how much of the clip reaches the pose, 0..1 (0 reads as 1). |
| `int BoneCount` | 0 until the skeleton asset has resolved. |
| `bool AttachToSocket(Entity parent, string socket)` / `DetachFromSocket()` / `long AttachedSocketId` | Rides a named socket on `parent`'s rig every frame — a weapon in a hand. Parenting is half of it deliberately: this is `SetParent` plus a socket name, not a second notion of "attached to". The socket must exist on the parent's skeleton; a name that matches nothing leaves the entity where it is rather than snapping it to the parent's origin. |
| `float GetAnimationCurve(string curve, float fallback = 0)` / `bool TryGetAnimationCurve(string curve, out float value)` | Reads a named float curve off the playing clip at the current playhead — "how far through the reload am I". Not a notify: a curve always has a value, a notify fires once. Use the `Try` form when "not there" needs to read differently from "reads zero". |
| `bool SetControlRig(string rigAsset, float weight = 1.0f)` | **New in 0.5.0.** Binds a control rig, so the sampled pose is modified procedurally before skinning — IK, aim constraints, anything the `.ocrig` names. The odd one out on this page: `CControlRig` is registered **at runtime**, not built in, so it is attached by name (there is no `Component` enum value for it — see [§6](#6-entity--the-handle-to-a-thing-in-the-world)) and only works at all in a host that registered it. `weight` is written on every call, deliberately: a component's storage arrives zero-filled with no constructor run, so the field's own `= 1.0f` in-class default never executes here — leaving it unwritten would attach a rig that scales every operation to nothing, indistinguishable from one that simply does not fit the skeleton. |

A control rig can also be attached from an Aver Node graph via the `SetControlRig` node, wired through
the same `Entity.SetControlRig` above and a new `aver_scene_component` export that resolves `CControlRig`'s
runtime-registered id by name — the same mechanism the `HasComponent(string)`/`AddComponent(string)`
overloads in [§6](#6-entity--the-handle-to-a-thing-in-the-world) use.

---

## 16. `Aver.UI` — the game's HUD

Namespace `Aver.UI`, in the assembly of the same name (`scripting/csharp/Aver.UI/Hud.cs`). It references
nothing else — a HUD is drawn in screen coordinates and knows nothing about entities or the world — and
P/Invokes the native `Aver.UI.Abi` (`modules/ui.abi/include/aver/ui/ui_abi.h`), which is the seam that
exists so a HUD can be authored in the game's language rather than in engine C++.

**Screen pixels, top-left origin, +Y down.** Not the world's centimetres and not normalised coordinates:
a UI is authored against a resolution, and every layout number a designer types is a pixel.

Three rules carry most of the design:

- **You draw from a tick; the HOST owns the frame.** The host clears the list once per frame *before*
  anything ticks and submits it after (`aver_ui_begin_frame` and `submitGameUi` in
  `sandbox/src/SandboxApp.cpp`). There is deliberately no `Begin` for a game to call: a game that
  cleared the list would erase whatever another system had contributed, and the last one to run would
  win with nothing anywhere to say so. One consequence to expect — gameplay ticks only while `Playing`,
  so a HUD drawn from `OnTick` is absent in EDITOR, which is correct rather than a bug to chase.
- **Anchor to `Hud.Viewport`, never to the window.** In the editor the game is drawn into a dockspace
  panel and the two rectangles differ; a HUD laid out against the window sits partly under the editor's
  own chrome. In a shipped build they are the same rectangle and nothing changes.
- **There is no text.** Nothing in the engine can rasterise a glyph, so there is no `Hud.Text` to call.
  A HUD today is rectangles, bars and tinted quads. There is no widget tree above the draw list either:
  a HUD is laid out by the code that draws it.

**`Hud`** — static, and the whole drawing surface:

| Member | Description |
|---|---|
| `Rect Viewport` | The rectangle this frame's UI is laid out against, in backbuffer pixels. |
| `Layer CurrentLayer` | The band subsequent draws land in. **Reset to `Layer.Content` at the start of every frame**, so set it in each frame that wants another band. An out-of-range value is ignored, not clamped — clamping would move a widget to a band its author did not choose and nothing would notice. |
| `void Box(Rect r, Colour c)` / `void Box(float x, float y, float w, float h, Colour c)` | A solid rectangle. |
| `void Frame(Rect r, float thickness, Colour border, Colour fill)` | A border drawn **inside** the bounds: a frame that grew its own bounds would not fit the layout that positioned it, and every caller would subtract the thickness back off by hand. |
| `void Bar(Rect r, float fraction, Colour border, Colour empty, Colour full)` | A 1px-framed bar filled left to right; `fraction` is clamped to 0..1. |
| `void Image(Rect r, ulong texture, Colour tint, float u0 = 0, float v0 = 0, float u1 = 1, float v1 = 1)` | A textured quad. |
| `void PushClip(Rect r)` / `void PopClip()` | Nested clips **intersect**, so a child can never escape its parent by pushing a larger rectangle. That is containment by construction, not a promise each element keeps. |
| `int DrawCallCount` | Draw calls this frame's UI will cost — the whole list, so it includes anything the host contributed. For a debug readout, not for logic. |

`Image`'s `texture` is `0` for the built-in white texel, or a value the renderer recognises. **There is
no managed way to obtain one of those values yet**, so in practice a game passes `0` and tints; a
texture bound this way must also be premultiplied, because the colour is.

**`Layer`** — `Background` (0), `Content` (1, the default), `Overlay` (2), `Tooltip` (3), `Debug` (4).
Coarse and named on purpose: a widget picks the band it belongs in rather than a depth number that would
make every z decision global, so adding a tooltip does not mean auditing every other widget. Order
*within* a layer is the order you drew in.

**`Colour`** — `Colour.Rgb(232, 228, 220)` (opaque unless a fourth byte is given), `Colour.Argb(0xFFE8E4DC)`,
`WithAlpha(float)` (clamped 0..1), `Colour.Lerp(from, to, t)`. Alpha is **straight**: the
premultiplication the renderer needs happens on the native side, and the packing the vertex actually
holds is internal precisely so a game never carries it around.

**`Rect`** — a `readonly record struct (float X, float Y, float Width, float Height)` with `Right`,
`Bottom`, `Inset(by)` (negative grows), and:

```csharp
Rect.Anchored(Rect container, float ax, float ay, float w, float h,
              float offsetX = 0f, float offsetY = 0f)
```

Anchoring rather than absolute placement, because a HUD outlives the resolution it was authored at: an
element pinned to the bottom-right by subtraction walks off-screen the moment the window is smaller than
the number that was subtracted.

```csharp
using Aver.Framework;
using Aver.UI;

[AverClass("BP_PlayerHud")]
public sealed class PlayerHud : AverActor
{
    // PostPhysics: draw what the frame settled on, not what it was asked for.
    public static void Configure(ClassBuilder b) => b.Ticks(TickGroup.PostPhysics);

    static readonly Colour Ink   = Colour.Argb(0xFFE8E4DC);
    static readonly Colour Panel = Colour.Argb(0xB0141820);   // 69% alpha — proves the blend
    static readonly Colour Hurt  = Colour.Rgb(232, 76, 46);

    [Editable(Min = 0f, Max = 1f)] public float Health = 1f;

    public override void OnTick(float dt)
    {
        Rect vp = Hud.Viewport;                       // NOT the window
        Rect bar = Rect.Anchored(vp, 0f, 1f, 260f, 18f, offsetX: 24f, offsetY: -24f);

        Hud.PushClip(vp);                             // nothing escapes into the editor's chrome
        Hud.Box(bar.Inset(-6f), Panel);               // Layer.Content — the default, every frame
        Hud.Bar(bar, Health, Ink, Panel, Colour.Lerp(Hurt, Ink, Health));
        Hud.PopClip();

        if (Health <= 0f)                             // a full-screen wash, over the HUD
        {
            Hud.CurrentLayer = Layer.Overlay;         // set per frame; it does not persist
            Hud.Box(vp, Hurt.WithAlpha(0.35f));
        }
    }
}
```

---

## 17. `Aver.Materials` — surfaces authored in C#

Namespace `Aver.Materials` (`scripting/csharp/Aver.Materials/`). Like `Aver.UI` it is a leaf: a material
describes a surface and knows nothing about entities, the world or a device.

**The C# is the source; the `.ocmat` is build output.** A `.cs` under `Content/Materials` declares a
surface. *Compile C#* builds it into the project's one script assembly, then `avermatc`
(`scripting/csharp/Aver.MaterialCompiler/`) reflects over that assembly, runs every `Configure`, and
writes one `.ocmat` per material into `<project>/Binaries/Materials`. The engine looks in
`Binaries/Materials` **first**, then `Content/Materials`, then a content-relative path
(`SandboxApp::materialForSurface`) — so a project that has adopted C# materials gets the built file and
one that has not keeps working exactly as it did with hand-authored `.ocmat`.

That ordering is why editing the generated file is a mistake with a delay on it: it is a build artefact,
the next compile overwrites it, and the header it carries says so. The Details panel's **Save to C#**
writes back to the `.cs` for the same reason (`modules/formats/include/aver/formats/MaterialScript.hpp`);
it rewrites only the body of `Configure`, preserves the `.Comment(…)` calls — authored prose that cannot
be re-derived from a material's numbers — and declines outright rather than half-succeeding on a file it
did not parse.

**Declaring one.** `[AverMaterial("M_Crate")]` gives the **bound name** a mesh references. The name is
given in the attribute rather than taken from the class name for the reason `[AverClass]` does it:
renaming a C# class should not silently unbind every mesh that used it. Two classes claiming one name is
refused outright, not last-one-wins, since which of the two won would depend on reflection order.

`Configure` is **`static`**. A material is never instantiated and never ticks — it is a description the
compiler runs once, at build time, and nothing about it exists at run time — so there is no instance for
a virtual call to dispatch on. The compiler finds it by reflection and reports a `[AverMaterial]` class
that lacks one. `Material` is the base the templates derive; it carries nothing and exists so a material
is findable by type rather than only by attribute.

```csharp
using Aver.Materials;

namespace MyGame.Materials;

[AverMaterial("M_Crate")]
public sealed class Crate : Material
{
    public static void Configure(MaterialBuilder b) => b
        .Comment("Painted wood, scuffed. Roughness stays high — a gloss here reads as plastic.")
        .BaseColor(0.62f, 0.48f, 0.31f)
        .Metallic(0f)
        .Roughness(0.78f)
        .Texture(Slot.BaseColor, "Textures/crate_basecolor.png")
        .Texture(Slot.Normal,    "Textures/crate_normal.png");
}
```

Bind it from a class recipe with `b.Mesh("Meshes/crate.ocmesh", "M_Crate")`, or at run time with
`entity.SetMaterial("M_Crate")`.

**`MaterialBuilder`** — every call returns the builder, so a declaration is one chain:

| Member | Description |
|---|---|
| `Comment(string)` | A line written into the output above the parameters. Carried through on purpose: the generated file is what somebody debugging a surface opens, and a bare table of numbers with the reasoning left behind in a `.cs` is how a value becomes mysterious. |
| `Shader(Shading)` | The shading model. `Shading.Standard` — the metallic-roughness BRDF — is the only one. |
| `Blending(Blend, float cutoff = 0.5f)` | `Opaque` / `Masked` (the cutoff applies here only) / `Translucent` / `Additive`. |
| `Culling(Cull)` | `Back` / `Front` / `None`. `None` implies two-sided. |
| `CastShadow(bool)` | On by default. |
| `WorldUv(bool)` / `Tiling(float centimetres)` | Project UVs from world space at N cm per tile — what untextured blockout geometry wants. `Tiling` is written whether or not world UVs are on, so toggling the mode off and back on does not reset it. |
| `BaseColor(r, g, b, a = 1)` | Linear multiplier. |
| `Metallic(v)` / `Roughness(v)` | Multipliers, both defaulting to 1 — that is, "whatever the metalRough texture says". |
| `Emissive(r, g, b)` | Added, not multiplied, so it lights itself and nothing else. |
| `NormalScale(v)` / `OcclusionStrength(v)` | Strength of the normal map / of baked occlusion. |
| `Reflectance(v)` / `F90(v)` | Dielectric reflectance at normal incidence (0.04 is almost every non-metal) and at grazing incidence. |
| `Ior(v)` | New in 0.5.0. Refractive index of the substrate — 1.0 vacuum, ~1.33 water, ~1.5 window glass, ~2.42 diamond. Sets the critical angle for total internal reflection, and is the quantity `Reflectance` is physically derived from: setting one without the other can describe a substance that does not exist. Defaults to 1.5; only written to the `.ocmat` when changed from that. |
| `Transmission(v)` | New in 0.5.0. `[0,1]` how optically see-through the substrate is — pulls blended coverage toward `1 - transmission` before the view-angle Fresnel lifts it back, and scales the diffuse lobe so a transmissive surface does not also scatter its full base colour back at the viewer on top of what passes through. **Not** refraction: light does not bend. Omitted from the `.ocmat` at 0. |
| `CoatWeight(v)` / `CoatRoughness(v)` / `CoatF0(v)` | New in 0.5.0. `[0,1]` clear coat over the base material — car paint, varnish, wet stone — plus the coat film's own roughness and its normal-incidence reflectance (0.04, IOR 1.5, is ordinary lacquer). Whether the renderer *evaluates* the coat at all is a project-wide setting (`RENDER.LAYEREDBSDF`), not this call. All three are omitted from the `.ocmat` while `CoatWeight` is 0. |
| `SubsurfaceWeight(v)` / `SubsurfaceRadius(v)` | New in 0.5.0. A wrap-diffuse weight `[0,1]` (0 is the feature's own off switch) plus a thickness proxy that widens the view-dependent back-scatter lobe — what makes a leaf or an ear light up with the sun behind it. **Not a BSSRDF**: no transport across the mesh, no per-texel thickness. `SubsurfaceRadius` is meaningless while the weight is 0, and both fit the material block's existing padding, so authoring neither changes a material's size. Omitted from the `.ocmat` while the weight is 0 — gates byte-identical output at the shipped default. |
| `Texture(Slot, string contentRelativePath)` | Bind a texture. The path is relative to the project's content root; backslashes are normalised to forward slashes. |

**Volume absorption is not on this list.** `attenuationColor`/`attenuationDistance` (glTF
`KHR_materials_volume` — tint and per-channel exponential falloff for what a transmissive material does
to light passing through it) reached `.ocmat` and the material graph in 0.5.0, but `MaterialBuilder` has
no `AttenuationColor`/`AttenuationDistance` call. A C#-authored material cannot describe volume
absorption yet; hand-author the `.ocmat` fields or drive them from a material graph instead.

**`Slot`** — `BaseColor`, `MetalRough` (metallic in blue, roughness in green: the glTF packing),
`Normal`, `Occlusion`, `Emissive`.

**Colour space is a property of the slot and is deliberately not authorable.** `Texture` takes no
colour-space argument. Base colour and emissive are always sRGB, normal is always a normal map, and the
rest are always linear; letting a caller choose would only let a caller choose wrong. The emitter writes
the slot's space into the `TEX` line and the reader validates it, so the two cannot quietly disagree.

> The `.ocmat` grammar has **two writers** — this builder and `modules/formats/src/OcMat.cpp` — which is
> a real cost, paid so that baking a material does not require a loaded engine on the build path. The
> guard against them drifting is weaker than it sounds, and worth knowing before trusting it:
> `testGeneratedByCsharp` in `tests/formats/src/MaterialTest.cpp` parses a **pasted, verbatim** sample
> of `MaterialBuilder.Emit`'s output with the C++ reader. Nothing runs the C# emitter, so a change made
> only to `Emit` goes unnoticed until somebody re-pastes the sample. (The comment in `MaterialBuilder.cs`
> claiming a `MaterialCompilerTest` does this is wrong — no such test exists.)

---

## 11. Attributes

| Attribute | Use |
|---|---|
| `[AverClass("Name")]` | Names a gameplay class. Optional `Parent = "…"` (inferred from the base type otherwise). |
| `[AverGameMode("Name")]` | Names a GameMode. `DefaultPawnClass = "…"`, `PlayerControllerClass = "…"` name (by string) the pawn and controller it spawns. |
| `[Editable]` | A field editable in the Details panel and stored in native storage (survives Play/Stop and hot reload). Optional `Min` / `Max`. |
| `[Model]` | Marks a model-tree field the editor manages. |



## 12. Enums

| Enum | Values |
|---|---|
| `BeginReason` | `Spawn` (0), `Play` (1), `Reload` (2) |
| `EndReason` | `Destroy` (0), `Stop` (1), `Reload` (2), `Travel` (3) |
| `TickGroup` | `PrePhysics` (0), `Physics` (1), `PostPhysics` (2) |
| `PlayState` | `Editor` (0), `Playing` (1), `Paused` (2) |
| `CameraView` | `FirstPerson`, `ThirdPerson` |
| `Component` | `Local` (1), `World` (2), `Hierarchy` (3), `Name` (4), `Tags` (5), `MeshRenderer` (6), `Light` (7), `Camera` (8), `SkeletalMesh` (9), `Animator` (10), `ParticleEmitter` (11) |



## 13. `ClassBuilder` and `ActorBuilder`

`ClassBuilder` writes a class's **defaults** — the recipe run once per class, in `static void
Configure(ClassBuilder b)`:

| Member | Description |
|---|---|
| `void Mesh(string meshPath, string material = "")` | Give the class's own entity a mesh (and optional material). |
| `void PointLight(float intensityLux, float rangeCm)` | Add a point light. |
| `void Camera(float fovDegrees, float nearCm, float farCm)` | Add a camera. |
| `void Ticks(TickGroup group, int order = 0)` | Ask the class to tick, in `group`. |

`ActorBuilder` writes the editor-owned **model tree** inside `BuildModels`; hand code rarely touches it.
`ModelHandle` is the handle a placed model returns.

**How a mesh path resolves.** `Mesh("Meshes/crate.ocmesh")` stores no string: the path is hashed
(fnv1a64) to the `u64` ObjectId the `CMeshRenderer` field actually holds. Every `.ocmesh` under the
project's `Content` is loaded at project open and registered under the hash of its **content-relative
path with forward slashes** — so that exact spelling is the contract between a C# class, a level
placement and the loader, and a path spelt any other way hashes to a different number and silently draws
nothing. Two primitives are registered without a file behind them: `Meshes/cube.ocmesh` and
`Meshes/sphere.ocmesh`. The `material` argument is a *name*, resolved to its `i32` handle the same way
`Entity.SetMaterial` resolves one — see [§17](#17-avermaterials--surfaces-authored-in-c) for where that
name comes from.



## 15. Maths — `Vec3`, `Quat`, `Rot`

In `Aver.Scene` (centimetres; +X forward, +Y right, +Z up; left-handed).

- **`Vec3`** — `float X, Y, Z`; `new Vec3(x, y, z)`; statics `Forward` `(1,0,0)`, `Right` `(0,1,0)`,
  `Up` `(0,0,1)`, `Zero`, `One`; `+ - *` operators.
- **`Quat`** — `float X, Y, Z, W`; static `Identity`; `Quat.FromAxisAngle(Vec3 axis, float radians)`; `*`.
- **`Rot`** — degrees a viewport gizmo edits: `float Yaw` (about up), `Pitch` (about right), `Roll` (about
  forward); `new Rot(yaw, pitch, roll)`; `Rot.Zero`; `Quat ToQuat()`.

---

## 18. Worked examples

### A player character (WASD + mouse, first/third person)

```csharp
[AverClass("BP_Hero")]
public sealed class Hero : AverCharacter
{
    public static void Configure(ClassBuilder b)
    {
        b.Mesh("Meshes/hero.ocmesh");
        b.Ticks(TickGroup.PrePhysics);
    }

    public override void OnTick(float dt)
    {
        DriveWithInput(dt);                                    // WASD walks, mouse turns
        if (Input.GetKeyDown(Key.V))                           // V toggles the view
            CameraViewMode = CameraViewMode == CameraView.ThirdPerson
                ? CameraView.FirstPerson : CameraView.ThirdPerson;
    }
}
```

### A GameMode that hands out that character

```csharp
[AverGameMode("BP_GameMode", DefaultPawnClass = "BP_Hero", PlayerControllerClass = "PlayerController")]
public sealed class MyGameMode : AverGameMode
{
    public override void OnPostLogin(Entity controller) =>
        Log.Info($"{controller.Name} logged in, driving {controller.As<AverPlayerController>()?.Possessed.Name}");
}
```

> `DefaultPawnClass`/`PlayerControllerClass` above set the same native class fields an Aver Node graph
> sets with a `CLASS <Name> GameMode pawn=<className> controller=<className>` line, and begin-play
> possesses through the same path either way — C# and graphs are two writers to one mechanism, not two
> different ones. As of 0.3.0 the graph path actually possesses what it names; before, a graph-only
> GameMode could declare a pawn and controller and still render from a default camera. The engine now
> ships a game built entirely that way — **First Person**, five graphs and one map, no `.cs` file
> anywhere in it.

### A spawner that scatters pickups and tags them

```csharp
[AverClass("BP_Spawner")]
public sealed class Spawner : AverActor
{
    const uint Pickup = 0x1;

    public override void OnBeginPlay(BeginReason reason)
    {
        for (int i = 0; i < 5; i++)
        {
            Coin? c = Spawn<Coin>(new Vec3(i * 100f, 0f, 20f));
            c?.Self.AddTag(Pickup);
        }
        Log.Info($"{System.Linq.Enumerable.Count(Game.WithTag(Pickup))} pickups in the world");
    }
}

[AverClass("BP_Coin")]
public sealed class Coin : AverActor
{
    public static void Configure(ClassBuilder b) => b.Mesh("Meshes/coin.ocmesh");
}
```



## 19. Not yet available

The gameplay object model above is complete. What follows is what a script still **cannot** reach,
stated as absences rather than left to be discovered:

- **Audio.** The pieces exist — a mixer with voices, 3D panning and attenuation, a WASAPI device, the
  `.ocaudio` container, and a C seam (`modules/audio`, `modules/audio.wasapi`, `modules/audio.abi`, and
  `modules/formats`; see `docs/AUDIO.md`). **Corrected in 0.5.0:** the device is now opened at engine
  startup (`sandbox/src/SandboxApp.cpp`) rather than only by the Sound Editor's own preview button — so
  every `PlaySound`/`PlaySoundAt` node fired during a real Play session before this release was a silent
  no-op regardless of what the 0.4.0 changelog claimed, and now is not. That fix is on the native/Aver
  Node side only: there is still **no managed C# binding** onto the mixer, so a C# script still cannot
  play a sound.
- **Text, and widgets, in the UI.** `Aver.UI` (§16) draws rectangles and tinted quads. Nothing in the
  engine rasterises a glyph, so there is no `Hud.Text`; and there is no widget tree above the draw list,
  so layout is whatever the drawing code computes. There is also no input routing into the UI — a HUD is
  drawn, not clicked.
- **Timers and coroutines.** No `SetTimer`, no `yield`. Count down in `OnTick`.
- **Networking.** `modules/net` exists; nothing in it is reachable from C#.
- **Hot-reload state migration** beyond `[Editable]` fields.
- **Volume absorption from `MaterialBuilder`.** `attenuationColor`/`attenuationDistance` reached
  `.ocmat` and the material graph in 0.5.0; the C# builder has no call for either — see [§17](#17-avermaterials--surfaces-authored-in-c).
- **A real gamepad.** The ABI shape (`GamepadButton`, `GamepadAxis`, `Input.GetGamepadButton`/
  `GetGamepadAxis`) landed in 0.5.0; nothing polls a physical device into it yet — see [§8](#8-input).
- **Import beyond glTF.** The Content Browser's Import *converts* `.gltf`/`.glb` into one `.ocmesh` per
  mesh and registers the result immediately (`SandboxApp::importModel`); every other file type is copied
  in unchanged. That is right for a `.png` a material names — a texture is decoded and uploaded straight
  from the source file, with its colour space taken from the slot that binds it — and wrong for an FBX,
  which is not read at all. No texture is converted, compressed or packed at import.

These are the natural next layers; the object model is the foundation they plug into.

---

