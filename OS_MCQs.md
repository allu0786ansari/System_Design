# Operating Systems — 50 MCQs for NatWest SDE Assessment Prep

Organized by subtopic. Format: Question → 4 options → **Answer** → short explanation.

---

## Process Management (Q1–10)

**Q1.** A process in the "waiting" (blocked) state is:
A) Currently executing on the CPU  B) Waiting for an event (e.g., I/O completion) to occur  C) Ready to run but not scheduled  D) Terminated
**Answer: B**

**Q2.** The Process Control Block (PCB) does NOT typically store:
A) Process ID  B) Program counter  C) CPU registers  D) Source code of the program
**Answer: D** — PCB stores process state/metadata, not the actual source code.

**Q3.** Context switching refers to:
A) Switching between two threads of the same process only  B) Saving the state of a running process and loading the state of another  C) Switching the CPU's clock speed  D) Compiling a program
**Answer: B**

**Q4.** A zombie process is one that:
A) Is still actively running  B) Has completed execution but still has an entry in the process table (parent hasn't read its exit status)  C) Has infinite loop  D) Uses zero memory
**Answer: B**

**Q5.** An orphan process is a process:
A) Whose parent has terminated before it, and it gets adopted by init/systemd  B) That never had a parent  C) That has no PID  D) That is always killed immediately
**Answer: A**

**Q6.** `fork()` in a Unix-like OS:
A) Creates a new thread within the same process  B) Creates a new child process, a near-copy of the parent  C) Terminates the current process  D) Switches to kernel mode
**Answer: B**

**Q7.** The main difference between a process and a thread is:
A) Threads share the same address space within a process; processes have separate address spaces  B) Processes are faster to create than threads  C) Threads cannot run concurrently  D) There is no difference
**Answer: A**

**Q8.** Which of these is NOT a valid process state in the typical 5-state model?
A) New  B) Ready  C) Running  D) Compiling
**Answer: D** — Standard states: New, Ready, Running, Waiting/Blocked, Terminated.

**Q9.** Inter-Process Communication (IPC) mechanisms include:
A) Pipes, shared memory, message queues  B) Only global variables  C) Only function calls  D) Compilation flags
**Answer: A**

**Q10.** A race condition occurs when:
A) Multiple processes run at different priorities  B) The outcome of execution depends on the timing/interleaving of uncontrolled events (e.g., unsynchronized shared data access)  C) A process runs out of memory  D) The CPU clock is too slow
**Answer: B**

---

## CPU Scheduling (Q11–20)

**Q11.** Which scheduling algorithm can cause starvation of long processes?
A) FCFS  B) Shortest Job First (SJF), preemptive/non-preemptive  C) Round Robin  D) FCFS with aging
**Answer: B** — Short jobs keep getting prioritized, so long jobs may starve indefinitely.

**Q12.** Round Robin scheduling uses:
A) A fixed time quantum for each process, cycling through the ready queue  B) Priority-only scheduling  C) First-come-first-served with no interruption  D) Shortest job first
**Answer: A**

**Q13.** A very small time quantum in Round Robin scheduling leads to:
A) Better throughput always  B) High context-switching overhead, reducing efficiency  C) No effect on performance  D) Starvation
**Answer: B**

**Q14.** Priority scheduling can lead to starvation of low-priority processes; the common fix is:
A) Ignoring priority entirely  B) Aging — gradually increasing the priority of waiting processes  C) Reducing time quantum  D) Disabling interrupts
**Answer: B**

**Q15.** In Shortest Remaining Time First (SRTF), scheduling decisions are made:
A) Only once at process arrival  B) Preemptively, whenever a new process arrives with a shorter remaining time  C) Never  D) Only at process termination
**Answer: B**

**Q16.** FCFS (First Come First Served) scheduling is:
A) Preemptive  B) Non-preemptive  C) Always optimal  D) Priority-based
**Answer: B**

**Q17.** The "convoy effect" is associated with:
A) Round Robin  B) FCFS, where short processes wait behind one long process  C) SJF  D) Priority scheduling
**Answer: B**

