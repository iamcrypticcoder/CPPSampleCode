


# Concurrency in C

Concurrency is the ability of a computer system to execute multiple sequences of instructions simultaneously. This does not necessarily mean they are running at the exact same time (as in parallelism) but that the system manages multiple tasks to appear as though they are being executed at the same time.

Concurrency improves the efficiency and responsiveness of programs.


## 1. Processes vs. Threads

A process is an independent program in execution with its own private virtual address space, file descriptors, and resources. A thread is an independent flow of execution inside a process.

All threads in a process share:
- the same address space (global variables, heap, code),
- open file descriptors,
- signal handlers and the process ID.

Each thread has its own:
- stack (local variables),
- register set and program counter,
- thread ID and errno,
- signal mask and scheduling priority.

| | Process | Thread |
|---|---|---|
| Address space | Private | Shared |
| Creation cost | High | Low |
| Context switch | Expensive | Cheaper |
| Communication | Needs IPC | Shared memory (direct) |
| Fault isolation | Strong (crash is contained) | Weak (one bad thread can crash all) |
| Data sharing | Explicit | Implicit (must synchronize) |

**Use threads** for concurrency within one program that shares data heavily (parallel computation, handling many connections). **Use processes** when you need isolation or fault tolerance. 

The dominant threading API on Linux/Unix is **POSIX threads** (*pthreads*), declared in `<pthread.h>`. C11 also standardized a smaller, portable threading API in `<threads.h>`.


## 2. Creating and Running a Thread

A thread is created with `pthread_create`:

```c
int pthread_create(pthread_t *thread,
                   const pthread_attr_t *attr,
                   void *(*start_routine)(void *),
                   void *arg);
```

- `thread` — output: receives the new thread's ID.
- `attr` — attributes (stack size, detach state…); `NULL` = defaults.
- `start_routine` — the function the thread runs. It takes a `void *`
  and returns a `void *`.
- `arg` — the single argument passed to `start_routine`.
- **Return value:** `0` on success, or a positive error number on failure. Pthreads functions **do not set `errno`** — they *return* the error code. Use `strerror()` to print it.

**Example: Running a single thread**
```c
#include <pthread.h>
#include <stdio.h>
#include <string.h>

void *worker(void *arg) {
    printf("Hello from the worker thread!\n");
    return NULL;
}

int main(void) {
    pthread_t tid;
    int rc = pthread_create(&tid, NULL, worker, NULL);
    if (rc != 0) {
        fprintf(stderr, "pthread_create: %s\n", strerror(rc));
        return 1;
    }
    /* Wait for the worker to finish before main returns, otherwise the
       process (and all its threads) may exit before it runs. */
    pthread_join(tid, NULL);
    printf("Worker finished; main exiting.\n");
    return 0;
}
```

Output:
```
Hello from the worker thread!
Worker finished; main exiting.
```
Why the `pthread_join`? When `main` returns (or the process calls `exit`), the *entire* process terminates, taking every thread with it. Without the join, the program might exit before `worker` ever prints.

**Example: Running many threads**
```c
#include <pthread.h>
#include <stdio.h>

#define N 5

void *say_hi(void *arg) {
    long id = (long)arg;               /* index passed by value */
    printf("Thread %ld reporting in\n", id);
    return NULL;
}

int main(void) {
    pthread_t t[N];
    for (long i = 0; i < N; i++)
        pthread_create(&t[i], NULL, say_hi, (void *)i);
    for (int i = 0; i < N; i++)
        pthread_join(t[i], NULL);

    return 0;
}
```

Output 1:
```
Thread 0 reporting in
Thread 1 reporting in
Thread 3 reporting in
Thread 4 reporting in
Thread 2 reporting in
```
Output 2:
```
Thread 1 reporting in
Thread 3 reporting in
Thread 4 reporting in
Thread 2 reporting in
Thread 0 reporting in
```
The output order is **not deterministic** — the scheduler decides when each thread runs. Never assume an ordering unless you enforce one.

## 3. Passing Arguments and Return Value

**Example: Passing Arguments**
```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    int   id;
    double base;
    char  label[16];
} task_arg;

void *worker(void *arg) {
    task_arg *a = arg;
    printf("[%s] id=%d base=%.1f\n", a->label, a->id, a->base);
    return NULL;
}

int main(void) {
    enum { N = 3 };
    pthread_t t[N];
    task_arg  args[N];                 /* one slot per thread */

    for (int i = 0; i < N; i++) {
        args[i].id   = i;
        args[i].base = i * 1.5;
        snprintf(args[i].label, sizeof args[i].label, "job-%d", i);
        pthread_create(&t[i], NULL, worker, &args[i]);
    }
    for (int i = 0; i < N; i++)
        pthread_join(t[i], NULL);
    return 0;
}
```
**Note:** `args` lives in main's stack, which stays valid until after the joins. If the argument objects were heap-allocated, the thread (or the joiner) would be responsible for freeing them.

