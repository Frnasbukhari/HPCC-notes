# COE 502 — Lecture 7 Quiz Cram Sheet

Coherence · Synchronization · Consistency. Uses the slides' notation and wording.

> **What to expect** (based on Quiz 1 and Assignment 2):
> - **Multiple choice** (about 2 pts each): definitions and one-line facts.
> - **One worked problem**, most likely one of: a **MESI/MSI state table**, a **directory table**, a **lock / barrier code question**, or an **SC vs TSO "is this result possible?"** question.
> - Your Assignment 2 answers were all correct, so use them as model answers too.

---

## 1. Must-know (if you only read one section)

1. **Coherence = one location. Consistency = order across different locations.** Coherence alone never rules out a reordering between two different variables.
2. **SWMR** (single writer, multiple readers): at any moment a block has **one writer OR many readers**, never both.
3. **MSI:** M = only copy, dirty, writable · S = read-only, clean · I = no valid copy.
4. **MESI adds E** = only copy, **clean** → can write **silently** (E → M with no bus transaction). Saves the BusUpgr on "read then write".
5. **MOESI adds O** = **shared but dirty**. The owner supplies the data and writes back later. Saves the **write-back on M → S**.
6. **Snooping:** the **bus is the point of serialization** (everyone sees requests in the same order). Doesn't scale beyond about a dozen cores.
7. **Directory:** each block has a **home node (H)** that keeps a **presence-bit vector + dirty bit** and contacts **only the sharers**. Size grows as **m × n²**.
8. **Naïve lock (load, test, store) fails:** two threads can both read 0 before either writes 1. Fix: an **atomic RMW** (T&S, SWAP, CAS, F&A) or **LL/SC**.
9. **SC:** one total order of all operations, and each thread keeps its **program order**. Nothing is reordered.
10. **TSO (x86, SPARC) relaxes only W → R** (a load can pass an earlier store, because of the store buffer). So **Dekker / store-buffering can give (0, 0)**, but the **flag pattern still works**.

---

## 2. Vocabulary

| Term | Meaning |
|---|---|
| **Coherence** | All processors see writes to **one** location in the same order; a read returns the most recent write |
| **Consistency model** | Specification of allowed orders of memory operations to **different** locations |
| **SWMR** | Single-writer / multiple-reader invariant |
| **Write-back** | Memory is updated only when a dirty block leaves the cache |
| **Stale / Current** | Memory's copy is out of date / up to date |
| **PrRd / PrWr** | My processor reads / writes |
| **BusRd** | Request a copy, **no** intent to modify |
| **BusRdX** | Request a copy **with** intent to modify (others invalidate) |
| **BusUpgr** | I'm in S and already have the data; others invalidate |
| **Flush** | The cache in M sends the block to the requester **and memory** |
| **Shared line** | A bus wire other caches assert on BusRd if they hold the block → requester goes to **S**, not E |
| **Home (H) / Requester (R) / Dirty (D) / Sharer (S)** | Directory roles |
| **Presence-flag vector** | n bits per block (p[i] = 1 → processor i has a copy) + dirty bit d |
| **True sharing miss** | Data was really changed by another core |
| **False sharing miss** | **Reading an unmodified word in an invalidated block** |
| **Critical section** | Code protected by lock/unlock; one thread at a time |
| **Barrier** | All threads must arrive before any continues |
| **Point-to-point (flag)** | Producer sets a flag, consumer waits on it; no lock needed (only one writer) |
| **RMW** | Read-modify-write, done **atomically** |
| **T&S** | Read the location, set it to 1 atomically; **R1 = 0 → acquired**, R1 = 1 → failed |
| **LL / SC** | Load-locked monitors the line; store-conditional **succeeds only if nobody wrote it since the LL**; on failure it doesn't write and sets R1 = 0 |
| **Store buffer** | Holds a core's stores so later loads don't wait; causes W → R reordering |
| **Store atomicity** | A write becomes visible to **all** processors at the same time |
| **Fence** | Orders instructions before it before instructions after it |
| **Acquire / Release** | Acquire: later operations can't move above it. Release: earlier operations can't move below it |

---

