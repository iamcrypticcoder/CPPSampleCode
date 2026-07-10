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
    int *result = malloc(sizeof *result);   /* heap: survives the thread */
    if (result) *result = n * n;
    return result;                           /* becomes join's retval  */
}

int main(void) {
    int n = 7;
    pthread_t tid;
    pthread_create(&tid, NULL, square, &n);
    void *ret;
    pthread_join(tid, &ret);                 /* ret <- what square returned */
    if (ret) {
        printf("%d squared is %d\n", n, *(int *)ret);
        free(ret);                           /* Don't forget to free the allocated memory */
    }
    return 0;
}
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
    printf("worker: running for 3 seconds\n");
    sleep(3);
    printf("worker: finished\n");
    return (void *)(long)99;
}

int main(void)
{
    pthread_t tid;
    pthread_create(&tid, NULL, worker, NULL);

    // Deadline is ABSOLUTE, not a duration: now + 1 second.
    struct timespec deadline;
    clock_gettime(CLOCK_REALTIME, &deadline);
    deadline.tv_sec += 1;

    // Attempt 1: worker needs 3 s, so this times out after 1s
    int rc = pthread_timedjoin_np(tid, NULL, &deadline);
    if (rc == ETIMEDOUT)
        printf("main:   timed out after 1 s, worker still running\n");

    // The thread is still joinable and MUST be reaped.
    // Try again with a later deadline (now + 5 s), which this time succeeds.
    clock_gettime(CLOCK_REALTIME, &deadline);
    deadline.tv_sec += 5;

    void *ret;
    rc = pthread_timedjoin_np(tid, &ret, &deadline);
    if (rc == 0)
        printf("main:   joined in time, worker returned %ld\n", (long)ret);
    else if (rc == ETIMEDOUT)
        printf("main:   still not done; giving up\n");

    return 0;
}
```

**Example: Detach threads**
```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

static void *handle_request(void *arg)
{
    int id = *(int *)arg;
    free(arg);                         /* we own the argument now */
    pthread_detach(pthread_self());    /* detach myself: no one joins me */
    printf("  [worker %d] handling request...\n", id);
    sleep(1);                          /* simulate work */
    printf("  [worker %d] done; resources auto-released on return\n", id);
    return NULL;                       /* return value is discarded */
}

int main(void)
{
    printf("server: accepting 5 requests\n");
    for (int i = 1; i <= 5; i++) {
        int *id = malloc(sizeof *id);  /* each thread gets its own arg */
        *id = i;

        pthread_t tid;
        if (pthread_create(&tid, NULL, handle_request, id) != 0) {
            perror("pthread_create");
            free(id);
            continue;
        }
    }
    sleep(2);
    printf("server: shutting down\n");
    return 0;
}
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
        printf("working... %d\n", i);
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

    printf("worker canceled\n");
    return 0;
}
```

**Example: Cleanup handlers**
```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

static void cleanup(void *arg) {
    printf("cleanup: freeing buffer\n");
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

    sleep(2);
    pthread_cancel(tid);
    pthread_join(tid, NULL);         

    printf("worker canceled and joined\n");
    return 0;
}
```


## 5. Thread Attributes
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

## 6. Thread Scheduling

**Example: Thread scheduling**
```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <sched.h>
#include <unistd.h>

typedef struct {
    int id;
} task_arg;

void* threadFunction(void* arg) {
    int id = ((task_arg*)arg)->id;
    printf("Thread is running id = %d\n", id);
    sleep(2);
    printf("Thread is finishing id = %d\n", id);
    pthread_exit(NULL);
}