**Example: Returning value**
```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

void *square(void *arg) {
    int n = *(int *)arg;
    int *result = malloc(sizeof *result);
    if (result) *result = n * n;
    return result;
}

int main(void) {
    int n = 7;
    pthread_t tid;
    pthread_create(&tid, NULL, square, &n);
    void *ret;
    pthread_join(tid, &ret);
    if (ret) {
        printf("%d squared is %d\n", n, *(int *)ret);
        free(ret); 
    }
    return 0;
}
```

Output:
```
7 squared is 49
```

**Note:** A thread can also finish by calling `pthread_exit(retval)`, which is equivalent to returning `retval` from `start_routine` but works from any depth of the call stack.

## 4. Joining, Waiting, and Detaching
Every thread is created either joinable (the default) or detached. This choice determines how its resources are reclaimed.

`pthread_join` blocks until thread terminates, then stores its return value into *retval (pass NULL to ignore it) and frees the thread's bookkeeping resources.

- Each joinable thread must be joined **exactly once**. Joining the same thread twice, or joining a detached thread, is undefined behavior.
- A joinable thread that is never joined becomes a **zombie** — its resources leak until the process exits. This is the thread equivalent of a zombie process.
- A joinable thread that is never joined becomes a zombie and leaks resources until the process exits. Detaching prevents that for fire-and-forget threads.
- A thread is either joined or detached — never both, and never twice. Calling pthread_join on a detached thread (or detaching twice) is undefined behavior.
- With no join, memory ownership must be explicit: in the example, each worker frees its own argument, since no joiner exists to clean up.
- The whole process exiting kills detached threads mid-run. If they must finish, coordinate shutdown (a semaphore, a condition variable, or an atomic "outstanding tasks" counter) rather than relying on sleep as the demo does

**Example: Waiting with timeout**

```c
#define _GNU_SOURCE          /* required: pthread_timedjoin_np is a GNU ext. */
#include <pthread.h>
#include <errno.h>
#include <stdio.h>
#include <time.h>
#include <unistd.h>

static void *worker(void *arg)
{
    printf("Worker: running for 3 seconds\n");
    sleep(3);
    printf("Worker: finished\n");
    return (void *)(long)99;
}

int main(void)
{
    pthread_t tid;
    pthread_create(&tid, NULL, worker, NULL);

    /* Deadline is ABSOLUTE, not a duration: now + 1 second. */
    struct timespec deadline;
    clock_gettime(CLOCK_REALTIME, &deadline);
    deadline.tv_sec += 1;

    /* Attempt 1: worker needs 3 s, so this times out after 1 s. */
    int rc = pthread_timedjoin_np(tid, NULL, &deadline);
    if (rc == ETIMEDOUT)
        printf("Main:   timed out after 1 s, worker still running\n");

    /* The thread is still joinable and MUST be reaped. Try again with a
       later deadline (now + 5 s), which this time succeeds. */
    clock_gettime(CLOCK_REALTIME, &deadline);
    deadline.tv_sec += 5;

    void *ret;
    rc = pthread_timedjoin_np(tid, &ret, &deadline);
    if (rc == 0)
        printf("Main:   joined in time, worker returned %ld\n", (long)ret);
    else if (rc == ETIMEDOUT)
        printf("Main:   still not done; giving up\n");

    return 0;
}
```

Output:
```
Worker: running for 3 seconds
Main:   timed out after 1 s, worker still running
Worker: finished
Main:   joined in time, worker returned 99
```

**Example: Detach threads**
```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

/* Handle one "request". The argument is heap-allocated by the caller;
   THIS thread owns it and must free it, because nobody will join to
   clean up after it. */
static void *handle_request(void *arg)
{
    int id = *(int *)arg;
    free(arg);                         /* we own the argument now */
    pthread_detach(pthread_self());    /* detach myself: no one joins me */
    printf("  [Worker %d] handling request...\n", id);
    sleep(1);                          /* simulate work */
    printf("  [Worker %d] done; resources auto-released on return\n", id);
    return NULL;                       /* return value is discarded */
}

int main(void)
{
    printf("Server: accepting 5 requests\n");

    for (int i = 1; i <= 5; i++) {
        int *id = malloc(sizeof *id);  /* each thread gets its own arg */
        *id = i;

        pthread_t tid;
        if (pthread_create(&tid, NULL, handle_request, id) != 0) {
            perror("pthread_create");
            free(id);
            continue;
        }
        /* We do NOT join tid. The worker detaches itself, so it cleans
           up on its own. main immediately loops to the next request. */
    }

    /* main can't join anyone, so give the detached workers time to run
       before the process exits (which would kill them). In real code
       this would be the server's normal event loop, not a sleep. */
    sleep(2);
    printf("Server: shutting down\n");
    return 0;
}
```
Output:
```
Server: accepting 5 requests
  [Worker 3] handling request...
  [Worker 2] handling request...
  [Worker 4] handling request...
  [Worker 5] handling request...
  [Worker 1] handling request...
  [Worker 3] done; resources auto-released on return
  [Worker 4] done; resources auto-released on return
  [Worker 2] done; resources auto-released on return
  [Worker 1] done; resources auto-released on return
  [Worker 5] done; resources auto-released on return
Server: shutting down
```

