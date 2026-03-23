# Simple Concurrent Key-Value Store in Rust

A simple, experimental, in-memory key-value store built from scratch using only the Rust standard library.  
It features a TCP server, concurrency, persistence (AOF + snapshots), and graceful shutdown – a perfect exercise in systems programming.

## Features

- TCP server – Listens for commands over a simple line-based protocol.
- Concurrent clients – Each connection is handled in a separate thread.
- In-memory storage – HashMap<String, String> protected by an RwLock.
- Persistence  
  - Append-Only Log (AOF): every mutating command is logged; the log is replayed on startup.  
  - Optional snapshot (SAVE/LOAD commands) to dump the entire dataset to a file.
- Graceful shutdown – The server stops accepting new connections and waits for existing ones to finish.
- No external crates – Only std is used.
- Error handling – Custom error types, proper Result returns, and logging.

## Getting Started

### Prerequisites

- Rust (stable) – the code is tested with Rust 1.70+.

### Building

```
git clone <your-repo>
cd rust-kv-store
cargo build --release
```

### Running

```
cargo run --release
```

By default, the server binds to 127.0.0.1:6379 and stores data in ./data.

#### Configuration via Environment Variables

| Variable      | Default         | Description                 |
|---------------|-----------------|-----------------------------|
| KV_PORT       | 6379            | Port to listen on           |
| KV_ADDR       | 127.0.0.1:6379  | Full address (overrides port) |
| KV_DATA_DIR   | ./data          | Directory for AOF and snapshots |

Example:
```
export KV_PORT=8080
export KV_DATA_DIR=/var/lib/kvstore
cargo run --release
```

## Protocol

The protocol is line-based. Each command is a single line with arguments separated by spaces.  
Responses end with a newline (\n).

### Commands

| Command             | Format                         | Response                              |
|---------------------|--------------------------------|---------------------------------------|
| SET key value       | SET foo bar                    | OK\n                                  |
| GET key             | GET foo                        | value\n or (nil)\n                    |
| DEL key             | DEL foo                        | :1\n (number of deleted keys)         |
| SAVE                | SAVE                           | OK\n                                  |
| LOAD                | LOAD                           | OK\n                                  |
| SHUTDOWN            | SHUTDOWN                       | OK\n (server stops)                   |

All responses are terminated by \n.  
If an error occurs, the response is ERROR <message>\n.

## Usage Examples

Using telnet or nc:

```
$ telnet 127.0.0.1 6379
Trying 127.0.0.1...
Connected to localhost.
SET name Alice
OK
GET name
Alice
DEL name
:1
GET name
(nil)
SAVE
OK
SHUTDOWN
OK
Connection closed by foreign host.
```

## Persistence

### Append-Only Log (AOF)

Every mutating command (SET, DEL) is appended to appendonly.aof inside the data directory.  
On startup, the log is replayed to rebuild the in-memory state.

### Snapshots

- SAVE – writes the entire current state to snapshot.rdb.  
- LOAD – loads the last snapshot into memory (overwrites current data).

Snapshots are simple tab-separated files: each line is key\tvalue\n.  
The AOF and snapshot are independent; you can use both, one, or none.

## Design Overview

- Thread-per-connection – Each client connection is handled in its own std::thread. This keeps the implementation simple and uses only the standard library. For higher concurrency, a thread pool could be added.
- Shared state – The Store is wrapped in Arc<RwLock<HashMap<...>>> to allow concurrent reads and exclusive writes.
- AOF logging – The AOF is protected by a Mutex because multiple threads may try to write to it simultaneously. Every mutating command is logged synchronously (flushed immediately) to ensure durability.
- Graceful shutdown – An AtomicBool flag is set when a shutdown is requested (via the SHUTDOWN command or typing shutdown in the server’s console). The main accept loop stops, and each connection thread checks the flag before reading the next command.
- Error handling – A custom KvError enum is used throughout, with From implementations for std::io::Error, std::str::Utf8Error, and PoisonError. All operations return Result.
- Logging – Simple eprintln! with timestamps is used; a mutex prevents interleaved output.

## Limitations & Future Improvements

- No thread pool – Under high load, spawning a thread per connection may become expensive. A thread pool could be added using std::sync::mpsc channels.
- Signal handling – The current implementation uses a console input to trigger shutdown; in production you would use proper signal handlers (SIGINT, SIGTERM). (This can be done with a crate like ctrlc, but the constraint was no external crates.)
- No authentication – The server is open to anyone who can connect.
- Simple snapshot format – The snapshot is a plain text file; for production, you might want a more robust binary format.
- Blocking writes – The AOF flushes after every command, which may be slow. You could batch writes or use BufWriter with periodic flushing.
- No replication – This is a single-node store.

## License

This project is provided under the MIT License. Feel free to use and modify it.

## Acknowledgments

This project is a classic systems programming exercise that demonstrates many core Rust concepts: concurrency, ownership, error handling, file I/O, and networking, all without any external dependencies.
