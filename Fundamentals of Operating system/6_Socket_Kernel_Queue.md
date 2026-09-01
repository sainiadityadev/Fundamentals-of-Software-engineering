# Socket and Kernel Queues


When a backend application "listens" on a port, it is not directly handling TCP connections — the kernel does all of that. <br>The application's job is to set up a socket, let the kernel manage the handshake, and then call `accept()` to pull completed connections out of the kernel's queues.

## What is a Socket?

A socket is just a communication endpoint.<br>
In Linux, it is a File descriptor. And an Object in windows.<br>
A file descriptor is just an integer represents a resource from process's perspective.

### Two Sockets are there:

Listening Socket: created when your process calls `listen()`. It has an IP and port bound to it.<br>
It does not carry data; it is purely a door waiting for clients to knock.

Connection Socket: created per accepted client connection via `accept()`. It has a full 4-tuple: source IP, source port, destination IP, destination port. This is where actual data flows.

### Listening on IP and Port:

When you call `listen()`, you must bind to a specific IP address and port.<br>
A machine can have multiple NICs (Network Interface Cards), each with its own IP. You can listen on a specific IP (e.g., `127.0.0.1` for loopback during development) or on all interfaces using `0.0.0.0` (IPv4).<br>
Listening on all interfaces (`0.0.0.0`) is dangerous in production. If your machine has a NIC with a public IP, your service becomes publicly exposed. This is exactly how most MongoDB and Elastic search breaches happened.

## Kernel Queues

Every listening socket gets two queues managed entirely by the kernel.

### 1. SYN Queue (Incomplete Connection Queue)

When a client initiates a TCP connection, it sends a SYN. The kernel receives this SYN, places an entry in the SYN queue, and replies with a SYN-ACK. At this point, the connection is incomplete — we are waiting for the client's final ACK to complete the three-way handshake.

Each entry in the SYN queue holds the partial connection state (client IP, port, sequence numbers, etc.).

### 2. Accept Queue (Completed Connection Queue)

Client sends the ACK, kernel matches it to the SYN queue entry, removes it from the SYN queue, and moves a fully established connection into the accept queue. The connection is complete from a TCP perspective but not handed to the application yet.<br>
(The kernel implements these internally as hash tables, not literal linked-list queues).

Both queues have a maximum size called the Backlog. You specify this when calling `listen(fd, backlog)`. If the accept queue fills up, the kernel cannot move newly completed connections into it, which means it stops consuming SYN queue slots, which means new SYNs get rejected. 

### Three-Way Handshake — What Actually Happens

Client sends SYN → kernel receives it, adds entry to SYN queue, replies with SYN-ACK.<br>
Client sends ACK → kernel matches it to the SYN queue entry.<br>
Kernel removes entry from SYN queue, creates a full connection, and places it in the accept queue.<br>
Application calls `accept()` → kernel removes the connection from the accept queue, creates a new file descriptor(Connection Socket) in the calling process's file descriptor table, and returns it to the application.

All handshaking is done by the kernel. The application is only responsible for calling `accept()`.

## Connection File Descriptors and Per-Connection Queues

When `accept()` is called, a new file descriptor is created and lives in the calling process's PCB (Process Control Block).<br>
The kernel creates two additional queues per-connection:<br>
Receive Buffer/Queue — inbound data arriving from the client is placed here by the kernel until the application reads it.<br>
Send Buffer/Queue — outbound data written by the application is placed here; the kernel drains it to the NIC.

If you have 1000 accepted connections, you have 1000 file descriptors and 2000 kernel-managed queues.<br>
The client side mirrors this: a client also has a connection socket with a send queue and a receive queue, but no listening socket.

## SYN Flood Attack

A malicious client sends many SYNs but never sends the final ACK. <br>
Each unanswered SYN occupies a slot in the SYN queue. Fill the SYN queue completely and no new connections can be established. This is a denial-of-service.

To prevent this:

Timeouts — if the final ACK does not arrive within a configured timeout (e.g., ~100–500ms), the SYN entry is removed from the queue.

SYN Cookies — the server encodes connection state into the SYN-ACK itself rather than storing state in the SYN queue. This makes the SYN queue stateless and SYN floods ineffective.

## Socket Sharding: How multiple process can accept connections.

### Approach 1: Fork after listen

A single process calls `listen()` and gets a file descriptor. It then `fork()` a child process and it inherits all file descriptors including the listening socket FD. Both parent and child now hold FDs pointing to the same underlying socket (same SYN queue, same accept queue). Either process can call `accept()` and pull connections.

### Approach 2: Socket Sharding via SO_REUSEPORT

