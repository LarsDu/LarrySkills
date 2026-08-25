---
name: godot-multiplayer-high-level-api
description: Godot 4.x multiplayer via the high-level MultiplayerAPI — ENet, MultiplayerSynchronizer/MultiplayerSpawner, @rpc annotations, authority, and secure client-server design. Use this instead of the authoritative-prediction skill for games that don't need client-side prediction.
---

# Godot Multiplayer — High-Level MultiplayerAPI (ENet)

Scope: ENet peer setup, `@rpc`, authority, spawner/synchronizer, and don't-trust-the-client design for simple client-server and "host" games in Godot 4.x. Does NOT cover client-side prediction/reconciliation (see `godot-multiplayer-authoritative-prediction`) or WebSocket/WebRTC (see `godot-multiplayer-webrtc-websocket`).

## Peer setup

Do:

```gdscript
# Server (peer id is always 1)
var peer := ENetMultiplayerPeer.new()
peer.create_server(port, MAX_PLAYERS)
multiplayer.multiplayer_peer = peer

# Client
var peer := ENetMultiplayerPeer.new()
peer.create_client(host_ip, port)
multiplayer.multiplayer_peer = peer
```

Don't:

- Forget to null out the peer on disconnect; error messages misattribute. Set `multiplayer.multiplayer_peer = OfflineMultiplayerPeer.new()`.
- Assume ENet handles NAT traversal. ENet is UDP — Internet hosts need the UDP port forwarded; bare ENet won't punch through NAT with anyone else.

## `@rpc` anatomy

`@rpc(mode, sync, transfer, channel)` — the two flags people mix up:

- mode: `"authority"` (only the node's authority can call it) vs `"any_peer"`
- sync: `"call_remote"` (don't run locally) vs `"call_local"` (also run on the caller)
- transfer: `"reliable"` | `"unreliable"` | `"unreliable_ordered"`
- channel: last arg, 0-127

Do — the intent-then-validate pattern:

```gdscript
func _unhandled_input(e: InputEvent) -> void:
    if e.is_action_pressed("fire") and is_multiplayer_authority():
        request_fire.rpc_id(1)                 # send intent to server only

@rpc("any_peer", "call_local", "reliable")
func request_fire() -> void:
    var sender := multiplayer.get_remote_sender_id()
    if not _can_fire(sender): return           # server validates
    spawn_projectile.rpc(sender)

@rpc("authority", "call_local", "reliable")
func spawn_projectile(owner_id: int) -> void:
    _do_spawn(owner_id)
```

Don't:

- Trust client-set positions/hp/results. Clients send INTENT; the server validates, then broadcasts authoritative state. Add rate limits on input RPCs.
- Let reliable chat/log traffic share channel 0 with gameplay. Put chat/telemetry on a separate channel so a slow reliable stream doesn't block gameplay bytes. Prefer `unreliable_ordered` + channels for homogeneous streams.

## Authority

- Server is the default authority for a node; peers check `is_multiplayer_authority()`.
- `set_multiplayer_authority(id)` reassigns. Typical: server sets authority to the player's peer id for that player's own character node.
- RPCs must live on `Node`-derived classes only. `Resource`-based RPCs silently fail.
- RPC arguments do not serialize arbitrary `Object` or `Callable`.

## The checksum trap

Every `@rpc` method must exist with IDENTICAL declaration on both client and server — including methods a side never calls. Godot hashes all RPCs in a script; a mismatch breaks the handshake and the error message misattributes the offending method.

## Spawner / Synchronizer

Do:

- Use `MultiplayerSpawner` to instantiate per-connection scenes so NodePaths match exactly across peers (`add_child(node, true)` for deterministic names).
- Use `MultiplayerSynchronizer` to replicate properties each network tick (Replication dock: per-property `Spawn`/`Sync` flags, `replication_interval`, `set_visibility_for()` for scoping).

Don't:

- Build nodes imperatively and hope NodePaths line up. Deterministic spawn paths are required or RPC targets resolve differently per peer.

## Pitfalls quick list

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| RPC never fires / disconnect on handshake | Checksum mismatch — a script differs between peers | Compare script versions; keep every script's RPC list identical |
| Node refused / spawn mismatch | NodePath divergent across peers | Spawn via MultiplayerSpawner, deterministic order |
| Authority gate rejects clients | `is_multiplayer_authority()` false on who you expected | Confirm `set_multiplayer_authority()` was applied on the server |
| Mobile: no network at all | Android `INTERNET` permission missing in export preset | Enable `INTERNET` in the Android export preset |

## Sources

- https://docs.godotengine.org/en/stable/tutorials/networking/high_level_multiplayer.html
- https://docs.godotengine.org/en/stable/tutorials/networking/websocket.html
- https://docs.godotengine.org/en/stable/tutorials/networking/webrtc.html
