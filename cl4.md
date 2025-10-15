# CloudLink 4 ("CL4") Protocol

CL4 was a major architectural revision of the CloudLink protocol. It moved away from the inconsistent formats of its predecessors to a unified JSON-based protocol and introduced powerful new concepts like **listeners** (for event-driven communication) and **rooms** (for message multicasting). While client-server, it laid the conceptual groundwork for many features in CL5.

## Client-to-Server Schema

All communication from a CL4 client to the server is a single JSON object.

```js
{
  "cmd": "...",
  "val": "...",
  "name": "...",
  "id": "...",
  "listener": "..."
}
```

`cmd` is a string that defines the command. `val` is the primary payload. `name` is used for variables, `id` is used for private message recipients, and `listener` is an optional key to tag a packet for an asynchronous response.

| `opcode` | Description | Dialect Notes |
| :--- | :--- | :--- |
| **`handshake`** | Sent upon connection to negotiate the protocol. | **v0.1.8**: Does not send this command.<br>**v0.1.9**: Sends a simple command with no `val` (e.g., `{"cmd": "handshake", "listener": "..."}`).<br>**v0.2.0**: Sends a detailed `val` object containing client info (e.g., `{"language": "Scratch", ...}`). |
| **`setid`** | Sets the client's username. `val` contains the username. | Consistent across all CL4 dialects. |
| **`link`** | Subscribes the client to one or more rooms. `val` contains the room ID(s). | **v0.1.9 and earlier**: `val` is a stringified JSON array (e.g., `"[\"default\"]"`).<br>**v0.2.0**: `val` is a native JSON array (e.g., `["default"]`). |
| **`unlink`** | Unsubscribes the client from all rooms. | Consistent across all CL4 dialects. |
| **`gmsg`** | A global message sent to all clients in the client's subscribed rooms. | The `val` can be any JSON-serializable type in **v0.2.0**, but is typically a string in older dialects. |
| **`pmsg`** | A private message sent to a specific recipient, specified by the `id` key. | The `val` can be any JSON-serializable type in **v0.2.0**, but is typically a string in older dialects. |
| **`gvar`** / **`pvar`** | A variable sync message. `name` specifies the variable name, and `val` is its new value. | The `val` can be any JSON-serializable type in **v0.2.0**, but is typically a string in older dialects. |

## Server-to-Client Schema

All communication from a CL4-aware server to the client is also a single JSON object.

```js
{
  "cmd": "...",
  "val": "...",
  "origin": { ... },
  "name": "...",
  "listener": "...",
  "code": "...",
  "mode": "..."
}
```

`origin` is an object identifying the original sender. `listener` is an echo of the client's tag. `code` is for status messages, and `mode` is used for differential `ulist` updates.

| `opcode` | Description | Dialect Notes |
| :--- | :--- | :--- |
| **`gmsg`** / **`pmsg`** | A global or private message payload. | Consistent across all CL4 dialects. |
| **`gvar`** / **`pvar`** | A variable update. `name` specifies the variable, `val` is its new value. | Consistent across all CL4 dialects. |
| **`ulist`** | The list of connected users in a room. | **v0.1.9 and earlier**: The `val` is a single string with usernames separated by semicolons (e.g., `"UserA;UserB;"`).<br>**v0.2.0**: The `val` is a JSON array of user objects (`[{"username": "UserA", ...}]`) and a `mode` key (`set`, `add`, `remove`) is used for differential updates. |
| **`statuscode`** | A status message from the server, often in response to a command with a `listener`. | Support begins with late **CL3 (v0.1.7)** and is standard in all CL4 dialects. |
| **`server_version`** | Informs the client of the server's version. | This is the modern format. **CL3-era** servers would send this same info inside a `direct` command with a `vers` subcommand. |
| **`motd`** | Sends the server's Message of the Day. | Support begins with late **CL3 (v0.1.7)** and is standard in all CL4 dialects. |
| **`client_obj`** | Sent after a successful handshake to provide the client with its own user object as the server sees it. | Exclusive to **v0.2.0** and newer. |
