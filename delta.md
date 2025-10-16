> **⚠️ WARNING! HERE BE DRAGONS! ⚠️**
> 
> This is a provisional and incomplete draft of the CloudLink 5.1 "Delta" protocol. The features, commands, and schemas described are for discussion and are subject to change.
>
> Last revision: October 15, 2025 @ 6:02 PM EDT.

# CloudLink 5.1 ("Delta") Protocol

CL5.1 "Delta" is a revision of the CloudLink 5 protocol that transitions to a **fully decentralized, pure peer-to-peer network** as its core. It removes the reliance on a mandatory central session server.

To enhance this core, the protocol introduces two optional, P2P-based server types:

  * A **Discovery Server** can be used to help peers find each other and negotiate lobbies using a `user@designation` system.
  * A **Bridge Server** can be used to translate communications between CL5.1 clients and clients running legacy CL2/3/4 protocols.

Key concepts include decentralized lobby controls, host migration, and a mesh-like message bridging system.

## Protocol Schema

All communication is considered "in-band" between peers. The packet schema is a JSON object with optional keys to support advanced routing.

```js
{
  "opcode": "...",
  "payload": { ... },
  "origin": "...",
  "target": "...",
  "ttl": 3,
  "id": "...",
  "method": "..."
}
```

  * `opcode`: A string that defines the packet's purpose.
  * `payload`: Any arbitrary JSON-serializable data.
  * `origin`: **(Required if using a bridge)** The unique instance ID of the original peer that sent the message.
  * `target`: **(Required if using a bridge)** The unique instance ID of the final recipient. This can be a specific peer ID or a broadcast indicator (e.g., `*`).
  * `ttl`: **(Optional)** Time-To-Live. A number used for bridged/relayed messages. Each peer that forwards the message decrements this value. If `ttl` reaches 0, the packet is dropped.
  * `id` & `method`: Optional properties for variable/list synchronization.

-----

### Peer-to-Peer Commands

These commands are sent directly between connected peers.

| `opcode` | Description |
| :--- | :--- |
| **`NEGOTIATE`** | Sent by a client immediately after connecting to another peer. The `payload` contains an object with arbitrary key-value pairs detailing the client's capabilities (e.g., supported features, loaded plugins) for feature negotiation. |
| **`HOST_LOBBY`** | Sent from a peer to its connections to announce it is the host of a lobby. The `payload` contains the lobby's state (name, password status, max players, etc.). |
| **`REQUEST_JOIN`**| Sent from a new peer to any existing peer in a lobby to request entry. The receiving peer will forward this request to the current host. |
| **`MANAGE_LOBBY`**| A dual-purpose command. Sent from the host to all peers to announce a change in the lobby's state (e.g., `{"method": "lock"}`). Also sent from a peer to the host to request an action (e.g., `{"method": "kick", ...}`). |
| **`TRANSFER_HOST`** | Sent from the current host to another peer (`target`) to designate them as the new host. The recipient then sends a `HOST_LOBBY` broadcast. |
| **`LOBBY_STATE`** | Sent by the host to a new peer in response to `REQUEST_JOIN`. The `payload` contains the complete current state of the lobby, granting entry. |
| **`PEER_JOIN`** | Broadcast by the host to all peers when a new user has successfully joined. The `payload` is the new peer's user object. |
| **`PEER_LEFT`** | Broadcast by the host when a peer disconnects. The `payload` contains the peer's ID. |
| **`G_MSG`** | A Global Message. The sending peer sets `target` to `*` and sends it to its direct connections. Peers forward it based on the `ttl`. |
| **`P_MSG`** | A Private Message. The `target` is a specific peer ID. Peers will attempt to bridge the message if not directly connected. |
| `G_VAR`/`P_VAR` | Variable Sync message. Works like `G_MSG`/`P_MSG`. |
| `G_LIST`/`P_LIST`| List Sync message. Works like `G_MSG`/`P_MSG`. |
| `NEW_CHAN` | A control message sent to a `target` peer to negotiate a new, named data channel. |
| `HANGUP`/`DECLINE`| Voice call control signals sent to a specific `target` peer. |

-----

### Discovery Server Commands (Optional)

These commands are used to interact with an optional Discovery Server for matchmaking and peer resolution.

| `opcode` | Description |
| :--- | :--- |
| **`REGISTER_LOBBY`** | Sent by a host peer to the Discovery Server to make its lobby publicly listed. |
| **`UNREGISTER_LOBBY`**| Sent by a host peer to remove its lobby listing from the Discovery Server. |
| **`LIST_LOBBIES`** | Sent by any peer to request a list of all public lobbies. |
| **`QUERY_PEER`** | Sent by any peer to resolve a `user@designation` address into a PeerJS connection ID. |

-----

### Relay Server Commands (Optional)

These commands are used to bridge communication with clients on older CL2/3/4 protocols.

| `opcode` | Description |
| :--- | :--- |
| **`RELAY_MSG`** | Sent from a CL5.1 client to the Relay Server. The `payload` contains the original CL5.1 packet and a `legacy_target` field specifying the older client. The Relay translates and forwards the message. |
| **`BRIDGE_MSG`** | Sent from the Relay Server to a CL5.1 client when a message arrives from a legacy client. The `payload` contains the translated message and information about the legacy sender. |
