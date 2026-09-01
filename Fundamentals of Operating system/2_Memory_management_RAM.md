# Memory Management

Volatile memory requires constant voltage to retain data.<br>

## Types of RAM

### SRAM (Static RAM)

Structure: 1 bit = 1 flip flop = 6 transistors<br>
Speed is fast, no refresh needed.<br>
Expensive to manufacture, 48 transistors to represent 1 byte.<br>
Used in CPU `L1`/`L2`/`L3` cache.

### DRAM (Dynamic RAM)

Structure: 1 bit = 1 transistor + 1 capacitor<br>
Speed is slower, because of refresh overhead<br>
Much cheaper to manufacture.<br>
General Used RAM.

## How capacitor stores bit?

A capacitor stores electrical charge. If charge is above a threshold → bit is 1. Below threshold → bit is 0. <br>But capacitors discharge naturally over time (milliseconds), So the 0 or 1 stored in it can eventually become incorrect.<br>
So in every few milliseconds, every capacitor must be refreshed.<br>
-> Read the capacitor, this drains it completely.<br>
-> Buffer the value in sense amplifier.<br>
-> Write value back to capacitor with full charge.<br>
Who triggers refresh? DRAM controller handles this.

## Sense amplifiers

They are the shared buffers between the capacitor array and the outside world. <br>They are not capacitor-based — they hold values stably.<br>
When you read a capacitor, its value is drained into the sense amplifier.<br>
The sense amplifier can then be read by the CPU or written back to the capacitor.

## RAM Structure

A RAM stick is called a DIMM (Dual Inline Memory Module)<br>
The DIMM contains multiple DRAM chips.<br>
Inside each DRAM we have multiple Banks. A bank is rectangular array of memory cells.<br>
Every intersection of a row and column represents a DRAM cell.

The OS and memory controller parse bits to determine:<br>
Which bank the address lives in<br>
Which row within that bank<br>
Which column within that row

Each bank has one shared set of sense amplifiers(DRAM row buffer)<br>
We can only have one open (active) row per bank at a time.<br>
Opening a row = draining all capacitors in that row into the sense amplifiers(DRAM row buffer)<br>
If you need to open a different row in the same bank, you must first close the current row (write all sense amplifier values back to capacitors), then open the new row.

This row open/close cycle is the most expensive operation in DRAM access. It costs on the order of 100 nanoseconds.

Each bank has its own sense amplifiers, You can have an active row open in bank 0 and a different active row open in bank 1 simultaneously. This lets the memory controller work with multiple banks in parallel, improving performance.

## DRAM Evolution

Asynchronous DRAM: CPU and RAM operate on independent clocks, out of phase.<br>
Synchronous DRAM(SDRAM): CPU and RAM clocks are synchronized.

### DDR SDRAM (Double Data Rate)

SDRAM only transferred data on the rising edge of the clock. DDR transfers on both rising and falling edges. If original SDRAM sent 1 bit/cycle, DDR sends 2 bits/cycle.

There are multiple DDR generations, like DDR1, DDR2, DDR3, DDR4, DDR5.<br>
The CPU never reads a single byte from the RAM, CPU always loads 64 bytes at a time from RAM. This is called one cache line.

## Burst Reads

The CPU does not read a single byte or a single word from RAM in isolation. When you request any address, the memory controller opens the entire DRAM row and returns a burst of 64 bytes of sequential data. This is a hardware-level behavior — you asked for one thing, you get 64 bytes whether you like it or not. This is fundamental to how cache lines work.

When DDR4 reads data, it sends 64 bytes at a time and CPU needs 64 bytes at a time, so it sends enough data to fill exactly one cache line.<br>
DDR5 able to read twice as much data in one read, 64 x 2 = 128 bytes at a time, but CPU needs 64 bytes,<br>
So DDR5 uses two channels to send data, 64 bytes each, this way 2 CPU cores can now access memory simultaneously — one per channel. 

For a single sequential 64-byte read, DDR4 and DDR5 perform identically.<br>
DDR5 has advantage under concurrent access from multiple cores. In DDR4, if one core is reading a bank, another core requesting a nearby address is blocked for ~100ns. <br>DDR5 splits each DIMM into two independent channels, allowing parallel requests to be served.

## Spatial Locality and the Cache Line

