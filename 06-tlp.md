# COE 502 — Lecture 6: Multiprocessor Systems (Thread-Level Parallelism)

**Parallel Processing Architectures** · KFUPM · Dr. Abdulaziz Tabbakh

> These notes follow deck `06-TLP.pdf`. Anything marked **[note]** is extra context not on the slides.
>
> **This deck is short but load-bearing.** It describes the *hardware* that the rest of the course
> is about — and its final slide names the two problems (cache coherence and memory consistency)
> that topic 7 and **course outcome 2** are devoted to.

---

## Contents

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

## 1. What thread-level parallelism is

> **Thread-Level parallelism**
> - **Have multiple program counters**
> - **Uses MIMD model**
> - **Targeted for tightly-coupled shared-memory multiprocessors**
>
> **For *n* processors, need *n* threads**
>
> **Amount of computation assigned to each thread = grain size**
> - Threads can be used for data-level parallelism, but **the overheads may outweigh the benefit**

### Unpacking the three bullets

**"Multiple program counters"** is the crispest definition of TLP you will get. A **program counter** is the register holding the address of the next instruction to execute. One PC = one instruction stream. ILP and DLP both operate within a *single* stream, so they have **one** PC. TLP means **several** — genuinely independent streams, each with its own place in its own program.

**"Uses MIMD model"** — Flynn's Multiple Instruction, Multiple Data (Lecture 1 §19). Every processor may be executing a different instruction on different data.

**"Tightly-coupled shared-memory multiprocessors"** — *tightly coupled* means the processors are close together and share memory, as opposed to a loosely-coupled cluster that communicates by messages. This deck is about the shared-memory branch of Lecture 4's axis 2.

### For *n* processors you need *n* threads

Obvious once stated, and it's the practical constraint: **hardware parallelism is useless without software parallelism to feed it.** A 64-core machine running a single-threaded program uses one core. The other 63 idle.

This is the whole reason Lecture 1 ended by saying performance became the programmer's problem in 2004.

### Grain size

> **Amount of computation assigned to each thread = grain size.**

Same concept as **parallelism granularity** from Lecture 4 §9, under a different name.

| Grain | Means | Consequence |
|---|---|---|
| **Coarse** | lots of work per thread | coordination overhead negligible; fewer threads available |
| **Fine** | little work per thread | more parallelism available; overhead can dominate |

> ⚠️ **"Threads can be used for data-level parallelism, but the overheads may outweigh the benefit."**
>
> You *can* exploit DLP with threads (that is exactly what the SPMD matrix multiply in Lecture 4
> did). But thread creation, scheduling and synchronisation are expensive. If each thread only
> processes a handful of array elements, you spend more time managing threads than computing.
>
> That is the Mortar curve from Lecture 3, and the $P_{opt} = \sqrt{T_{MM}/T_{init}}$ formula from
> Lecture 4, restated in one sentence.

---

## 2. Two types: SMP and DSM

> **Symmetric multiprocessors (SMP)**
> - **Small number of cores**
> - **Share single memory with uniform memory latency**
>
> **Distributed shared memory (DSM)**
> - **Memory distributed among processors**
> - **Non-uniform memory access/latency (NUMA)**
> - **Processors connected via direct (switched) and non-direct (multi-hop) interconnection networks**

```
   SMP — one shared memory                 DSM — memory per node
┌────┐ ┌────┐ ┌────┐ ┌────┐          ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
│ P  │ │ P  │ │ P  │ │ P  │          │Multi-│ │Multi-│ │Multi-│ │Multi-│
├────┤ ├────┤ ├────┤ ├────┤          │core  │ │core  │ │core  │ │core  │
│cache│ │cache│ │cache│ │cache│        │ MP   │ │ MP   │ │ MP   │ │ MP   │
└──┬─┘ └──┬─┘ └──┬─┘ └──┬─┘          └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘
   └──────┴───┬──┴──────┘             Mem│I/O  Mem│I/O  Mem│I/O  Mem│I/O
        ┌─────┴──────┐                   └────────┴────────┴────────┘
        │Shared cache│                   ┌────────────────────────┐
        └─────┬──────┘                   │ Interconnection network│
    ┌─────────┴────┐                     └────────────────────────┘
    │ Main memory  │  I/O                  (more nodes below)
    └──────────────┘
```

