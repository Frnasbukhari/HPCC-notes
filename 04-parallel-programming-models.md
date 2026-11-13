# COE 502 — Lecture 4: Parallel Programming Models

**Parallel Processing Architectures** · KFUPM · Dr. Abdulaziz Tabbakh · slides: `04 - Parallel Programing Models.pdf`

> **How to read these notes**
> - This deck is an **overview**: later lectures cover each part in depth. Each section opens with **The idea**.
> - 💡 = analogy or extra help (not on the slides). ⚠️ = exam trap, or a spot where the slide is wrong or confusing.

---

## The whole lecture in 30 seconds

Every parallel program has to answer two questions:

| Question | Options |
|---|---|
| **How do I split the work?** | **Data-level**: same code on different pieces of data · **Function-level**: different functions, like an assembly line |
| **How do the workers share results?** | **Shared memory**: everyone reads and writes the same memory (threads) · **Message passing**: each has private memory and they send messages (processes) |

The deck shows one example (matrix multiplication) written both ways, and ends with why **more processors isn't always faster**.

---

## Contents

- [Key words](#key-words)

1. [Why parallel computing is difficult](#1-why-parallel-computing-is-difficult)
2. [The four core concepts](#2-the-four-core-concepts)
3. [Programming model abstractions](#3-programming-model-abstractions)
4. [The programmer's two jobs](#4-the-programmers-two-jobs)
5. [Data-level parallelism](#5-data-level-parallelism)
6. [Function-level parallelism](#6-function-level-parallelism)
7. [How threads exchange information](#7-how-threads-exchange-information)
8. [The example: matrix multiplication](#8-the-example-matrix-multiplication)
9. [Shared-memory version](#9-shared-memory-version)
10. [Message-passing version](#10-message-passing-version)
11. [Parallelism overhead](#11-parallelism-overhead)
12. [Summary sheet](#12-summary-sheet)
13. [Self-test](#13-self-test)

---

## Key words

| Word | Meaning |
|---|---|
| **Thread** | A worker in **shared memory**: shares one address space with the other threads |
| **Process** | A worker in **message passing**: has its own private memory |
| **Address space** | The range of memory a worker can see and use |
| **API** | Application Programming Interface: a library you call from C/C++/Fortran |
| **OpenMP / pthreads** | Shared-memory APIs (OpenMP = Open Multi-Processing; pthreads = POSIX threads) |
| **MPI** | Message-Passing Interface: the message-passing API |
| **pid** | The worker's ID number (0 … nproc − 1) |
| **Critical section** | Code only one thread may run at a time |
| **Lock / unlock** | Enter / leave a critical section |
| **Barrier** | Every thread waits until all threads have reached this point |
| **SEND / RECV** | Send a message to / receive a message from another process |
| **Reduction** | Combining many values into one (e.g. a sum): each worker makes a partial result, then they're combined |

---

## 1. Why parallel computing is difficult

**The idea:** our normal way of thinking about programs (one processor, one step at a time) no longer applies; there are several different machine types, often mixed together; tools can't hide this from you; and getting speed requires knowing the hardware.

> - Underlying machine model is **more complex than (and different from)** the familiar **"von Neumann"** model
> - There are **different machine models**, with their own challenges and opportunities
> - **Most systems are a mix** of more than one model
> - **Existing tools are not advanced enough** to shield the programmer from these difficulties
> - Parallel programming is usually about **high performance**, which requires **detailed knowledge of the underlying hardware**

- **Von Neumann model:** one processor, one memory, instructions run one at a time in order. Every normal language assumes this.
- **A mix:** a real cluster is shared memory **inside** each machine and message passing **between** machines, so you often write both styles.
- **Tools:** compilers automate a lot, but **not** parallelism. You still split the work yourself.

---

## 2. The four core concepts

**The idea:** every parallel system answers four questions: two about **control** (who does what, and in what order) and two about **data** (how data is named and moved).

| Aspect | Concept | Question |
|---|---|---|
| **Control** | **Work distribution** | How is parallelism created and expressed? |
| | **Synchronization** | How are the dependencies between parallel computations enforced? |
| **Data** | **Naming** | How is common data referenced by parallel computations? |
| | **Communication** | How is data passed between parallel computations? |

> ⚠️ Likely exam question: "What are the two aspects related to control?" → **work distribution and synchronization**.

### Model vs hardware

> - The **parallel programming model** answers these questions **as seen by the programmer**.
> - The **parallel hardware design** answers them **as implemented by the hardware**.
> - They **can be different**, but **to achieve high performance, they should match each other**.

**Example of a mismatch:** writing shared-memory-style code on a message-passing cluster. Software can fake shared memory, but every normal variable access secretly becomes a network message. The code is **correct but slow**.

---

## 3. Programming model abstractions

**The idea:** you don't learn a new language. You add a **library (API)** to C/C++ or Fortran.

> Defines how parallel computations can be expressed in a high-level programming language:
> - **Simple extensions through an API** of common languages such as C/C++ or Fortran
> - **Shared-memory:** OpenMP or pthreads
> - **Message-passing:** MPI
> - **Parallelizing compilers** translate a sequential program into parallel threads
> - Threads or processes must **communicate and coordinate** their activity (synchronization)

| | Shared memory | Message passing |
|---|---|---|
| API | OpenMP, pthreads | MPI |
| Workers are called | **threads** | **processes** |
| Memory | one shared address space | private per process |

(Parallelizing compilers only handle simple loops well, so in practice the programmer does the work.)

---

## 4. The programmer's two jobs

**The idea:** (1) **split** the work, and (2) **coordinate** the workers so the final answer is exactly what one processor would have produced.

> - **Partitioning:** work that can be parallelized must be partitioned to **threads (shared memory) / processes (message passing)**.
> - **Coordination:** work done by threads must be coordinated **as if the work was done by a single processor**. It involves two actions:
>   - **Synchronize** parallel threads with each other
>   - **Communicate** partial results between threads

> 📌 **"As if done by a single processor"** = the parallel answer must always equal the serial answer, no matter how the threads happen to be scheduled.

---

## 5. Data-level parallelism

**The idea:** **split the data**, and run the **same code** on each piece. It works when each element's calculation doesn't depend on another's. The bigger the data, the more parallelism you get.

> Computations of different data elements (e.g. in an array) are **independent of each other**:
> - Partition the data set; apply the same processing to each partition
> - **SPMD** (single program, multiple data), **not SIMD**
> - Massive parallelism
> - Expressed in **loops with no loop-carried dependency**
> - **As the problem size is scaled up, more parallelism can often be found**

> 💡 Grading 400 exam papers with 4 teachers: each takes 100 papers and uses the same marking scheme. More papers → you could use more teachers.

### SPMD vs SIMD (they look alike, but aren't)

| | **SIMD** | **SPMD** |
|---|---|---|
| Is a… | **hardware** design | **programming** style |
| Same… | **instruction**, at the same moment (lockstep) | **program**, but each worker runs it independently |
| Can workers take different branches? | ❌ no | ✅ yes (each has its own program counter) |
| Example | vector unit, GPU | the matrix-multiply code in §9 (each thread uses its own `pid`) |

### Loop-carried dependency

= one iteration needs a value produced by an **earlier** iteration. **Test:** does any iteration read something another iteration writes?

```c
for (i = 0; i < N; i++)  a[i] = b[i] * 2;      // ✅ independent → parallel
for (i = 1; i < N; i++)  a[i] = a[i-1] + 1;    // ❌ needs the previous iteration
for (i = 0; i < N; i++)  sum += a[i];          // ❌ all iterations update sum → use a reduction (§8)
```

---

## 6. Function-level parallelism

**The idea:** give **different functions** to different processors and pass the data through them like an **assembly line**. The parallelism is limited to the number of functions, and it **doesn't grow** with the data.

> - **Independent functions are executed on different processors**
> - Usually used in **streaming applications**: the functions form a **function pipeline** the data stream flows through
> - **More complex, limited parallelism**
> - **Combine both:** function + data parallelism

```
frames → [ decode ] → [ colour fix ] → [ scale ] → [ encode ] → out
            P0            P1              P2          P3
```

- Like the instruction pipeline (Lecture 2): **throughput goes up**, and the **slowest stage limits everyone**.
- 4 stages = at most 4 processors busy, whether the video is 1 minute or 1 hour long.

| | Data-level | Function-level |
|---|---|---|
| Split by | data | functions |
| Parallelism | as many as there are data elements (**massive**) | as many as there are functions (**limited**) |
| Grows with problem size? | ✅ yes | ❌ no |

---

## 7. How threads exchange information

**The idea:** in **shared memory**, one thread writes a variable and another reads it, so communication is **hidden** in normal loads and stores. In **message passing**, a process must **explicitly send** to a named receiver. Either way: **never assume one processor is faster than another**.

| | **Shared memory** | **Message passing** |
|---|---|---|
| How | T1 **stores** to a shared variable, T2 **loads** it | T1 **SENDs**, T2 **RECVs** |
| Communication | **implicit** (normal loads/stores) | **explicit** (send/receive statements in the code) |
| Does T1 need to know T2? | no | yes: the **destination** and **what to send** |
| Must handle | **synchronization and ordering**, and **how hardware interleaves accesses** | — |

> 💡 **Shared memory** = a whiteboard in a shared office: you write, others read it. **Message passing** = email: nobody sees your work unless you send it to them by name.

The "how hardware interleaves accesses" point is covered in Lecture 7 (memory consistency).

> ### ⛔ No assumptions on the relative speed of processors
> You may **never** reason "T1 will have finished by the time T2 gets here". Threads can be paused by the OS, stalled on cache misses, or slowed by other work. **Correctness must come from explicit synchronization (locks, barriers, messages), never from timing.** Code that "always worked in testing" can still be wrong.

---

## 8. The example: matrix multiplication

**The idea:** C = A × B, then add up all elements of C. Computing C is **fully parallel**; adding them into one `sum` is **not**. The fix is a **reduction**: each worker keeps its own `mysum`, and the partial sums are combined at the end.

```c
1.  sum = 0;
2.  for (i=0; i<N; i++)
3.    for (j=0; j<N; j++){
4.      C[i,j] = 0;
5.      for (k=0; k<N; k++)
6.        C[i,j] = C[i,j] + A[i,k]*B[k,j];   // ← boxed on the slide
7.      sum += C[i,j];                        // ← boxed on the slide
8.    }
```

`C[i,j]` = row *i* of A times column *j* of B (multiply pairs, add them up).

| Line | Dependency? | So… |
|---|---|---|
| **6** | ❌ none: each `C[i,j]` uses only A and B | all N² elements can be computed in parallel |
| **7** | ✅ every iteration reads and writes the **same** `sum` | can't simply be split → **reduction** |

> 📌 That's the shape of most parallel problems: lots of independent work, plus a small combining step that needs care.

---

## 9. Shared-memory version

**The idea:** each thread takes a block of rows of C. The multiply loops are **unchanged**. Then two tools make the final sum correct: a **barrier** (wait until everyone has finished multiplying) and a **lock** (add to `sum` one thread at a time).

```c
/* A, B, C, BAR, LV and sum are shared; all other variables are private */
1a  low   = pid*N/nproc;      /* pid = 0 ... nproc-1      */
1b  hi    = low + N/nproc;    /* this thread's rows of A  */
1c  mysum = 0; sum = 0;
2   for (i=low; i<hi; i++)
3     for (j=0; j<N; j++){
4       C[i,j] = 0;
5       for (k=0; k<N; k++)
6         C[i,j] = C[i,j] + A[i,k]*B[k,j];
7       mysum += C[i,j];
8     }
9   BARRIER(BAR);
10  LOCK(LV);
11    sum += mysum;
12  UNLOCK(LV);
```

### Lines 1a–1b: splitting the rows

Every thread runs **the same code** and works out its own rows from its `pid` (**SPMD**). For N = 100, nproc = 4:

| pid | low | hi | rows |
|---|---|---|---|
| 0 | 0 | 25 | 0–24 |
| 1 | 25 | 50 | 25–49 |
| 2 | 50 | 75 | 50–74 |
| 3 | 75 | 100 | 75–99 |

### Lines 2–8: unchanged

A, B and C are shared, so every thread just **reads them with normal loads and stores**. Nothing has to be sent.

### Line 10–12: the lock (one at a time)

`sum += mysum` is really three steps: **read** sum → **add** → **write** sum. Two threads can interleave them:

| Step | Thread 1 (mysum = 50) | Thread 2 (mysum = 80) | sum |
|---|---|---|---|
| 1 | reads 100 | | 100 |
| 2 | | reads 100 | 100 |
| 3 | writes 150 | | 150 |
| 4 | | writes 180 | **180 ❌** (should be 230) |

The lock makes the three steps happen **without interruption**.

### Line 9: the barrier (wait for everyone)

Without it, a fast thread could add its `mysum` while a slow thread is still multiplying.

> ⚠️ **Lock ≠ barrier. You need both.**
> - **Lock** = **who** may go in: one at a time.
> - **Barrier** = **when** anyone may go on: only after all have arrived.
>
> 💡 Lock = the key to a single bathroom. Barrier = a tour group waiting at the bus until everyone is back.

### Granularity

> **Parallelism granularity** = the amount of work a thread does in parallel with others. **For a fixed problem size, granularity decreases as the number of processors increases.**

N = 1000 rows: 4 threads → 250 rows each (coordination is negligible); 1000 threads → 1 row each (coordination costs about as much as the work). §11 turns this into a formula.

---

## 10. Message-passing version

**The idea:** the computation is the same, but now **process 0 must send the data out** and **collect the results back** by hand. In return, you **don't need locks or barriers**: the messages themselves provide the ordering.

### ① Distribute the input data

```c
1a myN = N/nproc;
1b if (pid == 0) {
1c   for (i=1; i<nproc; i++) {
1d     k = i*N/nproc;
1g     SEND(&A[k][0], myN*N*sizeof(float), i, IN1);   // rows k.. of A to process i
1h     SEND(&B[0][0], N*N*sizeof(float), i, IN2);     // ALL of B to process i
       }
   } else {
1j   RECV(&A[0][0], myN*N*sizeof(float), 0, IN1);
1k   RECV(&B[0][0], N*N*sizeof(float), 0, IN2);
   }
```
(The slide's braces are a bit loose; read it as: process 0 runs the SEND loop, everyone else runs the RECVs.)

**Format:** `SEND(address, size in bytes, destination, tag)` / `RECV(address, size in bytes, source, tag)`. The **tag** (IN1, IN2, SUM, RES) labels the kind of message, so A and B don't get mixed up.

**Why only part of A, but all of B?** To compute its rows of C, a process needs **its rows of A** but **every column of B**. So B is sent in full to every process. (In shared memory, B just sits in memory for free.)

### ② Compute (same loops)

```c
mysum = 0;
for (i=0; i<myN; i++)          // local rows start at 0
  for (j=0; j<N; j++){
    C[i,j] = 0;
    for (k=0; k<N; k++)
      C[i,j] = C[i,j] + A[i,k]*B[k,j];
    mysum += C[i,j];
  }
```

**Why `i = 0` and not `low`?** Each process stores its rows **starting at its own row 0**. Global row *k* becomes local row 0. For N = 100, nproc = 4:

| Process | Global rows of A it handles | Stored locally as rows |
|---|---|---|
| 0 | 0–24 (keeps its own) | 0–24 |
| 1 | 25–49 | 0–24 |
| 2 | 50–74 | 0–24 |
| 3 | 75–99 | 0–24 |

> ⚠️ Older versions of these notes drew this mapping shifted by one (rows 0–24 → p1, …). The code sends global rows `k = i*N/nproc` to process *i*, so **process i gets rows i×25 to i×25+24**, and process 0 keeps rows 0–24.

### ③ Collect the result

```c
9  if (pid == 0) {
10   sum = mysum;
11   for (i=1; i<nproc; i++) {
12     RECV(&mysum, sizeof(float), i, SUM);
13     sum += mysum;
14   }
15   for (i=1; i<nproc; i++) {
       k = i*N/nproc;
16     RECV(&C[k][0], myN*N*sizeof(float), i, RES);   // local rows → global row k
20   }
21 } else {
22   SEND(&mysum, sizeof(float), 0, SUM);
23   SEND(&C[0][0], myN*N*sizeof(float), 0, RES);
24 }
```

### Why no LOCK and no BARRIER?

| Not needed | Because |
|---|---|
| **Lock** | `sum` is in **process 0's private memory**; only it touches `sum`, and it adds the partial sums **one at a time** in a loop |
| **Barrier** | a **RECV waits until the data arrives**, so process 0 can't finish collecting before everyone has sent |

> 📌 **The trade:**
> - **Message passing:** you write all the **data movement**; synchronization comes **free**.
> - **Shared memory:** data movement is **free**; you write all the **synchronization**.

### Slide 14: how SEND/RECV work

- Each node has its **own processor and memory**; there's **no shared address space**.
- Data must be **explicitly distributed**, and results **collected at the end**.
- **SEND** copies data from the sender's memory **to a buffer at the receiver**; **RECV** copies it **from the buffer into local memory**.
- **Synchronous** SEND/RECV **block** (wait) until both sides confirm the exchange; **asynchronous** ones don't wait.
- Implemented by a **combination of hardware and software**.

⚠️ With synchronous SEND, if two processes both SEND before either RECVs, both wait forever: **deadlock**.

| Typical bug | Shared memory | Message passing |
|---|---|---|
| Forgot synchronization | **wrong answer**, silently | — |
| Forgot / mismatched a SEND or RECV | — | program **hangs** |

---

## 11. Parallelism overhead

**The idea:** more processors split the work into smaller pieces, but **starting each thread costs time, one after another**. At some point, adding processors makes the program **slower**. The best number is a square root.

> It takes **T_MM / P** to multiply on **P** cores, and **T_init** to start each thread. **Thread initiation is sequential.**

$$T = P \times T_{init} + \frac{T_{MM}}{P}$$

| Term | As P grows |
|---|---|
| $T_{MM}/P$ (the work, shared out) | gets **smaller** |
| $P \times T_{init}$ (starting the threads) | gets **bigger** |

The total falls, reaches a minimum, then rises. Set the derivative to zero:

$$\frac{dT}{dP} = T_{init} - \frac{T_{MM}}{P^2} = 0 \quad\Rightarrow\quad \boxed{P = \sqrt{\frac{T_{MM}}{T_{init}}}}$$

> If T_MM is **100 times** T_init → P = √100 = **10 cores**.

**Worked example:** T_MM = 400 ms, T_init = 4 ms.
- P = √(400 / 4) = √100 = **10**
- T = 10 × 4 + 400 / 10 = 40 + 40 = **80 ms**, so speedup = 400 / 80 = **5×**
- At the optimum, the two terms are **equal** (40 = 40).

**Check with more cores** (P = 40): T = 40 × 4 + 400 / 40 = 160 + 10 = **170 ms**. That's slower than with 10.

---

## 12. Summary sheet

| | Shared memory | Message passing |
|---|---|---|
| Workers | threads | processes |
| Memory | one shared address space | private per process |
| Communication | implicit (loads/stores) | explicit (SEND/RECV) |
| Synchronization | **your job** (locks, barriers) | **free** (comes with messages) |
| Data distribution | **free** | **your job** |
| API | OpenMP, pthreads | MPI |
| Typical bug | wrong answer (race) | hang (deadlock) |

| Four core concepts | |
|---|---|
| **Control** | work distribution · synchronization |
| **Data** | naming · communication |

| | Data-level | Function-level |
|---|---|---|
| Split by | data | functions (pipeline) |
| Parallelism | massive, grows with the problem | limited, doesn't grow |
| Style | SPMD, loops with no loop-carried dependency | streaming |

**Formulas:** $T = P \times T_{init} + T_{MM}/P$ · $P_{opt} = \sqrt{T_{MM}/T_{init}}$

**Must remember:**
- ⛔ **No assumptions on the relative speed of processors.**
- **Lock** = one at a time · **Barrier** = wait for all. The matrix multiply needs **both**.
- **SPMD ≠ SIMD** (programming style vs hardware).
- **Model and hardware should match** for high performance.
- **Granularity drops** as processors increase (fixed problem size).

---

## 13. Self-test

<details>
<summary><b>1.</b> Name the two aspects of a parallel system and the two concepts under each.</summary>

**Control:** work distribution, synchronization. **Data:** naming, communication.
</details>

<details>
<summary><b>2.</b> Can the programming model differ from the hardware? Should it?</summary>

It **can** (what the programmer sees may be far from the implementation), but **to achieve high performance they should match**. Otherwise you get correct but slow code.
</details>

<details>
<summary><b>3.</b> SPMD vs SIMD?</summary>

**SIMD** = hardware: one instruction on many data in lockstep, no divergence. **SPMD** = programming style: every worker runs the same program independently and can take different branches.
</details>

<details>
<summary><b>4.</b> Which loops parallelise directly?</summary>

```c
(a) for (i=0;i<N;i++) a[i] = b[i] + c[i];
(b) for (i=1;i<N;i++) a[i] = a[i-1] * 2;
(c) for (i=0;i<N;i++) total += a[i];
(d) for (i=0;i<N;i++) a[i] = a[i] * 2;
```
(a) ✅ · (b) ❌ needs the previous iteration · (c) ❌ directly, but ✅ as a **reduction** · (d) ✅
</details>

<details>
<summary><b>5.</b> Why does function-level parallelism give "limited parallelism"?</summary>

The number of parallel workers equals the number of **functions** (pipeline stages), and it **doesn't grow** when the data grows.
</details>

<details>
<summary><b>6.</b> In the shared-memory version, why are both a barrier and a lock needed?</summary>

**Lock:** `sum += mysum` is read-add-write; without the lock, two threads can interleave and lose an update. **Barrier:** threads finish at different times; without it, partial sums could be added before every thread has finished multiplying.
</details>

<details>
<summary><b>7.</b> In the message-passing version, why does each process get part of A but all of B?</summary>

Each row of C needs **one row of A** and **every column of B**. So A can be split by rows, but every process needs the whole of B.
</details>

<details>
<summary><b>8.</b> Why does the message-passing version need no lock or barrier?</summary>

**No lock:** `sum` is private to process 0, which adds the partial sums one at a time. **No barrier:** RECV waits for the data, so process 0 can't finish before everyone has sent.
</details>

<details>
<summary><b>9.</b> T_MM = 900 ms, T_init = 9 ms. Optimal P, time, and speedup?</summary>

P = √(900 / 9) = √100 = **10**. T = 10 × 9 + 900 / 10 = 90 + 90 = **180 ms**. Speedup = 900 / 180 = **5×**.
</details>

<details>
<summary><b>10.</b> What does "no assumptions on the relative speed of processors" mean in practice?</summary>

You can't rely on one thread finishing before another. Ordering must be enforced with **explicit synchronization** (locks, barriers, messages), never with timing.
</details>

---

*Previous: [Lecture 3 — Performance](./03-performance.md)* · *Next: [Lecture 5 — Instruction-Level Parallelism](./05-instruction-level-parallelism.md)*
