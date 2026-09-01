# Kernel and Operating System

Kernel is a core software that manages the hardware.<br>
It handles CPU scheduling, memory management, Drivers, File system, Network stack.<br>
Operating system = Kernel + Tools like Shell, compiler, UI framework, etc.
OS makes the Kernel usable for end user and Developers.

## Kernel Resources and Responsibilities

Following are some resources which Kernel manages:<br>
CPU execute one instruction per cycle.<br>
When you compile a code, you compile it for a particular architecture(x86, ARM)
RAM, Hard disk(HDD, SSD), file system are handle by Kernel.<br>
The Kernel implements full network stack.
Data arrives on the wire(Ethernet, Fibre, etc), the signal is converted to bits.
NIC receives the Ethernet frames which contains the IP packet and the Kernel reads from the NIC buffer.<br>
Kernel is responsible for scheduling process on CPU cores.
If a process waits for an I/O(reading a file), the kernel suspends it and run something else.
From CPU perspective, waiting for disk I/O is millions of cycles, so there's no reason to sit idle.

## System Calls and Mode Switches

When a user process wants to do something that involves hardware or OS services(reading a file, allocate memory, send a packet), it issues a system call(`read()`, `write()`, etc)<br>
A system call triggers a mode switch, CPU transitions from user mode to Kernel mode.<br>
This switching involves:<br>
Saving all CPU registers(Current process state)<br>
Executing kernel function<br>
Restoring register and return to User mode. This is Expensive.<br>

## Programs and Processes

What a program is?<br>
Program is a compiled linked binary file stored on disk.<br>
It follows a specific executable format, Executable Linked Format(ELF) on linux and Portable Executable(PE) on windows.<br>
Only works for CPU architecture it was compiled for.

Process is program instance loaded in memory and actively executing.
Has its own stack memory, registers, file descriptor.

## Compilation

What compilation means?<br>
Translating high level source code(C, Java, Rust) into native machine code for a CPU architecture(x86, ARM).<br>
The output is one or more object files, compiled but not yet executable on their own.
Compilation translates source code into machine code, producing object files. If you compile 10 source files, you get 10 object files.<br>
Object files contain pure machine code but cannot run on their own. They lack the headers that specify where execution should start, where the heap is, where static data lives, etc.<br>
The linker takes all the object files needed for a program and merges them into a single executable.

## Linking

What linking means?<br>
Combines all object files and libraries into a single executable.

### Static linking

All the library code is bundled in the binary
Produces a large executable file, 
No dependency at runtime.

### Dynamic linking

The executable file only contains reference to the library. <br>
This library is `.so` on linux and `.dll` on windows.<br>
Produces small lightweight binary.<br>
Dependency at runtime.

Why this matters: If you copy just the executable to another machine and it uses dynamic linking, it may fail because the required shared libraries aren't present there. <br>This is exactly why Docker containers and statically linked binaries exist — to guarantee the entire dependency closure travels with the program.

## Process Control Block (PCB)


A Kernel data structure in memory that stores metadata about a process. Like a Process ID, Program counter, List of open file descriptor, etc.

## Program Counter (PC) / Instruction Pointer (IP)

Its a CPU register holds the memory address of next instruction to execute.

Why store Program Counter(PC) in CPU register, not memory?<br>
Program counter is updated millions of times every second.
Writing back to memory after every instruction is an overhead and expensive for CPU. <br>So PC lives in CPU register during execution.
It's only written back to PCB after the process is context switched.
So when OS context switch a process, PC is written to PCB of the process.

### CPU instruction cycle

CPU instruction cycle:
Fetch instruction from the memory address stored in PC.
Decode the fetched instruction.
Execute the instruction using ALU.
Increment program counter.

## Program Stack

Stack is Managed entirely by the compiler and CPU registers — no kernel involvement per allocation.<br>
Because of CPU registers, operations on stack are extremely fast.

Key CPU registers:<br>
Stack pointer(SP): Points to the top of the stack, moves on every allocation/deallocation<br>
Base pointer(BP): Points to the start of current function's stack frame.
Program counter(PC): Points to the currently executing instruction.

### Stack frame

When a function is called, a stack frame is created for it.<br>
The compiler knows at compile time exactly how much stack space the function needs (it can see all the local variables and their types). Stack space is computed statically.<br>
SP is decremented by the total bytes needed for the new frame.<br>

All local variables in a function are contiguous in memory. One cache line fetch often brings in multiple variables simultaneously.<br>
When data is laid out contiguously in memory, the CPU cache works maximally efficiently. A single cache line fetch (typically 64 bytes) pulls in multiple adjacent variables at once.<br>
Rule of thumb: things close to each other in memory = fast. Things scattered = slow.
And that is why Stack is Cache friendly.

Each function call pushes a new frame onto the stack. 
The chain of saved base pointers forms a linked list through the stack — this is what debuggers use to generate stack traces.

Stack allocation is much faster than heap allocation.<br>
Stack allocation is a single CPU instruction — decrement the stack pointer. No system call, no allocator, no lock.

## Data section

The data section holds global and static variables accessible across all functions within a process.<br>
The data section is fixed in size (determined at compile time), but the content is mutable. You can change the value of an integer global variable.<br>
The compiler can do a static scan and determine the exact size of the data section before the program ever runs. No runtime allocation is needed.