Both are **shared memory** from the programmer's point of view — one global address space. The difference is **where the memory physically sits**, and therefore how long it takes to reach.

| | **SMP** | **DSM** |
|---|---|---|
| Memory | one shared pool | distributed among processors |
| Latency | **uniform** (UMA) | **non-uniform** (NUMA) |
| Scale | **small** number of cores | large |
| Interconnect | typically a bus | switched / multi-hop networks |

**[note] Why SMP can't scale.** All processors contend for one memory and one interconnect. Add processors and you add traffic through a fixed-bandwidth resource, so each processor gets a smaller share. Past a modest count (tens), the interconnect saturates and extra cores buy nothing. DSM fixes this by giving each node its own memory — at the price of making *where your data lives* matter to performance.

---

## 3. Shared-memory physical architecture

> **Almost always there is a hierarchical structure**
> - **Within socket, within a machine, across machines**
> - **Contrary to the flat view of shared-memory programming models**
>
> **At each level of hierarchy, there are**
> - multiple **processing elements** (cores, sockets, boxes, …)
> - multiple **memory elements** (caches, DRAM modules, …)
> - an **interconnection network** connecting them together

> 📌 **"Contrary to the flat view of shared-memory programming models"** is the sentence to
> remember. OpenMP presents one flat address space where every access looks alike. The hardware
> underneath is a **hierarchy** with latencies differing by orders of magnitude depending on
> whether the data is in your L1, another core's L2, another socket's DRAM, or another machine.
>
> This is Lecture 4 slide 3's warning made concrete: *the programming model and the hardware can
> differ, but for high performance they should match.* Here the mismatch is permanent and
> deliberate — and it is why "it compiles and runs correctly" says nothing about whether it's fast.

### Two configurations at each level

> At each level, one of **two general configurations** w.r.t. processor/memory connection:

```
      DECOUPLED CONFIGURATION                  COUPLED CONFIGURATION
  ┌───┐  ┌───┐  ┌───┐  ┌───┐              ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐
  │ P │  │ P │  │ P │  │ P │              │  P  │ │  P  │ │  P  │ │  P  │
  └─┬─┘  └─┬─┘  └─┬─┘  └─┬─┘              ├───┬─┤ ├───┬─┤ ├───┬─┤ ├───┬─┤
    │      │      │      │                │Mem│R│ │Mem│R│ │Mem│R│ │Mem│R│
  ┌─┴──────┴──────┴──────┴─┐              └─┬───┘ └─┬───┘ └─┬───┘ └─┬───┘
  │      Interconnect      │                │       │       │       │
  └─┬──────┬──────┬──────┬─┘              ┌─┴───────┴───────┴───────┴─┐
    │      │      │      │                │       Interconnect        │
  ┌─┴─┐  ┌─┴─┐  ┌─┴─┐  ┌─┴─┐              └───────────────────────────┘
  │Mem│  │Mem│  │Mem│  │Mem│
  └───┘  └───┘  └───┘  └───┘
```

| | **Decoupled** | **Coupled** |
|---|---|---|
| Memory sits | on the **far side** of the interconnect | **with** the processor, behind a router (R) |
| Every access | crosses the interconnect | local access is **free**; remote crosses |
| Latency | uniform | **non-uniform** |
| Leads to | UMA | NUMA |

**[note]** The `R` in the coupled diagram is a **router** — the coupled node packages processor, memory and routing together. That's what slide 12 calls a **direct network** ("glueless MP"): no separate router chips needed.

---

## 4. Four multiprocessor organizations

Slide 5 walks a progression from impractical to what everyone actually builds.

