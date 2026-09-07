---
name: godot-multiplayer-webrtc-websocket
description: Godot 4.x realtime networking for the web or NAT-traversal scenarios — WebSocket (TCP, message-based) vs WebRTC (P2P over ICE, needs signaling). Use when exporting to HTML5, when a backend/chat needs TCP, or when peers won't reach each other directly.
---

# Godot Multiplayer — WebSocket and WebRTC

Scope: choosing and wiring WebSocket (`WebSocketPeer`) and WebRTC (`WebRTCPeerConnection`/`WebRTCDataChannel`/`WebRTCMultiplayerPeer`) in Godot 4.x, including the web export and NAT-traversal pitfalls. Covers the transport, not the high-level gameplay RPC layer.

## Choose the right transport

Do (decision guide):

| Need | Transport |
| --- | --- |
| Chat, turn-based, backend REST-ish traffic | WebSocket (TCP, message-based) |
| Fast-paced realtime over the web or NAT traversal | WebRTC (UDP-over-ICE, P2P) |
| Native desktop game with plain LAN | ENet high-level API (see godot-multiplayer-high-level-api) |

Don't: use WebSocket for low-latency action gameplay — TCP head-of-line blocking and handshake latency make it the wrong tool when WebRTC or ENet is available.

## WebSocket

Do:

```gdscript
var ws := WebSocketPeer.new()
ws.connect_to_url("wss://host/socket")

func _process(delta: float) -> void:
    ws.poll()
    state = ws.get_ready_state()
    if state == WebSocketPeer.STATE_OPEN:
        while ws.get_available_packet_count() > 0:
            var pkt: Packet = ws.get_packet()
```

- Use `wss://` (TLS) unless you control transport security end to end. `ws://` is plaintext.
- `poll()` must be called every frame; the peer does nothing on its own.
- For the high-level API with a WebSocket client/server, use `WebSocketMultiplayerPeer` instead of a raw `WebSocketPeer`.

## WebRTC

WebRTC is P2P over ICE/DTLS/SDP. Two peers cannot connect without a signaling server that exchanges SDP offers/answers and ICE candidates before the direct connection forms.

Do:

```gdscript
var pc := WebRTCPeerConnection.new()
pc.initialize()
pc.session_description_created.connect(func(type, sdp):
    signaling_send(type, sdp)                  # to the other peer, via your server
)
# on remote candidate: pc.add_ice_candidate(...)
var channel := pc.create_data_channel("game") # or on_negotiation_needed
```

Don't:

- Assume WebRTC works out of the box on native. It's built into the HTML5 export, but native builds need the `godotengine/webrtc-native` GDExtension plugin.
- Skip the signaling server. Direct SDP exchange requires SOMETHING brokering the first handshake.
- Pick WebRTC for a game that's always LAN — it adds ICE/signaling for no benefit.

## Pitfalls quick list

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Mobile: no network at all | `INTERNET` permission missing in Android export preset | Enable `INTERNET` permission in the export preset |
| WebSocket never connects | Forgot `poll()` per frame, or used `ws://` against a `wss://` host | Call `poll()` every frame; match scheme |
| WebRTC connection hangs | No signaling server / wrong address | Implement SDP + ICE exchange broker first |
| Native build fails WebRTC | `webrtc-native` GDExtension not bundled | Add `godotengine/webrtc-native` to the native build |

## Sources

- https://docs.godotengine.org/en/stable/tutorials/networking/websocket.html
- https://docs.godotengine.org/en/stable/tutorials/networking/webrtc.html
