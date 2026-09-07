# LarrySkills

Reusable Claude Code agents and skills. Skills are atomic, few-shot Do/Don't references (not tutorials) meant to correct or supplement what a model already knows, validated against current official documentation and community sources rather than written from memory. No emojis.

Structured as a Claude Code plugin: `agents/` and `skills/` are auto-discovered.

## Agents

- `agents/aiml-engineer-agent.md` — ML/AI engineering: PyTorch, torchvision,
  HuggingFace, Ray, Lightning, and test-writing for ML code.
- `agents/godot-gamedev-agent.md` — Godot 4.x game development: navigation AI,
  multiplayer networking, GDScript/C#, shaders.
- `agents/blender-agent.md` — 3D modeling and procedural mesh generation in
  Blender via the Python API.

Each agent's system prompt lists the skills it owns and when to consult each
one via the Skill tool, and every agent ends its skill list with
`programming-best-practices`.

## Skills

Shared:
- `programming-best-practices` — non-dogmatic SOLID/DRY/YAGNI judgment lens,
  aimed specifically at the premature-abstraction failure mode common in
  LLM-generated code. Used by all three agents.

AI/ML (`aiml-engineer-agent`):
- `python` — general pitfalls not covered by library-specific skills
  (mutable defaults, late-binding closures, circular imports, packaging,
  asyncio mistakes).
- `pytorch`, `torchvision`, `huggingface`, `ray`, `lightning` — current-version
  Do/Don't pairs for each library.
- `python-test-review-write` — writing, reviewing, and critiquing pytest
  suites, including a checklist for AI-generated tests.

Godot (`godot-gamedev-agent`):
- `godot-ai-navmesh` — NavMesh baking and NavigationAgent pathing/avoidance.
- `godot-multiplayer-high-level-api` — ENet/RPC/MultiplayerSynchronizer.
- `godot-multiplayer-authoritative-prediction` — server authority, client
  prediction/reconciliation, lag compensation.
- `godot-multiplayer-webrtc-websocket` — web export and NAT-traversal
  networking.
- `godot-csharp-migration` — GDScript-to-C# API differences.
- `godot-shaders` — Godot Shading Language plus six effect recipes.

Blender (`blender-agent`):
- `blender-mesh-modeling` — bpy/bmesh procedural mesh construction pitfalls.

## Layout

```
LarrySkills/
  .claude-plugin/plugin.json
  agents/*.md
  skills/<skill-name>/SKILL.md
```
