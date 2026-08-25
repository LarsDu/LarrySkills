---
name: godot-shaders
description: Godot 4.x Shading Language — shader types, screen_texture gotchas, and six validated effect recipes (toon/cel, outline, dissolve, water, glitch, hologram). Consult before writing fragment/vertex effects or screen-space distortions.
---

# Godot Shaders

Scope: Godot Shading Language (GLSL ES 3.0 subset) for Godot 4.x `canvas_item`, `spatial`, and `particles` shaders, plus six effect recipes. Does not cover sky/fog shaders or GDScript-side material plumbing.

## Basics

- Six shader types: `canvas_item` (2D), `spatial` (3D), `particles`, `sky`, `fog`, `texture_blit`. Built-ins differ per type.
- Stages: `vertex()`, `fragment()`, `light()`. Renderer order is vertex -> fragment, then light per relevant light.
- Shader deps: do not define functions inside `fragment()`/`vertex()`.

## The Godot 4 `SCREEN_TEXTURE` trap

Godot 4 REMOVED the built-in `SCREEN_TEXTURE`. Do:

```glsl
uniform sampler2D screen_texture : hint_screen_texture, filter_linear_mipmap;
vec4 c = textureLod(screen_texture, SCREEN_UV, 0.0);
```

Don't:

- Reference `SCREEN_TEXTURE` in a 4.x shader — compile error.
- Overlap two screen-texture shaders without a `BackBufferCopy` node between them. In 3D, the screen texture is captured after the opaque pass and BEFORE transparent objects — transparents are NOT captured, and any material reading it is treated as transparent itself.

## Recipes

### 1. Toon / cel (3D)

Prefer built-in render modes when possible: `render_mode diffuse_toon, specular_toon;` or `diffuse_lambert_wrap, specular_disabled`. For custom banding, write a `light()`:

```glsl
shader_type spatial;
void light() {
    float ndotl = max(0.0, dot(NORMAL, LIGHT_DIRECTION));
    ndotl = floor(ndotl * 3.0) / 3.0;             // quantize to 3 bands
    DIFFUSE_LIGHT += ndotl * ATTENUATION * LIGHT_COLOR;
    SPECULAR_LIGHT += LIGHT_AREA_SPECULAR_MULTIPLIER * ATTENUATION * LIGHT_COLOR * SPECULAR_AMOUNT;
}
```

### 2. Outline

3D inverted hull — a second material pass with front faces culled, vertices pushed along the normal:

```glsl
shader_type spatial;
render_mode cull_front;
uniform float width = 0.05;
void vertex() { VERTEX += NORMAL * width; }
void fragment() { ALBEDO = vec3(0.0); ALPHA = 1.0; }
```

2D alpha-ring outline — sample `TEXTURE` alpha at ring offsets around each pixel:

```glsl
shader_type canvas_item;
uniform vec4 outline_color : source_color = vec4(0.0, 0.0, 0.0, 1.0);
uniform float glow_size = 2.0;
void fragment() {
    vec2 ps = 1.0 / TEXTURE_PIXEL_SIZE.xy * glow_size * TEXTURE_PIXEL_SIZE;
    float a = 0.0;
    a += texture(TEXTURE, UV + ps * vec2(-1, -1)).a;
    a += texture(TEXTURE, UV + ps * vec2(1, -1)).a;
    a += texture(TEXTURE, UV + ps * vec2(-1, 1)).a;
    a += texture(TEXTURE, UV + ps * vec2(1, 1)).a;
    float outline_mask = step(0.1, a) * (1.0 - texture(TEXTURE, UV).a);
    COLOR = mix(texture(TEXTURE, UV), outline_color, outline_mask);
}
```

### 3. Dissolve (2D)

