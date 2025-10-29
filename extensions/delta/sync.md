# CLΔ Sync Plugin
Enables list/variable synchronization between players for CLΔ Core.

CLΔ Sync is a powerful plugin that allows you to synchronize Scratch variables and lists between different players in real-time without needing any additional code. It's built to be highly efficient, de-duplicating redundant data and batching events to keep your project running fast with minimal network traffic.

## What is a Tag?

A **Tag** is the most important concept in this plugin. Think of it as a **public, network-safe nickname** or a **mailing label** for your variable.

* **The Problem:** Your "my variable" on "Sprite1" is not the same as your friend's "my variable" on their "Sprite1." They are two different variables with different internal IDs.
* **The Solution:** You both agree on a unique, shared "tag," like `"player_1_x_position"`.
    * **You (Player 1)** link *your* variable to the tag `"player_1_x_position"`.
    * **Your Friend (Player 2)** links *their* variable to the *exact same tag*.

Now, when you update your variable, the plugin sends a message saying: "Hey everyone, the variable tagged `"player_1_x_position"` just changed to `120`." Your friend's plugin, which is listening for that tag, receives the message and automatically updates *their* bound variable to `120`.

**Tags *must* be unique for each variable or list you want to sync.**

---

## Getting Started
*CLΔ Core is required for the sync plugin to work. Make sure you load it first before loading CLΔ Sync.*

Let's sync a "global variable" (on the Stage) between two players.

### Player A (Sender)
1.  Go to the Stage and create a new variable named `(Player A X)`.
2.  Drag out this block and set it up like this:
    * **[enable]** broadcast of **[global variable]** **[Player A X]** in channel **[default]** and assign it as tag **[p1_x]**

Now, any time you change `(Player A X)`, its new value is broadcast to everyone in the "default" channel with the tag "p1_x".

### Player B (Receiver)
1.  Go to the Stage and create a new variable named `(Player A's X)`.
2.  Drag out the *exact same* block. The only difference is your local variable name.
    * **[enable]** broadcast of **[global variable]** **[Player A's X]** in channel **[default]** and assign it as tag **[p1_x]**

### Result
Now, whenever Player A changes their `(Player A X)` variable, Player B will instantly see their `(Player A's X)` variable update to match. You have successfully synced a variable!

To stop syncing, simply run the same block with **[disable]** selected. This will un-link the variable and restore it to a normal, non-networked state.

---

## How It Works

* **Change Detection** - The plugin wraps your variable in a special "proxy" object that instantly detects when its value changes.
* **De-duplication** - If you set a variable to the *same value* it already has (e.g., `set [my_var v] to (10)` when it's already 10), the plugin knows not to send an unnecessary network message. This dramatically reduces network spam.
* **Event Batching** - Instead of sending a message for every single change (which can be 30 times a second!), the plugin collects all unique changes that happen in a small time window (e.g., 100ms) and sends them as a single, efficient packet.
* **Idle Detection** - If nothing changes for more than a second, the sync engine goes to sleep to save resources, waking up instantly when a new change occurs.

---

## Reference

### Sync Commands
Use these blocks to start or stop syncing a variable.

> **[MODE] broadcast of [TYPE] [VAR] in channel [CHANNEL] and assign it as tag [TAG]**
> This is the main block. It links a local variable/list to a network tag and sends any changes to *everyone* in that channel.
> * **MODE:** `enable` or `disable`.
> * **TYPE:** The scope and type of your variable (e.g., `global variable`, `local list`).
> * **VAR:** The name of your variable or list.
> * **CHANNEL:** The "room" to send the data to.
> * **TAG:** The unique network nickname for this variable.

> **[MODE] multicast of [TYPE] [VAR] in channel [CHANNEL] with peers in [MCAST] list [LIST] and assign it as tag [TAG]**
> Same as broadcast, but it only sends changes to the peers whose usernames are in your specified **[LIST]**. Useful for teams.
> The value of **[LIST]** can be changed during runtime, and accordingly, the peers you multicast to.

> **[MODE] unicast of [TYPE] [VAR] with [PEER] in channel [CHANNEL] and assign it as tag [TAG]**
> Same as broadcast, but it only sends changes to a *single* peer, specified by their username. Useful for direct messages or trading.

### Sync Tags (Reporters)
Use these blocks to check the status of your tags.

> **is tag [TAG] in use?**
> Returns `true` if you have already used this tag to sync a variable/list.

> **tag [TAG] currently bound to**
> Reports what a tag is linked to on your machine (e.g., "global variable: Player A X").

> **is tag [TAG] broadcasting in channel [CHANNEL]?**
> Returns `true` if the tag is currently set to broadcast mode in that specific channel.

> **is tag [TAG] multicasting...**
> Returns `true` if the tag is currently set to multicast mode with that specific list.

> **is tag [TAG] unicasting...**
> Returns `true` if the tag is currently set to unicast mode with that specific peer.

---