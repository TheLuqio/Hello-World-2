# TCP Chat Application

A multi-user terminal chat system built from scratch in Python using only the standard library. Users connect to a shared server, choose a username, and exchange messages inside rooms — no external packages required.

---

## Features

- Room-based messaging — users only receive messages from people in the same room
- JSON-over-newline framing protocol (delimiter framing, same technique as IRC/SMTP)
- Multi-threaded server — each connected client runs in its own daemon thread
- Two-thread client — one thread reads stdin, one thread receives and prints messages
- Username validation (alphanumeric, max 20 characters, must be unique)
- Structured logging to console and `server.log`
- Five message types: `login`, `chat`, `system`, `command`, `error`

---

## Project Structure

```
repo/
├── server.py        # Multi-threaded chat server
├── client.py        # Terminal chat client (two-thread design)
├── config.py        # Constants: host, port, message types, limits
├── server.log       # Runtime log file (created on first run)
├── commit_2/        # Intermediate version (rooms + JSON added, bug present)
├── commit_3/        # Final version (bug fixed, /invite and /rooms added)
└── explanation.txt  # Detailed code walkthrough
```

---

## How to Run

Open two terminal windows in the project directory.

**Terminal 1 — start the server:**
```bash
python server.py
```

**Terminal 2 — connect a client (repeat for additional users):**
```bash
python client.py
# or with explicit host and port:
python client.py 127.0.0.1 9090
```

The client will prompt for a username, then connect to the `general` room automatically.

> Requires Python 3.10+ (uses `str | None` union type hints). No `pip install` needed.

---

## Available Commands

| Command | Description |
|---|---|
| `/create <room>` | Create a new room and move into it |
| `/join <room>` | Move into an existing room |
| `/invite <user>` | Send another user an invitation to your current room |
| `/list` | Show all connected usernames |
| `/rooms` | Show all rooms and their member counts |
| `/quit` | Disconnect from the server |

Anything typed without a leading `/` is sent as a chat message to your current room.

---

## Protocol

Every message on the wire is a single line of JSON terminated by `\n`:

```json
{"type": "chat", "room": "general", "data": "hello!", "timestamp": "14:32:01"}
```

**Message types:**

| Type | Direction | Purpose |
|---|---|---|
| `login` | client → server | First message sent; carries the chosen username |
| `chat` | both | A normal chat message |
| `system` | server → client | Notifications (join/leave events, invitations) |
| `command` | client → server | A slash-command (`/join`, `/create`, etc.) |
| `error` | server → client | Something went wrong |

---

## Architecture

### Server (`server.py`)

Two shared data structures, each protected by its own lock:

- `clients` — `dict[username → socket]` guarded by `clients_lock`
- `rooms` — `dict[room_name → set[usernames]]` guarded by `rooms_lock`

Two separate locks are used intentionally: `broadcast()` snapshots the room member set under `rooms_lock`, releases it, then acquires `clients_lock` to send. Holding both simultaneously while doing socket I/O would stall every other thread.

Per-client lifecycle in `handle_client()`:
1. **Login** — wait for a JSON `login` message, validate username, register in `clients`
2. **Message loop** — read newline-delimited JSON; dispatch to `handle_command()` or `broadcast()`
3. **Cleanup** (`finally`) — remove from room and `clients` dict, close socket; runs even on exception

### Client (`client.py`)

- **Main thread** — blocks on `input()`, packages typed text as JSON, sends to server
- **Receiver thread** (daemon) — reads incoming JSON and calls `format_and_print()`; killed automatically when the main thread exits

`format_and_print()` output format:
```
[general] alice: hello!  (14:32:01)   ← chat
*** bob joined the room.               ← system
[ERROR] Room 'xyz' not found.          ← error
```

---

## Configuration (`config.py`)

| Constant | Default | Description |
|---|---|---|
| `SERVER_HOST` | `127.0.0.1` | Bind address |
| `SERVER_PORT` | `9090` | Listening port |
| `DEFAULT_ROOM` | `general` | Room all users join on login |
| `MAX_ROOM_USERS` | `50` | Maximum users per room |
| `MAX_USERNAME_LEN` | `20` | Maximum username length |
| `BUFFER_SIZE` | `4096` | TCP recv buffer size (bytes) |
| `LOG_FILE` | `server.log` | Log output file |
| `LOG_LEVEL` | `DEBUG` | Logging verbosity |

---

## Commit History

This repository preserves three development snapshots to show how the design evolved.

### commit\_1 (baseline)
- No rooms — everyone shares one global channel
- No JSON — messages are plain strings over the socket
- No `config.py` — constants hard-coded inline
- Simplest possible design: connect → username → chat → disconnect

### commit\_2 (rooms + JSON)
- JSON protocol introduced; all messages are now structured dicts with a `type` field
- `config.py` extracted; room system added (`room_create`, `room_join`, `room_leave`, `get_room_of`)
- Logging added (console + `server.log`)
- **Bug:** `/create` did not call `room_leave()` before joining the new room, so the user remained registered in the old room. `get_room_of()` would then return whichever room appeared first in the dict, causing chat messages to be delivered to the wrong room.
- Missing: `/invite` and `/rooms` commands
- Login was still a raw string, not JSON

### commit\_3 (final)
- **Bug fixed:** `/create` now calls `get_room_of()` + `room_leave()` before `room_join()`, matching the pattern `/join` already used
- `/invite` command added
- `/rooms` command added
- `format_and_print()` updated to show the `[room]` prefix on chat messages
- Login made protocol-consistent: client sends `{"type": "login", "username": "..."}` as JSON
- Username validation added: rejects empty, non-alphanumeric, or too-long names
- `MAX_USERNAME_LEN` added to `config.py`, removing the hard-coded magic number
