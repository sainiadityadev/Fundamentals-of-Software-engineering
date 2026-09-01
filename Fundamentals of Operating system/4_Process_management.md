# Process Management

## Program to Process

A program on disk (e.g., an `ELF` binary) is static, it contains the code, data, and metadata (where the heap starts, stack size). <br>
When a program comes into the RAM it becomes a process. A process is the live, executing version of that program.

A process has four dedicated memory regions:<br>
Text (Code) Segment — the executable instructions<br>
Data Segment — global and static variables<br>
Heap — dynamically allocated memory (grows upward)<br>
Stack — function call frames, local variables, return addresses (grows downward)

To the process, this all looks like dedicated, private memory — this is enforced by virtual memory.

## Process Control Block (PCB)

The kernel needs metadata to manage processes. This metadata is stored in the Process Control Block (PCB), which lives in kernel space. The kernel cannot let a process modify its own PCB directly.<br>
PCB contains Process ID(PID), Process state, Program Counter(PC), Scheduling info, etc.

Registers are only flushed from CPU to the PCB on a context switch or kernel mode transition.

The kernel maintains a process table — a data structure that maps PID → pointer to PCB. This allows O(1) or near-O(1) lookup of any PCB.

# Threads

## Why threads were invented?

To do multiple things concurrently, naive approach is to spin up multiple processes. But this is expensive:<br>
Each new process requires a new virtual memory mapping (new page table)<br>
New stack, heap, data, and code segments must be allocated<br>
Creating a PCB and all associated metadata takes time<br>
The TLB must be flushed on context switch because page tables differ between processes<br>
IPC between processes requires explicit mechanisms (pipes, sockets, shared memory).

A lot of this overhead is unnecessary if two concurrent execution paths belong to the same program and need access to the same data. What if they could just share everything except what absolutely must differ?

### What absolutely must differ for two independent execution paths?

The Program Counter (PC) — each thread is executing a different instruction in the code<br>
The Stack — each thread calls different functions, needs its own call frames.

Everything else — code, data, heap, file descriptors, page table, IPC handles — can be shared.

This is exactly what a thread is: a lightweight process that shares almost everything with its parent process.

## What a Thread Shares (with the parent process)

Code segment — all threads execute the same binary<br>
Data segment — global variables are shared<br>
Heap — all threads allocate from and access the same heap<br>
Page table — same virtual-to-physical mapping (no TLB flush needed on thread switch within same process)<br>
PCB — file descriptors, IPC handles, shared memory mappings, open sockets.<br>
Open file descriptors — a socket opened by the parent is immediately accessible to all threads.

## What a Thread Owns Exclusively

Program Counter — its own execution pointer<br>
Stack — The thread's dedicated stack still lives inside the parent process's virtual memory.<br>
Register set — link register, general-purpose registers, etc.

When switching between processes, the entire TLB must be flushed because page table is different, but when you context-switch between two threads of the same process, the page table does not change. The TLB does not need to be flushed. This is a major performance win vs. switching between two processes.

## Thread Control Block(TCB)

Just as the kernel maintains a PCB per process, it maintains a Thread Control Block (TCB) per thread.<br>
TCB contains Thread ID(TID), Pointer to parent PCB, Program counter(PC), Stack pointer(SP), etc.

The kernel maintains a thread table mapping TID → pointer to TCB. Implemented as a hash table for fast lookup. All of this lives in kernel space.<br>
When a thread blocks (I/O wait, mutex wait), the kernel saves its register state into the TCB and schedules another thread

# Fork System call

`fork()` creates a new process by cloning the calling process.<br>
Child process gets new PCB, new stack, heap, data, but due to CoW, physical memory is not actually copied until writes happen.

If `fork()` would have duplicated all memory at fork time, it would be catastrophically slow.<br>
Instead Kernel uses Copy on Write(CoW)<br>
In CoW,<br>
Both Child and parent process shares the same page.<br>
Both process reads to the same physical memory.<br>
But when either of the process writes, only then Kernel copies the page to a new physical frame, and make the updating process's page table point to new frame.

# Context Switching

CPU has no concept of a process, it only knows how to fetch and execute instructions. <br>It reads program counter (PC), fetches the instruction at that address, executes it, increments the PC, and repeats. <br>It does not know or care whether those instructions belong to "process A" or "process B."<br>
That abstraction is provided by Kernel.

## What is a context?

Execution state of a process at a moment. It includes,
All CPU registers, Program Counter(PC), Stack Pointer(SP), PTBR, PCB.<br>
All this information is stored in PCB.<br>
Kernel does not update PCB on every instruction, that would be expensive, it updates only at time of context switch.

Page Table Base Register(PTBR): It holds the physical address of process's page table.

## Scenario: Switching from Process 100 to Process 200

### Step 1 — CPU is executing Process 100.

The PTBR points to Process 100's page table. The PC is at some instruction address (say, `0x4008`). All registers reflect Process 100's state. The PCB for Process 100 in memory still holds the old PC value, it has not been updated since the last switch. This is intentional and acceptable.

### Step 2 — The OS decides to switch.

The kernel (not the CPU) makes the decision to preempt Process 100 and give the CPU to Process 200.

### Step 3 — Save current context(Save current PCB)

Step 3 — Save current context(Save current PCB)<br>
The kernel writes all current CPU register values into Process 100's PCB in kernel memory. This is a write to memory: approximately 100 ns at minimum. 

