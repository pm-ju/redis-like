# Redis-Inspired Key-Value Server

A small C++17 TCP key-value server developed in phases. Phase 2 adds bounded concurrency, synchronized storage, lazy TTL expiration, and real request pipelining while preserving the Phase 1 text protocol.

## Architecture

```text
Clients
   |
   v
TCP Server / accept loop
   |
   v
Bounded socket-task queue (64 pending tasks)
   |
   v
Worker pool (configurable, no detached threads)
   |
   v
ClientSession (one complete ordered stream per worker task)
   |
   v
CommandParser --> thread-safe KvStore --> lazy TTL
```

The accept loop owns newly accepted descriptors until a task is successfully submitted. A worker then owns one descriptor for the entire client session and closes it through RAII. This means commands from one connection stay sequential and ordered, while different connections execute concurrently.

## Phase history

Phase 1 established POSIX TCP networking, line parsing, storage abstraction, defensive limits, the interactive client, and basic integration tests. Phase 2 adds a `std::thread` worker pool, a condition-variable task queue, shared-mutex-protected storage, TTL, bounded backpressure, concurrent tests, and graceful worker shutdown.

## Commands

- `PING` returns `PONG`.
- `SET <key> <value>` stores a persistent value. Values may contain spaces.
- `SET <key> <value> <ttl_seconds>` stores a value with a non-negative TTL.
- `GET <key>` returns the value or `(nil)`.
- `DEL <key>` returns `(integer) 1` or `(integer) 0`.
- `HELP` lists commands.
- `QUIT` returns `BYE` and closes the connection.

Because the protocol preserves values containing spaces, a final numeric token is interpreted as the optional TTL. For example, `SET session Rupdip 10` expires after ten seconds, while `SET message hello world` stores `hello world`. TTL `0` expires immediately; negative, overflowing, and values above ten years are rejected. A non-numeric final token remains part of a space-containing value because the line protocol cannot otherwise distinguish it from ordinary value text. Expiration is lazy: `GET` and `DEL` remove expired entries when they encounter them.

Example:

```text
> SET name Rupdip
OK
> SET age 22
OK
> GET name
Rupdip
> SET session Rupdip 10
OK
> GET session
Rupdip
> DEL name
(integer) 1
```

## Build and run

```sh
mkdir build && cd build
cmake ..
cmake --build .
ctest --output-on-failure
```

Start the server on `127.0.0.1:6380`:

```sh
./kv_server
./kv_server 6380 4
```

The second form selects four workers. The default is `std::thread::hardware_concurrency()` with a fallback of four; valid configured values are 1 through 256. Connect with the interactive client:

```sh
./kv_client
./kv_client 127.0.0.1 6380
```

## Concurrency and shutdown

The queue is protected by a mutex and condition variable. It accepts at most 64 waiting socket tasks; a full queue receives `(error) server busy` and the connection is closed. Workers drain queued tasks during shutdown, and active sessions use a receive timeout to observe SIGINT and exit. All worker threads are joined.

`KvStore` uses a `std::shared_mutex` to protect its `std::unordered_map`. Writes use exclusive locking. Reads also use an exclusive lock because lazy expiration may erase an expired entry; this keeps the expiration check and deletion atomic and understandable.

Pipelined commands are buffered until newline boundaries. Multiple complete commands in one TCP read and commands split across reads are both supported. Responses are emitted in the same order for each connection. CRLF input is accepted.

Lines are limited to 4096 bytes, keys to 256 bytes, values to 2048 bytes, and pending queue tasks to 64. The server is local and uses IPv4 POSIX sockets; it does not provide authentication, persistence, clustering, epoll, asynchronous I/O, or protocol framing such as RESP.

## Tests and sanitizer build

CTest includes:

- unit tests for storage, TTL, parser edge cases, and the worker pool;
- real TCP integration tests for pipelining, partial reads, CRLF, TTL, reconnects, shutdown, and concurrent clients;
- a 20-client, 100-operation-per-client stress test.

For GCC or Clang, an optional ThreadSanitizer build is available:

```sh
cmake -S . -B build-tsan -DENABLE_THREAD_SANITIZER=ON
cmake --build build-tsan
ctest --test-dir build-tsan --output-on-failure
```

## Phase 3 preview

Phase 3 can add a dedicated load generator, throughput and p50/p95/p99 latency measurement, performance tuning, and larger benchmark matrices. Those features are intentionally outside this Phase 2 implementation.