| | Organization | Properties |
|---|---|---|
| **(a)** | **"Dance-hall" / SMP** | All processors on one side, all memory modules on the other side of the interconnect. Access time to memory is **the same for all processors**. **Not practical; no cache** |
| **(b)** | **SMP with shared level-1 cache** | **No replication** but **higher hit latency** than in (c) |
| **(c)** | **SMP with private caches** | each processor has its own cache |
| **(d)** | **SMP with private caches and shared level-2 cache** | **Popular for chip multiprocessors (CMPs)** |

### Reading the progression

**(a) is the textbook picture and nobody builds it.** Without caches, every single memory access crosses the interconnect — which saturates immediately. It exists on the slide to establish the baseline that the other three improve on.

**(b) shares one L1 among all processors.** The upside is stated: **no replication** — only one copy of any line exists, so there is no coherence problem at all. The downside is **higher hit latency**, because a cache shared by *n* processors must be physically larger and further away, and must arbitrate between simultaneous requests.

**(c) gives each processor its own cache.** Hit latency is now low, because the cache is small and right next to its processor.

> ⚠️ **And this is the exact moment the coherence problem is created.** Private caches mean the
> same memory line can sit in **several caches at once**. If one processor writes its copy, the
> others are silently stale. Organization (b) avoided this by having no replication; (c) buys speed
> and inherits the problem.

**(d) is (c) plus a shared L2 — and it is what modern chips are.** Private L1s for speed, a shared L2 (or L3) for capacity and as a common point where coherence can be managed. The slide notes it is **popular for chip multiprocessors (CMPs)**, which is the formal name for a multicore chip.

> 📌 **The whole of topic 7 exists because the industry chose (d) over (b)** — it traded a
> hardware-maintained coherence problem for lower hit latency. Keep that trade in mind: coherence
> is not an accident, it's the price of private caches.

---

## 5. Who makes the memory shared?

> **Hardware shared memory:** hardware routes memory requests from processor to memory element containing the requested location
> - And **maintains "cache coherence"** in the process
>
> **Software shared memory:** software makes remote memory accessible
> - Example: **Distributed Shared Virtual Memory**
>   - Threads see a **single unified virtual memory space**
>   - **Virtual memory manager (in the OS)** distributes virtual memory pages across multiple machines
>   - Uses **network to deliver a page** to the requesting machine
>
> **HW SM makes physical memory shared**
> **SW SM makes a higher abstraction (e.g. virtual mem) shared**

| | **Hardware shared memory** | **Software shared memory** |
|---|---|---|
| What is shared | **physical** memory | a **higher abstraction** (virtual memory) |
| Who does the work | hardware routing + coherence protocol | the OS virtual memory manager |
| Unit of transfer | **cache line** (~64 bytes) | **page** (~4 KB) |
| Over what | the interconnect | the network |
| Coherence | maintained in hardware | managed in software |

**[note] How software shared memory actually works.** It reuses the page-fault mechanism from Lecture 2 §10. A thread touches a virtual address whose page lives on another machine → page fault → the OS handler fetches that page over the network and maps it in. The program thinks it has one flat address space; the OS is quietly shipping 4 KB pages between machines.

The catch is **granularity**. The unit is a whole page, so two threads writing *different variables that happen to share a page* will bounce it back and forth endlessly. That's **false sharing** — the same pathology exists in hardware coherence at cache-line granularity, where it is one of the most common causes of mysteriously slow parallel code.

---

## 6. Shared vs point-to-point networks

> **Shared medium (a.k.a. bus)**
> - 🔴 **All elements connected to a shared physical medium**
> - **Low latency, low bandwidth**
> - **Doesn't scale to large number of elements**
> - **Simpler protocols (e.g. cache coherence)**
>
> **Point-to-point networks**
> - 🔴 **Each link is only connected to two end points**
> - Many examples: **fully connected, ring, crossbar, (fat) tree, mesh, torus, hypercube, …**
> - **High latency (many "hops"), higher bandwidth per element**
>   - **Scales to 1000s of elements**
> - **Complex protocols (e.g. cache coherence)**

