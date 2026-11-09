# COE 502 — Lecture 6: Multiprocessor Systems (Thread-Level Parallelism)

**Parallel Processing Architectures** · KFUPM · Dr. Abdulaziz Tabbakh · slides: `06-TLP.pdf`

> **How to read these notes**
> - Sections follow the slides in order. Each one opens with **The idea**. For a 5-minute refresh, read only those.
> - 💡 = analogy or extra help (not on the slides). ⚠️ = exam trap, or a spot where the slide's wording is confusing.

---

## The whole lecture in 30 seconds

Lecture 5 squeezed more speed out of **one** instruction stream. This lecture puts **several processors** in one machine, each running its own thread, all sharing one memory.

The programmer is told memory is **one flat space**. The hardware underneath is not flat: some memory is close, some is far, and every processor keeps its own **cached copies**. That gap causes everything in this deck:

| The hardware does this… | …so this matters | Section |
|---|---|---|
| Memory sits in different places | Some accesses are slower (**NUMA**), so where data lives matters | 2, 3, 9 |
| Processors are joined by a network | Bus vs point-to-point decides latency, bandwidth and scale | 6–8 |
| Each processor keeps private cached copies | Copies can disagree → **cache coherence** | 4, 11 |
| Writes can become visible in a different order | → **memory consistency** | 11 |
| Several threads update shared data | → hardware **synchronization** (atomic operations) | 11 |

---

## Contents

