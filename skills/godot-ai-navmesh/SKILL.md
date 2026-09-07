---
name: godot-ai-navmesh
description: Godot 4.x AI navigation with NavigationServer and NavMesh. Agents that need pathfinding, movement toward a target, avoidance, or baking a NavigationRegion should consult this skill before writing movement code.
---

# Godot AI Navigation (NavMesh)

Scope: NavigationMesh baking, NavigationAgent path-following, NavigationServer sync timing, and avoidance/RVO for Godot 4.x. Does not cover godot-multiplayer-* skills, character controller physics, or state machines.

## Core model

- A navmesh describes the traversable area for an agent's CENTER position. The agent's collision radius is ignored by pathfinding. Clamp edges by baking with `agent_radius`, not by relying on runtime collision.
- Navigation is independent of rendering and physics. If the visual/physical walkable area changed, the navmesh must be re-baked. There is no "just use the collision shapes" magic.
- `NavigationAgent*` never moves its parent node. You write the movement in `_physics_process`; the agent only reports where to head next.

## Baking

Do:

```gdscript
var region := NavigationRegion3D.new()
region.navigation_mesh = NavigationMesh.new()
region.navigation_mesh.agent_radius = 0.5          # margin for locomotion
region.navigation_mesh.cell_size = 0.25            # match map/grid scale
region.navigation_mesh.parsed_geometry_type = NavigationMesh.PARSED_GEOMETRY_STATIC_COLLIDERS
region.navigation_mesh.collision_mask = 0b0010
region.navigation_mesh.source_geometry_mode = NavigationMesh.SOURCE_GEOMETRY_GROUPS
region.navigation_mesh.source_geometry_group_name = "nav_world"
region.bake_navigation_mesh(on_thread = true)      # can run at runtime
```

Don't:

- Parse the whole SceneTree when a node group suffices. Use `source_geometry_mode` = groups and tag only the geometry agents should walk on.
- Default to the smallest `cell_size` everywhere. Raise `cell_size`/`cell_height` as far as the gameplay allows; small cells + large meshes (e.g. TileMaps) tank pathfinding.
- Use `Watershed` partition on flat/blocky geometry. Prefer `Monotone` or `Layers` for faster bakes on structured geometry; `Watershed` pays off only on organic/uneven terrain.

## Sync timing (the #1 NavMesh mistake)

NavigationServer syncs all changes at the END of the physics frame, not immediately. Setters are queued; getters the same frame return stale values.

Symptom: agent teleports to target instantly, or paths queried in `_ready()` come back empty/wrong.

Fix:

```gdscript
func _ready():
    setup.call_deferred()

func setup():
    await get_tree().physics_frame          # let the map sync once
    agent.target_position = goal

# or guard RNG-dependent queries
func is_nav_ready() -> bool:
    return NavigationServer2D.map_get_iteration_id(agent.get_navigation_map()) != 0
```

Note: avoidance uses a threadpool too — `agent_set_*` calls are queued, so `agent_get_map()` may not reflect a fresh map change until sync. Don't query immediately after setting.

## Path following

Do:

- Call `get_next_path_position()` ONCE per `_physics_process`, then move toward it.
- Return early before pathing once finished.

```gdscript
func _physics_process(delta: float) -> void:
    if agent.is_navigation_finished():
        return
    var next := agent.get_next_path_position()
    velocity = global_position.direction_to(next) * speed
    global_position += velocity * delta
```

Symptom -> cause -> fix:

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Agent dances between two points | New path requested every frame, or `path_max_distance` too small for current speed | Request a new path only on target change or periodic interval; raise `path_max_distance` |
| Agent backtracks/overshoots then corrects | Movement speed exceeds what `path_desired_distance` allows | Raise `path_desired_distance` and `path_max_distance` together |
| One-frame look-behind at navmesh edges | The path point sits slightly behind the agent's facing near mesh borders | Acceptable precision trade-off; don't "fix" by re-pathing every frame |

## Avoidance (RVO)

Avoidance is a separate simulation from pathfinding and physics:

- Off by default. Sets `avoidance_enabled = true`, connects `velocity_computed`, and must set `.velocity` every physics frame. Apply the `safe_velocity` from the signal, not your own computed velocity.
- `radius` is body size, not stopping distance.
- `neighbor_distance`, `max_neighbors`, `time_horizon_agents`, `time_horizon_obstacles`, `max_speed`, `avoidance_layers`/`mask`, `avoidance_priority`, `height` all modulate the RVO simulation.

Don't:

- Use avoidance-only without a `target_position`. Without a target, `safe_velocity` stays the zero vector.
- Rely on avoidance for pathfinding around obstacles. Obstacles do NOT affect pathfinding, only avoidance. A head-on test against a static wall will fail (RVO passes to a side by assumption).
- Keep avoidance enabled when unused — it costs a threadpool simulation. Disable `enabled_avoidance`.

Obstacles: `affect_navigation_mesh` (carves cells OUT of the rasterized mesh; nothing added) vs `avoidance_enabled` (changes agent steering). Disable what you don't need.

## Performance

- Baking knobs above (geometry scope, cell size, partition type) are the main levers.
- A sudden heavy spike on UNREACHABLE targets is expected: the search explodes over large meshes that lack simplified chunks. Simplify or split large meshes.

## Sources

- https://docs.godotengine.org/en/stable/tutorials/navigation/navigation_using_navigationmeshes.html
- https://docs.godotengine.org/en/stable/tutorials/navigation/navigation_using_navigationagents.html
- https://docs.godotengine.org/en/stable/tutorials/navigation/navigation_using_navigationservers.html
- https://docs.godotengine.org/en/stable/tutorials/navigation/navigation_using_navigationobstacles.html
- https://docs.godotengine.org/en/stable/tutorials/navigation/navigation_optimizing_performance.html