| | **Shared medium (bus)** | **Point-to-point** |
|---|---|---|
| Structure | everything on one physical medium | each link joins exactly **two** endpoints |
| Latency | **low** | **high** — many hops |
| Bandwidth | **low** (shared among all) | **higher per element** |
| Scaling | poor — tens | **1000s of elements** |
| Coherence protocol | **simpler** | **complex** |

### Why the coherence protocol difficulty flips

This is the most important row and the slides don't explain it, so:

**On a bus, every message is seen by everyone, automatically.** It is a broadcast medium. So a cache can simply *listen* to every transaction and react — "someone is writing address X and I have a copy, so I'll invalidate mine." That is **snooping**, and it is nearly free because the broadcast already exists.

**On a point-to-point network there is no broadcast.** A message goes from A to B and nobody else sees it. So if a processor wants to write a line, it has no way to tell the others — unless something *tracks who has copies*. That something is a **directory**, and directory protocols are substantially more complex.

> 📌 **Bus ⇒ snooping. Point-to-point ⇒ directory.** That pairing is the main structural fact of
> topic 7, and slide 7 is where it is seeded.

**[note] "Low latency, low bandwidth" for a bus looks contradictory** until you separate the two. Latency is low because the destination is *one hop away* — the wire physically reaches everyone. Bandwidth is low because that one wire is **shared**, so only one transfer can happen at a time and every added processor gets a smaller slice.

A point-to-point network inverts both: several links carry traffic simultaneously (high aggregate bandwidth) but a message may traverse many hops to arrive (high latency).

---

## 7. Three real interconnects

The deck gives one example at each level of the hierarchy from slide 4.

### Chip level — Intel Xeon E5-2600 (slide 8)

> - **Double ring interconnect**
> - **Connecting 8 cores and 8 banks of L3**

Eight cores down the sides, eight 2.5 MB L3 banks in the middle, two counter-rotating rings joining them, plus stops for QPI, PCIe and the memory controller.

**[note] Why a ring, and why two?** A ring is cheap — each stop needs only two links — and it scales reasonably to a dozen or so stops. Two rings running in **opposite directions** halves the worst-case hop count: instead of travelling up to 7 stops one way, you pick whichever direction is shorter, so at most 4.

Note also that the L3 is **banked** — the 20 MB is split into 8 pieces, each physically next to a core, with addresses interleaved across them. So L3 access time already depends on which bank holds your address. **NUMA exists inside a single chip.**

### Node level — Intel QuickPath Interconnect (slide 9)

> - **Fully connected**
> - **Connecting processor sockets to each other and IO hubs**
> - **Memory directly connected to processor sockets using a memory bus**

Four CPU sockets, each with its own memory, every socket linked directly to every other, plus two I/O hubs.

This is the **coupled configuration** from slide 4 at socket scale — and because memory hangs off each socket, reaching another socket's memory costs an extra hop. **This is textbook NUMA**, and it is what you are running on any multi-socket server.

**[note] Fully connected works here only because there are four sockets.** A fully connected network of *n* nodes needs *n(n−1)/2* links — 6 for four nodes, but 4,950 for a hundred. It does not scale, which is why the rack level uses switches instead.

### Rack level — SGI Altix 4700 (slide 11)

> - **Crossbar switches**
> - **Connecting nodes (server blades) to each other**

Router boards with crossbar switches joining ten blades, linked onward by NUMAlink.

A **crossbar** is a switch that can connect any input to any output simultaneously, as long as no two want the same output. Full connectivity without *n²* wires between endpoints — you pay for it in the switch, which itself costs *n²* internally but is built once in one chip rather than as cabling.

### Reading the three together

| Level | Interconnect | Topology | Scale |
|---|---|---|---|
| **Chip** | Xeon E5-2600 double ring | ring | 8 cores |
| **Node** | Intel QPI | fully connected | 4 sockets |
| **Rack** | SGI Altix NUMAlink | crossbar switches | 10 blades per enclosure |

> 📌 **Three levels, three different topologies, in one machine.** That is slide 4's "almost always
> there is a hierarchical structure" made real — and it is why the flat view presented to the
> programmer is a fiction, however convenient.

