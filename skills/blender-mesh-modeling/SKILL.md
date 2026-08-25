---
name: blender-mesh-modeling
description: Blender procedural mesh modeling and editing via the Python API (bpy/bmesh/bpy.ops). Use whenever creating, modifying, UV-unwrapping, triangulating, or repairing meshes inside Blender, whether through Blender's own script execution (blender --background --python) or an MCP-style socket bridge (e.g. blender-mcp) that runs generated code inside a Blender session.
---

# Blender mesh modeling (bpy / bmesh)

Targets Blender 4.x (bpy API current as of the 4.5 generation of stubs). Scope: procedural creation and topology manipulation of meshes, UV unwrapping, and topology hygiene (N-gons, non-manifold geometry). Does not cover materials/compositing, rigging/animation, or high-level asset-pipeline orchestration.

## Execution model matters first

Two ways to run bpy code. The practical recommendation: write code that never depends on `bpy.ops` or ambient editor context, so it works in either model. The bmesh object-mode construction path is the safe default.

- Mode A: `blender --background --python script.py`. Full API, zero bridge, but no window/area context — `bpy.ops` poll-fails often and there is no live viewport feedback.
- Mode B: MCP-style socket bridge (blender-mcp architecture). An addon in GUI Blender (or under `xvfb-run -a blender`) owns a socket; external MCP server forwards commands, which queue onto Blender's main thread (bpy is not thread-safe). Gives live viewport screenshots and richer feedback, but refuses `--background`.

Whatever the mode, never assume ambient selection/active-object state. Set it explicitly or avoid `bpy.ops` entirely.

## Choosing the right mesh API

- `bmesh` for topology mutation (extrude, bevel, inset, dissolve, booleans, editing UVs). It is a full editable copy that you must `free()`.
- `Mesh.from_pydata` for simple one-shot static construction from plain vertex/face arrays (no repeated edits).

### from_pydata misuse

Do:
```python
me = bpy.data.meshes.new("MyMesh")
me.from_pydata(verts, [], faces)   # faces = list of index sequences
me.update()                        # refresh loops/edges/tessellation caches
```

Don't:
```python
me.from_pydata(verts, edges, [0, 1, 2, 3])  # faces entries must be indexable sequences, not bare ints
```
Out-of-range face indices silently drop vertices and produce "internal error setting the array". Always call `me.update()` after populating.

## The bpy.ops context trap (the #1 agent failure)

`bpy.ops.*` operators poll their context (active object, selection, mode) and raise if it is missing. In an agent loop, do not assume anything is selected, active, or in the right mode.

Don't:
```python
bpy.ops.object.duplicate()   # poll fail: no active object
bpy.ops.mesh.subdivide()     # poll fail: not in edit mode
```

Do (2.8+ API — note `view_layer.objects.active`, not `scene.objects.active`):
```python
bpy.ops.object.select_all(action='DESELECT')
obj.select_set(True)
bpy.context.view_layer.objects.active = obj
bpy.ops.object.mode_set(mode='EDIT')
bpy.ops.mesh.select_all(action='SELECT')
bpy.ops.mesh.normals_make_consistent(inside=False)
bpy.ops.object.mode_set(mode='OBJECT')
```

## bmesh lifecycle

Edit-mode bmesh requires the object to actually be in edit mode, which itself requires GUI/ops context. For agent code, prefer the object-mode path — it is headless-safe and context-independent.

Don't:
```python
bm = bmesh.from_edit_mesh(me)   # RuntimeError unless me is in edit mode
```

Do (object mode — the preferred default for generated code):
```python
bm = bmesh.new()
bm.from_mesh(me)                      # or bm.from_object(ob, depsgraph)
bm.normal_update()                    # recalc normals
bm.transform(mat)                     # apply transforms directly on bm
bm.to_mesh(me)
bm.free()                             # must free explicitly
```

Do (in-GUI/edit-mode contexts only):
```python
bpy.context.view_layer.objects.active = obj
bpy.ops.object.mode_set(mode='EDIT')
bm = bmesh.from_edit_mesh(obj.data)
# ... mutate ...
bmesh.update_edit_mesh(obj.data, loop_triangles=True)
bpy.ops.object.mode_set(mode='OBJECT')
```