Multiple independent processes each create their own socket and bind to the same IP + port using the `SO_REUSEPORT` socket option. Each process has a distinct socket with its own separate accept queue, but all listen on the same port.<br>
The kernel load-balances and distributes completed connections across multiple accept queues. Each process calls `accept()` on its own socket and its own accept queue.<br>
To prevent a malicious process from intercepting packets on an already-listened port, the linux kernel requires the processes sharing the address to have the same effective User ID.

## Packet Processing from NIC

On receiving bytes from NIC, for every packet, the kernel creates an internal data structure called socket buffer (`sk_buff`).<br>
These structures carry the packet data plus metadata.<br>
Kernel then performs significant operation such as,<br>
Checks whether the packet is damaged or malformed.<br>
Reads IP and TCP/UDP headers.<br>
Finds which socket owns it.

On high-throughput machines, it is a recommended practice to dedicate one or two CPU cores exclusively to the NIC interrupt, kernel networking stack so that this processing does not contend with application threads. This is achievable via CPU affinity and IRQ pinning.

## Data Flow Summary

### Inbound (client → server):
```
Client sends data → NIC receives it → kernel creates `sk_buff` → kernel places data in connection's Receive Buffer/Queue → application reads from the file descriptor (syscall: `read`/`recv`) → data moves to userspace.
```
### Outbound (server → client):
```
Application writes to file descriptor (syscall: `write`/`send`) → data placed in connection's Send Queue → kernel drains Send Queue to NIC → NIC transmits to client.
```
## Writing to a socket doesn't mean immediate transmission

When application calls `write(socket_fd)`, the kernel does not immediately send a packet. The data sits in the send buffer first.

This is the exact same pattern as the page cache:

Write to page cache → data eventually flushed to disk<br>
Write to send buffer → data eventually flushed to the network

The kernel accumulates data in the send buffer and decides when to flush based on several factors, most notably the Nagle Algorithm.

### Nagle Algorithm

Nagle Algorithm reduces the number of small TCP segments over the network. <br>
Kernel holds data in send buffer and delays transmission if,<br>
There is unacknowledge data in flight, we haven't received ACK of a previous packet.<br>
Data to send is smaller than Max. Segment size(MSS).<br>
TCP/IP headers alone cost 40+ bytes. Sending 1 byte per packet wastes enormous bandwidth.

When Nagle Algorithm causes problems: The curl / TLS handshake case:<br>
Daniel Stenberg (curl author) was debugging a 300–500ms latency spike during TLS connection setup.<br>
Client sends a small TLS ClientHello packet -> unacknowledged data exists and payload is small → wait<br>
Result: 300–500ms stall before TLS handshake even gets started.

Fix: Disable Nagle Algorithm for this connection using `TCP_NODELAY` socket option. This tells the kernel: send immediately, even if the payload is tiny and there's unacknowledged data.

## TCP Flow Control

### The problem: slow readers

If your application is slow to read from sockets, not calling `read()` fast enough(e.g., doing heavy CPU work between reads, or blocked on I/O elsewhere), the receive buffer fills up.

Receive buffer fills<br>
Kernel advertises a smaller receive window in outgoing ACKs<br>
Sender sees the shrinking window and slows down its transmission rate<br>
If the window hits zero, the sender stops entirely and waits

This is TCP Flow Control, a built-in mechanism to prevent fast senders from overwhelming slow receivers.

### Two windows involved

Flow Control Window (receive window): Reflects how much buffer space the receiver has. Receiver controls this. Tells the sender "here's how much I can absorb right now."

Congestion Window (`cwnd`): Reflects network capacity — how many bytes can be in-flight without overwhelming the routers between sender and receiver. This is managed by the sender.

The effective send rate is always: `min(flow_control_window, congestion_window)`

## TCP Connection Identification

Every TCP connection is uniquely identified by a 4-tuple:<br>
(source IP, source port, destination IP, destination port)<br>
When a packet arrives at NIC, the kernel looks up this 4-tuple in its connection table, identifies which process and which file descriptor owns that connection, and locates the associated receive buffer.

## Full Data path, Complete flow:

### Inbound data flow

```
Packet arrives at NIC -> NIC DMA-writes it into RAM -> Kernel identifies connection via 4-tuple lookup -> Kernel creates a `sk_buff` and writes data to receive buffer -> application calls `read(fd)` -> copy to userspace heap.
```
### Outbound data flow
```
App builds data in userspace -> `write(socket_fd)` copies to kernel send buffer -> If applicable, Nagle Algorithm decides when to flush -> Kernel builds TCP segment + IP packet -> Sent to NIC -> network
```
# Socket programming patterns for Backend Systems