---

## 8. Organizing point-to-point networks

> **Network topology: organization of network**
> - **Tradeoff performance (connectivity, latency, bandwidth) and cost**
>
> **Router chips**
> - Networks that require **separate router chips** are **indirect**
> - Networks that use **processor/memory/router packages** are **direct**
>   - Fewer components, **"Glueless MP"**

```
        INDIRECT TREE                           DIRECT MESH
             ┌───┐                        ┌───────┐    ┌───────┐
        ┌───▶│ R │◀───┐                   │ CPU($)│    │ CPU($)│
        │    └───┘    │                   ├────┬──┤    ├──┬────┤
     ┌──┴┐         ┌──┴┐                  │Mem │R │◀──▶│R │ Mem│
  ┌─▶│ R │◀─┐   ┌─▶│ R │◀─┐               └────┴┬─┘    └┬─┴────┘
  │  └───┘  │   │  └───┘  │                     │       │
┌─┴──┬─┐ ┌──┴─┬─┐ ...                           ▼       ▼
│Mem │R│ │Mem │R│                         ┌────┬──┐    ┌──┬────┐
├────┴─┤ ├────┴─┤                         │Mem │R │◀──▶│R │ Mem│
│CPU($)│ │CPU($)│                         ├────┴──┤    ├──┴────┤
└──────┘ └──────┘                         │ CPU($)│    │ CPU($)│
                                          └───────┘    └───────┘
```

| | **Indirect** | **Direct** |
|---|---|---|
| Routing done by | **separate router chips** | routers **integrated** into the processor/memory package |
| Component count | more | **fewer** — "Glueless MP" |
| Example on the slide | tree | mesh |

**[note] "Glueless"** is industry slang: "glue logic" is the extra support chips a design needs around its main components. A glueless multiprocessor needs none — you wire the processor packages straight to each other, because each already contains its router. Cheaper boards, fewer parts, lower latency.

Note the direct-mesh picture is the **coupled configuration** from slide 4, and the indirect tree is closer to **decoupled**. The same two shapes keep recurring at every level.

---

## 9. UMA and NUMA

> **Uniform Memory Access (UMA):** **Equal latency to memory from all processors**
> - **Simpler software, doesn't matter where you put data**
> - **Lower peak performance**
> - **Common in bus-based systems**
>
> **Non-Uniform Memory Access (NUMA):** **Local memory access faster than remote**
> - **More complex software: where you put data matters**
> - **Higher peak performance: assuming proper data placement**

```
         UMA EXAMPLE                              NUMA EXAMPLE
  ┌──────┐┌──────┐┌──────┐┌──────┐        ┌──────┐┌──────┐┌──────┐┌──────┐
  │CPU($)││CPU($)││CPU($)││CPU($)│        │CPU($)││CPU($)││CPU($)││CPU($)│
  └──┬───┘└──┬───┘└──┬───┘└──┬───┘        ├───┬──┤├───┬──┤├───┬──┤├───┬──┤
     │       │       │       │            │Mem│R ││Mem│R ││Mem│R ││Mem│R │
  ┌──┴───────┴───────┴───────┴──┐         └─┬────┘└─┬────┘└─┬────┘└─┬────┘
  │        Interconnect          │           │       │       │       │
  └──┬───────┬───────┬───────┬──┘         ┌──┴───────┴───────┴───────┴──┐
     │       │       │       │            │        Interconnect          │
  ┌──┴──┐┌──┴──┐┌──┴──┐┌──┴──┐            └──────────────────────────────┘
  │ Mem ││ Mem ││ Mem ││ Mem │
  └─────┘└─────┘└─────┘└─────┘
```

Same two shapes as slide 4's decoupled/coupled — **UMA is the decoupled configuration, NUMA is the coupled one.**

### The trade-off, stated honestly

> ⚠️ **The counter-intuitive pair:** UMA has **lower peak performance** but **simpler software**;
> NUMA has **higher peak performance** but **only assuming proper data placement**.