```glsl
shader_type canvas_item;
uniform sampler2D noise_tex : filter_nearest;
uniform float dissolve_amount : hint_range(0.0, 1.0) = 0.5;
uniform float edge_thickness = 0.1;
uniform vec4 edge_color : source_color = vec4(1.0, 0.3, 0.1, 1.0);
void fragment() {
    float n = texture(noise_tex, UV * 8.0).r;
    float lo = dissolve_amount * (1.01 + edge_thickness) - edge_thickness;
    float hi = lo + edge_thickness;
    float edge = step(lo, n) - step(hi, n);
    vec4 tex = texture(TEXTURE, UV);
    tex.a *= step(hi, n);
    COLOR = mix(tex, edge * edge_color, edge);
}
```

### 4. Water / vertex displacement with screen distortion

```glsl
shader_type spatial;
uniform sampler2D noise_tex;
uniform sampler2D screen_texture : hint_screen_texture, filter_linear_mipmap;
uniform float distortion_strength = 0.02;
void vertex() {
    VERTEX.y += sin(VERTEX.x * 4.0 + TIME * 2.0) * 0.05;   // waves
}
void fragment() {
    vec2 nuv = UV + vec2(TIME * 0.05, TIME * 0.03);
    vec2 distortion = (texture(noise_tex, nuv).rg - 0.5) * distortion_strength;
    vec3 col = textureLod(screen_texture, SCREEN_UV + distortion, 0.0).rgb;
    ALBEDO = col;
}
```

### 5. Screen-space glitch / heat distortion

```glsl
shader_type canvas_item;
uniform sampler2D noise_tex;
uniform float distortion_amount = 0.02;
void fragment() {
    vec2 nuv = UV + vec2(0.0, TIME * 0.2);
    vec2 distortion = (texture(noise_tex, nuv).rg - 0.5) * distortion_amount;
    COLOR = texture(TEXTURE, UV + distortion);
}
```

Glitch variant: slice UV rows via `floor(UV.y * rows)` + time-randomized offset, so whole scanlines jump.

### 6. Hologram / energy barrier

```glsl
shader_type canvas_item;
uniform sampler2D screen_texture : hint_screen_texture, filter_linear_mipmap;
uniform vec4 barrier_color : source_color = vec4(0.2, 0.7, 1.0, 0.8);
uniform float pulse_speed = 2.0;
uniform float cell = 8.0;
void fragment() {
    vec2 uv = UV * cell;
    vec2 gv = fract(uv) - 0.5;
    float hex_dist = length(gv);
    float pulse = sin(TIME * pulse_speed) * 0.3 + 0.7;
    vec4 tex = texture(TEXTURE, UV);
    // hex-grid mask + edge brightening via screen-space rim
    float hex_mask = smoothstep(0.2, 0.0, abs(hex_dist - 0.5)) * 0.3;
    vec3 col = mix(tex.rgb, barrier_color.rgb, pulse * 0.6 + hex_mask);
    COLOR = vec4(col, barrier_color.a);
}
```

For a classic hologram: `spatial`, `unshaded`, blue-tinted alpha base, scanned-line bands via `smoothstep` of `frac(UV.y * lines - TIME)`, additive rim.

## Particles

- `shader_type particles` runs on `GPUParticles2D/3D` only — `CPUParticles*` CANNOT run particle shaders.
- Two stages: `start()` and `process()`; mutate `VELOCITY`, `CUSTOM`, `TRANSFORM`, `LIFETIME`, read `TIME`, `EMITTER_VELOCITY`; fork with `emit_subparticle()` + `FLAG_*`.

## Sources

- https://docs.godotengine.org/en/stable/tutorials/shaders/shader_reference/shading_language.html
- https://docs.godotengine.org/en/stable/tutorials/shaders/shader_reference/spatial_shader.html
- https://docs.godotengine.org/en/stable/tutorials/shaders/screen-reading_shaders.html
- https://docs.godotengine.org/en/stable/tutorials/shaders/shader_reference/particle_shader.html
- https://github.com/haowg/GODOT-VFX-LIBRARY
- https://github.com/gamedevserj/Godot-Shaders
