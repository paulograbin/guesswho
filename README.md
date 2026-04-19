# Guess Who? - Multiplayer Network Game

A multiplayer implementation of the classic **"Guess Who?"** board game, built with Java Swing and TCP sockets. Two players connect over the network — one hosts the server, the other joins as a client — and take turns asking yes/no questions to deduce each other's secret character.

## How the Game Works

1. **One player starts the server**, the other connects as a client
2. Each player is **randomly assigned a secret character** that only their opponent can see
3. Players take turns asking **yes/no questions** via the built-in chat to narrow down which character the opponent holds
4. Click on character portraits to **eliminate** them from your board (toggles them on/off)
5. When you think you know the answer, hit **"Palpite"** (Guess) — guess correctly and you win; guess wrong and you lose immediately

### Characters

The game features 5 characters to choose from:

| Character | |
|---|---|
| Emma Watson | Emilia Clarke |
| Natalie Portman | Scarlett Johansson |
| Mila Kunis | |

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      Initial.java                       │
│               (Launcher - picks role)                   │
│         no args → Server  |  "client" → Client          │
└────────────────────┬──────────────────┬─────────────────┘
                     │                  │
          ┌──────────▼──────┐  ┌────────▼─────────┐
          │  ServidorCC.java│  │  ClienteCC.java   │
          │  (Server GUI)   │  │  (Client GUI)     │
          │                 │  │                   │
          │  ServerSocket   │  │  Socket            │
          │  port 1111      │◄─┤  → 127.0.0.1:1111 │
          └────────┬────────┘  └────────┬──────────┘
                   │                    │
                   │   Mensagem objects │
                   │◄──(serialized)────►│
                   │   via TCP socket   │
                   │                    │
          ┌────────▼────────────────────▼──────────┐
          │          Shared Domain Model            │
          │  Personagem.java  - character card      │
          │  Mensagem.java    - network message     │
          │  Assets.java      - codes & resources   │
          └────────────────────────────────────────┘
```

### Key Components

| File | Role |
|------|------|
| `Initial.java` | Entry point. Displays a launcher window with buttons to host (server), connect (client), or read the rules. Also acts as CLI router: run with no args for server, `client` arg for client. |
| `ServidorCC.java` | Server-side JFrame. Opens a `ServerSocket` on port 1111, waits for one client, randomly picks a secret character, and validates incoming guesses. |
| `ClienteCC.java` | Client-side JFrame. Connects to the server, receives the server's secret character selection, randomly picks its own, and sends guesses/chat messages. |
| `Personagem.java` | Domain model for a character card. Holds a name, photo (`ImageIcon`), and a visibility flag that toggles when the player clicks to eliminate it. |
| `Mensagem.java` | Serializable message envelope sent over the socket. Contains a message code (int), sender name, text payload, and timestamp. |
| `Assets.java` | Constants: image resource paths and the protocol's message codes (`Codigos`). |

### Network Protocol

Communication uses **Java Object Serialization** over TCP. Both sides exchange `Mensagem` objects through `ObjectOutputStream`/`ObjectInputStream`. Each message carries an integer code that determines how the receiver handles it:

| Code | Constant | Meaning |
|------|----------|---------|
| `-1` | `TERMINATE` | End the game / disconnect |
| `0` | `SERVER_CHAR_SELECTED` | Server tells client which character was picked for the server |
| `1` | `CLIENT_CHAR_SELECTED` | Client tells server which character was picked for the client |
| `2` | `CLIENT_GUESS` | Client submits a guess |
| `3` | `RECEBEUMENSAGEMCHAT` | Chat message (either direction) |
| `6` | `SINCRONIZA` | Sync/keep-alive |
| `9` | `SERVER_GUESS` | Server submits a guess |
| `10` | `SERVER_WON` | Server won the game |
| `11` | `CLIENT_WON` | Client won the game |

### Game Flow Sequence

```
  Server                              Client
    │                                    │
    │  (listens on port 1111)            │
    │◄───────── TCP connect ─────────────│
    │                                    │
    │── SERVER_CHAR_SELECTED (code 0) ──►│  "Server's secret is X"
    │◄── CLIENT_CHAR_SELECTED (code 1) ──│  "Client's secret is Y"
    │                                    │
    │◄─────── Chat (code 3) ────────────►│  Yes/no questions
    │                                    │
    │◄── CLIENT_GUESS (code 2) ─────────│  Client guesses
    │── CLIENT_WON or SERVER_WON ───────►│  Result
    │                                    │
    │── SERVER_GUESS (code 9) ──────────►│  Server guesses
    │◄── (client validates locally) ─────│  Result