NUMA is not unconditionally faster. It is faster *if* each thread's data sits in its own node's memory. Get the placement wrong — every thread hammering one node's memory — and NUMA is **worse** than UMA, because you've added hops without adding locality.

**[note]** This is why real systems expose NUMA to software: Linux has `numactl` and a first-touch allocation policy (a page is placed in the memory of whichever node first writes it), and OpenMP has thread-affinity controls. All of it exists so the programmer can get data placement right — which is precisely the "more complex software" the slide warns about.

It is also the sharpest example of slide 4's point: the programming model says memory is flat, the hardware says it isn't, and the gap shows up as a performance cliff rather than a compile error.

---

## 10. Why caches matter even more in a multiprocessor

> Cache memory hierarchy is important for **single-processor** systems to bridge the **speed gap** between processor and memory.
>
> It is important in a **multiprocessor** for **two respects**:
> 1. **The interconnect increases the latency to memory.**
> 2. **Since memory and interconnect are shared resources, their bandwidth is precious.** The cache hierarchy helps reduce the penalty of memory accesses and the **pressure on the memory and interconnect**.

Two distinct arguments, and the second is the new one.

**Reason 1 — latency.** On a uniprocessor, memory is ~200 cycles away (Lecture 2 §8). On a multiprocessor you add the interconnect on top: a hop or several, arbitration, possibly another node's memory controller. The gap the cache has to bridge got wider.

**Reason 2 — bandwidth, and this one is specific to multiprocessors.** The interconnect and memory are **shared**. Every miss you take consumes bandwidth that every *other* processor also needs. So a cache miss on a multiprocessor is not just slow for you — it **slows everyone down**.

> 📌 Put the two together and you get the central scaling fact: **a cache miss costs more and hurts
> others.** It is why miss rate matters disproportionately here, and why organization (d) from
> slide 5 — private L1 plus shared L2 — is what everyone builds.

---

## 11. The two big issues

The deck's closing slide, and the agenda for the rest of the course:

> **Two big ones**
> - **Cache coherence** (single memory location/variable)
> - **Memory consistency model** (multiple memory locations/variables)
>
> **Closely related — but often confused**
>
> **A (slightly) smaller one (very important nonetheless)**
> - **Synchronization support**
>   - Special instructions, e.g. **atomic operations**
>   - Advanced features such as **full/empty bits**

### The distinction, since he flags that it's often confused

The parenthetical is the entire definition, so hold onto it:

| | **Cache coherence** | **Memory consistency** |
|---|---|---|
| Scope | **ONE** memory location | **MULTIPLE** memory locations |
| Question | *Do all processors agree on the sequence of values this variable took?* | *Do all processors agree on the order in which operations to **different** variables happened?* |
| Fails when | P1 writes X; P2 still reads the old X from its stale cached copy | P1 writes A then B; P2 sees B's new value but A's old one |

**Coherence** is about keeping copies of **the same thing** in step. It is a hardware problem created by private caches — exactly the problem organization (c)/(d) introduced in §4.

**Consistency** is about the **ordering between different things**. Even with perfect coherence on every individual variable, a machine may reorder or delay writes, so another processor sees them in a different order than they were issued.

A quick example of the second:

```
    P1                      P2
    A = 1;                  while (flag == 0) ;   // wait
    flag = 1;               print(A);             // prints... 0?
```

Every variable here can be perfectly coherent and `A` can still print as 0 — if the hardware let `flag`'s write become visible before `A`'s. The memory consistency model is the contract that says what reorderings are allowed.

> 🔮 **This is where Lecture 4 slide 8's throwaway bullet lands:** *"need to know how the hardware
> interleaves accesses from different processors."* That was the seed; this is the harvest.

### Synchronization support

> - **Special instructions, e.g. atomic operations**
> - **Advanced features such as full/empty bits**