- [Key words](#key-words)

1. [What thread-level parallelism is](#1-what-thread-level-parallelism-is)
2. [Two types: SMP and DSM](#2-two-types-smp-and-dsm)
3. [Shared-memory physical architecture](#3-shared-memory-physical-architecture)
4. [Four multiprocessor organizations](#4-four-multiprocessor-organizations)
5. [Who makes the memory shared?](#5-who-makes-the-memory-shared)
6. [Shared vs point-to-point networks](#6-shared-vs-point-to-point-networks)
7. [Three real interconnects](#7-three-real-interconnects)
8. [Organizing point-to-point networks](#8-organizing-point-to-point-networks)
9. [UMA and NUMA](#9-uma-and-numa)
10. [Why caches matter even more in a multiprocessor](#10-why-caches-matter-even-more-in-a-multiprocessor)
11. [The two big issues](#11-the-two-big-issues)
12. [Summary sheet](#12-summary-sheet)
13. [Self-test](#13-self-test)

---

## Key words

| Word | Meaning |
|---|---|
| **Thread** | One independent stream of instructions, with its own program counter |
| **Program counter (PC)** | The register holding the address of the next instruction to run |
| **Core / processor** | Hardware that runs one thread at a time |
| **Socket** | One physical processor chip on the motherboard (it can contain many cores) |
| **Node** | One unit of a bigger machine: processor(s) + memory, sometimes + I/O |
| **Interconnect / network** | The wiring that lets processors and memories talk to each other |
| **Cache line** | The unit a cache stores and moves, typically 64 bytes |
| **Latency** | How long one access takes |
| **Bandwidth** | How much data can move per second in total |
| **Hop** | One link a message crosses on its way to its destination |

---

## 1. What thread-level parallelism is

**The idea:** run **several independent instruction streams at once**, one per processor. Each stream has its own program counter. To keep *n* processors busy you need *n* threads.

> **Thread-level parallelism**
> - Have **multiple program counters**
> - Uses **MIMD** model
> - Targeted for **tightly-coupled shared-memory multiprocessors**
>
> For *n* processors, need *n* threads
>
> Amount of computation assigned to each thread = **grain size**
> - Threads can be used for data-level parallelism, but the overheads may outweigh the benefit

**The three bullets in plain words:**
- **Multiple program counters:** one PC = one instruction stream. ILP (Lecture 5) and vector/DLP work inside **one** stream. TLP means **several** streams, each at its own place in its own code.
- **MIMD** (Flynn, Lecture 1): **M**ultiple **I**nstruction, **M**ultiple **D**ata. Each processor can run different instructions on different data.
- **Tightly coupled shared memory:** the processors sit close together and share one memory. A loosely coupled cluster, which talks by sending messages, is the opposite.

> 💡 **ILP vs TLP in a kitchen.** ILP is one cook doing a few independent steps of the same recipe at once. TLP is several cooks, each following their own recipe card (their own PC), all using the same pantry (shared memory).

**"For n processors, need n threads":** hardware parallelism is useless without software parallelism. A 64-core machine running a single-threaded program uses 1 core, and the other 63 sit idle. That's why performance became the programmer's job once clock speeds stopped rising (Lecture 1).

### Grain size

**Grain size = how much work each thread gets.** It's the same idea as *granularity* in Lecture 4.

| Grain | Work per thread | Good | Bad |
|---|---|---|---|
| **Coarse** | a lot | overhead barely matters | fewer threads to spread across processors |
| **Fine** | a little | lots of parallelism available | overhead can dominate |

> ⚠️ **"Overheads may outweigh the benefit."** Creating, scheduling and synchronizing threads costs time. Split an array of 1,000 numbers across 100 threads and each one adds only 10 numbers, which is far less work than it takes to start and coordinate the thread.
>
> 💡 Asking 10 friends to help fold 10 shirts: explaining the job takes longer than folding them yourself. Same lesson as the overhead curve in Lecture 3 and the optimal-*P* formula in Lecture 4.

---

## 2. Two types: SMP and DSM

**The idea:** both give the programmer **one shared memory**. The difference is **where the memory physically is**. In SMP it's one shared pool, equally far from everyone. In DSM it's split up, with a piece in each node, so some memory is near and some is far.

> **Symmetric multiprocessors (SMP)**
> - Small number of cores
> - Share single memory with uniform memory latency
>
> **Distributed shared memory (DSM)**
> - Memory distributed among processors
> - Non-uniform memory access/latency (NUMA)
> - Processors connected via direct (switched) and non-direct (multi-hop) interconnection networks

**SMP: one shared memory**
```
 ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐
 │  P  │ │  P  │ │  P  │ │  P  │
 │cache│ │cache│ │cache│ │cache│
 └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘
    └───────┴───┬───┴───────┘
         ┌──────┴──────┐
         │ shared cache│
         └──────┬──────┘
         ┌──────┴──────┐
         │ main memory │
         └─────────────┘
```

**DSM: a piece of memory in every node** (each node can itself be a multicore chip)
```
 ┌───────┐   ┌───────┐   ┌───────┐   ┌───────┐
 │   P   │   │   P   │   │   P   │   │   P   │
 │ cache │   │ cache │   │ cache │   │ cache │
 ├───┬───┤   ├───┬───┤   ├───┬───┤   ├───┬───┤
 │Mem│I/O│   │Mem│I/O│   │Mem│I/O│   │Mem│I/O│
 └───┴┬──┘   └───┴┬──┘   └───┴┬──┘   └───┴┬──┘
      └───────────┴─────┬─────┴───────────┘
           ┌────────────┴────────────┐
           │ interconnection network │
           └─────────────────────────┘
```

| | **SMP** | **DSM** |
|---|---|---|
| Memory | one shared pool | split across the nodes |
| Access time | **the same for everyone** (UMA) | **local fast, remote slow** (NUMA) |
| Scale | small (a few to tens of cores) | large |
| Interconnect | usually a bus | switched / multi-hop networks |

> 💡 **SMP** = one library in the town centre: everyone walks the same distance, but it gets crowded. **DSM** = every house has its own bookshelf, and you can borrow from a neighbour's. Your own shelf is instant; a neighbour's takes a walk.

**Why SMP doesn't scale:** every processor goes through the **same** interconnect to the **same** memory. Add processors and each one gets a smaller share of that fixed bandwidth. Past a few tens of cores, adding more buys nothing. DSM fixes this by giving each node its own memory. The price: **where your data lives now affects speed.**

> ⚠️ **Two meanings of "direct".** Here (H&P wording), "direct (switched)" vs "non-direct (multi-hop)" describes how far apart nodes are. In §8 (slide 12), **direct** means *the router is built into the node* and **indirect** means *separate router chips*. Answer using the meaning of the slide the question is about.

---

## 3. Shared-memory physical architecture

**The idea:** the programming model says memory is **flat** (every access looks the same). Real hardware is a **hierarchy**: within a chip, within a machine, across machines. Each level has its own processors, memories and network, and each level further away is slower.

> - Almost always there is a **hierarchical structure**
>   - Within socket, within a machine, across machines
>   - **Contrary to the flat view of shared-memory programming models**
> - At each level of hierarchy, there are
>   - multiple **processing elements** (cores, sockets, boxes, …)
>   - multiple **memory elements** (caches, DRAM modules, …)
>   - an **interconnection network** connecting them together

> 💡 Your desk drawer, the cabinet in your office, the archive in the basement, a warehouse across town. To your manager it's all just "the company files", but how long it takes to fetch one depends a lot on where it is.

**Why it matters:** OpenMP code that *runs correctly* can still be **slow** if its data sits far away. The flat model hides the distance; it doesn't remove it. (Lecture 4: the model and the hardware can differ, but for speed they should match.)

### Two ways to connect processors and memory (at any level)

**Decoupled:** memory is on the **far side** of the network. Every access crosses it, so everyone waits the same time.
```
 ┌───────┐  ┌───────┐  ┌───────┐  ┌───────┐
 │   P   │  │   P   │  │   P   │  │   P   │
 └───┬───┘  └───┬───┘  └───┬───┘  └───┬───┘
 ┌───┴──────────┴──────────┴──────────┴───┐
 │              interconnect              │
 └───┬──────────┬──────────┬──────────┬───┘
 ┌───┴───┐  ┌───┴───┐  ┌───┴───┐  ┌───┴───┐
 │  Mem  │  │  Mem  │  │  Mem  │  │  Mem  │
 └───────┘  └───────┘  └───────┘  └───────┘
```

**Coupled:** each processor has **its own memory** next to it, plus a router (**R**). Local access skips the network; remote access crosses it.
```
 ┌───────┐  ┌───────┐  ┌───────┐  ┌───────┐
 │   P   │  │   P   │  │   P   │  │   P   │
 ├───┬───┤  ├───┬───┤  ├───┬───┤  ├───┬───┤
 │Mem│ R │  │Mem│ R │  │Mem│ R │  │Mem│ R │
 └───┴─┬─┘  └───┴─┬─┘  └───┴─┬─┘  └───┴─┬─┘
       │          │          │          │
 ┌─────┴──────────┴──────────┴──────────┴─┐
 │              interconnect              │
 └────────────────────────────────────────┘
```

| | **Decoupled** | **Coupled** |
|---|---|---|
| Memory sits | across the network | next to the processor |
| Access time | the same for all | local fast, remote slow |
| Gives you | **UMA** | **NUMA** |

These two shapes come back again and again: SMP/DSM (§2), indirect/direct networks (§8), UMA/NUMA (§9).

---

## 4. Four multiprocessor organizations

**The idea:** four ways to arrange processors, caches and memory, from impractical to what every modern chip uses. The key moment is **(c)**: giving each processor a **private cache** makes it fast, but creates the **cache coherence** problem.

```
 (a) Dance-hall       (b) Shared L1        (c) Private caches   (d) Private L1 + shared L2
 P   P   P   P        P   P   P   P        P   P   P   P        P   P   P   P
 │   │   │   │        │   │   │   │        $   $   $   $        L1  L1  L1  L1
 └───┴─┬─┴───┘        └───┴─┬─┴───┘        └───┴─┬─┴───┘        └───┴─┬─┴───┘
  interconnect          shared L1           interconnect          shared L2
 ┌───┬─┴─┬───┐              │                    │                    │
 M   M   M   M            memory               memory               memory
```
($ = a private cache)

| | Organization | What the slide says | In plain words |
|---|---|---|---|
| **(a)** | **"Dance-hall"** SMP | All memory on the other side of the interconnect; same access time for all processors. **Not practical; no cache** | Every single access crosses the network, which jams immediately. It's only here as the baseline. |
| **(b)** | SMP with **shared L1** | **No replication**, but **higher hit latency** than (c) | One cache for everyone, so there's only ever one copy (no coherence problem). But a cache shared by many must be bigger and further away, and requests queue up. |
| **(c)** | SMP with **private caches** | — | Each processor has a small, close cache, so hits are fast. ⚠️ But the **same data can now be in several caches** at once. |
| **(d)** | Private caches + **shared L2** | **Popular for CMPs** | Private L1s for speed, plus one big shared L2 (or L3) for capacity and as a common meeting point. This is what multicore chips look like. **CMP** (chip multiprocessor) is the formal name for a multicore chip. |

### Why (c) creates the coherence problem

| Step | P1's cache | P2's cache | Memory |
|---|---|---|---|
| P1 reads X | X = 5 | — | 5 |
| P2 reads X | X = 5 | X = 5 | 5 |
| P1 writes X = 9 | X = **9** | X = 5 | 5 *(not updated yet, write-back)* |
| P2 reads X | 9 | **5 ❌ stale** | 5 |

> 💡 Everyone gets a photocopy of the shared plan. One person edits their copy. The others keep working from the old version and don't know it.

> 📌 The industry chose (d) over (b): fast private caches, in exchange for having to keep copies in agreement. **Coherence isn't an accident; it's the price of private caches.** Topic 7 is about paying it.

---

## 5. Who makes the memory shared?

**The idea:** memory can be made shared by **hardware**, which shares the physical memory and moves cache lines, or by **software** (the OS), which shares virtual memory and moves whole pages over the network.

> **Hardware shared memory:** hardware routes memory requests from processor to the memory element containing the requested location, and **maintains "cache coherence"** in the process
>
> **Software shared memory:** software makes remote memory accessible
> - Example: **Distributed Shared Virtual Memory**
>   - Threads see a single unified virtual memory space
>   - Virtual memory manager (in the OS) distributes virtual memory pages across multiple machines
>   - Uses network to deliver a page to the requesting machine
>
> **HW SM makes physical memory shared. SW SM makes a higher abstraction (e.g. virtual mem) shared.**

| | **Hardware shared memory** | **Software shared memory** |
|---|---|---|
| What is shared | **physical** memory | **virtual** memory |
| Who does the work | hardware: routing + coherence | the OS virtual memory manager |
| Unit moved | **cache line** (~64 B) | **page** (~4 KB) |
| Moves over | the interconnect | the network |

**How software shared memory works** (it reuses page faults from Lecture 2):
1. A thread touches an address whose page lives on another machine.
2. That causes a **page fault**.
3. The OS fetches the page over the network and maps it in locally.
4. The thread carries on, never knowing the page came from elsewhere.

### ⚠️ The catch: false sharing

Two threads write **different variables that happen to sit in the same unit** (same page, or same cache line).

```
one 64-byte cache line:   [ countA | countB | ...unused... ]
Thread 1 on P1: countA++  (over and over)
Thread 2 on P2: countB++  (over and over)
```
The threads never touch each other's variable, but the system moves the **whole line** (or page). So it keeps bouncing P1 → P2 → P1 …, and both threads run slowly. It's worse with pages (4 KB), but it also happens with hardware cache lines, and it's a common cause of mysteriously slow parallel code. **Fix:** put the variables in separate lines or pages (padding).

---

## 6. Shared vs point-to-point networks

**The idea:** a **bus** is one shared wire that everyone hears: one hop away, but only one transfer at a time, so it doesn't scale. A **point-to-point** network has many separate links: several transfers at once and it scales to thousands of nodes, but a message may cross many hops.

> **Shared medium (a.k.a. bus)**
> - All elements connected to a shared physical medium
> - Low latency, low bandwidth
> - Doesn't scale to large number of elements
> - Simpler protocols (e.g. cache coherence)
>
> **Point-to-point networks**
> - Each link is only connected to two end points
> - Many examples: fully connected, ring, crossbar, (fat) tree, mesh, torus, hypercube, …
> - High latency (many "hops"), higher bandwidth per element
>   - Scales to 1000s of elements
> - Complex protocols (e.g. cache coherence)

**Bus:** one wire that everyone is attached to
```
 P1    P2    P3    P4
 │     │     │     │
═╧═════╧═════╧═════╧═══
```

**Point-to-point (a ring here):** each link joins exactly two nodes
```
 P1 ──── P2
 │       │
 P4 ──── P3
```

| | **Bus** | **Point-to-point** |
|---|---|---|
| Latency | **low**: one hop to anyone | **high**: may cross many hops |
| Bandwidth | **low**: one transfer at a time, shared | **higher**: many links work at once |
| Scales to | tens | **thousands** |
| Coherence protocol | **simpler** (snooping) | **complex** (directory) |

**"Low latency but low bandwidth" isn't a contradiction.** Latency is low because the wire reaches everyone directly. Bandwidth is low because only one transfer can use the wire at a time, so 16 processors each get roughly 1/16 of it.

### Why coherence is easier on a bus

> 💡 **Bus = a meeting room.** Everyone hears everything. If P1 announces "I'm writing X", anyone holding a copy of X hears it and throws their copy away. That's **snooping**: caches listen to all traffic, and it costs almost nothing, because the broadcast already happens.
>
> **Point-to-point = phone calls.** A call from P1 to memory is heard by no one else. So someone has to **keep a list of who has a copy** of each line and call them individually. That list is a **directory**, and directory protocols are much more complex.

> 📌 **Bus → snooping. Point-to-point → directory.** That pairing is the main structure of topic 7.

---

## 7. Three real interconnects

**The idea:** one real machine at each level of the hierarchy, each using a **different network shape**. Real systems mix them.

| Level | Example | Shape | Joins |
|---|---|---|---|
| **Chip** | Intel Xeon E5-2600 | **double ring** | 8 cores + 8 L3 banks |
| **Node** (one server) | Intel QuickPath Interconnect (QPI) | **fully connected** | 4 sockets + I/O hubs |
| **Rack** | SGI Altix 4700 | **crossbar switches** | server blades |

### Chip level: Xeon E5-2600, double ring (slide 8)

Eight cores and eight L3 banks (2.5 MB each, 20 MB in total) sit on two rings, plus stops for QPI, PCIe and the memory controller.
- **Why a ring?** Each stop needs only two links, so it's cheap, and it works fine for about a dozen stops.
- **Why two rings, in opposite directions?** A message can go whichever way is shorter. With 8 stops, a one-way ring can take up to 7 hops; with two directions it's at most 4.
- **The L3 is split into banks**, each next to a core. A core reaching its own bank is faster than reaching the far side of the chip. So **access time already depends on location inside one chip.**

### Node level: Intel QPI, fully connected (slide 9)

Four sockets, each with **its own memory**, and every socket linked directly to every other one.
- This is the **coupled** shape (§3) at socket scale. Another socket's memory costs an extra hop → classic **NUMA**. Any multi-socket server works this way.
- **Fully connected only works for small counts.** Links needed = n(n−1)/2:

| Nodes | Links |
|---|---|
| 4 | 6 |
| 8 | 28 |
| 100 | 4,950 |

### Rack level: SGI Altix 4700, crossbar switches (slide 11)

Router boards with **crossbar switches** join the server blades, which are linked onward by **NUMAlink**.
- A **crossbar** is a switch that can connect **any input to any output at the same time**, as long as no two inputs want the same output.
- 💡 Like an old telephone switchboard: many calls at once, each on its own path.
- You get full connectivity without running a cable between every pair of nodes. The switch is complex inside, but it's built once, in one place.

> 📌 Three levels, three different shapes, in one machine. That's §3's "hierarchical structure" in real hardware, and why the flat memory the programmer sees is a convenient fiction.

---

## 8. Organizing point-to-point networks

**The idea:** a network's shape (**topology**) trades performance for cost. Routing is done either by **separate router chips** (**indirect**) or by routers **built into each node** (**direct**, "glueless").

> - **Network topology:** organization of network
>   - Tradeoff performance (connectivity, latency, bandwidth) and cost
> - **Router chips**
>   - Networks that require **separate router chips** are **indirect**
>   - Networks that use **processor/memory/router packages** are **direct**
>     - Fewer components, **"Glueless MP"**

**Indirect (tree):** the routers are separate chips that only route
```
             [R]
           ┌──┴──┐
          [R]   [R]
          ┌┴┐   ┌┴┐
          N N   N N        N = node (CPU + cache + memory)
```

**Direct (mesh):** every node has its own built-in router
```
 N ── N ── N
 │    │    │
 N ── N ── N               N = CPU + cache + memory + router
 │    │    │
 N ── N ── N
```

| | **Indirect** | **Direct** |
|---|---|---|
| Routing done by | separate router chips | a router inside each node package |
| Parts needed | more | **fewer** ("glueless") |
| Slide's example | tree | mesh |
| Looks like | decoupled (§3) | coupled (§3) |

**"Glueless":** "glue logic" means the extra support chips a design needs around its main parts. A glueless multiprocessor needs none: you wire the processor packages straight to each other, because each one already contains its router. Fewer parts, cheaper boards, lower latency.

---

## 9. UMA and NUMA

**The idea:** **UMA** = every processor reaches every memory in the same time, so data placement doesn't matter. **NUMA** = your own memory is faster than other nodes' memory. NUMA can be faster overall, **but only if each thread's data is in its own node.**

> **Uniform Memory Access (UMA):** equal latency to memory from all processors
> - Simpler software, doesn't matter where you put data
> - Lower peak performance
> - Common in bus-based systems
>
> **Non-Uniform Memory Access (NUMA):** local memory access faster than remote
> - More complex software: where you put data matters
> - Higher peak performance: assuming proper data placement

The pictures are the two shapes from §3: **UMA = decoupled, NUMA = coupled.**

> 💡 **UMA** = all files in one central archive: everyone walks the same distance, and it gets crowded. **NUMA** = each office has its own cabinet: your own cabinet is seconds away, someone else's is down the corridor. Great **if** you keep your files in your own office.

### Example: why placement matters

Illustrative numbers: 4 nodes, local access 100 ns, remote access 300 ns.

| Placement | Threads on nodes 1–3 | Node 0's memory | Result |
|---|---|---|---|
| **Good:** each thread's data in its own node | all accesses local, 100 ns | normal load | ✅ fast; all 4 memories work in parallel |
| **Bad:** all data on node 0 | every access remote, 300 ns | handles everyone's traffic, becomes a bottleneck | ❌ can be **worse than UMA** |

> ⚠️ **NUMA is not automatically faster.** "Higher peak performance" holds only *with proper data placement*. Bad placement adds hops without adding any locality.

This is why real systems let software control placement: Linux's `numactl`, and the **first-touch** policy (a page is placed in the memory of the node that first writes it), plus OpenMP thread-affinity settings. That's the "more complex software" on the slide.

---

## 10. Why caches matter even more in a multiprocessor

**The idea:** on one processor, caches hide slow memory. On a multiprocessor they matter for **two more reasons**: the network makes memory **even slower**, and memory and network bandwidth are **shared**, so **your cache miss slows everyone else down too.**

> Cache memory hierarchy is important for single-processor systems to bridge the speed gap between processor and memory.
>
> It is important in a multiprocessor for two respects:
> 1. The interconnect increases the latency to memory.
> 2. Since memory and interconnect are shared resources, their bandwidth is precious. The cache hierarchy helps reduce the penalty of memory accesses and the pressure on the memory and interconnect.

| Reason | Plain words |
|---|---|
| **1. Latency** | Memory was already ~200 cycles away on one processor (Lecture 2). Now add network hops, waiting for the network, and maybe another node's memory. A miss costs even more. |
| **2. Bandwidth** (new) | The network and memory are **shared**. Every miss uses bandwidth that all other processors also need. |

> 💡 One car on an empty road vs one car in rush hour. Every extra trip (miss) you make adds to the traffic that everyone else is stuck in.

> 📌 **On a multiprocessor, a cache miss costs more *and* hurts others.** That's why everyone builds organization (d): private L1s plus a shared L2.

---

## 11. The two big issues

**The idea:** sharing memory between processors raises two big questions. **Coherence:** do all processors see the same values for **one** variable? **Consistency:** do they see writes to **different** variables in the same order? Plus a third, smaller one: **synchronization**, i.e. hardware support (atomic operations) for locks.

> **Two big ones**
> - **Cache coherence** (single memory location/variable)
> - **Memory consistency model** (multiple memory locations/variables)
>
> **Closely related — but often confused**
>
> **A (slightly) smaller one (very important nonetheless)**
> - **Synchronization support**
>   - Special instructions, e.g. atomic operations
>   - Advanced features such as full/empty bits

### Coherence vs consistency

| | **Cache coherence** | **Memory consistency** |
|---|---|---|
| About | **ONE** memory location | **SEVERAL** memory locations |
| Question | Does everyone agree on the values *this one variable* took? | Does everyone agree on the **order** of writes to *different* variables? |
| Goes wrong when | P2 keeps reading an old copy of X after P1 wrote it (the table in §4) | P1 writes A then B; P2 sees the new B but the old A |
| Caused by | private caches holding copies | hardware reordering or delaying writes |

**A consistency example:**
```
    P1                      P2
    A = 1;                  while (flag == 0) ;   // wait for the signal
    flag = 1;               print(A);             // prints 0 ?!
```
P1 sets A, then raises the flag. P2 waits for the flag, then reads A. It should print **1**. But if the hardware lets the write to `flag` become visible **before** the write to `A`, P2 can see `flag = 1` while A is still 0, and print **0**.

Each variable on its own can be perfectly coherent; the problem is the **order between them**. The **memory consistency model** is the hardware's contract for which reorderings are allowed.

> 💡 You text a friend "dinner's ready", then "come downstairs". If the messages arrive in the wrong order, both are correct on their own, but the order breaks the meaning.

(This is the "need to know how the hardware interleaves accesses from different processors" point from Lecture 4.)

### Synchronization support

**Why hardware has to help.** Lecture 4's `sum += mysum` is really three steps: read, add, write. Two threads can interleave them:

| Step | Thread A (mysum = 5) | Thread B (mysum = 3) | `sum` in memory |
|---|---|---|---|
| 1 | read sum → 10 | | 10 |
| 2 | | read sum → 10 | 10 |
| 3 | 10 + 5 = 15 | | 10 |
| 4 | | 10 + 3 = 13 | 10 |
| 5 | write 15 | | 15 |
| 6 | | write 13 | **13 ❌** (should be 18) |

A **lock** stops this. But the lock itself is the same kind of read-then-write:
```
while (L == 1) ;   // wait until the lock is free
L = 1;             // take it
```
Both threads can read L = 0 before either writes 1, and both take the lock. The same race, one level down.

**Atomic operations** fix it. An **atomic** instruction does its read and write **as one indivisible step**, so no other processor can get in between. Examples: **test-and-set** (read the old value and set it to 1, in one step), **compare-and-swap**, **load-linked / store-conditional**. Ordinary loads and stores can't do this; it needs special instructions.

**Full/empty bits:** each memory word gets an extra bit saying "full" or "empty". A read can **wait until the word is full** (someone has written it), and a write can wait until it's empty. Every word becomes a tiny synchronization point.
> 💡 A mailbox with a flag: you only collect the letter once the flag is up.

Elegant, but it costs extra bits on every word, so only a few machines used it (e.g. the Tera/Cray MTA).

---

## 12. Summary sheet

### Definitions

| Term | Definition |
|---|---|
| **TLP** | multiple program counters · MIMD · tightly-coupled shared-memory multiprocessors |
| **Grain size** | amount of computation assigned to each thread |
| **SMP** | small number of cores sharing one memory with **uniform** latency |
| **DSM** | memory distributed among processors · **non-uniform** latency (NUMA) |
| **UMA** | same latency from all processors · simpler software · lower peak performance |
| **NUMA** | local faster than remote · placement matters · higher peak **with proper placement** |
| **Direct network** | router inside each processor/memory package ("glueless MP") |
| **Indirect network** | needs separate router chips |
| **Cache coherence** | agreement about **one** memory location |
| **Memory consistency** | ordering across **several** memory locations |
| **Atomic operation** | read + write done as one indivisible step |

### The four organizations

| | Organization | Key point |
|---|---|---|
| (a) | dance-hall SMP | not practical; no cache |
| (b) | shared L1 | no replication, but higher hit latency |
| (c) | private caches | fast hits, ⚠️ **creates the coherence problem** |
| (d) | private caches + shared L2 | popular for CMPs: what multicore chips are |

### Networks

| | Bus | Point-to-point |
|---|---|---|
| Latency | low | high (many hops) |
| Bandwidth | low (shared) | higher per element |
| Scale | tens | 1000s |
| Coherence | simpler → **snooping** | complex → **directory** |

| Real example | Level | Shape |
|---|---|---|
| Xeon E5-2600 | chip | double ring |
| Intel QPI | node | fully connected (n(n−1)/2 links) |
| SGI Altix 4700 | rack | crossbar switches |

### Things worth memorising

- **TLP = several program counters.** ILP and DLP work inside one.
- **n processors need n threads**, and a grain that's too fine drowns in overhead.
- **The hardware is a hierarchy**, not the flat memory the programming model shows.
- **Decoupled → UMA, coupled → NUMA.** The same two shapes appear at every level.
- **Private caches create coherence**; the industry accepted that for speed.
- **NUMA is faster only with good data placement.**
- **A miss on a multiprocessor costs more and slows everyone.**
- **Coherence = one variable. Consistency = order across variables.**
- **Locks need atomic instructions**, because a lock is itself a read-modify-write.

### ⚠️ Traps

| Trap | Remember |
|---|---|
| "direct" network | slide 3 (DSM) and slide 12 use the word differently |
| NUMA faster? | only with proper data placement |
| (b) vs (c) | (b) has no coherence problem; (c) creates it |
| Bus latency vs bandwidth | low latency (one hop) and low bandwidth (shared wire) at the same time |
| Coherence vs consistency | one location vs order across many |

### Links to earlier lectures

- **Grain size** = granularity (L4) = the overhead curve (L3)
- **Flat model vs hierarchical hardware** = the model/hardware mismatch (L4)
- **Software shared memory** reuses page faults (L2)
- **Consistency** = "how hardware interleaves accesses" (L4)
- **Atomic operations** are what make L4's `LOCK`/`UNLOCK` possible

---

## 13. Self-test

<details>
<summary><b>1.</b> Give the three-bullet definition of thread-level parallelism.</summary>

- **Multiple program counters**
- Uses the **MIMD** model
- Targeted for **tightly-coupled shared-memory multiprocessors**

One PC = one instruction stream, so several PCs = several independent streams. ILP and DLP both work inside a single stream.
</details>

<details>
<summary><b>2.</b> What is grain size, and why can using threads for data-level parallelism backfire?</summary>

**Grain size** = the amount of computation assigned to each thread.

Creating, scheduling and synchronizing threads costs time. If each thread only handles a few array elements, **the overheads outweigh the benefit**: more time goes into managing threads than into computing.
</details>

<details>
<summary><b>3.</b> Distinguish SMP from DSM.</summary>

**SMP:** a small number of cores sharing **one** memory with **uniform** latency.

**DSM:** memory **distributed** among processors, **non-uniform** latency (NUMA), joined by switched / multi-hop networks.

Both give the programmer one shared address space; they differ in where the memory physically sits.
</details>

<details>
<summary><b>4.</b> What does "contrary to the flat view of shared-memory programming models" mean?</summary>

The programming model presents memory as one flat space where every access looks the same. Real hardware is a **hierarchy** (chip, machine, across machines), and access time depends on where the data is. Correct code can still be slow if its data is far away.
</details>

<details>
<summary><b>5.</b> Decoupled vs coupled configuration: which gives UMA and which NUMA?</summary>

**Decoupled:** memory on the far side of the interconnect; every access crosses it → same latency for all → **UMA**.

**Coupled:** memory next to each processor (with a router); local is fast, remote crosses the network → **NUMA**.
</details>

<details>
<summary><b>6.</b> Of the four organizations on slide 5, which creates the cache coherence problem, and which is used in modern chips?</summary>

**(c) private caches** creates it: the same line can sit in several caches, so one processor's write leaves the other copies stale. (b), with a shared L1, has no replication and no coherence problem, but pays in hit latency.

**(d) private caches + shared L2** is what modern chips use ("popular for CMPs").
</details>

<details>
<summary><b>7.</b> Hardware vs software shared memory: what is shared in each, and in what unit?</summary>

**Hardware:** **physical** memory. The hardware routes requests and keeps caches coherent. Unit: the **cache line** (~64 B).

**Software:** **virtual** memory. The OS spreads pages across machines and fetches them over the network on a page fault. Unit: the **page** (~4 KB).
</details>

<details>
<summary><b>8.</b> What is false sharing, and how do you fix it?</summary>

Two threads write **different** variables that sit in the **same** cache line (or page). The system moves the whole unit, so it bounces between processors even though the threads never share data. Fix: put the variables in separate lines or pages (padding).
</details>

<details>
<summary><b>9.</b> How can a bus be both low latency and low bandwidth?</summary>

**Low latency:** the destination is one hop away, because the wire reaches everyone directly.
**Low bandwidth:** that wire is shared, so only one transfer happens at a time and each processor gets a small slice.
</details>

<details>
<summary><b>10.</b> Why is cache coherence simpler on a bus than on a point-to-point network?</summary>

A bus is a **broadcast**: every cache sees every transaction, so caches can listen and invalidate their copies (**snooping**).

On a point-to-point network nobody else sees a message, so something has to **track who has copies** of each line (a **directory**). That is much more complex.
</details>

<details>
<summary><b>11.</b> A fully connected network has 8 nodes. How many links? Why isn't it used at rack scale?</summary>

8 × 7 / 2 = **28** links. The count grows with n² (100 nodes would need 4,950), so it's only used for small counts, like QPI's 4 sockets. Racks use crossbar switches instead.
</details>

<details>
<summary><b>12.</b> Direct vs indirect networks (slide 12)?</summary>

**Indirect:** needs **separate router chips** (e.g. a tree).
**Direct:** each processor/memory package contains its own router (e.g. a mesh): fewer components, "glueless MP".
</details>

<details>
<summary><b>13.</b> Is NUMA always faster than UMA?</summary>

**No.** NUMA has higher peak performance **only with proper data placement**. If all the data sits on one node, other threads pay remote latency on every access and that node's memory becomes a bottleneck, which can be worse than UMA. UMA's advantage: simpler software, placement doesn't matter.
</details>

<details>
<summary><b>14.</b> Give the two extra reasons caches matter in a multiprocessor.</summary>

1. **The interconnect increases memory latency**, so misses cost more.
2. **Memory and interconnect bandwidth are shared and precious**, so each miss also slows down the other processors. Caches reduce that pressure.
</details>

<details>
<summary><b>15.</b> Coherence vs consistency, one line each. Then: in the <code>A = 1; flag = 1;</code> example, which one is the problem?</summary>

**Coherence:** agreement on the values of **one** memory location.
**Consistency:** agreement on the **order** of writes to **different** locations.

The flag example is **consistency**: A and flag can each be coherent, but P2 sees the write to flag before the write to A.
</details>

<details>
<summary><b>16.</b> Why do we need atomic instructions if we already have locks?</summary>

Because **the lock itself** is a read-then-write (`while (L == 1); L = 1;`), and two threads can both read L = 0 and both take the lock. An atomic instruction such as **test-and-set**, **compare-and-swap** or **load-linked / store-conditional** does the read and the write as one indivisible step, so only one thread can win.
</details>

<details>
<summary><b>17.</b> What are full/empty bits?</summary>

An extra bit on each memory word marking it **full** or **empty**. A read can wait until the word is full, and a write until it's empty, so every word works as a small synchronization point. It costs extra bits on every word, which is why few machines (e.g. Tera/Cray MTA) used it.
</details>

---

## References (slide 16)

- Michel Dubois, Murali Annavaram, and Per Stenström. 2012. *Parallel Computer Organization and Design.* Cambridge University Press, USA.
- David A. Patterson and John L. Hennessy. *Computer Architecture: A Quantitative Approach.* Morgan Kaufmann, 6th edition.
- Slides by Prof. Nima Honarmand (Stony Brook University).

---

*Previous: [Lecture 5 — Instruction-Level Parallelism](./05-instruction-level-parallelism.md)*
