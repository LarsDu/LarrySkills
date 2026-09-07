---
name: godot-csharp-migration
description: Migrating Godot 4.x GDScript to C# and writing idiomatic C# in the engine — API surface, exports, signals, onready, collections marshaling, Variant conversion. Consult when a project is partially or fully C# and you are adding or porting node scripts.
---

# Godot GDScript-to-C# Migration

Scope: idiomatic C# for Godot 4.x node scripts — naming, lifecycle, exports, signals, collections, Variant conversion. Does not cover engine internals like server APIs.

## API surface

Do:

- Godot API is PascalCase and property-based, not getters/setters. `node.Name = "x"`, not `node.set_name("x")`. `node.Position += Vector2.Right`.
- Lifecycle: `public override void _Ready()`, `_Process(double delta)`, `_PhysicsProcess(double delta)`.
- Node access: `GetNode<T>("Path")`, `GetParent<T>()`, `GetTree()`. Singletons are static: `Input.IsActionPressed("jump")`; use `.Singleton` for `GodotObject` members.

Don't copy GDScript getters: `set_name()` / `get_name()` etc. are not the C# API.

## Global scope equivalences

| GDScript | C# |
| --- | --- |
| `sin(x)`, `lerp(a,b,t)`, `PI` | `Mathf.Sin`, `Mathf.Lerp`, `Mathf.Pi` |
| `print()`, `randf_range()` | `GD.Print`, `GD.RandRange` |
| `is_instance_valid(x)` | `GodotObject.IsInstanceValid(x)` |
| `preload("res://...")` | `GD.Load("res://...")` / `ResourceLoader.Load` |

Note: there is no `preload` in C#; do the load in `_Ready()` or a static field via `GD.Load`.

## Exports

Do:

```csharp
[Export] public int MaxHealth = 100;
[Export(PropertyHint.Range, "0,100,1")] public float Damage = 10f;
```

Don't: export with `public` non-property fields thinking the hint is optional and default assignment is auto-applied. Godot saves the default value you assign, so assigning in the field initializer is correct. Connections/edit-time visibility only appear after a (re)build.

## Signals

GDScript `signal died(where: Vector2)` becomes:

```csharp
[Signal] public delegate void DiedEventHandler(Vector2 where);
```

Do:

- `EmitSignal(SignalName.Died, position)` to raise; `Died += handler;` to subscribe, `Died -= handler;` to unsubscribe.
- Give signal args Variant-compatible types. Custom payload classes must inherit `GodotObject`.

Don't:

- Forget to unsubscribe with `-=` in `_ExitTree`. Missed disconnects surface as `System.ObjectDisposedException` later. Watch for it.

## `@onready` — no equivalent

GDScript `@onready var lbl := $Label` has no C# counterpart. Resolve in `_Ready()`:

```csharp
private Label _lbl;
public override void _Ready() { _lbl = GetNode<Label>("Label"); }
```

## Collections — the marshaling cost rule

Godot collections (`Godot.Collections.Array`, `Array<T>`, `Dictionary`) are C++-backed wrappers that marshal on EVERY element access. This is the biggest perf trap in C# Godot.

Do:

- Use Godot collections only at engine boundaries: exported arrays, RPC arguments, signal payloads, anything handed to a Godot API.
- Use .NET generics everywhere else — `List<T>`, `Dictionary<K,V>`, `System.Array` (`byte[]`, `float[]`, `Vector2[]`).
- Prefer Godot collection instance methods (`List.Sort()`, `.Add()`) over LINQ, which iterates and marshals per call in hot paths.

Don't: put a `List<Enemy>` of gameplay state in a `Godot.Collections.Array` — you pay marshaling on every `nameof` access, every frame.

## Variant conversion

- Implicit `T -> Variant` conversion is automatic (e.g. passing a `Vector2` into a `Variant` slot).
- Explicit `Variant -> T` is `Variant.From<T>` or `Variant.CreateFrom<T>`; wrong-type explicit conversions can yield surprising values silently. Use `Variant.Obj` to switch on `VariantType` if you need to branch on type.

## Class system

- Every node script must be a `public partial class` (declaration may include `partial`; the generated bridge code requires it).
- Use `GodotObject`/`Resource` subclasses for non-Node data, not `Node`-derived classes.

## Performance

- C# has no GDScript interpretation overhead, but still pays marshaling when crossing into engine APIs. Hot loops over engine calls (arrays, RPCs) move cost; profile with dotnet tooling rather than guessing.

## Sources

- https://docs.godotengine.org/en/stable/tutorials/scripting/c_sharp/c_sharp_differences.html
- https://docs.godotengine.org/en/stable/tutorials/scripting/c_sharp/c_sharp_exports.html
- https://docs.godotengine.org/en/stable/tutorials/scripting/c_sharp/c_sharp_signals.html
- https://docs.godotengine.org/en/stable/tutorials/scripting/c_sharp/c_sharp_collections.html
- https://docs.godotengine.org/en/stable/tutorials/scripting/c_sharp/c_sharp_variant.html