**[note]** An **atomic operation** completes indivisibly — no other processor can observe or interfere with it halfway through. This is the hardware that makes `LOCK`/`UNLOCK` from Lecture 4 §9 possible: recall `sum += mysum` is really read-add-write, and the lock exists to stop those three steps interleaving. Something has to implement the lock itself atomically, and ordinary loads and stores cannot — hence instructions like test-and-set, compare-and-swap, or load-linked/store-conditional.

**Full/empty bits** are a tag on each memory word marking it full or empty, so a read can block until a write has happened. It turns every memory location into a tiny synchronisation primitive — elegant, and expensive in bits, which is why only a few machines (notably the Tera/Cray MTA) ever did it.

This is **course outcome 4**: *"Implement synchronization methods for shared memory and message passing parallel computers."*

---

## 12. Summary sheet

### Definitions worth memorising

| Term | Definition |
|---|---|
| **TLP** | Multiple program counters; MIMD; tightly-coupled shared-memory multiprocessors |
| **Grain size** | Amount of computation assigned to each thread |
| **SMP** | Small number of cores sharing single memory with **uniform** latency |
| **DSM** | Memory distributed among processors; **non-uniform** latency (NUMA) |
| **UMA** | Equal latency from all processors; simpler software; lower peak performance |
| **NUMA** | Local faster than remote; harder software; higher peak **given proper placement** |
| **Direct network** | Router integrated with processor/memory package — "glueless MP" |
| **Indirect network** | Requires separate router chips |
| **Cache coherence** | Agreement about a **single** memory location |
| **Memory consistency** | Ordering across **multiple** memory locations |

### The four organizations

| | Organization | Key property |
|---|---|---|
| (a) | Dance-hall SMP | Not practical; no cache |
| (b) | SMP + shared L1 | No replication, but higher hit latency |
| (c) | SMP + private caches | ⚠️ creates the coherence problem |
| (d) | SMP + private caches + shared L2 | **Popular for CMPs** — what modern chips are |

### Networks

| | Shared medium (bus) | Point-to-point |
|---|---|---|
| Latency | low | high (many hops) |
| Bandwidth | low | higher per element |
| Scale | tens | **1000s** |
| Coherence | **simpler** → snooping | **complex** → directories |

### The through-lines from earlier lectures

- **Grain size** = parallelism granularity (L4 §9) = the Mortar curve (L3 §8)
- **"Contrary to the flat view of programming models"** = model/hardware mismatch (L4 slide 3)
- **Private caches ⇒ coherence** = replicated caches + write-back (L2 §9)
- **"How hardware interleaves accesses"** (L4 slide 8) = memory consistency, named here
- **Atomic operations** = what makes L4's `LOCK`/`UNLOCK` implementable

---

## 13. Self-test

<details>
<summary><b>1.</b> Give the three-bullet definition of thread-level parallelism.</summary>

- **Multiple program counters**
- Uses the **MIMD** model
- Targeted for **tightly-coupled shared-memory multiprocessors**

The first is the crispest: one PC = one instruction stream, so several PCs = several independent streams. ILP and DLP both live inside a single stream.
</details>

<details>
<summary><b>2.</b> What is grain size, and why can using threads for data-level parallelism backfire?</summary>

**Grain size** = the amount of computation assigned to each thread.

Threads carry real creation, scheduling and synchronisation overhead. If each thread only processes a few array elements, **the overheads outweigh the benefit** — you spend more time managing threads than computing. That is the Mortar curve from Lecture 3 and the $P_{opt}=\sqrt{T_{MM}/T_{init}}$ result from Lecture 4.
</details>

<details>
<summary><b>3.</b> Distinguish SMP from DSM.</summary>

**SMP** — small number of cores sharing a **single** memory with **uniform** memory latency.

**DSM** — memory **distributed** among processors, **non-uniform** access latency (NUMA), processors connected via direct (switched) and non-direct (multi-hop) interconnection networks.

Both present one global address space to the programmer; they differ in where memory physically sits.
</details>

<details>
<summary><b>4.</b> Of the four organizations on slide 5, which creates the cache coherence problem and which is used in modern chips?</summary>

