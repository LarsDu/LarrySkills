---
name: palette-lowpoly
description: Rapid low-poly game-asset modeling via palette-texture colorization — block-model from primitives with extrude/scale/transform, then colorize by pinning UV faces onto swatches of a shared palette texture (or a photograph) instead of texture painting. Covers humanoid/quadruped/mech rigging with minimal IK armatures, weight-fix recipes, and animation/export conventions. Use when creating stylized low-poly characters, creatures, props, or when the user mentions palette textures, swatch colorization, or low-poly game assets.
---

# Palette-texture low-poly modeling and rigging

A workflow popularized in the indie low-poly game-asset community: block-model with primitives and extrude/scale/transform, then "colorize" by making UV islands infinitely small and parking them on solid-color swatches of a shared palette texture. No texture painting, no seams, no per-asset textures. One material and a few tiny textures serve every asset in the game (batching-friendly, ~1 MB VRAM, edges stay crisp at any zoom). Practitioners report ~10x speed vs. unwrap+paint; a rigged character in 10–20 minutes is a normal target.

Targets Blender 3.x–4.x+ via bpy. All code below was tested headless on Blender 4.x (`blender --background --python`).

## Golden rule of the colorization contract

Every place you want a different color needs its own polygon (loop cut / inset / extrude a face there first). In exchange, UVs are pinned to single palette pixels, so there is zero texel bleeding, no UV padding, and infinite zoom sharpness — but no per-pixel detail (stripes, decals, damage need either new geometry or a different technique).

## Palette texture and material

Textures (all PNG, never JPEG — lossy compression smears swatch colors):

- **BaseColor** — grid of solid swatches: hue rows, greys, skin tones, nature gradient columns (wood/grass/rock).
- **Emission** — same resolution, all black except the emissive swatches (same positions as their BaseColor counterparts).
- **Attributes** — per-pixel material props: **R = metallic**, **G = smoothness** (shader computes roughness = 1 − G), **B = animated-scroll flag** (1 only on the "animated strip" column/row).

Texture settings — the #1 forget-to-break-it item:

- Image Texture node interpolation **`Closest`** (not Linear) on *all three* textures, or colors blur between swatches.
- Extension **`Repeat`** (animated strip relies on UV wrapping).
- In game engines: no compression, no mipmaps (pixel-perfect sampling).

Material node graph (the canonical community "palette" material):

```
BaseColor tex ──────────────────► Principled Base Color
Emission tex ───────────────────► Principled Emission Color   (strength 10–20, HDR for bloom)
Attributes tex ─► SeparateColor ─┬─ R ──────────────────────► Principled Metallic
                                 ├─ G ─► Math(SUBTRACT, 1-G) ► Principled Roughness
                                 └─ B ─► animated UV offset (below)
```

Headless build (tested):

```python
mat = bpy.data.materials.new("PaletteMaterial"); mat.use_nodes = True
nt = mat.node_tree; bsdf = nt.nodes["Principled BSDF"]
def tex(img):
    n = nt.nodes.new("ShaderNodeTexImage"); n.image = img
    n.interpolation = 'Closest'; n.extension = 'Repeat'
    return n
t_base, t_emis, t_attr = tex(base), tex(emission), tex(attributes)
nt.links.new(t_base.outputs['Color'], bsdf.inputs['Base Color'])
nt.links.new(t_emis.outputs['Color'], bsdf.inputs['Emission Color'])
bsdf.inputs['Emission Strength'].default_value = 10.0
sep = nt.nodes.new("ShaderNodeSeparateColor")
nt.links.new(t_attr.outputs['Color'], sep.inputs['Color'])
nt.links.new(sep.outputs['Red'],   bsdf.inputs['Metallic'])
inv = nt.nodes.new("ShaderNodeMath"); inv.operation = 'SUBTRACT'; inv.inputs[0].default_value = 1.0
nt.links.new(sep.outputs['Green'], inv.inputs[1])
nt.links.new(inv.outputs['Value'],  bsdf.inputs['Roughness'])
```

Generating palette images (any solid-swatch grid works — hand-drawn, generated, or converted from pixel-art palettes such as those on lospec.com; keep it power-of-two: 8×8, 16×16, 256×256):