## 3. Tricks and traps

| Trap | Remember |
|---|---|
| "Can coherence prevent the (0, 0) result?" | **No**: different locations. You need a fence, an RMW, or acquire/release |
| Read miss when **another cache already has it** (MESI) | requester goes to **S**, not E (the shared line is asserted) |
| Read miss when **nobody** has it (MESI) | **E**, data from memory |
| Cache in **E** sees a BusRd | E → **S** (memory is already current) |
| Cache in **M** sees a BusRd (MSI/MESI) | **Flush** to requester **and memory**, M → **S**, memory now **current** |
| Cache in **M** sees a BusRdX | **Flush**, M → **I** |
| Write while in **S** | **BusUpgr** (not BusRdX: you already have the data) |
| Write while in **I** | **BusRdX** |
| Write while in **E** | **no bus transaction** |
| Memory column | **Stale** whenever some cache is in **M** (or **O**); otherwise **Current** |
| BusUpgr arriving at a cache in M | **impossible** (SWMR) |
| Bigger cache → coherence misses? | **No change** (the data was invalidated, not evicted) |
| More processors → ? | **True sharing misses grow** |
| Bigger block → ? | Fewer compulsory misses, **more false sharing** |
| Directory size | **m × n²**: double n → **4×** directory |
| Barrier counter | **incrementing** needs a lock/RMW; **reading** it in the while loop doesn't |
| Barrier reuse | the count must be **reset** safely, or fast threads get lost and others deadlock |
| T&S traffic | every T&S is a **write** → invalidates everyone. **Test-and-test&set** spins on ordinary loads (hits in S) → no traffic |
| LL/SC | detection uses the **coherence invalidation** (it clears the LL-bit) |
| Busy waiting vs blocking | spin if **scheduling overhead > expected wait** or **no other task** |
| Update vs invalidate protocol | update wins **only if every written value is consumed** |
| TSO keeps | R→R, R→W, W→W (drops only **W→R**) |
| PSO also drops | **W→W** |
| Weak ordering | drops all ordinary orderings; **SYNCH** = two-way wall |
| Release consistency | like WO, but acquire and release are **one-way** walls |

---

## 4. Recipes for the worked problems

### A. Filling a MESI state table

For each operation, ask **who has the block right now**:

**Read by Pk**
| Others hold it in… | Pk gets | Others become | Transaction | Memory |
|---|---|---|---|---|
| nobody | **E** | — | BusRd (from memory) | Current |
| E or S | **S** | E → S | BusRd (shared line) | Current |
| M | **S** | M → **S** | BusRd + **Flush** (cache-to-cache) | **Current** (updated) |

**Write by Pk**
| Pk is in… | Transaction | Others become | Memory |
|---|---|---|---|
| M | none | — | Stale |
| E | **none (silent E → M)** | — | Stale |
| S | **BusUpgr** | all → I | Stale |
| I | **BusRdX** (+ **Flush** if someone was in M) | all → I | Stale |

(MSI is the same, without E: a read miss with nobody else holding the block → **S**.)

### B. Filling a directory (MSI) table

Every request goes to **H** first.

| Request | Directory before | Messages | Directory after |
|---|---|---|---|
| Read | I or S{…} | req R→H, reply(data) H→R | S{… + R} |
| Read | M{X} | req R→H, **forward H→X**, reply X→H (X: M→S, memory updated), reply H→R | S{X, R} |
| Write | S{…} | req R→H, **inv H→each sharer**, **ack sharer→H**, reply H→R | M{R} |
| Write | M{X} | req R→H, forward H→X, reply X→H (X: M→I), reply H→R | M{R} |

A remote dirty read = **4 hops**. Optimised: X replies **directly** to R, then notifies H → **3 hops** on the critical path.

### C. "Is this result possible under SC / TSO?" (the cycle method)

1. Write the **program-order** edges for each thread (e.g. S0(A) → L0(B)).
2. Add an edge for each load, based on what it returned:
   - read the **new** value → **store → load**
   - read the **old** value (0) → **load → store** (the load happened before the store)
