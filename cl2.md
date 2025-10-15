# CloudLink 2 ("CL2") Protocol
CloudLink 2 was a major milestone protocol of the CloudLink project. It is characterized by its asymmetric communication format: clients send commands as multi-line strings, while the server responds with structured JSON.  CL2 was the first mature version of CloudLink to use a standardized packet format that the original CloudLink failed at.

## Client-to-Server Schema
All communication from a CL2 client to the server is a multi-line string, with the command header on the first line and parameters on subsequent lines separated by a newline character (`\n`).

**Format:** `<%{COMMAND}}>\n{PARAM_1}\n{PARAM_2}\n...`

| `opcode` | Description | Dialect Notes |
| :--- | :--- | :--- |
| **`<%sn>`** | **Set Name**. Sets the client's username. `PARAM_1` is the username. | Foundational command, consistent across all CL2 versions. |
| **`<%ds>`** | **Disconnect**. Informs the server of disconnection. `PARAM_1` is the username. | Foundational command. |
| **`<%rf>`** | **Refresh**. Requests a fresh user list from the server. | Foundational command. |
| **`<%gs>`** | **Global Stream**. Broadcasts data to all connected clients. `PARAM_1` is the sender, `PARAM_2` is the data. | Foundational command. |
| **`<%ps>`** | **Private Stream**. Sends data to a specific client. `PARAM_1` is the sender, `PARAM_2` is the recipient, and `PARAM_3` is the data. Also used to communicate with server APIs by using a special recipient ID (e.g., `%CA%`) and a JSON string as the data payload. | Foundational command, but its use for APIs was expanded in later versions. |
| **`<%sh>`** | **Special Handshake**. Sent upon connection to request advanced features from a compatible server. | Exclusive to **Late CL2 (v0.1.5 / S2.2+)**. |
| **`<%rl>`** / **`<%rt>`** | **Register Link / Return**. Establishes or breaks a direct data link with another client. | Exclusive to **Late CL2 (v0.1.5 / S2.2+)**. |
| **`<%l_g>`** / **`<%l_p>`**| **Linked Global/Private**. Sends data or variables within an established client link. `PARAM_1` is a numeric mode (`0` for data, `1` for variable). | Exclusive to **Late CL2 (v0.1.5 / S2.2+)**. |

## Server-to-Client Schema
All communication from the server to a CL2 client is a JSON object.

**Format:** `{"type": "...", "data": "...", "id": "..."}`

| `type` | Description | Dialect Notes |
| :--- | :--- | :--- |
| **`gs`** | **Global Stream**. A broadcast message containing global data in the `data` key. | Foundational response. |
| **`ps`** | **Private Stream**. A private message containing data in the `data` key and the recipient in the `id` key. | Foundational response. |
| **`ul`** | **User List**. The `data` key contains a single string of all usernames, separated by semicolons (e.g., `"UserA;UserB;"`). | The semicolon-string format is a key feature of CL2. |
| **`ru`** | **Refresh Usernames**. A command from the server instructing clients to re-send their usernames for a hard sync. | Foundational response. |
| **`direct`** | A special packet containing metadata, like the server version (`"data": {"type": "vers", "data": "..."}`). | Sent in response to a **Late CL2 (v0.1.5 / S2.2+)** handshake. |
| **`sf`** | **Special Feature**. A wrapper packet for responses (like `gs` or variable updates) sent to a client that has completed a handshake. | Exclusive to **Late CL2 (v0.1.5 / S2.2+)**. |
| **`ca`**, **`cd`**, **`dd`** | API-specific responses for CloudAccount, CloudCoin, and CloudDisk, respectively. | Support for these APIs was added throughout the CL2 lifecycle. |
