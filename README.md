# STOMP Messaging System

A distributed, real time messaging system implemented across three languages: a Java STOMP protocol server, a C++ command line client, and a Python persistence server. The system delivers live World Cup match events to subscribed clients over TCP/IP using a publish/subscribe model, and generates per user match summaries from the received event stream. Built as the final project for the Systems Programming (SPL) course at Ben Gurion University of the Negev.

## Tech Stack

| Area | Technologies |
| :--- | :--- |
| Languages | Java, C++11, Python 3 |
| Networking | TCP/IP sockets, STOMP 1.2 protocol |
| Concurrency | Java threads, Thread Per Client, Reactor pattern, thread safe collections |
| Data | JSON event parsing, SQL (SQLite) persistence |
| Build | Maven, GNU Make |

## Architecture

The system is composed of three independent processes that communicate over TCP.

```
┌──────────────────┐        STOMP over TCP        ┌──────────────────┐
│   C++ Client     │ ◄─────────────────────────►  │   Java Server    │
│                  │         (port 7777)          │                  │
│ • Socket I/O     │                              │ • Thread Per     │
│ • JSON parsing   │                              │   Client / Reactor│
│ • Game state     │                              │ • Channel routing │
│ • Summaries      │                              │ • Frame codec     │
└──────────────────┘                              └────────┬─────────┘
                                                           │ TCP (port 7778)
                                                           ▼
                                                  ┌──────────────────┐
                                                  │  Python SQL      │
                                                  │  Server          │
                                                  │ • SQLite storage │
                                                  │ • Users, logins, │
                                                  │   report history │
                                                  └──────────────────┘
```

### Java STOMP Server
A concurrent server implementing the STOMP messaging protocol on top of a reusable networking framework. It supports two interchangeable server architectures selected at startup: a Thread Per Client model backed by blocking connection handlers, and a non blocking Reactor model driven by NIO selectors and a fixed size actor thread pool. The server owns a shared connections registry that maps client identifiers to active handlers and maintains subscription tables per channel, so a single SEND frame fans out to every subscriber of a topic. All shared state is guarded for concurrent access, since many client threads read and mutate it simultaneously.

### C++ Client
A command line client that maintains two concurrent execution paths: one reading user keyboard input and one listening for inbound frames from the socket. It encodes user commands into STOMP frames, parses incoming MESSAGE frames back into typed event objects, and reads match event files in JSON format using a header only JSON library. The client keeps a local, per channel and per user view of game state (general statistics, team statistics, and an ordered event log) and renders a formatted match summary file on demand.

### Python SQL Server
A standalone data server that accepts null terminated commands over TCP and persists them to SQLite. It stores registered users, login and logout sessions, and the history of submitted event reports, giving the system durable state that survives a restart of the messaging server.

## Features

- **Full STOMP frame support**: CONNECT, SUBSCRIBE, UNSUBSCRIBE, SEND, and DISCONNECT, with the corresponding CONNECTED, MESSAGE, RECEIPT, and ERROR responses.
- **Publish/subscribe channel routing**: clients join topics by game name and receive every event published to that topic in real time.
- **Two server concurrency models**: the same protocol implementation runs unchanged under Thread Per Client or under a non blocking Reactor, demonstrating a clean separation between protocol logic and transport layer.
- **Custom encoder and decoder**: a byte level codec that reassembles STOMP frames from arbitrary TCP packet boundaries using null terminated framing.
- **User session management**: login with credential validation, rejection of duplicate active sessions, and clean state teardown on disconnect.
- **JSON event ingestion**: match event files are parsed on the client and published to the correct channel as structured frames.
- **Automated match summaries**: the client aggregates received events per user and writes a chronological, human readable report to disk.
- **Persistent storage layer**: users, sessions, and report submissions are recorded in SQLite through a dedicated Python service.
- **Robust error handling**: malformed frames, unauthorized operations, and unknown subscriptions produce descriptive ERROR frames and terminate the offending connection gracefully.

## How to Run

> **Prerequisites**: JDK 8 or higher, Maven, a C++11 capable compiler (g++), Boost system library, and Python 3.

The system requires three terminals. All paths are relative to the project root at `Assignment 3 SPL Skeleton/Assignment 3 SPL`.

**1. Start the Python SQL server** (listens on port 7778)

```bash
cd data
python3 sql_server.py
```

**2. Start the Java STOMP server** (listens on port 7777)

```bash
cd server
mvn compile
mvn exec:java -Dexec.mainClass="bgu.spl.net.impl.stomp.StompServer" -Dexec.args="7777 tpc"
```

Replace `tpc` with `reactor` to run the non blocking server implementation.

**3. Build and start the C++ client**

```bash
cd client
mkdir -p bin
make
./bin/StompWCIClient
```

**Example client session**

```text
login 127.0.0.1:7777 alice pass
join Germany_Japan
report data/events1.json
summary Germany_Japan alice summary.txt
logout
```

A summary file is produced once the client has received a kickoff event, at least one goal, a halftime event, and a final whistle for that channel.

## Repository Structure

```text
server/     Java STOMP server, networking framework, and protocol implementation
client/     C++ client, connection handler, protocol logic, and event model
data/       Python SQL server and SQL client
client/data/ Sample JSON match event files
tests/      Client side unit tests
```

## About

Developed as Assignment 3 of the Systems Programming course (SPL), Department of Computer Science, Ben Gurion University of the Negev.