## N-gons and triangulation

N-gons are valid Blender faces (quads/tris are a special case), but they carry hidden internal triangulation and break booleans, sculpting, animation deformation, and many export checks. Triangulate at the end for game assets.

Do:
```python
bm = bmesh.new()
bm.from_mesh(me)
bmesh.ops.triangulate(bm, faces=bm.faces, quad_method='BEAUTY', ngon_method='BEAUTY')
bm.to_mesh(me)
bm.free()
```

## Non-manifold geometry

Caused by internal faces, disconnected elements, or zero-thickness areas; breaks booleans, modifiers, refraction rendering, and 3D printing. Detect via `select_non_manifold` (in edit mode); cheap fix:

```python
bmesh.ops.dissolve_degenerate(bm, edges=bm.edges, dist=0.00001)  # kill zero-area/zero-length
```

## UV unwrapping

- Operator path (needs edit mode + active/selected object):
  ```python
  bpy.ops.object.mode_set(mode='EDIT')
  bpy.ops.mesh.select_all(action='SELECT')
  bpy.ops.uv.smart_project(island_margin=0.02)               # robust agent default
  # or: bpy.ops.uv.unwrap(method='ANGLE_BASED', margin=0.001)
  bpy.ops.object.mode_set(mode='OBJECT')
  ```
- Headless-safe bmesh path (no ops/context; note `bmesh.ops` exposes NO unwrap operator in 4.x stubs):
  ```python
  uv = bm.loops.layers.uv.new("UVMap")
  for f in bm.faces:
      for loop in f.loops:
          loop[uv].uv = (loop.vert.co.x, loop.vert.co.y)
  ```

## Naming and collection hygiene

Generated content should be isolated so the agent's work is easy to delete and never collides with user data.

Do:
```python
col = bpy.data.collections.new("Agent_Props")
bpy.context.scene.collection.children.link(col)
me = bpy.data.meshes.new("Prop_HexBolt")
ob = bpy.data.objects.new("Prop_HexBolt", me)
col.objects.link(ob)
```

Conventions: `Category_Part` names (the auto-`.001` suffix guarantees uniqueness), and one dedicated child collection per agent session.

## Workflow (Verify-loop)

1. Inspect the current scene/objects before mutating anything (get_scene_info / get_object_info equivalent if a bridge exposes them).
2. Act in small, idempotent chunks — one script per logical step, not one giant build.
3. Verify after each step: read back `obj.dimensions` / world bounding box, take a viewport screenshot when available.
4. Only touch the assets within scope of the request; leave the rest of the scene alone.

## Sources

- blender-mcp: https://github.com/ahujasid/blender-mcp (addon.py / server.py)
- arjun988/blender-skills: https://github.com/arjun988/blender-skills (skill format + orchestrator pattern)
- bpy.types.Mesh.from_pydata: https://docs.blender.org/api/current/bpy.types.Mesh.html
- bmesh module & types: https://docs.blender.org/api/current/bmesh.html ; https://docs.blender.org/api/current/bmesh.types.BMesh.html
- bmesh.ops.triangulate: https://docs.blender.org/api/current/bmesh.ops.html
- bpy.ops.uv.unwrap / smart_project: https://docs.blender.org/api/current/bpy.ops.uv.html
- bpy.data.collections: https://docs.blender.org/api/current/bpy.data.collections.html
- Active object via view_layer (2.8+): https://blender.stackexchange.com/questions/72647 ; https://blender.stackexchange.com/questions/134175
- from_pydata face format: https://blender.stackexchange.com/questions/30890 ; https://blender.stackexchange.com/questions/40635
- N-gon/triangulation: https://blender.stackexchange.com/questions/1684
- Non-manifold detection: https://blender.stackexchange.com/questions/7910
- UV unwrap from bmesh: https://blender.stackexchange.com/questions/45270 ; https://blender.stackexchange.com/questions/41531