3. **Cycle → impossible under SC.** No cycle → possible.
4. **TSO:** delete program-order edges that are **store → load** (different addresses), then check for a cycle again.
5. **PSO:** also delete **store → store**. **WO / RC:** delete all ordinary edges unless there's a fence or sync between them.

---

## 5. Practice problems with answers

### P1. MESI table (3 processors, write-back, x starts uncached, memory Current)

a. P0 writes x · b. P1 reads x · c. P1 writes x · d. P2 reads x · e. P0 reads x · f. P2 writes x

<details><summary>Answer</summary>

| Operation | P0 | P1 | P2 | Mem | Transaction |
|---|---|---|---|---|---|
| P0 write | M | I | I | Stale | BusRdX (data from memory) |
| P1 read | S | S | I | Current | BusRd + Flush, cache-to-cache P0 → P1 (memory updated) |
| P1 write | I | M | I | Stale | BusUpgr (P0 invalidates) |
| P2 read | I | S | S | Current | BusRd + Flush, cache-to-cache P1 → P2 |
| P0 read | S | S | S | Current | BusRd, **shared line asserted → S, not E** |
| P2 write | I | I | M | Stale | BusUpgr (P0, P1 invalidate) |

**Trap in step e:** P0's read miss gives **S**, because P1 and P2 already hold copies.
</details>

### P2. Same sequence under MOESI: what changes?

<details><summary>Answer</summary>

| Operation | P0 | P1 | P2 | Mem | Note |
|---|---|---|---|---|---|
| P0 write | M | I | I | Stale | BusRdX |
| P1 read | **O** | S | I | **Stale** | P0 supplies the data, **no write-back** |
| P1 write | I | M | I | Stale | BusUpgr |
| P2 read | I | **O** | S | **Stale** | P1 supplies |
| P0 read | S | O | S | Stale | the owner P1 supplies |
| P2 write | I | I | M | Stale | BusUpgr (O and S → I) |

Memory is **never written**; the O state saved two write-backs.
</details>

### P3. Bus transactions for private data

P0 alone runs: read x, write x, write x (no other core ever touches x). How many bus transactions under MSI and under MESI?

<details><summary>Answer</summary>

- **MSI:** BusRd (I → S) + BusUpgr (S → M) = **2**; the second write hits in M.
- **MESI:** BusRd (I → **E**, nobody else has it) + silent E → M = **1**.

This is the slide's "read miss followed by write causes two bus accesses" problem that E solves.
</details>

### P4. Directory-based MSI table

a. P1 reads x · b. P2 reads x · c. P0 writes x · d. P1 reads x · e. P2 writes x. Give the cache states, directory state, memory, and messages. How many hops does step d take, and with the optimization?

<details><summary>Answer</summary>

| Operation | P0 | P1 | P2 | Dir | Mem | Messages |
|---|---|---|---|---|---|---|
| P1 read | I | S | I | S{P1} | Current | req P1→H, reply H→P1 |
| P2 read | I | S | S | S{P1,P2} | Current | req P2→H, reply H→P2 |
| P0 write | M | I | I | M{P0} | Stale | req P0→H, inv H→P1 & H→P2, ack P1→H & P2→H, reply H→P0 |
| P1 read | S | S | I | S{P0,P1} | Current | req P1→H, forward H→P0, reply P0→H (P0: M→S), reply H→P1 |
| P2 write | I | I | M | M{P2} | Stale | req P2→H, inv H→P0 & H→P1, acks →H, reply H→P2 |

Step d = **4 hops** (R→H→D→H→R). Optimised: **3 hops** (R→H→D→R), then D (or R) notifies H off the critical path.
</details>

### P5. Directory overhead

n = 64 nodes, block = 64 bytes (b = 512 bits). (a) Full presence-vector overhead? (b) Limited pointers with i = 2? (c) What happens to a full directory's size if n doubles?

<details><summary>Answer</summary>

(a) n / (b + n) = 64 / (512 + 64) = 64 / 576 ≈ **11.1%**
(b) i log n / (b + i log n) = 2 × 6 / (512 + 12) = 12 / 524 ≈ **2.3%** (log₂ 64 = 6)
(c) Size = m × n², so doubling n → **4×** the directory.
</details>