## Cache Line Mechanics

The important thing to understand is that the CPU never goes to RAM and asks for a single variable.<br>
It always fetches a block of memory called a cache line. A cache line is typically 64 bytes.<br>

Memory is always fetched in 64-byte cache lines. If you read one variable at address X, you automatically get the surrounding 63 bytes in L1 cache. Any variable whose address falls within that 64-byte window is now hot in cache.<br>

This cache line behaviour is why arrays are so fast and performant, because they store their elements contiguous, next to each other in memory.<br>
Let us say an array of integers(4 byte)<br>
Read `arr[0]`, CPU will load address from `arr[0]` to `arr[15]`, from RAM in L1 cache.<br>
So reading `arr[0]` is around 100 ns, but reading `arr[1]` or `arr[2]` till `arr[15]` is 1-2 ns.

## Shared Data Section: Concurrency and Cache Invalidation

Since all threads in a process share the same data section, any thread can read or write global/static variables. This creates two classes of problems:
1. Data race
Two threads writing to the same global variable without synchronization will corrupt each other's updates. You must use mutexes or atomic operations to protect shared mutable state.

2. Cache invalidation
CPUs have per core L1 and L2 cache. When Thread A (on Core 1) writes to a global variable, Core 2 must invalidate its cached copy of that cache line and re-fetch from memory the next time Thread B reads it.<br>
If threads are constantly writing to shared globals, you're continuously thrashing caches.<br>
Rule of thumb: Reads of shared globals are cheap (cache-friendly). Writes to shared globals are expensive (cache-invalidating) and dangerous (race conditions).

That's why global variables are discouraged in production code: Beyond readability, it's a performance hazard in multithreaded programs. Every write flushes cache lines across all cores that have read that variable, causing repeated DRAM accesses.

## Heap Memory


It is used for dynamic runtime memory allocation.<br>
Unlike stack, which is automatically managed by function execution, heap memory persist until explicitly freed.<br>
Heap memory is the foundation for all large, unpredictable data storage: dynamic arrays, linked lists, objects.

### Why heap memory exists?

Why heap memory exists?<br>
stack can only hold data whose size and lifetime is known at compile time and bounded to the current function scope.<br>
We want multiple function using same instance of object.
Anything that outlive the function that created it, allocated in a size determined at runtime, must go on Heap.

A pointer is a variable whose value is a memory address.<br>
Like any other variable, it must itself be stored somewhere in memory.
The pointer itself takes 4 bytes(32 bit system) or 8 byte(64 bit system). The data it points to is separate and can be any size.<br>

A pointer can live anywhere: stack, heap, or data segment. It can point to any region: stack, heap, or data.

malloc() is a system call, it triggers Kernel mode switch.<br>
Kernel allocates the memory, even if you ask for 4 bytes, minimum allocation unit from the OS is one page(4 KB)

The pointer lives on the stack. The data lives on the heap.<br>
Function allocates memory on heap, stores pointer as a local variable.
Function return, local pointer is destroyed(stack frame popped), but heap memory is still allocated to the process, no reference to it exists.<br>
The Kernel does not reclaim it because from its perspective the process still owns it.

A Memory Leak occurs when heap memory is allocated but never freed, and all pointers to it go out of scope.
Garbage collection and reference counting exist specifically to solve this.

### Dangling Pointers and Double free

These are more dangerous than leaks.<br>
When we reference to an Object that does not exist, it is dangling pointer.
It occurs when heap memory is freed, but pointer still holds the (now invalid) address.<br>
It result in unpredictable behaviour, the process might even crash.

Double free occurs when free() is called twice on the same pointer.

This is a primary motivation for languages like Rust, moving memory safety checks from runtime to compile time.<br>
Dangling pointer, Double free are impossible in Rust.

### Why Stack is faster?

No kernel involvement. Stack allocation is just decrementing the stack pointer — a single instruction. No system call, no mode switch, no header.<br>
Deallocation is free. When a function returns, the stack pointer moves back up. All local data is instantly "freed" in `O(1)`. No bookkeeping required.<br>
Cache locality. Local variables declared in the same function are contiguous in memory. When the CPU reads a variable, it loads a 64-byte cache line from RAM.

### Why the heap is slower?

Kernel mode switch. Every malloc() and free() is a mode switch — all registers must be saved, context loaded into kernel mode, and restored afterward.<br>
Fragmentation and random access. Heap allocations are scattered in memory. Reading two heap objects likely results in two separate cache line loads. Random access patterns destroy cache efficiency.

## Escape Analysis (Go, JVM)

Some compilers are smart enough to detect that a heap allocation is unnecessary.<br>
Escape Analysis scans the code at compile time. If a variable allocated with malloc/new never escapes the function that created it (i.e., its pointer is never passed to another function or stored globally), the compiler can allocate it on the stack instead.
The pointer and the value both live on the stack, and both are destroyed when the function returns.

Result: all the performance benefits of stack allocation, with the ergonomics of heap-allocation syntax. 

If the pointer does escape (gets passed to another function, stored in a global, or returned), escape analysis backs off and uses the heap.