### Step 4 - Update process status

Process 100 transitions from RUNNING → READY (it was preempted, not blocked). Process 200 transitions from READY → RUNNING.

### Step 5 — Flush the TLB.

Process 200 has a completely different page table. Any virtual address that Process 100 cached in the TLB is now meaningless or potentially dangerous (wrong mapping). The TLB must be invalidated.<br>
every new memory access for process 200 is a miss, must fetch it from RAM.

### Step 6 — Load new context(Load new PCB)

The kernel reads Process 200's PCB from memory and loads all register values into the CPU.

### Step 7 — CPU resumes, unaware anything changed.

The CPU just continues executing. It sees a new PC, a new PTBR, new registers. It does not know a switch happened. It just fetches the next instruction.

## Process vs. Thread Context Switching


Switching between thread A of process 1 and thread B of process 2 is just as expensive as switching between two processes. <br>Threads are only cheaper to switch when they belong to the same process.<br>
Different process means must save and restore PCB, flush the TLB.<br>
Threads of same process share same PCB, PTBR does not change, no need to flush TLB.

## When Does Context Switching Happen?

### 1. Scheduling Decisions

The scheduling algorithm determines which process runs next, on which CPU core, and for how long. Common algorithms are First-Come First-Served (FCFS), Shortest Job First (SJF), Round Robin.

### 2. I/O Blocking

When a process makes a blocking I/O call (disk read, network call, etc.), the OS transitions it to BLOCKED state and immediately context-switches to a process in a READY state. Disk I/O can take milliseconds — during which the CPU could execute millions of instructions. Blocking the CPU while waiting for I/O is unacceptable.

### 3. Preemptive Multitasking (Timer-based)

The kernel uses a hardware timer interrupt to forcibly preempt a running process after a fixed time quantum .This prevents any single process from monopolizing the CPU.<br>

If thread A is blocked on I/O, the kernel can schedule thread B of the same process, on the same core. Thread B then benefits from, No TLB flush (same page table), Warm L1/L2 caches (same memory working set), No PTBR change.

# Reliability & Fault Isolation

One reason OS use separate processes (despite expensive context switch) is isolation and security. Processes have separate address spaces, a crash in one process cannot corrupt another. <br>
Threads share address space, a bug in one thread can corrupt the entire process.

Postgresql uses one backend process per client connection, if one process crashes, the supervisor can clean it up while the server remains alive.

Browser like Chromium uses multi process architecture primarily for security. A renderer handling a malicious website is sandboxed; it should not directly access your files, other sites’ memory, etc. A renderer crash normally kills only its tab. Each renderer process itself also uses many threads.

Nginx: it uses a master process + multiple worker processes. Each worker is usually event-driven with `epoll`/`kqueue`.

# Concurrency

Concurrency is the art of splitting a program into multiple processes or threads so they can execute "at the same time," making better use of CPU resources

Parallelism — tasks are literally executing simultaneously on different physical cores.

Concurrency — tasks are interleaved so rapidly (nanoseconds apart) that they feel simultaneous, but may share a single CPU. Hyper-Threading is a hardware-level example: one physical core, two logical execution contexts (two sets of registers, two program counters), sharing one ALU.

## CPU bound vs IO bound

A CPU bound task always requires CPU, it computes continously.<br>
Kernel preempts it, so that other runnable thread gets fairness.<br>
A CPU bound task might do this:

```
compute → compute → compute → time slice ends → scheduler switches it → compute again.
```

Example, Encryption/Decryption TLS involves XOR-heavy symmetric ops, image/video processing, JSON processing, Protocol parsing like HTTP.

An IO bound task asks for an IO, it cannot make progress until the external source responds.<br>
Kernel immediately runs another runnable thread instead.<br>
An IO bound task might do this:

```
send database query → block waiting 20 ms → receive result → do 0.2 ms CPU work → block again.
```

Example, Database query, Network call, File read, socket read.

So key question is, If I gave this task an entire CPU core, would it stay busy computing?<br>
CPU bound -> YES.<br>
IO bound -> NO, it would soon wait.

With CPU-bound work, 1,000 active threads on an 8-core CPU just fight for the same 8 cores. More threads usually means more scheduling overhead, not more throughput.<br>
With I/O-bound work, 1,000 requests can be reasonable because most are sleeping while waiting for I/O; a small number are actually using CPU at any moment.

## Concurrency hazards

### Race condition

When two or more threads access a shared variable concurrently and at least one is writing, the result depends on the non-deterministic interleaving of operations.

```
let us say a = 1
T1 reads a → gets 1
T2 reads a → gets 1 (before T1 writes back)
T1 increments in register → 2, stores a = 2
T2 increments in register → 2, stores a = 2
Result: a = 2 instead of 3. One increment is lost.
```

The root cause: read-modify-write is not atomic at the hardware level

### Mutex lock
A binary lock. Only one thread can hold it at a time.

Thread must acquire the mutex before entering the critical section.<br>
If another thread already holds it, the requesting thread blocks (gets descheduled — the OS won't waste a core spinning on it).<br>
The holding thread releases the mutex after completing its work.<br>
The OS unblocks one waiting thread, which then acquires the mutex and proceeds.

### Semaphore

It is a counter based synchronization primitive. Any thread can signal it.<br>
`wait()` — decrement the counter. If counter is already 0, block until it becomes positive.<br>
`signal()` — increment the counter. Unblocks one waiting thread if any.