### P6. Ordering: is the result possible?

```
Initially A = B = 0
P0            P1
A = 1         r1 = B
B = 1         r2 = A
```
Can **r1 = 1, r2 = 0** happen under (a) SC, (b) TSO, (c) PSO?

<details><summary>Answer</summary>

Edges:
- Program order: S0(A) → S0(B), L1(B) → L1(A)
- r1 = 1 (new B): S0(B) → L1(B)
- r2 = 0 (old A): L1(A) → S0(A)

Chain: S0(A) → S0(B) → L1(B) → L1(A) → S0(A) = **cycle**.

(a) **SC: impossible.**
(b) **TSO: impossible.** TSO only drops store → load; here the program-order edges are store → store and load → load, which TSO keeps. (This is the flag pattern, which works on x86.)
(c) **PSO: possible.** PSO also drops store → store, so B = 1 can become visible before A = 1, and the cycle breaks.
</details>

### P7. Store buffering (Dekker)

```
P0            P1
A = 1         B = 1
r0 = B        r1 = A
```
Can r0 = r1 = 0 happen under SC? Under TSO? Does coherence prevent it? How do you fix it?

<details><summary>Answer</summary>

- **SC: no.** S0(A) → L0(B) → S1(B) → L1(A) → S0(A) is a cycle.
- **TSO: yes.** Store → load isn't enforced: A = 1 waits in P0's **store buffer** and B = 1 in P1's, so both loads read 0 from the cache.
- **Coherence: no**: A and B are different locations.
- **Fix:** a **fence** between the store and the load on **both** processors (or make the store or the load an **RMW**).
</details>

### P8. Fix the flag pattern for a relaxed model

```
P0                  P1
data = 42;          while (flag == 0);
flag = 1;           r = data;
```

<details><summary>Answer</summary>

With **release / acquire** (slide 69, solution 3):
```
P0                              P1
data = 42;                      while (Load.acquire(flag) == 0);
Store.release(flag, 1);         r = data;          // r = 42
```
- **Release:** all earlier operations (data = 42) are globally performed **before** the flag store.
- **Acquire:** later operations (r = data) can't start until the flag load is done.

With fences instead: `data = 42; fence; flag = 1;` and `while (flag == 0); fence; r = data;`.
Coherence alone isn't enough: it orders writes to **one** location, but data and flag are **two** locations.
</details>

### P9. Locks

(a) Why does `while (lock != 0); lock = 1;` fail? (b) Write the acquire with T&S and say which value means success. (c) Write T&S using LL/SC.

<details><summary>Answer</summary>

(a) T0 reads lock = 0 and exits the loop; T1 reads lock = 0 and exits the loop; T0 sets lock = 1; T1 sets lock = 1. **Both enter.** The gap between test and set isn't atomic.

(b)
```
Lock:   T&S  R1, lock
        BNEZ R1, Lock     ; R1 = 0 → acquired, R1 = 1 → retry
        RET
Unlock: SW   R0, lock     ; release by storing 0
        RET
```
(c)
```
T&S:    ADDI R1, R0, 1
        LL   Rx, lock
        SC   R1, lock
        BEQZ R1, T&S      ; SC failed (another write since LL) → retry
        RET
```
If SC fails, it **doesn't write** and sets R1 = 0, so the code retries. Polling with ordinary loads first (test-and-test&set) spins on the **cached S copy**, which causes no bus traffic.
</details>

### P10. Coherence or consistency?

Label each as a **coherence** or a **consistency** problem:
1. P1 writes X = 5, but P2 keeps reading X = 3 from its own cache forever.
2. P1 writes data, then flag; P2 sees the new flag but the old data.
3. P1 and P2 write X; P3 sees 1 then 2, P4 sees 2 then 1.
4. P3 sees A's write before B's, while P4 sees B's write before A's.

<details><summary>Answer</summary>

1. **Coherence** (one location, stale copy).
2. **Consistency** (two locations, order).
3. **Coherence** (writes to **one** location must be seen in the same order by all).
4. **Consistency** (two locations; this needs store atomicity).
</details>
