# CloudLink 5 ("Omega") Protocol
CL5 was the first version of CloudLink to switch away from a solely server-client to a hybrid server-client/peer-to-peer architecture. 

## In-band (peer-to-peer) schema
```js
{
  "opcode": "...",
  "payload": { ... },
  "id": "...",
  "method": "..."
}
```

`opcode` is a string that defines what the packet is used for. 
`payload` can be any arbitrary data type that can be serialized using JSON.
`id` and `method` are optional properties that only apply when using the variable/list data synchronization feature.

| `opcode`       | Description                                                                                                                                                                                  |
| -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| G_MSG          | A Global Message intended to be broadcast.                                                                                                                                                   |
| P_MSG          | A Private Message intended only for the recipient.                                                                                                                                           |
| G_VAR/P_VAR    | A Variable Sync message. The payload contains the new value, and an id key specifies which networked variable to update.                                                                     |
| G_LIST/P_LIST  | List Sync message. The payload contains the new data, an id key specifies the networked list, and a method key (`set`, `replace`, `length`, etc.) specifies how the list should be modified. | 
| NEW_CHAN       | A control message sent on the `default` channel to negotiate and open a new, named data channel between the two peers.                                                                       |
| HANGUP/DECLINE | Voice call control signals sent between peers to manage the state of a call.                                                                                                                 |

## Out-of-band (server-to-client) schema
```js
{
  "opcode": "...",
  "payload": { ... },
}
```

Akin to the in-band schema, both `opcode` and `payload` are retained and have the same functionality. `id` and `method` are not included.

| `opcode`            | Description                                                                                                                                                 |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| INIT                | Initializes a session with the server. A valid username, token, or credential must be supplied. An optional public key can be passed through for E2EE.      |  
| CREATE_LOBBY        | Requests the server to create a new lobby. The payload includes the lobby name, max players, password, and other settings.                                  |
| JOIN_LOBBY          | Requests to join an existing lobby. The payload includes the lobby name and a password if required.                                                         |
| LIST_LOBBIES        | Requests an updated list of all public lobbies from the server.                                                                                             |
| FIND_LOBBY          | Requests detailed information about a single public lobby.                                                                                                  |
| MANAGE_LOBBY        | Sent by the lobby host to manage the lobby (e.g., kick players, change password, transfer ownership). The payload contains a method and args.               |
| KEEPALIVE           | Sent periodically to prevent the WebSocket connection from timing out, if enabled.                                                                          |
| INIT_OK             | The server's successful response to an `INIT` request. The payload contains the client's newly assigned `instance_id`, `user_id`, and confirmed `username.` |
| NEW_PEER/PEER_JOIN  | Informs clients in a lobby that a new peer has joined. The payload is a user object containing the new peer's ID, username, and public key.                 |
| PEER_LEFT           | Informs clients that a peer has left the lobby. The payload contains the ID of the disconnected peer.                                                       |
| LIST_ACK/FIND_ACK   | The server's response to `LIST_LOBBIES` or `FIND_LOBBY`, with the requested lobby data in the payload.                                                      |
| CREATE_ACK/JOIN_ACK | Confirms that a lobby was successfully created or joined.                                                                                                   |
| LOBBY_CLOSED        | An alert that is sent to all clients when the host closes the lobby.                                                                                        |
| TRANSITION          | Informs a client that its role or "mode" has changed (e.g., from "peer" to "host").                                                                         |
| RELAY               | Instructs a client to connect to a specific peer that will act as a TURN-like relay for the lobby.                                                          |
