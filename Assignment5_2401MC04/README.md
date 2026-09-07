# Operating Systems Lab — Assignment 5

## Process Synchronization using xv6

**Roll Number:** 2401MC04
**Name:** Priyam Das

---

## Overview

This assignment implements classical process synchronization problems inside the
xv6 operating system.

The implemented problems are:

1. **Mutual Exclusion using Peterson's Algorithm**
2. **Producer-Consumer Problem using a Bounded Buffer**
3. **Readers-Writers Problem**
4. **Dining Philosophers Problem** — _Not implemented_

The implementations require modifications to the xv6 kernel as well as the
addition of user-level programs. Shared memory and kernel-level semaphore
system calls were added to support synchronization between processes.

---

## Repository Structure

```text
Assignment5_2401MC04/
│
├── Kernel_2401MC04/
│   ├── syscall.c
│   ├── syscall.h
│   ├── sysproc.c
│   └── vm.c
│
├── User_2401MC04/
│   ├── peterson.c
│   ├── prodcons.c
│   ├── readwrite.c
│   ├── user.h
│   └── usys.S
│
├── Output_logs_2401MC04/
│   └── transcript.txt
│
├── Makefile
└── Assignment_5_Process_Synchronization_xv6.md.pdf
```

---

# Kernel Modifications

Two main mechanisms were added to xv6 to support the synchronization programs.

## 1. Shared Memory

A new `shm_get()` system call was added.

The kernel maintains a single shared physical page and maps it into the
address space of processes at the virtual address:

```text
0x60000000
```

The same physical page is therefore accessible by multiple processes, allowing
the user programs to maintain shared variables such as counters, buffer
indices, reader counts, and synchronization state.

The shared-memory implementation is located primarily in:

```text
Kernel_2401MC04/vm.c
Kernel_2401MC04/sysproc.c
Kernel_2401MC04/syscall.c
```

---

## 2. Semaphore System Calls

Three semaphore-related system calls were added:

```text
sem_init(id, value)
sem_wait(id)
sem_post(id)
```

The kernel maintains an array of semaphores. Each semaphore contains:

- A spinlock protecting the semaphore state
- A count representing the available resources

When `sem_wait()` is called while the count is zero, the calling process
sleeps using xv6's `sleep()` mechanism.

When `sem_post()` increments the count, waiting processes are woken using
`wakeup()`.

This provides blocking counting-semaphore behavior using xv6's existing
process sleep/wakeup primitives.

---

# Question 1 — Peterson's Algorithm

## Objective

Peterson's Algorithm is used to provide mutual exclusion between two processes
without using an OS-provided lock for the actual critical-section algorithm.

The implementation uses the following shared variables:

```text
flag[2]
turn
counter
```

The parent creates one child process. Both processes execute Peterson's
algorithm for 10 iterations.

Each process:

1. Sets its `flag` to indicate that it wants to enter the critical section.
2. Sets `turn` to the other process.
3. Busy-waits while the other process wants to enter and it is the other
   process's turn.
4. Enters the critical section.
5. Increments the shared counter.
6. Prints its process ID and the counter value.
7. Clears its flag.
8. Executes a remainder section before trying again.

The shared memory is obtained using:

```c
shm_get();
```

The shared counter is expected to reach:

```text
20
```

because both processes execute the critical section 10 times.

### Source

```text
User_2401MC04/peterson.c
```

### Running

After starting xv6:

```text
$ peterson
```

### Expected Result

The output should show alternating/interleaved critical-section execution while
the counter increases without lost updates.

The recorded execution reaches:

```text
Final counter = 20
```

The complete execution log is available in:

```text
Output_logs_2401MC04/transcript.txt
```

---

# Question 2 — Producer-Consumer Problem

## Objective

The Producer-Consumer problem is implemented using a fixed-size circular
buffer shared between a producer and a consumer process.

The default buffer size is:

```text
5
```

The buffer size can also be supplied as a command-line argument.

For example:

```text
$ prodcons 5
```

The producer generates the integers:

```text
1 ... 20
```

and inserts them into the circular buffer.

The consumer removes the values and prints them.

---

## Synchronization

Three semaphores are used:

| Semaphore | Purpose                                              |
| --------- | ---------------------------------------------------- |
| `empty`   | Number of empty slots in the buffer                  |
| `full`    | Number of occupied slots                             |
| `mutex`   | Provides mutual exclusion while accessing the buffer |

They are initialized as:

```text
empty = buffer size
full  = 0
mutex = 1
```

The producer performs:

```text
wait(empty)
wait(mutex)

insert item

post(mutex)
post(full)
```

The consumer performs:

```text
wait(full)
wait(mutex)

remove item

post(mutex)
post(empty)
```

The buffer uses two shared indices:

```text
in
out
```

Both indices wrap around using the configured buffer size, making the buffer
circular.

### Source

```text
User_2401MC04/prodcons.c
```

### Running

```text
$ prodcons
```

or with a custom buffer size:

```text
$ prodcons 5
```

### Result

The recorded output demonstrates production and consumption of all 20 items:

```text
Produced: 1
Consumed: 1
Produced: 2
Produced: 3
...
Produced: 20
Consumed: 20
```

The producer and consumer do not access the shared buffer simultaneously
because access to the buffer is protected by the mutex semaphore.

---

# Question 3 — Readers-Writers Problem

## Objective

The Readers-Writers problem is implemented using a **readers-priority**
solution.

The implementation creates:

```text
3 reader processes
2 writer processes
```

The shared state consists of:

```text
shared_data
read_count
```

Both variables are stored in the shared-memory page.

---

## Synchronization

Three semaphores are used:

```text
sem 3 → protects read_count
sem 4 → controls access to shared_data
sem 5 → provides additional synchronization between readers and writers
```

### Reader

When a reader enters:

1. It acquires the reader-count protection.
2. Increments `read_count`.
3. If it is the first reader, it acquires the writer lock.
4. Releases the reader-count protection.
5. Reads `shared_data`.
6. After reading, decrements `read_count`.
7. If it is the last reader, it releases the writer lock.

This allows multiple readers to access `shared_data` concurrently.

### Writer

A writer acquires exclusive access before modifying `shared_data`.

It:

1. Acquires the writer lock.
2. Increments `shared_data`.
3. Prints the updated value.
4. Releases the writer lock.

Therefore, writers cannot execute concurrently with readers or another writer.

### Source

```text
User_2401MC04/readwrite.c
```

### Running

```text
$ readwrite
```

### Recorded Result

The execution demonstrates multiple readers accessing the same value:

```text
Reader 6 reads data = 0, active readers = 1
Reader 7 reads data = 0, active readers = 2
Reader 8 reads data = 0, active readers = 3
```

followed by exclusive writer updates:

```text
Writer 9 writes data = 1
Writer 10 writes data = 2
```

The complete execution is available in:

```text
Output_logs_2401MC04/transcript.txt
```

---

# Question 4 — Dining Philosophers

## Status

**Not implemented.**

The Dining Philosophers problem was part of the assignment specification but
has not been implemented in the current version of this repository.

---

# Building and Running xv6

Navigate to the directory containing the xv6 `Makefile`:

```bash
cd Assignment5_2401MC04
```

Clean previous build artifacts:

```bash
make clean
```

Build xv6:

```bash
make
```

Start xv6 using QEMU:

```bash
make qemu
```

Once the xv6 shell starts, the implemented programs can be executed using:

```text
$ peterson
$ prodcons
$ readwrite
```

---

# User Programs Added

The `Makefile` has been updated to build the following user programs:

```text
_peterson
_prodcons
_readwrite
```

These programs are therefore included in the xv6 filesystem during the build
process.

---

# Output Logs

Execution output is stored in:

```text
Output_logs_2401MC04/transcript.txt
```

The transcript contains QEMU execution logs for:

- Peterson's Algorithm
- Producer-Consumer
- Readers-Writers

---

# Summary

| Question | Problem              | Status          |
| -------- | -------------------- | --------------- |
| Q1       | Peterson's Algorithm | Implemented     |
| Q2       | Producer-Consumer    | Implemented     |
| Q3       | Readers-Writers      | Implemented     |
| Q4       | Dining Philosophers  | Not Implemented |

This assignment demonstrates process synchronization in xv6 through shared
memory, Peterson's mutual-exclusion algorithm, counting semaphores, circular
buffers, and the Readers-Writers synchronization problem.