When RAM opens a row, it is costly(draining all capacitors, filling sense amplifiers). So instead of fetching only the one value the CPU asked for, hardware fetches a full 64-byte cache line containing nearby data.<br>
This is why sequential memory access is faster than random memory access: after one element is loaded, nearby elements are usually already available in cache. Arrays and flat buffers benefit from this.<br>
Accessing memory in random patterns (pointer chasing, scattered struct fields, hash table lookups) forces repeated row open/close cycles. <br>That's why cache-friendly data structures (arrays, flat buffers) dramatically outperform pointer-heavy ones (linked lists, trees with heap-allocated nodes) even at the same algorithmic complexity.<br>
One detail: programs use virtual addresses, while RAM uses physical addresses. Addresses that look adjacent to your program are usually adjacent within a page, but may not remain adjacent across page boundaries. True spatial locality requires physical adjacency, not just virtual adjacency.

# Cache Hierarchy in CPU

L1-I cache(Instruction cache), L1-D Cache(Data cache)<br>
L2 cache, L3 cache.

## Read Path: Instruction Fetch

When the CPU wants to read an instruction from memory, how does the data travel from RAM to the CPU?<br>
Read Path: Instruction Fetch<br>
Consider a simple program executing the instruction at address 640 (virtual address, translated to physical by the MMU and page tables)<br>
1. The program counter (PC) points to address 640.<br>
2. The kernel asks the CPU to load the instruction at 640.<br>
3. The CPU sends the request to RAM.<br>
4. RAM opens the corresponding DRAM row, drains the capacitors into the sense amplifiers, and returns a 64-byte burst back to the CPU.<br>
5. This 64-byte cache line is loaded into the `L1 I-Cache`.<br>
Cost: ~100 nanoseconds.

If that next instruction falls within the same 64-byte cache line (which it almost certainly does for sequential code), the fetch costs only ~1 nanosecond — it's already in the L1 cache.

## Write Path

When the CPU wants to write a value to a memory address, how does that value get stored in the DRAM cells?<br>
Write Path<br>
To write a value (e.g., write 22 to physical address 1024):<br>
1. The CPU issues a write command to the memory controller.<br>
2. The DRAM row containing address 1024 is opened — the entire row is drained into the sense amplifiers.<br>
3. The CPU modifies the specific bytes in the sense amplifiers.<br>
4. The sense amplifiers write the entire modified row back to the capacitors.

CPU never write directly to DRAM capacitors. The sense amplifiers are the intermediary.

# Data alignment

CPU require that memory address of the data, must be a multiple of the size of that data type.<br>
If `int` is 4 bytes, so it gets stored at address multiple of 4.<br>
If `char` is 1 byte, so it gets stored at address multiple of 1.<br>
If `double` is 8 bytes, so it gets stored at address multiple of 8.

```
struct foo {
    char c;      // 1 byte
    double d;    // 8 bytes
    short s;     // 2 bytes
    int i;       // 4 bytes
};
```

## Memory layout (wasteful)

Address 0: c (1 byte)<br>
Addresses 1–7: padding (wasted — double must start at multiple of 8)<br>
Addresses 8–15: d (8 bytes)<br>
Addresses 16–17: s (2 bytes)<br>
Addresses 18–19: padding (int must start at multiple of 4)<br>
Addresses 20–23: i (4 bytes)

Total: 24 bytes for what is logically 15 bytes of data. 9 bytes wasted.

## Optimized layout (reorder members by size, largest first)

```
struct foo {
    double d;    // 8 bytes at address 0
    int i;       // 4 bytes at address 8
    short s;     // 2 bytes at address 12
    char c;      // 1 byte at address 14
};
```

This packs into 16 bytes with minimal padding.

This is not just academic. Google optimized the Linux kernel's TCP/IP stack by reordering fields in the IP header struct. The result was that more useful data fit within a single 64-byte cache line read.

# Why virtual memory exists?

Memory allocated to a process must be contiguous, as process loads and unload gaps form. This is  Fragmentation.<br>
External fragmentation: Free memory exists, but it is not contiguous.<br>
Memory Layout:

```
+------+----+------+----+------+
| Used |Free| Used |Free| Used |
+------+----+------+----+------+
        2KB         3KB
```

Total Free Memory = 5 KB<br>
If a process needs 4 KB contiguous memory, it cannot be allocated because the free space is scattered into 2 KB and 3 KB chunks.

Internal Fragmentation: OS allocates memory in fixed size chunks called pages(1 page = 4KB).<br>
If a process needs 4097 bytes, the OS allocates two 4KB pages — wasting 4095 bytes in the second page. OS doesn't know how much of the allocated block the app actually uses.

Security: Without virtual memory, you'd need to coordinate exact physical addresses between processes to share data. With virtual memory, since a process cannot directly access physical memory, it cannot modify other process data.

Not enough memory: If a program needs more memory than available RAM, it simply can't run. Virtual memory solves this via swapping.

## Pages and Frames