## 5. Terminating and Canceling Threads

A thread ends in one of four ways:
1. Its start_routine returns.
2. It calls `pthread_exit(retval)`.
3. Another thread cancels it with `pthread_cancel`.
4. Any thread calls `exit()` or main returns — this kills the whole process and all threads immediately.

**Example: Cancel thread**
```c
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>

static void *worker(void *arg)
{
    (void)arg;
    for (int i = 1; ; i++) {
        printf("Working... %d\n", i);
        sleep(1);              /* cancellation point: cancel acts here */
    }
    return NULL;              /* never reached */
}

int main(void)
{
    pthread_t tid;
    pthread_create(&tid, NULL, worker, NULL);

    sleep(3);                 /* let it run a few iterations */

    pthread_cancel(tid);      /* request cancellation */
    pthread_join(tid, NULL);  /* wait for it to actually stop */

    printf("Worker canceled\n");
    return 0;
}
```

Output:
```
Working... 1
Working... 2
Working... 3
Worker canceled
```

**Example: Cleanup handlers**
```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

static void cleanup(void *arg) {
    printf("Cleanup: freeing buffer\n");
    free(arg);
}

void *worker(void *arg) {
    (void)arg;
    char *buf = malloc(1024);

    printf("Point 1\n");
    pthread_cleanup_push(cleanup, buf);

    for (int i = 1; i <= 2; i++) {
        sleep(1);
    }

    printf("Point 2\n");
    pthread_cleanup_pop(1);
    return NULL;
}

int main(void) {
    pthread_t tid;
    pthread_create(&tid, NULL, worker, NULL);

    sleep(5);
    pthread_cancel(tid);
    pthread_join(tid, NULL);

    printf("Worker canceled and joined\n");
    return 0;
}
```


## 6. Thread Attributes
The second argument in the `pthread` create function is an attribute object you can customize or leave as default by setting it to NULL.

This is how you can customize the attributes object:
1. Change the thread state to joinable or detached.
2. Set a scheduling policy.
3. Set scheduling priority.
4. Change the size of the stack memory associated with the thread.

Key attributes:

| Attribute | Setter | Meaning |
|---|---|---|
| Detach state | `setdetachstate` | Joinable or detached |
| Stack size | `setstacksize` | Per-thread stack (default ~8 MB on Linux) |
| Stack address | `setstack` | Caller-supplied stack memory |
| Guard size | `setguardsize` | Overflow guard page(s) |
| Scheduling policy | `setschedpolicy` | `SCHED_OTHER`, `SCHED_FIFO`, `SCHED_RR` |
| Scheduling priority | `setschedparam` | Priority within the policy |
| Inherit scheduling | `setinheritsched` | Inherit from creator or use explicit |
| Scope | `setscope` | Contention scope (system/process) |

Each setter has a matching getter (`pthread_attr_getstacksize`, etc.). Setting a large stack matters for deeply recursive threads; setting a small one matters when you spawn thousands of threads and want to save address space.

## 7. Thread Scheduling