int main(void) {
    enum { N = 5 };
    pthread_t t[N];
    task_arg args[N];

    pthread_attr_t attr;
    struct sched_param param;
    int policy;

    // Initialize the attribute object
    pthread_attr_init(&attr);

    // Set the scheduling policy to SCHED_FIFO
    pthread_attr_setschedpolicy(&attr, SCHED_FIFO);

    // Set the priority
    int maxPriority = sched_get_priority_max(SCHED_FIFO);
    int minPriority = sched_get_priority_min(SCHED_FIFO);
    param.sched_priority = (maxPriority + minPriority) / 2;
    pthread_attr_setschedparam(&attr, &param);

    int status = pthread_attr_setstacksize(&attr, 1024 * 1024);
    if (status != 0) {
        fprintf(stderr, "Error setting stack size\n");
        exit(EXIT_FAILURE);
    }
    
    // Create the thread with the specified attributes
    for (int i = 0; i < N; i++) {
        args[i].id = i;
        int status = pthread_create(&t[i], &attr, threadFunction, &args[i]);
        if (status != 0) {
            printf("Error creating thread\n");
            exit(-1);
        }
    }

    // Destroy the attribute object
    pthread_attr_destroy(&attr);

    // Wait for the thread to finish
    for (int i = 0; i < N; i++)
        pthread_join(t[i], NULL);
    return 0;
}
```

## 7. Thread Pool

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

## 8. Thread Synchronization

Because threads share memory, concurrent access to the same data must be coordinated. Without it you get race conditions: results that depend on unpredictable scheduling. The canonical example is counter++, which is really read → add → write — three steps that can interleave between threads and lose updates.

```c
/* race.c — demonstrates a lost-update race
   Two threads each increment a shared counter 1,000,000 times.
   The final value is almost never 2,000,000.                       */
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

### 8.1 Mutexes
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

### 8.2 Condition Variables

**Example: Condition variables**

```c
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>

/* --- the three things that ALWAYS go together --- */
static int             ready = 0;                          /* 1. the DATA  */
static pthread_mutex_t lock  = PTHREAD_MUTEX_INITIALIZER;  /* 2. its LOCK  */
static pthread_cond_t  cond  = PTHREAD_COND_INITIALIZER;   /* 3. the SIGNAL*/

static void *worker(void *arg)
{
    (void)arg;
    printf("[worker] preparing data (5 s)...\n");
    sleep(5);

    pthread_mutex_lock(&lock);      /* lock before touching `ready` */
    ready = 1;                      /* change the state             */
    printf("[worker] ready = 1, signaling\n");
    pthread_cond_signal(&cond);     /* wake a waiter                */
    pthread_mutex_unlock(&lock);    /* now the waiter can proceed   */
    return NULL;
}

int main(void)
{
    pthread_t tid;
    pthread_create(&tid, NULL, worker, NULL);

    pthread_mutex_lock(&lock);      /* lock before testing `ready` */
    while (!ready) {                /* ALWAYS a while, never an if */
        printf("[main]   ready == 0, going to sleep\n");
        pthread_cond_wait(&cond, &lock);   /* unlock + sleep, then relock */
        printf("[main]   woke up, re-checking ready\n");
    }
    printf("[main]   ready == 1, proceeding\n");
    pthread_mutex_unlock(&lock);
    pthread_join(tid, NULL);
    return 0;
}
```

Ref:
- https://www.geeksforgeeks.org/linux-unix/condition-wait-signal-multi-threading/

## 8.3 Read-Write Locks

A read-write lock in C using the POSIX threads (pthreads) library allows multiple threads to read a shared resource simultaneously while ensuring that only one thread can write at a time. This is particularly useful when the number of read operations is much higher than write operations, as it reduces the overhead of locking. The functions `pthread_rwlock_t, pthread_rwlock_rdlock, pthread_rwlock_wrlock`, and `pthread_rwlock_unlock` are used to manage this type of lock.

Here is a concise example of how to use pthread_rwlock_t and related functions in C:
- Initialize the read-write lock using PTHREAD_RWLOCK_INITIALIZER for static allocation or pthread_rwlock_init() for dynamic allocation.
- Acquire a read lock using pthread_rwlock_rdlock() when a thread needs to read the shared resource.
- Acquire a write lock using pthread_rwlock_wrlock() when a thread needs to modify the shared resource.
- Release the lock using pthread_rwlock_unlock() after the operation is complete.
- Destroy the lock using pthread_rwlock_destroy() when it is no longer needed.


**Example: Read Write Locks**

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

Ref: 
- https://stackoverflow.com/questions/12033188/how-would-you-implement-your-own-reader-writer-lock-in-c11


## 8.4 Barriers

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
    int nsecs = random() % 5 + 1;
    sleep(nsecs);
    time(&end);
    printf("Thread %ld: after sleep at %s", threadNum, ctime(&end));

    pthread_barrier_wait(&bar);         /* rendezvous */
    printf("thread %ld: phase 2 (all synced)\n", threadNum);
    return NULL;
}

int main(void) {
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

## 8.5 Semaphores