```python
import numpy as np
def make_palette(name, size, swatches):  # swatches: [(x, y, (r,g,b,a)), ...] pixels, y=0 is bottom row
    img = bpy.data.images.new(name, width=size, height=size, alpha=False)
    arr = np.zeros((size, size, 4), dtype=np.float32)
    for x, y, rgba in swatches: arr[y, x] = rgba
    img.pixels = arr.ravel().tolist(); img.pack()
    return img
```

Layout discipline: dedicate fixed cells — e.g. a row of skin tones, a row of greys, a metallic cell (Attributes R=1 there), an emissive cell (non-black in the Emission texture), an animated strip (Attributes B=1). Record the layout as constants so UV pinning stays exact.

Commercial "palette pack" variants extend this with tintable regions (recolor whole swatch groups via shader color parameters, so one material instance recolors all "shirts"), grunge/pattern/damage effects, and named nature gradients. The pin-UV-to-swatch mechanic is identical — treat them as drop-in upgrades of the same three textures.

### Using a photograph as the color source

A photo can replace the hand-made palette entirely. Pin UV points onto regions of a reference photo and each face picks up that pixel's color:

- **Why**: when colors must come from the real world — skin tones sampled from a face photo, fabric from a wardrobe shot, wood tones from a plank photo, brand colors from product photography — a photo gives you a ready-made, naturally harmonized "palette" of thousands of adjacent shades.
- **How**: same pinning move as below — every loop of a face gets the exact UV of the chosen pixel. Keep `Closest` interpolation so each face reads one flat texel.
- **Subtle variation for free**: pin neighboring faces to neighboring texels (e.g. walk the UV point across a cheek area, or down a wood-grain plank) and you get gentle, natural color shifts between faces — richer than a flat swatch, without any texture painting.
- **Gradients**: deliberately stretch one face's UVs across a photo's gradient region (sky, wood, sunset) for painted-looking vertical shading — the same trick as gradient columns in a synthetic palette.
- **Caveats**: photos are not flat colors — pin to the smallest possible point (never leave an island stretched unless you want the gradient), avoid busy/high-frequency regions unless you want noisy faces; the image must travel with the asset (pack it into the .blend or ship it next to the model); and colors are only as stable as the source file — for engine-portable, guaranteed-flat colors, pre-sample the photo into a swatch grid first (quantize dominant colors into an 8×8/16×16 palette image, then pin against that).

## UV swatch colorization (the core move)

Interactive equivalent: UV Editing tab → face-select the region → in the UV editor `A`, `S`, `0`, `Enter` (scale UVs to zero → a point) → `G` and drop the point on a swatch. Selection recipes: `L` select linked, `Ctrl+Numpad+`/`-` grow/shrink, `Alt+click` loop select; "UV sync selection" toggle links 3D and UV selections.

Headless equivalent — better than scaling to zero: write the **exact swatch-center UV** into every loop of the face:

```python
def swatch_uv(n, col, row_from_top):   # n = grid size (texture is n×n)
    return ((col + 0.5) / n, 1.0 - (row_from_top + 0.5) / n)

bm = bmesh.new(); bm.from_mesh(me)
uv = bm.loops.layers.uv.new("UVMap")
for f in faces_to_color:               # your selection of bmesh faces
    u, v = swatch_uv(N, col, row)
    for loop in f.loops: loop[uv].uv = (u, v)
bm.to_mesh(me); bm.free()
```

Variants:

- **Fake shading / AO**: pin "shadowed" faces (undersides, interiors, hat inner) to a *darker swatch of the same hue* instead of relying on lighting.
- **Gradient faces**: deliberately stretch one face's UVs across a gradient column (e.g. wood getting darker downward) — this is the only time UVs are not point-pinned.
- **Emissive faces**: pin to the emissive cell (same UV on BaseColor and Emission textures → glows with bloom). Eevee ≥4.2 has no Bloom checkbox — use Compositing → Glare node (Bloom) with viewport compositor set to "Always".
- **Animated/cycling colors**: pin to the animated strip. Standard driver: `V' = V − AttributesB × (frame/36)` via TexCoord→SeparateXYZ→(SUBTRACT per-channel)→CombineXYZ fed as vector to BaseColor+Emission textures; the frame divisor sets cycle length (36 frames). Static swatches have B=0 so they are unaffected.
- **Symptom → fix**: a *gradient appears across a face* = its UVs aren't point-sized / span swatches → re-pin to one cell center.

