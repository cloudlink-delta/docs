> **⚠️ WARNING! HERE BE DRAGONS! ⚠️**
> 
> This is a provisional and incomplete draft of the CloudLink 5.1 "Delta" protocol. The features, commands, and schemas described are for discussion and are subject to change.
>
> Last revision: October 26, 2025 @ 12:41 PM EDT.

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
  "ttl": 1,
  "id": "...",
  "method": "..."
}
```

  * `opcode`: **(Required)** A string that defines the packet's purpose.
  * `payload`: Any arbitrary JSON-serializable data.
  * `origin`: **(Required for bridge/relay/broadacasts)** The unique instance ID of the original peer that sent the message.
  * `target`: **(Required for bridge/relay/broadcasts)** The unique instance ID of the final recipient. This can be a specific peer ID or a broadcast indicator (e.g., `*`).
  * `ttl`: **(Required)** Time-To-Live. A number used for bridged/relayed messages. Each peer decrements this value before interpreting the packet. If `ttl` is below 0, the packet will be dropped. The default value is 1.
  * `id` & `method`: Optional properties for variable/list synchronization.

-----

### Peer-to-Peer Commands

These commands are sent directly between connected peers.

| `opcode` | Description |
| :--- | :--- |
| **`NEGOTIATE`** | Sent by a client immediately after connecting to another peer. The `payload` contains an object with arbitrary key-value pairs detailing the client's capabilities (e.g., supported features, loaded plugins) for feature negotiation. |
| **`G_MSG`** | A Global Message. The sending peer sets `target` to `*` and sends it to its direct connections. Peers forward it based on the `ttl`. |
| **`P_MSG`** | A Private Message. The `target` is a specific peer ID. Peers will attempt to bridge the message if not directly connected. |
| `G_VAR`/`P_VAR` | Variable Sync message. Works like `G_MSG`/`P_MSG`. |
| `G_LIST`/`P_LIST`| List Sync message. Works like `G_MSG`/`P_MSG`. |
| `NEW_CHAN` | A control message sent to a `target` peer to negotiate a new, named data channel. |
| `PING`/`PONG` | Control messages used for computing RTT. |
| `HANGUP`/`DECLINE`| Voice call control signals sent to a specific `target` peer. |
| `WARNING`/`VIOLATION`| State control messages. `WARNINGS` are safe and user-correctable. `VIOLATION`s will result in mandatory disconnects. |

#### Packet-specific syntax

##### **`NEGOTIATE`**
**Negotiation is mandatory** for all variants of the Delta protocol.
```js
{
  "opcode": "NEGOTIATE",
  "payload": {
    "version": {       // "version" is not intended for protocol versioning. It is purely for user-reference.
      "type": string,  // Can be any value you want. Examples: Scratch, Go...
      "major": int,
      "minor": int,
      "patch": int,
    },
    "spec_version": 0, // Spec version is recommended for protocol versioning. Leave as 0.
    "plugins": [       // i.e. "chat", "sync", "discovery", ...
      string...
    ],
    "is_bridge": bool,
    "is_relay": bool,
    "is_discovery": bool,
  }
}
```

##### **`PING`/`PONG`**
Request messages start with `PING` and a `t1` value, containing a timestamp encoded as a numerical timestamp.

```js
{
  "opcode": "PING",
  "payload": {
    t1: Date.now(),
  },
  "ttl": 1,
}
```

A reply message to `PING` should contain the original `t1` timestamp as well as a new numerical timestamp in `t2`.

```js
{
  "opcode": "PONG",
  "payload": {
    t1: (PING payload).t1;
    t2: Date.now();
  },
  "ttl": 1,
}
```

-----

### Discovery Server Commands

These commands are used to interact with an optional Discovery Server for matchmaking and peer resolution.

| `opcode` | Description |
| :--- | :--- |
| **`LOBBY_INFO`** | Requests specific details about a specified lobby ID. |
| **`CONFIG_HOST`** | Creates a new lobby with a unique ID and settings. |
| **`CONFIG_PEER`** | Joins a lobby with the specified ID and settings. |
| **`LOBBY_LIST`** | Returns a list of all active, visible, password-free, and unlocked lobbies. |
| `LOCK`/`UNLOCK` | Toggles the lock state of the lobby to prevent/allow new members from connecting. |
| `SIZE`| Updates the maximum number of members allowed in the lobby. Set to -1 to allow unlimited members. |
| `KICK` | Forcefully disconnects a member from the lobby. |
| `HIDE`/`SHOW` | Toggles lobby visibility in the public lobby list, or through event broadcasts. |

#### Packet-specific syntax

##### **`CONFIG_HOST`**
Requesting a lobby will use the following packet syntax:

```js
{
  "opcode": "CONFIG_HOST",
  "payload": {
    "lobby_id": any, // must be serializable
    "password": string,
    "max_peers": int, // -1 or null = "unlimited", otherwise > 0
    "locked": bool,
    "hidden": bool,
    "metadata": any,
  },
  "ttl": 1,
}
```

If the specified ID exists:

```js
{
  "opcode": "LOBBY_EXISTS",
  "ttl": 1,
}
```

A successful reply will return the following:

```js
{
  "opcode": "CONFIG_HOST_ACK",
  "payload": any, // lobby_id
  "ttl": 1,
}
```

Upon creation (if not `hidden`), the following will be broadcasted:

```js
{
  "opcode": "NEW_LOBBY",
  "payload": any, // lobby_id
  "ttl": 1,
}
```

##### **`CONFIG_PEER`**
Joining a lobby will use the following packet syntax:

```js
{
  "opcode": "CONFIG_PEER",
  "payload": {
    "lobby_id": any, // must be serializable
    "password": string,
  },
  "ttl": 1,
}
```

If the specified ID does not exist:

```js
{
  "opcode": "LOBBY_NOTFOUND",
  "ttl": 1,
}
```

A successful reply will return the following:

```js
{
  "opcode": "CONFIG_PEER_ACK",
  "payload": string, // lobby_id
  "ttl": 1,
}
```

All members of the lobby (including the host) will be
sent the following:

```js
{
  "opcode": "PEER_JOIN",
  "payload": string, // instance ID
  "ttl": 1,
}
```

Some lobbies may have a password set. If the lobby
has one, three possible outcomes can occur.

If the lobby has a password and the value of `password`
is empty:

```js
{
  "opcode": "PASSWORD_REQUIRED",
  "ttl": 1,
}
```

If the `password` value is invalid:

```js
{
  "opcode": "PASSWORD_FAIL",
  "ttl": 1,
}
```

A valid `password` will return this before `CONFIG_PEER_ACK`:

```js
{
  "opcode": "PASSWORD_ACK",
  "ttl": 1,
}
```

##### **`LOBBY_LIST`**
Requesting the list does not take any arguments. 

```js
{
  "opcode": "LOBBY_LIST",
  "ttl": 1,
}
```

The response will return a list of lobbies. This will also be sent to the client upon initial connection to the discovery server.

```js
{
  "opcode": "LOBBY_LIST",
  "payload": [any...], // values will be serialized
  "ttl": 1,
}
```

##### **`LOBBY_INFO`**
Requesting info requires a target.

```js
{
  "opcode": "LOBBY_INFO",
  "payload": any, // target, must be serializable
  "ttl": 1,
}
```

The response will return an object.

```js
{
  "opcode": "LOBBY_INFO",
  "payload": {
    "lobby_id": any,
    "host": string, // host instance ID
    "current_peers": int,
    "max_peers": int,
    "hidden": bool,
    "locked": bool,
    "metadata": any,
  }
  "ttl": 1,
}
```

-----

### Bridge Server Commands

These commands are used to bridge communication with clients on older CL2/3/4 protocols.

| `opcode` | Description |
| :--- | :--- |
| **`...`** | . . . |

### Relay Server Commands

These commands are used to improve the performance of broadcast messages.

| `opcode` | Description |
| :--- | :--- |
| **`...`** | . . . |