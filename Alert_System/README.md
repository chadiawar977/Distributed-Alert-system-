# Distributed Alert System — UDP-based (Java)

## GIN527 – Based on Prof. Joseph Zalaket's UDP lecture slides

---

## Overview

This system lets security personnel (admins) send real-time alerts to groups
of registered users over UDP. It is composed of three programs:

| Program       | Role                              |
| ------------- | --------------------------------- |
| `AlertServer` | Central hub — routes all messages |
| `AdminClient` | Used by a security person (admin) |
| `UserClient`  | Used by a regular group member    |

**Key design rules:**

- Groups are disjoint: each group belongs to exactly one admin.
- One admin can own multiple groups.
- Usernames are the only identity (no passwords).
- All state is stored in plain text files under `data/`.

---

## Protocol

All messages are plain UTF-8 strings, fields separated by `|`.

### Client → Server

```
REGISTER|<username>|<groupName>
LEAVE|<username>|<groupName>
QUESTION|<username>|<groupName>|<text>
CREATE_GROUP|<adminName>|<groupName>
ALERT|<adminName>|<groupName>|<message>
ALERT|<adminName>|ALL|<message>         ← broadcast to all admin's groups
INBOX|<adminName>
REPLY|<adminName>|<questionId>|<replyText>
```

### Server → Client (acknowledgements)

```
OK|<detail>
ERROR|<detail>
QUESTION_ID|<id>|<detail>
INBOX|EMPTY
INBOX\n<id>|<user>|<ip>|<port>|<group>|<text>|<status>\n...
```

### Server → Client (pushed, no request needed)

```
ALERT|<groupName>|<message>       ← pushed to all group members
REPLY|<adminName>|<replyText>     ← pushed to the original question asker
```

---

## File Storage (`data/` directory)

```
data/
  groups.txt              ← groupName|adminName  (one line per group)
  members_<group>.txt     ← username|ip|port     (one line per member)
  inbox_<admin>.txt       ← id|user|ip|port|group|text|status
```

All files are human-readable and can be inspected/edited with any text editor.

---

## How to Compile

```bash
mkdir out
javac -d out src/alertsystem/AlertServer.java \
             src/alertsystem/AdminClient.java  \
             src/alertsystem/UserClient.java
```

---

## How to Run

### 1. Start the server (once)

```bash
java -cp out alertsystem.AlertServer
```

The server listens on **UDP port 6700**.
The `data/` directory is created automatically in the current working directory.

### 2. Start an admin (security person)

```bash
java -cp out alertsystem.AdminClient localhost
```

On first prompt enter your admin username, e.g. `alice`.

#### Admin commands:

```
create <groupName>              — create a new group (disjoint; yours alone)
alert <groupName|ALL> <msg>    — send alert to one or all of your groups
inbox                          — view questions left by members
reply <questionId> <text>      — reply to a specific question
help
quit
```

### 3. Start a user (group member)

```bash
java -cp out alertsystem.UserClient localhost
```

On first prompt enter your username, e.g. `bob`.

#### User commands:

```
register <groupName>            — join a group (group must exist)
leave <groupName>               — leave a group
question <groupName> <text>     — leave a question for the group's admin
help
quit
```

---

## Example Session

**Terminal 1 — Server**

```
=== Alert Server started on port 6700 ===
```

**Terminal 2 — Admin "alice"**

```
> create zone-a
[Server] OK | Group 'zone-a' created. Members can now register.

> create zone-b
[Server] OK | Group 'zone-b' created. Members can now register.

> alert zone-a Intruder detected at gate 3!
[Server] OK | Alert delivered to 2 member(s) across 1 group(s)

> inbox
┌─── INBOX ──────────────────────────────────────────┐
│ ID      : 1714000000000
│ From    : bob (group: zone-a)
│ Question: Is the north exit safe?
│ Status  : PENDING
│────────────────────────────────────────────────────│
└────────────────────────────────────────────────────┘

> reply 1714000000000 Yes, north exit is clear.
[Server] OK | Reply sent to 'bob'
```

**Terminal 3 — User "bob"**

```
> register zone-a
[Server] OK | Registered 'bob' to group 'zone-a'

╔══════════════════════════════════════════════════╗
║  🚨  SECURITY ALERT  🚨                         ║
║  Group  : zone-a                                 ║
║  Message: Intruder detected at gate 3!           ║
╚══════════════════════════════════════════════════╝

> question zone-a Is the north exit safe?
[Server] QUESTION_ID | 1714000000000 | Question sent to admin of group 'zone-a'

┌── Reply from alice ──────────────────────────────┐
│ Yes, north exit is clear.
└────────────────────────────────────────────────────┘
```

---

## Multiple Security Persons (Admins)

Each admin owns disjoint groups:

- `alice` owns `zone-a`, `zone-b`
- `charlie` owns `zone-c`, `zone-d`

`alice` cannot send alerts to `zone-c` (the server rejects with ERROR).
Users can register to any group regardless of which admin owns it.

---

## Important UDP Notes (from the lecture slides)

- UDP is **connectionless** and **unreliable** — packets may be lost.
- The server uses `DatagramSocket` (port 6700) and `DatagramPacket` exactly
  as shown in the UDPServer/UDPClient examples in the slides.
- Clients bind to an OS-assigned free port; the server records the ip+port
  at registration time and uses it to push alerts and replies.
- No `setSoTimeout` is used on the server (it blocks on `receive()` forever,
  serving one request at a time in the main loop, as in the lecture example).
- The `UserClient` sets `setSoTimeout(3000)` only to avoid blocking forever
  while waiting for a registration acknowledgement.

---

## Port Reference

| Component  | Port       | Direction   |
| ---------- | ---------- | ----------- |
| Server     | 6700 (UDP) | listens     |
| Admin/User | random     | OS-assigned |

All on the same host for local testing; replace `localhost` with the server's
IP address for a real network deployment.