**Example: Thread scheduling**
```c
#include "scheduling_thread_2.h"

#define _GNU_SOURCE
#include <pthread.h>
#include <sched.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct {
    int id;
    int prio;
} task_arg;

static void burn(void) {                 /* CPU-bound, no syscalls */
    volatile double x = 0;
    for (long i = 0; i < 300000000L; i++) x += i * 0.5;
}

void *threadFunction(void *arg) {
    task_arg *a = arg;
    printf("  START  id=%d prio=%d\n", a->id, a->prio);
    burn();
    printf("    FINISH id=%d prio=%d\n", a->id, a->prio);
    return NULL;
}

int main(void) {
    enum { N = 5 };
    pthread_t t[N];
    task_arg args[N];
    cpu_set_t cpus;
    struct sched_param param;
    int lo = sched_get_priority_min(SCHED_FIFO);

    /* main runs above every worker, so the loop finishes
       before any worker is allowed to start */
    param.sched_priority = lo + N + 1;
    if (sched_setscheduler(0, SCHED_FIFO, &param) != 0) {
        perror("sched_setscheduler (run with sudo?)");
        exit(1);
    }

    CPU_ZERO(&cpus);
    CPU_SET(0, &cpus);                   /* everyone on CPU 0 */

    for (int i = 0; i < N; i++) {
        pthread_attr_t attr;
        pthread_attr_init(&attr);
        pthread_attr_setinheritsched(&attr, PTHREAD_EXPLICIT_SCHED);  /* the key line */
        pthread_attr_setschedpolicy(&attr, SCHED_FIFO);
        pthread_attr_setaffinity_np(&attr, sizeof(cpus), &cpus);

        args[i].id = i;
        args[i].prio = lo + 1 + i;       /* thread 4 highest, thread 0 lowest */
        param.sched_priority = args[i].prio;
        pthread_attr_setschedparam(&attr, &param);

        int rc = pthread_create(&t[i], &attr, threadFunction, &args[i]);
        if (rc != 0) { fprintf(stderr, "create: %s\n", strerror(rc)); exit(1); }
        pthread_attr_destroy(&attr);
    }

    for (int i = 0; i < N; i++) pthread_join(t[i], NULL);
    return 0;
}
```

**Output:**
```
  START  id=0 prio=2
  START  id=1 prio=3
  START  id=2 prio=4
  START  id=3 prio=5
  START  id=4 prio=6
    FINISH id=4 prio=6
    FINISH id=3 prio=5
    FINISH id=2 prio=4
    FINISH id=1 prio=3
    FINISH id=0 prio=2
```

## 8. Thread Pool