**Q18.** Turnaround time is defined as:
A) Completion time − Arrival time  B) Completion time − Burst time  C) Waiting time only  D) Burst time only
**Answer: A**

**Q19.** Which scheduling algorithm is provably optimal for minimizing average waiting time (given all jobs available at time 0)?
A) FCFS  B) SJF (Shortest Job First)  C) Round Robin  D) Priority scheduling
**Answer: B**

**Q20.** Multi-level feedback queue scheduling allows a process to:
A) Stay in one queue forever  B) Move between queues based on its behavior/CPU burst history  C) Never be scheduled  D) Skip the ready queue entirely
**Answer: B**

---

## Memory Management (Q21–32)

**Q21.** Paging solves which problem in memory management?
A) External fragmentation  B) Internal fragmentation only, and eliminates the need for contiguous allocation  C) CPU scheduling  D) Deadlock
**Answer: B** — Paging eliminates external fragmentation (since pages are placed anywhere in equal-sized frames) but can still have internal fragmentation within the last page.

**Q22.** A page fault occurs when:
A) A page is not currently in physical memory and must be fetched from disk  B) A page is corrupted  C) The CPU is idle  D) Two processes access the same page simultaneously
**Answer: A**

**Q23.** The main advantage of segmentation over paging is:
A) Fixed-size blocks  B) It matches the logical view of a program (code, data, stack as separate segments)  C) No fragmentation at all  D) Faster than paging always
**Answer: B**

**Q24.** Thrashing occurs when:
A) The CPU is used at 100% efficiently  B) A process spends more time paging (swapping) than executing actual instructions  C) A process never accesses memory  D) Two threads share memory perfectly
**Answer: B**

**Q25.** The Least Recently Used (LRU) page replacement algorithm replaces:
A) The most recently used page  B) The page that hasn't been used for the longest time  C) A random page  D) The first page loaded
**Answer: B**

**Q26.** Belady's Anomaly refers to:
A) A situation where increasing the number of page frames increases the number of page faults (occurs in FIFO)  B) A CPU scheduling bug  C) A deadlock scenario  D) A type of segmentation fault
**Answer: A**

**Q27.** Virtual memory allows:
A) Programs to use more memory than physically available RAM, using disk as an extension  B) Faster CPU clock speeds  C) Elimination of all page faults  D) Direct disk access bypassing RAM
**Answer: A**

**Q28.** The Translation Lookaside Buffer (TLB) is used to:
A) Cache recent virtual-to-physical address translations for faster access  B) Store process priorities  C) Schedule processes  D) Manage file systems
**Answer: A**

**Q29.** In the "Optimal Page Replacement" algorithm, the page replaced is the one that:
A) Was used most recently  B) Will not be used for the longest time in the future  C) Was loaded first  D) Is chosen randomly
**Answer: B** — Theoretical best-case, not implementable in practice since it requires future knowledge.

**Q30.** Internal fragmentation occurs when:
A) Free memory is scattered in small non-contiguous blocks between allocated regions  B) Allocated memory block is larger than what's actually needed, wasting space within the block  C) A process cannot be loaded  D) Memory is fully utilized
**Answer: B**

**Q31.** External fragmentation is a common problem in:
A) Paging systems only  B) Contiguous memory allocation / variable partitioning  C) Segmentation only, never  D) Cache memory
**Answer: B**

**Q32.** Which of these is NOT a typical page replacement algorithm?
A) FIFO  B) LRU  C) Optimal  D) Round Robin
**Answer: D** — Round Robin is a CPU scheduling algorithm, not a page replacement algorithm.

---

## Deadlocks (Q33–42)

**Q33.** The four necessary conditions for deadlock are:
A) Mutual exclusion, hold and wait, no preemption, circular wait  B) Priority, scheduling, memory, I/O  C) Atomicity, consistency, isolation, durability  D) Paging, segmentation, caching, virtual memory
**Answer: A**

**Q34.** "Hold and wait" as a deadlock condition means:
A) A process holds at least one resource while waiting to acquire additional resources held by others  B) A process waits without holding anything  C) Resources are never held  D) Two processes swap resources
**Answer: A**