Every server built on sockets follows the same conceptual pipeline, but different systems split it across execution units differently.

Listener — owns the file descriptor bound to a port, waiting for incoming connection requests.

Acceptor — pulls connections off the OS accept queue, creating a new file descriptor (socket) per accepted connection.

Reader — reads raw bytes off a connection's kernel receive queue into user-space memory.

Parser — interprets those raw bytes according to a protocol's format (HTTP, SSH, DHCP, etc.), extracting headers and structure. This is CPU bound work since protocols have defined formats that must be decoded.

Decryptor — handles SSL/TLS encryption and decryption.

Processor/Worker — takes the parsed, semantically meaningful request for the application and actually executes the business logic to fulfill it.

A request is just a string of bytes that has been parsed into something with application-level (Layer 7) semantics.<br>
As a backend engineer, you can keep all of them in a single thread, or split any subset of them into dedicated threads, processes.

## Example

Node.js — Single-threaded event loop<br>
Node.js runs listener, acceptor, reader, and JS callback execution all on a single thread.

# Asynchronous I/O: Non-blocking Reads and Writes

Most syscalls in modern OS kernels (`read`, `write`, `accept`) are blocking by default. When a thread invokes one of these calls, it cannot advance its program counter until the operation completes, and it will typically be context-switched off the CPU while waiting.

A syscall like `read()` blocks when there's no data in the receive buffer.<br>
`accept()` blocks when there's no pending connection in the accept queue.<br>
The calling thread is blocked until data/connection becomes available. Reading from a file is always ready conceptually (data exists on disk), but the operation is still slow enough to be blocking.

## Why this is a problem?

If a single process/thread is responsible for servicing multiple connections in a loop, blocking on one connection means every other connection waits, even if their data already arrived. This kills concurrency.

## Two architectural strategies for async I/O:

Readiness-based: Ask the OS "is this file descriptor ready?" Only call the actual read/write once you know it won't block. Examples: `select`, `epoll` (Linux), `kqueue` (Mac).<br>
(NOTE: readiness-based approaches don't work well for file I/O, because files don't have a meaningful "not ready" state — they're "always ready" but slow.)

Completion-based: Submit the job to the OS and let the kernel perform the work; you get notified when it's done. Examples: Windows IOCP (I/O Completion Ports), Linux `io_uring`.

## How `select()` works?

Pass a list of file descriptors to `select()`<br>
The kernel blocks the thread, until atleast 1 FD is ready.<br>
On return, kernel does not tell you which FD is ready, you must loop through all FDs and call `FD_ISSET()` on each to find out.<br>
Once you find the ready FD(s), call the actual `read()`/`write()`/`accept()`.

Major drawback of `select()` is that, O(n) scan required every time to discover which FD(s) are ready, since the kernel doesn't tell you directly.

## How `epoll()` works? (Linux Standard)

Its a 3 step API.<br>
`epoll_create()` : it creates an epoll instance once.<br>
`epoll_ctl()` : Register interest in specific FDs.<br>
`epoll_wait()` : Put your worker/event-loop thread to sleep until something happens with registered FDs.

As packets arrive on monitored connections(FDs), the kernel asynchronously updates the internal data structure, marking which FDs have new events and returns only the FDs that are actually ready, not the full set.<br>
That is why epoll is better than select, No O(n) userspace scan — `epoll_wait()` returns an array containing only the ready FDs. This avoids the copy-everything-every-time problem of select.

## How `io_uring` works ?

`epoll` and `select` are readiness based. <br>
`io_uring` is completion based.<br>
we handover interested FDs to `epoll`, `epoll` tells us when any of these socket is ready for I/O, then our application performs syscall(`read`, `write`, `send`).<br>
epoll doesn't actually perform the `read()` or `write()`, backend application does, `epoll` only tells us when socket is ready.

`io_uring` combines these 2 operations.<br>
Instead of just telling these FDs are ready, Kernel also perform `read()`/`write()` operation for you.<br>
The application doesn't need to perform actual syscall.<br>
`io_uring` is built around shared memory between user space and kernel space.<br>
Two ring buffers(circular queue) live in shared memory: a Submission Queue (SQ) and a Completion Queue (CQ).<br>
Whatever operation process wants to perform, it creates an entry/job in submission queue (SQE).<br>
`READ` socket A, `ACCEPT`, `WRITE` socket B... Each entry has a unique identifier.<br>
Kernel consumes request from submission queue and if socket has data then perform operation.<br>
When operation completes, Kernel creates an entry in Completion queue(CQE).

why `epoll` is not suitable for file I/O, but `io_uring` works with files?