**(c) SMP with private caches** creates it — private caches mean the same line can sit in several caches at once, so one processor's write leaves the others stale. Organization (b), with a shared L1, has **no replication** and therefore no coherence problem at all, but pays in hit latency.

**(d) SMP with private caches and shared level-2 cache** is what modern chips use — the slide says *"popular for chip multiprocessors (CMPs)."*
</details>

<details>
<summary><b>5.</b> Why does a bus make cache coherence simpler than a point-to-point network does?</summary>

A bus is a **broadcast medium** — every element sees every transaction automatically. So a cache can simply listen and react ("someone is writing X and I hold a copy, so I'll invalidate it"). That is **snooping**, and it is nearly free.

A point-to-point network has **no broadcast** — a message goes A→B and nobody else sees it. So something must explicitly track who holds copies: a **directory**. Directory protocols are substantially more complex.
</details>

<details>
<summary><b>6.</b> How can a bus be both "low latency" and "low bandwidth"?</summary>

They measure different things.

**Latency is low** because the destination is one hop away — the wire physically reaches everyone.
**Bandwidth is low** because that wire is **shared**: only one transfer at a time, and every added processor gets a smaller slice.

A point-to-point network inverts both — several links carry traffic simultaneously (high aggregate bandwidth) but messages traverse many hops (high latency).
</details>

<details>
<summary><b>7.</b> Hardware vs software shared memory — what is shared in each, and at what granularity?</summary>

**Hardware SM** shares **physical memory**. Hardware routes requests to the memory element holding the location and maintains cache coherence. Granularity: the **cache line** (~64 bytes).

**Software SM** shares **a higher abstraction** — virtual memory. The OS virtual memory manager distributes pages across machines and uses the network to deliver a page to the requesting machine. Granularity: the **page** (~4 KB).
</details>

<details>
<summary><b>8.</b> Is NUMA always faster than UMA?</summary>

**No.** NUMA has **higher peak performance — assuming proper data placement.** Get the placement wrong (every thread hammering one node's memory) and NUMA is *worse* than UMA, because you've added hops without adding locality.

UMA's compensating advantage: **simpler software, doesn't matter where you put data.**
</details>

<details>
<summary><b>9.</b> Give the two reasons caches matter in a multiprocessor beyond the single-processor reason.</summary>

1. **The interconnect increases the latency to memory** — the gap the cache must bridge is wider than on a uniprocessor.
2. **Memory and interconnect are shared resources, so their bandwidth is precious.** The cache reduces **pressure on the memory and interconnect** — meaning your miss doesn't just slow you down, it slows everyone down.
</details>

<details>
<summary><b>10.</b> Cache coherence vs memory consistency — state the difference in one line each.</summary>

**Cache coherence** concerns a **single** memory location: do all processors agree on the sequence of values that one variable took?

**Memory consistency** concerns **multiple** memory locations: do all processors agree on the order in which operations to *different* variables became visible?

The deck notes they are *"closely related but often confused."* Perfect coherence on every individual variable still permits a processor to observe two writes in the wrong order.
</details>

<details>
<summary><b>11.</b> Why is atomic-operation support necessary, given we already have locks?</summary>

Because **the lock itself has to be implemented**. Lecture 4 showed `sum += mysum` is really read-add-write, and that the lock prevents those steps interleaving — but acquiring the lock is the same kind of read-modify-write and faces the same race.

Ordinary loads and stores cannot do it. You need a hardware instruction that completes **indivisibly** — test-and-set, compare-and-swap, or load-linked/store-conditional. That is **course outcome 4**.
</details>

---

## References (slide 16)

- Michel Dubois, Murali Annavaram, and Per Stenström. 2012. *Parallel Computer Organization and Design.* Cambridge University Press, USA.
- David A. Patterson and John L. Hennessy. *Computer Architecture: A Quantitative Approach.* Morgan Kaufmann, 6th edition.
- Slides by Prof. Nima Honarmand (Stony Brook University).

---

*Previous: [Lecture 5 — Instruction-Level Parallelism](./05-ilp.md)*