```

## Prerequisites

- **Java 23+** (managed via [SDKMAN](https://sdkman.io/) — see `.sdkmanrc`)
- **Maven 3.6+**

If you use SDKMAN, the correct Java version is applied automatically when you `cd` into the project:

```bash
sdk env install   # one-time setup
```

## Building

```bash
mvn clean package
```

This produces:
- `target/untitled-1.0-SNAPSHOT.jar` — the main JAR
- `target/dependency-jars/` — all runtime dependencies

## Running

You need **two terminals** (or two machines on the same network):

**Terminal 1 — Start the server:**

```bash
java -cp "target/untitled-1.0-SNAPSHOT.jar:target/dependency-jars/*" com.paulograbin.Initial
```

**Terminal 2 — Start the client:**

```bash
java -cp "target/untitled-1.0-SNAPSHOT.jar:target/dependency-jars/*" com.paulograbin.Initial client
```

> **Note:** The client currently connects to `127.0.0.1:1111` (localhost). To play over a network, you would need to change the IP in `ClienteCC.java`.

## UI Layout

Each player sees the same layout:

```
┌──────────────────────────────────────────────────┐
│  [Emma]  [Emilia]  [Natalie]  [Scarlett]  [Mila] │  ← Click to eliminate
├────────────────────────────────┬─────────────────┤
│  Chat area (read-only)         │ Escolhido: ...  │  ← Your secret character
│                                │                 │
│  [Message input] [Enviar]      │ [Palpite]       │  ← Guess button
│                                │ [Finalizou]     │
└────────────────────────────────┴─────────────────┘
```

- **Top panel:** 5 character portraits. Click one to toggle it (eliminated characters show a placeholder image).
- **Bottom-left:** Chat panel for asking yes/no questions. Type a message and click "Enviar" (Send).
- **Bottom-right:** Shows your secret character label, a "Palpite" (Guess) button to submit your final answer, and a "Finalizou jogada" (End turn) button.

## Project Structure

```
guesswho/
├── pom.xml                          # Maven build config (Java 23)
├── .sdkmanrc                        # SDKMAN Java version
├── lib/                             # Local JARs (IntelliJ forms, TableLayout)
├── src/main/
│   ├── java/com/paulograbin/
│   │   ├── Initial.java             # Launcher / entry point
│   │   ├── ServidorCC.java          # Server GUI + networking
│   │   ├── ClienteCC.java           # Client GUI + networking
│   │   ├── Personagem.java          # Character model
│   │   ├── Mensagem.java            # Network message envelope
│   │   └── Assets.java              # Constants (image paths, codes)
│   └── resources/images/            # Character portrait images
└── javascriptServer/                # Experimental Node.js TCP client
    ├── server.js
    ├── index.html
    └── package.json
```

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Language | Java 23 |
| GUI | Swing + [FlatLaf](https://www.formdev.com/flatlaf/) (modern look-and-feel) |
| Layout | GroupLayout (NetBeans form editor) |
| Networking | `java.net.ServerSocket` / `Socket` with Java Object Serialization |
| Build | Maven |
| Java version manager | SDKMAN |

## Experimental: Node.js Server

The `javascriptServer/` directory contains an experimental Node.js TCP client that connects to the Java server on port 1111. It uses raw TCP (`net` module) and a Socket.IO-based HTML frontend. This is a work-in-progress and is **not integrated** with the main game logic.

## Author

Paulo Henrique Grolli Grabin
