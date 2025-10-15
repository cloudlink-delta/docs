# CloudLink 3 ("CL3") Protocol

CloudLink 3 was a critical transitional phase for the protocol. It marked the complete shift away from the legacy multi-line string format (CL2) to a more modern, fully JSON-based system. While it laid the groundwork for CL4, it remained a straightforward, polling-based client-server protocol and did not include advanced features like rooms or listeners.

## Client-to-Server Schema

All communication from a CL3 client to the server is a single JSON object. The structure is simpler than CL4, with some commands requiring an `origin` key.

```js
{
  "cmd": "...",
  "val": "...",
  "name": "...",
  "id": "...",
  "origin": "..."
}
```

`cmd` is the command, `val` is the payload, `name` is for variables, `id` specifies a recipient, and `origin` is used to identify the sender in private messages.

| `opcode` | Description | Dialect Notes |
| :--- | :--- | :--- |
| **`setid`** | Sets the client's username. The `val` key contains the username. | Consistent across all CL3 dialects. |
| **`direct`** | A command for direct server interaction. Used by clients to identify their type. | **v0.1.7**: The client sends a payload of `{"cmd": "type", "val": "scratch"}` upon connection to identify itself. |
| **`gmsg`** | A global message sent to all clients. The `val` key contains the data payload. | Consistent across all CL3 dialects. |
| **`pmsg`** | A private message. `val` is the data, `id` is the recipient, and `origin` is the sender's username. | Consistent across all CL3 dialects. |
| **`gvar`** / **`pvar`** | A variable sync message. `name` specifies the variable name, and `val` is its new value. `pvar` also requires `id` and `origin`. | Consistent across all CL3 dialects. |

## Server-to-Client Schema

Server responses are also JSON objects, but the structure can be less consistent than in CL4. Metadata is often nested within a `direct` command.

```js
{
  "cmd": "...",
  "val": "...",
  "name": "...",
  "origin": "..."
}
```

| `opcode` | Description | Dialect Notes |
| :--- | :--- | :--- |
| **`gmsg`** / **`pmsg`** | A global or private message payload. | Consistent across all CL3 dialects. |
| **`gvar`** / **`pvar`** | A variable update. `name` specifies the variable, `val` is its new value. | Consistent across all CL3 dialects. |
| **`ulist`** | The list of connected users. The `val` is a single string with usernames separated by semicolons (e.g., `"UserA;UserB;"`). | This string-based format is a key characteristic of CL3 and earlier protocols. |
| **`direct`** | A command used by the server to send metadata to the client. The `val` is a nested object containing a subcommand (`vers`, `motd`) and its value. | **v0.1.7**: This is the primary method for sending the server version (`vers`) and Message of the Day (`motd`).<br>**v0.1.5**: Does not support receiving `motd`. |
| **`statuscode`** | A status message from the server, often in response to a command like `setid`. | Support for receiving status codes begins with dialect **v0.1.7**. |
| **`ds`** | A command sent from the server to indicate it is shutting down, instructing the client to disconnect. | Consistent across all CL3 dialects. |