Whole-object recolor = move one UV point. Palette swap = replace the texture. Engine support: Unity handles point UVs fine; recent Unreal does too (older versions didn't).

## Block modeling workflow

1. **Start primitive** (usually the default cube), `Tab`, scale/move into the largest mass of the subject.
2. **Order for characters**: pelvis box first → split the pelvis bottom face (loop cut at x=0 for the groin) → extrude legs down from the two bottom faces → extrude torso up from the pelvis top → arms outward from the chest side faces → neck → big head. Mechs: hip block first → legs → belly → arms → head sunk into body.
3. **Ops vocabulary**: `E` extrude (twice at knees/elbows — at least one loop so limbs bend), `S`/`G`/`R` with axis locks, `Ctrl+R` loop cuts, `I` inset (`I,I` = individual), `Alt+E` extrude along face normals (belts, armor panels, hair volume; hold `Alt` mid-op for even thickness), `Alt+S` shrink/fatten, `Ctrl+B` bevel, `K` knife (adds auto-connecting edges — avoid on deformable bodies), `Shift+D` + RMB-snap-back to duplicate a face into a new part, `O` proportional edit (use `Alt+O` connected-only on multi-part meshes), `.` pivot → Individual Origins, `,` orientation → Normal (`S,Z,0` flattens).
4. **Mirror**: half-model + Mirror modifier **with Clipping on**, X axis (or the built-in *Auto Mirror* add-on). Disable clipping only momentarily (eyes, scaling at centerline) then re-enable — verts crossing center break the mesh. For asymmetric details: apply the mirror (`Ctrl+A`) as *late* as possible, then edit one side.
5. **Scale trick**: `S, Z, 0, Enter` flattens selection to Z=0 (feet flat on ground), with `.` → Individual Origins for multiple islands.

Topology rules:

- **Character bodies are ONE continuous mesh grown by extrusion** — see the next section. Disconnected shells are for rigid props and small detachable details only (eyes, hats, gear, pegs, armor plates).
- Loop cuts go **around** limbs/waist/head, never vertical bands through the body — they add polys and hurt deformation.
- Keep joint loops (knee, elbow, ankle, wrist) even on ultra-low-poly bodies; a single-bone "stilt" leg cannot bend.
- N-gons/tris are fine on **static props**; quads around joints for anything armature-deformed.
- For props, many small **disconnected parts inside one object** is the norm (select with `L`); saves cleanup.

### Extruded continuous body (characters)

Limbs are extruded out of the main body mass, never assembled from separate box shells. This is what makes skinning work: bone heat spreads smoothly across shared vertices, and joints blend naturally. A detached limb box can only ever be 100% one bone — it shears at the seam instead of bending.

- **Legs**: extrude each pelvis-bottom half straight down, extrude again past the knee (that second ring is the joint loop), flatten the foot (`S,Z,0`), stretch its front verts forward for a toe.
- **Torso**: extrude the pelvis top face up in segments — every segment ring is a natural color band (belt, shirt stripes).
- **Arms**: extrude the chest side face(s) outward as a region, then scale the new face down to arm cross-section (this makes the tapered shoulder), extrude on for upper arm → forearm → hand.
- **Neck/head**: extrude the torso top face up and scale it narrow for the neck, wide again for the head; extrude head bands for beard/eyes/crown color zones.

Headless, this is all `bmesh.ops` (context-free, unlike `bpy.ops.mesh.*`):

```python
r = bmesh.ops.extrude_face_region(bm, geom=faces)     # the E key
nv = {g for g in r['geom'] if isinstance(g, bmesh.types.BMVert)}
for v in nv: v.co.z -= 0.28                            # the G key
# optional S key: remap the new cap verts to a target cross-section (collectively,
# so shared verts move once and the cap stays fused)
cx = sum(v.co.x for v in nv) / len(nv)                 # ...scale about centroid per axis
# walls are NOT returned in 'geom' -- find them via adjacency:
new_faces = {f for v in nv for f in v.link_faces}
walls = [f for f in new_faces if not set(f.verts) <= nv]   # 2 old + 2 new verts
caps  = [f for f in new_faces if set(f.verts) <= nv]       # all-new verts = next extrude base
```

Notes: the original base face **survives** each extrusion as a single hidden interior boundary (no z-fight, but dissolve it if you need clean topology); split a face with `bmesh.ops.bisect_plane` (loop-cut equivalent — dedupe the `geom` list or it raises); call `bm.normal_update()` before classifying walls by `face.normal` for per-face colors (belt buckle front, boot toe, sole).

Character conventions (game-ready):

- Cartoon proportions: ~3 heads tall, head ≈ torso ≈ legs; exaggerate.
- Height ~1.8–2 m; **origin at (0,0,0) between the feet**; model **facing +Y** (Blender's green forward; Unity/Unreal import-friendly).
- Backface Culling on while modeling (catches flipped normals); fix with recalculate-normals-outside.
- Apply scale (`Ctrl+A`) before edit work or inset behaves wrong.

## Character detail recipes

- **Eyes** (detached, the default): duplicate a face near the face (`Shift+D` + snap back, clipping off → scale → move → extrude slightly), color black; tall vertical eyes, no nose/mouth. Cheap alt: `I` inset a face and recolor the center black. Extrude pupils slightly to avoid z-fighting.
- **Belt**: 95% of the time just recolor a waist loop black; raised belt = `Alt+E` extrude-along-normals on a waist loop.
- **Hats — integrated** (default choice): loop cut + extrude up from the head. **Detached** (knock-off-able): duplicate top faces → shape → `P` separate → **Child Of** constraint to head bone + Set Inverse (see rigging). Watch interior poke-through and fix its origin (cursor-to-selected → origin to cursor).
- **Hair**: color head faces; volume = `Alt+E` along normals then re-pin matching swatch; long hair needs weight painting (blend head + upper-spine bones).
- **Clothes that inherit weights** (the headline trick): duplicate the body's own faces (`Shift+D` snap-back → `P` separate) — the garment copies the skin's vertex weights exactly, so it deforms for free. Solid version (cheaper): `Alt+S` inflate + `F` cap rings. Sleeves: delete the end face and re-extrude from the elbow so hand weights don't leak in. New `Ctrl+R` loops interpolate inherited weights automatically. Garment verts you *move* carry wrong weights: fix via per-vertex Copy in the vertex-group panel, or `Remove from All` + assign to one bone.
- **Template cloning**: keep a hidden `_template` character; `Shift+D` copies inherit weights → variants take minutes. Cloning breaks shape keys — rebuild each from Basis (Blend from Shape → Basis 100%, then re-sculpt the expression) and keep canonical names if game-engine morph targets matter: Basis, Closed, Angry, Confused, Happy, Sad, Surprised, UpperFat/Skinny, ArmsFat/Skinny, LegsFat/Skinny, FeetBig/Small, UpperMuscles, Pregnant, HairLong/Short, Prop1-3Big/Small, Feature1-3.
- **Quadruped/animal sets**: build ONE blocky base animal, `Shift+D` + modify each copy (delete/reshape legs, ears, neck length) — keeps a consistent style across the set.

## Rigging — humanoid (custom minimal armature)

Bone structure (`.L`/`.R` suffix mandatory for Symmetrize):

| Chain | Bones | Notes |
|---|---|---|
| Center | `pelvis` → `spine 1` → `spine 2` → `head` | connected chain; pelvis bone at hips pointing up Z; slight natural S-curve |
| Arm (each side) | `shoulder.L` (disconnected, sits at clavicle) → `upper arm.L` → `lower arm.L` → `hand.L` | elbow gets a slight backward bend so IK/FK fold direction is unambiguous |
| Leg (each side) | `upper leg.L` → `lower leg.L` → `foot.L` | upper leg **parented to pelvis with Keep Offset** (dotted line, not connected); knee slight forward dent; foot bone runs heel (just above ground) → toe (at ground level) — the toe is the roll pivot |
| IK helpers (each side) | `ik leg pole.L`, `ik leg target.L` | short stubs extruded from the knee and ankle joints (side view) then clear-parented; `use_deform = False` on both; pole ends up just in front of the kneecap, target stays at the ankle |

Headless construction (tested): build `.L` side with `arm.edit_bones`, then **`bpy.ops.armature.select_all(action='SELECT')` before `bpy.ops.armature.symmetrize()`** — symmetrize silently returns CANCELLED without full selection (new edit bones aren't fully selected by default). Set `Viewport Display → In Front` so bones show through the mesh.

**Roll before anything else**: front-orthographic view + `Shift+N` (Recalculate Roll, View Axis) interactively; headless `bpy.ops.armature.calculate_roll(type='GLOBAL_POS_Z')` works. Skipping this breaks mirrored-pose paste and the pole-angle flip later.

Leg IK (per side, pose mode on the `lower leg` bone):

```python
c = pb.constraints.new('IK')
c.target = arm_ob;      c.subtarget     = 'ik leg target.L'
c.pole_target = arm_ob; c.pole_subtarget = 'ik leg pole.L'
c.chain_count = 2                      # thigh + shin only
c.pole_angle  = math.radians(90)       # knee points forward; if leg flips sideways, roll/angle is off
```

Helper-bone geometry (what makes the rig read and animate conventionally):

- `ik leg pole.L`: a short stub whose body sits just in front of / above the kneecap, pointing forward (+Y). Only its position matters to the solver.
- `ik leg target.L`: a short stub with its **head on the ankle joint**, pointing forward roughly parallel to the foot bone, kept at or above ground level. Its head position is the IK goal; its local rotation is what drives the foot roll through the Copy Rotation constraint — a forward-pointing target keeps that rotation intuitive, and the stub stays visible on/above the ground instead of poking through the floor.
- `foot.L`: head (heel) just above the ground, tail at the toe **on** the ground — the toe is the pivot when the foot rolls.

Foot rig (keeps the sole flat when the pelvis lowers):

1. `foot` bone: `use_inherit_rotation = False`.
2. `COPY_ROTATION` constraint on `foot`, subtarget `ik leg target.L`, **target_space and owner_space both `LOCAL`**.
3. Invert axes until rotating the target rolls the foot correctly — which axes varies with rig orientation; verify by rotating the target.

**Arms stay FK** (arm IK costs more than it saves on low-poly; IK/FK switching is workflow overhead). Fingers: one static hand bone covers 99%; only extrude finger/thumb chains (thumb `Ctrl+P` Keep Offset to hand) if the character must grip props.

### Skinning

- Organic bodies: mesh → Shift-select armature → `Ctrl+P` → Armature Deform **With Automatic Weights** (`bpy.ops.object.parent_set(type='ARMATURE_AUTO')` — verified headless). Usually good enough outright.
- **Detached eyes / gear**: auto weights can't handle them. Edit mode → `L` select linked → vertex groups **Remove from All** → set active group (`head`) → **Assign** (100% to one bone). Same recipe for backpacks (`spine`), beards, etc.
- **Detached hats/props as separate objects**: don't skin — **Child Of** object constraint → target armature, bone `head`, then **Set Inverse**.
- **Hip/bum bleed**: leg weights leaking into the hip/buttock bulge when the leg rotates → weight-paint the leg bone to 0 on those verts.
- Long clothes/skirts: loop cuts for deformation geometry, want *both* spine and leg weights; re-run automatic weights after mesh changes; paint while posed to fix poke-through. Philosophy: "make it just good enough" — minor penetration is invisible on low-poly + fast animation.

## Rigging — mechs / rigid hard-surface

**Never automatic weights on rigid parts** (plates shear when joints rotate). Use **With Empty Groups** then assign each rigid block 100% to its bone:

```python
bpy.ops.object.parent_set(type='ARMATURE')   # creates empty groups named after bones
# edit mode: L-select linked verts per block → vg = obj.vertex_groups['upper arm.L']; vg.add(vert_indices, 1.0, 'REPLACE')
```

Pre-split plates into separate linked blocks (duplicating faces creates them naturally). Penetration between moving plates is accepted, not weight-blended. Same armature pattern as humanoid (2 spines — one spine is too limiting), plus finger bones if it grips a weapon. Weapons/rocket pods = separate objects with **Child Of** (spine or hand bone) + Set Inverse, pivot modeled at the grip. Modeling notes: armor panels = `I` inset + `Alt+E` along normals; cockpit/eye lights = emissive cells; metallic cells from Attributes R.

## Rigging — quadrupeds / creatures

- **Minimal quadruped**: 6 bones total — `root` (body, horizontal), `head`, and one leg bone per leg (`front leg.L`, `back leg.L`, …). Legs deliberately **not** connected to the spine chain (Keep Offset parenting) — rigid assignment doesn't need a chain. Skin with Empty Groups + 100% assignment per body part (same as mech). Optional tail bone.
- **Fish/long creatures**: 4–5 connected spine bones along the body, **automatic weights** are fine (organic). Skip fin bones.
- Gotcha that bites every project: a few verts assigned to the wrong bone (e.g. head rotation moves the whole body) → select those verts → vertex groups Remove from All → re-assign correctly.

## Animation and export conventions

- First action = the rest pose, named `_TPose` (leading underscore sorts it to the top / becomes the FBX preview pose). Key all bones, Location+Rotation, frame 1.
- **Fake User (shield icon) on every action** or orphaned ones are purged on restart. `F` prefix in the list = protected, `O` = orphan.
- Auto-keying on; pose with G/R (`,` toggles Global/Local for rotations).
- Walk cycle: key poses Contact → Recoil → Passing → High every 3–4 frames; mirror-half via select-all `Ctrl+C` then **`Ctrl+Shift+V`** (paste mirrored — requires the roll recalibration above); loop end frame = N−1 (don't duplicate frame 1); make loops with `Shift+E` → Make Cyclic (F-Modifier).
- **No bone stretching** for engine export (Unity doesn't support it — characters go airborne).
- Export FBX with character facing +Y, origin at feet, actions individually chosen.

## Do / Don't

```text
Do: pin UVs to exact swatch centers (all loops of a face to one pixel)
Don't: leave UVs spanning cells — that's the "gradient across my face" bug

Do: Closest interpolation + Repeat extension on every palette texture node
Don't: use Linear filtering, JPEG palettes, compression, or mipmaps

Do: use a photo as the color source when real-world tones matter
Don't: stretch UV islands over busy photo regions unless you want noise

Do: mirror modifier with clipping on; apply late for asymmetry
Don't: let verts cross the mirror centerline with clipping off

Do: extrude limbs out of the main body mass — one continuous mesh per character
Don't: attach limbs as separate box shells (auto weights can't blend across the gap)

Do: loop cuts at knees/elbows/ankles even on ultra-low-poly rigs
Don't: add vertical loop cuts through the body or knife-cut deformables

Do: Empty Groups + 100% per-block weights for mechs and quadruped legs
Don't: use automatic weights on rigid plates (they shear)

Do: recalc bone roll (front view / GLOBAL_POS_Z) before IK and pose mirroring
Don't: forget use_deform=False on IK pole/target helper bones

Do: put the IK target's head on the ankle joint, stub pointing forward along the foot
Don't: dangle IK target/pole bones below the ground plane

Do: duplicate body faces to make clothes (weights come free)
Don't: expect auto weights to handle detached eyes/hats — assign to one bone
```

## Sources

- Blender manual — Image Texture node (interpolation/extension): https://docs.blender.org/manual/en/latest/render/shader_nodes/textures/image.html
- Blender manual — Principled BSDF inputs: https://docs.blender.org/manual/en/latest/render/shader_nodes/shader/principled.html
- Blender API — Armature Edit Bones: https://docs.blender.org/api/current/bpy.types.Armature.html#bpy.types.ArmatureEditBones
- Blender API — IK constraint (target/pole/chain_count/pole_angle): https://docs.blender.org/api/current/bpy.types.KinematicConstraint.html
- Blender API — Copy Rotation constraint (spaces/invert): https://docs.blender.org/api/current/bpy.types.CopyRotationConstraint.html
- Blender API — Child Of constraint (set_inverse): https://docs.blender.org/api/current/bpy.types.ChildOfConstraint.html
- Blender API — Vertex groups (assign/remove): https://docs.blender.org/api/current/bpy.types.VertexGroups.html
- Blender API — Armature symmetrize / calculate roll: https://docs.blender.org/api/current/bpy.ops.armature.html
- Technique note: this workflow circulates widely in the indie low-poly game-asset community; public-domain (CC0) palette texture packs with ready-made Blender materials are compatible with this recipe and drop in directly.
