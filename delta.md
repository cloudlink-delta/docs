# CloudLink 5.1 ("Delta") Protocol

CL5.1 "Delta" is a revision of the CloudLink 5 protocol that transitions to a **WebRTC-based peer-to-peer network** as its core data layer. 

While pure P2P data transfer is the primary goal, the protocol still requires a mandatory **Signaling Server** (e.g., a standard PeerJS server) to exchange initial connection parameters (ICE candidates, SDP offers/answers). Once connected, peers communicate directly.

To enhance this core, the protocol introduces optional, specialized server/client roles:
* A **Discovery Server** to help peers find each other, negotiate lobbies, and resolve `username@designation` identities (e.g. `discovery@SOMEWHERE-EARTH`).
* A **Bridge Server** (Reserved for future use) to translate communications between CL5.1 clients and clients running legacy CL2/3/4 protocols.
* A **Relay Server** (Reserved for future use) to optimize mesh networking and broadcasts.

## Protocol Schema

All communication is formatted as "in-band" JSON payloads sent across WebRTC DataChannels.

```js
{
  "opcode": "...",
  "payload": { ... },
  "origin": "...",
  "target": "...",
  "ttl": 1,
  "id": "...",
  "method": "...",
  "listener": "..."
}
```

* `opcode`: **(Required)** A string that defines the packet's purpose.
* `payload`: Any arbitrary JSON-serializable data.
* `origin`: **(Required for bridge/relay/broadcasts)** The unique instance ID of the original peer that sent the message.
* `target`: **(Required for bridge/relay/broadcasts)** The unique instance ID of the final recipient. This can be a specific peer ID or a broadcast indicator (`*`).
* `ttl`: **(Required)** Time-To-Live. Included for forward-compatibility with multi-hop mesh routing. Currently, reference clients require `ttl >= 0` to accept a packet but execute 1-hop direct broadcasts without decrementing. The default value is `1`.
* `id` & `method`: Optional properties for variable/list synchronization and RPC handling.
* `listener`: Optional property to tag packets and await responses (used primarily in Mesh/RPC commands).

---

## Peer-to-Peer Commands

These commands are sent directly between connected peers.

| `opcode` | Description |
| :--- | :--- |
| **`NEGOTIATE`** | Sent immediately upon opening the default channel. Contains an object detailing the client's capabilities (version, plugins, roles). |
| **`G_MSG`** / **`P_MSG`** | Global/Private Message. `G_MSG` targets `*` (broadcast to all direct connections), while `P_MSG` targets a specific peer ID. |
| **`G_VAR`** / **`P_VAR`** | Variable Sync message. Synchronizes state across peers. |
| **`G_LIST`** / **`P_LIST`** | List Sync message. Synchronizes array states across peers using specific mutation methods. |
| **`G_MESH`** / **`P_MESH`** | RPC (Remote Procedure Call) broadcasts/unicasts. If the `method` is `req`, recipients must reply with an `ack`. |
| **`NEW_CHAN`** | Control message sent to negotiate a new, named WebRTC data channel. |
| **`PING`** / **`PONG`** | Control messages used for computing Round-Trip Time (RTT) and clock offset synchronization. |
| **`CALL`** / **`ANSWER`** / **`HANGUP`** / **`DECLINE`** | Voice call signaling used to orchestrate WebRTC MediaStreams. |
| **`WARNING`** / **`VIOLATION`** | State control messages. `WARNING`s are user-correctable. `VIOLATION`s trigger an immediate, mandatory client-side disconnect sequence. |
| **`HELLO`** | Advertises a peer's preferred name and designation. Used by the Discovery plugin to populate local identity caches. |

### Packet-Specific Syntax (P2P)

#### **`G_LIST` / `P_LIST`**
Used to synchronize global/local lists. `payload` depends on the `method`. The `id` must be a unique string (network tag).
```js
{
  "opcode": "G_LIST" | "P_LIST",
  "payload": ...,
  "method": "reset" | "set" | "length" | "replace",
  "id": "my_network_tag",
  "ttl": 1
}
```

