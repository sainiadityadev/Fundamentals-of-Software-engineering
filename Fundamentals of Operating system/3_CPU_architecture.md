# CPU Execution Cycle

CPU works in Fetch -> Decode -> Execute -> Write back

## ALU

The ALU executes the current instruction.<br>
At the hardware level, everything reduces to logic gates. The ALU does not understand high-level "add" instructions, the CU decodes the machine instruction and tells the ALU which gate operations to perform.

## Control Unit (CU)

The CU decodes the next instruction.<br>
Decoding machine code into operations the ALU or MMU understands<br>
Dispatching to the right component: math goes to ALU, memory reads go to MMU.

## MMU

The MMU fetches the instruction after that.<br>
MMU handles all memory operations for the CPU core.<br>
Virtual-to-physical address translation via the TLB.<br>
Cache lookups (`L1` → `L2` → `L3` → main memory)

## Registers

Registers are the fastest storage in the entire computing hierarchy, Modern CPUs use 64-bit registers(8 bytes per register).<br>
Types of Registers are:<br>
Program Counter (PC): points to the next instruction to fetch.<br>
Instruction Register (IR): holds the currently executing instruction<br>
Stack Pointer (SP) and Base Pointer (BP): track the call stack<br>
General-Purpose Registers: temporary storage for computation.

## Cache Hierarchy (L1, L2, L3)

Accessing main memory costs ~100 nanoseconds. The cache hierarchy exists to reduce how often you pay that cost.<br>
L1-I(Instruction), L1-D(Data), L2 cache exists per core.<br>
L3 is shared across all cores.

L1 is split into instruction and data caches primarily so the CPU can fetch instructions and access data simultaneously.

## Single Processor, Multiple Cores


Each core gets dedicated:<br>
ALU<br>
Control Unit<br>
Registers<br>
L1-I and L1-D caches<br>
L2 cache (in many architectures)

Shared between cores on the same processor:<br>
L3 cache (most common shared level)<br>
Access to the system bus and DRAM

## Hyperthreading or SMT

Hyperthreading presents one physical core as two logical cores to the OS. What it actually does: duplicate only the registers (the per-thread state), but share everything else — ALU, MMU, caches.<br>
A context switch between processes primarily means swapping out registers: the program counter, stack pointer, base pointer, and general-purpose registers. The core's actual execution units (ALU, FPU, cache) stay the same.<br>
So Intel's insight was: add a second set of registers per core (essentially just memory, cheap to add), and let two threads run "simultaneously" on the same physical core, each with their own register state, but sharing L1-I and L1-D cache, L2 cache, ALU, CU.<br>
When one thread cannot use certain resources, for example, while waiting for data from memory—the other thread can use some of the available capacity. Both threads may also execute during the same CPU cycle using different execution units.<br>
This improves total throughput, but it does not double performance because the logical threads still compete for shared resources.

Hyper-threading is not always a win. The two logical threads share the L1 and L2 caches.<br>
in some production environments, engineers have disabled hyper-threading because it introduced unpredictable latency spikes — fast then slow then fast — caused by cache contention between co-scheduled threads.<br>
When it works well: when both logical threads belong to the same application, they share cache lines and get high L1 hit rates.<br>
When it hurts: when two completely unrelated processes are co-scheduled on the same physical core, they fight over limited cache space.

## CISC Architecture

One instruction can perform multiple operations: load from memory, compute, store back.<br>
`x86`, `AMD64`.

## RISC Architecture

One instruction can perform only one operation.<br>
`ARM`, `Apple M1`, Mobile CPUs.

## CPU Clock Cycles

A CPU running at 3 GHz executes 3 billion clock cycles per second. This does not mean 3 billion instructions per second.<br>
one instruction may take many cycles.<br>
At 1 GHz, 1 clock cycle ≈ 1 ns. A RAM access at 100 ns wastes ~100 cycles. A disk access at 5 ms wastes millions of cycles. The CPU could have executed hundreds or thousands of instructions in that time.

This is why the OS switches processes during I/O — no point leaving the CPU idle.

## Pipelining

Originally, CPUs executed one instruction completely before starting the next: fetch → decode → execute → write → then fetch the next. This is inefficient because each hardware unit sits idle while another is working.<br>
Solution is pipelining parallelism.<br>
Since each stage uses a different hardware component, we can overlap them:

```
Clock Cycle:   1       2       3       4       5
Instruction 1: Fetch   Decode  Execute Write
Instruction 2:         Fetch   Decode  Execute  Write
Instruction 3:                 Fetch   Decode   Execute ...
```

While instruction 1 is being decoded, instruction 2 is already being fetched. The CPU is never idle between stages.

## Branch Prediction

But Pipelining creates a problem: when the CPU hits a conditional branch (`if`/`else`, loops), it doesn't know which instruction to fetch next. Rather than waiting, CPU predicts which branch will be taken.

Now consider the following code:

```
if (age >= 18) {
    allowEntry();
} else {
    denyEntry();
}
```

The CPU reaches the if, but the comparison result may not be ready yet.<br>
It has two choices:<br>
1. Stop and wait.<br>
2. Guess which path will be taken and continue working.<br>
The CPU usually chooses to guess. This is called Branch prediction.

When the prediction is correct(age >= 18), the CPU keeps the work it has already performed. Very little time is lost.<br>
When the prediction is incorrect(age is 13), The CPU speculatively starts executing `allowEntry()`, It discards those speculative results and clears the incorrect instructions from the pipeline.<br>
This is called a pipeline flush, and it costs CPU cycles.<br>
Branch prediction is normally very accurate because programs frequently follow patterns.

The CPU guesses and temporarily performs future work. If the guess is wrong, it removes the official results—but some invisible traces, such as cache changes, may remain. Spectre uses those traces to infer secret data.<br>
Branch Prediction is one of the reasons for Spectre Vulnerability(2018).

## CPU Utilization and Parallelism

CPUs are extraordinarily fast. Most of the time, a CPU is sitting idle, waiting on memory fetches, cache misses, or disk reads. To combat this waste, CPU architects developed a set of techniques to keep the processor busy every single cycle.<br>
pipelining, parallelism, hyperthreading, and SIMD.

Parallelism is programmer's responsibility. Unlike pipelining, the CPU doesn't do this for you.<br>
Programmer must Split work across cores/threads.

## Single Instruction Multiple Data (SIMD)

Single Instruction Multiple Data (SIMD)<br>
Traditional arithmetic in CPU is scalar: one instruction operates on one pair of values.<br>
To add four pairs of numbers (`a1+b1`, `a2+b2`, `a3+b3`, `a4+b4`), you need four separate add instructions.

SIMD extends the ALU to operate on vectors, packed arrays of values in registers, instead of single scalar values. A single SIMD instruction can load a vector of, say, four 32-bit integers into a 128-bit register (4 × 32 = 128 bits), and a second vector into another 128-bit register, and then add all four pairs in a single instruction.
