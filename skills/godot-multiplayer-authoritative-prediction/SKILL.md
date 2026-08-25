---
name: godot-multiplayer-authoritative-prediction
description: Godot 4.x server-authoritative multiplayer with client-side prediction, reconciliation, and lag compensation for action games. Use only when the high-level MultiplayerAPI skill would suffer from visible input latency — otherwise prefer godot-multiplayer-high-level-api.
---

# Godot Multiplayer — Server-Authoritative With Prediction

Scope: server simulation of movement, client-side prediction + reconciliation, snapshot interpolation, and server-side lag compensation for fast-paced action games in Godot 4.x. This is the most complex networking skill here — reach for the simple ENet skill unless latency on the client is actually a gameplay problem.

## Architecture

Do:

- Clients send raw input/intent via unreliable RPC; the server simulates the fixed-step `_physics_process` on the `PhysicsServer`. `MultiplayerSynchronizer` replicates authoritative state back.
- Separate input replication from state replication. Input: `@rpc("any_peer", "call_remote", "unreliable")`. Picks/shots that must land: reliable.
- Keep a small render-lag buffer (snapshot interpolation). Applying latest server position immediately without interpolation causes rubber-banding.

```gdscript
# client -> server: broadcast inputs for the current frame
@rpc("any_peer", "call_remote", "unreliable")
func input_rpc(buffer: PackedInt64Array) -> void:
    # server: buffer[0] = input seq, buffer[1] = key mask
    _history.store_input(get_remote_sender_id(), buffer)
```

## Client-side prediction

- The client runs the same movement code locally so the player sees instant response, and records input sequence numbers.
- When a server state snapshot arrives:

Do:

- Reconcile the predicted position to the snapshot, then re-simulate ONLY the unacknowledged input tail (inputs since the snapshot's seq).

Don't:

- Re-simulate all past inputs from the beginning — that defeats prediction and doubles the cost. Track the last acknowledged server seq and replay from there.
- Apply server positions every frame without interpolation (rubber-banding).

## Lag compensation (hit registration)

- Do NOT trust client-reported hit positions. The client clicks where it SAW the target; by the time that reaches the server the target moved.
- Server rewinds interplated object histories to the sent time, then does the hit test. This is the classic "rewind to authoritative recorded position" technique.

## Determinism requirements

- Fixed timestep (physics engine) and a deterministic, seeded RNG on the server for gameplay logic.
- Never use per-frame-seeded `rand()`/`randf_range()` client-side for anything that affects state that should agree across peers. Predict and server results must be reproducible.
- Serialize nothing game-decision-critical through wall-clock time.

## Pitfalls quick list

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Rubber-banding | Applying server state same-frame | Interpolate through a snapshot buffer, render a few ticks behind |
| Desync after N seconds | Re-simulating full input history, or nondeterministic RNG/logic | Replay only unacknowledged tail; seed deterministically |
| Hit forward on player 2's screen, missed | Client-reported hit position | Server rewind to recorded history |
| Input arrives at server one tick late | mixed reliable input stream | Keep input RPC `unreliable`; reserve reliable for events |

## Sources

- https://docs.godotengine.org/en/stable/tutorials/networking/high_level_multiplayer.html
- https://www.gabrielgambetta.com/client-side-prediction-live-demo.html
- https://gafferongames.com/post/what_every_programmer_needs_to_know_about_game_networking/
- https://github.com/Peter229/GodotClientSidePrediction
- https://github.com/SummonSteve/Paced-Multiplayer