#### **`G_MESH` / `P_MESH`**
Used for Remote Procedure Calls. May only be sent on the `default` data channel.
```js
{
  "opcode": "G_MESH" | "P_MESH",
  "payload": "eventName",
  "method": "" | "req",
  "listener": "uuid-string-here", // Required if method is 'req'
  "ttl": 1
}
```
Recipients of a `req` method must execute the RPC and reply with:
```js
{
  "opcode": "G_MESH" | "P_MESH",
  "payload": "eventName",
  "method": "ack",
  "listener": "uuid-string-here",
  "ttl": 1
}
```

---

## Discovery Server Commands

Discovery features are optional. When enabled, clients connect to a designated server (e.g., `discovery@SOMEWHERE-EARTH`) to resolve user identities and negotiate matchmaking lobbies. 

### Client -> Server Requests

| `opcode` | Description |
| :--- | :--- |
| **`REGISTER`** | Registers the client's preferred username with the server. |
| **`QUERY`** | Requests the `username@designation` mapping for a specific instance ID. |
| **`LOBBY_LIST`** | Requests an array of all active, visible, unlocked lobbies. |
| **`LOBBY_INFO`** | Requests detailed metadata about a specific lobby ID. |
| **`CONFIG_HOST`** | Requests the creation of a new lobby. |
| **`CONFIG_PEER`** | Requests to join an existing lobby. |
| **`LOCK`** / **`UNLOCK`** | (Host) Toggles if new members can join. |
| **`HIDE`** / **`SHOW`** | (Host) Toggles visibility in the public `LOBBY_LIST`. |
| **`SIZE`** | (Host) Updates max capacity (`-1` for unlimited). |
| **`PASSWORD`** | (Host) Sets or updates the lobby password. |
| **`KICK`** | (Host) Forcefully removes a peer from the lobby. |
| **`TRANSFER`** | (Host) Transfers host privileges to another peer. |
| **`CLOSE`** | (Host) Shuts down the lobby, disconnecting all members. |
| **`LEAVE`** | (Peer) Gracefully exits the current lobby. |

### Server -> Client Events & Acknowledgments

| `opcode` | Description |
| :--- | :--- |
| **`DISCOVER`** | Announces the presence of another server that needs to be connected by the client. Currently, it only announces bridge servers. |
| **`REGISTER_ACK`** | Confirms successful username registration. |
| **`QUERY_ACK`** | Returns the identity mapping for a requested peer. |
| **`CONFIG_HOST_ACK`** | Confirms successful lobby creation. |
| **`CONFIG_PEER_ACK`** | Confirms successful lobby join. |
| **`LOBBY_EXISTS`** | Error: Attempted to host a lobby ID that is already taken. |
| **`LOBBY_NOTFOUND`** | Error: Attempted to join or query a lobby that doesn't exist. |
| **`PASSWORD_ACK`** | Confirms a lobby password change was successful. |
| **`PASSWORD_REQUIRED`** | Error: Attempted to join a lobby without providing a required password. |
| **`PASSWORD_FAIL`** | Error: Provided an incorrect lobby password. |
| **`CONFIG_REQUIRED`** | Error: Attempted to perform a lobby action without being in a lobby. |
| **`NEW_LOBBY`** | Broadcasted to all peers when a new (public) lobby is created. |
| **`PEER_JOIN`** | Sent to lobby members when a new peer joins, prompting P2P connection attempts. |
| **`PEER_LEFT`** | Sent to lobby members when a peer leaves. |
| **`KICK_ACK`** | Sent to the Host to confirm a kick was successful/failed. |
| **`KICKED`** | Sent to a peer to notify them they have been forcefully removed. |
| **`TRANSITION`** | Informs a peer their role has changed (e.g., promoted to `host` or demoted to `peer`). |
| **`CLOSE_ACK`** / **`LOBBY_CLOSED`** | Confirms the lobby was shut down. |
| **`LEAVE_ACK`** | Confirms the client successfully exited the lobby. |
| **`AUTO_REGISTER`** | Confirms successful automatic username registration. Only implemented for bridge servers at this time. |

---

## Future Implementations

The following server types are reserved in the `NEGOTIATE` payload (`is_bridge`, `is_relay`) but are currently undocumented and lack implementation in the reference client architecture.

### Bridge Server Commands
Reserved for translating communication between CL5.1 clients and legacy CL2/3/4 protocols over WebSockets.

### Relay Server Commands
Reserved for True Mesh capabilities, offloading bandwidth-heavy broadcasts from client connections to dedicated relay nodes utilizing `ttl` decrements.