OS divides virtual address space into fixed-size chunks called pages(1 page = 4KB).<br>
Physical memory is divided into same-sized chunks called frames.<br>
A virtual page maps to a physical frame.

Each process has its own page table, which is stored in RAM.<br>
Page table is a data structure that maps virtual page numbers to physical frame numbers.<br>
Because page tables live in RAM, every memory access requires a memory read just to translate the address — that's expensive.

## Translation Lookaside buffer(TLB)

TLB is a hardware cache (inside the CPU) of recently-used page table entries. It caches the virtual → physical mapping so you don't need a memory read for every translation.<br>
TLB hit: translation is instant.<br>
TLB miss: must read page table from memory (costly)

On a process context switch, the TLB must be flushed, because virtual address 1000 in process A maps somewhere completely different than virtual address 1000 in process B<br>
Thread context switches within the same process do NOT require a TLB flush, because they share same virtual address mapping.

### Address translation flow

Process accesses virtual address X<br>
CPU sends X to the MMU<br>
MMU checks TLB — if hit, returns physical address directly<br>
If TLB miss, MMU reads the process's page table from physical memory<br>
Finds the frame number, computes physical address<br>
Loads translation into TLB<br>
Memory access proceeds at the physical address

## Shared Memory via Virtual Memory

Multiple virtual pages (from different processes) can map to the same physical frame.<br>
-> Same program, multiple instances: If you spawn 5 copies of a program, some portion of the memory is shared. (What happens if you run a same file multiple times?)<br>
-> IPC shared memory: Processes explicitly share a region by mapping their virtual pages to the same frames.<br>
-> Forked processes: On `fork()`, parent and child share all physical frames initially (zero copy). Reads are free. Writes trigger copy-on-write.

## Copy on Write (CoW)

When a process `forks()`, Child process inherits all parent's virtual → physical mappings<br>
No data is actually copied<br>
Child and parent can read from the same page, but if any of the child or parent process writes to a shared page, the OS copies that page to a new physical frame and updates the writing process's page table entry to point to the new frame.<br>
Real world example: Redis persistence: To persist data without halting writes, Redis forks the main process.

## Swapping(Paging to Disk)

When physical memory is exhausted:<br>
The OS picks pages that haven't been accessed recently(tracked via access bits — itself expensive)<br>
Writes those pages to disk.<br>
Marks the page table entries as "swapped" and stores the disk location instead of a frame number<br>
Frees up those physical frames for new allocations.

When a process access "swapped" out page, MMU sees no valid frame and triggers a Page fault.<br>
Page fault causes a kernel mode switch (user → kernel).<br>
No free frame is available, kernel may reclaim a frame currently used by another process.<br>
Updates the page table entry and resume the process.<br>
Page fault causes slowdown, this is Thrashing.

# Direct Memory Access (DMA)

Normally, any I/O operation goes through the CPU. Peripheral triggers an interrupt, CPU switches to kernel mode, executes the Interrupt Service Routine (ISR), ISR reads data from the peripheral's buffer into CPU cache, CPU writes data to destination memory address.<br>
CPU acts as a bridge between Peripheral and RAM.<br>
For small operations like keyboard or mouse input, this is fine. But for large transfers network packets arriving at a NIC, reading a big file from disk, routing everything through the CPU is wasteful because CPU isn't doing any computation on the data. It's just moving bytes.<br>
DMA Controller is a hardware that allows peripherals (NIC, disk, USB controllers) to transfer data directly to and from RAM without routing that data through the CPU.

Kernel is responsible for initializing DMA operations and allocate physical address(not virtual address) to the DMA controller.<br>
Kernel then signals the peripheral "here's the address, Go".<br>
DMA controller performs the transfer directly between peripheral and RAM.<br>
When complete, DMA controller interrupts the CPU to signal completion.

Every peripheral works on interrupt.<br>
Keyboard writes the keystroke to its internal buffer, and issues an interrupt to CPU.<br>
CPU pauses current process, switches to kernel mode and run the ISR, a Kernel function.<br>
Mouse input has highest input priority, When a CPU-hungry process starves the interrupt handler, you see mouse lag and jitter. That's the interrupt not executing in time.

DMA only makes sense for large transfer, like when NIC receiving bulk network data, disk reads/writes of large files, The initialization cost (buffer allocation, address) is paid once, and you transfer MBs or GBs with no CPU involvement.<br>
DMA does not make sense for Keyboard, mouse, single-byte events.

NICs use DMA to write incoming packets directly to kernel ring buffers in RAM.<br>
but DMA only moves the network packet between the NIC and RAM. The CPU still performs most of the meaningful processing, like processing network protocol, TLS handshake, HTTPS decryption, parsing http request