![Thread poll Concept](https://miro.medium.com/v2/resize:fit:720/format:webp/1*bW8heZuEzTDPpw1HmpNZmA.png)

Spawning a fresh thread per task is wasteful when tasks are short and numerous: creation/teardown cost dominates, and unbounded threads thrash the scheduler. A **thread pool** creates a fixed set of worker threads once, then feeds them tasks through a shared queue. This is the producer-consumer pattern.

Components:

-   a **task queue** (protected by a mutex),
-   a **condition variable** so idle workers sleep until work arrives,
-   worker threads that loop: lock → wait for a task → run it → repeat,
-   a **shutdown flag** for graceful termination.

```c
TODO
```

## 9. Thread Synchronization

Because threads share memory, concurrent access to the same data must be coordinated. Without it you get race conditions: results that depend on unpredictable scheduling. The canonical example is counter++, which is really read → add → write — three steps that can interleave between threads and lose updates.

```c
#include <pthread.h>
#include <stdio.h>

static long counter = 0;                 /* shared, unprotected */

void *bump(void *arg) {
    (void)arg;
    for (int i = 0; i < 1000000; i++)
        counter++;                        /* NOT atomic: read-add-write */
    return NULL;
}

int main(void) {
    pthread_t a, b;
    pthread_create(&a, NULL, bump, NULL);
    pthread_create(&b, NULL, bump, NULL);
    pthread_join(a, NULL);
    pthread_join(b, NULL);
    printf("counter = %ld (expected 2000000)\n", counter);
    return 0;
}
```

```
Output:
counter = 1007751 (expected 2000000)
```

### 9.1 Mutexes
A mutex (mutual exclusion lock) ensures that only one thread at a time executes a critical section. Lock before touching shared data, unlock after.

```c
pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;   /* static init */
pthread_mutex_lock(&m);                          /* blocks until acquired */
/* critical section */
pthread_mutex_unlock(&m);
```

```c
#include <pthread.h>
#include <stdio.h>

static long counter = 0;
static pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;

void *bump(void *arg) {
    (void)arg;
    for (int i = 0; i < 1000000; i++) {
        pthread_mutex_lock(&lock);
        counter++;                        /* now protected */
        pthread_mutex_unlock(&lock);
    }
    return NULL;
}

int main(void) {
    pthread_t a, b;
    pthread_create(&a, NULL, bump, NULL);
    pthread_create(&b, NULL, bump, NULL);
    pthread_join(a, NULL);
    pthread_join(b, NULL);
    printf("counter = %ld\n", counter);   /* reliably 2000000 */
    return 0;
}
```

```
Output:
counter = 2000000
```

**Rules of thumb**
- Keep critical sections short — hold the lock only as long as needed.
- Always unlock on every path (including error returns). Cleanup handlers or a single-exit style help.
- A thread that locks a non-recursive mutex it already holds deadlocks itself.
- Protect every access to shared data with the same mutex. One unprotected write anywhere reintroduces the race.

```c
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE);
pthread_mutex_init(&m, &attr);
pthread_mutexattr_destroy(&attr);
```

| Type | Behavior on relocking / errors |
|---|---|
| `PTHREAD_MUTEX_NORMAL` | Fast; self-relock deadlocks; no error checks |
| `PTHREAD_MUTEX_ERRORCHECK` | Returns `EDEADLK` on self-relock; catches bugs |
| `PTHREAD_MUTEX_RECURSIVE` | Same thread may lock N times; must unlock N times |
| `PTHREAD_MUTEX_DEFAULT` | Implementation-defined (usually NORMAL) |

A **recursive** mutex lets one thread lock it repeatedly — useful when a locked function calls another function that locks the same mutex. It keeps an ownership count and only releases on the final unlock. Use sparingly; needing recursion often signals a design that could be restructured.

Other useful mutex attributes: `setrobust` (recover a lock whose owner died) and `setprotocol` (priority inheritance, to combat priority inversion in real-time systems).

### 9.2 Condition Variables

**Example: Condition variables**

A mutex protects data. A condition variable lets threads wait for a state without busy-polling. It always pairs with a mutex and a predicate (a boolean condition on shared data).

```c
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>

/* --- the three things that ALWAYS go together --- */
static int             ready = 0;                          /* 1. the DATA  */
static pthread_mutex_t lock  = PTHREAD_MUTEX_INITIALIZER;  /* 2. its LOCK  */
static pthread_cond_t  cond  = PTHREAD_COND_INITIALIZER;   /* 3. the SIGNAL*/

static void* worker1(void *arg) {
    (void)arg;
    printf("[Worker 1] Working for 5 seconds...\n");
    sleep(5);

    pthread_mutex_lock(&lock);
    ready = 1;
    printf("[Worker 1] ready = 1, signaling\n");
    pthread_cond_signal(&cond);
    pthread_mutex_unlock(&lock);
    return NULL;
}

static void* worker2(void *arg) {
    pthread_mutex_lock(&lock);
    while (!ready) {
        printf("[Worker 2] Ready value is 0. Waiting for value change\n");
        pthread_cond_wait(&cond, &lock);    // Unlock and sleep until signal received
        printf("[Worker 2] Woke up. Rechecking ready\n");
    }
    printf("[Worker 2] Out of while loop. Notice your while loop wasn't running all time.\n");
    pthread_mutex_unlock(&lock);
}

int main(void) {
    pthread_t tid1, tid2;
    pthread_create(&tid1, NULL, worker1, NULL);
    pthread_create(&tid2, NULL, worker2, NULL);
    pthread_join(tid1, NULL);
    pthread_join(tid2, NULL);
}
```

Output:
```
[Worker 1] Working for 5 seconds...
[Worker 2] Ready value is 0. Waiting for value change
[Worker 1] ready = 1, signaling
[Worker 2] Woke up. Rechecking ready
[Worker 2] Out of while loop. Notice your while loop wasn't running all time.
```

Ref:
- https://www.geeksforgeeks.org/linux-unix/condition-wait-signal-multi-threading/

### 9.3 Read-Write Locks

A read-write lock in C using the POSIX threads (pthreads) library allows multiple threads to read a shared resource simultaneously while ensuring that only one thread can write at a time. This is particularly useful when the number of read operations is much higher than write operations, as it reduces the overhead of locking. The functions `pthread_rwlock_t, pthread_rwlock_rdlock, pthread_rwlock_wrlock`, and `pthread_rwlock_unlock` are used to manage this type of lock.

Here is a concise example of how to use pthread_rwlock_t and related functions in C:
- Initialize the read-write lock using PTHREAD_RWLOCK_INITIALIZER for static allocation or pthread_rwlock_init() for dynamic allocation.
- Acquire a read lock using pthread_rwlock_rdlock() when a thread needs to read the shared resource.
- Acquire a write lock using pthread_rwlock_wrlock() when a thread needs to modify the shared resource.
- Release the lock using pthread_rwlock_unlock() after the operation is complete.
- Destroy the lock using pthread_rwlock_destroy() when it is no longer needed.


**Example: Read Write Locks**

When data is read often but written rarely, a mutex is overly strict — it blocks readers that could safely run concurrently. A read-write lock allows many simultaneous readers but gives writers exclusive access.

```c
#include <stdio.h>
#include <pthread.h>
#include <stdlib.h>
#include <unistd.h>

pthread_rwlock_t lock = PTHREAD_RWLOCK_INITIALIZER;
int shared_resource = 0;

void* reader(void* arg) {
    int id = *(int*)arg;
    for (int i = 0; i < 3; i++) {
        pthread_rwlock_rdlock(&lock); // Acquire read lock
        printf("Reader %d: Read value = %d\n", id, shared_resource);
        pthread_rwlock_unlock(&lock); // Release read lock
        usleep(rand() % 100000); // Simulate work
    }
    return NULL;
}

void* writer(void* arg) {
    int id = *(int*)arg;
    for (int i = 0; i < 2; i++) {
        pthread_rwlock_wrlock(&lock); // Acquire write lock
        shared_resource++;
        printf("Writer %d: Updated value = %d\n", id, shared_resource);
        pthread_rwlock_unlock(&lock); // Release write lock
        usleep(rand() % 100000); // Simulate work
    }
    return NULL;
}

int main() {
    pthread_t readers[5], writers[2];
    int reader_ids[5], writer_ids[2];

    // Create reader threads
    for (int i = 0; i < 5; i++) {
        reader_ids[i] = i + 1;
        pthread_create(&readers[i], NULL, reader, &reader_ids[i]);
    }

    // Create writer threads
    for (int i = 0; i < 2; i++) {
        writer_ids[i] = i + 1;
        pthread_create(&writers[i], NULL, writer, &writer_ids[i]);
    }

    // Wait for all threads to finish
    for (int i = 0; i < 5; i++) {
        pthread_join(readers[i], NULL);
    }
    for (int i = 0; i < 2; i++) {
        pthread_join(writers[i], NULL);
    }

    // Destroy the read-write lock
    pthread_rwlock_destroy(&lock);

    return 0;
}
```

Output:
```
Reader 4: Read value = 0
Reader 1: Read value = 0
Reader 5: Read value = 0
Reader 3: Read value = 0
Reader 2: Read value = 0
Writer 1: Updated value = 1
Writer 2: Updated value = 2
Reader 1: Read value = 2
Reader 3: Read value = 2
Writer 1: Updated value = 3
Reader 2: Read value = 3
Reader 2: Read value = 3
Reader 3: Read value = 3
Writer 2: Updated value = 4
Reader 4: Read value = 4
Reader 1: Read value = 4
Reader 5: Read value = 4
Reader 5: Read value = 4
Reader 4: Read value = 4
```

Ref: 
- https://stackoverflow.com/questions/12033188/how-would-you-implement-your-own-reader-writer-lock-in-c11


### 9.4 Barriers

A **barrier** makes a group of threads wait until _all_ of them arrive, then releases them together — ideal for phased/iterative parallel algorithms where every thread must finish phase _N_ before any starts phase _N+1_.

```c
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>

#define N 3
static pthread_barrier_t bar;

void *worker(void *arg) {
    long threadNum = (long) arg;
    time_t start, end;

    time(&start);
    printf("Thread %ld: before sleep at %s", threadNum, ctime(&start));
    // Sleep for a random time
    int nsecs = rand() % 6 + 1;
    sleep(nsecs);
    time(&end);
    printf("Thread %ld: after sleep at %s", threadNum, ctime(&end));

    pthread_barrier_wait(&bar);         /* rendezvous */
    printf("thread %ld: phase 2 (all synced)\n", threadNum);
    return NULL;
}

int main(void) {
    srand(time(0));
    pthread_t t[N];
    pthread_barrier_init(&bar, NULL, N);
    for (long i = 0; i < N; i++)
        pthread_create(&t[i], NULL, worker, (void *)i);
    for (int i = 0; i < N; i++)
        pthread_join(t[i], NULL);
    pthread_barrier_destroy(&bar);
    return 0;
}
```

Output:
```
Thread 2: before sleep at Mon Jul 13 15:41:37 2026
Thread 0: before sleep at Mon Jul 13 15:41:37 2026
Thread 1: before sleep at Mon Jul 13 15:41:37 2026
Thread 2: after sleep at Mon Jul 13 15:41:39 2026
Thread 0: after sleep at Mon Jul 13 15:41:42 2026
Thread 1: after sleep at Mon Jul 13 15:41:42 2026
thread 1: phase 2 (all synced)
thread 0: phase 2 (all synced)
thread 2: phase 2 (all synced)
```

### 9.5 Semaphores

A **semaphore** is a counter with two atomic operations: _wait_ (decrement, blocking at zero) and _post_ (increment). It generalizes a mutex — a binary semaphore (count 0/1) acts like a lock, while a counting semaphore limits access to _N_ identical resources. POSIX semaphores live in `<semaphore.h>`

Semaphore usage pattern:
```c
#include <semaphore.h>

sem_t sem;
sem_init(&sem, 0, 3);      /* pshared=0 (threads), initial value 3 */

sem_wait(&sem);            /* P: decrement; block while 0 */
/* use one of the 3 permits */
sem_post(&sem);            /* V: increment; wake a waiter */

sem_destroy(&sem);
```

A semaphore limiting concurrency to 3:

```c
#include <pthread.h>
#include <semaphore.h>
#include <stdio.h>
#include <unistd.h>

static sem_t slots;                    /* at most 3 in the pool area */

void *worker(void *arg) {
    long id = (long)arg;
    sem_wait(&slots);                  /* acquire a permit */
    printf("Worker %ld entered\n", id);
    sleep(1);                          /* hold the resource */
    printf("Worker %ld leaving\n", id);
    sem_post(&slots);                  /* release the permit */
    return NULL;
}

int main(void) {
    enum { N = 8 };
    pthread_t t[N];
    sem_init(&slots, 0, 3);            /* only 3 concurrent clients */
    for (long i = 0; i < N; i++)
        pthread_create(&t[i], NULL, worker, (void *)i);
    for (int i = 0; i < N; i++)
        pthread_join(t[i], NULL);
    sem_destroy(&slots);
    return 0;
}

```

Output:
```
Worker 0 entered
Worker 4 entered
Worker 1 entered
Worker 0 leaving
Worker 4 leaving
Worker 1 leaving
Worker 2 entered
Worker 3 entered
Worker 6 entered
Worker 2 leaving
Worker 3 leaving
Worker 6 leaving
Worker 7 entered
Worker 5 entered
Worker 7 leaving
Worker 5 leaving
```

### 9.6 Atomic Operations
For simple shared variables (counters, flags), locking is overkill. C11 `<stdatomic.h>` provides **atomic types** whose operations are indivisible and properly ordered across threads — no mutex required.

**Basic Usage:**
```c
#include <stdatomic.h>

atomic_int counter = 0;

atomic_fetch_add(&counter, 1);      /* counter += 1, atomically */
int v = atomic_load(&counter);      /* read */
atomic_store(&counter, 0);          /* write */

/* Compare-and-swap: the backbone of lock-free algorithms */
int expected = v;
atomic_compare_exchange_strong(&counter, &expected, v + 1);
```

Rewriting the race-free counter without any lock:

```c
/* atomic.c — gcc -Wall atomic.c -o atomic -pthread */
#include <pthread.h>
#include <stdatomic.h>
#include <stdio.h>

static atomic_long counter = 0;

void *bump(void *arg) {
    (void)arg;
    for (int i = 0; i < 1000000; i++)
        atomic_fetch_add(&counter, 1);      /* lock-free, correct */
    return NULL;
}

int main(void) {
    pthread_t a, b;
    pthread_create(&a, NULL, bump, NULL);
    pthread_create(&b, NULL, bump, NULL);
    pthread_join(a, NULL);
    pthread_join(b, NULL);
    printf("counter = %ld\n", atomic_load(&counter));  /* 2000000 */
    return 0;
}
```

Output:
```
counter = 2000000
```

### 9.7 Thread Local Storage

Thread local storage (TLS) is a feature introduced in C++ 11 that allows each thread in a multi-threaded program to have its own separate instance of a variable. In simple words, we can say that each thread can have its own independent instance of a variable. Each thread can access and modify its own copy of the variable without interfering with other threads.

Properties of Thread Local Storage (TLS)

- Lifetime: The lifetime of a TLS variable begins when it is initialized and ends when the thread terminates.
- Visibility: TLS variables have visibility at the thread level.
- Scope: TLS variables have scope depending on where they are declared
    
```c
#include <pthread.h>
#include <stdio.h>
#include <threads.h>

thread_local int calls = 0;         /* independent in every thread */

void *worker(void *arg) {
    for (int i = 0; i < (int)(long)arg; i++)
        calls++;                    /* touches only THIS thread's copy */
    printf("thread %lu made %d calls\n",
           (unsigned long)pthread_self(), calls);
    return NULL;
}

int main(void) {
    pthread_t a, b;
    pthread_create(&a, NULL, worker, (void *)3);
    pthread_create(&b, NULL, worker, (void *)7);
    pthread_join(a, NULL);          /* prints 3 */
    pthread_join(b, NULL);          /* prints 7 */
    return 0;
}
```

Output:
```
thread 136721526163008 made 3 calls
thread 136721515677248 made 7 calls
```

Reference:
- https://www.geeksforgeeks.org/cpp/thread_local-storage-in-cpp-11/

### 9.8 One-Time Initialization

To run an initializer exactly once even when many threads race to trigger it, use `pthread_once`:

```c
static pthread_once_t once = PTHREAD_ONCE_INIT;
static FILE *log_file;

static void open_log(void) {              /* runs exactly one time */
    log_file = fopen("app.log", "a");
}

void log_msg(const char *s) {
    pthread_once(&once, open_log);        /* first caller runs it; */
    fprintf(log_file, "%s\n", s);         /* others wait then proceed */
}
```

```cpp
#include <pthread.h>
#include <stdio.h>

static pthread_once_t once = PTHREAD_ONCE_INIT;
static int config;

static void init_config(void) {
    config = 42;
    printf("init_config ran once\n");
}

static int worker(void *arg) {
    int id = *(int *)arg;
    pthread_once(&once, init_config);
    printf("worker %d sees config=%d\n", id, config);
    return 0;
}

int main(void) {
    enum { N = 5 };
    pthread_t t[N];
    int id[N];

    for (int i = 0; i < N; i++) {
        id[i] = i;
        pthread_create(&t[i], NULL, worker, &id[i]);
    }
    for (int i = 0; i < N; i++)
        pthread_join(t[i], NULL);

    return 0;
}
```
The C11 equivalent is `call_once` with a `once_flag`. This is the correct, race-free way to do lazy singleton initialization.

```c
#include <pthread.h>
#include <threads.h>
#include <stdio.h>

static once_flag init_flag = ONCE_FLAG_INIT;   /* mandatory static initializer */
static int config;

static void init_config(void)
{
    config = 42;                       /* expensive one-time setup */
    printf("init_config ran once\n");
}

static int worker(void *arg)
{
    int id = *(int *)arg;
    call_once(&init_flag, init_config); /* only the first caller runs init_config */
    printf("worker %d sees config=%d\n", id, config);
    return 0;
}

int main(void)
{
    enum { N = 5 };
    pthread_t t[N];
    int id[N];

    for (int i = 0; i < N; i++) {
        id[i] = i;
        thrd_create(&t[i], worker, &id[i]);
    }
    for (int i = 0; i < N; i++)
        thrd_join(t[i], NULL);

    return 0;
}
```

Output:
```
init_config ran once
worker 0 sees config=42
worker 4 sees config=42
worker 2 sees config=42
worker 1 sees config=42
worker 3 sees config=42
```

Reference:
- https://stackoverflow.com/questions/1228025/pthread-key-t-and-pthread-once-t


## 10. Deadlocks and Race Conditions

**Race condition** — the result depends on unsynchronized timing between threads (like the counter++ example). Fix by protecting all shared access with the same synchronization, or by using atomics.

**Deadlock** — threads wait on each other forever. The classic case is lock-ordering: thread 1 holds A and wants B; thread 2 holds B and wants A. Neither can proceed.

```c
/* DEADLOCK: two threads acquire two mutexes in opposite orders */
void *t1(void *_) {
    pthread_mutex_lock(&A);
    pthread_mutex_lock(&B);   /* waits for t2 to release B */
    ...
}
void *t2(void *_) {
    pthread_mutex_lock(&B);
    pthread_mutex_lock(&A);   /* waits for t1 to release A */
    ...
}
```

## 11. C11 `<threads.h>`

C11 standardized a threading API independent of POSIX. It's a thin,
portable layer (a strict subset of pthreads functionality) — useful for
code that must compile on non-POSIX platforms.

Naming maps closely to pthreads:

| POSIX | C11 `<threads.h>` |
|---|---|
| `pthread_t` | `thrd_t` |
| `pthread_create` | `thrd_create` |
| `pthread_join` | `thrd_join` |
| `pthread_detach` | `thrd_detach` |
| `pthread_exit` | `thrd_exit` |
| `pthread_mutex_t` | `mtx_t` |
| `pthread_mutex_lock` | `mtx_lock` |
| `pthread_cond_t` | `cnd_t` |
| `pthread_cond_wait` | `cnd_wait` |
| `pthread_key_t` / TLS | `tss_t`, `thread_local` |
| `pthread_once` | `call_once` |

A key difference: a C11 thread function has signature `int (*)(void *)` — it takes a `void *` and returns an `int` (not a `void *`).

```c
/* c11threads.c — gcc -Wall c11threads.c -o c11threads -pthread */
#include <threads.h>
#include <stdio.h>

int worker(void *arg) {
    int n = *(int *)arg;
    printf("C11 thread got %d\n", n);
    return n * 2;                       /* returned via thrd_join */
}

int main(void) {
    thrd_t t;
    int arg = 21;

    if (thrd_create(&t, worker, &arg) != thrd_success) {
        fprintf(stderr, "thrd_create failed\n");
        return 1;
    }

    int result;
    thrd_join(t, &result);              /* result <- worker's return */
    printf("thread returned %d\n", result);
    return 0;
}
```