**Q35.** Deadlock prevention differs from deadlock avoidance in that prevention:
A) Ensures at least one of the four necessary conditions never holds  B) Uses runtime resource-allocation state checks (like Banker's Algorithm)  C) Detects deadlock after it occurs  D) Ignores deadlocks entirely
**Answer: A** — Avoidance (like Banker's Algorithm) dynamically checks safe states at runtime; prevention structurally rules out a condition.

**Q36.** The Banker's Algorithm is used for:
A) Deadlock detection  B) Deadlock avoidance by only granting requests that leave the system in a safe state  C) CPU scheduling  D) Memory compaction
**Answer: B**

**Q37.** A "safe state" in the Banker's Algorithm means:
A) The system can allocate resources to all processes in some order without deadlock  B) All resources are currently free  C) No process is waiting  D) The system has crashed
**Answer: A**

**Q38.** Which technique resolves deadlock after it has already occurred?
A) Prevention  B) Avoidance  C) Detection and Recovery  D) None of the above
**Answer: C**

**Q39.** A common way to recover from deadlock is:
A) Ignore it and restart the OS  B) Preempt resources from one or more processes, or terminate one/more deadlocked processes  C) Increase CPU speed  D) Add more RAM
**Answer: B**

**Q40.** Circular wait in deadlock means:
A) Processes form a cycle, each waiting for a resource held by the next  B) A single process waits forever alone  C) No resources are held  D) All processes run in parallel without waiting
**Answer: A**

**Q41.** Which of these can break the "no preemption" condition to prevent deadlock?
A) Never taking resources away from a process  B) Forcibly taking resources away from a process if needed  C) Increasing time quantum  D) Ignoring priority
**Answer: B**

**Q42.** A resource-allocation graph with a cycle in a system with single-instance resources implies:
A) No deadlock  B) A deadlock exists  C) A page fault  D) Thrashing
**Answer: B** — (Note: with multi-instance resources, a cycle indicates a *possible* deadlock, not a guaranteed one.)

---

## File Systems & I/O (Q43–46)

**Q43.** An inode in a Unix file system stores:
A) The actual file content only  B) Metadata about a file (permissions, size, pointers to data blocks) but not the filename  C) Only the filename  D) The file system's total capacity
**Answer: B**

**Q44.** A hard link and a symbolic (soft) link differ in that a symbolic link:
A) Points to another filename/path, and breaks if the target is deleted  B) Shares the same inode as the original file  C) Cannot cross file systems, unlike hard links  D) Is identical to a hard link
**Answer: A** — Hard links share the same inode; symbolic links are separate files that just point to a path.

**Q45.** Disk scheduling algorithms like SCAN and C-SCAN aim to:
A) Minimize seek time by ordering disk head movement efficiently  B) Maximize page faults  C) Schedule CPU processes  D) Manage virtual memory
**Answer: A**

**Q46.** Buffering in I/O operations is primarily used to:
A) Slow down I/O intentionally  B) Handle speed mismatch between devices and smooth out data transfer  C) Delete temporary files  D) Manage process priorities
**Answer: B**

---

## Concurrency & Synchronization (Q47–50)

**Q47.** A semaphore used strictly for mutual exclusion (only takes values 0 or 1) is called a:
A) Counting semaphore  B) Binary semaphore (mutex)  C) Spinlock only  D) Monitor
**Answer: B**

**Q48.** The main purpose of a mutex is to:
A) Allow multiple threads to access a critical section simultaneously  B) Ensure only one thread accesses a critical section at a time  C) Speed up context switching  D) Manage memory allocation
**Answer: B**

**Q49.** A monitor in concurrent programming is:
A) A hardware display device  B) A high-level synchronization construct that encapsulates shared data with mutual exclusion built in  C) A type of scheduler  D) A page replacement algorithm
**Answer: B**

**Q50.** The Producer-Consumer problem is typically solved using:
A) Semaphores (e.g., empty, full, mutex) to coordinate access to a shared bounded buffer  B) Round Robin scheduling  C) Paging  D) Deadlock detection alone
**Answer: A**
